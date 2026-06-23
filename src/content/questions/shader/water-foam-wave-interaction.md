---
title: "水面泡沫与波浪交互：如何让水流过礁石时溅起真实的浪花？"
category: "shader"
level: 3
tags: ["水面渲染", "泡沫", "Gerstner波", "Flow Map", "交互", "URP"]
hint: "波浪用 Gerstner 叠加 + 泡沫用深度差/流速检测 + 流动方向用 Flow Map——三者联动才有真实感"
related: ["shader/water-caustics", "rendering/urp-volumetric-fog", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

面试官给你看一段参考视频——海浪拍打礁石，水流撞到障碍物后溅起白色泡沫，泡沫随水流方向消散。然后说：

> "我们要做一个海岛场景，水面需要能跟场景中的物体交互——流过礁石时有泡沫，流速变化的地方有浪花。你是 TA，给我一套完整的水面 Shader 方案。URP 管线，移动端也要能跑。"

这是米哈游、叠纸、鹰角等做开放世界或海战游戏时的经典 TA 面试题。考察的是**自然现象观察能力 + Shader 系统设计 + 多技术组合能力**。

### ✅ 核心要点

1. **水面 = 几何波动 + 表面着色 + 交互反馈**：三层缺一不可，纯静态水面一看就假
2. **Gerstner 波叠加是位移标准方案**：多个不同频率/振幅/方向的 Gerstner 波叠加，比正弦波更真实（波峰尖锐、波谷平缓）
3. **泡沫生成有两个驱动源**：深度差（浅水区/障碍物周围）和流速差（Flow Map 突变区域）
4. **Flow Map 控制流向**：一张 RG 通道编码流向向量的纹理，驱动 UV 偏移和泡沫分布
5. **移动端降级策略**：Gerstner 波在顶点着色器做（控制顶点数），泡沫用简化噪声 + 深度检测

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：水面有波浪起伏 → 流过礁石溅起泡沫 → 泡沫沿水流方向消散 → 深浅水颜色不同
     ↓ 倒推
波浪起伏 = 顶点位移
  ├── 方案：多 Gerstner 波叠加（3-4层不同参数）
  └── 移动端：1-2层 Gerstner + 法线贴图伪造细节
     ↓ 倒推
泡沫产生 = 哪里该有泡沫？
  ├── 来源1：深度差 → 浅水区/障碍物周围（相机深度 vs 场景深度）
  ├── 来源2：流速变化 → Flow Map 梯度大的地方
  └── 来源3：波峰 → Gerstner 波 Y 值超过阈值的区域
     ↓ 倒推
泡沫消散 = 泡沫要有生命周期感
  ├── Flow Map 驱动 UV 滚动 → 泡沫沿流向移动
  └── 噪声纹理 mask → 边缘随机消散，不是硬边
     ↓ 倒推
深浅水颜色 = 水的吸收与散射
  ├── 深度差 → 用场景深度减去水面深度得到"水的厚度"
  ├── 浅水 → 偏青色/浅蓝（散射为主）
  └── 深水 → 偏深蓝/黑（吸收为主）
     ↓ 倒推
URP 集成 = Shader Graph 或 手写 HLSL
  ├── 关键：获取场景深度纹理（_CameraDepthTexture）
  └── 关键：半透明排序 + 折射采样
```

#### 知识点拆解（倒推树）

```
水面泡沫与波浪交互
├── 波浪位移系统
│   ├── Gerstner 波公式：P(x,t) = (x + Q·A·D·cos(w·D·x + φ·t), A·sin(...))
│   │   参数：振幅A、频率w、方向D、速度φ、陡度Q
│   ├── 多波叠加策略（2-4层，不同频率/方向/振幅）
│   ├── LOD 策略：远处减少波层数，近处全精度
│   ├── 法线计算：从位移偏导数推导（或单独法线贴图）
│   └── 顶点着色器 vs 片段的性能取舍
├── 泡沫系统
│   ├── 深度差检测（相机深度纹理 - 水面深度 → 水厚度）
│   │   └── 水厚度 < 阈值 → 生成泡沫 mask
│   ├── 流速梯度检测（Flow Map 的偏导数 → 流速突变区域）
│   ├── 波峰检测（Gerstner 波 Y 位移 > 阈值）
│   ├── 泡沫纹理混合（2-3 层噪声纹理不同频率叠加）
│   └── 泡沫生命周期（淡入淡出：用时间噪声控制 alpha）
├── Flow Map 系统
│   ├── Flow Map 编码：RG 通道 = [-1,1] 方向向量
│   ├── UV 偏移：uv += flowDir * flowSpeed * time
│   ├── 时间相位混合（避免 UV 偏移跳变）：两个相位 0-1 循环交叉淡入
│   └── Flow Map 生成方式：Houdini 烘焙 / 美术手绘 / 程序化
├── 水体光学
│   ├── 深度渐变（Beer-Lambert 吸收）
│   ├── 折射（采样场景颜色 + UV 扰动）
│   ├── 反射（Planar Reflection / SSR / Cubemap）
│   ├── 菲涅尔效应（视角越平反射越强）
│   └── 焦散（Caustics — 见关联问题）
└── URP 工程实现
    ├── Shader Graph vs HLSL（Graph 适合快速迭代，HLSL 适合性能控制）
    ├── 深度纹理开启（URP Asset → Depth Texture）
    ├── 半透明渲染队列设置
    └── 多平台 Quality 分支
```

#### 代码实现

**完整水面 Shader 核心（HLSL for URP）**

```hlsl
// WaterSurface.shader — 核心结构
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

// --- Gerstner 波 ---
struct GerstnerWave {
    float3 direction;  // 波传播方向（XZ平面）
    float  amplitude;  // 振幅
    float  frequency;  // 频率
    float  speed;      // 速度
    float  steepness;  // 陡度 [0,1]
};

// 单个 Gerstner 波的位移贡献
float3 GerstnerPosition(float2 pos, GerstnerWave wave, float time)
{
    float2 d = normalize(wave.direction.xz);
    float phase = wave.frequency * dot(d, pos) + wave.speed * time;
    float c = cos(phase);
    float s = sin(phase);

    float3 displacement;
    displacement.x = wave.steepness * wave.amplitude * d.x * c;
    displacement.z = wave.steepness * wave.amplitude * d.y * c;
    displacement.y = wave.amplitude * s;

    return displacement;
}

// 多波叠加（VivLod 控制远处减少计算）
float3 GetWaveDisplacement(float2 pos, float time, int waveCount)
{
    float3 total = 0;
    // 4层波，参数错开
    GerstnerWave waves[4] = {
        {float3(1,0,0.3),  0.5, 0.8, 0.8, 0.6},
        {float3(0.6,0,1),  0.3, 1.2, 1.0, 0.4},
        {float3(-0.8,0,1), 0.2, 1.8, 1.3, 0.3},
        {float3(1,0,-0.5), 0.15,2.5, 1.6, 0.2}
    };

    [unroll]
    for (int i = 0; i < waveCount; i++)
    {
        total += GerstnerPosition(pos, waves[i], time);
    }
    return total;
}

// --- 顶点着色器 ---
struct Attributes { float4 positionOS : POSITION; float2 uv : TEXCOORD0; };
struct Varyings {
    float4 positionCS : SV_POSITION;
    float2 uv : TEXCOORD0;
    float3 worldPos : TEXCOORD1;
    float4 screenPos : TEXCOORD2;
};

Varyings Vert(Attributes input)
{
    Varyings output;
    float3 posOS = input.positionOS.xyz;

    // Gerstner 顶点位移（仅近处做4层，远处降为2层）
    float dist = length(TransformObjectToWorld(posOS).xz - _WorldSpaceCameraPos.xz);
    int waveCount = dist < 50 ? 4 : 2;
    float3 displacement = GetWaveDisplacement(posOS.xz, _Time.y, waveCount);
    posOS += displacement;

    output.positionCS = TransformObjectToHClip(posOS);
    output.worldPos = TransformObjectToWorld(posOS);
    output.screenPos = ComputeScreenPos(output.positionCS);
    output.uv = input.uv;
    return output;
}

// --- 片段着色器 ---
TEXTURE2D(_FlowMap);        SAMPLER(sampler_FlowMap);
TEXTURE2D(_FoamNoise);      SAMPLER(sampler_FoamNoise);
TEXTURE2D(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);

float _FoamAmount;
float _FoamDepthThreshold;
float _WaterDepth;
float3 _ShallowColor;
float3 _DeepColor;
float _FlowSpeed;

half4 Frag(Varyings input) : SV_Target
{
    float3 viewDirWS = normalize(_WorldSpaceCameraPos - input.worldPos);

    // === 1. 深度差计算（水的厚度）===
    float rawDepth = SampleSceneDepth(input.screenPos.xy / input.screenPos.w);
    float sceneDepth = LinearEyeDepth(rawDepth, _ZBufferParams);
    float waterDepth = LinearEyeDepth(input.positionCS.z, _ZBufferParams);
    float depthDiff = sceneDepth - waterDepth; // 水的"厚度"

    // === 2. Flow Map 采样 ===
    float2 flowDir = SAMPLE_TEXTURE2D(_FlowMap, sampler_FlowMap, input.uv).rg * 2 - 1;
    float flowMag = length(flowDir);
    float2 flowOffset = flowDir * _FlowSpeed * _Time.y;

    // 双相位混合（避免 UV 滚动跳变）
    float phase1 = frac(_Time.y * 0.1);
    float phase2 = frac(_Time.y * 0.1 + 0.5);
    float flowBlend = abs(phase1 * 2 - 1); // 0→1→0 三角波

    // === 3. 深度渐变颜色 ===
    float depthFactor = saturate(depthDiff / _WaterDepth);
    float3 waterColor = lerp(_ShallowColor, _DeepColor, depthFactor);

    // === 4. 泡沫生成 ===
    // 泡沫来源1：浅水区深度差
    float foamByDepth = smoothstep(_FoamDepthThreshold, 0, depthDiff);

    // 泡沫来源2：流速突变（Flow Map 梯度）
    float2 flowGradient = float2(ddx(flowMag), ddy(flowMag));
    float foamByFlow = saturate(length(flowGradient) * 10);

    // 泡沫来源3：Gerstner 波峰
    float waveHeight = GetWaveDisplacement(input.worldPos.xz, _Time.y, 2).y;
    float foamByWave = smoothstep(0.3, 0.5, waveHeight);

    // 合成泡沫 mask
    float foamMask = saturate((foamByDepth + foamByFlow * 0.5 + foamByWave * 0.3) * _FoamAmount);

    // 泡沫纹理（双相位混合）
    float foam1 = SAMPLE_TEXTURE2D(_FoamNoise, sampler_FoamNoise, input.uv * 4 + flowOffset * phase1).r;
    float foam2 = SAMPLE_TEXTURE2D(_FoamNoise, sampler_FoamNoise, input.uv * 4 + flowOffset * phase2).r;
    float foamTex = lerp(foam1, foam2, flowBlend);

    float3 foamColor = float3(1, 1, 1) * foamTex;
    float3 finalColor = lerp(waterColor, foamColor, foamMask * foamTex);

    // === 5. 菲涅尔反射 ===
    float3 normalWS = normalize(float3(0, 1, 0)); // 简化：实际应从 Gerstner 偏导计算
    float fresnel = pow(1.0 - saturate(dot(viewDirWS, normalWS)), 5);
    float3 skyColor = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture,
        input.screenPos.xy / input.screenPos.w + normalWS.xz * 0.02).rgb;
    finalColor = lerp(finalColor, skyColor, fresnel * 0.6);

    return half4(finalColor, 0.9);
}
```

### ⚡ 实战经验

1. **Gerstner 波的波数选择**：移动端 1-2 层够了，配合法线贴图伪造高频细节；PC/主机 3-4 层 + FFT 频谱
2. **顶点密度是硬约束**：Gerstner 位移需要足够的网格密度，水面 mesh 通常用 32×32 或 64×64 tessellation，否则波浪锯齿严重
3. **Flow Map 双相位混合是行业标准**：单相位 UV 滚动每隔一段时间会"跳"一下，双相位交叉淡入消除跳变
4. **泡沫不要只靠一个源**：只有深度差 → 看起来像海拔图；只有流速 → 看起来像扫描线；三者叠加才自然
5. **折射 UV 扰动幅度**：移动端用 0.01-0.02 的小幅度扰动，太大会导致半透明排序错误
6. **水面 Tessellation 性能**：移动端不要用硬件 Tessellation，改用 Distance-based mesh subdivision（近处 mesh 细，远处 mesh 粗）
7. **调试利器**：单独输出 foamMask 到颜色通道，确认泡沫分布是否合理

### 🎯 能力体检清单

- [ ] 能否手推 Gerstner 波公式？它和正弦波的本质区别是什么？
- [ ] Flow Map 的双相位混合原理是什么？如果不用会出什么问题？
- [ ] 深度差计算时，半透明物体（水面本身）的深度值如何处理？URP 中 `_CameraDepthTexture` 包含半透明物体吗？
- [ ] 移动端水面通常只需要 2 层 Gerstner 波，但远处波浪看起来太"假"，如何用法线贴图补充细节？
- [ ] 菲涅尔效应用的 `pow(1-NdotV, 5)` 和 Schlick 近似有什么区别？水面用哪个更合适？
- [ ] 如果场景中有船在行驶，如何做出船尾尾迹的泡沫？（提示：RenderTexture + 拖尾粒子）

### 🔗 相关问题

- [水面焦散效果](../shader/water-caustics.md)
- [URP 体积雾实现](../rendering/urp-volumetric-fog.md)
- [URP 自定义后处理 Pass](../rendering/custom-post-processing-urp.md)
