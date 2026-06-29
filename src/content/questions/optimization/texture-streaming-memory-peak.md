---
title: "手游纹理流式加载内存峰值：加载场景时OOM崩溃，如何定位和消除Texture Streaming的内存尖峰？"
category: "optimization"
level: 3
tags: ["Texture Streaming", "内存峰值", "OOM", "Mipmap", "内存预算", "Asset Loading", "移动端", "Unity"]
hint: "罪魁祸首不是纹理总量而是峰值——场景切换时旧纹理还没卸载、新纹理全分辨率涌入，Memory Budget 和 Mipmap Streaming 联手削峰"
related: ["optimization/texture-streaming-mipmap-prefetch", "optimization/gpu-memory-budget", "optimization/open-world-streaming-loading", "optimization/mobile-package-size-2gb-to-500mb"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的开放世界手游，玩家从 A 区域传送到 B 区域时，iOS 上偶发 OOM 崩溃（内存峰值达到 1.8GB）。Profiler 显示纹理占了 900MB+，但场景设计总纹理量只有 600MB。传送瞬间有一段内存交叠期：A 区域纹理还在内存里没卸掉，B 区域全分辨率纹理已经涌入。怎么定位和解决这个峰值问题？」

这是做开放世界、大型场景项目（原神、星穹铁道、白夜极光等）的经典性能面试题——考察的是对 Unity 纹理加载生命周期、Memory Budget 策略和内存峰值管理的深度理解。

### ✅ 核心要点

1. **峰值 > 总量是 OOM 的真正元凶**：不是纹理总量超标，而是加载/卸载时序导致瞬时峰值远超稳态
2. **Texture Streaming 的 Memory Budget 机制**：Unity 的 Mipmap Streaming 只加载可见像素需要的 Mipmap 级别，但 Budget 限制是全局的，不能防止单次场景切换的瞬时涌入
3. **分帧加载（Time-sliced Loading）**：把 B 区域的纹理按优先级分批加载，不要一帧全加载
4. **强制降级预加载（Preload Low-Mip）**：先加载低 Mipmap 级别的纹理（省内存），再逐步提升
5. **引用计数管理（Reference Counting）**：确保 A 区域纹理在 B 区域加载前就开始异步卸载
6. **内存水位线监控（Memory Watermark）**：运行时持续监控 RSS / Footprint，超阈值触发紧急纹理卸载

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
问题：传送 A→B 瞬间内存 1.8GB → OOM
        ↑
分析1：为什么峰值 > 总量？
  ├── A 区域纹理还没被卸载（Resources.UnloadUnusedAssets 需要一帧延迟）
  ├── B 区域纹理全分辨率一次性涌入（Resources.Load / Addressables 加载过快）
  └── 两者叠加 → 峰值 = A纹理 + B纹理 > 1.5GB
        ↑
解决1：错开加载/卸载时序
  ├── Step 1：传送触发 → 先异步卸载 A 区域纹理（分帧，3-5帧完成）
  ├── Step 2：加载 B 区域纹理的低 Mipmap 版本（每张只加载 64×64）
  ├── Step 3：逐步提升 B 区域纹理 Mipmap（按距离/可见性排序）
  └── 峰值 = max(A纹理递减, B纹理递增) ≈ 总量 × 0.6
        ↑
解决2：Texture Streaming Memory Budget 控制
  ├── 全局 Budget 设为 400MB（移动端安全值）
  ├── 超出 Budget 时自动降低远距离纹理 Mipmap
  └── 但注意：Budget 不控制「纹理加载过程」只控制「最终驻留 Mipmap」
        ↑
解决3：Addressables 分组策略
  ├── 按区域分组（A组 / B组），同组一起加载/卸载
  ├── 标记为「预加载」和「按需加载」两级
  └── 释放时调 Addressables.Release → 触发 GC
        ↑
解决4：运行时内存监控
  ├── Profiler.GetTotalAllocatedMemoryLong() → GC Heap
  ├── UnityEngine.Profiling.Profiler.GetRuntimeMemorySize() → 纹理内存
  ├── 超过水位线 → 强制 Resources.UnloadUnusedAssets()
  └── 极端情况 → 主动降低全局 Texture Quality（QualitySettings.masterTextureLimit）
```

#### 知识点拆解（倒推树）

```
纹理流式加载内存峰值
├── 峰值产生原因分析
│   ├── 加载-卸载时序交叠
│   │   ├── Resources.UnloadUnusedAssets 是同步的（卡帧！）
│   │   ├── Addressables.Release 是异步的（但 GC 延迟不可控）
│   │   └── 新资源加载速度 > 旧资源释放速度 → 峰值积压
│   ├── Mipmap Streaming 的盲区
│   │   ├── Streaming 只控制「驻留的 Mipmap 级别」
│   │   ├── 加载时先全分辨率载入 → 再降级 → 加载瞬间就是峰值
│   │   └── 需要配合 QualitySettings.masterTextureLimit 或 Preload Low-Mip
│   ├── AssetBundle 引用计数问题
│   │   ├── 同一个 AB 被多个对象引用 → 引用计数不归零 → 不释放
│   │   ├── 静态引用（static 变量）阻止 GC
│   │   └── 事件订阅未取消导致内存泄漏
│   └── iOS vs Android 差异
│       ├── iOS 没有 Swap，OOM 直接 crash（Jetsam 机制）
│       ├── Android 有 Low Memory Killer（LMK），优先杀后台
│       └── iOS 的 footprint（脏页）比 RSS 更关键
├── Mipmap Streaming 深度
│   ├── Unity Mipmap Streaming System
│   │   ├── Memory Budget（MB）→ 全局上限
│   │   ├── Mipmap Streaming Priority → 哪些纹理优先保留高 Mipmap
│   │   ├── Mipmap Streaming Padding → UV padding 安全边距
│   │   └── 只影响 Texture2D，不影响 RenderTexture / Cubemap
│   ├── 纹理加载策略
│   │   ├── 初始 Mipmap Level = max(可用 Mipmap, Budget 控制后的 Level)
│   │   ├── 摄像机可见性触发 → 计算需要的 Mipmap 级别
│   │   └── 不可见 → 降级到最低 Mipmap（1×1）但不卸载纹理
│   └── 限制
│       ├── 不能防止「加载过程」的峰值（只管驻留状态）
│       ├── UI 纹理默认不参与 Streaming（需要手动开启）
│       └── 动态 Atlas 的 Streaming 支持有限
├── 场景切换优化策略
│   ├── Phase 1：卸载阶段（Unload Phase）
│   │   ├── 触发 Addressables.Release（异步引用计数递减）
│   │   ├── 等待 2-3 帧让 GC 完成
│   │   ├── 可选：Resources.UnloadUnusedAssets（同步，有卡帧风险）
│   │   └── 目标：释放 60-70% 的旧区域纹理内存
│   ├── Phase 2：低分辨率预加载（Low-Mip Preload）
│   │   ├── 加载所有 B 区域纹理但强制 Mipmap Level = 高级别（如 4 = 64×64）
│   │   ├── 此时每张纹理只占极小内存（256B ~ 16KB）
│   │   └── 玩家看到 B 区域但模糊（可叠加雾效遮盖）
│   ├── Phase 3：逐步提升分辨率（Progressive Enhancement）
│   │   ├── 按距离/优先级逐步降低 Mipmap Level（提高分辨率）
│   │   ├── 每帧只提升 N 张纹理（分帧）
│   │   └── Memory Budget 确保总量不超标
│   └── Phase 4：稳态（Steady State）
│       └── 所有可见纹理达到预算允许的最高 Mipmap
├── Addressables 分组策略
│   ├── 按「场景/区域」分组（RegionGroup_A, RegionGroup_B）
│   ├── 按优先级分层
│   │   ├── Critical：地形、角色材质（预加载，不允许 Streaming 降级）
│   │   ├── Normal：建筑、植被（正常 Streaming）
│   │   └── Lazy：远距 LOD 纹理（按需加载）
│   └── 同一 AssetBundle 内的纹理一起加载
│       └── 避免过大的 AB（> 50MB），否则单次加载峰值高
├── 运行时内存监控
│   ├── iOS footprint 监控
│   │   ├── mach_task_basic_info.resident_size（RSS）
│   │   ├── phys_footprint（iOS 真正的 OOM 指标）
│   │   └── 超过 ~1.5GB（iPhone X 及以下）→ 随时可能 OOM
│   ├── Android 内存监控
│   │   ├── Debug.getNativeHeapAllocatedSize()
│   │   ├── Graphics 内存：Debug.GetGraphicsMemory()
│   │   └── LMK 阈值因设备而异
│   └── 紧急降级
│       ├── 超过 Red Line → 强制 UnloadUnusedAssets
│       ├── 超过 Orange Line → 降全局 Texture Quality（masterTextureLimit = 1）
│       └── 超过 Yellow Line → 停止新的纹理加载请求
└── 进阶方案
    ├── 自研纹理加载器（绕过 Resources / Addressables）
    │   ├── 直接用 UnityWebRequest 从本地加载 PNG/ASTC → Texture2D
    │   ├── 完全控制加载时序和内存分配
    │   └── 代价：失去 Unity 的 AB 缓存和依赖管理
    ├── GPU Memory Pool（纹理池化）
    │   ├── 预分配一大块 Texture Atlas
    │   └── 子纹理在 Atlas 中 blit，避免单独分配
    └── Memory Mapped Textures
        ├── 直接从磁盘 mmap 纹理数据（不经过托管堆）
        └── iOS 的 mmap → 系统按需分页，物理内存可被系统回收
```

#### 代码实现

**分阶段场景切换管理器：**

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;
using UnityEngine.Profiling;

public class SceneTransitionMemoryManager : MonoBehaviour
{
    [Header("阶段配置")]
    [SerializeField] private float unloadWaitFrames = 3;       // Phase 1 等待帧数
    [SerializeField] private int preloadMipBias = 4;            // Phase 2 初始降级 Mipmap
    [SerializeField] private int texturesPerFrame = 8;          // Phase 3 每帧提升多少纹理
    [SerializeField] private float steadyStateDelay = 2f;       // Phase 3 总时长上限

    [Header("内存水位线 (MB)")]
    [SerializeField] private long yellowLine = 1200;   // 停止新加载
    [SerializeField] private long orangeLine = 1500;   // 降级全局 Texture Quality
    [SerializeField] private long redLine = 1700;      // 紧急 Unload

    private List<AsyncOperationHandle> _currentSceneHandles = new();
    private List<AsyncOperationHandle> _newSceneHandles = new();

    /// <summary>
    /// 执行场景切换（4 阶段管线）
    /// </summary>
    public IEnumerator TransitionScene(AssetReferenceScene newSceneRef)
    {
        // === Phase 1: 卸载旧场景纹理 ===
        Debug.Log("[SceneTransition] Phase 1: Unloading old scene textures...");
        foreach (var handle in _currentSceneHandles)
        {
            if (handle.IsValid()) Addressables.Release(handle);
        }
        _currentSceneHandles.Clear();

        // 等待 GC 完成（分帧等待，避免单帧卡顿）
        for (int i = 0; i < unloadWaitFrames; i++)
        {
            yield return null;
            // 检查内存是否回落
            long mem = GetCurrentMemoryMB();
            if (mem < yellowLine * 0.7f) break; // 已经够低了，提前结束
        }

        // === Phase 2: 低分辨率预加载 ===
        Debug.Log("[SceneTransition] Phase 2: Low-Mip preload...");
        QualitySettings.globalTextureMipmapLimit = preloadMipBias; // 全局强制降级

        var loadHandle = Addressables.LoadAssetAsync<GameObject>(newSceneRef);
        _newSceneHandles.Add(loadHandle);
        yield return loadHandle;

        Instantiate(loadHandle.Result);

        // === Phase 3: 逐步提升分辨率 ===
        Debug.Log("[SceneTransition] Phase 3: Progressive enhancement...");
        float phase3Start = Time.realtimeSinceStartup;
        int currentLimit = preloadMipBias;

        while (currentLimit > 0)
        {
            // 检查内存水位
            long mem = GetCurrentMemoryMB();
            if (mem > orangeLine)
            {
                Debug.LogWarning($"[SceneTransition] 内存 {mem}MB 超过 Orange Line，暂停提升");
                yield return new WaitForSeconds(0.5f);
                continue;
            }

            // 每帧降低 Mipmap Bias（提升 1 级分辨率）
            currentLimit--;
            QualitySettings.globalTextureMipmapLimit = currentLimit;
            yield return new WaitForSeconds(0.1f); // 给 Streaming 系统时间处理
        }

        QualitySettings.globalTextureMipmapLimit = 0; // 完全恢复
        Debug.Log("[SceneTransition] Phase 4: Steady state reached");

        // 移动 handle 引用
        _currentSceneHandles = new List<AsyncOperationHandle>(_newSceneHandles);
        _newSceneHandles.Clear();
    }

    /// <summary>
    /// 运行时内存监控（每帧检查）
    /// </summary>
    void Update()
    {
        long mem = GetCurrentMemoryMB();

        if (mem > redLine)
        {
            Debug.LogError($"[Memory] RED LINE {mem}MB → Emergency unload!");
            Resources.UnloadUnusedAssets();
            System.GC.Collect();
        }
        else if (mem > orangeLine)
        {
            Debug.LogWarning($"[Memory] ORANGE LINE {mem}MB → Degrading texture quality");
            if (QualitySettings.globalTextureMipmapLimit < 2)
                QualitySettings.globalTextureMipmapLimit = 2;
        }
    }

    long GetCurrentMemoryMB()
    {
        // Unity Profiler 获取已分配内存（包括 native）
        long allocated = Profiler.GetTotalAllocatedMemoryLong();
        long reserved = Profiler.GetTotalReservedMemoryLong();
        // 对于移动端，reserved 更接近系统看到的 footprint
        return reserved / (1024 * 1024);
    }
}
```

**纹理引用计数管理器：**

```csharp
using System.Collections.Generic;
using UnityEngine;

public class TextureReferenceManager : MonoBehaviour
{
    public static TextureReferenceManager Instance { get; private set; }

    // 纹理路径 → 引用计数
    private Dictionary<string, int> _refCounts = new();
    // 纹理路径 → 已加载的 Texture2D
    private Dictionary<string, Texture2D> _loadedTextures = new();

    void Awake() { Instance = this; }

    public Texture2D LoadTexture(string path)
    {
        if (_loadedTextures.TryGetValue(path, out var tex))
        {
            _refCounts[path]++;
            return tex;
        }

        // 新加载：先以低 Mipmap 加载
        var texData = Resources.Load<Texture2D>(path);
        if (texData == null) return null;

        _loadedTextures[path] = texData;
        _refCounts[path] = 1;
        return texData;
    }

    public void ReleaseTexture(string path)
    {
        if (!_refCounts.ContainsKey(path)) return;

        _refCounts[path]--;
        if (_refCounts[path] <= 0)
        {
            // 引用归零 → 标记为可释放
            // 不立即释放！延迟 2 秒，避免频繁加载/卸载
            StartCoroutine(DelayedRelease(path, 2f));
        }
    }

    System.Collections.IEnumerator DelayedRelease(string path, float delay)
    {
        yield return new WaitForSeconds(delay);

        // 二次检查：延迟期间可能又被引用了
        if (_refCounts.GetValueOrDefault(path, 0) <= 0)
        {
            var tex = _loadedTextures[path];
            Resources.UnloadAsset(tex);
            _loadedTextures.Remove(path);
            _refCounts.Remove(path);
            Debug.Log($"[TexRefMgr] Released texture: {path}");
        }
    }

    /// <summary>
    /// 紧急释放所有引用计数为 0 的纹理
    /// </summary>
    public int EmergencyReleaseAll()
    {
        int released = 0;
        var keys = new List<string>(_loadedTextures.Keys);
        foreach (var path in keys)
        {
            if (_refCounts.GetValueOrDefault(path, 0) <= 1)
            {
                var tex = _loadedTextures[path];
                Resources.UnloadAsset(tex);
                _loadedTextures.Remove(path);
                _refCounts.Remove(path);
                released++;
            }
        }
        Debug.Log($"[TexRefMgr] Emergency released {released} textures");
        return released;
    }

    /// <summary>
    /// 获取当前纹理总内存
    /// </summary>
    public long GetTotalTextureMemory()
    {
        long total = 0;
        foreach (var kv in _loadedTextures)
        {
            if (kv.Value != null)
            {
                total += UnityEngine.Profiling.Profiler.GetRuntimeMemorySizeLong(kv.Value);
            }
        }
        return total;
    }
}
```

**Mipmap Streaming 预加载工具：**

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using System.IO;

public class TextureStreamingPreloadBaker : EditorWindow
{
    [MenuItem("Tools/TA/Texture Streaming Preload Baker")]
    static void Open() => GetWindow<TextureStreamingPreloadBaker>("Streaming Baker");

    void OnGUI()
    {
        if (GUILayout.Button("分析场景纹理总内存"))
        {
            AnalyzeSceneTextures();
        }
        if (GUILayout.Button("模拟 Mipmap Streaming 降级效果"))
        {
            SimulateStreamingDowngrade();
        }
    }

    void AnalyzeSceneTextures()
    {
        var textures = FindObjectsOfType<MeshRenderer>();
        var texSet = new HashSet<Texture2D>();

        foreach (var r in textures)
        {
            foreach (var mat in r.sharedMaterials)
            {
                if (mat == null) continue;
                foreach (var prop in mat.GetTexturePropertyNames())
                {
                    var tex = mat.GetTexture(prop) as Texture2D;
                    if (tex != null) texSet.Add(tex);
                }
            }
        }

        long totalFull = 0;
        long totalMip4 = 0;
        long totalMip7 = 0;

        foreach (var tex in texSet)
        {
            int size = tex.width * tex.height * (tex.format == TextureFormat.ASTC_6x6 ? 1 : 4);
            totalFull += size;
            // Mip level 4: 1/16 的大小
            totalMip4 += size / 16;
            // Mip level 7: 1/128 的大小
            totalMip7 += size / 128;
        }

        Debug.Log($"[纹理分析] 场景纹理统计：");
        Debug.Log($"  纹理数量: {texSet.Count}");
        Debug.Log($"  全分辨率: {totalFull / 1024 / 1024}MB");
        Debug.Log($"  Mip Level 4 (1/16): {totalMip4 / 1024 / 1024}MB");
        Debug.Log($"  Mip Level 7 (1/128): {totalMip7 / 1024 / 1024}MB");
        Debug.Log($"  峰值估算 (Full→Mip4 过渡): {(totalFull + totalMip4) / 2 / 1024 / 1024}MB");
    }

    void SimulateStreamingDowngrade()
    {
        var textures = FindObjectsOfType<MeshRenderer>();
        var texSet = new HashSet<Texture2D>();

        foreach (var r in textures)
        {
            foreach (var mat in r.sharedMaterials)
            {
                if (mat == null) continue;
                foreach (var prop in mat.GetTexturePropertyNames())
                {
                    var tex = mat.GetTexture(prop) as Texture2D;
                    if (tex != null) texSet.Add(tex);
                }
            }
        }

        // 逐级降低 Mipmap，观察内存变化
        for (int mipBias = 0; mipBias <= 7; mipBias++)
        {
            QualitySettings.globalTextureMipmapLimit = mipBias;
            System.Threading.Thread.Sleep(200); // 等 Streaming 系统处理

            long mem = UnityEngine.Profiling.Profiler.GetTotalReservedMemoryLong();
            Debug.Log($"  Mipmap Limit = {mipBias} → Reserved Memory = {mem / 1024 / 1024}MB");
        }

        QualitySettings.globalTextureMipmapLimit = 0; // 恢复
    }
}
#endif
```

**峰值削减策略对比表：**

| 策略 | 峰值削减 | 实现难度 | 卡顿风险 | 适用场景 |
|------|----------|----------|----------|----------|
| 分帧加载 | 40-50% | 中 | 低 | 所有项目 |
| 低 Mip 预加载 | 60-70% | 中 | 中（初始模糊） | 开放世界 |
| Addressables 分组释放 | 30-40% | 低 | 低 | 场景切换 |
| 引用计数 + 延迟释放 | 20-30% | 高 | 低 | 长生命周期 |
| 内存水位线紧急降级 | 应急 | 中 | 高（掉帧） | OOM 防护 |
| 自研 Mmap 加载器 | 70-80% | 极高 | 低 | 旗舰项目 |

### ⚡ 实战经验

- **先测稳态内存，再测峰值内存**：很多团队只测稳态（角色站在场景中不动），但 OOM 往往发生在场景切换/快速移动的瞬间。用 Instruments（iOS）或 Android Profiler 录制传送操作的内存曲线
- **iOS phys_footprint 是真正的 OOM 判据**：不是 RSS，不是 Virtual Memory，是 phys_footprint。Unity Profiler 显示的 Reserved Memory ≈ footprint，但差异可达 200-300MB（Unity 不统计的部分）
- **Resources.UnloadUnusedAssets 是双刃剑**：能立即释放内存，但同步操作会卡 50-200ms。在 Loading Screen 遮盖下调用是安全的，在战斗中调用是灾难
- **ASTC 压缩纹理的 Streaming 行为不同**：ASTC 6×6 每个纹理块固定 0.5625 bytes/pixel，Mipmap 降级的内存收益不如未压缩纹理明显。但 ASTC 的加载 I/O 更快，总体仍然推荐
- **UI 纹理不参与 Streaming**：默认情况下 Canvas 使用的纹理不启用 Mipmap（UI 不需要 Mipmap），所以 Streaming 系统不管它们。大量 UI Atlas 是隐形内存大户
- **Addressables 的 Release 不立即释放**：调用 Release 后引用计数递减，但底层 AssetBundle 和 GC 回收有延迟。手动调用 `Resources.UnloadUnusedAssets()` 可以加速，但有卡顿代价
- **场景内传送也要处理**：同一个大场景中从东边飞到西边，虽然没换场景，但新区域的纹理加载同样会造成峰值。需要把分帧加载逻辑做成通用的「区域加载器」

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道为什么 OOM（总量够但峰值超标） | 加载-卸载时序交叠 | 学 Unity 资源加载生命周期 |
| Texture Streaming 设了 Budget 但没用 | Streaming 只管驻留不管加载过程 | 学 Streaming 的工作原理和限制 |
| 传送时卡 200ms | UnloadUnusedAssets 同步卡顿 | 学异步卸载 + 分帧策略 |
| iOS 比 Android 更容易 OOM | phys_footprint 机制 | 学 iOS 内存管理和 Jetsam |
| 不知道怎么测峰值 | Profiler 使用 | 学 Unity Memory Profiler + Xcode Instruments |
| Addressables Release 后内存没降 | GC 延迟和引用泄漏 | 学引用计数管理和 GC 触发 |

### 🔗 相关问题

- 如何在不卡顿的情况下做场景内的无缝流式加载？（提示：分块加载 + 异步双缓冲）
- ASTC vs ETC2 在 Texture Streaming 下的内存表现差异？（提示：ASTC 块大小固定，降级收益不同）
- 如何在 Android 上做低内存设备适配？（提示：QualitySettings + 设备分级 + 动态 Budget）
- 如果 Texture Streaming Budget 设得很低，画面会怎么劣化？有什么补偿手段？（提示：远处纹理糊 → 加雾效/DOF 遮盖）
