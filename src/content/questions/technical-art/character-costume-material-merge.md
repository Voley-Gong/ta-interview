---
title: "角色换装系统：20套时装 + 5种体型，如何做到不爆 Draw Call 又不穿模？"
category: "technical-art"
level: 3
tags: ["换装系统", "Draw Call", "材质合并", "Mesh合并", "骨骼绑定", "LOD", "手游优化"]
hint: "关键在于 Shared Material + Mesh Combine + 骨骼复用三管齐下，而不是每件衣服独立材质"
related: ["technical-art/character-outfit-swap-skeleton-sharing", "technical-art/shader-template-system", "optimization/drawcall-500-to-100"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的二次元手游有 20 套时装、5 种体型（S/M/L/儿童/特殊），玩家可以在角色装扮界面自由搭配上衣、下装、鞋子、头饰。问题是：换装界面渲染 8 个部位 × 独立材质 = Draw Call 飙到 40+，战斗中同屏 10 个角色直接爆到 400+。给我一套完整的换装渲染方案，Draw Call 要控制在合理范围内，同时不能穿模。」

### ✅ 核心要点

1. **部位拆分策略**：角色按骨骼绑定区域拆分为 Head / Body_Top / Body_Bottom / Feet / Hands / Hair / Accessory_Front / Accessory_Back
2. **材质合并（Material Atlas）**：同一 shader 的部位使用同一材质，通过 UV 区域切换贴图
3. **动态 Mesh 合并**：运行时将同体型同 shader 的部位 Mesh 合并为一个 DrawCall
4. **骨骼复用**：所有时装共享同一套骨架（Skeleton），不同体型用 Retargeting 适配
5. **穿模解决方案**：物理碰撞胶囊体 + 骨骼加权修正 + Mesh 预处理（遮罩贴图标记隐藏区域）
6. **LOD 联动**：换装系统的每个部位都需要配套 LOD，中远景自动降级

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终目标：同屏 10 个角色 × 各 8 部位 = 80 个渲染对象，Draw Call 控制在 30 以内
                ↑
倒推1：10 个角色不能每个 8 Draw Call → 必须合并 → 每角色 2-3 Draw Call（身体/头发/特效）
倒推2：合并 Mesh 需要同 Material → 时装材质需要模板化 → 制作 Material Atlas
倒推3：不同体型的同一件衣服 Mesh 不同 → 需要每种体型各做一版 → 但材质可以共用
倒推4：换装不需要实时变化 → 可以在「确认换装」时做合并 → 用 Job System 加速
倒推5：穿模问题 → 骨骼层做物理约束 + Mesh 层做遮罩 → 双重保险
倒推6：战斗中需要 LOD → 换装系统必须预生成 LOD → 工具链自动化
```

#### 知识点拆解（倒推树）

```
换装系统渲染优化
├── 部位拆分与规范（Part System）
│   ├── 骨骼分区命名规范（Head_Bone → Head_Part）
│   ├── Mesh 拆分规范（每个部位独立 Mesh + UV 隔离区）
│   ├── 材质分配规则（同 shader 归为一组，最多 3 组：Skin / Cloth / Metal）
│   └── 穿模遮罩权重（Weight Paint 标记隐藏过渡区）
├── 材质合并方案（Material Atlas）
│   ├── UV 二次映射（将不同部位贴图打包到一张 Atlas，UV 映射到对应区域）
│   ├── Shader 模板化（统一 Shader + 变体控制参数 → 不同外观）
│   ├── 材质实例化（Material Property Block 设置每角色独立颜色）
│   └── 贴图压缩方案（Atlas 用 ASTC 6x6，角色贴图总量控制 < 8MB）
├── 动态 Mesh 合并（Runtime Combine）
│   ├── 合并时机（换装确认时合并，不是每帧合并）
│   ├── 合并算法（CombineMesh API / 自写 Job System 合并）
│   ├── 骨骼权重处理（合并后 BoneWeights 数组重映射）
│   ├── 法线/切线缝合（接缝处平滑处理）
│   └── 合并缓存（同组合缓存 Combined Mesh，换装时只更新差异部分）
├── 骨骼复用与体型适配（Skeleton Retargeting）
│   ├── 共享骨架（Shared Skeleton，所有时装绑定到同一骨架）
│   ├── 体型 Retargeting（S/M/L 体型 = 骨骼缩放 + 比例调整）
│   ├── 动画复用验证（同一动画在 5 种体型上不能严重变形）
│   └── 儿童体型特殊处理（头身比不同，需要独立的骨骼缩放预设）
├── 穿模解决方案（Clipping Prevention）
│   ├── 方案A：物理布料（Cloth Component / Magica Cloth / 物理骨骼链）
│   ├── 方案B：遮罩权重（身体 Mesh 在穿衣服部位权重=0，渲染时被衣服遮住）
│   ├── 方案C：骨骼约束（限制骨骼旋转范围，防止穿透）
│   └── 方案D：Mesh 预处理（DCC 阶段手动修整穿模区域）
├── LOD 联动系统
│   ├── 每个时装部位 × 每个体型 = 需要生成 LOD 0/1/2/3
│   ├── LOD 生成工具链（Simplygon / InstaLOD / 自写 Maya 脚本）
│   ├── LOD 切换时的材质降级（LOD2 去掉 Detail Map，LOD3 纯色）
│   └── 合并策略随 LOD 变化（LOD0 = 分部位渲染保质量，LOD2+ = 全合并求性能）
└── 内存管理
    ├── 换装资源懒加载（只在装扮界面加载全量，战斗中只加载当前装备）
    ├── AssetBundle 拆分（按角色 ID + 部位类型拆分）
    ├── 内存池复用（同体型的 Mesh 合并缓冲区复用）
    └── GC 控制（合并操作避免产生大量临时数组）
```

#### 代码实现

**1. 动态 Mesh 合并核心代码（C# / Unity）**

```csharp
using UnityEngine;
using Unity.Collections;
using Unity.Jobs;
using System.Collections.Generic;

public class CostumeCombineSystem : MonoBehaviour
{
    [Header("骨骼引用")]
    public Transform[] bones;           // 共享骨架
    public SkinnedMeshRenderer smr;     // 合并后渲染的 SMR

    // 部位槽位
    public CostumePart head;
    public CostumePart bodyTop;
    public CostumePart bodyBottom;
    public CostumePart feet;
    public CostumePart hands;
    public CostumePart hair;

    private Material sharedSkinMat;
    private Material sharedClothMat;

    /// <summary>
    /// 换装确认时调用：合并所有部位的 Mesh
    /// </summary>
    public void CombineCostume()
    {
        // 按 Material 分组
        var skinParts = new List<CostumePart>();
        var clothParts = new List<CostumePart>();

        if (head?.currentMesh && head.materialGroup == MaterialGroup.Skin)
            skinParts.Add(head);
        if (hands?.currentMesh) skinParts.Add(hands);
        if (hair?.currentMesh) skinParts.Add(hair);
        if (bodyTop?.currentMesh) clothParts.Add(bodyTop);
        if (bodyBottom?.currentMesh) clothParts.Add(bodyBottom);
        if (feet?.currentMesh) clothParts.Add(feet);

        // 合并 Cloth 组（通常是最重的一组）
        if (clothParts.Count > 0)
        {
            Mesh combinedCloth = CombineMeshes(clothParts);
            smr.sharedMesh = combinedCloth;
            smr.sharedMaterial = sharedClothMat;
        }

        // 合并 Skin 组（头部+手部通常可以合并到身体）
        // 实际项目中可能需要分两个 SMR
    }

    private Mesh CombineMeshes(List<CostumePart> parts)
    {
        // 1. 收集所有顶点数据
        int totalVerts = 0, totalTris = 0;
        foreach (var p in parts)
        {
            totalVerts += p.currentMesh.vertexCount;
            totalTris += p.currentMesh.triangles.Length;
        }

        var vertices = new List<Vector3>();
        var normals = new List<Vector3>();
        var uvs = new List<Vector2>();
        var boneWeights = new List<BoneWeight>();
        var triangles = new List<int>();

        int vertexOffset = 0;

        foreach (var p in parts)
        {
            Mesh m = p.currentMesh;
            int vc = m.vertexCount;

            // 坐标已经是局部坐标，直接追加
            vertices.AddRange(m.vertices);
            normals.AddRange(m.normals);
            uvs.AddRange(m.uv);

            // BoneWeights：骨骼索引需要重映射
            // 假设所有部件共享同一套骨骼，索引不变
            boneWeights.AddRange(m.boneWeights);

            // 三角形索引偏移
            int[] tris = m.triangles;
            for (int i = 0; i < tris.Length; i++)
                triangles.Add(tris[i] + vertexOffset);

            vertexOffset += vc;
        }

        Mesh combined = new Mesh();
        combined.name = "Combined_Costume";
        combined.SetVertices(vertices);
        combined.SetNormals(normals);
        combined.SetUVs(0, uvs);
        combined.boneWeights = boneWeights.ToArray();
        combined.SetTriangles(triangles, 0);
        combined.RecalculateBounds();
        combined.RecalculateTangents();

        return combined;
    }

    /// <summary>
    /// 使用 MaterialPropertyBlock 设置每角色独立参数（不产生新材质实例）
    /// </summary>
    public void SetCharacterColors(Color primary, Color secondary, Color emission)
    {
        MaterialPropertyBlock mpb = new MaterialPropertyBlock();
        smr.GetPropertyBlock(mpb);
        mpb.SetColor("_PrimaryColor", primary);
        mpb.SetColor("_SecondaryColor", secondary);
        mpb.SetColor("_EmissionColor", emission);
        smr.SetPropertyBlock(mpb);
    }
}

[System.Serializable]
public class CostumePart
{
    public string partName;
    public Mesh currentMesh;
    public MaterialGroup materialGroup;
    public BoneWeight[] boneWeights;
}

public enum MaterialGroup
{
    Skin,
    Cloth,
    Metal,
    Effect
}
```

**2. UV Atlas 映射方案（Shader 端）**

```hlsl
// 角色统一 Shader - 换装版
// 通过 _PartID 控制不同部位的 UV 偏移，实现一图多用

float4 _AtlasOffset[8]; // 每个部位在 Atlas 中的 UV 偏移和缩放

// 在顶点着色器中传递 _PartID
struct VertexInput {
    float4 vertex   : POSITION;
    float3 normal   : NORMAL;
    float2 uv       : TEXCOORD0;
    float2 uv2      : TEXCOORD1; // uv2.x = partID
};

// 片元着色器中根据 partID 采样 Atlas
float2 GetAtlasUV(float2 uv, float partID)
{
    int idx = (int)partID;
    float4 offset = _AtlasOffset[idx];
    return uv * offset.zw + offset.xy; // scale + offset
}

half4 frag(v2f i) : SV_Target
{
    float2 atlasUV = GetAtlasUV(i.uv, i.uv2.x);

    // 主贴图
    half4 baseCol = tex2D(_MainTex, atlasUV);

    // 法线贴图
    half3 normalTS = UnpackNormal(tex2D(_BumpMap, atlasUV));

    // ... 光照计算
    return finalColor;
}
```

**3. 换装穿模遮罩权重（Maya Python 预处理脚本）**

```python
import maya.cmds as cmds

def generate_clipping_mask(body_mesh, cloth_mesh, influence_radius=0.02):
    """
    在穿衣服的身体区域，将身体顶点权重置零
    防止身体 Mesh 在衣服内部被渲染
    """
    # 获取身体顶点
    body_verts = cmds.ls(f"{body_mesh}.vtx[*]", flatten=True)
    cloth_verts = cmds.ls(f"{cloth_mesh}.vtx[*]", flatten=True)

    body_positions = [cmds.xform(v, q=True, t=True, ws=True) for v in body_verts]
    cloth_positions = [cmds.xform(v, q=True, t=True, ws=True) for v in cloth_verts]

    # 对每个身体顶点，检查是否在衣服内部（距离判定）
    hidden_indices = []
    for i, bp in enumerate(body_positions):
        for cp in cloth_positions:
            dist = ((bp[0]-cp[0])**2 + (bp[1]-cp[1])**2 + (bp[2]-cp[2])**2)**0.5
            if dist < influence_radius:
                hidden_indices.append(i)
                break

    # 将隐藏顶点的蒙皮权重清零
    skin_cluster = find_skin_cluster(body_mesh)
    if skin_cluster:
        for idx in hidden_indices:
            cmds.setAttr(f"{skin_cluster}.weightList[{idx}].weights[0]", 0.0)

    print(f"[Clipping Mask] 处理完成: 隐藏 {len(hidden_indices)} 个身体顶点")

def find_skin_cluster(mesh):
    """找到 Mesh 关联的 SkinCluster"""
    history = cmds.listHistory(mesh, pruneDag=True)
    skin_clusters = cmds.ls(history, type="skinCluster")
    return skin_clusters[0] if skin_clusters else None
```

### ⚡ 实战经验

> **踩坑1：合并后接缝可见**
> 多个部位 Mesh 合并后，接缝处的法线不连续，光照会出现硬边。解决方案：合并前在接缝处做**法线平滑缝合**（Average Normals），或者用 Maya 的 `Transfer Attributes` 工具传递法线。
>
> **踩坑2：BoneWeight 骨骼索引错乱**
> 不同部位 Maya 导出时骨骼顺序可能不同，合并后 BoneWeights 中的 boneIndex 指向了错误的骨骼。必须在合并时做**骨骼索引重映射表**，确保所有部位使用统一的骨骼索引。
>
> **踩坑3：MaterialPropertyBlock 与 SRP Batcher 冲突**
> URP 下使用 MaterialPropertyBlock 会打断 SRP Batcher，性能不升反降。解法：用 `CBUFFER` 内置参数 + `per-instance` 数据，或者放弃 SRP Batcher 改用 GPU Instancing。
>
> **踩坑4：换装界面 vs 战斗中的性能差异**
> 换装界面需要高质量分开渲染（方便预览），战斗中需要合并。推荐做两套渲染路径：`CostumePreview` 模式（分部位）和 `BattleMerged` 模式（合并），切换时重新构建 Mesh。
>
> **经验值**：一个成熟的换装系统，从 DCC 规范到运行时合并，工具链代码通常在 3000-5000 行。不要低估规范制定的工时。

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明盲区在 |
|--------|-----------|----------------|
| 合并 Mesh 后法线断裂怎么处理？ | 法线缝合/共享法线 | Mesh 合并细节 |
| MaterialPropertyBlock 会影响 SRP Batcher 吗？ | 会，打断 batching | URP 渲染优化 |
| 不同体型的同一件衣服怎么复用？ | 骨骼 Retargeting | 骨骼绑定系统 |
| 换装穿模的根本解法是什么？ | 多层保险：物理+遮罩+约束 | 美术规范+技术方案 |
| 20 套时装的贴图怎么管理？ | Material Atlas + UV 二次映射 | 贴图管线 |
| 合并 Mesh 时三角形索引为什么要偏移？ | 顶点数变了，索引需要重映射 | 底层 Mesh 数据结构 |
| LOD2 时换装还需要分部位吗？ | 不需要，合并更优 | LOD 策略 |
| AssetBundle 怎么按换装拆分？ | 按角色ID+部位类型拆，懒加载 | 资源管理 |

### 🔗 相关问题

- [角色换装骨骼复用](../technical-art/character-outfit-swap-skeleton-sharing) — 骨骼层面如何共享
- [Shader 模板系统](../technical-art/shader-template-system) — 材质模板化的基础设施
- [Draw Call 从500降到100](../optimization/drawcall-500-to-100) — 合并通用策略
- [Shader LOD 质量分级](../technical-art/shader-lod-quality-tier-system) — 换装系统的 LOD 配套
