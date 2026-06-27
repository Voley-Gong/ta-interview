---
title: "实时体积云：如何在手游中实现可交互的动态云层渲染？"
category: "rendering"
level: 4
tags: ["VolumetricCloud", "RayMarching", "噪声", "WeatherTexture", "URP"]
hint: "核心是光线步进采样 3D 噪声纹理重建云体——但真正的难点是性能预算与视觉品质的平衡"
related: ["rendering/urp-volumetric-fog", "optimization/mobile-overheating-gpu-analysis", "rendering/forward-plus-cluster"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的开放世界游戏需要动态体积云——云会随时间变化形状、被风吹动、能影响地面阴影，玩家能飞到云层中。PC 端要高品质，移动端要能跑。给我你的方案和技术选型。」

### ✅ 核心要点

1. **Ray Marching 重建云体**：从相机发射光线，在云层高度范围内步进采样 3D 噪声
2. **FBM 分形噪声**：多层 Perlin/Worley 噪声叠加，模拟真实云的细节层次
3. **Weather Texture 控制大局**：用 2D 贴图控制云的覆盖率和形状分布
4. **Beer-Lambert 光照近似**：光线穿透云层的吸收/散射，云底暗、云顶亮
5. **移动端必须降级**：降低步进次数、简化噪声层数、或用 2D 面片云做底 LOD
6. **地面云阴影**：将云覆盖率投影到地面，作为光照遮挡项

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：天空有动态体积云，云会飘动、变形，影响地面阴影，玩家可穿云
                ↑
倒推1：云是「有体积的」→ 不能用贴图，需要 Ray Marching 采样 3D 噪声
倒推2：云要有「真实形状」→ FBM = 多层 Perlin + Worley 噪声叠加
倒推3：云要「随时间变化」→ Weather Map（R=coverage, G=type, B=height）控制分布
倒推4：云要有「光照」→ 向光源二次步进计算光穿透率（Beer-Lambert 定律）
倒推5：云要「影响地面」→ 将云层投影到地面纹理，作为阴影遮罩
倒推6：云要「飘动」→ 噪声 UV 偏移 + 风向风速时间参数
倒推7：移动端要能跑 → 远距离降为 2D 面片，近距离减少步进数
```

#### 知识点拆解（倒推树）

```
实时体积云渲染
├── Ray Marching 核心
│   ├── 光线定义：起点=相机位置，方向=像素视图方向
│   ├── 云层高度范围：rayStart → rayEnd（与大气层求交）
│   ├── 步进策略：固定步长 vs 自适应步长（远处大步近处小步）
│   └── 采样累积：密度累加 → 透射率 → 最终颜色
├── 噪声系统
│   ├── 3D Perlin Noise（基础形状）
│   ├── 3D Worley Noise（细节分裂，细胞状边缘）
│   ├── FBM 分形叠加：noise = perlin * w1 + worley * w2 + ...
│   ├── 时间偏移：noise(uv + windDir * _Time * windSpeed) 实现飘动
│   └── 噪声烘焙 vs 实时计算（3D Texture 烘焙到 128³ 或 256³）
├── Weather Map（天气控制）
│   ├── R 通道：云覆盖率（0=晴空, 1=密云）
│   ├── G 通道：云类型（积云/层云/卷云混合权重）
│   ├── B 通道：云底高度偏移
│   └── 随时间演化的过渡（晴天→多云→暴雨）
├── 光照模型
│   ├── Beer-Lambert：transmittance = exp(-σ * density * stepSize)
│   ├── 向光源二次步进：从采样点向太阳方向再步进 N 次
│   ├── Henyey-Greenstein 相位函数：前向散射增强，背向散射减弱
│   └── 多次散射近似（可以用预计算查找表）
├── 地面云阴影
│   ├── 方案A：将云层 height density 投影到地面 shadow map
│   ├── 方案B：单独 RT 渲染 top-down 云覆盖率 → 作为全局光照遮挡
│   └── 方案C：基于 Weather Map 直接生成地面阴影贴图（最省）
├── 性能优化（分级策略）
│   ├── PC 高品质：64步进 + 5层FBM + 光照16步进
│   ├── 主机/中端：32步进 + 3层FBM + 光照8步进
│   ├── 移动端高画质：16步进 + 2层FBM + 简化光照
│   ├── 移动端低端：2D 面片云 + 法线贴图伪造体积感
│   └── LOD 切换：相机距离触发 Ray Marching ↔ 面片云
└── URP 实现路径
    ├── Full-screen Blit（Renderer Feature 注入全屏后处理 Pass）
    ├── 深度重建：使用 CameraDepthTexture 还原世界坐标
    ├── 天空盒融合：云与 procedural sky / HDRI sky 的混合
    └── 剔除优化：只在天空区域（深度=远）执行 ray marching
```

#### 代码实现

**体积云 Ray Marching 核心伪代码（HLSL，全屏 Pass）：**

```hlsl
// === 体积云 Ray Marching 核心 ===
// 输入：相机位置、像素世界方向、Weather Map

float4 FragCloud(Varyings input) : SV_Target
{
    float3 rayOrigin = _WorldSpaceCameraPos;
    float3 rayDir = normalize(input.worldPos - rayOrigin);
    
    // 1. 计算云层 AABB 求交（云层在 heightStart ~ heightEnd 之间）
    float2 rayStartEnd = RaySphereIntersect(rayOrigin, rayDir, 
        float3(0, _CloudLayerStart, 0), float3(0, _CloudLayerEnd, 0));
    
    // 如果没穿过云层，直接返回透明
    if (rayStartEnd.x < 0) return float4(0, 0, 0, 0);
    
    // 2. 光线步进采样
    float3 startPos = rayOrigin + rayDir * rayStartEnd.x;
    float3 endPos = rayOrigin + rayDir * rayStartEnd.y;
    float pathLength = distance(startPos, endPos);
    
    int stepCount = _StepCount; // PC:64, Mobile:16
    float stepSize = pathLength / stepCount;
    float3 stepDir = rayDir * stepSize;
    
    float transmittance = 1.0;  // 透射率
    float3 scatteredLight = 0;  // 累积散射光
    
    [loop]
    for (int i = 0; i < stepCount; i++)
    {
        float3 samplePos = startPos + stepDir * i;
        
        // 3. 采样密度（FBM 噪声 + Weather Map）
        float density = SampleCloudDensity(samplePos);
        if (density <= 0.01) continue;
        
        // 4. Beer-Lambert 光照计算
        float lightTransmittance = SampleLightTransmittance(samplePos, _SunDirection);
        float3 luminance = _SunColor * lightTransmittance + _AmbientColor;
        
        // Henyey-Greenstein 相位函数
        float phase = HenyeyGreenstein(dot(rayDir, _SunDirection), _Anisotropy);
        
        // 5. 累积
        float sigma = density * _Absorption;
        float stepTransmittance = exp(-sigma * stepSize);
        scatteredLight += luminance * phase * transmittance * (1.0 - stepTransmittance) / sigma;
        transmittance *= stepTransmittance;
        
        // 提前退出：几乎不透明了就不用继续步进
        if (transmittance < 0.01) break;
    }
    
    return float4(scatteredLight, 1.0 - transmittance);
}

// === 云密度采样 ===
float SampleCloudDensity(float3 pos)
{
    // 高度衰减：云层中间最密，上下边缘稀疏
    float heightGradient = ComputeHeightGradient(pos.y);
    
    // Weather Map 控制覆盖率
    float2 weatherUV = pos.xz * _WeatherScale + _WindOffset * _Time.yy;
    float weather = SAMPLE_TEXTURE2D(_WeatherMap, sampler_WeatherMap, weatherUV).r;
    
    // 3D 噪声 FBM
    float3 noiseUV = pos * _NoiseScale + _WindOffset * _Time.y * _WindSpeed;
    float noise = FBM3D(noiseUV, _OctaveCount); // PC:5, Mobile:2
    
    // 组合：密度 = 高度 × 天气覆盖率 × 噪声形状
    float density = saturate(heightGradient * weather - noise * _DetailModulation);
    
    return max(density, 0);
}

// === 向光源步进（简化版） ===
float SampleLightTransmittance(float3 pos, float3 lightDir)
{
    float transmittance = 1.0;
    int lightSteps = _LightStepCount; // PC:8, Mobile:2
    
    [loop]
    for (int j = 0; j < lightSteps; j++)
    {
        pos += lightDir * _LightStepSize;
        float d = SampleCloudDensity(pos);
        transmittance *= exp(-d * _Absorption * _LightStepSize);
    }
    return transmittance;
}

// === Henyey-Greenstein 相位函数 ===
float HenyeyGreenstein(float cosTheta, float g)
{
    float g2 = g * g;
    return (1 - g2) / pow(1 + g2 - 2 * g * cosTheta, 1.5) * 0.25;
}
```

### ⚡ 实战经验

- **3D 噪声纹理烘焙是关键**：实时计算 3D Perlin 在移动端直接爆炸。预烘焙到 128³ 的 3D Texture（RGBA：不同频率的噪声分层），运行时只做采样和 FBM 叠加
- **自适应步进**：远处用大步长（8m/步），近处用小步长（0.5m/步），总步数不变但品质提升。用 `stepSize = lerp(nearStep, farStep, distanceRatio)`
- **half resolution rendering**：体积云渲染在半分辨率 RT 上，然后上采样 + 双边滤波去块，性能省一半
- **云阴影的省钱方案**：不用二次 ray marching，直接把 Weather Map 当 shadow map 用——正交投影到地面，`cloudShadow = weatherMap.uv * cloudDensity * step(pos.y / cloudHeight)`
- **米哈游的做法**：原神的体积云是预烘焙 + 高度雾混合 + 天气状态插值，移动端几乎不做实时 ray marching
- ** temporal reprojection**：上一帧云结果做 temporal accumulation，本帧只补充新像素，4 步就能达到 16 步的品质（但运动时有拖影）
- **玩家穿云的体验**：当相机进入云层内部，密度函数需要平滑过渡（云内 → 透明 → 云外），否则会有突变

### 🎯 能力体检清单

- [ ] 能否解释 Ray Marching 的基本原理？为什么不能用光栅化做体积云？（光栅化 vs 光线追踪的本质区别）
- [ ] Beer-Lambert 定律的物理含义是什么？在云渲染中怎么用？（光学厚度与透射率的关系）
- [ ] Henyey-Greenstein 相位函数的 `g` 参数从前向散射（+0.8）变为后向散射（-0.5），视觉效果有何不同？
- [ ] 3D 噪声纹理 128³ 和 256³ 在显存和品质上的权衡？（VRAM 预算）
- [ ] 移动端做体积云最大的瓶颈是什么？GPU 算力 vs 带宽 vs 纹理采样？（ALU bound vs memory bound 分析）
- [ ] 如果云在天际线处有明显的 banding（条带），是什么原因？怎么修复？（步进次数不足 → dithering/蓝色噪声抖动起始位置）
- [ ] 如何让云随游戏内时间（早→午→晚）自然变化颜色？（方向光颜色 × 密度 × 大气散射的耦合）
- [ ] Weather Map 的三通道分别控制什么？如果要做「晴天转暴雨」的动画，你会怎么做？（材质参数插值 vs Weather Map 混合）

### 🔗 相关问题

- [URP 体积雾](rendering/urp-volumetric-fog) — 体积雾与体积云的技术栈高度重合
- [移动端 GPU 过热分析](optimization/mobile-overheating-gpu-analysis) — 体积云是移动端发热大户
- [Forward+ Cluster](rendering/forward-plus-cluster) — 体积云的光照步进与 cluster 的交互
