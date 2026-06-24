---
title: "面试官要你在URP下实现可控的屏幕空间阴影柔化——PCF与PCSS的工程化落地方案"
category: "rendering"
level: 4
tags: ["URP", "阴影", "PCF", "PCSS", "屏幕空间", "ShadowMap", "RendererFeature", "可变柔度"]
hint: "PCF是均匀柔化（快），PCSS是物理柔化（真实）——核心都是ShadowMap多采样+权重混合，PCSS额外根据距离自适应柔度"
related: ["rendering/urp-renderer-feature", "rendering/custom-ssao-urp", "rendering/planar-projection-shadow", "rendering/urp-custom-outline-render-feature"]
---

## 参考答案

### 🎬 场景描述

面试官在白板上画了一个角色站在大太阳下的示意图：

> "我们项目用的是 Unity URP，主光阴影用的是内置的 Cascaded Shadow Map。现在美术反馈说阴影边缘太硬了，像刀切一样——他们想要那种自然柔和的半阴影过渡，太阳越大阴影越柔，离物体越远阴影越柔。
>
> URP 自带的 Soft Shadows 开了但效果不够。你怎么实现一个可控的阴影柔化方案？"

这是腾讯天美、米哈游、叠纸等做高品质渲染的 **TA 高级岗经典题**。考察 Shadow Map 原理、PCF/PCSS 算法理解、URP 自定义 Renderer Feature 的工程能力。

### ✅ 核心要点

1. **问题本质**：Shadow Map 是硬阴影（每个像素要么在阴影里要么不在），柔化 = 在阴影边缘做多采样插值
2. **PCF（Percentage Closer Filtering）**：在 Shadow Map 上做多点采样 + 权重混合，产生均匀柔化
3. **PCSS（Percentage Closer Soft Shadows）**：根据遮挡距离自适应调节采样半径——离遮挡物越远越柔（物理正确）
4. **URP 实现路径**：自定义 Renderer Feature → 替换/叠加阴影 Pass → Blit 到屏幕
5. **性能关键**：采样次数 vs 柔化质量的权衡，旋转采样圆盘（Poisson Disk）减少采样数

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：阴影边缘自然柔化，距离遮挡物越远越柔
                    ↓
需要什么：每个像素的阴影值不再是 0 或 1，而是 0~1 的渐变
                    ↓
方法1：PCF（Percentage Closer Filtering）
  ├── 在当前像素的 Shadow Map UV 周围取 N 个点
  ├── 每个点做深度比较（0 或 1）
  ├── 把 N 个结果取平均 → 0~1 的柔化值
  └── 缺点：柔化程度均匀，不随距离变化
                    ↓
方法2：PCSS（Percentage Closer Soft Shadows）
  ├── Step A：Blocker Search —— 搜索周围哪些点是遮挡物
  ├── Step B：计算平均遮挡距离（blocker distance）
  ├── Step C：根据遮挡距离计算采样半径（越远越大）
  └── Step D：在计算出的半径内做 PCF
  → 效果：遮挡物近 = 硬阴影，遮挡物远 = 柔阴影（物理正确）
                    ↓
工程实现：
  ├── URP Renderer Feature：自定义阴影 Pass
  ├── 输入：主光源 Shadow Map + 深度图
  ├── 处理：PCSS 计算
  └── 输出：柔化后的阴影遮罩，与画面混合
```

#### 知识点拆解（倒推树）

```
屏幕空间阴影柔化
├── Shadow Map 基础
│   ├── 原理：从光源视角渲染深度图，比较深度判断遮挡
│   ├── Cascaded Shadow Map（CSM）：多级阴影贴图处理大场景
│   ├── 深度精度：Depth Bias / Normal Bias 消除 Shadow Acne
│   └── URP 中的阴影：ShadowPass → ScreenSpaceShadowPass → 最终混合
├── PCF（Percentage Closer Filtering）
│   ├── 基本采样：3x3 / 5x5 / 7x7 采样网格
│   ├── Poisson Disk 采样：旋转的不规则采样（减少 Banding）
│   ├── 旋转抖动（Dither）：每像素不同偏移，消除重复纹理
│   └── 采样次数权衡：4次（低品质）→ 16次（高品质）→ 64次（离线品质）
├── PCSS（Percentage Closer Soft Shadows）
│   ├── Blocker Search Step：在一定半径内搜索 Shadow Map 中的遮挡物
│   ├── Penumbra Size 计算：penumbra = (receiverDepth - blockerDepth) * lightSize / blockerDepth
│   ├── 自适应 PCF：用 penumbra size 作为采样半径
│   └── 级联处理：不同 CSM 级别用不同的搜索半径
├── URP 工程实现
│   ├── ScriptableRendererFeature：注册自定义 Pass
│   ├── ScriptableRenderPass：执行阴影柔化的渲染逻辑
│   ├── Blit：把屏幕纹理传给自定义 Shader
│   ├── Shadow Map 获取：renderer.cameraDepthTarget / m_ShadowMapTexture
│   └── Shader 关键字：_MAIN_LIGHT_SHADOWS / _SHADOWS_SOFT
└── 性能优化
    ├── 采样数控制：移动端 4~8 次 PCF，PC 端 16 次 PCSS
    ├── 分辨率控制：半分辨率计算阴影柔化，然后 Upscale
    ├── Early Out：阴影区域外直接跳过
    └── 时间复用：上一帧阴影结果 + 当前帧混合（TAA 思路）
