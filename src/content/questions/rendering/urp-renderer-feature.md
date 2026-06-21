---
title: "URP 下自定义 Renderer Feature：从零实现一个全屏描边后处理"
category: "rendering"
level: 3
tags: ["URP", "Renderer Feature", "后处理", "CommandBuffer", "渲染管线"]
hint: "核心考点：ScriptableRendererFeature 生命周期 + RenderPass 注入时机 + RT 管理 + Blit 链"
related: ["shader/npr-outline-cartoon", "rendering/post-process-chain"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们项目用 URP，策划要求实现一个功能：选中场景中的关键 NPC 时，NPC 身上出现一层发光描边（类似《塞尔达》的选中高亮）。不能用改 Shader 的方式（因为 NPC 共享材质），给我一个 Renderer Feature 方案。」

### ✅ 核心要点

1. **Renderer Feature 是 URP 的扩展点**：在不修改内置管线的情况下注入自定义 Pass
2. **选中物体隔离渲染**：用 Layer + RenderState 替换，单独渲染选中物体到 RT
3. **描边算法选择**：Sobel/Roberts 边缘检测（卷积）vs 法线外扩（双 Pass 模型放大）
4. **RT 生命周期管理**：`ConfigureInput` → `Blit` → 释放，避免内存泄漏
5. **性能权衡**：全屏后处理 vs 几何外扩，移动端选后者更稳

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：只有选中的 NPC 有发光描边，其他物体不变
                ↑
倒推1：不能改材质 → 必须在管线层面注入（Renderer Feature）
倒推2：需要分离选中物体 → 用 Layer 做掩码，单独渲染到一张 RT
倒推3：需要描边 → 对这张 RT 做边缘检测卷积
倒推4：需要发光 → 边缘检测结果做 Blur 后叠加回主画面
倒推5：需要可控开关 → Feature 的 renderPassEvent 控制注入时机
```

#### 知识点拆解（倒推树）

```
URP 自定义 Renderer Feature
├── URP 架构理解
│   ├── ScriptableRendererFeature（入口：Create + AddRenderPasses）
│   ├── ScriptableRenderPass（核心：Execute + OnCameraSetup/Teardown）
│   ├── RenderPassEvent 枚举（BeforeRenderingTransparents / AfterRenderingOpaques / AfterRenderingPostProcessing）
│   └── RenderingUtils.CopyTexture / Blitter.BlitCameraTexture
├── 选中物体隔离
│   ├── LayerMask 过滤
│   ├── DrawingSettings override（替换材质为纯色白）
│   ├── FilteringSettings override（只渲染选中 Layer）
│   └── RenderTargetHandle / RTHandle 管理
├── 描边算法
│   ├── 方案A：边缘检测卷积（Sobel 算子，全屏 Pass）
│   ├── 方案B：法线+深度边缘检测（更准确但需 DepthNormal）
│   └── 方案C：双 Pass 法线外扩（不需要全屏 Pass，移动端推荐）
├── RT 管理（URP 14+ RTHandle API）
│   ├── RTHandles.Alloc → 临时 RT
│   ├── ConfigureInput(Texture Depth) 确保深度可用
│   └── ReleaseTemporaryRT / RTHandle.Release
└── Blit 链
    ├── 主画面 → 描边 RT（卷积/外扩）
    ├── 描边 RT → Blur（可选）
    └── Blur 结果 → 叠加回主画面（Additive Blend）
```

#### 代码实现

**Renderer Feature（入口）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class OutlineFeature : ScriptableRendererFeature
{
    public LayerMask outlineLayer;       // 选中物体的 Layer
    public Material outlineMaterial;     // 描边后处理材质
    public RenderPassEvent passEvent = RenderPassEvent.AfterRenderingOpaques;

    OutlineRenderPass _pass;

    public override void Create()
    {
        _pass = new OutlineRenderPass(outlineLayer, outlineMaterial)
        {
            renderPassEvent = passEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (outlineMaterial == null) return;
        _pass.Setup(renderer.cameraColorTargetHandle);
        renderer.EnqueuePass(_pass);
    }

    protected override void Dispose(bool disposing) => _pass?.Dispose();
}
```

**Render Pass（核心逻辑）：**

```csharp
class OutlineRenderPass : ScriptableRenderPass
{
    LayerMask _layer;
    Material _mat;
    RTHandle _outlineTex;
    RTHandle _source;

    static readonly int OutlineTexID = Shader.PropertyToID("_OutlineTex");
    static readonly int TempTexID = Shader.PropertyToID("_TempTex");

    public OutlineRenderPass(LayerMask layer, Material mat)
    {
        _layer = layer;
        _mat = mat;
    }

    public void Setup(RTHandle source) => _source = source;

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData data)
    {
        // 分配描边用的临时 RT
        var desc = data.cameraData.cameraTargetDescriptor;
        desc.depthBufferBits = 0;
        RenderingUtils.ReAllocateIfNeeded(ref _outlineTex, desc, name: "_OutlineTex");
    }

    public override void Execute(ScriptableRenderContext ctx, ref RenderingData data)
    {
        CommandBuffer cmd = CommandBufferPool.Get("OutlineFeature");
        cmd.Clear();

        // Step 1: 隔离渲染选中物体到 _outlineTex
        var drawSettings = CreateDrawingSettings(
            new ShaderTagId("UniversalForward"),
            ref data, RenderingData.defaultLightMode);

        // 替换为纯色白材质（通过 OverrideMaterial）
        drawSettings.overrideMaterial = _mat;
        drawSettings.overrideMaterialPassIndex = 0;

        var filterSettings = new FilteringSettings(_layer);

        cmd.SetRenderTarget(_outlineTex);
        cmd.ClearRenderTarget(false, true, Color.clear);
        ctx.ExecuteCommandBuffer(cmd);
        cmd.Clear();

        // 执行隔离绘制
        ctx.DrawRenderers(data.cullResults, ref drawSettings, ref filterSettings);

        // Step 2: 边缘检测卷积
        cmd.GetTemporaryRT(TempTexID, data.cameraData.cameraTargetDescriptor);
        cmd.SetGlobalTexture("_OutlineTex", _outlineTex);
        Blitter.BlitCameraTexture(cmd, _outlineTex, TempTexID, _mat, 1); // pass 1: edge detect
        Blitter.BlitCameraTexture(cmd, TempTexID, _source, _mat, 2);     // pass 2: composite

        ctx.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public override void OnCameraCleanup(CommandBuffer cmd)
        => cmd.ReleaseTemporaryRT(TempTexID);

    public void Dispose() => _outlineTex?.Release();
}
```

**描边 Shader（边缘检测 + 合成）：**

```hlsl
// Pass 1: Sobel 边缘检测
half4 FragEdge(Varyings IN) : SV_Target
{
    float2 uv = IN.uv;
    float2 texel = _MainTex_TexelSize.xy;

    // 采样 3x3 邻域
    half l = SAMPLE_TEXTURE2D(_OutlineTex, sampler_OutlineTex, uv - float2(texel.x, 0)).a;
    half r = SAMPLE_TEXTURE2D(_OutlineTex, sampler_OutlineTex, uv + float2(texel.x, 0)).a;
    half d = SAMPLE_TEXTURE2D(_OutlineTex, sampler_OutlineTex, uv - float2(0, texel.y)).a;
    half u = SAMPLE_TEXTURE2D(_OutlineTex, sampler_OutlineTex, uv + float2(0, texel.y)).a;

    // Sobel 梯度
    half gx = abs(r - l);
    half gy = abs(u - d);
    half edge = saturate(gx + gy);

    return half4(_OutlineColor.rgb * edge, edge);
}

// Pass 2: 叠加到主画面
half4 FragComposite(Varyings IN) : SV_Target
{
    half4 src = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv);
    half edge = SAMPLE_TEXTURE2D(_OutlineTex, sampler_OutlineTex, IN.uv).a;
    return lerp(src, half4(_OutlineColor.rgb, 1), edge * _OutlineIntensity);
}
```

**方案对比表：**

| 方案 | 适用管线 | 性能（移动端） | 描边质量 | 实现复杂度 |
|------|----------|----------------|----------|------------|
| Renderer Feature + 边缘检测 | URP | 中（全屏 Pass ×2） | 好 | ★★★☆☆ |
| Renderer Feature + 法线外扩 | URP | 高（几何 Pass ×2） | 优秀 | ★★★★☆ |
| 修改物体 Shader（模板测试） | 任意 | 最高 | 一般 | ★★☆☆☆ |
| 后处理全屏描边（无隔离） | 任意 | 低 | 差（全屏描边） | ★☆☆☆☆ |

### ⚡ 实战经验

- **RTHandle vs RenderTargetHandle**：URP 14+ 迁移到 RTHandle API，旧代码用 `RenderingUtils.ReAllocateIfNeeded` 而非 `GetTemporaryRT`
- **多相机问题**：Feature 默认对所有相机生效，用 `renderingData.cameraData.cameraType` 过滤掉 Preview 相机
- **移动端性能**：边缘检测卷积是全屏 Pass，中低端机 GPU 压力大。移动端优先选法线外扩方案
- **MSAA 兼容**：隔离渲染 RT 用 `resolveTarget` 才能正确拿到 MSAA 结果，否则边缘有锯齿

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道在哪继承 | URP 扩展机制 | 读 ScriptableRendererFeature/Pass 源码 |
| 选中物体隔离不出来 | DrawingSettings / FilteringSettings | 学 URP 的 DrawRenderers API |
| RT 画面全黑 | RT 生命周期管理 | OnCameraSetup → Execute → OnCameraCleanup |
| 描边有残影 | RT 没清空 | 每帧 ClearRenderTarget(false, true, Color.clear) |
| 合批断裂 | overrideMaterial 影响 SRP Batcher | 理解 overrideMaterial 和 per-instance 区别 |

### 🔗 相关问题

- 如何实现「多个选中物体各自不同颜色描边」？（提示：自定义顶点色 / stencil ID 区分）
- Renderer Feature 和后处理 Volume Component 的区别？什么时候用哪个？
- 如何在 HDRP 中实现同样的描边效果？两套管线 API 的差异在哪？
