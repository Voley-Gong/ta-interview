---
title: "开放世界大地图流式加载：如何消除卡顿并控制内存？"
category: "optimization"
level: 4
tags: ["开放世界", "流式加载", "SceneStreaming", "内存管理", "LOD", "Addressables", "GPU剔除"]
hint: "核心是把世界切成 Grid 分块，用异步加载 + 双缓冲 + 可预测预加载消除卡顿，用 LOD + Impostor + 远景代理控制同屏渲染量"
related: ["optimization/loading-stall-hitch-spike", "optimization/gpu-memory-budget", "rendering/gpu-driven-pipeline", "optimization/million-grass-rendering"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们在做一款开放世界手游，地图 4km × 4km。现在的问题是：玩家骑坐骑快速移动时，前方区域加载不及时导致画面弹出（pop-in），用同步加载又会有明显卡顿。内存方面，高端机没问题但低端机直接 OOM。给我一套完整的流式加载方案，从场景切分到内存管理到渲染优化都要覆盖。」

### ✅ 核心要点

1. **Grid 分块是基础**：世界切成 Chunk（如 128m × 128m），按距离异步加载/卸载
2. **预测性预加载**：根据玩家速度和朝向，提前加载前方 Chunk（消除 pop-in）
3. **双阶段加载消除卡顿**：先异步加载到内存（不阻塞主线程），再在帧间分批 Instantiate
4. **内存分级策略**：近距全精度 → 中距 LOD → 远距 Impostor → 超远距 Heightmap only
5. **Addressables 是引擎利器**：按 Chunk 打包 AssetBundle，运行时按需加载

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：4km×4km 开放世界 → 快速移动无弹窗 → 低端机不 OOM → 帧率稳定 30fps
                ↑
倒推1：无弹窗 → 必须在玩家到达前加载好前方区域 → 预测性预加载
倒推2：无卡顿 → 加载不能阻塞主线程 → 异步加载 + 分帧实例化
倒推3：不 OOM → 同时驻留内存的 Chunk 必须有限 → 分级 LOD + 及时卸载
倒推4：帧率稳定 → 同屏渲染量必须可控 → GPU Driven + Impostor + 遮挡剔除
倒推5：包体可控 → 资源按 Chunk 打包，不打包不加载 → Addressables
```

#### 知识点拆解（倒推树）

```
开放世界流式加载
├── 世界切分（World Partitioning）
│   ├── Grid Chunk 划分（固定大小 vs 自适应）
│   │   ├── 固定 128m/256m（简单、可预测）
│   │   └── 自适应四叉树（密度高的区域更细）
│   ├── Chunk 内容组织
│   │   ├── 地形高度图（Heightmap）—— 必须常驻或大范围预载
│   │   ├── 静态物体（建筑/岩石/植被）—— 按_chunk_打包
│   │   ├── 导航网格（NavMesh）—— 按_chunk_切片
│   │   └── 光照数据（Lightmap / Light Probe）—— 按需加载
│   └── Chunk 元数据
│       ├── AABB 包围盒（用于可见性判断）
│       ├── 内存预估大小（用于预算管理）
│       └── 依赖关系（共享材质/纹理的引用计数）
├── 加载策略
│   ├── 距离分级
│   │   ├── 0-64m：全精度（Active Zone）
│   │   ├── 64-256m：LOD 1-2（Streaming Zone）
│   │   ├── 256-1024m：Impostor / 远景代理（Background Zone）
│   │   └── 1024m+：不渲染（仅逻辑存在）
│   ├── 预测性预加载
│   │   ├── 基于速度向量预测（velocity × lookAheadTime）
│   │   ├── 基于路径预测（坐骑/载具沿道路行驶）
│   │   └── 基于传送门/快速旅行点
│   └── 优先级队列
│       ├── 玩家正前方 Chunk 优先级最高
│       ├── 视线范围内 Chunk 次之
│       └── 背后 Chunk 优先卸载
├── 异步加载管线
│   ├── Stage 1：AssetBundle 异步加载（Addressables.LoadAssetAsync）
│   │   └── 不阻塞主线程，IO 在后台线程完成
│   ├── Stage 2：分帧实例化
│   │   ├── 每帧 Instantiate 预算（如 2ms 内）
│   │   ├── 大型物体（城堡/山体）优先实例化
│   │   └── 小物体批量实例化（植被/石头）
│   └── Stage 3：渲染就绪
│       ├── Light Probe 注入
│       ├── NavMesh 拼接
│       └── 物理碰撞体启用
├── 内存管理
│   ├── 驻留 Chunk 预算
│   │   ├── 高端机：9×9 Chunk 驻留 ≈ 400MB
│   │   ├── 中端机：7×7 Chunk 驻留 ≈ 200MB
│   │   └── 低端机：5×5 Chunk 驻留 ≈ 120MB
│   ├── LRU 卸载策略
│   │   ├── 距离最远 + 不在视线 → 优先卸载
│   │   ├── 引用计数归零 → Addressables.Release
│   │   └── Resources.UnloadUnusedAssets（谨慎使用，耗时）
│   └── 内存碎片治理
│       └── 定期全量卸载重建（切场景时）
├── 渲染优化（同屏控制）
│   ├── GPU Driven Rendering
│   │   ├── Compute Shader 合批提交（间接绘制）
│   │   ├── GPU 可见性剔除（Hi-Z / Cluster）
│   │   └── 减少 CPU → GPU 提交次数
│   ├── Impostor（广告牌）
│   │   ├── 远距建筑/树木渲染为 2D Sprite
│   │   ├── 预烘焙多角度快照
│   │   └── 1 个 DrawCall 渲染整个远景城镇
│   ├── 遮挡剔除（Occlusion Culling）
│   │   ├── 内置 OC（烘焙，适合室内）
│   │   └── 运行时 OC（GPU Driven Hi-Z，适合开放世界）
│   └── Terrain 优化
│       ├── Heightmap 分块（匹配 Chunk 尺寸）
│       ├── Splat Map 分块（材质混合图）
│       └── 远距地形降采样（顶点密度递减）
└── 制作规范
    ├── Chunk 命名/路径规范（chunk_x{X}_z{Z}）
    ├── 每 Chunk 资源上限（顶点数/贴图大小）
    ├── 植被/小物件用 GPU Instancing（不占 Chunk 实例化预算）
    └── 自动化打包脚本（按 Chunk 生成 AssetBundle）
```

#### 代码实现

**Chunk 管理器核心（Unity C#）：**

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class WorldChunkStreamer : MonoBehaviour
{
    [Header("Chunk Settings")]
    [SerializeField] private int chunkSize = 128;
    [SerializeField] private int activeRadius = 2;      // 全精度半径（chunk 数）
    [SerializeField] private int streamRadius = 4;      // 流式加载半径
    [SerializeField] private int unloadRadius = 5;      // 超出则卸载

    [Header("Performance Budget")]
    [SerializeField] private float instantiateBudgetMs = 2f; // 每帧实例化预算
    [SerializeField] private int maxConcurrentLoads = 3;     // 最大并发加载数

    private Dictionary<Vector2Int, ChunkState> _chunks = new();
    private Queue<ChunkLoadTask> _loadQueue = new();
    private int _activeLoadCount;

    private struct ChunkState
    {
        public Vector2Int coord;
        public bool loaded;
        public GameObject root;
        public AsyncOperationHandle handle;
    }

    private struct ChunkLoadTask
    {
        public Vector2Int coord;
        public int priority; // 越小越优先
    }

    void Update()
    {
        Vector2Int playerChunk = WorldToChunkCoord(transform.position);
        UpdateChunkLoading(playerChunk);
        ProcessLoadQueue();
        UnloadDistantChunks(playerChunk);
    }

    /// <summary>
    /// 根据玩家位置更新需要加载的 Chunk
    /// </summary>
    void UpdateChunkLoading(Vector2Int playerChunk)
    {
        // 预测向量：基于玩家速度
        Vector3 velocity = GetComponent<Rigidbody>().velocity;
        Vector2Int predictDir = new Vector2Int(
            Mathf.RoundToInt(Mathf.Clamp(velocity.x, -1, 1)),
            Mathf.RoundToInt(Mathf.Clamp(velocity.z, -1, 1))
        );

        for (int dx = -streamRadius; dx <= streamRadius; dx++)
        {
            for (int dz = -streamRadius; dz <= streamRadius; dz++)
            {
                Vector2Int coord = playerChunk + new Vector2Int(dx, dz);
                if (_chunks.ContainsKey(coord) && _chunks[coord].loaded) continue;

                // 优先级计算：切比雪夫距离 + 朝向加权
                int chebyshev = Mathf.Max(Mathf.Abs(dx), Mathf.Abs(dz));
                int directionBonus = (dx * predictDir.x + dz * predictDir.y) > 0 ? -2 : 0;
                int priority = chebyshev + directionBonus;

                if (chebyshev <= streamRadius)
                {
                    EnqueueChunkLoad(coord, priority);
                }
            }
        }
    }

    void EnqueueChunkLoad(Vector2Int coord, int priority)
    {
        if (_chunks.TryGetValue(coord, out var state) && state.loaded) return;

        _loadQueue.Enqueue(new ChunkLoadTask { coord = coord, priority = priority });
    }

    /// <summary>
    /// 分帧处理加载队列
    /// </summary>
    void ProcessLoadQueue()
    {
        if (_activeLoadCount >= maxConcurrentLoads) return;

        float frameStart = Time.realtimeSinceStartup;

        while (_loadQueue.Count > 0 && _activeLoadCount < maxConcurrentLoads)
        {
            var task = _loadQueue.Dequeue();
            StartCoroutine(LoadChunkAsync(task.coord));

            // 检查帧预算（加载请求本身很轻，重的是实例化）
            if ((Time.realtimeSinceStartup - frameStart) * 1000f > instantiateBudgetMs)
                break;
        }
    }

    IEnumerator LoadChunkAsync(Vector2Int coord)
    {
        _activeLoadCount++;
        string address = $"chunks/chunk_{coord.x}_{coord.y}";

        var handle = Addressables.LoadAssetAsync<GameObject>(address);
        yield return handle;

        if (handle.Status != AsyncOperationStatus.Succeeded)
        {
            Debug.LogWarning($"[Streamer] Failed to load chunk: {address}");
            _activeLoadCount--;
            yield break;
        }

        // 分帧实例化（大型 Chunk 可能需要拆成多帧）
        GameObject chunkRoot = Instantiate(handle.Result);
        chunkRoot.name = $"Chunk_{coord.x}_{coord.y}";
        chunkRoot.transform.position = new Vector3(
            coord.x * chunkSize, 0, coord.y * chunkSize);

        _chunks[coord] = new ChunkState
        {
            coord = coord,
            loaded = true,
            root = chunkRoot,
            handle = handle
        };

        _activeLoadCount--;
    }

    /// <summary>
    /// 卸载距离过远的 Chunk
    /// </summary>
    void UnloadDistantChunks(Vector2Int playerChunk)
    {
        List<Vector2Int> toRemove = new();

        foreach (var kvp in _chunks)
        {
            int dist = Mathf.Max(
                Mathf.Abs(kvp.Key.x - playerChunk.x),
                Mathf.Abs(kvp.Key.y - playerChunk.y));

            if (dist > unloadRadius)
            {
                // 先卸载资源再释放引用
                if (kvp.Value.root != null)
                    Destroy(kvp.Value.root);
                if (kvp.Value.handle.IsValid())
                    Addressables.Release(kvp.Value.handle);

                toRemove.Add(kvp.Key);
            }
        }

        foreach (var key in toRemove)
            _chunks.Remove(key);
    }

    Vector2Int WorldToChunkCoord(Vector3 worldPos)
    {
        return new Vector2Int(
            Mathf.FloorToInt(worldPos.x / chunkSize),
            Mathf.FloorToInt(worldPos.z / chunkSize));
    }
}
```

**内存预算分级表：**

| 设备等级 | Active Radius | Stream Radius | Unload Radius | 驻留 Chunk 数 | 预估内存 | 目标帧率 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 高端（8Gen3/A17） | 3 (7×7) | 5 | 6 | 49 | ~450MB | 60fps |
| 中端（8Gen1/A14） | 2 (5×5) | 4 | 5 | 25 | ~220MB | 30-60fps |
| 低端（855/A10） | 1 (3×3) | 3 | 4 | 9 | ~90MB | 30fps |

### ⚡ 实战经验

- **地形高度图不能按 Chunk 卸载**：地形是连续的，如果高度图跟着 Chunk 卸载，远处山体会突然消失。正确做法是高度图全量常驻或大范围缓存，只分块卸载地表植被和建筑
- **NavMesh 拼接是隐藏成本**：每个 Chunk 有自己的 NavMesh 片段，Chunk 加载后需要 `NavMesh.AddNavMeshData()` 拼接，卸载时移除——忘记移除会导致内存泄漏
- **快速旅行（传送）是极端 case**：传送时全部旧 Chunk 卸载 + 全部新 Chunk 加载，需要专门的 Loading Screen 过渡，不能用流式加载逻辑处理
- **Addressables 的内存碎片**：反复加载/卸载不同大小的 Bundle 会导致 Mono 堆碎片化，建议每 10-15 分钟在安全场景（如进副本）做一次全量卸载重建
- **植被不进 Chunk 包**：草、石头等海量小物件用 GPU Instancing + Compute Shader 独立管理，不走 Chunk 实例化管线——否则实例化预算全被草吃掉了
- **Profile 工具链**：Frame Debugger（验证 DrawCall） + Memory Profiler（内存快照对比） + Addressables Event Viewer（加载耗时可视化）三件套缺一不可

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 玩家快速移动时前方弹出 | 预测性预加载 / 速度向量预测 | 学预测算法 + 加载优先级队列 |
| 加载 Chunk 时掉帧 | 同步加载 vs 异步 + 分帧实例化 | 学 Addressables 异步 API + 帧预算分配 |
| 低端机 OOM | 驻留 Chunk 预算 / 设备分级 | 学设备性能分级 + 动态调整策略 |
| 远处山体消失 | 高度图常驻策略 | 学 Terrain 系统 + 分块 Heightmap |
| 百万级物件帧率崩溃 | GPU Driven Rendering | 学 Compute Shader 间接绘制 + Hi-Z 剔除 |

### 🔗 相关问题

- 如何在玩家传送到未加载区域时平滑过渡？（提示：Loading Screen + 强制同步加载最小范围 Chunk）
- 多人联机下流式加载怎么同步？（提示：Server 权威 + 客户端按需加载 + 位置插值补偿）
- 移动端 Addressables 热更方案怎么设计？（提示：CDN + 版本号比对 + 增量 Bundle 下载）
