---
title: "CPU/GPU 同步停顿：为什么你的 GPU 利用率只有 30%？"
category: "optimization"
level: 3
tags: ["CPU-GPU同步", "Pipeline Stall", "Frame Timing", "性能分析", "RenderDoc"]
hint: "GPU 利用率低不一定是 Shader 慢，可能是 CPU 端在等 GPU（或反过来），Frame Quest 和 RenderDoc 能看到 pipeline bubble"
related: ["optimization/loading-stall-hitch-spike", "optimization/srp-batcher-break-cause", "optimization/mobile-gpu-profiling-toolchain"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们游戏的帧时间统计显示 GPU 时间只有 8ms，CPU 主线程也只有 6ms，理论上能跑 120FPS，但实际只跑 45FPS。帧时间曲线有周期性的 22ms 尖峰。你怎么排查？」

### ✅ 核心要点

1. **问题本质**：CPU 和 GPU 之间的命令提交与回读存在同步点（fence/stall），导致流水线气泡（pipeline bubble）
2. **典型来源**：`GraphicsSettings.WaitForGPU`、`ReadPixels`/`GetData` 回读、条件渲染查询、帧起始/结束 fence
3. **排查工具链**：Frame Timing Manager → RenderDoc Timeline → XCode GPU Capture → Snapdragon Profiler
4. **解决方向**：消除回读、异步查询、双缓冲/三缓冲、管线化 CPU 提交
5. **移动端特有**：TBDR 架构下 tile resolve 回读、VSync 锁定导致额外等待

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终现象：理论 120FPS → 实际 45FPS，帧时间有 22ms 尖峰
                ↑
倒推1：GPU 时间 8ms + CPU 时间 6ms = 14ms ≠ 22ms → 8ms 不见了
倒推2：消失的时间在哪？→ CPU 在等 GPU（或反过来）的 fence 信号
倒推3：谁在等？→ 排查所有 GPU 回读操作（ReadPixels, GetData, GetProperty, condition render）
倒推4：找到同步点 → 用异步替代（AsyncGPUReadback, fence with timeout）
倒推5：消除周期性尖峰 → 检查是否每 N 帧触发一次（GC、资源加载、Streaming）
```

#### 知识点拆解（倒推树）

```
CPU/GPU 同步停顿
├── 同步点来源（Stall Sources）
│   ├── 显式回读
│   │   ├── Texture2D.ReadPixels() → 强制 flush GPU pipeline
│   │   ├── ComputeBuffer.GetData() → CPU 等 GPU 完成
│   │   ├── AsyncGPUReadback.Request() → 异步但仍有 frame delay
│   │   └── RenderTexture → Texture2D → 截图/反射探针
│   ├── 隐式同步
│   │   ├── WaitForEndOfFrame（FrameTimingManager 统计）
│   │   ├── 条件渲染 Occlusion Query 回读
│   │   ├── TransformFeedback / Compute 写入后立刻 CPU 读取
│   │   └── 渲染目标同时被 CPU 读写（共享资源）
│   └── 帧间依赖
│       ├── 上一帧 GPU 未完成 → 当前帧 CPU 提交被阻塞
│       ├── VSync 锁定：GPU Present 等 VBlank
│       └── SwapChain 缓冲深度不足（双缓冲 vs 三缓冲）
├── 排查工具链
│   ├── Unity Frame Timing Manager（2019.4+）
│   │   ├── FrameTiming.cpuFrameTime / gpuFrameTime / cpuMainThreadTime
│   │   ├── cpuRenderThreadTime → 渲染线程是否在等
│   │   └── FrameTimingManager.CaptureFrameTimings()
│   ├── RenderDoc → Timeline View 看 GPU pipeline bubble
│   ├── Xcode GPU Capture（iOS）→ 看 Metal Encoder 间隔
│   ├── Snapdragon Profiler（Android/Adreno）→ GPU Timeline
│   └── Mali Streamline / Mali Offline Compiler（Mali GPU）
├── TBDR 架构特有问题
│   ├── Tile Resolve 回读：GBuffer/Depth 从 tile 内存回到主存
│   ├── Store Action：不必要的 StoreOp.Store 导致额外带宽
│   ├── Deferred Rendering 下 tile 内存压力更大
│   └── Vulkan/Metal 的 Render Pass 兼容性影响 tile 合并
├── 解决方案
│   ├── 消除回读
│   │   ├── 用 Compute Shader 在 GPU 端处理（避免 CPU 回读）
│   │   ├── 用 AsyncGPUReadback 替代同步 ReadPixels
│   │   └── Occlusion Culling 用 GPU-driven（Compute）替代 CPU 回读
│   ├── 管线化
│   │   ├── 多帧延迟回读（当前帧读 N-2 帧的结果）
│   │   ├── Triple Buffering（减少 VSync 等待）
│   │   └── 命令缓冲预录制（Vulkan/Metal secondary command buffer）
│   ├── 渲染线程优化
│   │   ├── 减少 SetPass / Material property 设置（SRP Batcher）
│   │   ├── GPU Instancing / Indirect Draw 减少 Draw Call 数量
│   │   └── 合并小 Pass → 减少 pipeline flush
│   └── 资源加载
│       ├── 异步加载 + 分帧（避免单帧卡顿）
│       ├── Texture Streaming 分帧上传
│       └── AssetBundle 异步实例化
└── Frame Budget 分配（移动端 16.67ms @ 60FPS）
    ├── CPU Main Thread: ≤ 6ms（逻辑 + 物理 + AI）
    ├── CPU Render Thread: ≤ 4ms（提交 Draw Call）
    ├── GPU: ≤ 10ms（渲染全部 Pass）
    └── 安全余量: 2-3ms（GC、IO、同步）
```

#### 代码实现

**Frame Timing Manager 诊断脚本：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;

public class FrameTimingDebugger : MonoBehaviour
{
    private FrameTiming[] _timings = new FrameTiming[1];
    private float _timer = 0f;

    void Update()
    {
        _timer += Time.unscaledDeltaTime;
        if (_timer < 0.5f) return; // 每 0.5 秒采样一次
        _timer = 0f;

        uint count;
        FrameTimingManager.GetTimings(1, _timings, out count);
        if (count == 0) return;

        var t = _timings[0];
        float cpuFrame = (float)t.cpuFrameTime;
        float gpuFrame = (float)t.gpuFrameTime;
        float cpuMain = (float)t.cpuMainThreadTime;
        float cpuRender = (float)t.cpuRenderThreadTime;

        // 计算 Stall 时间
        // 理论帧时间 = max(CPU, GPU)，实际帧时间 = cpuFrame
        float theoreticalFrame = Mathf.Max(cpuMain + cpuRender, gpuFrame);
        float stallTime = cpuFrame - theoreticalFrame;

        Debug.Log($"[FrameTiming] CPU Frame: {cpuFrame:F1}ms | GPU: {gpuFrame:F1}ms"
                + $" | CPU Main: {cpuMain:F1}ms | CPU Render: {cpuRender:F1}ms"
                + $" | Stall: {stallTime:F1}ms");

        if (stallTime > 3f)
        {
            Debug.LogWarning($"⚠️ CPU/GPU Stall detected: {stallTime:F1}ms wasted!"
                           + " Check for GPU readback or fence sync.");
        }
    }
}
```

**AsyncGPUReadback 替代同步回读：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;

public class SafeGPUReadback
{
    // ❌ 错误：同步回读，阻塞 GPU pipeline
    public Color[] ReadPixelsSync(RenderTexture rt)
    {
        var tex = new Texture2D(rt.width, rt.height, TextureFormat.RGBA32, false);
        RenderTexture.active = rt;
        tex.ReadPixels(new Rect(0, 0, rt.width, rt.height), 0, 0);
        tex.Apply();
        return tex.GetPixels();
    }

    // ✅ 正确：异步回读，不阻塞
    public void ReadPixelsAsync(RenderTexture rt, System.Action<Color[]> callback)
    {
        AsyncGPUReadback.Request(rt, 0, TextureFormat.RGBA32, (request) =>
        {
            if (request.hasError)
            {
                Debug.LogError("GPU Readback failed!");
                return;
            }
            var data = request.GetData<Color>();
            var colors = data.ToArray();
            callback?.Invoke(colors);
        });
    }
}
```

**常见同步点排查清单（RenderDoc 验证）：**

| 操作 | 是否同步 | 替代方案 | 排查方法 |
|------|----------|----------|----------|
| `Texture2D.ReadPixels` | ✅ 强同步 | `AsyncGPUReadback.Request` | RenderDoc Timeline 看气泡 |
| `ComputeBuffer.GetData` | ✅ 强同步 | `AsyncGPUReadback.Request(buffer)` | Xcode GPU Capture |
| `RenderTexture.active = rt` + CPU 读取 | ✅ 强同步 | GPU Compute → Texture → Async Readback | Frame Timing stall > 3ms |
| `Camera.Render()` 手动调用 | ⚠️ 可能同步 | 考虑是否必要 | Profiler CPU 模块 |
| `QualitySettings.vSyncCount` | ⚠️ VSync 等 | 不设 VSync + Application.targetFrameRate | Frame Timing gpuFrameTime ≈ 16.67 |
| `WaitForEndOfFrame` 协程 | ⚠️ 等 GPU | 避免在此做重逻辑 | Profiler 标记 |

### ⚡ 实战经验

- **真实案例**：某项目截图功能用 `ReadPixels` 导致每次截图卡 18ms——改用 `AsyncGPUReadback` 后完全消除卡顿，代价是截图延迟 2 帧显示
- **周期性尖峰排查**：22ms 尖峰每 5 帧出现一次 → 排查发现是反射探针（Reflection Probe）每 5 帧刷新一次，用 `Time.slicing` 改为分帧刷新后解决
- **移动端 VSync 陷阱**：`QualitySettings.vSyncCount = 1` 在 60Hz 屏幕上强制锁 16.67ms。如果 GPU 只需 10ms，那 6.67ms 就被浪费了——用 `Application.targetFrameRate = 120`（如果支持）配合 vSync 调整
- **渲染线程是瓶颈**：当 `cpuRenderThreadTime > cpuMainThreadTime` 时，说明渲染线程在批量提交命令——SRP Batcher、GPU Instancing、Indirect Draw 是直接解药
- **Pipeline Bubble 在 Mali 上的表现**：Mali Frame Builder 之间的空闲周期——用 Mali Streamline 的 GPU Activity 图能看到明显的空白段

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 stall 时间怎么计算 | Frame Timing API | 学 FrameTimingManager 各字段含义 |
| 不知道哪些操作会触发同步 | GPU 回读机制 | 学 GPU Pipeline + Fence/Event 原理 |
| 排查不出 stall 来源 | Profiler 工具使用 | 学 RenderDoc Timeline + Unity Profiler |
| 改了 AsyncGPUReadback 但数据延迟 | 异步回读的帧延迟 | 理解 N-2 帧延迟 + 双缓冲设计 |
| 移动端依然有周期性卡顿 | TBDR tile resolve | 学移动端 GPU 架构 + Render Pass 优化 |

### 🔗 相关问题

- AsyncGPUReadback 的帧延迟（通常 2-3 帧）在逻辑上怎么处理？（提示：版本号 / 帧号对齐）
- Vulkan 的 vkFence 和 Metal 的 shared event 有什么区别？
- 如何用 Compute Shader 实现完全 GPU-driven 的可见性剔除，彻底消除 CPU 回读？
