---
title: "混合光线追踪软阴影：UE5 里 Lumen 的阴影怎么和传统 ShadowMap 混合？"
category: "rendering"
level: 4
tags: ["RayTracing", "ShadowMap", "混合管线", "UE5", "Lumen", "软阴影", "HybridRendering"]
hint: "核心是 Screen Space Ray Tracing + 远距离 Ray March + ShadowMap 回退——不是纯 RT，而是三管线融合"
related: ["rendering/ray-tracing-hybrid-pipeline", "rendering/hiz-screen-space-reflection", "rendering/temporal-anti-aliasing-taa"]
---

## 参考答案

### 🎬 场景描述

面试官打开一段 UE5 项目演示，角色站在一棵大树的复杂枝叶阴影下，阴影边缘有非常自然的软化和半影过渡。然后说：

> "我们现在用的是 Cascade ShadowMap + Distance Field Ambient Occlusion，但远景阴影品质太差、半影效果不自然。Lumen 的 RT 阴影在高端 PC 上效果很好但移动端跑不动。你是 TA，给我一套混合方案——在 PC 上用 RT 软阴影，移动端退化到 CSSM + 屏幕空间方案，中间不要有明显的品质断层。"

这是腾讯/米哈游等做高品质项目时的 **高级 TA 面试题**，考察的是对全局光照系统、混合渲染管线的理解，以及跨平台品质分级的架构能力。

### ✅ 核心要点

1. **混合管线思维**：不是 RT 替代 ShadowMap，而是 ShadowMap（近）→ Screen Space Ray Traced（中）→ RT/DFAO（远）的分层融合
2. **半影的物理正确性**：PCSS（Percentage-Closer Soft Shadows）根据遮挡距离动态调整滤波核大小
3. **Screen Space Ray Tracing（SSRT）**：在 Hi-Z 层级上做屏幕空间光线步进，无需硬件 RT 支持
4. **UE5 Lumen 阴影机制**：Surface Cache + Mesh Distance Field + RT（可选），理解每层做什么
5. **品质分级策略**：RT Tier（PC）→ SSRT Tier（主机/高端移动）→ CSSM + VSM（低端移动），无缝降级

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色站在复杂枝叶下 → 近处阴影锐利 → 远处阴影柔和半影 → 过渡自然无断层
                    ↓
倒推1：阴影品质取决于「半影的物理正确性」
  ├── 传统 CSSM：固定滤波核 → 半影均匀但假
  ├── PCSS：遮挡距离 → 动态核大小 → 物理正确半影
  └── RT Shadow：光线追踪 → 天然软阴影（面光源）
                    ↓
倒推2：近处用高精度方案（0~20m）
  ├── RT Shadow：TraceDistanceField 或硬件 RT（面光源阴影）
  ├── 回退：PCSS 优化版 Cascade ShadowMap
  └── 混合权重：Distance-based blend
                    ↓
倒推3：中距离用屏幕空间方案（20~80m）
  ├── SSRT（Screen Space Ray Tracing）：在 Hi-Z 上步进
  ├── 优势：无需额外 Draw Call，利用已有深度
  └── 局限：屏幕外信息丢失 → 需要距离场补偿
                    ↓
倒推4：远距离用距离场/全局方案（80m+）
  ├── DFAO（Distance Field Ambient Occlusion）：方向性遮蔽
  ├── Lumen Surface Cache：场景表面缓存，近似全局阴影
  └── VLM（Virtual Shadow Map）：超高分辨率虚拟阴影图
                    ↓
倒推5：跨平台品质分级
  ├── PC（RTX 显卡）：硬件 RT 阴影 → 最高品质
  ├── PC（非 RTX）/ 主机：SSRT + DFAO 混合
  ├── 移动端高端：CSSM + PCSS 滤波 + VSM
  └── 移动端低端：CSSM + Poisson Disk 滤波