```

#### 代码实现

**1. URP Renderer Feature（C#）**：

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class SoftShadowFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingOpaques;
        [Range(1, 64)] public int pcfSamples = 16;
        [Range(0.001f, 0.1f)] public float blockerSearchRadius = 0.02f;
        [Range(0.5f, 10f)] public float lightSize = 2.0f;   // 太阳大小，越大越柔
        public bool halfResolution = true;                    // 半分辨率性能优化
    }

    public Settings settings = new();
    private SoftShadowPass _pass;

    public override void Create()
    {
        _pass = new SoftShadowPass(settings)
        {
            renderPassEvent = settings.renderPassEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        // 只在有主光源阴影时执行
        if (renderingData.lightData.mainLightIndex < 0) return;
        if (!renderingData.shadowData.supportsMainLightShadows) return;

        _pass.Setup(renderer);
        renderer.EnqueuePass(_pass);
    }
}

public class SoftShadowPass : ScriptableRenderPass
{
    private readonly SoftShadowFeature.Settings _settings;
    private readonly Material _shadowMaterial;
    private RTHandle _shadowTexture;

    private static readonly int PCFSamplesID = Shader.PropertyToID("_PCFSamples");
    private static readonly int BlockerRadiusID = Shader.PropertyToID("_BlockerSearchRadius");
    private static readonly int LightSizeID = Shader.PropertyToID("_LightSize");
    private static readonly int ShadowMapID = Shader.PropertyToID("_SoftShadowMap");

    public SoftShadowPass(SoftShadowFeature.Settings settings)
    {
        _settings = settings;
        var shader = Shader.Find("Hidden/Custom/PCSS");
        _shadowMaterial = CoreUtils.CreateEngineMaterial(shader);
    }

    public void Setup(ScriptableRenderer renderer)
    {
        // 在 Setup 中获取 Shadow Map
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        if (_settings.halfResolution)
        {
            desc.width /= 2;
            desc.height /= 2;
        }
        RenderingUtils.ReAllocateIfNeeded(ref _shadowTexture, desc, name: "_SoftShadowTex");
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        var cmd = CommandBufferPool.Get("CustomSoftShadow");

        cmd.SetGlobalInt(PCFSamplesID, _settings.pcfSamples);
        cmd.SetGlobalFloat(BlockerRadiusID, _settings.blockerSearchRadius);
        cmd.SetGlobalFloat(LightSizeID, _settings.lightSize);

        // Blit: 深度图 + ShadowMap → PCSS 计算 → 输出柔化阴影
        Blitter.BlitCameraTexture(cmd, _shadowTexture, _shadowTexture, _shadowMaterial, 0);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public override void OnCameraCleanup(CommandBuffer cmd)
    {
        // RTHandle 由 ReAllocateIfNeeded 管理
    }
}
```

**2. PCSS Shader（HLSL）**：

