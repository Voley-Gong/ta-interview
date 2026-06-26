---
title: "手游 GPU Profiling 实战：帧率不稳但不知道瓶颈在哪，你的排查路径是什么？"
category: "optimization"
level: 3
tags: ["GPU Profiling", "Mali", "Adreno", "RenderDoc", "Arm Mobile Studio", "Snapdragon Profiler", "性能分析", "帧率优化"]
hint: "先分 CPU/GPU 瓶颈 → 再用对应工具链定位 → 最后看是顶点/像素/带宽/纹理哪一层"
related: ["optimization/mobile-overheating-gpu-analysis", "optimization/vertex-bound-bottleneck", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 圠景描述

面试官说：「我们的游戏在高通 8 Gen 2 上跑 60fps，但每隔几秒就掉到 45fps 然后恢复，Graphics Settings 都调过了，Shader 也优化了，但查不到到底是什么导致的卡顿。你能用过的 GPU Profiling 工具来描述你的排查路径吗？从第一步到最终定位，假设你有 Android 设备和 PC。」

（追问：如果换成 Mali GPU 设备呢？工具链和排查思路有什么不同？）

### ✅ 核心要点

1. **先定性：CPU 瓶颈还是 GPU 瓶颈**——用 Unity Profiler / Unreal GPU Visualizer 做初步分离
2. **再定量：用芯片厂商专属工具**——Adreno 用 Snapdragon Profiler，Mali 用 Arm Mobile Studio（Streamline + Mali Offline Compiler）
3. **GPU 内部细分**——顶点处理 / 光栅化 / 像素着色 / 纹理采样 / 带宽，每一层有对应的 Profiling 指标
4. **Frame Capture 逐 Draw Call 分析**——RenderDoc 抓帧，逐 Pass 看 Overdraw / State Change / 贴图带宽
5. **尖刺分析（Spike Hunting）**——周期性掉帧很可能是 Asset 加载 / GC / 纹理上传，不是 GPU 渲染本身
6. **可复现性**——Profile 结果必须排除热节流干扰（手机降频会导致假阳性）

### 📖 深度展开

#### 解决思路（从现象倒推原因）

```
现象：60fps 周期性掉到 45fps，每几秒一次
                ↑
倒推1：是稳定掉帧还是周期性尖刺？
     → 「每几秒一次」说明是周期性 → 优先排查非 GPU 渲染因素
倒推2：排除热节流了吗？
     → 8Gen2 跑 60fps 持续 5 分钟后会降频 → 先用散热背夹锁定频率
倒推3：尖刺出现在 CPU 还是 GPU？
     → Unity Profiler 看 CPU main thread vs GPU timeline
     → 如果 CPU spike 但 GPU 正常 → 不是渲染问题 → 查 Asset Loading / GC
倒推4：如果确认是 GPU spike
     → Snapdragon Profiler 抓帧 → 看 GPU 占用时间分布
     → 哪个 Render Pass 时间暴增？
倒推5：Pass 级定位
     → 是 Draw Call 太多（Vertex Processing bound）？
     → 还是 Shader 太复杂（Fragment bound）？
     → 还是贴图太大（Bandwidth bound）？
倒推6：针对性优化
```

#### 知识点拆解（倒推树）

```
GPU Profiling 排查路径
├── 第一步：环境标准化（排除干扰）
│   ├── 散热控制（散热背夹 / 恒温环境，排除热节流）
│   ├── 电池状态（满电 + 充电状态测试，排除降频）
│   ├── 后台进程清理（杀掉无关后台 App）
│   ├── 分辨率锁定（固定渲染分辨率，排除 DSR 动态分辨率干扰）
│   └── 固定场景测试（用固定相机位置做可复现 Benchmark）
├── 第二步：CPU vs GPU 分离
│   ├── Unity：Window > Analysis > Profiler
│   │   ├── CPU Usage：看 Main Thread 和 Render Thread
│   │   ├── GPU Usage：看 GPU Time per Frame（需要启用 GPU Profiling）
│   │   ├── Memory：看 GC Alloc（GC spike 是常见掉帧元凶）
│   │   └── Rendering：看 Draw Call / SetPass / Triangles / Vertices
│   ├── Unreal：stat gpu / stat scenerendering
│   │   ├── Frame Time = CPU Time + GPU Time
│   │   ├── GPU Visualizer（Ctrl+Shift+,）看各 Pass 耗时
│   │   └── stat unit 看整体分解
│   └── 判定逻辑
│       ├── GPU Time > CPU Time → GPU bound → 进入第三步
│       ├── CPU Time > GPU Time 且 Render Thread 高 → 渲染提交瓶颈 → DC 优化
│       └── CPU Main Thread spike 但 GPU 正常 → 逻辑/GC/IO 问题 → 非渲染
├── 第三步：GPU 厂商工具链
│   ├── Adreno（高通）→ Snapdragon Profiler
│   │   ├── Mode 1: Real-time GPU Metrics（实时监控）
│   │   │   ├── GPU Frequency（频率，降频会显示）
│   │   │   ├── GPU % Utilization（利用率）
│   │   │   ├── Vertex / Fragment / Compute 占比
│   │   │   └── Memory Bandwidth（DDR 带宽使用量）
│   │   ├── Mode 2: Frame Capture（抓帧分析）
│   │   │   ├── 逐 Draw Call 耗时
│   │   │   ├── Pipeline State（Shader / Texture / Render State）
│   │   │   ├── Texture bandwidth per draw
│   │   │   └── Overdraw visualization
│   │   └── Mode 3: Trace（系统级 Trace）
│   │       ├── CPU + GPU + IO 时间线
│   │       └── 定位 CPU-GPU 同步等待
│   ├── Mali（Arm）→ Arm Mobile Studio
│   │   ├── Streamline（系统级性能监控）
│   │   │   ├── Mali GPU Counters（80+ 个硬件计数器）
│   │   │   │   ├── MaliCore：Cycle / Task / Warps
│   │   │   │   ├── MaliTiler（顶点处理）：Vertex / Vertex reuse
│   │   │   │   └── MaliFragment（像素处理）：Quads / Overdraw / Z-rate
│   │   │   ├── CPU Performance Counters
│   │   │   └── Memory & Bus（带宽热力图）
│   │   ├── Mali Offline Compiler（Shader 性能预估）
│   │   │   ├── 编译 Shader 到 Mali 汇编
│   │   │   ├── 输出：register pressure / arithmetic cycles / texture accesses
│   │   │   └── 不需要设备就能预估 Shader 性能
│   │   ├── Graphics Analyzer（帧调试）
│   │   │   └── 类似 RenderDoc，但针对 Mali 优化
│   │   └── Performance Advisor（自动报告生成）
│   │       └── 生成 HTML 报告，自动标注瓶颈和建议
│   └── Apple GPU（iOS）→ Instruments → Metal System Trace
│       ├── GPU 时间线（Vertex / Fragment / Compute）
│       └── 设备散热监控（Thermal State）
├── 第四步：瓶颈定位（四大瓶颈理论）
│   ├── 1. Geometry/Vertex Bound（几何瓶颈）
│   │   ├── 指标：Tiler Cycle / Vertex Cycle 高
│   │   ├── 症状：减少三角形数量后帧率明显上升
│   │   ├── 常见原因：Mesh 过密 / Skinned Mesh 顶点多 / LOD 缺失
│   │   └── 验证方法：RenderDoc 看 Vertex Count per Draw Call
│   ├── 2. Fill Rate/Fragment Bound（填充率瓶颈）
│   │   ├── 指标：Fragment Cycle / Quads Rendered 高
│   │   ├── 症状：降低分辨率后帧率明显上升
│   │   ├── 常见原因：Overdraw 过高 / Fragment Shader 过重 / 透明物体排序
│   │   └── 验证方法：Snapdragon 的 Overdraw View / Mali 的 Overdraw Counter
│   ├── 3. Bandwidth Bound（带宽瓶颈）
│   │   ├── 指标：DDR Read/Write 总量 / L2 Cache Miss Rate
│   │   ├── 症状：降低贴图分辨率后帧率上升 / 多个全屏 Pass 叠加时帧率下降
│   │   ├── 常见原因：高分辨率贴图 / 过多全屏后处理 / MSAA 高倍率
│   │   └── 验证方法：Mali 的 L2 Cache Miss / Snapdragon 的 Memory Bandwidth
│   └── 4. Shader Compute Bound（算力瓶颈）
│       ├── 指标：Arithmetic Cycle / Register Pressure 高
│       ├── 症状：简化 Shader 后帧率上升 / 光照复杂场景帧率低
│       ├── 常见原因：Shader 数学运算过重 / 分支太多 / 寄存器溢出
│       └── 验证方法：Mali Offline Compiler 编译报告 / Snapdragon Shader Analysis
├── 第五步：尖刺专项（Spike Hunting）
│   ├── 周期性 GPU spike 可能原因
│   │   ├── 动态贴图上传（Texture Upload to GPU，如 RuntimeTexture2D.LoadRawTextureData）
│   │   ├── Render Texture 分配/释放（GC + VRAM 分配）
│   │   ├── Shader 编译（首次渲染某材质变体时编译卡顿 → 预热 Shader）
│   │   ├── GPU 频率波动（DVFS 动态调频 → 锁定 GPU 频率测试）
│   │   └── VSync 双缓冲卡顿（帧提交与 VSync 错位 → 检查 Present Timing）
│   └── 排查方法
│       ├── Unity Profiler 的 GPU Module 逐帧看
│       ├── Snapdragon Trace 模式看 CPU-GPU 时间线
│       ├── 在可疑区域加 Debug 标记（CommandBuffer.BeginSample）
│       └── 二分法注释 Pass（注释一半 Pass，看 spike 是否消失）
└── 第六步：优化验证闭环
    ├── 优化前：记录基线数据（GPU Time / Draw Call / Bandwidth）
    ├── 优化后：相同场景相同相机，对比数据
    ├── 多设备验证（高通 + Mali + 至少 3 档性能设备）
    └── 持续监控（CI 中集成性能 Benchmark）
```

#### 代码实现

**1. Unity GPU Profiling 标记代码**

```csharp
using UnityEngine;
using UnityEngine.Rendering;

public class GPUProfileMarkers : MonoBehaviour
{
    // 自定义 Profiler Marker（在 Profiler 中显示为独立条目）
    static readonly UnityEngine.Profiling.ProfilerMarker sm_opaquePass =
        new UnityEngine.Profiling.ProfilerMarker("Custom.OpaquePass");
    static readonly UnityEngine.Profiling.ProfilerMarker sm_postProcess =
        new UnityEngine.Profiling.ProfilerMarker("Custom.PostProcess");
    static readonly UnityEngine.Profiling.ProfilerMarker sm_uiPass =
        new UnityEngine.Profiling.ProfilerMarker("Custom.UIPass");

    public Camera cam;
    public CommandBuffer cb_opaque;
    public CommandBuffer cb_post;

    void OnEnable()
    {
        // 在特定 Camera Event 注入 Profiler 标记
        cam.AddCommandBuffer(CameraEvent.BeforeForwardOpaque, CreateMarkerCB("== OPAQUE START =="));
        cam.AddCommandBuffer(CameraEvent.AfterForwardOpaque, CreateMarkerCB("== OPAQUE END =="));
        cam.AddCommandBuffer(CameraEvent.BeforeImageEffects, CreateMarkerCB("== POSTPROCESS START =="));
        cam.AddCommandBuffer(CameraEvent.AfterImageEffects, CreateMarkerCB("== POSTPROCESS END =="));
    }

    private CommandBuffer CreateMarkerCB(string label)
    {
        CommandBuffer cb = new CommandBuffer { name = label };
        // 空 CommandBuffer，只用于在 GPU Timeline 中做标记
        return cb;
    }

    // GPU 内存监控（每秒采样）
    float gpuMemTimer = 0;
    void Update()
    {
        gpuMemTimer += Time.deltaTime;
        if (gpuMemTimer >= 1f)
        {
            gpuMemTimer = 0;
            // Unity 2021+ 的 Graphics 内存 API
            UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong();
            // 用 Profiler.Log 以在 Logcat 中追踪
            Debug.Log($"[GPU Mem] Allocated: {UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong() / 1024 / 1024}MB");
        }
    }
}
```

**2. Unreal Engine GPU 分析命令**

```cpp
// 控制台命令（开发构建）

// 1. 基本 GPU 耗时分解
stat gpu

// 2. 详细渲染统计
stat scenerendering

// 3. 单帧 GPU Trace（生成 .ucache 文件，用 Insights 打开）
trace.start gpu
// ... 运行游戏 ...
trace.stop

// 4. 锁定 GPU 频率（排除 DVFS 干扰，需要 Android Root 或工程机）
// 在设备 Shell 中：
// Adreno: echo performance > /sys/class/kgsl/kgsl-3d0/devfreq/governor
// Mali:   echo performance > /sys/class/misc/mali0/device/devfreq/governor

// 5. 渲染 Pass 详析
stat dumpticks
// 输出到 Saved/Profiling/ 目录，可逐行分析
```

**3. Mali Streamline 核心计数器速查表**

```
// 在 Streamline 中需要关注的 Mali 计数器（按瓶颈分类）

// === Geometry/Vertex Bound ===
ARM_Mali-Tiler_Cycles            // Tiler 总周期数（高=顶点瓶颈）
ARM_Mali-Tiler_Vertices          // 处理的顶点数
ARM_Mali-Tiler_Primitives        // 处理的三角形数
ARM_Mali-Tiler_VertexReuse       // 顶点复用率（低=Mesh 拓扑差）

// === Fragment/Fill Rate Bound ===
ARM_Mali-Core_Cycles             // Core 总周期数
ARM_Mali-Core_Fragments          // 处理的像素数
ARM_Mali-Core_Quads             // Quad 数量（Overdraw 指标）
ARM_Mali-Core_Thread Starvation  // 线程饥饿（高=CPU 提交不及时）

// === Bandwidth Bound ===
ARM_Mali-L2_Cycles              // L2 Cache 周期
ARM_Mali-L2_External_Read_Axis // 外部读取（DDR）→ 高=带宽瓶颈
ARM_Mali-L2_External_Write_Axis// 外部写入
// 计算 Bandwidth = (Read_Beats + Write_Beats) * Bus_Width * Frequency

// === Shader Compute Bound ===
ARM_Mali-Core_Execute_AlU    // ALU 指令周期（高=计算瓶颈）
ARM_Mali-Core_Execute_Load_Store // Load/Store 周期
ARM_Mali-Core_Texture        // 纹理采样周期

// === 综合判定公式 ===
// Bound type = argmax across:
//   VertexBound = Tiler_Cycles / Total_Cycles
//   FragmentBound = Fragment_Cycles / Total_Cycles
//   BandwidthBound = External_Beats / Max_Beats_Per_Frame
//   ComputeBound = Execute_ALU / Total_Execute
```

### ⚡ 实战经验

> **踩坑1：热节流假阳性**
> 测试手机跑 5 分钟后帧率下降 30%，以为是 Shader 问题，实际是 SoC 热节流。**所有 GPU Profiling 测试必须用散热背夹**，否则数据不可信。在 Snapdragon Profiler 中监控 GPU Frequency，如果频率从 800MHz 降到 400MHz，就是降频了。
>
> **踩坑2：Unity GPU Profiling 不准确**
> Unity Editor 模式下的 GPU Profiling 数据与真机差异很大（Editor 用桌面 GPU，真机用移动 GPU）。必须在 Player Build + Development Build + 连接 Profiler 才有意义。进一步：Unity 的 GPU Profiler 在 Android 上需要 OpenGLES3 + `Enable GPU Profiling` 选项，Vulkan 下可能不支持。
>
> **踩坑3：RenderDoc 性能不等于真机**
> RenderDoc 在 PC 上抓帧分析，用的是桌面 GPU 的性能数据。移动端 TBDR 架构（Tile-Based Deferred Rendering）的瓶颈与桌面 IMR 不同。RenderDoc 适合分析 Draw Call 结构和状态切换，**不适合做性能分析**。性能分析必须用 Snapdragon Profiler / Streamline。
>
> **踩坑4：Mali Offline Compiler 与实际性能差异**
> Mali Offline Compiler 给出的是理论 ALU 周期数，不包含纹理采样延迟和带宽。实际 GPU 运行时可能因 Cache Miss 而慢 2-3 倍。OC 报告适合做 Shader 之间的**相对比较**（Shader A 比 B 多 50% 周期），不适合做**绝对预测**。
>
> **踩坑5：周期性掉帧的真正元凶**
> 实际项目中，「每几秒掉帧一次」最高频的原因排序：
> 1. GC.Collect()（占 40%）→ 优化内存分配
> 2. AssetBundle 加载（占 25%）→ 预加载 / 异步加载
> 3. 纹理上传到 GPU（占 15%）→ Texture Streaming 配置
> 4. Shader 编译（占 10%）→ Shader 预热（Shader Warm-up）
> 5. GPU DVFS 调频（占 5%）→ 锁频
> 6. 实际 GPU 渲染（占 5%）→ 本题的表面问题，但实际概率最低
>
> **经验法则**：先排 1-4，再排 5-6，比直接分析 GPU 渲染效率高 10 倍。

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明盲区在 |
|--------|-----------|----------------|
| CPU 瓶颈和 GPU 瓶颈怎么区分？ | Profiler 对比 CPU Time vs GPU Time | 性能分析基础 |
| Adreno 和 Mali 的工具链有什么不同？ | Snapdragon Profiler vs Arm Mobile Studio | 多平台工具经验 |
| TBDR 架构对性能分析有什么影响？ | Tile 中间存储，带宽模式与 IMR 不同 | 移动 GPU 架构 |
| 怎么判断是顶点瓶颈还是像素瓶颈？ | Tiler Cycle vs Fragment Cycle 指标对比 | GPU 管线理解 |
| 周期性掉帧最常见的 3 个原因是什么？ | GC / Asset 加载 / 纹理上传 | 实战排查经验 |
| 热节流怎么排除？ | 散热背夹 + 监控 GPU 频率 | 测试方法论 |
| RenderDoc 能做性能分析吗？ | 不能，只做结构分析，性能用厂商工具 | 工具适用范围 |
| Mali Offline Compiler 的局限性是什么？ | 不含带宽和 Cache 影响，适合相对比较 | 工具理解深度 |
| GPU 频率波动（DVFS）怎么锁定？ | adb shell 写 governor sysfs | Android 系统层 |
| Shader 预热怎么做？ | 预渲染材质变体收集 → 收集关键词 → 启动时渲染 | URP Shader 管理 |

### 🔗 相关问题

- [手机 GPU 过热分析](../optimization/mobile-overheating-gpu-analysis) — 热节流专题
- [顶点处理瓶颈](../optimization/vertex-bound-bottleneck) — Vertex Bound 深入
- [GPU 带宽优化](../optimization/gpu-bandwidth-optimization) — Bandwidth Bound 深入
- [GPU 内存预算](../optimization/gpu-memory-budget) — VRAM 分配策略