```

#### 知识点拆解（倒推树）

```
混合软阴影系统
├── ShadowMap 基础
│   ├── Cascade Shadow Map（CSM）原理与裂级
│   ├── 阴影痤疮（Shadow Acne）与偏移（Bias）
│   ├── PCF（Percentage-Closer Filtering）
│   └── PCSS（Percentage-Closer Soft Shadows）
│       ├── Blocker Search：搜索最近遮挡体
│       ├── Penumbra Estimation：根据遮挡距离计算半影宽度
│       └── Adaptive Filter Kernel：动态滤波核
├── Virtual Shadow Map（VSM / LSM）
│   ├── Virtual Texture 思想：只渲染可见区域的阴影图
│   ├── Page-based allocation：按屏幕像素需求分配 Mipmap Page
│   ├── UE5 Virtual Shadow Maps：级联 + 分页
│   └── 性能：4K 阴影分辨率但只渲染可见的 Page
├── Screen Space Ray Tracing（SSRT）
│   ├── Hi-Z Tracing：在深度金字塔上做光线步进
│   ├── 优势：零额外几何体渲染，复用 Depth Pre-Pass
│   ├── 局限：屏幕外遮挡信息丢失、薄几何体穿透
│   └── 修复方案：DFAO / RT 补偿屏幕外遮挡
├── Distance Field Shadow
│   ├── Mesh Distance Field（MDF）预计算
│   ├── Cone Tracing：从表面沿光线方向追踪距离场锥
│   ├── DFAO（Directional DFAO）：方向性环境光遮蔽
│   └── 性能与内存：MDF 数据量巨大，需要流式加载
├── 硬件 Ray Tracing Shadow
│   ├── RT Shadow Pipeline：BLAS/TLAS → TraceRay
│   ├── 面光源阴影：多光线采样平均（重要性采样）
│   ├── Denoiser：SVGF / RL 上时域降噪
│   └── 半分辨率 RT + 上采样（TAAU/TSR）
├── UE5 Lumen 阴影机制
│   ├── Surface Cache：场景表面辐照度缓存
│   ├── Mesh Distance Field Tracing：软 RT 替代方案
│   ├── Final Gather：多bounce 间接光含方向性阴影
│   └── Lumen Quality Settings：High/Medium/Low 分级
├── 混合权重与无缝过渡
│   ├── Distance-based blend：距离决定各层贡献权重
│   ├── Dither transition：用 dither pattern 消除硬边
│   ├── Temporal accumulation：多帧累积降噪
│   └── 防止「阴影闪烁」：时域稳定性优先
└── 跨平台品质分级
    ├── Device Profile：GPU 等级 → Shadow Quality Tier
    ├── Fallback chain：RT → SSRT → VSM → CSSM
    └── 性能预算：阴影渲染预算 2-3ms（60fps 标准）
```

#### 代码实现

**1. PCSS 软阴影 Shader（HLSL，URP 兼容）**

```hlsl
// PCSS (Percentage-Closer Soft Shadows)
// 输入：阴影图、UV、深度、Bias
// 输出：0~1 的阴影系数（1=无阴影，0=全阴影）

float PCSS(float2 shadowUV, float depth, float2 texelSize)
{
    // === Step 1: Blocker Search ===
    // 在一定范围内搜索最近遮挡体的平均深度
    float blockerSum = 0.0;
    float blockerCount = 0.0;
    float searchRadius = 25.0 * texelSize.x; // 搜索半径

    [unroll]
    for (int x = -3; x <= 3; x++)
    {
        [unroll]
        for (int y = -3; y <= 3; y++)
        {
            float2 offset = float2(x, y) * searchRadius / 3.0;
            float shadowDepth = SAMPLE_TEXTURE2D(_ShadowMap, sampler_ShadowMap,
                                                   shadowUV + offset).r;
            if (shadowDepth < depth - 0.001)
            {
                blockerSum += shadowDepth;
                blockerCount += 1.0;
            }
        }
    }

    // 无遮挡体 → 无阴影
    if (blockerCount < 1.0) return 1.0;

    float avgBlockerDepth = blockerSum / blockerCount;

    // === Step 2: Penumbra Estimation ===
    // 半影宽度 ∝ (receiverDepth - blockerDepth) / blockerDepth
    float penumbraWidth = ((depth - avgBlockerDepth) / avgBlockerDepth)
                          * searchRadius * 5.0;
    penumbraWidth = clamp(penumbraWidth, 1.0, 25.0);

    // === Step 3: PCF with Adaptive Kernel ===
    float shadow = 0.0;
    float kernelRadius = penumbraWidth * texelSize.x;

    [unroll]
    for (int x2 = -3; x2 <= 3; x2++)
    {
        [unroll]
        for (int y2 = -3; y2 <= 3; y2++)
        {
            float2 offset = float2(x2, y2) * kernelRadius / 3.0;
            float shadowDepth = SAMPLE_TEXTURE2D(_ShadowMap, sampler_ShadowMap,
                                                   shadowUV + offset).r;
            shadow += (shadowDepth < depth - 0.001) ? 0.0 : 1.0;
        }
    }

    return shadow / 49.0;
}
```

**2. Screen Space Ray Traced Shadow（Hi-Z）**

```hlsl
// 屏幕空间光线追踪阴影
// 原理：从像素位置出发，向光源方向步进，
//       如果击中几何体（深度被挡），则该像素在阴影中

