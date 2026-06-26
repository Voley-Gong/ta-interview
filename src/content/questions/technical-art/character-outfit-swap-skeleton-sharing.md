---
title: "角色换装系统：骨骼共享与材质合批怎么权衡？"
category: "technical-art"
level: 3
tags: ["换装系统", "骨骼共享", "Skinning", "DrawCall", "材质合批", "手游优化"]
hint: "核心矛盾：共享骨骼降低动画计算成本，但换装部件材质不同导致 DrawCall 上升——需要在骨骼复用和材质合批之间找平衡点"
related: ["optimization/drawcall-500-to-100", "technical-art/character-material-spec-workflow", "technical-art/skeletal-animation-precision-compression"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款二次元手游，角色需要支持实时换装——上衣、裤子、鞋子、配饰可以自由组合。目前每个部件都是独立 SkinnedMeshRenderer，导致一个角色 6-8 个 DrawCall，同屏 10 个角色就是 60-80 个 DrawCall。给我一个换装系统的 TA 方案，既要骨骼动画正确，又要控制 DrawCall。」

### ✅ 核心要点

1. **骨骼共享是基础**：所有部件共用一套骨架（Skeleton），部件只存 Mesh + 蒙皮权重
2. **DrawCall 是核心瓶颈**：每个独立材质 = 一个 DrawCall，需要从材质合批入手
3. **合批策略分层**：能合并材质的合并（GPU Texture Atlas），不能合并的用 GPU Instancing
4. **换装槽位系统**：Slot-based 装配，运行时动态组合 Mesh 或动态切换可见性
5. **内存与包体考量**：每套装备的资源量 × 可能的组合数 = 潜在的包体爆炸

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色实时换装 → 动画正确 → DrawCall 最少 → 内存可控
                ↑
倒推1：动画要正确 → 所有部件必须共享同一套骨骼层级
倒推2：DrawCall 要少 → 材质相同的部件合并，不同材质考虑 Atlas
倒推3：换装要灵活 → Slot 系统（头/身/手/腿/脚/配饰），每槽动态加载
倒推4：内存要可控 → 资源拆分策略（共享骨骼 + 独立 Mesh + 材质模板）
倒推5：上线要稳 → 制作规范 + 自动化检查工具
```

#### 知识点拆解（倒推树）

```
换装系统 TA 方案
├── 骨骼系统
│   ├── 共享骨架（Prefab 化的 Skeleton Root）
│   ├── SkinnedMeshRenderer.bones 数组共享
│   ├── 部件只导出 Mesh + BindPose + Weight（不含骨骼节点）
│   └── 动画只在骨架根节点播放一次（Animator 只有 1 个）
├── DrawCall 优化
│   ├── 方案A：动态合并 Mesh（运行时 CombineMesh）
│   │   ├── 前提：材质相同（或 Atlas 兼容）
│   │   ├── 优点：1 个角色 1 个 DrawCall
│   │   └── 缺点：合并耗时、内存翻倍、无法 per-part 剔除
│   ├── 方案B：GPU Texture Atlas + 材质复用
│   │   ├── 将不同装备贴图打包到 Atlas
│   │   ├── 调整 UV offset 切换装备
│   │   └── 条件：装备贴图尺寸统一、纹理格式一致
│   ├── 方案C：Draw Call 级别容忍 + 严格预算
│   │   ├── 单角色 ≤ 3 DrawCall（body + face + accessory）
│   │   └── 用 SRP Batcher 减少 SetPass 开销
│   └── 方案D：Mesh Bake 离线合并（适合固定套装）
├── 装配系统设计
│   ├── Slot 定义（枚举：Head/Body/Hand/Leg/Foot/Accessory）
│   ├── 部件 Prefab 结构：Mesh + Material Ref + Bone Ref
│   ├── 运行时装配流程
│   │   ├── 1. 加载骨架 Prefab
│   │   ├── 2. 按配置加载各 Slot 部件
│   │   ├── 3. 绑定 bones 数组到 SMR
│   │   └── 4. 触发一次 RebuildBounds（避免 culling 错误）
│   └── 卸载策略（换下装备 → UnloadAsset）
├── 制作规范
│   ├── 骨骼命名规范（严格一致，否则 bones 数组对不上）
│   ├── UV 规范（Atlas 方案要求 0-1 空间 + padding）
│   ├── 贴图规格（尺寸/格式/通道分配一致）
│   └── 蒙皮权重限制（最多 4 bones/vertex，权重归一化）
└── 性能监控
    ├── DrawCall 预算分配（主角 3 / NPC 2 / 怪物 1）
    ├── 内存预算（每套装备 Mesh + Texture ≤ X MB）
    └── Frame Debugger 验证合批效果
```

#### 代码实现

**运行时换装装配核心（Unity C#）：**

```csharp
using System.Collections.Generic;
using UnityEngine;

public enum OutfitSlot { Head, Body, Hand, Leg, Foot, Accessory }

public class CharacterOutfitSystem : MonoBehaviour
{
    [SerializeField] private Transform skeletonRoot;
    [SerializeField] private Transform[] bones; // 共享骨骼数组

    // 当前装备的部件
    private Dictionary<OutfitSlot, GameObject> _equippedParts = new();

    // 部件配置（从配置表/服务器读取）
    [System.Serializable]
    public class OutfitConfig
    {
        public OutfitSlot slot;
        public Mesh mesh;
        public Material material;
        public string[] boneNames; // 对应 bones 数组的名字
    }

    /// <summary>
    /// 装备一个部件到指定槽位
    /// </summary>
    public void Equip(OutfitConfig config)
    {
        // 先卸载旧部件
        Unequip(config.slot);

        // 创建 GO 并挂载 SkinnedMeshRenderer
        GameObject partObj = new GameObject($"Part_{config.slot}");
        partObj.transform.SetParent(skeletonRoot, false);

        var smr = partObj.AddComponent<SkinnedMeshRenderer>();
        smr.sharedMesh = config.mesh;
        smr.sharedMaterial = config.material;

        // 绑定共享骨骼——关键！
        smr.bones = ResolveBones(config.boneNames);
        smr.rootBone = skeletonRoot;

        // 重建包围盒，防止被错误剔除
        smr.updateWhenOffscreen = true;
        smr.RebuildBounds();

        _equippedParts[config.slot] = partObj;
    }

    /// <summary>
    /// 卸载槽位部件
    /// </summary>
    public void Unequip(OutfitSlot slot)
    {
        if (_equippedParts.TryGetValue(slot, out var partObj))
        {
            Destroy(partObj);
            _equittedParts.Remove(slot);
        }
    }

    /// <summary>
    /// 按名字解析骨骼引用（要求骨骼命名严格规范）
    /// </summary>
    private Transform[] ResolveBones(string[] boneNames)
    {
        var resolved = new Transform[boneNames.Length];
        for (int i = 0; i < boneNames.Length; i++)
        {
            // 在骨骼池中查找匹配
            resolved[i] = FindBoneRecursive(skeletonRoot, boneNames[i]);
            if (resolved[i] == null)
                Debug.LogError($"[OutfitSystem] Bone not found: {boneNames[i]}");
        }
        return resolved;
    }

    private Transform FindBoneRecursive(Transform parent, string name)
    {
        if (parent.name == name) return parent;
        for (int i = 0; i < parent.childCount; i++)
        {
            var found = FindBoneRecursive(parent.GetChild(i), name);
            if (found != null) return found;
        }
        return null;
    }
}
```

**材质 Atlas 方案——运行时切换 UV offset：**

```csharp
public class OutfitMaterialAtlas : MonoBehaviour
{
    [SerializeField] private SkinnedMeshRenderer smr;
    [SerializeField] private Texture2D atlasTexture; // 4x4 装备图集
    [SerializeField] private int atlasCols = 4;
    [SerializeField] private int atlasRows = 4;

    private static readonly int UVOffsetID = Shader.PropertyToID("_UVOffset");
    private static readonly int UVScaleID = Shader.PropertyToID("_UVScale");

    /// <summary>
    /// 切换到图集中第 index 个装备
    /// </summary>
    public void SwitchOutfit(int index)
    {
        float col = index % atlasCols;
        float row = index / atlasCols;

        Vector2 scale = new Vector2(1f / atlasCols, 1f / atlasRows);
        Vector2 offset = new Vector2(col / (float)atlasCols, row / (float)atlasRows);

        var mpb = new MaterialPropertyBlock();
        smr.GetPropertyBlock(mpb);
        mpb.SetVector(UVScaleID, scale);
        mpb.SetVector(UVOffsetID, offset);
        smr.SetPropertyBlock(mpb);
    }
}
```

**方案对比表：**

| 方案 | DrawCall/角色 | 换装灵活度 | 内存占用 | 实现难度 | 适用场景 |
|------|:---:|:---:|:---:|:---:|------|
| 独立 SMR（无优化） | 6-8 | ★★★★★ | 低 | ★ | 原型阶段 |
| 动态 CombineMesh | 1 | ★★★ | 高（副本） | ★★★ | 固定套装、低同屏 |
| Texture Atlas + UV Offset | 2-3 | ★★★★ | 中 | ★★★ | 二次元手游（主流） |
| GPU Instancing | 1-2 | ★★ | 低 | ★★★★ | NPC / 怪物群体 |
| 离线 Mesh Bake | 1 | ★ | 低 | ★★ | 套装不可拆分 |

### ⚡ 实战经验

- **骨骼命名是生命线**：在项目第一天就确立骨骼命名规范，并用工具检查每个导出的部件——命名不一致会导致 `smr.bones` 对不上，蒙皮全乱
- **Atlas 方案的 padding 陷阱**：装备贴图在图集中必须留至少 2px padding（移动端建议 4px），否则边缘会出现相邻装备的像素渗透
- **SRP Batcher 是免费午餐**：确保所有装备材质用同一个 Shader + CBUFFER 兼容，SetPass Call 能从 N 降到 1
- **面部不要参与合批**：面部表情系统（Blend Shape / 面部骨骼）通常需要独立材质，强制合并会导致表情失效
- **内存比 DrawCall 更致命**：在低端机上，每套装备多 2MB 贴图 × 100 套 = 200MB，必须做按需加载 + LRU 缓存
- **RebuildBounds 容易忘**：运行时绑定骨骼后必须调一次 `smr.RebuildBounds()`，否则角色可能被相机错误剔除（表现为角色闪烁或消失）

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么共享骨骼 | SkinnedMeshRenderer.bones 原理 | 学 Unity 蒙皮渲染底层 |
| DrawCall 压不下来 | 材质合批 / SRP Batcher | 学 URP 合批规则与 CBUFFER |
| 换装后动画错乱 | 骨骼命名规范 / BindPose | 学 DCC 导出设置与骨骼映射 |
| Atlas 后贴图有缝隙 | UV padding / mipmap bleeding | 学 Texture Atlas 打包规范 |
| 低端机内存溢出 | 资源按需加载策略 | 学 Addressables / LRU 缓存 |

### 🔗 相关问题

- 如何在不重启游戏的情况下热更新一套新装备？（提示：Addressables 远程加载 + 运行时装配）
- 如果角色需要胖瘦变化（体型 morph），换装系统怎么适配？（提示：Bone Scale + Mesh Morph 的协调）
- 百人同屏时换装系统的 DrawCall 预算怎么分配？（提示：LOD 降级 + Impostor 替换）
