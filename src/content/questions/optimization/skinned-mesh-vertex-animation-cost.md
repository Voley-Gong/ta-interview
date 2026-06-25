---
title: "角色SKinned Mesh多了3倍顶点量，GPU顶点阶段直接爆红——怎么把皮肤蒙皮的开销降下来？"
category: "optimization"
level: 3
tags: ["Skinned Mesh", "顶点蒙皮", "GPU Skin", "Compute Shader", "顶点压缩", "骨骼数量控制"]
hint: "GPU蒙皮+骨骼数量上限+顶点稀疏化三斧子下去，顶点阶段从8ms降到2ms"
related: ["optimization/vertex-bound-bottleneck", "technical-art/skeletal-animation-jitter-precision", "optimization/shader-variant-explosion"]
---

## 参考答案

### 🎬 场景描述

面试官打开帧分析工具，指着 GPU Timeline 上顶点处理阶段的一片红色说：

> "我们项目一屏同屏 15 个角色，每个角色 4 万面、60 根骨骼。现在顶点阶段直接吃了 8ms，GPU 时间 22ms 帧率不到 45fps。美术说面数不能再砍了，你作为 TA 怎么把蒙皮顶点的开销降下来？"

这是米哈游、网易等重角色项目的经典优化题——角色多 + 骨骼复杂 = GPU 顶点瓶颈。

### ✅ 核心要点

1. **GPU Skinning 是第一优先级**：CPU 端逐顶点计算蒙皮矩阵的开销远大于 GPU 端，必须将蒙皮计算移到 GPU（Compute Shader 或 Vertex Shader 内）
2. **骨骼数量直接决定指令数**：每根骨骼 = 一次矩阵乘法（4 条 dot 指令）。60 根骨骼全权重 = 60 次矩阵乘，实际最多 4 根骨骼影响/顶点 = 4 次，但骨骼总量决定纹理矩阵查找大小
3. **顶点稀疏化 / LOD Mesh**：远距离角色用低顶点 LOD Mesh（5k 面），蒙皮开销线性下降
4. **DCC 端重拓扑**：很多顶点是无意义的（T-pose 姿态下的平直边），重拓扑减少冗余顶点
5. **Compute Skin + Culled Vertex Skip**：用 Compute Shader 预计算蒙皮结果到 StructuredBuffer，然后只对可见顶点采样

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：顶点阶段从 8ms 降到 2-3ms
     ↓ 倒推
瓶颈分析 = 顶点数(40k × 15) × 每顶点蒙皮成本(4 bone weights)
     ↓ 倒推
三个维度同时优化：
  ├── 维度1：减少每顶点计算 → GPU Skinning + Compute Shader 预计算
  ├── 维度2：减少顶点总量 → LOD + 远距离 Imposter
  └── 维度3：减少骨骼矩阵开销 → 骨骼数量裁剪 + 矩阵纹理化
     ↓ 倒推
落地步骤：
  Step 1：Profiler 确认是否 CPU Skinning（验证热点）
  Step 2：切换到 GPU Skinning（Unity 自动或手动 Compute）
  Step 3：骨骼上限限制（60→24，DCC 端 Rig 重建）
  Step 4：LOD 重建（40k→10k→3k→Imposter）
  Step 5：顶点重拓扑（去除冗余顶点）