```hlsl
// Hidden/Custom/PCSS
Shader "Hidden/Custom/PCSS"
{
    SubShader { Tags { "RenderType" = "Opaque" } Pass { /* ... */ } }

    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Shadows.hlsl"

    // Poisson Disk 采样点（32个，黄金角旋转分布）
    static const float2 poissonDisk[32] = {
        float2(0.36485, 0.11128), float2(-0.21611, 0.35667),
        float2(-0.34587, -0.21357), float2(0.28193, -0.31872),
        // ... 更多 Poisson 采样点
    };

    TEXTURE2D(_MainLightShadowmapTexture);
    SAMPLER(sampler_MainLightShadowmapTexture);
    float4 _MainLightShadowmapTexture_TexelSize;

    int _PCFSamples;
    float _BlockerSearchRadius;
    float _LightSize;

    // === Blocker Search：搜索遮挡物的平均深度 ===
    float SearchBlocker(float2 uv, float receiverDepth)
    {
        int blockers = 0;
        float blockerDepth = 0.0;

        // 在一定半径内搜索
        int sampleCount = min(_PCFSamples, 16); // Blocker Search 用较少采样
        for (int i = 0; i < sampleCount; i++)
        {
            // 旋转 Poisson Disk（每像素不同偏移，消除 Banding）
            float angle = frac(uv.x * 1234.0 + uv.y * 5678.0) * 6.283185;
            float s, c; sincos(angle, s, c);
            float2 offset = mul(float2x2(c, -s, s, c), poissonDisk[i % 32]);
            offset *= _BlockerSearchRadius * _MainLightShadowmapTexture_TexelSize.xy * 10.0;

            float shadowDepth = SAMPLE_TEXTURE2D(_MainLightShadowmapTexture,
                sampler_MainLightShadowmapTexture, uv + offset).r;

            if (shadowDepth < receiverDepth)
            {
                blockers++;
                blockerDepth += shadowDepth;
            }
        }

        if (blockers == 0) return -1.0;  // 无遮挡物，不在阴影中
        return blockerDepth / blockers;
    }

    // === Penumbra Size：计算半阴影半径 ===
    float PenumbraSize(float blockerDepth, float receiverDepth)
    {
        // penumbra = lightSize * (receiverDepth - blockerDepth) / blockerDepth
        float penumbra = _LightSize * (receiverDepth - blockerDepth) / blockerDepth;
        return max(penumbra, 0.001);
    }

    // === PCF Filter：多点采样混合 ===
    float PCFFilter(float2 uv, float receiverDepth, float filterRadius)
    {
        float sum = 0.0;
        for (int i = 0; i < _PCFSamples; i++)
        {
            // 旋转 Poisson Disk
            float angle = frac(uv.x * 1234.0 + uv.y * 5678.0) * 6.283185;
            float s, c; sincos(angle, s, c);
            float2 offset = mul(float2x2(c, -s, s, c), poissonDisk[i % 32]);
            offset *= filterRadius * _MainLightShadowmapTexture_TexelSize.xy;

            float shadowDepth = SAMPLE_TEXTURE2D(_MainLightShadowmapTexture,
                sampler_MainLightShadowmapTexture, uv + offset).r;

            sum += (shadowDepth < receiverDepth) ? 0.0 : 1.0;
        }
        return sum / _PCFSamples;
    }

    // === PCSS 主函数 ===
    float PCSS(float2 uv, float receiverDepth)
    {
        // Step 1: Blocker Search
        float blockerDepth = SearchBlocker(uv, receiverDepth);
        if (blockerDepth < 0.0) return 1.0;  // 无遮挡

        // Step 2: Compute Penumbra Size
        float penumbra = PenumbraSize(blockerDepth, receiverDepth);

        // Step 3: PCF with adaptive radius
        return PCFFilter(uv, receiverDepth, penumbra);
    }

    float4 Fragment(Varyings input) : SV_Target
    {
        float2 uv = input.uv;
        // 重建世界坐标 → 转到光源空间获取 receiverDepth
        float3 worldPos = ComputeWorldSpacePosition(uv, depth, unity_MatrixInvVP);
        float4 shadowCoord = TransformWorldToShadowCoord(worldPos);

        // 从 shadowCoord 获取 UV 和 depth
        float2 shadowUV = shadowCoord.xy;
        float receiverDepth = shadowCoord.z;

        // PCSS 计算柔化阴影
        float shadow = PCSS(shadowUV, receiverDepth);

        return float4(shadow.xxx, 1.0);
    }
    ENDHLSL
}
```

### ⚡ 实战经验

> **经验1**：PCSS 的 Blocker Search 是性能大头。移动端建议固定一个较小的搜索半径（如 3x3），而不是完全自适应。
>
> **经验2**：`_MainLightShadowmapTexture_TexelSize.xy` 是 Shadow Map 单像素大小。用它来缩放采样偏移，确保不同 Cascade 级别下柔化程度一致。
>
> **经验3**：半分辨率渲染阴影柔化是最有效的性能优化——视觉差异极小，性能提升近 4 倍。完成后 Upscale 回全分辨率。
>
> **经验4**：Poisson Disk 的旋转抖动是消除 Banding 的关键。如果不旋转，16 次采样在阴影边缘会有明显的条纹纹理。
>
> **经验5**：URP 14+ 已内置 `Soft Shadows` 选项（Project Settings → Quality → Shadows → Soft Shadows），但它只是 3x3 PCF。自定义 PCSS 是为了实现物理正确的距离自适应柔化。
>
> **经验6**：`_LightSize` 参数是美术可调的「太阳大小」——值越大阴影越柔。给美术暴露这个参数，让他们自己控制风格。

### 🎯 能力体检清单

| 如果答不上来... | 说明你的盲区是... |
|---|---|
| 说不清 Shadow Map 的基本原理 | 实时阴影基础知识不牢 |
| 不知道 PCF 和 PCSS 的区别 | 阴影柔化算法知识缺失 |
| 不会在 URP 中写自定义 Renderer Feature | URP 扩展能力不足 |
| 不知道 Poisson Disk 采样是什么 | 采样理论 + 图形学数学基础薄弱 |
| 说不清 Penumbra Size 公式的推导 | 光学几何 + 半阴影物理理解不够 |
| 不知道半分辨率渲染优化的做法 | 移动端性能优化经验不足 |

### 🔗 相关问题

- [URP 自定义 Renderer Feature 怎么写？](rendering/urp-renderer-feature)
- [URP 下自定义 SSAO](rendering/custom-ssao-urp)
- [平面投影阴影怎么做？](rendering/planar-projection-shadow)
- [URP 卡通描边 Render Feature](rendering/urp-custom-outline-render-feature)
