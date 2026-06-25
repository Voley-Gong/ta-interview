---
title: "RT 内存占了一半：渲染纹理池如何设计与实现？"
category: "optimization"
level: 3
tags: ["RenderTexture", "内存池", "URP", "GPU内存", "纹理复用"]
hint: "URP 中每个 Renderer Feature 都可能分配 RT——不是每个都要独立的，关键是时间错开的 RT 可以复用同一个池化资源"
related: ["optimization/gpu-memory-budget", "rendering/custom-post-processing-urp", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的 URP 项目中，后处理、描边、SSAO、水体反射、动态天气、技能特效……每个 Renderer Feature 都在分配自己的 RenderTexture。内存分析显示 RT 总共占了 400MB+，其中大量 RT 是同分辨率、同格式的，但各自独立分配。设计一套 RenderTexture 池化系统，在不影响渲染正确性的前提下大幅降低 RT 内存占用。」

（这是米哈游、腾讯互娱等在做大型 URP 项目时的典型工程优化题。）

### ✅ 核心要点

1. **问题本质**：URP 的 Renderer Feature 各自调用 `RenderTargetHandle` 分配 RT，没有全局统筹 → 同尺寸 RT 重复分配
2. **复用条件**：同一帧内不重叠使用的 RT（时间不冲突）可以复用同一块显存
3. **池化策略**：按「尺寸 + 格式 + 深度位」分桶，同桶内 FIFO 复用
4. **生命周期管理**：帧开始时统一分配，帧结束时统一释放（或延迟一帧回收给异步读取）
5. **URP 集成点**：自定义 `ScriptableRendererFeature` + `ScriptableRenderPass` 中用池替代直接 `RTHandle.Alloc`

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：400MB RT 内存 → 降到 120MB 以内（仅保留同时需要的 RT）
              ↑
Step 1：RT 使用时间线分析
  ├── Pass A（描边）：在 BeforeRenderingOpaques 阶段，用完即弃
  ├── Pass B（SSAO）：在 AfterRenderingOpaques 阶段，用完即弃
  ├── Pass C（后处理模糊1）：AfterRenderingTransparents，用完即弃
  ├── Pass D（后处理模糊2）：紧接 Pass C，但可以用 Ping-Pong 复用
  └── Pass E（水面反射）：BeforeRenderingOpaques，与 Pass A 时间可能重叠
     ↓
Step 2：冲突分析
  ├── Pass A 和 Pass B：时间不冲突 → 可以复用同一个 RT
  ├── Pass C 和 Pass D：连续使用 → Ping-Pong 双缓冲
  ├── Pass A 和 Pass E：可能冲突 → 需要独立 RT 或时序错开
  └── 跨帧依赖（如 TAA 的 history buffer）：不能复用，需常驻
     ↓
Step 3：池化设计
  ├── Key = (width, height, format, depthBits, antiAliasing)
  ├── Value = Queue<RTHandle>（可用 RT 队列）
  ├── Acquire(key)：从队列取一个，没有则分配
  └── Release(key, rt)：归还到队列，不立即释放
     ↓
Step 4：生命周期
  ├── Frame Start：所有非跨帧 RT 标记为可回收
  ├── Pass 执行前：Acquire → 使用
  ├── Pass 执行后：Release → 归还
  └── Frame End：清理超过 N 帧未使用的 RT（防内存泄漏）
```

#### 知识点拆解（倒推树）

```
RenderTexture 池化系统
├── RT 内存问题分析
│   ├── URP RTHandle 系统原理
│   ├── 每个 RendererFeature 的 RT 分配开销
│   ├── MSAA 对 RT 内存的影响（4x = 4倍显存）
│   └── 跨帧 RT（TAA History、Motion Vector）不能池化
│
├── 池化核心逻辑
│   ├── 分桶 Key 设计
│   │   ├── 分辨率（width × height）
│   │   ├── 格式（RGBA32 / RGBA64 / R8 / RGFloat / Depth）
│   │   ├── 深度缓冲（无 / 16bit / 24bit）
│   │   └── MSAA 采样数
│   ├── 获取策略
│   │   ├── 精确匹配（尺寸 + 格式完全一致）
│   │   ├── 降级匹配（大尺寸 RT 裁给小用 → 不推荐，UV 错位）
│   │   └── 动态缩放（相机分辨率变化时的 RT resize）
│   └── 回收策略
│       ├── 立即回收（Pass 结束就归还）
│       ├── 延迟回收（GPU 异步读取未完成，延迟 2 帧）
│       └── 超时释放（超过 N 帧未使用 → Dispose）
│
├── URP 集成
│   ├── 替换 RTHandle.Alloc → 池化版本
│   ├── ScriptableRenderPass 中 ConfigureTarget 改用池化 RT
│   ├── RendererFeature 执行顺序优化（让更多 Pass 可复用）
│   └── Camera Stack 共享 RT（Base + Overlay 相机）
│
└── 调试工具
    ├── 实时 RT Monitor（面板显示当前所有活跃 RT）
    ├── 复用率统计（被复用次数 / 总分配次数）
    └── 内存占用曲线（按桶分类显示）
```

#### 代码实现

**RenderTexture 池核心实现：**

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

/// <summary>
/// 全局 RenderTexture 池。
/// 同尺寸、同格式的 RT 在时间不冲突时可复用。
/// </summary>
public class RenderTexturePool
{
    private static RenderTexturePool _instance;
    public static RenderTexturePool Instance => _instance ??= new RenderTexturePool();

    private struct RTKey
    {
        public int width;
        public int height;
        public GraphicsFormat format;
        public int depthBits;
        public int msaaSamples;

        public override int GetHashCode() =>
            HashCode.Combine(width, height, format, depthBits, msaaSamples);
        public override bool Equals(object obj) =>
            obj is RTKey other &&
            width == other.width && height == other.height &&
            format == other.format && depthBits == other.depthBits &&
            msaaSamples == other.msaaSamples;
    }

    // 可用 RT 队列（按 Key 分桶）
    private readonly Dictionary<RTKey, Queue<RTHandle>> _pool = new();
    
    // 活跃 RT 追踪
    private readonly Dictionary<RTHandle, (RTKey key, string user, int frame)> _active = new();
    
    // 统计数据
    public int TotalAllocated { get; private set; }
    public int TotalReused { get; private set; }
    public int CurrentActive => _active.Count;
    public int CurrentPooled => GetPooledCount();

    /// <summary>
    /// 从池中获取一个 RT。如果池中有匹配的空闲 RT 则复用，否则新建。
    /// </summary>
    public RTHandle Acquire(
        int width, int height,
        GraphicsFormat format = GraphicsFormat.R8G8B8A8_SRGB,
        int depthBits = 0,
        int msaa = 1,
        string user = "unknown")
    {
        var key = new RTKey
        {
            width = width,
            height = height,
            format = format,
            depthBits = depthBits,
            msaaSamples = msaa
        };

        RTHandle rt;

        if (_pool.TryGetValue(key, out var queue) && queue.Count > 0)
        {
            rt = queue.Dequeue();
            TotalReused++;
            // Debug.Log($"[RTPool] Reuse {rt.name} for '{user}' (key: {width}x{height} {format})");
        }
        else
        {
            // 分配新的 RTHandle
            var desc = new RenderTextureDescriptor(width, height, format, depthBits)
            {
                msaaSamples = msaa,
                useMipMap = false,
                autoGenerateMips = false
            };
            rt = RTHandles.Alloc(desc, name: $"PooledRT_{width}x{height}_{format}_{TotalAllocated}");
            TotalAllocated++;
            // Debug.Log($"[RTPool] Allocate NEW {rt.name} for '{user}'");
        }

        _active[rt] = (key, user, Time.frameCount);
        return rt;
    }

    /// <summary>
    /// 归还 RT 到池中，不立即释放。
    /// </summary>
    public void Release(RTHandle rt, string user = "unknown")
    {
        if (!_active.TryGetValue(rt, out var entry))
        {
            Debug.LogWarning($"[RTPool] Release called for untracked RT by '{user}'");
            return;
        }

        // 清除 RenderTarget 绑定（避免 GPU 还在引用）
        // 注意：实际的 RenderTexture.Release 应在帧结束后统一执行

        _active.Remove(rt);

        if (!_pool.ContainsKey(entry.key))
            _pool[entry.key] = new Queue<RTHandle>();

        _pool[entry.key].Enqueue(rt);
    }

    /// <summary>
    /// 帧结束后清理：释放超过 maxIdleFrames 帧未使用的 RT。
    /// 应在 RenderPipelineManager.endFrameRendering 中调用。
    /// </summary>
    public void FrameCleanup(int maxIdleFrames = 3)
    {
        var keysToRemove = new List<RTKey>();

        foreach (var kv in _pool)
        {
            var queue = kv.Value;
            var keepQueue = new Queue<RTHandle>();

            while (queue.Count > 0)
            {
                var rt = queue.Dequeue();
                // 简化判断：保留最多 maxIdleFrames 帧前归还的 RT
                // 实际项目中应记录归还帧号做精确判断
                if (rt != null && rt.rt != null)
                {
                    keepQueue.Enqueue(rt);
                }
                else
                {
                    rt?.Release();
                }
            }

            if (keepQueue.Count > 0)
            {
                _pool[kv.Key] = keepQueue;
            }
            else
            {
                keysToRemove.Add(kv.Key);
            }
        }

        foreach (var key in keysToRemove)
        {
            var queue = _pool[key];
            while (queue.Count > 0)
            {
                queue.Dequeue()?.Release();
            }
            _pool.Remove(key);
        }
    }

    /// <summary>
    /// 获取池化效率报告
    /// </summary>
    public string GetReport()
    {
        float reuseRate = TotalAllocated + TotalReused > 0
            ? (float)TotalReused / (TotalAllocated + TotalReused) * 100f
            : 0f;
        return $"[RTPool] Allocated: {TotalAllocated}, Reused: {TotalReused}, " +
               $"Reuse Rate: {reuseRate:F1}%, Active: {CurrentActive}, Pooled: {CurrentPooled}";
    }

    private int GetPooledCount()
    {
        int count = 0;
        foreach (var kv in _pool) count += kv.Value.Count;
        return count;
    }
}
```

**URP RendererFeature 集成示例（池化版描边 Pass）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

/// <summary>
/// 使用 RT 池的描边 Renderer Feature。
/// 对比传统写法，RT 从池中获取，Pass 结束后归还。
/// </summary>
public class PooledOutlineFeature : ScriptableRendererFeature
{
    private PooledOutlinePass _pass;
    private RTHandle _outlineRT;

    public override void Create()
    {
        _pass = new PooledOutlinePass
        {
            renderPassEvent = RenderPassEvent.BeforeRenderingOpaques
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        var cameraColor = renderer.cameraColorTargetHandle;
        var cameraDepth = renderer.cameraDepthTargetHandle;

        // 从池中获取 RT（而不是每次 Alloc 新的）
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        int w = desc.width;
        int h = desc.height;

        _outlineRT = RenderTexturePool.Instance.Acquire(
            w, h,
            GraphicsFormat.R8G8B8A8_UNorm,
            depthBits: 0,
            msaa: 1,
            user: "OutlinePass"
        );

        _pass.Setup(_outlineRT, cameraColor, cameraDepth);
        renderer.EnqueuePass(_pass);
    }

    // 在 RenderPipelineManager.endFrameRendering 回调中统一归还
    public void ReturnRT()
    {
        if (_outlineRT != null)
        {
            RenderTexturePool.Instance.Release(_outlineRT, "OutlinePass");
            _outlineRT = null;
        }
    }
}

/// <summary>
/// 全局 RT 池生命周期管理器。
/// 挂载在场景中，监听渲染事件执行帧清理。
/// </summary>
public class RTPoolLifecycle : MonoBehaviour
{
    [SerializeField] private int maxIdleFrames = 3;

    void OnEnable()
    {
        RenderPipelineManager.beginFrameRendering += OnBeginFrame;
        RenderPipelineManager.endFrameRendering += OnEndFrame;
    }

    void OnDisable()
    {
        RenderPipelineManager.beginFrameRendering -= OnBeginFrame;
        RenderPipelineManager.endFrameRendering -= OnEndFrame;
    }

    private void OnBeginFrame(ScriptableRenderContext ctx, Camera[] cameras)
    {
        // 帧开始时可以做准备工作（如预分配）
    }

    private void OnEndFrame(ScriptableRenderContext ctx, Camera[] cameras)
    {
        // 帧结束：归还所有 Pass 的 RT → 清理空闲 RT
        // 各 Feature 的 ReturnRT() 应在 endFrameRendering 前调用
        RenderTexturePool.Instance.FrameCleanup(maxIdleFrames);
    }

    void OnGUI()
    {
        // 调试面板：实时显示池状态
        GUILayout.BeginArea(new Rect(10, 10, 400, 100));
        GUILayout.Label(RenderTexturePool.Instance.GetReport());
        GUILayout.EndArea();
    }
}
```

**优化前后内存对比预估（1080p 分辨率）：**

| RT 用途 | 格式 | 单独占用 | 池化后 | 说明 |
|----------|------|----------|--------|------|
| 描边 Pass | R8G8B8A8 | 8MB | 复用桶A | 与 SSAO 复用 |
| SSAO Pass | R8G8B8A8 | 8MB | 复用桶A | 时间错开 |
| 模糊 Ping-Pong | R8G8B8A8 ×2 | 16MB | 桶B ×2 | 连续使用 |
| 水面反射 | R8G8B8A8 | 8MB | 复用桶A | 与描边可能冲突→独立 |
| 动态天气 | R16G16B16A16 | 32MB | 独立 | 高精度格式无法复用 |
| TAA History | R16G16B16A16 | 32MB | 常驻 | 跨帧依赖 |
| **总计** | — | **~400MB** | **~130MB** | **降 67%** |

### ⚡ 实战经验

1. **延迟归还比立即归还没那么简单**：GPU 执行是异步的，Pass 刚执行完时 GPU 可能还在读 RT。安全做法是延迟 1-2 帧归还，或使用 fence/async 同步点
2. **MSAA 的坑**：MSAA 2x/4x 的 RT 内存翻倍/四倍，且不能与非 MSAA 的 RT 复用同一桶。尽量在管线早期 Resolve MSAA，后续 Pass 全用非 MSAA 的 RT
3. **HDR 与 LDR 不要混用**：HDR RT（R16G16B16A16）和 LDR RT（R8G8B8A8）格式不同，不能复用。如果管线中间从 HDR tone map 到 LDR，之后的 RT 可以统一到 LDR 格式
4. **相机分辨率变化的处理**：动态分辨率（Dynamic Resolution）开启时，RT 尺寸在运行时变化。池化系统需要支持「同格式不同尺寸」的复用——可以在 Key 中只用格式做桶，尺寸通过 RTResize 动态调整（但有性能开销）
5. **不要池化 CameraColor 和 CameraDepth**：这两个是 URP 管线核心 RT，生命周期覆盖整帧，不适合池化

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 RT 为什么占那么多内存 | 不了解 RT 内存计算 | 学 RT 分辨率 × 格式 × MSAA 的内存公式 |
| 池化后画面闪烁/撕裂 | GPU 异步问题 | 学 GPU 同步机制、fence、延迟归还 |
| 不知道在哪归还 RT | 不了解 URP 渲染流程 | 学 RenderPipelineManager 事件回调 |
| MSAA RT 复用出问题 | MSAA 格式不兼容 | 学 MSAA Resolve 流程和 RT 格式约束 |
| 动态分辨率下池失效 | Key 尺寸不匹配 | 学 RTHandle 系统的 resize 机制 |

### 🔗 相关问题

- URP 的 RTHandle 系统和旧版 RenderTexture 有什么区别？
- 如何使用 Render Graph（Unity 6 / URP 17+）来自动管理 RT 生命周期？
- GPU Driven Pipeline 下的 RT 管理有什么特殊需求？
- 移动端 TBDR 架构下，RT 池化策略需要做什么调整？