```

#### 知识点拆解（倒推树）

```
Skinned Mesh 顶点优化
├── 蒙皮原理
│   ├── Linear Blend Skinning（LBS）公式：v' = Σ(wi × Mi × v)
│   ├── 每顶点最多 4 bones 影响（行业标准）
│   ├── 矩阵数组的存储方式（常量寄存器 vs 纹理 vs SSBO）
│   └── CPU Skinning vs GPU Skinning 性能差异（10-50x）
│
├── GPU Skinning 实现路径
│   ├── Unity 内置 GPU Skinning 开关 & 限制
│   ├── Compute Shader 蒙皮流水线
│   │   ├── 输入：原始顶点 Buffer + 骨骼矩阵 Buffer
│   │   ├── 计算：每个 Thread 计算一个顶点的蒙皮结果
│   │   └── 输出：Skinned Vertex Buffer → 间接渲染 DrawIndexedIndirect
│   ├── 矩阵纹理化方案（移动端兼容性好）
│   │   └── 将骨骼矩阵编码到 RGBA Float 纹理中
│   └── 顶点着色器内蒙皮方案（最简单但有指令开销）
│
├── 骨骼系统优化
│   ├── 骨骼数量裁剪策略
│   │   ├── 主骨骼（Spine/Head/Leg/Arm）保留
│   │   ├── 辅助骨骼（Finger/Face/Toe）远距离剔除
│   │   └── LOD Bone：不同 LOD 绑定不同骨骼层级
│   ├── 骨骼矩阵的每帧更新优化
│   │   ├── 只更新有动画的骨骼（脏标记）
│   │   └── 矩阵乘法并行化（Job System / Compute）
│   └── 面部骨骼单独处理（Blend Shape / Texture Animation 替代）
│
├── LOD 与顶点策略
│   ├── 角色LOD 顶点数阶梯（40k → 15k → 5k → 800 Imposter）
│   ├── 远距离 Imposter（Billboard + 法线深度贴图）
│   └── 顶点稀疏化工具（Mesh Simplify with Bone Weight Preservation）
│
└── Profiler 层面
    ├── RenderDoc / Frame Debug 看 Draw Call 的顶点数
    ├── Snapdragon Profiler / Xcode GPU Frame
    │   ├── 确认顶点阶段占比（如果 >30% 就是顶点瓶颈）
    │   └── 寄存器压力分析（太多 bone weights 导致寄存器溢出）
    └── Unity Profiler 的 CPU Skinning 标记
```

#### 代码实现

**1. Compute Shader GPU Skinning 核心代码：**

```hlsl
// SKIN_COMPUTE.compute
#pragma kernel CSMain

StructuredBuffer<float3> _Positions;
StructuredBuffer<float3> _Normals;
StructuredBuffer<float4> _BoneWeights;   // xyzw = 4 bones 的权重
StructuredBuffer<uint4> _BoneIndices;     // xyzw = 4 bones 的索引
StructuredBuffer<float4x4> _BoneMatrices; // 当前帧骨骼矩阵

RWStructuredBuffer<float3> _OutPositions;
RWStructuredBuffer<float3> _OutNormals;

[numthreads(64, 1, 1)]
void CSMain(uint id : SV_DispatchThreadID)
{
    if (id >= _VertexCount) return;

    float4 weights = _BoneWeights[id];
    uint4 indices = _BoneIndices[id];

    float3 pos = _Positions[id];
    float3 nrm = _Normals[id];
    float3 skinnedPos = float3(0, 0, 0);
    float3 skinnedNrm = float3(0, 0, 0);

    // 4 bone LBS
    [unroll]
    for (int i = 0; i < 4; i++)
    {
        float w = weights[i];
        if (w > 0.0)
        {
            float4x4 m = _BoneMatrices[indices[i]];
            skinnedPos += mul(m, float4(pos, 1.0)).xyz * w;
            skinnedNrm += mul((float3x3)m, nrm) * w;
        }
    }

    _OutPositions[id] = skinnedPos;
    _OutNormals[id]   = normalize(skinnedNrm);
}
```

**2. Unity C# 端调度 + 间接渲染：**

```csharp
// GPUSkinningDispatcher.cs
public class GPUSkinningDispatcher : MonoBehaviour
{
    private ComputeShader _skinCompute;
    private GraphicsBuffer _skinnedPositions;
    private GraphicsBuffer _skinnedNormals;