float SSTracedShadow(float3 worldPos, float3 lightDir,
                     float2 screenUV, float depth)
{
    // 射线起点和方向（屏幕空间）
    float3 rayOriginWS = worldPos;
    float3 rayDirWS = normalize(-lightDir);

    // 步进参数
    const int MAX_STEPS = 32;
    const float stepSize = 0.5;     // 初始步长（世界单位）
    const float thickness = 0.3;     // 厚度阈值（防止穿透薄几何）

    float3 currentPos = rayOriginWS + rayDirWS * stepSize;

    [loop]
    for (int i = 0; i < MAX_STEPS; i++)
    {
        // 投影到屏幕空间
        float4 clipPos = mul(_ViewProjMatrix, float4(currentPos, 1.0));
        float2 sampleUV = clipPos.xy / clipPos.w * 0.5 + 0.5;

        // UV 超出屏幕 → 射线离开屏幕空间
        if (any(sampleUV < 0) || any(sampleUV > 1))
            return 1.0; // 无屏幕空间遮挡信息，默认无阴影

        // 采样场景深度
        float sceneDepth = SAMPLE_TEXTURE2D(_CameraDepthTexture,
                                             sampler_CameraDepthTexture, sampleUV).r;
        float sceneDepthLinear = LinearEyeDepth(sceneDepth, _ZBufferParams);
        float rayDepthLinear = LinearEyeDepth(clipPos.z / clipPos.w, _ZBufferParams);

        // 深度差
        float depthDiff = sceneDepthLinear - rayDepthLinear;

        // 击中检测：射线深度 ≈ 场景深度（在厚度范围内）
        if (depthDiff > 0 && depthDiff < thickness)
        {
            // 被遮挡 → 在阴影中
            // 根据步数做距离衰减（远处阴影更柔和）
            float fade = 1.0 - float(i) / float(MAX_STEPS);
            return smoothstep(0.0, 0.5, fade) * 0.0; // 0 = 全阴影
        }

        // 步进（可以换 Hi-Z 加速）
        currentPos += rayDirWS * stepSize;
    }

    return 1.0; // 未击中任何遮挡体 → 无阴影
}
```

**3. 混合权重函数（C#）**

```csharp
using UnityEngine;

public class HybridShadowController : MonoBehaviour
{
    [Header("Shadow Quality Tier")]
    [SerializeField] private ShadowTier tier = ShadowTier.Hybrid;

    [Header("Distance Thresholds")]
    [SerializeField] private float rtShadowDistance = 20f;      // RT 阴影距离
    [SerializeField] private float ssrtShadowDistance = 80f;    // SSRT 阴影距离
    [SerializeField] private float dfShadowDistance = 200f;     // DF 阴影距离

    [Header("References")]
    [SerializeField] private Light directionalLight;
    [SerializeField] private Material shadowBlendMaterial;

    private static readonly int RTShadowDistID = Shader.PropertyToID("_RTShadowDistance");
    private static readonly int SSRTShadowDistID = Shader.PropertyToID("_SSRTShadowDistance");
    private static readonly int DFShadowDistID = Shader.PropertyToID("_DFShadowDistance");
    private static readonly int ShadowTierID = Shader.PropertyToID("_ShadowTier");

    enum ShadowTier
    {
        FullRT = 0,       // PC: 硬件 RT
        SSRT = 1,         // 主机/高端移动: 屏幕空间 RT
        CSSM_PCSS = 2,    // 移动: Cascade + PCSS
        CSSM_PCF = 3      // 低端移动: Cascade + PCF
    }

    void Start()
    {
        // 根据设备能力选择 Tier
        tier = DetermineShadowTier();
        ApplyTierSettings();
    }

    ShadowTier DetermineShadowTier()
    {
        // 检查硬件 RT 支持
        if (SystemInfo.supportsRayTracing)
        {
            Debug.Log("[Shadow] Using Full RT tier");
            return ShadowTier.FullRT;
        }

        // 检查 Compute Shader 支持（SSRT 需要）
        if (SystemInfo.supportsComputeShaders &&
            SystemInfo.graphicsMemorySize > 2048)
        {
            Debug.Log("[Shadow] Using SSRT tier");
            return ShadowTier.SSRT;
        }

        // 检查 GPU 等级
        int shaderLevel = SystemInfo.graphicsShaderLevel;
        if (shaderLevel >= 50)
        {
            Debug.Log("[Shadow] Using CSSM + PCSS tier");
            return ShadowTier.CSSM_PCSS;
        }

        Debug.Log("[Shadow] Using CSSM + PCF tier (low-end)");
        return ShadowTier.CSSM_PCF;
    }

