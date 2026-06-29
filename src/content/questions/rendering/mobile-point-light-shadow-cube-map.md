---
title: "移动端点光源实时阴影：CubeMap ShadowMap 的性能与质量怎么平衡？"
category: "rendering"
level: 3
tags: ["ShadowMap", "CubeMap", "点光源", "实时阴影", "移动端", "URP"]
hint: "点光源阴影 = 6 面 CubeMap 渲染开销巨大，移动端必须用 Dual-Paraboloid 或 1-face 近似来砍成本"
related: ["rendering/shadow-acne-peter-panning-fix-urp", "rendering/deferred-multi-light", "optimization/mobile-gpu-occupancy-bottleneck"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款移动端 ARPG，关卡中有个篝火（点光源），主策要求篝火必须投射实时阴影——角色走近时影子方向要正确。但手机上 6 面 CubeMap ShadowMap 开销太大。你怎么解决？」

### ✅ 核心要点

1. **问题本质**：点光源向全方向发光，阴影需要 6 面 CubeMap（=6 次 Draw Pass），移动端不可接受
2. **降维方案**：Dual-Paraboloid Shadow Map（DPSM）——2 面代替 6 面，开销降 2/3
3. **进一步优化**：单面近似（只渲染最关键方向）、烘焙 + 实时混合
4. **质量补偿**：DPSM 在边缘有畸变，需要采样修正和 bias 调整
5. **URP 落地**：需自定义 Renderer Feature 或 Pass，URP 原生不支持点光源实时阴影

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色走近篝火 → 影子方向正确 → 性能不能崩
                ↑
倒推1：需要点光源 ShadowMap → CubeMap 是标准方案，但 6 面 = 6 次 Pass
倒推2：6 面太多 → 能不能用更少面数？→ Dual-Paraboloid（2 面）
倒推3：DPSM 畸变问题 → 需要抛物面投影 + 采样修正
倒推4：2 面还是贵 → 是否所有光源都需要？→ 近距离才开启（距离剔除）
倒推5：移动端 bandwidth → ShadowMap 分辨率降到 256 甚至 128
倒推6：URP 不原生支持 → 自定义 ScriptableRendererFeature
```

#### 知识点拆解（倒推树）

```
移动端点光源实时阴影
├── CubeMap ShadowMap（标准方案）
│   ├── 6 面 Render Pass（Geometry → +X/-X/+Y/-Y/+Z/-Z）
│   ├── GeometryShader 单 Pass 技术（移动端不支持）
│   ├── 开销分析：6 × 场景 Draw Call，移动端直接爆炸
│   └── 适用：PC/Console，移动端不推荐
├── Dual-Paraboloid Shadow Map（DPSM）
│   ├── 原理：两个半球面投影 → 前后各 1 张 RT（2 面）
│   ├── 抛物面坐标变换：float2 uv = pos.xy / (1 + pos.z)
│   ├── 畸变问题：边缘拉伸大 → 需要细分足够高的几何体
│   ├── Bias 调整：边缘斜率大，需要 ReceiverPlaneDepthBias
│   └── 开销：2 × 场景 Draw Call，是 CubeMap 的 1/3
├── 进一步优化策略
│   ├── 距离剔除：只有玩家/关键角色在 N 米内才开启
│   ├── 单面近似：只渲染最关键方向（如朝上+水平）
│   ├── ShadowMap 复用：静态光源 → 烘焙 CubeMap，动态对象叠加
│   ├── 分辨率控制：128~256 CubeMap on Mobile
│   └── 光源数量限制：同时只允许 1 个实时阴影点光源
├── URP 实现路径
│   ├── 自定义 ScriptableRendererFeature
│   ├── 多 Pass 编排：先渲染 ShadowMap，再在 Lighting Pass 中采样
│   ├── Shader 端：WorldToParaboloid 矩阵变换
│   └── ShaderKeyword 控制是否启用
└── 替代方案（何时不用实时阴影）
    ├── 烘焙 GI + 实时角色 Contact Shadow
    ├── Blob Shadow（圆形暗斑 + 方向偏移模拟）
    └── 前向渲染下点光源只做照明、不做阴影（常见移动端方案）
```

#### 代码实现

**Dual-Paraboloid 坐标变换（HLSL）：**

```hlsl
// World → Paraboloid UV（front/back）
float2 WorldToParaboloidUV(float3 worldPos, float3 lightPos, float lightRange, bool isFront)
{
    float3 dir = worldPos - lightPos;
    float3 L = normalize(dir);
    
    // 范围裁剪
    float dist = length(dir);
    if (dist > lightRange) discard;
    
    // front hemisphere: L.z > 0, back: L.z < 0
    if (isFront && L.z < 0) discard;
    if (!isFront && L.z > 0) discard;
    
    // 抛物面投影
    float2 uv;
    uv.x = L.x / (1.0 + abs(L.z));
    uv.y = L.y / (1.0 + abs(L.z));
    
    // remap to [0, 1]
    uv = uv * 0.5 + 0.5;
    return uv;
}

// 采样 DPSM 阴影
float SampleDPSMShadow(Texture2D shadowMap, SamplerState clampSampler,
                       float3 worldPos, float3 lightPos, float lightRange,
                       float3 normal, float bias)
{
    float3 dir = worldPos - lightPos;
    float3 L = normalize(dir);
    float dist = length(dir) / lightRange; // normalized [0,1]
    
    // 选择 front 或 back 面
    bool useFront = L.z >= 0;
    float2 uv = WorldToParaboloidUV(worldPos, lightPos, lightRange, useFront);
    
    // 采样深度
    float storedDepth = shadowMap.Sample(clampSampler, uv).r;
    
    // 抛物面空间下的深度：distance from light along paraboloid
    float paraboloidDepth = dist;
    
    // 边缘 bias 补偿（边缘斜率大，需要更大 bias）
    float edgeFalloff = 1.0 - saturate(length(uv - 0.5) * 2.0);
    float dynamicBias = bias / max(edgeFalloff, 0.1);
    
    // PCF 3x3 软阴影
    float shadow = 0;
    float2 texelSize = float2(1.0 / 256.0, 1.0 / 256.0);
    [unroll] for (int x = -1; x <= 1; x++)
    [unroll] for (int y = -1; y <= 1; y++)
    {
        float2 offsetUV = uv + float2(x, y) * texelSize;
        float s = shadowMap.Sample(clampSampler, offsetUV).r;
        shadow += (paraboloidDepth - dynamicBias > s) ? 0.0 : 1.0;
    }
    shadow /= 9.0;
    
    return shadow;
}
```

**URP 自定义点光源阴影 Renderer Feature（C# 框架）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class PointLightShadowFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public LayerMask shadowCasterMask = ~0;
        public int shadowMapSize = 256;
        public float shadowDistance = 15f;
        public RenderPassEvent passEvent = RenderPassEvent.BeforeRenderingShadows;
    }
    
    public Settings settings = new Settings();
    private PointLightShadowPass _pass;

    public override void Create()
    {
        _pass = new PointLightShadowPass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        // 查找场景中需要投射阴影的点光源
        var pointLights = FindActivePointLights(renderingData);
        if (pointLights.Count == 0) return;
        
        _pass.Setup(pointLights);
        renderer.EnqueuePass(_pass);
    }
    
    // ... FindActivePointLights 实现
}
```

**方案对比表：**

| 方案 | Pass 数 | ShadowMap 分辨率 | 内存开销 | 质量 | 适用场景 |
|------|---------|------------------|----------|------|----------|
| CubeMap 6 面 | 6 | 512³ | 高 | 最好 | PC/主机 |
| Dual-Paraboloid | 2 | 256×256×2 | 中 | 较好（边缘畸变） | 移动端首选 |
| 单面近似 | 1 | 256×256 | 低 | 一般 | 性能极度紧张 |
| 烘焙 + Contact Shadow | 0（运行时） | 烘焙纹理 | 极低 | 好但静态 | 静态光源场景 |

### ⚡ 实战经验

- **项目实战选择**：大多数移动端 ARPG 用「1 个实时阴影点光源 + 距离剔除（8m 内开启）」的方案，DPSM 是最佳平衡点
- **DPSM 畸变陷阱**：低面数模型在 DPSM 边缘会出现阴影撕裂，需要在 ShadowCaster Pass 中强制提高 tessellation 或使用 pre-tessellated mesh
- **ShadowMap 复用**：篝火等固定光源可以预渲染 ShadowMap 到纹理，运行时只更新动态物体的阴影贡献
- **性能数据参考**：Snapdragon 888 上，256 分辨率 DPSM（2 面）渲染一个中等场景（~200 draw call）约耗时 0.8ms，CubeMap 6 面约 2.4ms——差距 3 倍
- **URP 限制提醒**：URP 2022 LTS 原生不支持点光源实时阴影，需要完全自定义 Renderer Feature；Unity 6 SRP 开始有实验性支持

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 CubeMap ShadowMap 的 6 面怎么来的 | CubeMap Shadow 原理 | 学习 cubic shadow mapping 流程 |
| DPSM 抛物面投影写不出 | 抛物面坐标变换数学 | 复习立体几何 + 抛物面参数化 |
| 阴影边缘有 acne/龟裂 | DPSM bias 调整 | 学习 ReceiverPlaneDepthBias 和 slope-scale bias |
| URP 中不知道在哪注入阴影 Pass | URP 渲染顺序 | 学 ScriptableRendererFeature 生命周期 |
| 做出来但性能还是差 | ShadowMap 分辨率 / Draw Call | 学 GPU Performance Profiler 分析 |

### 🔗 相关问题

- 方向光阴影用 CSM（Cascade Shadow Map）时，级联分辨率怎么分配？
- 半透明物体（玻璃、水面）的阴影怎么处理？（提示：大部分引擎忽略半透明阴影）
- 多个点光源同时投射阴影，如何做光源优先级排序？
