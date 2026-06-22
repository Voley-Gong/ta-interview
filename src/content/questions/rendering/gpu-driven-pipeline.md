---
title: "面试官问：UE5 的 Nanite 原理是什么？不用 Nanite 手游上怎么实现 GPU-Driven Pipeline？"
category: "rendering"
level: 4
tags: ["GPU-Driven", "Nanite", "Mesh Shader", "视锥剔除", "遮挡剔除", "Cluster", "Compute Shader", "UE5"]
hint: "核心：Nanite = Cluster 化 mesh + 可见性剔除全部搬到 GPU + Mesh Shader 输出——理解数据流比背概念重要"
related: ["rendering/forward-plus-cluster", "optimization/drawcall-500-to-100", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官说：「你简历上写了熟悉现代渲染管线。那请你讲讲 UE5 Nanite 的核心原理——不是让你背官网介绍，而是从渲染管线角度分析它解决了什么问题、怎么解决的。

然后追问：如果我们现在做一个手游，引擎是 Unity URP，不用 Nanite，能不能实现 GPU-Driven Pipeline？你会怎么设计？」

追问一：Nanite 的 Cluster 大小为什么是 128 个三角形？太大或太小有什么问题？

追问二：GPU-Driven Pipeline 和传统 CPU-Driven Draw Call 的核心区别是什么？为什么说 Draw Call 数量在 GPU-Driven 下不再是瓶颈？

### ✅ 核心要点

1. **Nanite 的核心问题**：传统管线 CPU 逐 Mesh 提交 Draw Call，CPU 成为瓶颈；且逐 Mesh 粒度的视锥/遮挡剔除太粗糙
2. **三步走架构**：① 离线 Cluster 化 → ② GPU 上做 Cluster 级别可见性剔除 → ③ Mesh Shader / Indirect Draw 渲染
3. **Cluster 是最小调度单位**：把大 Mesh 切成 128 三角形的小 Cluster，每个 Cluster 独立做 LOD 和剔除
4. **GPU-Driven 精髓**：剔除和提交都在 GPU 上完成，CPU 只上传一次 Indirect Draw Arguments，不再逐帧 CPU → GPU 往返
5. **手游可行方案**：用 Compute Shader 做 Cluster 剔除 + `DrawProceduralIndirect`（Unity）或 `vkCmdDrawIndirect`（Vulkan）

### 📖 深度展开

#### 解决思路（从 Nanite 效果倒推架构）

```
最终效果：数百万三角形直接导入引擎，无需手动 LOD，帧率稳定 60fps
              ↑
倒推1：面数这么高为什么不卡？→ 不是每帧都画全部三角形
倒推2：怎么知道画哪些？→ GPU 上对所有 Cluster 做可见性判断
倒推3：Cluster 是什么？→ 离线把 Mesh 切成 128 三角形的小块
倒推4：可见性怎么判断？→ 视锥剔除 + Hi-Z 遮挡剔除，全部在 Compute Shader 中
倒推5：GPU 剔除后怎么告诉渲染管线？→ 生成 Indirect Draw Arguments
倒推6：LOD 怎么做？→ Cluster 级别的 LOD 选择（根据屏幕投影面积选 LOD）
倒推7：数据怎么存？→ 每个 Cluster 有 bounding sphere + LOD 链表
```

#### 知识点拆解（倒推树）

```
GPU-Driven Rendering Pipeline
├── Cluster 化（离线预处理）
│   ├── Mesh 分割策略（128 三角形/cluster，空间连贯性聚类）
│   ├── Cluster 边界计算（Bounding Sphere / Cone for normal culling）
│   ├── LOD 链构建（Cluster Group → 父 Cluster → 合并简化）
│   └── 数据序列化（Position 只存量化坐标，节省带宽）
├── GPU 可见性剔除（运行时 Compute Shader）
│   ├── Frustum Culling（Cluster Bounding Sphere vs 视锥 6 面）
│   ├── Occlusion Culling（Hi-Z Depth Pyramid 采样对比）
│   ├── Backface / Normal Cone Culling（Cluster 法线锥剔除）
│   └── LOD Selection（屏幕空间投影面积 → 选 LOD 级别）
├── Indirect Draw 提交
│   ├── Compute Shader 输出 visible cluster list
│   ├── 生成 DrawIndexedIndirect Arguments（vertex count/instance count/index offset）
│   ├── CPU 零负担：不需要 Readback，GPU 自己生成自己消费
│   └── Mesh Shader 范式（Task Shader 做剔除 → Mesh Shader 输出三角形）
├── 传统管线对比
│   ├── CPU-Driven：CPU 遍历场景 → 逐 Mesh 剔除 → 逐 Mesh 提交 Draw Call
│   ├── GPU-Driven：CPU 上传一次 → GPU Compute 剔除 → GPU Indirect Draw
│   └── 为什么 Draw Call 不再是瓶颈：Indirect Draw 可以一个 call 画数万 Cluster
└── 手游适配
    ├── Unity URP 方案：Compute Buffer + DrawProceduralIndirect
    ├── Vulkan 方案：vkCmdDrawIndirect + Multi-Draw Indirect
    ├── 移动端限制：Mesh Shader 支持有限（需要 Vulkan 1.1+ / Metal2），退而求 Indirect Draw
    └── 带宽优化：Cluster 数据用 StructureBuffer 紧凑排列，减少随机访问
```

#### 代码实现

**Unity URP 下的 GPU-Driven Pipeline 简化实现：**

```csharp
// === CPU 侧：上传 Cluster 数据 + 设置 Indirect Draw ===

using UnityEngine;
using Unity.Collections;

public class GPUDrivenRenderer : MonoBehaviour
{
    // 每个 Cluster 的数据结构
    struct ClusterData
    {
        public Vector4 boundingSphere; // xyz=center, w=radius
        public int indexOffset;        // 在全局 Index Buffer 中的偏移
        public int indexCount;         // 三角形数 * 3
        public int lodLevel;           // LOD 级别
        public int padding;
    }

    private ComputeBuffer clusterBuffer;     // 所有 Cluster 数据
    private ComputeBuffer visibleBuffer;      // 可见 Cluster 列表
    private ComputeBuffer indirectArgsBuffer; // Indirect Draw 参数
    private ComputeShader cullingCS;          // 剔除 Compute Shader

    void InitClusters(Mesh[] meshes)
    {
        // 1. 离线 Cluster 化（简化版：假设每 128 三角形一个 Cluster）
        var clusters = new NativeArray<ClusterData>(meshCount * clusterPerMesh, Allocator.Persistent);
        // ... 填充 bounding sphere, index offset 等 ...

        clusterBuffer = new ComputeBuffer(clusters.Length, System.Runtime.InteropServices.Marshal.SizeOf<ClusterData>());
        clusterBuffer.SetData(clusters);

        visibleBuffer = new ComputeBuffer(clusters.Length, System.Runtime.InteropServices.Marshal.SizeOf<ClusterData>());
        indirectArgsBuffer = new ComputeBuffer(5, sizeof(uint), ComputeBufferType.IndirectArguments);
    }

    void Update()
    {
        // 2. 每帧 GPU 剔除
        cullingCS.SetMatrix("_ViewProjMatrix", Camera.main.projectionMatrix * Camera.main.worldToCameraMatrix);
        cullingCS.SetBuffer(0, "_ClusterBuffer", clusterBuffer);
        cullingCS.SetBuffer(0, "_VisibleBuffer", visibleBuffer);
        cullingCS.SetBuffer(0, "_IndirectArgsBuffer", indirectArgsBuffer);
        cullingCS.Dispatch(0, Mathf.CeilToInt(clusterCount / 64f), 1, 1);

        // 3. Indirect Draw（CPU 不需要知道可见结果）
        Graphics.DrawProceduralIndirect(material, bounds, MeshTopology.Triangles, indirectArgsBuffer);
    }
}
```

```hlsl
// === Compute Shader: GPU Cluster 剔除 ===

#pragma kernel KMain

struct ClusterData
{
    float4 boundingSphere; // xyz=center, w=radius
    uint indexOffset;
    uint indexCount;
    uint lodLevel;
    uint padding;
};

RWStructuredBuffer<ClusterData> _VisibleBuffer  : register(u0);
RWStructuredBuffer<uint>        _IndirectArgs   : register(u1);
StructuredBuffer<ClusterData>   _ClusterBuffer  : register(t0);
Texture2D<float>                _HiZDepthPyramid : register(t1); // Hi-Z 深度金字塔
SamplerState                    _PointClamp;

float4x4 _ViewProjMatrix;
uint _ClusterCount;

groupshared uint g_visibleCount;
groupshared uint g_dispatchArgs[5]; // indexCount, instanceCount, startIndex, baseVertex, startInstance

[numthreads(64, 1, 1)]
void KMain(uint3 dtid : SV_DispatchThreadID, uint3 gtid : SV_GroupThreadID, uint3 gid : SV_GroupID)
{
    bool visible = false;

    if (dtid.x < _ClusterCount)
    {
        ClusterData c = _ClusterBuffer[dtid.x];

        // 1. 视锥剔除
        float4 clipPos = mul(_ViewProjMatrix, float4(c.boundingSphere.xyz, 1.0));
        float radius = c.boundingSphere.w;

        // 简化视锥测试：检查 clip space 边界
        if (clipPos.w > 0 &&
            clipPos.x > -clipPos.w - radius && clipPos.x < clipPos.w + radius &&
            clipPos.y > -clipPos.w - radius && clipPos.y < clipPos.w + radius &&
            clipPos.z > 0 - radius && clipPos.z < clipPos.w + radius)
        {
            // 2. Hi-Z 遮挡剔除
            float2 screenUV = clipPos.xy / clipPos.w * 0.5 + 0.5;
            float depthSample = _HiZDepthPyramid.SampleLevel(_PointClamp, screenUV, 3).r; // 采样粗粒度 LOD
            float clusterDepth = clipPos.z / clipPos.w;

            if (clusterDepth < depthSample + radius) // 在深度前面
            {
                visible = true;
            }
        }
    }

    // 3. 写入可见列表 + 计数
    if (visible)
    {
        uint idx;
        InterlockedAdd(g_visibleCount, 1, idx);
        if (idx < _ClusterCount)
            _VisibleBuffer[idx] = _ClusterBuffer[dtid.x];
    }

    GroupMemoryBarrierWithGroupSync();

    // 4. 组内第一个线程汇总 Indirect Draw 参数
    if (gtid.x == 0 && gid.x == 0)
    {
        // 这里简化：实际需要多组聚合
        _IndirectArgs[0] = g_visibleCount * 128 * 3; // indexCount per cluster
        _IndirectArgs[1] = 1;                          // instanceCount
        _IndirectArgs[2] = 0;                          // startIndex
        _IndirectArgs[3] = 0;                          // baseVertex
        _IndirectArgs[4] = 0;                          // startInstance
    }
}
```

**Nanite vs 传统管线对比表：**

| 维度 | 传统 CPU-Driven | Nanite / GPU-Driven |
|------|----------------|-------------------|
| 剔除粒度 | Mesh 级别 | Cluster（128 三角形）级别 |
| 剔除执行 | CPU | GPU Compute Shader |
| Draw Call | CPU 逐 Mesh 提交 | Indirect Draw（GPU 自生成） |
| LOD | 离线生成有限 LOD | Cluster 级自动 LOD 选择 |
| CPU 瓶颈 | 数千 Mesh 时 CPU 爆了 | CPU 几乎零负担 |
| 手游可行性 | 默认方案 | Unity 用 Indirect Draw 近似 |

### ⚡ 实战经验

- **Cluster 大小的权衡**：128 三角形是 Nanite 经验值——太小则 Cluster 数量爆炸（管理开销和剔除 overhead 增大），太大则剔除粒度粗糙（一个 Cluster 可见就要画全部三角形）
- **Hi-Z 遮挡剔除是关键**：光做视锥剔除不够，城市/室内场景中大量 Cluster 被建筑遮挡。需要维护上一帧的 Depth Pyramid 做本帧剔除（延迟一帧可接受）
- **移动端 Mesh Shader 支持情况**：iOS Metal 2.2+ 支持 Mesh Shader（A15+），Android 需要 Vulkan 1.1+ 且驱动质量参差不齐。手游建议先上 Indirect Draw 方案
- **调试极痛苦**：GPU-Driven 的数据全在 GPU 端，CPU Readback 调试会拖帧。建议做一个可视化 Debug View（把可见 Cluster 的 Bounding Sphere 画出来）
- **不要过度工程化**：如果场景只有几百个 Mesh，传统 CPU-Driven + GPU Instancing 就够了。GPU-Driven 的收益在数千到数万 Mesh 级别

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 Cluster 为什么存在 | 缺少 GPU 调度粒度理解 | 学习 Compute Shader 调度模型 |
| 视锥剔除能说但遮挡剔除不会 | 不了解 Hi-Z Pyramid | 学习 Hi-Z 遮挡剔除算法 |
| Indirect Draw 概念模糊 | 不了解 GPU 自主提交 | 学习 Vulkan/Metal Indirect Draw API |
| LOD 会说但 Cluster 级 LOD 不会 | 不理解 Cluster LOD 链 | 研究 Nanite LOD merge 算法 |
| 不知道手游能不能用 | 缺少移动端 API 支持了解 | 调研移动端 Mesh Shader / Indirect Draw 支持 |

### 🔗 相关问题

- Forward+ 的 Cluster 和 Nanite 的 Cluster 有什么区别？（提示：一个是灯光 Cluster，一个是几何 Cluster）
- 如果不用 Mesh Shader，纯 Compute Shader + Indirect Draw 能做到什么程度？
- UE5 的 Lumen 和 Nanite 是如何协作的？Lumen 的 GI 需要什么几何信息？