    void ApplyTierSettings()
    {
        Shader.SetGlobalInt(ShadowTierID, (int)tier);
        Shader.SetGlobalFloat(RTShadowDistID, rtShadowDistance);
        Shader.SetGlobalFloat(SSRTShadowDistID, ssrtShadowDistance);
        Shader.SetGlobalFloat(DFShadowDistID, dfShadowDistance);

        // 根据 Tier 配置 Unity 阴影设置
        var lightShadows = directionalLight.shadows;
        switch (tier)
        {
            case ShadowTier.FullRT:
                QualitySettings.shadowDistance = 150f;
                // RT 阴影由 Custom Pass 处理
                break;
            case ShadowTier.SSRT:
                QualitySettings.shadowDistance = 100f;
                break;
            case ShadowTier.CSSM_PCSS:
            case ShadowTier.CSSM_PCF:
                QualitySettings.shadowDistance = 60f;
                // 降低 Cascade 数量
                break;
        }
    }
}
```

**对比表：阴影方案全景**

| 方案 | 半影品质 | 最大距离 | 性能开销 | 适用平台 |
|------|---------|---------|---------|---------|
| CSSM + PCF | 固定核（假半影） | 级联范围 | 低（0.5ms） | 全平台 |
| CSSM + PCSS | 物理半影 | 级联范围 | 中（1-2ms） | PC/主机 |
| VSM（Virtual Shadow Map） | 高精度 | 大范围 | 中高（1.5-3ms） | PC/主机 |
| SSRT | 良好（屏幕空间内） | 屏幕范围 | 中（1-2ms） | 主机/高端移动 |
| DFAO（Distance Field） | 软遮蔽（非精确阴影） | 100m+ | 中（2-3ms） | PC |
| 硬件 RT Shadow | 最佳（物理正确） | 无限 | 高（3-6ms） | RTX 显卡 |
| Lumen（混合） | 良好 | 全场景 | 中高（2-4ms） | UE5 PC/主机 |

### ⚡ 实战经验

**最大的坑：SSRT 的屏幕边缘失效**

屏幕空间方案天生有局限——光线打到屏幕外的区域就没有遮挡信息了。解决方案是：
- 在 SSRT 射线离开屏幕时，自动回退到 DFAO 或 CSSM 阴影
- 用一个 `fadeMargin` 在屏幕边缘做平滑过渡，避免突然的阴影断裂
- UE5 Lumen 的做法：SSRT 失效区域用 Mesh Distance Field 补偿

**性能预算分配（60fps / 16.67ms）**

| 阴影层级 | 预算 | 说明 |
|---------|------|------|
| 近阴影（CSSM/RT） | 1.5ms | 高精度，距离最近 |
| 中阴影（SSRT/VSM） | 1.0ms | 屏幕空间方案 |
| 远阴影（DFAO/全局） | 0.5ms | 低精度近似 |
| 总计 | 3.0ms | 约总帧预算的 18% |

**VSM 是下一代方向**

UE5 的 Virtual Shadow Maps 已经在很多场景下取代了传统 Cascade：
- 分辨率可达 8K 级（但只渲染可见 Page）
- 大场景不需要多级 Cascade
- 与 Nanite 配合可以做到像素级精确阴影
- 移动端支持正在推进（GLES 3.2+ / Vulkan）

### 🎯 能力体检清单

| 检查项 | 如果你答不上来… |
|--------|----------------|
| 能解释 PCSS 比 PCF 好在哪里？ | → 阴影滤波原理不清晰 |
| 知道 VSM 的分页机制？ | → 虚拟纹理技术不熟 |
| 能说明 SSRT 为什么在屏幕边缘失效？ | → 屏幕空间方法局限性理解不足 |
| 理解 Lumen 里 Surface Cache 和 DF Tracing 各做什么？ | → UE5 GI 系统架构不熟 |
| 能设计跨平台阴影品质分级方案？ | → TA 架构设计能力不足 |
| 知道 RT Shadow 为什么要用 Denoiser？ | → RT 噪声和降噪原理不熟 |
| 理解阴影的「时域稳定性」为什么重要？ | → TAA/时域累积理解不足 |

### 🔗 相关问题

- [混合渲染管线：RT + Raster 的协同工作](rendering/ray-tracing-hybrid-pipeline)
- [Hi-Z 屏幕空间反射：SSRT 的另一种应用](rendering/hiz-screen-space-reflection)
- [TAA 时域抗锯齿：阴影稳定性的基础](rendering/temporal-anti-aliasing-taa)
