---
title: "手机发热降频到 30fps：如何做持续性能优化让游戏不掉帧？"
category: "optimization"
level: 4
tags: ["热降频", "持续性能", "移动端", "SustainedPerformance", "ThermalThrottling", "帧率稳定性", "功耗优化"]
hint: "关键不是峰值帧率——而是 30 分钟后还能稳 60fps。需要从 GPU 频率、负载分布、散热模型三方面系统优化"
related: ["optimization/mobile-overheating-gpu-analysis", "optimization/loading-stall-hitch-spike", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官说：

> "我们的游戏在 iPhone 14 Pro 上刚启动能跑 60fps，但玩 15 分钟后帧率开始波动，20 分钟后稳定掉到 35-40fps。用 Instruments 看发现 CPU/GPU 频率被降了。测试团队反馈夏天在室外玩更严重，甚至有设备自动降低屏幕亮度。你是 TA，怎么解决这个发热降频问题？"

这是腾讯/网易/米哈游等做长时游戏体验优化的 **高级 TA 面试题**。考察的是对移动端 SoC 功耗模型、热管理、持续性能（Sustained Performance）的理解。

### ✅ 核心要点

1. **热降频是自我保护机制**：SoC 温度超过阈值后，OS 降低 CPU/GPU 频率来减少发热，帧率自然下降
2. **峰值性能 ≠ 持续性能**：手机能短暂跑满频率，但 10-20 分钟后必然降频（散热设计决定持续性能上限）
3. **功耗 = 热量**：优化发热本质上是优化功耗——每减少 1W 功耗，就减少 1W 热量
4. **负载均衡是核心策略**：把瞬时高负载分摊到更多帧中，避免 SoC 频率冲高后触发降频
5. **品质自适应降级**：检测温度/帧率趋势，主动降低渲染品质，而不是被动等 OS 强制降频

### 📖 深度展开

#### 解决思路（从现象倒推根因和方案）

```
现象：60fps → 15分钟后 → 35fps
                    ↓
根因：SoC 温度超阈值 → OS 降频 → 渲染变慢 → 帧率下降
                    ↓
关键洞察：问题不是「渲染太慢」，而是「渲染太费电」
  ├── GPU 满频运行 15 分钟 → 芯片温度上升 → 降频保护
  └── 如果每帧功耗降低 → 温度不触发阈值 → 频率保持 → 帧率稳定
                    ↓
三层优化策略：
                    ↓
Layer 1：被动散热（硬件层面）
  ├── 不让 SoC 达到降频温度阈值
  ├── 方法：降低整体功耗，减少热量产生
  └── 指标：TDP（热设计功耗）控制在 SoC 持续性能区间
                    ↓
Layer 2：主动自适应（软件层面）
  ├── 实时检测温度趋势和帧率稳定性
  ├── 预测即将降频 → 提前降级品质
  ├── 方法：Dynamic Resolution + Quality Scaling
  └── 目标：帧率优先于画质
                    ↓
Layer 3：负载整形（算法层面）
  ├── 把「尖峰负载」平滑为「持续负载」
  ├── 方法：LOD 提前切换、粒子限流、渲染分帧
  └── 原理：避免 SoC 频率瞬间冲高（冲高→发热→降频的恶性循环）
```

#### 知识点拆解（倒推树）

```
手机热降频与持续性能优化
├── 移动端 SoC 热管理
│   ├── 温度传感器（SoC 内部 / 电池 / 屏幕）
│   ├── Thermal Throttling 机制
│   │   ├── Level 1（轻度）：降低 GPU 最高频率
│   │   ├── Level 2（中度）：降低 CPU 大核 + GPU 频率
│   │   ├── Level 3（重度）：降低屏幕亮度 + 全面降频
│   │   └── Level 4（极端）：弹出过热警告 + 强制暂停
│   ├── iOS vs Android 差异
│   │   ├── iOS：Core Thermal API，私有（开发者只能间接感知）
│   │   ├── Android：ThermalService / PowerManager API
│   │   └── Unity/UE 集成：SystemInfo.thermalStatus
│   └── 设备差异：SoC 散热设计（VC 均热板 vs 石墨片）
├── 功耗模型
│   ├── 动态功耗：P = C × V² × f
│   │   ├── C：电容（与晶体管数量成正比）
│   │   ├── V：电压（频率越高电压越高，平方关系！）
│   │   └── f：频率
│   ├── 静态功耗：漏电流（温度越高越大）
│   ├── GPU 功耗来源
│   │   ├── ALU 计算（Shader 复杂度）
│   │   ├── 纹理采样（带宽）
│   │   ├── 带宽（Frame Buffer / Depth / ShadowMap 读写）
│   │   └── ROP（像素输出）
│   └── CPU 功耗来源
│       ├── Draw Call 提交
│       ├── 蒙皮 / 物理 / AI
│       └── 内存访问（Cache Miss）
├── 持续性能策略
│   ├── Sustained Performance Mode（Android）
│   │   ├── WindowManager.setSustainedPerformanceMode()
│   │   ├── 系统保证不降频，但限制最高频率
│   │   └── 适用：游戏全程需要稳定帧率
│   ├── GameMode API（Android 12+）
│   │   ├── GameManager.setGameMode()
│   │   ├── PERFORMANCE：最高性能
│   │   ├── BATTERY：省电模式
│   │   └── STANDARD：平衡
│   └── iOS Low Power / Thermal Notifications
│       ├── ProcessInfo.thermalState
│       ├── NSProcessInfoPowerState
│       └── 热状态变化通知
├── 品质自适应系统
│   ├── 温度/帧率监控
│   │   ├── 滑动窗口平均帧率（不是瞬时帧率）
│   │   ├── 帧时间方差（jitter 检测）
│   │   ├── 温度趋势预测（线性外推 / 指数加权）
│   │   └── 降频检测：连续 N 帧帧时间 > 16.67ms
│   ├── 降级阶梯
│   │   ├── Stage 0：正常品质
│   │   ├── Stage 1：降低分辨率（0.9x → 0.8x）
│   │   ├── Stage 2：降低 LOD Distance
│   │   ├── Stage 3：降低后处理品质（关 Bloom / 降 SSAO 分辨率）
│   │   ├── Stage 4：降低粒子数量 / 阴影距离
│   │   └── Stage 5：极端——锁定 30fps
│   └── 恢复机制
│       ├── 温度回落后逐步恢复品质
│       ├── 避免频繁切换品质（抖动）
│       └── 最少持续 30 秒再切换
├── 负载整形
│   ├── 渲染分帧（Frame Splitting）
│   │   ├── 每 2 帧渲染一次阴影
│   │   ├── 每 3 帧更新一次环境光探针
│   │   └── 代价：阴影有 1-2 帧延迟（通常不可见）
│   ├── LOD 提前切换
│   │   ├── 正常 LOD 切换距离 × 0.8（提前降级）
│   │   └── 代价：远景品质下降（热降频时可以接受）
│   ├── 粒子限流
│   │   ├── 最大同屏粒子数上限
│   │   ├── 距离剔除（远处粒子不渲染）
│   │   └── LOD（远处粒子降低发射率）
│   └── GPU 频率管理
│       ├── Android：通过 Vulkan EXT_primitives_generated 估算负载
│       ├── 目标：让 GPU 不需要满频就能完成渲染
│       └── 原理：帧渲染时间 < 14ms 时 GPU 可以降频
├── 各阶段功耗分析
│   ├── CPU 逻辑（游戏逻辑/物理/AI）：~25-30%
│   ├── CPU 渲染（Draw Call/排序）：~10%
│   ├── GPU 渲染（Shader/ROP/Bandwidth）：~40-50%
│   ├── 内存带宽（纹理/FrameBuffer）：~15-20%
│   ├── 屏幕（亮度/刷新率）：~5-10%（OS 管理，游戏可控有限）
│   └── 网络/音频/传感器：~5%
└── 目标设定
    ├── 峰值功耗 < 5W（主流手机散热上限）
    ├── 持续功耗 < 3.5W（30 分钟不降频）
    ├── 帧率稳定 60fps（30 分钟测试）
    └── 设备表面温度 < 43°C（用户舒适阈值）
```

#### 代码实现

**1. 热状态监控系统（C#）**

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ThermalAdaptiveSystem : MonoBehaviour
{
    [Header("监控参数")]
    [SerializeField] private int frameTimeWindow = 180;  // 3秒 @ 60fps
    [SerializeField] private float throttleThresholdMs = 18.5f; // 降频检测阈值
    [SerializeField] private int throttleFrameCount = 30;        // 连续超时帧数

    [Header("品质阶梯")]
    [SerializeField] private QualityTier[] qualityTiers;

    [Header("恢复参数")]
    [SerializeField] private float recoveryHoldTime = 30f; // 恢复前最少保持时间
    [SerializeField] private float recoveryTempThreshold = 38f; // 恢复温度阈值

    private Queue<float> frameTimeHistory = new Queue<float>();
    private int currentTierIndex = 0;
    private float lastTierChangeTime;
    private float currentThrottleScore;

    // 品质阶梯定义
    [System.Serializable]
    public class QualityTier
    {
        public string name;
        public float resolutionScale = 1.0f;
        public float lodDistanceScale = 1.0f;
        public float shadowDistance = 60f;
        public int shadowCascadeCount = 4;
        public bool enableBloom = true;
        public bool enableSSAO = true;
        public int maxParticles = 5000;
        public int targetFrameRate = 60;

        public void Apply()
        {
            // 分辨率缩放
            ScalableBufferManager.ResizeTargets(
                Mathf.RoundToInt(Screen.width * resolutionScale),
                Mathf.RoundToInt(Screen.height * resolutionScale));

            // LOD 距离
            QualitySettings.lodBias = 1.0f / lodDistanceScale;

            // 阴影
            QualitySettings.shadowDistance = shadowDistance;
            QualitySettings.shadowCascades = shadowCascadeCount;

            // 后处理（通过全局 Shader 关键字控制）
            Shader.SetGlobalFloat("_QualityLevel", name.Contains("Ultra") ? 1.0f : 0.5f);

            // 目标帧率
            Application.targetFrameRate = targetFrameRate;

            Debug.Log($"[ThermalAdaptive] Applied tier: {name}");
        }
    }

    void Start()
    {
        // 默认品质阶梯（从高到低）
        if (qualityTiers == null || qualityTiers.Length == 0)
        {
            qualityTiers = new QualityTier[]
            {
                new QualityTier { name = "Ultra", resolutionScale = 1.0f,
                    lodDistanceScale = 1.0f, shadowDistance = 80f,
                    shadowCascadeCount = 4, enableBloom = true, enableSSAO = true,
                    maxParticles = 8000 },
                new QualityTier { name = "High", resolutionScale = 0.9f,
                    lodDistanceScale = 1.2f, shadowDistance = 60f,
                    shadowCascadeCount = 2, enableBloom = true, enableSSAO = true,
                    maxParticles = 5000 },
                new QualityTier { name = "Medium", resolutionScale = 0.8f,
                    lodDistanceScale = 1.5f, shadowDistance = 40f,
                    shadowCascadeCount = 1, enableBloom = true, enableSSAO = false,
                    maxParticles = 3000 },
                new QualityTier { name = "Low", resolutionScale = 0.7f,
                    lodDistanceScale = 2.0f, shadowDistance = 25f,
                    shadowCascadeCount = 1, enableBloom = false, enableSSAO = false,
                    maxParticles = 1500, targetFrameRate = 30 },
            };
        }

        currentTierIndex = 0;
        qualityTiers[0].Apply();
        lastTierChangeTime = Time.time;
    }

    void Update()
    {
        // === 记录帧时间 ===
        float frameTimeMs = Time.unscaledDeltaTime * 1000f;
        frameTimeHistory.Enqueue(frameTimeMs);
        if (frameTimeHistory.Count > frameTimeWindow)
            frameTimeHistory.Dequeue();

        // === 计算降频分数 ===
        currentThrottleScore = ComputeThrottleScore();

        // === 热状态检测 ===
        ThermalState thermalState = GetThermalState();

        // === 决策：是否降级品质 ===
        if (ShouldDowngrade(currentThrottleScore, thermalState))
        {
            Downgrade();
        }
        else if (ShouldUpgrade(thermalState))
        {
            Upgrade();
        }
    }

    float ComputeThrottleScore()
    {
        if (frameTimeHistory.Count < 30) return 0f;

        // 统计最近 N 帧中有多少帧超过了降频阈值
        int overBudgetCount = 0;
        foreach (var ft in frameTimeHistory)
        {
            if (ft > throttleThresholdMs) overBudgetCount++;
        }
        return (float)overBudgetCount / frameTimeHistory.Count;
    }

    bool ShouldDowngrade(float throttleScore, ThermalState thermal)
    {
        // 条件1：降频分数超过 50%（最近 3 秒有一半以上帧超时）
        if (throttleScore > 0.5f) return true;

        // 条件2：热状态为 Fair 或 Serious
        if (thermal >= ThermalState.Fair) return true;

        // 条件3：帧时间方差很大（jitter 严重，可能是间歇性降频）
        if (frameTimeHistory.Count > 60 && ComputeFrameTimeVariance() > 25f)
            return true;

        return false;
    }

    bool ShouldUpgrade(ThermalState thermal)
    {
        if (currentTierIndex == 0) return false;

        // 必须保持当前等级至少 30 秒
        if (Time.time - lastTierChangeTime < recoveryHoldTime) return false;

        // 温度恢复正常 + 降频分数极低
        if (thermal <= ThermalState.Nominal && currentThrottleScore < 0.1f)
            return true;

        return false;
    }

    float ComputeFrameTimeVariance()
    {
        // 计算帧时间的标准差
        float sum = 0f, sumSq = 0f;
        int count = 0;
        foreach (var ft in frameTimeHistory)
        {
            sum += ft;
            sumSq += ft * ft;
            count++;
        }
        float mean = sum / count;
        return (sumSq / count) - (mean * mean);
    }

    void Downgrade()
    {
        if (currentTierIndex < qualityTiers.Length - 1)
        {
            currentTierIndex++;
            qualityTiers[currentTierIndex].Apply();
            lastTierChangeTime = Time.time;
            Debug.LogWarning($"[ThermalAdaptive] ⚠️ Downgrading to {qualityTiers[currentTierIndex].name} " +
                           $"(throttle score: {currentThrottleScore:F2}, thermal: {GetThermalState()})");
        }
    }

    void Upgrade()
    {
        if (currentTierIndex > 0)
        {
            currentTierIndex--;
            qualityTiers[currentTierIndex].Apply();
            lastTierChangeTime = Time.time;
            Debug.Log($"[ThermalAdaptive] ✅ Upgrading to {qualityTiers[currentTierIndex].name}");
        }
    }

    // === 热状态获取（跨平台） ===
    enum ThermalState { Nominal, Fair, Serious, Critical }

    ThermalState GetThermalState()
    {
#if UNITY_IOS
        // iOS: 使用 ProcessInfo.thermalState（需要 Unity 插件桥接）
        // 0=Nominal, 1=Fair, 2=Serious, 3=Critical
        return (ThermalState)GetiOSThermalState();
#elif UNITY_ANDROID
        // Android: 使用 PowerManager.isThermalStatusModerate
        return (ThermalState)GetAndroidThermalStatus();
#else
        return ThermalState.Nominal;
#endif
    }

#if UNITY_IOS
    [System.Runtime.InteropServices.DllImport("__Internal")]
    static extern int GetiOSThermalState();
#endif

#if UNITY_ANDROID
    int GetAndroidThermalStatus()
    {
        // 通过 AndroidJavaClass 调用 PowerManager
        try
        {
            using (var unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer"))
            using (var activity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity"))
            using (var powerManager = activity.Call<AndroidJavaObject>("getSystemService", "power"))
            {
                int status = powerManager.Call<int>("getThermalStatus");
                // 0=None, 1=Light, 2=Moderate, 3=Severe, 4=Critical, 5=Emergency, 6=Shutdown
                return Mathf.Min(status, 3);
            }
        }
        catch { return 0; }
    }
#endif
}
```

**2. 渲染分帧调度器（降低瞬时 GPU 负载）**

```csharp
using UnityEngine;

/// <summary>
/// 渲染分帧调度器：将高开销操作分散到多帧执行
/// 目标：平滑 GPU 负载，避免频率冲高触发降频
/// </summary>
public class FrameSpreadScheduler : MonoBehaviour
{
    [Header("分帧配置")]
    [SerializeField] private int shadowUpdateInterval = 2;     // 每 2 帧更新阴影
    [SerializeField] private int reflectionUpdateInterval = 4; // 每 4 帧更新反射
    [SerializeField] private int envProbeUpdateInterval = 6;   // 每 6 帧更新环境探针

    private int frameCounter = 0;

    void Start()
    {
        // 根据 SoC 等级调整间隔
        SoCTier tier = DetectSoCTier();
        ApplyTierConfig(tier);
    }

    void Update()
    {
        frameCounter++;

        // 阴影更新（每 N 帧一次）
        bool updateShadows = (frameCounter % shadowUpdateInterval == 0);

        // 反射更新（每 N 帧一次）
        bool updateReflections = (frameCounter % reflectionUpdateInterval == 0);

        // 环境探针更新（每 N 帧一次）
        bool updateEnvProbes = (frameCounter % envProbeUpdateInterval == 0);

        // 通过全局变量通知 Shader 是否更新
        Shader.SetGlobalFloat("_ShadowUpdateWeight", updateShadows ? 1.0f : 0.0f);

        // 实际项目中：在这里切换 RenderFeature 的 enable 状态
        // 或通知 Lighting System 跳过本帧的特定 Pass
    }

    enum SoCTier { HighSnapdragon8Gen2, MidRange, LowEnd }

    SoCTier DetectSoCTier()
    {
        string deviceModel = SystemInfo.deviceModel;
        int memory = SystemInfo.systemMemorySize;

        // 简化判断（实际项目需要更详细的设备库）
        if (memory >= 8192 && SystemInfo.graphicsMemorySize >= 4096)
            return SoCTier.HighSnapdragon8Gen2;
        else if (memory >= 6144)
            return SoCTier.MidRange;
        else
            return SoCTier.LowEnd;
    }

    void ApplyTierConfig(SoCTier tier)
    {
        switch (tier)
        {
            case SoCTier.HighSnapdragon8Gen2:
                // 高端 SoC：可以每帧更新
                shadowUpdateInterval = 1;
                reflectionUpdateInterval = 3;
                envProbeUpdateInterval = 6;
                break;
            case SoCTier.MidRange:
                // 中端：适当分帧
                shadowUpdateInterval = 2;
                reflectionUpdateInterval = 4;
                envProbeUpdateInterval = 8;
                break;
            case SoCTier.LowEnd:
                // 低端：最大化分帧
                shadowUpdateInterval = 3;
                reflectionUpdateInterval = 6;
                envProbeUpdateInterval = 12;
                break;
        }
    }
}
```

**功耗预算分配表（以 Snapdragon 8 Gen 2 为基准）**

| 子系统 | 峰值功耗 | 持续功耗预算 | 优化策略 |
|--------|---------|------------|---------|
| GPU 渲染 | 3.5W | 2.0W | 降分辨率 / 简化 Shader / Alpha Test |
| CPU 游戏逻辑 | 2.0W | 1.2W | 多线程 Job System / 减少物理频率 |
| CPU 渲染 | 0.8W | 0.5W | 合批 / SRP Batcher |
| 内存带宽 | 1.5W | 1.0W | 贴图压缩 / 减少RenderTarget切换 |
| 总计 | 7.8W | 4.7W | — |

### ⚡ 实战经验

**温控模型：不是线性退化**

SoC 的温度-频率关系不是线性的，而是阶梯式跳变：
```
温度 45°C → GPU 频率正常
温度 48°C → GPU 降一级频率（-10%）
温度 52°C → GPU 降二级频率（-20%）+ CPU 大核降频
温度 55°C → 全面降频 + 屏幕亮度降低
温度 60°C → 弹出过热警告
```

**关键洞察：降频后功耗反而可以上升**

这是因为降频后帧时间变长，用户在同一场景停留时间更久，总能耗反而可能增加。这就是为什么**主动降级品质比被动等 OS 降频更有效**——你在 OS 降频之前就降低了渲染负载，让 GPU 以中等频率稳定运行，总功耗更低。

**夏天 vs 冬天的差异**

- 冬天（环境 10°C）：手机散热效率高，可能永远不降频
- 夏天（环境 35°C）：散热效率极低，5 分钟就可能降频
- **测试标准**：26°C 恒温室 + 标准手机壳，跑 30 分钟 benchmark
- **极限测试**：35°C 烘箱 + 充电状态，这是最严苛条件

**iOS 的特殊性**

iOS 的热管理比 Android 更激进（因为 iPhone 的散热设计受限于机身厚度）：
- `thermalState` 变为 `.fair` 时，系统已经在后台降低频率
- `.serious` 时会强制降低屏幕亮度
- `.critical` 时可能直接 kill 进程
- **策略**：在 `.fair` 就主动降级品质，不要等到 `.serious`

### 🎯 能力体检清单

| 检查项 | 如果你答不上来… |
|--------|----------------|
| 知道动态功耗 P=CV²f 的含义？ | → SoC 功耗模型基础不足 |
| 理解为什么降频后总功耗可能反而上升？ | → 功耗与帧率关系理解不够 |
| 能设计品质自适应降级阶梯？ | → 自适应品质系统设计经验不足 |
| 知道 iOS/Android 的 Thermal API？ | → 平台热管理 API 不熟 |
| 理解渲染分帧为什么能减少发热？ | → 负载整形原理不清晰 |
| 能说出移动端各子系统的功耗占比？ | → 移动端功耗分析能力不足 |
| 知道夏天/冬天对性能测试的影响？ | → 测试方法论有盲区 |

### 🔗 相关问题

- [手机 GPU 过热分析：Adreno/Mali 热特性差异](optimization/mobile-overheating-gpu-analysis)
- [卡顿与帧率抖动诊断：从 spike 到 sustained](optimization/loading-stall-hitch-spike)
- [GPU 带宽优化：移动端渲染的带宽管理](optimization/gpu-bandwidth-optimization)
