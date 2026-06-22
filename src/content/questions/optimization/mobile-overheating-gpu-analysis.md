---
title: "手游上线后发热严重 + 偶发掉帧——你拿到一段 GPU Frame Capture，怎么定位瓶颈？"
category: "optimization"
level: 3
tags: ["GPU性能分析", "移动端优化", "RenderDoc", "Arm Mali", "Adreno", "发热优化"]
hint: "先分 CPU bound 还是 GPU bound，GPU 内部再看带宽/ALU/纹理采样——用数据说话，别靠猜"
related: ["optimization/drawcall-500-to-100", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的一款手游上线后，用户反馈手机发热严重、玩 20 分钟后偶发掉帧。测试组给了一段 Android 上的 GPU Frame Capture（Adreno 设备），你打开一看发现这帧 18ms（目标 16.6ms）。怎么定位是哪个环节的问题，你会用哪些工具，最终怎么优化？」

这是一个**开放式性能分析题**，面试官不是在考某个具体知识点，而是在考你的**系统性排障思维**——能不能像侦探一样从数据出发，一步步缩小范围。

### ✅ 核心要点

1. **先定大类**：CPU bound vs GPU bound → 用 Profiler 的 CPU/GPU 时间分离判断
2. **GPU 内部再细分**：带宽 bound / ALU bound / 纹理采样 bound / 填充率 bound
3. **工具链**：Snapdragon Profiler（Adreno）/ Arm Streamline（Mali）/ RenderDoc（通用）+ Unity Profiler
4. **发热 = 持续高功耗** → 降频/降分辨率/减少后台计算
5. **偶发掉帧 ≠ 平均掉帧** → 找尖刺（spike），大概率是 IO 或 GC

### 📖 深度展开

#### 解决思路（从症状倒推根因）

```
症状：发热严重 + 偶发掉帧（20分钟后才出现）
                ↑
倒推1：发热 = GPU/CPU 持续高频运转 → 哪些 pass 在烧 GPU？
倒推2：偶发掉帧（非持续）→ spike 来源可能不是 GPU
倒推3：「20分钟后才出现」→ 可能是温度降频 / 内存碎片化 / 资源加载
倒推4：GPU Frame 显示 18ms → 瓶颈在 GPU 还是 CPU 主线程？
倒推5：分三条线并行排查 → GPU 渲染线 / CPU 主线程线 / 内存/IO 线
```

#### 知识点拆解（倒推树）

```
移动端性能瓶颈定位
├── 第一步：CPU vs GPU 分类
│   ├── Unity Profiler
│   │   ├── CPU main thread 时间 vs GPU job 时间
│   │   ├── Rendering / Physics / Scripts / Animation 各占比
│   │   └── 如果 GPU time > CPU time → GPU bound
│   ├── GPU Frame Capture 分析
│   │   ├── Snapdragon Profiler：Timeline / State / Stage 三视图
│   │   ├── 总帧时间分布：每个 Draw Call 耗时
│   │   └── 找到 Top 3 最耗时的 Draw / Dispatch
│   └── 快速判断法
│       ├── 关掉所有 shader 只渲染纯色 → 如果帧时间暴降 → shader 复杂度过高
│       ├── 降低 Render Resolution 到 50% → 如果帧时间达标 → 填充率 bound
│       └── 降低纹理分辨率 → 如果帧时间下降 → 带宽 bound
├── 第二步：GPU 内部分类（GPU bound 时）
│   ├── 带宽 bound（Memory Bandwidth Bound）
│   │   ├── 症状：半透明 overdraw 高、大量纹理采样
│   │   ├── 验证：Snapdragon Profiler 看 "Memory Bytes Read/Written"
│   │   ├── 优化方向：合批减少 pass、减少透明物体、贴图压缩
│   │   └── 典型杀手：UI 的 Canvas 重叠（Unity UI 是透明渲染）
│   ├── ALU bound（计算瓶颈）
│   │   ├── 症状：shader 复杂度高（多重光照、复杂数学运算）
│   │   ├── 验证：Mali 的 Arithmetic/Cycles metric
│   │   ├── 优化方向：shader LOD、简化光照模型、预计算
│   │   └── 典型杀手：片元着色器中的 for 循环采样多个光源
│   ├── 纹理采样 bound（Texture Bound）
│   │   ├── 症状：大纹理、无 Mipmap、各向异性过滤开太高
│   │   ├── 验证：Profiler 看纹理带宽占比
│   │   ├── 优化方向：Mipmap、ASTC 压缩、减少采样次数
│   │   └── 典型杀手：UI Atlas 不用 Mipmap（正确），但角色 4K 贴图没用压缩（错误）
│   ├── 填充率 bound（Fill Rate / ROP Bound）
│   │   ├── 症状：高分辨率 + 多重透明叠加 + 全屏后处理
│   │   ├── 验证：降低分辨率后帧时间改善明显
│   │   ├── 优化方向：降低 Render Scale、减少后处理 Pass、优化 UI 层级
│   │   └── 典型杀手：Bloom + DOF + Motion Blur 三件套全开
│   └── 几何处理 bound（Vertex Processing Bound）
│       ├── 症状：高模角色、骨骼多、Vertex Shader 中做大量计算
│       ├── 验证：简化模型后帧时间变化
│       ├── 优化方向：LOD、GPU Skinning 优化、减少 Vertex 计算
│       └── 典型杀手：角色 10 万三角面 + 每顶点光照计算
├── 第三步：发热专项分析
│   ├── 功耗 = 频率 × 电压² → 高频运行 = 高功耗
│   ├── GPU 频率曲线分析
│   │   ├── Snapdragon Profiler → Performance → GPU Frequency
│   │   ├── 如果 GPU 一直在最高频 → 说明渲染管线持续吃满
│   │   └── 目标：让 GPU 有"空闲呼吸"时间（帧内时间分片）
│   ├── 降频策略
│   │   ├── 主动限制 GPU 最高频率（Quality Settings → Frame Rate Cap）
│   │   ├── Adaptive Performance（Samsung/Unity）动态分辨率
│   │   └── 低端机目标帧率 30fps（留更多空闲时间散热）
│   └── 典型发热元凶
│       ├── UI 的 Canvas 每帧重建（Graphic Rebuild）
│       ├── 全屏后处理没有做 LOD 降级
│       ├── 物理 FixedUpdate 频率过高
│       └── 后台 Coroutine 持续运行
├── 第四步：偶发掉帧（spike）分析
│   ├── GC Spike
│   │   ├── Unity Profiler → Memory → GC Alloc 列
│   │   ├── 每帧 GC Alloc > 0 → 找到分配源头
│   │   └── 典型元凶：LINQ、foreach 对非优化集合、字符串拼接
│   ├── IO Spike
│   │   ├── Resources.Load / AssetBundle.LoadAsync 在主线程
│   │   ├── Addressables 加载未完全异步
│   │   └── 典型元凶：20分钟后可能触发某资源回收 + 重新加载
│   ├── 温度降频
│   │   ├── 手机持续运行 20 分钟后温度上升 → SoC 降频
│   │   ├── 降频后同样工作量耗时更长 → 掉帧
│   │   └── 验证：监控 GPU/CPU 频率曲线，看是否在掉帧时刻频率骤降
│   └── 内存压力
│       ├── 内存增长 → 系统 GC → 全局暂停
│       ├── AssetBundle 没有及时 Unload
│       └── 解决：定期 Resources.UnloadUnusedAssets()
└── 第五步：优化落地与验证
    ├── 建立性能基线
    │   ├── 固定场景的性能录制（每版本回归测试）
    │   ├── 目标机型分级：高/中/低各一台
    │   └── 自动化测试：Unity Test Framework + 性能采样
    ├── 逐项优化 + 逐项验证（不要一次改多个）
    │   ├── 每改一项，重新录制帧数据
    │   ├── A/B 对比帧时间、GPU 频率、温度曲线
    │   └── 回归检查：优化不能引入视觉退化
    └── 发布后监控
        ├── 接入性能统计 SDK
        ├── 收集真实用户帧率分布
        └── 设置发热投诉阈值告警
```

#### 代码实现

**Adreno GPU 频率监控（C# 工具脚本）**

```csharp
using UnityEngine;
using UnityEngine.Android;

/// <summary>
/// 移动端性能监控面板
/// TA 用它在真机上定位发热/掉帧根因
/// </summary>
public class PerformanceMonitor : MonoBehaviour
{
    [Header("监控配置")]
    [SerializeField] private float sampleInterval = 0.5f;
    [SerializeField] private int maxSamples = 120; // 60s @ 0.5s
    
    private float[] _frameTimes;
    private float[] _gpuTimes;
    private int _currentIndex;
    
    // Unity 2021+ 提供 FrameTiming
    private FrameTiming[] _frameTimings = new FrameTiming[1];
    
    void Start()
    {
        _frameTimes = new float[maxSamples];
        _gpuTimes = new float[maxSamples];
        InvokeRepeating(nameof(Sample), sampleInterval, sampleInterval);
    }
    
    void Sample()
    {
        // 记录帧时间
        _frameTimes[_currentIndex] = Time.unscaledDeltaTime * 1000f;
        
        // 获取 GPU 时间（需要 Unity 2021.2+）
        if (FrameTimingManager.GetLatestTimings(1, _frameTimings) > 0)
        {
            _gpuTimes[_currentIndex] = (float)_frameTimings[0].gpuFrameTime;
        }
        
        _currentIndex = (_currentIndex + 1) % maxSamples;
    }
    
    void OnGUI()
    {
        if (!showDebug) return;
        
        GUILayout.BeginArea(new Rect(10, 10, 400, 300));
        GUILayout.Label($"<color=white><b>=== Performance Monitor ===</b></color>");
        
        // 当前帧率
        float avgFrame = GetAverage(_frameTimes);
        float avgGpu = GetAverage(_gpuTimes);
        GUILayout.Label($"Frame: {avgFrame:F1}ms (FPS: {1000f/avgFrame:F0})");
        GUILayout.Label($"GPU: {avgGpu:F1}ms | CPU bound: {(avgGpu < avgFrame * 0.7f)}");
        
        // 尖刺检测
        float maxFrame = GetMax(_frameTimes);
        if (maxFrame > avgFrame * 2f)
        {
            GUILayout.Label($"<color=red>⚠ SPIKE: {maxFrame:F1}ms (avg: {avgFrame:F1}ms)</color>");
        }
        
        // GC  alloc 检查
        var gcSample = System.GC.GetTotalMemory(false);
        GUILayout.Label($"GC Memory: {gcSample / 1024 / 1024:F1}MB");
        
        // 温度等级（Android 专属 API，需要 NDK 或 Plugin）
        GUILayout.Label($"Battery Temp: {GetBatteryTemp()}°C");
        
        GUILayout.EndArea();
    }
    
    float GetAverage(float[] arr)
    {
        float sum = 0; int count = 0;
        for (int i = 0; i < arr.Length; i++)
        {
            if (arr[i] > 0) { sum += arr[i]; count++; }
        }
        return count > 0 ? sum / count : 0;
    }
    
    float GetMax(float[] arr)
    {
        float max = 0;
        for (int i = 0; i < arr.Length; i++)
            if (arr[i] > max) max = arr[i];
        return max;
    }
    
    float GetBatteryTemp()
    {
        // Android: /sys/class/power_supply/battery/temp
        // 返回值为 10 倍实际温度（如 350 = 35.0°C）
#if UNITY_ANDROID && !UNITY_EDITOR
        try
        {
            using (var reader = new System.IO.StreamReader("/sys/class/power_supply/battery/temp"))
            {
                string temp = reader.ReadToEnd().Trim();
                if (float.TryParse(temp, out float result))
                    return result / 10f;
            }
        }
        catch { }
#endif
        return -1f;
    }
    
    [SerializeField] private bool showDebug = true;
}
```

**Mali GPU 的 Streamline 分析流程（工具操作伪代码）**

```
Step 1: 连接设备 → adb connect <device_ip>
Step 2: 启动 Streamline Capture → 选择 Mali Counter 模板
Step 3: 运行游戏 5 分钟（覆盖正常 → 发热 → 掉频的过程）
Step 4: 分析 Timeline：
        - 找 GPU Frequency 持续高位的时间段
        - 对比该时间段内各 Mali Counter 的变化
        - 重点看：
          * MaliCoreActiveCycles / MaliShaderCoreCycles → ALU 占用率
          * MaliTexFetchInstrExecuted → 纹理采样次数
          * MaliL2CacheWrite / Read → 带宽
          * MaliTileMemWrite → Tile Memory 写入（带宽指标）
Step 5: 导出报告 → 与 Snapdragon Profiler 数据交叉验证
```

#### 面试追问预演

**追问1：「你查到是 UI 的 Canvas 导致每帧重建，占用 6ms GPU 时间。美术说 UI 不能减，你怎么优化？」**

> Canvas 重建是 Unity UI 的经典问题：
> 1. **Canvas 分层**：把静态 UI（边框、背景）和动态 UI（血条、伤害数字）拆到不同 Canvas，避免整体重建
> 2. **Canvas.willRenderCanvases 监控**：用 Profiler 确认是否每帧都在 rebuild
> 3. **UI Mesh 复用**：对于不变化的 UI 元素，考虑用自定义 Mesh 而非 CanvasRenderer
> 4. **如果是 overdraw 问题**：减少 UI 层级重叠、禁用不可见 UI 的 Canvas 组件（而不是只禁用 GameObject）
> 5. **极端方案**：核心 HUD 用专门的 World Space 渲染，脱离 Canvas 体系

**追问2：「发热问题你怎么和硬件厂商合作？」**

> 大厂一般有厂商技术合作渠道：
> - **高通**：Snapdragon Profiler + 直接对接技术顾问，可以拿到 GPU counter 的深度数据
> - **Arm（Mali）**：Streamline + Mobile Studio，Arm 官方有 TA 支持团队
> - **联发科**：Dimensity Profiler，相对年轻但可拿到频率/温度曲线
> - TA 的工作是：拿到厂商工具的分析报告 → 翻译成可操作的优化项 → 在项目内推进落地

**追问3：「如果这帧 18ms 但平均只有 15ms，你怎么决定优化优先级？」**

> 关键区分：**平均帧时间 vs P99 帧时间 vs 最大帧时间**
> - 平均 15ms 看似达标（60fps），但 P99 如果是 22ms → 每 100 帧掉一次
> - 发热看的是**平均值和持续性**（持续 15ms > 瞬间 22ms 的功耗影响）
> - 偶发 spike 看的是**最大值和频率**
> - 优先级策略：先优化发热（影响全体用户），再优化 spike（影响体验尖刺）
> - 如果 spike 来源于温度降频 → 优化发热本身就能同时解决两个问题

### ⚡ 实战经验

1. **永远先看 GPU 频率曲线**。如果 GPU 在持续最高频率运行，说明渲染管线一直在压榨硬件。优化目标不是帧时间最小化，而是让 GPU 有"喘息"时间——这直接对应功耗和发热。

2. **20 分钟后的掉帧，90% 是温度降频**。SoC 的温控策略通常在电池温度达到 38-42°C 时开始降频。你优化的不是帧率，而是**单位帧的 GPU 功耗**。

3. **Snapdragon Profiler 是 Adreno 的金标准，但 ARM Streamline 是更深的工具**。Arm 的 Mali Counter 体系比 Adreno 更细粒度（因为 Arm 自己造 IP，高通是买授权）。如果你两个都会用，面试加分很大。

4. **不要过度优化 CPU**。Unity 项目中如果 GPU bound，CPU 优化对帧率无感。先确认瓶颈再出手——"先诊断再开刀"是 TA 的基本素养。

5. **UI 是移动端性能的隐形杀手**。美术觉得 UI 不"重"，但 Canvas 的 Batch 重建和透明 overdraw 是移动端的头号性能消耗者。很多发热问题的根因是 UI 设计不合理。

### 🎯 能力体检清单

- [ ] **我能不能用 Snapdragon Profiler 或 Arm Streamline 做帧分析？** → 不能说明移动端性能工具链有盲区
- [ ] **我知不知道 CPU bound 和 GPU bound 的判断方法？** → 不知道说明性能分析基础不牢
- [ ] **我理解发热和帧率优化的区别吗？** → 不理解说明缺少移动端实战经验
- [ ] **我能不能识别 UI Canvas 的性能问题？** → 不能说明 Unity UI 优化经验不足
- [ ] **我有没有建立性能回归测试的习惯？** → 没有说明工程化思维需加强
- [ ] **面对"偶发"问题，我能不能区分 GC/IO/降频等不同原因？** → 不能说明系统级排障能力不足

### 🔗 相关问题

- [Draw Call 从 500 降到 100：合批策略怎么做？](../optimization/drawcall-500-to-100.md)
- [GPU 带宽优化：移动端怎么减少带宽消耗？](../optimization/gpu-bandwidth-optimization.md)
- [顶点处理瓶颈：角色太复杂导致 GPU 顶点阶段卡住](../optimization/vertex-bound-bottleneck.md)
