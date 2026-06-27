---
title: "Mali GPU 帧率骤降：如何诊断 FPR Rollback 导致的 Tile 回滚开销？"
category: "optimization"
level: 4
tags: ["Mali GPU", "TBR", "FPR", "Tile Memory", "Frame Debugger", "移动端优化", "Arm Mobile Studio"]
hint: "FPR Rollback 是 Mali 上的隐形杀手——GPU 被迫把 Tile 数据写回系统内存再读回来，带宽和延迟同时暴增。用 Streamline 抓 Frontend 和 Backend 的 Tile 内存命中率就能定位"
related: ["optimization/adreno-tile-based-bandwidth", "optimization/mobile-gpu-profiling-toolchain", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的游戏在骁龙 8 Gen 3 上跑 60fps 没问题，但在天玑 9300（Mali-G715）上偶尔会掉到 38fps，Frame Debugger 看不出明显异常——Draw Call 数没变、三角形数也正常。你怀疑是什么问题？怎么进一步定位？」

追问：「你提到了 FPR Rollback，能详细解释一下它为什么会在某些场景触发吗？哪些渲染操作最容易触发？」

### ✅ 核心要点

1. **TBR/TBDR 架构本质**：Mali GPU 把帧分成 Tile，每个 Tile 在片上 Tile Memory 中完成所有渲染再写回系统内存
2. **FPR (Frame Pipeline Rollback)**：当一个 Tile 还没渲染完就被迫写回系统内存、之后再读回来继续渲染——带宽暴增 2-3 倍
3. **触发条件**：Render Pass 间的依赖、过多 Pass、显存压力导致 Tile Memory 不足
4. **诊断工具**：Arm Streamline + Mali Offline Compiler + FrameAdvisor
5. **常见修复**：合并 Pass、使用 Tile Local Storage（TLS）、减少 Render Target 数量

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
现象：天玑 9300 上特定场景帧率从 60 骤降到 38
├── Frame Debugger 看不出异常 → 不是 Draw Call / 三角形问题
├── GPU 频率正常 → 不是热降频（排除 Thermal Throttling）
├── Mali 特有架构 → Tile-Based Deferred Rendering (TBDR)
│   └── 隐形开销在 Tile Memory ↔ System Memory 的回滚
├── 诊断路径
│   ├── Arm Streamline 抓 L2 Cache 命中率和外部内存带宽
│   ├── 看是否有 Bandwidth Spike（FPR 发生时带宽翻倍）
│   └── Mali FrameAdvisor 分析 Render Pass 结构
└── 验证 → 修复后带宽下降、帧率恢复
```

#### 知识点拆解（倒推树）

```
Mali GPU FPR Rollback 诊断
├── Mali TBDR 架构理解
│   ├── Tile Renderer 工作流
│   │   ├── Vertex Pass → 每个 Tile 的几何被分配到 Tile List
│   │   ├── Fragment Pass → 每个 Tile 在 Tile Memory 中完成所有 Fragment 操作
│   │   └── Tile Flush → 最终结果写回系统内存
│   ├── Tile Memory（片上 SRAM）
│   │   ├── 大小有限（G715 约 2-4MB，取决于配置）
│   │   ├── 如果一个 Tile 需要的 RT + 深度 + 模板 > Tile Memory 容量 → 回滚
│   │   └── 这就是 FPR 的根本原因
│   └── Transaction Elimination（不相关于此问题，但影响带宽）
├── FPR 触发的具体场景
│   ├── 场景1：Render Target 数量 × 分辨率 > Tile Memory
│   │   ├── MRT (Multiple Render Targets)：G-Buffer 4 RT + 深度 + 模板
│   │   ├── 高分辨率（1440p/4K）下单 Tile 像素更多
│   │   └── 修复：降低 RT 分辨率 / 合并 RT / 用 RGBA64 替代 RGBA32*4
│   ├── 场景2：Render Pass 之间的强制依赖
│   │   ├── Pass A 输出 → Pass B 输入 → Pass C 又读 Pass A 输出
│   │   ├── Mali 需要保留 Pass A 的 Tile，如果排不下就回滚
│   │   └── 修复：重排 Pass 顺序 / 合并相邻 Pass
│   ├── 场景3：Blit / Copy 操作打断 Tile Pipeline
│   │   ├── Graphics.Blit 在 URP 中创建独立 Pass
│   │   ├── 大量后处理全屏 Blit → 多次 Tile Flush
│   │   └── 修复：用 RendererFeature 在同一 Tile 内做后处理
│   └── 场景4：Compute Shader 与 Graphics 的资源依赖
│       ├── CS 写入纹理 → Graphics 立刻读取 → Mali 不能确定 Tile 内数据一致性
│       ├── 触发保守的 Flush（等同于 Rollback）
│       └── 修复：插入合理的 Barrier 或用 Frame Kicker 延迟 CS
├── 诊断工具链
│   ├── Arm Streamline（最关键）
│   │   ├── Mali GPU Counter: L2_CACHE_HIT_RATE → FPR 时命中率骤降
│   │   ├── Mali GPU Counter: EXTERNAL_MEM_BANDWIDTH → FPR 时带宽翻倍
│   │   ├── Mali GPU Counter: FRAGMENT_ACTIVE_CYCLES → 不变（误导！）
│   │   └── 对比 Adreno GPU Profiler 的差异
│   ├── Mali Offline Compiler
│   │   ├── 编译 Shader 看 Register Pressure
│   │   └── 高 Register → GPU 并行度下降 → 间接加剧 Tile Memory 压力
│   └── Unity Frame Debugger + Mali Integration
│       ├── 看 Pass 数量和依赖关系
│       └── 识别哪些 Blit 是多余的
└── 优化策略优先级
    ├── P0：减少 Render Target 数量 / 格式（最大收益）
    ├── P1：合并相邻后处理 Pass（URP 中用 Custom Renderer Feature）
    ├── P2：降低后处理分辨率（半分辨率 Blur / SSAO）
    ├── P3：Shader Register Pressure 优化（减少临时变量）
    └── P4：调整 Tile 大小（部分设备支持运行时配置）
```

#### 代码实现

**诊断脚本：自动统计 URP Pass 数量和 Blit 操作**

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using System.Collections.Generic;
using System.Text;

public class RenderPassAnalyzer : MonoBehaviour
{
    [Header("诊断输出")]
    [SerializeField] private bool analyzeOnStart = false;
    
    private void Start()
    {
        if (analyzeOnStart) AnalyzeRenderPasses();
    }
    
    [ContextMenu("分析 Render Pass")]
    public void AnalyzeRenderPasses()
    {
        var urpAsset = GraphicsSettings.currentRenderPipeline as UniversalRenderPipelineAsset;
        if (urpAsset == null)
        {
            Debug.LogError("[FPR诊断] 当前不是 URP 管线");
            return;
        }
        
        var sb = new StringBuilder();
        sb.AppendLine("=== URP Render Pass 分析报告 ===\n");
        
        // 1. Renderer Feature 分析
        var rendererData = urpAsset.scriptableRenderer as UniversalRenderer;
        sb.AppendLine("--- Renderer Features ---");
        
        // 遍历所有 Renderer Feature
        var rendererFeatures = typeof(ScriptableRenderer).GetField("m_RendererFeatures",
            System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);
        
        sb.AppendLine($"Renderer Feature 数量影响 Pass 数量，每个 Blit 类 Feature 都可能触发 FPR\n");
        
        // 2. 后处理 Pass 预估
        sb.AppendLine("--- 后处理 Pass 估算 ---");
        var volumeStack = VolumeManager.instance.stack;
        
        // 统计启用的后处理效果
        int postProcessPassEstimate = 0;
        
        if (volumeStack.GetComponent<Bloom>() != null && 
            volumeStack.GetComponent<Bloom>().IsActive())
        {
            sb.AppendLine("[Bloom] 预估 2 Pass（降采样 + 合成）= 可能 2 次 Tile Flush");
            postProcessPassEstimate += 2;
        }
        
        if (volumeStack.GetComponent<ColorAdjustments>() != null &&
            volumeStack.GetComponent<ColorAdjustments>().IsActive())
        {
            sb.AppendLine("[ColorAdjustments] 预估 1 Pass（URP 合并到 Uber Post）");
            postProcessPassEstimate += 0; // 合并到 Uber
        }
        
        if (volumeStack.GetComponent<DepthOfField>() != null &&
            volumeStack.GetComponent<DepthOfField>().IsActive())
        {
            sb.AppendLine("[DoF] 预估 2 Pass（CoC + 散景模糊）= 2 次 Tile Flush");
            postProcessPassEstimate += 2;
        }
        
        sb.AppendLine($"\n后处理预估额外 Pass: {postProcessPassEstimate}");
        sb.AppendLine($"FPR 风险: {(postProcessPassEstimate > 4 ? "🔴 高" : 
                       postProcessPassEstimate > 2 ? "🟡 中" : "🟢 低")}");
        
        // 3. Render Target 内存估算
        int width = Screen.width;
        int height = Screen.height;
        int tileSize = 64 * 64; // Mali 常见 Tile Size（可配置）
        int tilesPerFrame = Mathf.CeilToInt(width / 64f) * Mathf.CeilToInt(height / 64f);
        
        sb.AppendLine($"\n--- Tile Memory 估算 ---");
        sb.AppendLine($"屏幕: {width}x{height} = {tilesPerFrame} tiles (64x64)");
        sb.AppendLine($"每 Tile 像素: {tileSize}");
        sb.AppendLine($"每像素 Render Target 内存:");
        sb.AppendLine($"  Color (RGBA8):    4 bytes");
        sb.AppendLine($"  Depth (D24S8):    4 bytes");
        sb.AppendLine($"  HDR (RGBA16F):    8 bytes ← 如果开 HDR，Tile Memory 需求翻倍");
        sb.AppendLine($"  MRT GBuffer:      16-32 bytes ← 延迟渲染风险极高");
        
        int bytesPerPixelHDR = 8 + 4 + 8; // Color HDR + Depth + Bloom RT
        int tileMemoryHDR = tileSize * bytesPerPixelHDR;
        sb.AppendLine($"\nHDR 场景每 Tile 需求: {tileMemoryHDR / 1024} KB");
        sb.AppendLine($"Mali-G715 Tile Memory: ~1-2 MB per Shader Core");
        sb.AppendLine($"FPR 风险: {(tileMemoryHDR > 512 * 1024 ? "🔴 高 - 建议 HDR 降半分辨率" : "🟢 低")}");
        
        Debug.Log(sb.ToString());
    }
    
    [ContextMenu("输出 Mali Counter 对比表")]
    public void PrintMaliCounterGuide()
    {
        var sb = new StringBuilder();
        sb.AppendLine("=== Mali GPU Counter 速查表（FPR 诊断）===");
        sb.AppendLine();
        sb.AppendLine("| Counter | 正常值 | FPR 发生时 | 含义 |");
        sb.AppendLine("|---------|--------|------------|------|");
        sb.AppendLine("| L2_CACHE_HIT_RATE | >80% | <40% | Tile 数据在 L2 命中，回滚后未命中 |");
        sb.AppendLine("| EXTERNAL_MEM_READ_BEATS | 基线 | 2-3x 飙升 | 从系统内存重新读 Tile |");
        sb.AppendLine("| EXTERNAL_MEM_WRITE_BEATS | 基线 | 2-3x 飙升 | 先写回再读回 |");
        sb.AppendLine("| FRAGMENT_ACTIVE_CYCLES | 稳定 | 可能不变 | ⚠️ 误导指标！ |");
        sb.AppendLine("| TILE_MEM_WRITE_BEATS | 低 | 极高 | Tile 内写入正常，但回滚导致额外写 |");
        sb.AppendLine("| GPU_UTIL | ~60-80% | ~95%+ | GPU 在搬运数据而非渲染 |");
        sb.AppendLine();
        sb.AppendLine("关键判据：L2_CACHE_HIT_RATE 骤降 + EXTERNAL_MEM 双向飙升 = FPR 发生");
        
        Debug.Log(sb.ToString());
    }
}
#endif
```

**URP 后处理 Pass 合并示例（减少 Tile Flush）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

/// <summary>
/// 将多个后处理合并到单个 Fullscreen Pass，
/// 减少 Mali GPU 上 Tile Flush 次数
/// </summary>
public class MergedPostProcessFeature : ScriptableRendererFeature
{
    private MergedPostProcessPass mergedPass;
    
    public override void Create()
    {
        mergedPass = new MergedPostProcessPass
        {
            renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing
        };
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer, 
        ref RenderingData renderingData)
    {
        if (renderingData.cameraData.cameraType == CameraType.Game)
        {
            renderer.EnqueuePass(mergedPass);
        }
    }
    
    protected override void Dispose(bool disposing)
    {
        mergedPass?.Dispose();
    }
}

public class MergedPostProcessPass : ScriptableRenderPass
{
    private Material mergedMaterial;
    private RTHandle sourceTex;
    private RTHandle tempTex;
    
    private static readonly int BloomTexID = Shader.PropertyToID("_BloomTex");
    private static readonly int DoFTexID = Shader.PropertyToID("_DoFTex");
    
    public MergedPostProcessPass()
    {
        // 一个 Shader 中合并 Bloom + DoF + Color Grading
        var shader = Shader.Find("Hidden/Custom/MergedPostProcess");
        mergedMaterial = CoreUtils.CreateEngineMaterial(shader);
    }
    
    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        // 后处理用半分辨率 → Tile Memory 压力减半
        desc.width /= 2;
        desc.height /= 2;
        
        RenderingUtils.ReAllocateIfNeeded(ref tempTex, desc);
        sourceTex = renderingData.cameraData.renderer.cameraColorTargetHandle;
    }
    
    public override void Execute(ScriptableRenderContext context, 
        ref RenderingData renderingData)
    {
        var cmd = CommandBufferPool.Get("MergedPostProcess");
        
        // 一次性采样 Bloom 和 DoF（如果可以合并到同一 Shader）
        // 这比 URP 默认的分离 Pass 少 3-4 次 Tile Flush
        Blitter.BlitCameraTexture(cmd, sourceTex, tempTex, mergedMaterial, 0);
        Blitter.BlitCameraTexture(cmd, tempTex, sourceTex);
        
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
    
    public void Dispose()
    {
        tempTex?.Release();
        if (mergedMaterial != null) CoreUtils.Destroy(mergedMaterial);
    }
}
```

### ⚡ 实战经验

1. **"GPU 利用率很高"不等于"渲染效率高"**：FPR 发生时 GPU 利用率确实飙到 95%+，但它在搬运数据而不是渲染像素。这是 Mali 上最常见的误判——看 GPU 利用率高就以为没优化空间了

2. **HDR 在 Mali 上是 FPR 高危因素**：RGBA16F 的 Color RT 比 RGBA8 多一倍带宽，加上 Bloom 需要的降采样 RT，4K 分辨率下单 Tile 的内存需求轻松超过 Tile Memory。实战中很多移动端项目关闭 HDR 或只用 RGBA111110 格式就是这原因

3. **Adreno 和 Mali 的 Tile 大小不同**：Adreno 的 FlexRender 可动态调整 Tile 策略，Mali 的 Tile Size 相对固定。同一个场景在 Adreno 上没问题但在 Mali 上 FPR——这是因为两个架构的 Tile Memory 容量和调度策略不同

4. **URP 默认后处理是 Pass 分离的**：Bloom → DoF → Color Grading 各自一个 Blit Pass。在 Mali 上这意味着 6+ 次 Tile Flush。用自定义 Renderer Feature 合并后处理可以减到 1-2 次

5. **Compute Shader 的 Barrier 陷阱**：在 URP 中用 Compute Shader 做后处理（如 CACAO、GTAO）时，CS 和 Graphics 之间的资源屏障在 Mali 上会被保守处理为一个隐式的 Tile Flush。如果可能，尽量在 Fragment Shader 中做同样的事

### 🎯 能力体检清单

| 知识点 | 自检问题 | 盲区信号 |
|--------|----------|----------|
| TBR/TBDR 基本原理 | 能画出 Mali GPU 一个 Tile 的完整生命周期吗？ | ❌ 说不清 Tile Memory 和系统内存的关系 → 无法理解 FPR |
| FPR 触发条件 | 能列举 3 个触发 FPR 的具体渲染操作吗？ | ❌ 只知道"性能不好"→ 定位不到根因 |
| Arm Mobile Studio | 用过 Streamline 吗？知道哪些 Counter 指向 FPR？ | ❌ 没用过 → 只能靠猜，无法给数据支撑 |
| URP Pass 结构 | URP 默认 Bloom 消耗几个 Pass？为什么？ | ❌ 不清楚 → 不知道怎么合并 |
| 跨 GPU 适配 | 同一场景在 Adreno 正常、Mali 掉帧，你的排查思路？ | ❌ 直接说"Mali 不行" → 缺乏架构级理解 |

### 🔗 相关问题

- [Adreno Tile-Based 带宽优化](../optimization/adreno-tile-based-bandwidth.md) — 另一大移动 GPU 架构的带宽优化
- [移动端 GPU 性能分析工具链](../optimization/mobile-gpu-profiling-toolchain.md) — Arm/Qualcomm 工具链全景
- [GPU 带宽优化策略](../optimization/gpu-bandwidth-optimization.md) — 带宽优化的通用方法论
