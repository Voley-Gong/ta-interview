---
title: "团战60人同屏，角色不是Fill Rate瓶颈而是顶点处理爆了，怎么排查和优化？"
category: "optimization"
level: 4
tags: ["顶点处理", "同屏角色", "骨骼动画", "GPU Profiling", "LOD", "手游优化"]
hint: "顶点瓶颈不是砍面数这么简单——骨骼计算、Shader复杂度、Morph Target都可能是真凶"
related: ["optimization/vertex-bound-bottleneck", "optimization/skinned-mesh-vertex-animation-cost", "technical-art/skeletal-animation-precision-compression"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款 MOBA，团战时同屏最多 60 个角色。在骁龙 8 Gen 2 上测试，帧率从 60 掉到 38。我用了 Adreno Profiler 查，发现不是 Fill Rate 瓶颈（Overdraw 只有 2.1x），也不是带宽瓶颈，而是 **顶点处理阶段（Vertex Processing）占了 GPU 时间的 47%**。

角色平均 15 万面，有 3 套 LOD，最低 LOD 也有 3 万面。每个角色带骨骼动画 + Morph Target（面部表情）。

给我一套完整的排查和优化方案，目标 60 人同屏稳 60 帧。」

补充约束：
- 角色品质不能明显下降（这是 MOBA 的卖点）
- 面部表情不能砍（有特写镜头）

### ✅ 核心要点

1. **先拆顶点处理开销**：顶点变换（骨骼蒙皮）、Vertex Shader 指令、Morph Target 插值，三者的占比要分开
2. **LOD 切换策略**：团战时不可能 60 人都用 LOD0，需要基于距离 + 屏幕占比的动态 LOD
3. **骨骼优化**：减少骨骼数量、GPU 蒙皮 vs CPU 蒙皮、离屏角色降低骨骼更新频率
4. **Morph Target 优化**：远距离角色关掉 Morph Target，或烘焙到顶点色
5. **Shader 复杂度**：Vertex Shader 中的逐顶点计算（光照、风、波动）要分级
6. **实例化 + 合批**：相同角色用 GPU Instancing 减少顶点重复提交

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：60人同屏顶点处理从 47% 降到 ≤20%（约 3.3ms → ≤1.4ms）
           ↑
倒推1：47% 的顶点开销拆解 → 骨骼蒙皮 / VS指令 / Morph Target 各占多少？
倒推2：60人中多少人需要最高品质？→ 团战镜头附近 5-8 人用 LOD0，其余降级
倒推3：骨骼蒙皮开销能否降低？→ GPU蒙皮 + 骨骼数量削减 + 远距离降频更新
倒推4：Morph Target 是否所有角色都需要？→ 仅镜头中心角色开启
倒推5：Vertex Shader 逐顶点计算能否简化？→ LOD 越低，VS 越简单
倒推6：Draw Call 排列是否最优？→ Instancing 减少重复提交开销
```

#### 知识点拆解（倒推树）

```
60人同屏顶点瓶颈
├── 开销拆解（第一步必须做）
│   ├── 骨骼蒙皮占比（CPU→GPU 蒙皮方式？每顶点几根骨骼？）
│   ├── Vertex Shader ALU 占比（VS 指令数 / 寄存器压力）
│   ├── Morph Target 插值占比
│   ├── Tesselation / Geometry Shader（如果有）
│   └── 工具：Snapdragon Profiler / Mali Streamline / RenderDoc
├── LOD 策略
│   ├── 基于屏幕面积而非纯距离（近大远小的标准不同）
│   ├── 团战密度感知：人越多 LOD 切换越激进
│   ├── LOD0 面数目标：≤8万（当前15万偏高）
│   ├── LOD1 面数目标：≤3万
│   ├── LOD2 面数目标：≤1万（远景 / 团战背景角色）
│   └── LOD3（可选）：billboard / imposter
├── 骨骼蒙皮优化
│   ├── GPU 蒙皮（Compute Shader / Vertex Shader 蒙皮）
│   ├── 骨骼数量：主骨骼 ≤ 32 根（移动端 sweet spot）
│   ├── 每顶点骨骼影响数：4→2 for LOD1+
│   ├── 远距离骨骼更新降频（30fps 更新而非 60fps）
│   └── 动画 LOD（远处只播 idle + move，不播 subtle 动作）
├── Morph Target 优化
│   ├── 距离分级：< 15m 全量 Morph Target
│   ├── 15-30m：只保留眨眼和嘴部张合（3个 target）
│   ├── > 30m：完全关闭，或用纹理动画替代
│   └── Morph Target 数据压缩（float→half→uint8）
├── Vertex Shader 优化
│   ├── LOD0 VS：完整计算（逐顶点光照近似、头发摆动、布料模拟）
│   ├── LOD1 VS：去掉布料、简化光照
│   ├── LOD2 VS：只做 MVP 变换 + 基础 UV，光照全放 Fragment
│   ├── Shader 分支：#pragma multi_compile _LOD0 _LOD1 _LOD2
│   └── 寄存器分配优化（减少 temp variable 使用）
├── 合批与实例化
│   ├── 相同角色 GPU Instancing（5个相同英雄合并为 1 个 Draw Call）
│   ├── 骨骼矩阵上传：StructuredBuffer vs Uniform Array
│   └── 索引重排序（cache-friendly 顶点访问顺序）
└── 特殊场景
    ├── 隐身/迷雾角色：只渲染轮廓（outline pass only）
    ├── 死亡角色：快速 fade out 后停止渲染
    └── 复活等待：纯 UI，不渲染 3D 角色
```

#### 代码实现

**1. GPU 蒙皮核心代码（URP + Compute Shader）：**

```hlsl
// GPU Skinning via Compute Shader
// 将骨骼蒙皮计算从 Vertex Shader 前移到 Compute Shader

#pragma kernel SkinMesh

struct VertexData {
    float3 position;
    float3 normal;
    float2 uv;
    float4 weights;   // 4 bone weights
    uint4  boneIds;   // 4 bone indices
};

RWStructuredBuffer<float3> _SkinnedPositions;
RWStructuredBuffer<float3> _SkinnedNormals;
StructuredBuffer<VertexData> _Vertices;
StructuredBuffer<float4x4> _BoneMatrices; // 当前帧骨骼矩阵
uint _VertexCount;

[numthreads(64, 1, 1)]
void SkinMesh(uint3 id : SV_DispatchThreadID)
{
    if (id.x >= _VertexCount) return;

    VertexData v = _Vertices[id.x];
    float3 pos = float3(0, 0, 0);
    float3 nrm = float3(0, 0, 0);

    // 4骨骼蒙皮
    [unroll]
    for (int i = 0; i < 4; i++)
    {
        float w = v.weights[i];
        if (w > 0.001)
        {
            float4x4 m = _BoneMatrices[v.boneIds[i]];
            pos += mul(m, float4(v.position, 1.0)).xyz * w;
            nrm += mul((float3x3)m, v.normal) * w;
        }
    }

    _SkinnedPositions[id.x] = pos;
    _SkinnedNormals[id.x] = normalize(nrm);
}
```

**2. 基于密度的动态 LOD 分配（C#）：**

```csharp
using UnityEngine;
using System.Collections.Generic;

public class BattleLODManager : MonoBehaviour
{
    [Header("LOD Screen Coverage Thresholds")]
    [SerializeField] private float lod0ScreenPercent = 0.25f; // 占屏 25% 以上
    [SerializeField] private float lod1ScreenPercent = 0.10f;
    [SerializeField] private float lod2ScreenPercent = 0.03f;

    [Header("Battle Density")]
    [SerializeField] private int maxLOD0Characters = 6; // 团战时最多6个高精度
    [SerializeField] private float densityCheckRadius = 25f;

    private List<CharacterLODController> _activeCharacters = new();
    private Camera _mainCamera;

    void Update()
    {
        if (_activeCharacters.Count <= 12)
        {
            // 人少时正常 LOD
            foreach (var c in _activeCharacters)
                c.UpdateLODByScreenSize(lod0ScreenPercent, lod1ScreenPercent, lod2ScreenPercent);
        }
        else
        {
            // 团战模式：按距离排序，只保留 top N 为 LOD0
            UpdateBattleLOD();
        }
    }

    void UpdateBattleLOD()
    {
        // 按到镜头中心的屏幕距离排序
        Vector3 camPos = _mainCamera.transform.position;
        Vector3 camForward = _mainCamera.transform.forward;

        // 计算每个角色的优先级（屏幕占比 + 距离）
        _activeCharacters.Sort((a, b) => {
            float scoreA = Vector3.Dot(camForward, a.transform.position - camPos);
            float scoreB = Vector3.Dot(camForward, b.transform.position - camPos);
            return scoreB.CompareTo(scoreA); // 近的排前面
        });

        for (int i = 0; i < _activeCharacters.Count; i++)
        {
            var c = _activeCharacters[i];
            if (i < maxLOD0Characters)
            {
                c.ForceLOD(0); // 团战核心区域
            }
            else if (i < maxLOD0Characters * 2)
            {
                c.ForceLOD(1);
            }
            else
            {
                c.ForceLOD(2); // 背景角色
            }
        }
    }

    public void RegisterCharacter(CharacterLODController c) => _activeCharacters.Add(c);
    public void UnregisterCharacter(CharacterLODController c) => _activeCharacters.Remove(c);
}
```

**3. Morph Target 距离分级：**

```csharp
public class MorphTargetLOD : MonoBehaviour
{
    [SerializeField] private SkinnedMeshRenderer smr;
    [SerializeField] private float fullMorphDistance = 15f;
    [SerializeField] private float reducedMorphDistance = 30f;

    // 简化模式：只保留关键 morph
    [SerializeField] private int[] essentialMorphIndices = { 0, 1, 2 }; // 眨眼左、右、嘴

    private Transform _cam;
    private float _distance;
    private bool _isFullMorph = true;

    void Update()
    {
        _distance = Vector3.Distance(transform.position, _cam.position);

        if (_distance <= fullMorphDistance)
        {
            if (!_isFullMorph) EnableAllMorphs();
            _isFullMorph = true;
        }
        else if (_distance <= reducedMorphDistance)
        {
            if (_isFullMorph) DisableNonEssentialMorphs();
            _isFullMorph = false;
        }
        else
        {
            // 远距离完全关闭
            DisableAllMorphs();
            _isFullMorph = false;
        }
    }

    void EnableAllMorphs()
    {
        int count = smr.sharedMesh.blendShapeCount;
        for (int i = 0; i < count; i++)
            smr.SetBlendShapeWeight(i, GetOriginalWeight(i));
    }

    void DisableNonEssentialMorphs()
    {
        var essentialSet = new HashSet<int>(essentialMorphIndices);
        int count = smr.sharedMesh.blendShapeCount;
        for (int i = 0; i < count; i++)
        {
            if (!essentialSet.Contains(i))
                smr.SetBlendShapeWeight(i, 0f);
        }
    }

    void DisableAllMorphs()
    {
        int count = smr.sharedMesh.blendShapeCount;
        for (int i = 0; i < count; i++)
            smr.SetBlendShapeWeight(i, 0f);
    }

    float[] _originalWeights;
    void Start()
    {
        int count = smr.sharedMesh.blendShapeCount;
        _originalWeights = new float[count];
        for (int i = 0; i < count; i++)
            _originalWeights[i] = smr.GetBlendShapeWeight(i);
    }

    float GetOriginalWeight(int i) => _originalWeights != null && i < _originalWeights.Length ? _originalWeights[i] : 0f;
}
```

**优化效果对比表：**

| 优化手段 | 顶点处理耗时 | 效果影响 | 实现难度 | 优先级 |
|----------|-------------|----------|----------|--------|
| LOD0 面数 15万→8万 | -35% | 几乎无 | 美术返工 | ⭐⭐⭐⭐⭐ |
| GPU 蒙皮（Compute） | -20% | 无 | 中等 | ⭐⭐⭐⭐⭐ |
| Morph Target 距离分级 | -12% | 远景轻微 | 低 | ⭐⭐⭐⭐ |
| 团战密度 LOD 切换 | -25% | 背景角色 | 中等 | ⭐⭐⭐⭐⭐ |
| VS 分级（LOD2 精简） | -8% | 远景 | 中等 | ⭐⭐⭐ |
| 远距离骨骼更新降频 | -5% | 几乎无 | 低 | ⭐⭐⭐ |
| **合计** | **-约70%** | | | |
| **目标** | **≤20%** | | | |

### ⚡ 实战经验

- **顶点瓶颈 vs Fill Rate 瓶颈判断**：用 Profiler 看 GPU stage 占比。如果 Vertex Shader 占比 > 35%，基本就是顶点瓶颈。另一个判断：降低分辨率不改善 → 顶点瓶颈；降低分辨率改善明显 → Fill Rate 瓶颈
- **GPU 蒙皮的坑**：Adreno GPU 上 Compute Shader → Vertex Shader 的数据传递要注意 barrier 同步。建议用 Vertex Shader 直接蒙皮（StructuredBuffer 读骨骼矩阵），避免 Compute→VS 的同步开销
- **LOD0 面数是根本**：15 万面的 LOD0 在 MOBA 里确实偏高。行业标准是 LOD0 在 5-8 万面（角色）。这步做了，后面优化压力小很多
- **Morph Target 是隐形杀手**：每个 Morph Target 需要额外的顶点数据读取和插值。60 个角色 × 每个 15 个 Morph Target × 每个顶点 24 字节 = 巨大的带宽和 ALU 开销
- **Imposter 作为终极手段**：对于团战边缘的远处角色（屏幕上只有几个像素），换成预渲染的 Billboard 可以节省 100% 顶点开销。但 MOBA 里需要慎用，因为角色需要保持动画感

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么判断是顶点瓶颈 | GPU Pipeline 阶段理解不足 | 学 GPU 渲染管线各阶段 + Profiler 使用 |
| 不会 GPU 蒙皮 | Compute Shader / StructuredBuffer | 学 GPU Skinning 实现方案 |
| LOD 切换有跳变 | LOD 过渡方案不熟悉 | 学 dithering LOD transition / Smooth LOD |
| 60人帧率提升了但稳定不了 | 帧时间方差大 | 学帧预算分配 + CPU/GPU 双线 Profiling |
| Morph Target 占比算不清楚 | Morph Target 性能模型不清 | 学 Morph Target 的 GPU 开销模型 |

### 🔗 相关问题

- [顶点处理瓶颈排查](../optimization/vertex-bound-bottleneck.md)：通用顶点瓶颈排查方法论
- [蒙皮动画顶点开销](../optimization/skinned-mesh-vertex-animation-cost.md)：CPU vs GPU 蒙皮的性能对比
- [骨骼动画精度与压缩](../technical-art/skeletal-animation-precision-compression.md)：骨骼数据的存储优化
