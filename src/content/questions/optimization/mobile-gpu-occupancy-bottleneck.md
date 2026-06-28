---
title: "手游帧率不稳但 GPU 利用率才 40%？如何诊断移动端 GPU Occupancy 瓶颈？"
category: "optimization"
level: 4
tags: ["性能优化", "GPU Occupancy", "Adreno", "Mali", "Warp", "寄存器压力", "性能诊断"]
hint: "GPU 利用率低 ≠ 没活干，可能是 Warp 被阻塞（寄存器/纹理依赖/LDS 不足），需要从 Occupancy 维度而非纯时间维度分析"
related: ["optimization/adreno-tile-based-bandwidth", "optimization/mali-gpu-fpr-rollback-stall", "optimization/shader-variant-explosion"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们的手游在骁龙 888 上测试，Profile 显示 GPU 利用率只有 40%，但帧率就是稳不住——经常从 60 掉到 45-50，画面没有特别复杂的场景。Snapdragon Profiler 显示顶点处理和片段着色器时间都不长，你说问题出在哪里？

> 追问：如果你接手这个项目，你会从哪些维度排查？什么是 GPU Occupancy？为什么低利用率不代表 GPU 没有瓶颈？

这是腾讯（天美/光子）、网易（雷火）、字节（朝夕光年）等大厂高级 TA / 渲染程序员面试的高阶题。考察的不是"某个具体优化技巧"，而是**性能诊断的方法论**——能否从"GPU 利用率低"这个反直觉现象出发，深入 GPU 微架构层面找到真正的瓶颈。

### ✅ 核心要点

1. **GPU 利用率 ≠ Occupancy**：GPU 利用率 40% 意味着 GPU 有 60% 的时间在做"什么"，但不一定是空闲——可能在等待（内存回读、纹理依赖）、可能在串行执行（分歧 Warp）、可能 Occupancy 不足导致计算单元空转
2. **Occupancy = 活跃 Warp 数 / 最大 Warp 数**：寄存器压力、LDS 使用量、线程数都会影响一个 SM/Warp Engine 能同时容纳多少 Warp
3. **低帧率 + 低利用率 = Stall 型瓶颈**：不是计算量太大，而是 GPU 在等待——纹理采样延迟、寄存器溢出、分支分歧、Tile 切换开销
4. **诊断工具链**：Snapdragon Profiler（Adreno）、Mali Graphics Debugger（Mali）、XCode GPU Frame Capture（Apple GPU）各自的 Occupancy/Stall 指标

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
现象：GPU 利用率 40% + 帧率不稳（60→45）
         ↓
第一层分析：GPU 不是"忙不过来"，是"跑不快"
         ↓
第二层分析：为什么跑不快？
  ├── 可能A：Occupancy 太低 → 计算单元喂不饱（空转）
  ├── 可能B：Stall 太多 → Warp 在等待（内存/纹理/依赖）
  ├── 可能C：不均匀负载 → 有些帧特别重（突发）
  └── 可能D：CPU-GPU 同步点 → GPU 等 CPU 提交
         ↓
第三层分析：用工具定位是哪个"可能"
  ├── Snapdragon Profiler → 看 Stall 热点 + Occupancy
  ├── RenderDoc → 看 Draw Call 时间线 + 状态切换
  └── XCode GPU Capture → 看 Tile 内存带宽 + 寄存器使用
         ↓
第四层分析：针对性优化
```

#### 知识点拆解（倒推树）

```
GPU Occupancy 瓶颈诊断
├── 理解 GPU 并行模型
│   ├── CPU vs GPU 执行模型的根本差异
│   │   ├── CPU：低延迟（大缓存/分支预测/深流水线）
│   │   └── GPU：高吞吐（海量线程/延迟容忍/宽 SIMD）
│   ├── Warp/Wavefront 概念
│   │   ├── Adreno：Wave = 128 threads（灵活）
│   │   ├── Mali：Warp = 16~32 threads（可变）
│   │   └── Apple GPU：SIMD-group = 32 threads
│   └── 延迟隐藏：靠大量并发 Warp 填满流水线
│       └── 如果活跃 Warp 不够 → 流水线空转 → 低 Occupancy
│
├── Occupancy 的计算与限制因素
│   ├── 理论 Occupancy
│   │   ├── 受限于：每 SM 最大 Warp 数
│   │   ├── 受限于：寄存器文件大小 / 每个 Warp 使用寄存器数
│   │   ├── 受限于：LDS/共享内存容量
│   │   └── 受限于：最大线程数 / 最大线程组数
│   ├── 实际 Occupancy
│   │   └── 可能因分支分歧、资源使用不均匀而低于理论值
│   └── 为什么 Occupancy 重要但不等于性能
│       └── 高 Occupancy = 更多 Warp 隐藏延迟，但不保证高吞吐
│
├── Stall 类型分类（GPU 为什么"等待"）
│   ├── Texture Sample Stall
│   │   └── 纹理采样延迟（未命中缓存 → DRAM 回读 → 数百周期）
│   │   ├── 诊断：Profiter 显示 "Texture Fetch" Stall 占比高
│   │   └── 优化：减少纹理采样次数、提高缓存命中率（Mipmap！）
│   ├── Register Pressure → Spill
│   │   ├── 寄存器不够 → 数据溢出到内存（Spill） → 巨慢
│   │   ├── 诊断：Shader 编译报告的寄存器使用量
│   │   └── 优化：减少临时变量、简化 Shader 复杂度
│   ├── Branch Divergence
│   │   ├── 同一 Warp 内不同线程走了不同分支 → 串行执行
│   │   ├── 诊断：Warp 执行效率 < 80%
│   │   └── 优化：减少 Shader 中的动态分支
│   ├── Memory Dependency Stall
│   │   ├── SSBO/UAV 读写依赖 → 等待之前的写完成
│   │   └── 优化：减少随机写入、使用 Barrier 正确但不过度
│   └── Synchronization Stall
│       ├── Group Memory Barrier / Device Sync
│       └── Tile-Based Rendering 的 Tile 边界同步
│
├── 移动端特有问题
│   ├── TBDR（Tile-Based Deferred Rendering）
│   │   ├── Tile 大小有限（Adreno 通常 256x256 / 512x512）
│   │   ├── Tile 内处理完后写回系统内存 → 带宽
│   │   └── 如果 Tile 内有多个 Pass → Tile 切换开销
│   ├── 寄存器压力对 Tile 的影响
│   │   └── Shader 用的寄存器多 → 每个 Tile 能容纳的像素少 → Occupancy 降
│   ├── Adreno HSR（Hidden Surface Removal）
│   │   └── 如果 Shader 有 discard/clip → HSR 失效 → Overdraw 暴增
│   └── Mali 的 Warp 调度策略
│       └── Mali 的 Warp Scheduler 对纹理依赖敏感
│
└── 诊断工具链
    ├── Snapdragon Profiler（Adreno）
    │   ├── GPU Utilization（但要看 Stall breakdown）
    │   ├── Shader Stall Reasons（Texture/ALU/Memory）
    │   ├── Render Mode（是否在预期的 Direct/TBR 模式）
    │   └── Counter：%Time in Vertex / Fragment / Tiler
    ├── Mali Streamline / MSE（Mali）
    │   ├── Core Activity（Fragment/NonFragment/Tiler）
    │   ├── Thread/Lane Utilization
    │   ├── Texure Filter Stall / Load/Store Stall
    │   └── L2 Cache Hit Rate
    └── XCode GPU Frame Capture（Apple GPU）
        ├── Per-draw timing
        ├── Occupancy（线程组利用率）
        ├── Register count per shader
        └── Texture/Buffer bandwidth
```

#### 代码实现

**诊断脚本框架（Unity C#，自动化采集关键指标）：**

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using UnityEditor.Profiling;
using System.Text;
using System.IO;

/// <summary>
/// GPU Occupancy 诊断工具
/// 用法：在怀疑有问题的帧前后调用 RecordFrame()，导出分析报告
/// </summary>
public class GPUOccupancyDiagnostic : MonoBehaviour {
    [Header("Monitoring")]
    public bool autoMonitor = false;
    public float fpsWarningThreshold = 55f;
    
    [Header("Report")]
    public string outputPath = "GPU_Occupancy_Report.txt";
    
    private float[] _fpsHistory = new float[120];
    private int _fpsIndex = 0;
    private int _frameCount = 0;
    
    // 关键 GPU Counter（通过 Unity Profiler Recorder API）
    private Recorder _drawCallsRecorder;
    private Recorder _batchesRecorder;
    private Recorder _setPassRecorder;
    private Recorder _trianglesRecorder;
    private Recorder _verticesRecorder;
    
    void Start() {
        _drawCallsRecorder = Recorder.Get("Draw Calls");
        _batchesRecorder = Recorder.Get("Batches");
        _setPassRecorder = Recorder.Get("SetPass Calls");
        _trianglesRecorder = Recorder.Get("Triangles");
        _verticesRecorder = Recorder.Get("Vertices");
        
        foreach (var r in new[] { _drawCallsRecorder, _batchesRecorder,
                                  _setPassRecorder, _trianglesRecorder,
                                  _verticesRecorder }) {
            r.enabled = true;
        }
    }
    
    void Update() {
        // FPS 采样
        _fpsHistory[_fpsIndex] = 1f / Time.unscaledDeltaTime;
        _fpsIndex = (_fpsIndex + 1) % _fpsHistory.Length;
        _frameCount++;
        
        // 自动监控：检测异常帧
        if (autoMonitor && CurrentFPS() < fpsWarningThreshold) {
            CaptureSnapshot($"Low FPS Alert: {CurrentFPS():F1}");
        }
    }
    
    float CurrentFPS() {
        float sum = 0;
        int count = Mathf.Min(_frameCount, _fpsHistory.Length);
        for (int i = 0; i < count; i++) sum += _fpsHistory[i];
        return sum / count;
    }
    
    /// <summary>
    /// 诊断快照：抓取当前帧的关键指标
    /// </summary>
    public void CaptureSnapshot(string label = "") {
        StringBuilder sb = new StringBuilder();
        sb.AppendLine($"=== GPU Occupancy Diagnostic Snapshot ===");
        sb.AppendLine($"Time: {System.DateTime.Now:yyyy-MM-dd HH:mm:ss}");
        sb.AppendLine($"Label: {label}");
        sb.AppendLine();
        
        // === 帧率统计 ===
        sb.AppendLine("--- Frame Rate ---");
        sb.AppendLine($"Average FPS: {CurrentFPS():F1}");
        sb.AppendLine($"Min FPS (last 120f): {MinFPS():F1}");
        sb.AppendLine($"Max FPS (last 120f): {MaxFPS():F1}");
        sb.AppendLine($"Frame Time: {Time.unscaledDeltaTime * 1000:F2} ms");
        sb.AppendLine();
        
        // === 渲染统计 ===
        sb.AppendLine("--- Render Stats ---");
        sb.AppendLine($"Draw Calls: {_drawCallsRecorder.lastValue}");
        sb.AppendLine($"Batches: {_batchesRecorder.lastValue}");
        sb.AppendLine($"SetPass Calls: {_setPassRecorder.lastValue}");
        sb.AppendLine($"Triangles: {_trianglesRecorder.lastValue}");
        sb.AppendLine($"Vertices: {_verticesRecorder.lastValue}");
        sb.AppendLine();
        
        // === 诊断分析 ===
        sb.AppendLine("--- Diagnostic Analysis ---");
        AnalyzePotentialBottlenecks(sb);
        
        // === 优化建议 ===
        sb.AppendLine();
        sb.AppendLine("--- Recommendations ---");
        GenerateRecommendations(sb);
        
        string report = sb.ToString();
        Debug.Log(report);
        File.AppendAllText(outputPath, report + "\n\n");
    }
    
    void AnalyzePotentialBottlenecks(StringBuilder sb) {
        int drawCalls = _drawCallsRecorder.lastValue;
        int setPass = _setPassRecorder.lastValue;
        int batches = _batchesRecorder.lastValue;
        
        float fps = CurrentFPS();
        float frameMs = 1000f / Mathf.Max(fps, 1f);
        
        // SetPass 比率：高 = 状态切换频繁
        float setPassRatio = drawCalls > 0 ? (float)setPass / drawCalls : 0;
        if (setPassRatio > 0.5f) {
            sb.AppendLine($"⚠ SetPass/DrawCall ratio = {setPassRatio:P}（>50%），"
                       + "大量 Shader/Material 切换可能导致 GPU Stall");
        }
        
        // Batch 效率
        float batchEfficiency = drawCalls > 0 ? (float)batches / drawCalls : 0;
        if (batchEfficiency > 0.8f) {
            sb.AppendLine($"⚠ Batch/DrawCall ratio = {batchEfficiency:P}，"
                       + "合批效率低，Draw Call 碎片化");
        }
        
        // 帧时间预算分析
        if (frameMs > 16.67f && fps > 0) {
            float overBudget = frameMs - 16.67f;
            sb.AppendLine($"⚠ 超出 16.67ms 预算 {overBudget:F2}ms"
                       + $"（需要削减 {overBudget / frameMs:P} 的工作量）");
        }
        
        // 突发尖峰检测
        float minF = MinFPS();
        float maxF = MaxFPS();
        if (maxF > 0 && minF / maxF < 0.7f) {
            sb.AppendLine($"⚠ FPS 波动大（min={minF:F1}, max={maxF:F1}），"
                       + "可能存在突发负载或同步等待");
        }
    }
    
    void GenerateRecommendations(StringBuilder sb) {
        sb.AppendLine("1. 使用 Snapdragon Profiler / Mali Streamline 检查：");
        sb.AppendLine("   - GPU Stall Breakdown（Texture? ALU? Memory?）");
        sb.AppendLine("   - Occupancy %（目标 > 60%）");
        sb.AppendLine("   - 寄存器使用量（Shader 是否 Spill）");
        sb.AppendLine("2. 检查 Shader 复杂度：");
        sb.AppendLine("   - 是否有过多临时变量导致寄存器压力");
        sb.AppendLine("   - 是否有动态分支导致 Warp 分歧");
        sb.AppendLine("   - 是否有 discard/clip 破坏 HSR");
        sb.AppendLine("3. 检查纹理策略：");
        sb.AppendLine("   - 是否所有纹理都生成了 Mipmap");
        sb.AppendLine("   - 纹理格式是否合理（ASTC vs RGBA）");
        sb.AppendLine("   - 是否有大纹理频繁切换（Cache Thrashing）");
    }
    
    float MinFPS() {
        float min = float.MaxValue;
        int count = Mathf.Min(_frameCount, _fpsHistory.Length);
        for (int i = 0; i < count; i++) if (_fpsHistory[i] < min) min = _fpsHistory[i];
        return min == float.MaxValue ? 0 : min;
    }
    
    float MaxFPS() {
        float max = 0;
        int count = Mathf.Min(_frameCount, _fpsHistory.Length);
        for (int i = 0; i < count; i++) if (_fpsHistory[i] > max) max = _fpsHistory[i];
        return max;
    }
}
#endif
```

**Shader 寄存器审计要点（HLSL → 汇编分析）：**

```hlsl
// === 坏例子：高寄存器压力 Shader ===
// 问题：太多临时变量 + 嵌套计算 → 编译器使用 64+ 寄存器 → Occupancy 减半

half4 BadFragment(Varyings input) : SV_Target {
    // 每个 temp 变量占寄存器空间
    half3 light1 = CalculateLight(input, 0);  // 3 regs
    half3 light2 = CalculateLight(input, 1);  // 3 regs
    half3 light3 = CalculateLight(input, 2);  // 3 regs
    half3 light4 = CalculateLight(input, 3);  // 3 regs
    half3 light5 = CalculateLight(input, 4);  // 3 regs
    half3 ambient = CalculateAmbient(input);  // 3 regs
    half3 subsurface = CalculateSSS(input);   // 3 regs
    half3 reflection = CalculateReflection(input); // 3 regs
    half3 refraction = CalculateRefraction(input); // 3 regs
    half3 emission = CalculateEmission(input);     // 3 regs
    
    // 编译器需要保持所有中间值存活 → 寄存器爆炸
    half3 combined = light1 + light2 + light3 + light4 + light5
                   + ambient + subsurface + reflection + refraction + emission;
    return half4(combined, 1);
}

// === 好例子：低寄存器压力 ===
// 策略：逐光源累加、即时写入、减少存活变量
// 编译器只需 ~32 寄存器 → Occupancy 翻倍

half4 GoodFragment(Varyings input) : SV_Target {
    half3 finalColor = half3(0, 0, 0);
    
    // 逐光源累加，每次循环复用同一组寄存器
    finalColor += CalculateLight(input, 0);
    finalColor += CalculateLight(input, 1);
    finalColor += CalculateLight(input, 2);
    finalColor += CalculateLight(input, 3);
    finalColor += CalculateLight(input, 4);
    
    finalColor += CalculateAmbient(input);
    finalColor += CalculateSSS(input);
    finalColor += CalculateReflection(input);
    
    return half4(finalColor, 1);
}
```

**GPU 架构对照表（面试必背）：**

| 指标 | Adreno (骁龙) | Mali (天玑/Exynos) | Apple GPU (A系列/M系列) |
|------|---------------|---------------------|-------------------------|
| 架构名称 | Adreno 7xx/6xx | Valhall / Immortalis | Apple GPU 12~15 |
| 渲染模式 | FlexRender (TBR/Direct) | TBDR | TBDR |
| Tile 大小 | 256x256 ~ 512x512 | 16x16 ~ 64x64（可配） | 32x32（可变） |
| 隐面消除 | HSR（Hardware） | FWD-Tile + Z-Test | TBDR + Hidden Surface Removal |
| Wave 大小 | 128 threads（可变 64/128） | 16~32 threads | 32 threads |
| 寄存器文件 | 64KB/SP（变体多） | 128KB/核心 | ~96KB/核心 |
| Occupancy 关键 | 寄存器 + Tile 内存 | Warp 并发 + LDS | 线程组并发 + Tile 内存 |
| Profiler | Snapdragon Profiler | Mali Streamline + MSE | XCode GPU Capture |

### ⚡ 实战经验

1. **"GPU 利用率低"是最容易被误读的指标**：产品经理看到"GPU 才 40%"会说"是不是你们 Shader 太简单了"。实际上可能恰恰相反——Shader 太复杂（寄存器压力高）导致 Occupancy 低，GPU 的计算单元大部分时间在空转。这是**高级 TA 和初级 TA 的认知分水岭**
2. **Mipmap 是 Occupancy 的隐形杀手/救星**：没有 Mipmap 的纹理采样会导致 Cache Miss → Texture Stall → Warp 阻塞 → Occupancy 跌。一个 2048x2048 无 Mipmap 的贴图采样可能让整个 Warp 等待 200+ 周期。开了 Mipmap 后 Cache Hit Rate 可能从 30% → 90%，帧率直接稳了。这不是玄学，是 GPU 微架构的必然
3. **discard/clip 是移动端的定时炸弹**：在 Adreno 上，Shader 中用了 `clip()` 或 `discard` 会破坏 HSR（Hidden Surface Removal），导致被遮挡像素也被着色 → Overdraw 暴增 → 实际工作量远超预期。植物/栅栏/alpha-test 物体是重灾区
4. **寄存器压力的"悬崖效应"**：Shader 用了 31 个寄存器时 Occupancy 是 100%，加一个临时变量变成 33 个，Occupancy 直接掉到 50%（因为每 SM 的 Warp 数减半）。这不是线性退化而是阶梯式跳崖。编译器的 Shader Assembly 报告一定要看
5. **诊断顺序很重要**：先看 **帧率曲线**（稳态 vs 尖峰波动）→ 再看 **RenderDoc 时间线**（哪个 Pass 耗时长）→ 然后看 **Snapdragon Profiler Stall 分析**（具体是什么类型的等待）→ 最后看 **Shader 汇编**（寄存器/分支）。跳过前面步骤直接抠 Shader 是"用显微镜找大象"

### 🎯 能力体检清单

- [ ] **如果不知道 Occupancy 是什么** → 你需要补：GPU SIMT 执行模型、Warp/Wavefront 调度原理、延迟隐藏机制
- [ ] **如果不会用 Snapdragon Profiler / Mali Streamline** → 你需要补：移动端 GPU Profiler 的 Counter 体系、Stall 分析方法、如何关联 Shader 代码到 Profiler 热点
- [ ] **如果不懂寄存器压力** → 你需要补：Shader 编译过程（HLSL → DXBC/SPIR-V → GPU ISA）、寄存器分配策略、如何查看 Shader 编译报告中的寄存器使用量
- [ ] **如果不理解 TBDR 和 Occupancy 的关系** → 你需要补：Tile-Based 渲染管线、Tile 内存（TGM/LGM）的工作方式、Tile 内的并行度限制
- [ ] **如果不会区分 Stall 类型** → 你需要补：Texture Stall vs ALU Stall vs Memory Stall 的特征和优化方向、Branch Divergence 的检测方法

### 🔗 相关问题

- [Adreno Tile-Based 带宽优化](optimization/adreno-tile-based-bandwidth.md)：TBDR 架构下的带宽优化策略
- [Mali GPU FPR Rollback Stall](optimization/mali-gpu-fpr-rollback-stall.md)：Mali 特有的 Forward Pixel Kill 回退问题
- [Shader Variant Explosion](optimization/shader-variant-explosion.md)：变体过多如何间接导致 Occupancy 问题（Cache Thrashing）
- 如果 Shader 没有 discard 但帧率仍然不稳，你会怎么排查 Tile 切换开销？