    void Update()
    {
        // 1. 更新骨骼矩阵到 GPU
        _boneMatrixBuffer.SetData(_currentBoneMatrices);

        // 2. Dispatch Compute Shader
        int threadGroups = Mathf.CeilToInt(_vertexCount / 64f);
        _skinCompute.SetBuffer(0, "_Positions", _originalPositions);
        _skinCompute.SetBuffer(0, "_BoneWeights", _boneWeights);
        _skinCompute.SetBuffer(0, "_BoneIndices", _boneIndices);
        _skinCompute.SetBuffer(0, "_BoneMatrices", _boneMatrixBuffer);
        _skinCompute.SetBuffer(0, "_OutPositions", _skinnedPositions);
        _skinCompute.SetBuffer(0, "_OutNormals", _skinnedNormals);
        _skinCompute.Dispatch(0, threadGroups, 1, 1);

        // 3. 用 MeshDrawData 配合 Graphics.RenderMesh 间接渲染
        var rwp = new RenderParams(_skinnedMaterial);
        Graphics.RenderMesh(rwp, _skinnedMesh, 0, transform.localToWorldMatrix);
    }
}
```

**3. 骨骼 LOD 裁剪脚本（DCC 或引擎内）：**

```python
# bone_lod_stripper.py - Maya 脚本示例
# 根据距离/LOD 等级移除次要骨骼
BONE_PRIORITY = {
    'root': 0, 'spine_01': 0, 'spine_02': 0, 'head': 0,
    'arm_L': 0, 'arm_R': 0, 'leg_L': 0, 'leg_R': 0,
    'finger_01_L': 2, 'finger_02_L': 2,  # LOD2 以上才保留
    'toe_L': 2, 'toe_R': 2,
    'face_brow': 3, 'face_mouth': 3,      # LOD3 大特写才保留
}

def strip_bones_for_lod(skin_cluster, lod_level):
    """将超过 LOD 等级的骨骼权重清零并重新归一化"""
    weights = cmds.skinPercent(skin_cluster, query=True, value=True)
    influences = cmds.skinCluster(skin_cluster, query=True, influence=True)
    # 将低优先级骨骼权重转移到最近的父骨骼
    for bone in influences:
        if BONE_PRIORITY.get(bone, 1) > lod_level:
            # 转移权重到父骨骼
            parent = get_parent_bone(bone)
            transfer_weight(skin_cluster, bone, parent)
```

### ⚡ 实战经验

| 优化手段 | 顶点阶段耗时降幅 | 适用场景 | 风险 |
|---------|---------------|---------|------|
| CPU→GPU Skinning | -60%~80% | 所有平台 | 需要 SM 3.5+ |
| 骨骼数 60→24 | -20%~30% | 远距离角色 | 手指/面部动画丢失 |
| LOD Mesh (40k→10k) | -50%~70% | 中远距离 | 硬件 Morph Target 兼容 |
| Compute Skinning + Indirect Draw | -15%~25%（额外） | PS5/PC/Vulkan | 移动端 GLES3 兼容差 |
| 顶点重拓扑 | -10%~20% | 全场景 | 需要 DCC 人工修模 |

> **血泪经验**：某二次元项目角色从 CPU Skinning 切 GPU Skinning 后帧率从 35→55fps，但发现部分 Android 机型 GPU Skinning 走的回退路径反而更慢。**务必做机型分级，低端机走 CPU Skinning + 降面数的组合方案。**

### 🎯 能力体检清单

- [ ] 能否解释 LBS（Linear Blend Skinning）的数学公式和"Candy Wrapper"形变缺陷？
- [ ] 知道 Unity 中 `QualitySettings.skinWeights` 的四个等级对应的骨骼影响数（1/2/4/Unlimited）？
- [ ] 能否用 RenderDoc 确认当前项目是 CPU 还是 GPU Skinning？
- [ ] 知道 Compute Shader 方案在 OpenGL ES 3.0 上的限制（无 SSBO，需用纹理替代）？
- [ ] 是否做过骨骼矩阵编码到纹理的方案（RGBA Float = 矩阵一行，4 像素 = 一个矩阵）？
- [ ] 理解 DQS（Dual Quaternion Skinning）比 LBS 好在哪？为什么手游不用？

### 🔗 相关问题

- [顶点阶段瓶颈分析](../optimization/vertex-bound-bottleneck.md)
- [骨骼动画精度问题与压缩](../technical-art/skeletal-animation-jitter-precision.md)
- [Shader Variant 爆炸治理](../optimization/shader-variant-explosion.md)
