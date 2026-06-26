---
title: "URP 下如何实现 Hi-Z 屏幕空间反射？从 SSR 到 Hi-Z 的性能飞跃"
category: "rendering"
level: 4
tags: ["Hi-Z", "SSR", "屏幕空间反射", "URP", "Renderer Feature", "Compute Shader"]
hint: "Hi-Z 用分层 Mipmap 金字塔加速射线步进，把 SSR 的 O(N) 线性步进变成 O(logN)——面试中能说清这个就是加分项"
related: ["rendering/sspr-screen-space-planar-reflection", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们用 URP 做一个开放世界手游，地面反射效果太差——Planar Reflection 太贵，普通 SSR 性能也扛不住。你能不能在 URP 下实现一个 Hi-Z（Hierarchical-Z）屏幕空间反射？说说你的方案和性能预期。」

### ✅ 核心要点

1. **Hi-Z 核心思想**：构建深度金字塔（Depth Mipmap Hierarchy），在低分辨率层级做大步进，逐步收敛到高分辨率层级做精细检测
2. **vs 传统 SSR**：线性步进（Ray Marching）是 O(N)，Hi-Z 是 O(log N)——屏幕越大优势越明显
3. **URP 集成路径**：自定义 Renderer Feature + Compute Shader 生成 Hi-Z + 全屏后处理 Pass
4. **反射质量增强**：结合 Color Mipmap Pyramid 做roughness-based 模糊反射（类似 GGX 重要性采样近似）
5. **性能边界**：移动端需降采样 + 限制射线步数；PC 端可做半分辨率高质量版

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：湿润地面反射周围建筑/天空，粗糙表面反射模糊，光滑表面反射清晰
                ↑
倒推1：需要屏幕空间射线与场景几何相交 → Hi-Z Ray Tracing（分层加速）
倒推2：需要场景颜色做反射采样 → Copy Color Pass（或直接采样 Color Attachment）
倒推3：需要根据粗糙度模糊反射 → Color Mipmap Pyramid（Prefiltered 反射色）
倒推4：需要还原反射方向 → View-Space 反射向量计算（reflect(viewDir, normal)）
倒推5：需要与原始画面混合 → 基于 Fresnel + Roughness 的混合权重
倒推6：需要 URP 集成 → 自定义 Renderer Feature（BeforeRenderingPostProcessing）
```

#### 知识点拆解（倒推树）

```
Hi-Z 屏幕空间反射
├── Hi-Z 金字塔构建
│   ├── Compute Shader 生成 Depth Mipmap（min/max depth per 2x2 tile）
│   ├── 为什么用 max depth？（保守遮挡，避免穿透）
│   └── Mipmap 层级数 = log2(max(screenW, screenH))
├── Hi-Z Ray Marching 算法
│   ├── 起始层级选择（从最粗层级开始，逐步细化）
│   ├── 射线-高度图相交检测（Hiz Cell 上下界 vs 射线深度区间）
│   ├── 层级切换策略（相交则降层级，未相交则升层级/前进）
│   └── 终止条件（步数上限 / 命中像素 / 出屏）
├── 反射色采样
│   ├── Color Mipmap Pyramid（重要性采样近似 GGX 模糊）
│   ├── 屏幕边缘外推（Screen Border Reflection 修复）
│   └── 时间累积（Temporal Filter 减少噪点）
├── URP 集成
│   ├── RenderPass：BeforeRenderingPostProcessing
│   ├── Compute Buffer：存储 Hi-Z Texture（每帧更新）
│   ├── Blit 到临时 RT → 合成回 Camera Color
│   └── Stencil 标记反射面（只对地面/水面做 SSR）
└── 性能优化
    ├── 半分辨率 Hi-Z 构建（降采样后再生成金字塔）
    ├── 射线步数限制（移动端 16 步，PC 64 步）
    ├── Early Exit（遇到天空盒直接跳过）
    └── Temporal Reprojection（上一帧复用，减少每帧射线数）
```

#### 代码实现

**1. Hi-Z 金字塔生成（Compute Shader）：**

```hlsl
#pragma kernel HizMip

Texture2D<float> _SourceDepth;
RWTexture2D<float> _OutputDepth;
uint2 _SourceSize;
uint _MipLevel;

[numthreads(8, 8, 1)]
void HizMip(uint3 id : SV_DispatchThreadID)
{
    if (any(id.xy * 2 >= _SourceSize)) return;

    // 取 2x2 像素的最大深度值（保守遮挡）
    float d00 = _SourceDepth.Load(uint3(id.xy * 2, 0));
    float d10 = _SourceDepth.Load(uint3(id.xy * 2 + uint2(1, 0), 0));
    float d01 = _SourceDepth.Load(uint3(id.xy * 2 + uint2(0, 1), 0));
    float d11 = _SourceDepth.Load(uint3(id.xy * 2 + uint2(1, 1), 0));

    float maxDepth = max(max(d00, d10), max(d01, d11));
    _OutputDepth[id.xy] = maxDepth;
}
```

**2. Hi-Z Ray Marching（Fragment/Compute Shader 核心）：**

```hlsl
// Hi-Z Screen Space Ray Marching
// 输入：反射射线起点（view space）、反射方向、Hi-Z Texture
// 输出：命中 UV 坐标 + 命中标志

struct HizRayHit
{
    float2 hitUV;
    bool isHit;
};

HizRayHit HizRayMarch(float3 rayOriginVS, float3 rayDirVS,
                       Texture2D<float> hizTexture, int maxMipLevel,
                       int maxIterations)
{
    HizRayHit result;
    result.isHit = false;
    result.hitUV = float2(0, 0);

    float2 currentUV = projectToScreen(rayOriginVS);
    float3 currentPosVS = rayOriginVS;
    int currentMip = 0;

    [loop]
    for (int i = 0; i < maxIterations; i++)
    {
        // 步进
        currentPosVS += rayDirVS * stepSize;
        float2 sampleUV = projectToScreen(currentPosVS);

        // 检查出屏
        if (any(sampleUV < 0) || any(sampleUV > 1)) break;

        // 采样当前层级的 Hi-Z
        float sceneDepth = hizTexture.mips[currentMip].Load(int3(sampleUV * screenSizeAtMip(currentMip), 0));
        float rayDepth = currentPosVS.z; // view space depth

        if (rayDepth <= sceneDepth)
        {
            // 射线穿过了表面 → 命中
            if (currentMip == 0)
            {
                // 最精细层级命中
                result.isHit = true;
                result.hitUV = sampleUV;
                break;
            }
            else
            {
                // 降层级，精细搜索
                currentMip = max(0, currentMip - 1);
                // 回退一步
                currentPosVS -= rayDirVS * stepSize;
            }
        }
        else
        {
            // 射线在表面前方 → 前进（可选升层级加速）
            currentMip = min(currentMip + 1, maxMipLevel);
        }
    }

    return result;
}
```

**3. URP Renderer Feature 注册：**

```csharp
public class HiZSSRFeature : ScriptableRendererFeature
{
    private HiZSSRPass _pass;
    public ComputeShader hizCompute;
    public Material ssrMaterial;
    public RenderPassEvent passEvent = RenderPassEvent.BeforeRenderingPostProcessing;

    public override void Create()
    {
        _pass = new HiZSSRPass(hizCompute, ssrMaterial)
        {
            renderPassEvent = passEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        var cameraColor = renderer.cameraColorTargetHandle;
        _pass.Setup(cameraColor);
        renderer.EnqueuePass(_pass);
    }
}

// Pass 内核心逻辑
public class HiZSSRPass : ScriptableRenderPass
{
    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        // 1. 构建 Hi-Z 金字塔（Compute Shader Dispatch）
        // 每级 Dispatch：尺寸减半，直到 1x1
        int mipLevels = (int)Mathf.Log2(Mathf.Max(screenW, screenH));
        for (int mip = 1; mip < mipLevels; mip++)
        {
            hizCompute.SetInt("_MipLevel", mip);
            // Dispatch with appropriate thread group count
        }

        // 2. SSR 全屏 Pass
        cmd.Blit(colorTarget, tempTarget, ssrMaterial);
        cmd.Blit(tempTarget, colorTarget);
    }
}
```

**性能对比表：**

| SSR 方案 | 射线步数（1080p） | GPU 耗时（Adreno 650） | 反射质量 | 适用平台 |
|----------|-------------------|------------------------|----------|----------|
| 线性 Ray Marching | 128-256 步 | 4.2ms | 中等 | PC/主机 |
| Hi-Z Ray Marching | 16-32 步 | 1.1ms | 中高 | PC/主机/高端移动 |
| Planar Reflection | N/A（额外渲染） | 3-8ms（取决于场景） | 高 | PC/主机 |
| SSPR（平面反射） | 1 Pass | 0.8ms | 中（仅平面） | 移动端首选 |

### ⚡ 实战经验

- **移动端降采样策略**：Hi-Z 在 1/2 分辨率构建，SSR Pass 也在 1/2 分辨率执行，最后 Upscale 到全屏。移动端 1ms 以内可达
- **粗糙度模糊**：不要在 SSR Pass 里做真正的 GGX 重要性采样。预构建 Color Mipmap Pyramid，用 roughness 映射到 mip level，近似效果出奇地好
- **边缘失效修复**：SSR 天然无法反射屏幕外内容。可以用上一帧的 Color RT 做历史累积，或用 CubeMap 做边缘 Fallback
- **Temporal Filter 注意事项**：Hi-Z SSR 每帧射线方向加 jitter（Halton 序列），再用 Motion Vector 做 Temporal Reprojection。但注意动态物体拖影——用 Disocclusion Mask 排除
- **面试加分项**：提到 G-Buffer 重建法线（从深度梯度重建），避免额外 Normal RT 开销

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 Hi-Z 金字塔是什么 | 深度金字塔原理 | 学 Hierarchy-Z Buffer 可见性剔除 |
| 射线步进逻辑写不对 | View Space ↔ Screen Space 变换 | 复习投影矩阵推导 + 齐次坐标 |
| URP 下不知道怎么注入 Compute Pass | URP Renderer Feature 机制 | 学 ScriptableRendererFeature + ComputeBuffer |
| 反射粗糙模糊做不出 | Color Mipmap Prefilter | 学 Epic 的 Pre-filtered Environment Map 方案 |
| 移动端跑不动 | 移动端 GPU 架构理解 | 学 TBDR（Tile-Based Deferred Rendering）与 Compute 效率 |
| 动态物体拖影严重 | Temporal Anti-Aliasing 原理 | 学 TAA 的 Reprojection + Disocclusion |

### 🔗 相关问题

- SSR 和 Planar Reflection 各自的优缺点？什么场景下选哪个？
- 如何处理 SSR 在第三人称角色身上的反射瑕疵（角色反射穿模）？
- 延迟渲染管线下的 Hi-Z SSR 和前向渲染下有什么区别？
- 如果不用 Compute Shader（如 WebGL/GLES 3.0），如何在 Fragment Shader 中近似 Hi-Z？
