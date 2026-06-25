---
title: "Houdini+Python自动化地形河流生成管线：美术只要画一条线"
category: "pipeline"
level: 4
tags: ["Houdini", "HDA", "Python", "地形管线", "河流生成", "程序化", "Unity导出"]
hint: "核心=HDA暴露河流曲线→Houdini自动生成河道几何+Flowmap+碰撞体→Python批量导出到Unity，美术只需画一条曲线"
related: ["pipeline/houdini-vegetation-scatter", "pipeline/maya-lod-automation", "pipeline/blender-python-batch-export", "technical-art/lod-spec-and-qa"]
---

## 参考答案

### 🎬 场景描述

面试官展示一张开放世界地图，上面画了十几条河流，然后说：

> "我们的项目有 30km × 30km 的开放世界地图，需要几十条河流。现在美术手动在 Maya 里捏河道，一个人一周才能做一条河。我要你设计一套自动化管线：**美术在 Houdini 里画一条曲线 → 自动生成河道几何体、水流 Flowmap、河岸过渡遮罩、碰撞体，并导出到 Unity**。给我完整的管线架构。"

这是腾讯、网易、米哈游做开放世界项目的 **TA / 程序化管线高级面试题**。考察的是 Houdini Digital Asset (HDA) 设计、程序化生成逻辑、Python 自动化、以及 DCC 到引擎的导出管线全链路。

### ✅ 核心要点

1. **核心思路：曲线驱动**：美术只负责画河流中心线曲线，所有后续几何体由 Houdini 程序化生成
2. **HDA 封装**：将整个生成逻辑封装为 Houdini Digital Asset，美术在 Unity 中通过 Houdini Engine 调参
3. **四层输出**：河道几何体（Mesh）+ Flowmap（纹理）+ 河岸遮罩（纹理）+ 碰撞体（简化网格）
4. **Python 批量导出**：Python 脚本遍历所有河流，自动导出到 Unity 的 StreamingAssets
5. **版本控制友好**：河流数据保存为 JSON（曲线控制点 + 参数），几何体是派生产物不入库

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：Unity 中看到有水流动画、河岸过渡自然的河道
                ↑
倒推1：Unity 需要什么？
       ├── 河道 Mesh（几何体）
       ├── Flowmap（控制水流方向和速度的纹理）
       ├── 河岸遮罩（控制河岸过渡、植被密度）
       └── 碰撞体（玩家可走区域 / 不可走的河心）
倒推2：这些资产怎么生成？
       ├── 河道 Mesh = 曲线 → Sweep/Extrude → 沿曲线放样
       ├── Flowmap = 沿曲线计算切线方向 → 烘焙到 UV2 的纹理
       ├── 河岸遮罩 = 河道边缘距离场 → 膨胀衰减 → 纹理
       └── 碰撞体 = 河道 Mesh 简化 → Convex Hull 或手动 Box
倒推3：输入是什么？
       ├── 美术画的 Bezier 曲线（河流中心线）
       ├── 曲线上的自定义参数（宽度、深度、流速、河岸坡度）
       └── 地形高度场（Heightfield，用于匹配河道与地形）
倒推4：怎么封装成可复用管线？
       ├── HDA（Houdini Digital Asset）= 黑盒工具
       ├── 美术在 Houdini 或 Unity（通过 Houdini Engine）操作
       └── Python 脚本批量处理多条河流 → 自动导出
倒推5：版本控制怎么搞？
       ├── 河流数据 = JSON（曲线点 + 参数），入 Git
       ├── 几何体/纹理 = 派生产物，不入 Git（可重新生成）
       └── HDA 自身 = 版本化管理（如 v1.2.0）
```

#### 知识点拆解（倒推树）

```
Houdini 地形河流生成管线
├── Houdini 程序化生成
│   ├── 曲线处理
│   │   ├── Resample（等距重采样，控制河道精度）
│   │   ├── Curve参数映射（宽度/深度随曲线参数变化）
│   │   └── 地形匹配（曲线投影到 Heightfield）
│   ├── 河道几何体生成
│   │   ├── Sweep SOP（沿曲线放样截面）
│   │   ├── 自定义截面 Profile（U/V 型河道）
│   │   ├── Smooth（平滑几何体）
│   │   └── PolyReduce（LOD 生成）
│   ├── Flowmap 生成
│   │   ├── 沿曲线计算切线方向
│   │   ├── 雅可比矩阵计算流速变化
│   │   └── 烘焙到 UV2 纹理（RG = 方向，B = 速度）
│   ├── 河岸遮罩
│   │   ├── VDB SDF（距离场计算）
│   │   ├── 距离场 → 衰减曲线 → 黑白遮罩
│   │   └── 与地形 Heightfield 混合（河岸侵蚀效果）
│   └── 地形适配
│       ├── 河道切割地形（Carving）
│       ├── 河岸高度修正（平滑过渡到河道边缘）
│       └── 高度差检测（标记需要美术检查的断层）
├── HDA 封装
│   ├── HDA 参数设计（暴露给美术的参数集）
│   │   ├── 河流宽度、深度、河岸坡度
│   │   ├── 水流速度（影响 Flowmap 纹理生成）
│   │   ├── 导出精度（网格分辨率、纹理尺寸）
│   │   └── LOD 设置
│   ├── 类型锁与版本管理
│   └── 错误处理（曲线自交叉、非流形几何体）
├── Python 批量导出
│   ├── Houdini Python API（hou 模块）
│   │   ├── 遍历所有河流曲线节点
│   │   ├── 触发 Cook（重新生成几何体）
│   │   └── 导出 FBX / OBJ / EXR
│   ├── 自动导出路径映射
│   │   ├── 河道 Mesh → Unity Assets/Meshes/Rivers/
│   │   ├── Flowmap EXR → Unity Assets/Textures/Rivers/
│   │   └── 碰撞体 → Unity Assets/Meshes/Rivers/Colliders/
│   ├── 命名规范（River_01_Mesh.fbx, River_01_Flowmap.exr）
│   └── 增量导出（只重新导出修改过的河流）
├── Unity 集成
│   ├── Houdini Engine for Unity（实时预览）
│   ├── Import Settings 自动化（PostProcessor）
│   │   ├── Mesh → 自动加 Mesh Collider
│   │   ├── Flowmap → Texture Importer（无压缩/点过滤）
│   │   └── 遮罩 → Texture Importer（单通道）
│   └── 水体材质自动绑定（根据命名规范匹配 Shader）
└── 版本控制与协作
    ├── 河流数据序列化（JSON Schema）
    ├── HDA 版本管理（Semantic Versioning）
    ├── 美术协作流程（Curve → Review → Export → Unity）
    └── CI/CD 集成（Houdini Headless 自动 Cook + 导出）
```

#### 代码实现

**Houdini HDA 核心网络（HDA 内部节点树）：**

```
[输入: Curve + Heightfield]
    │
    ├── [Resample] 等距重采样曲线 (每 2m 一个点)
    │
    ├── [Attribute Wrangle] 曲线参数映射
    │   └── 根据曲线属性 @width, @depth, @slope
    │       设置每个点的河道截面参数
    │
    ├── [Sweep] 沿曲线放样河道截面
    │   └── 输入: 重采样曲线 + Profile 截面 (U 型/V 型)
    │       输出: 河道几何体
    │
    ├── [Smooth] 平滑河道几何体
    │
    ├── [Attribute Wrangle] 计算 Flowmap 方向
    │   └── v@flow_dir = normalize(v@tangent) // 沿曲线切线
    │       f@flow_speed = chf("flow_speed") * chf("width_ratio")
    │
    ├── [UV Texture] UV2 = Flowmap UV
    │   └── 使用 Spline Mapping 沿曲线展开 UV2
    │
    ├── [Bake Texture] 烘焙 Flowmap 到 UV2
    │   └── 输出: EXR 格式 (RG=方向, B=速度)
    │
    ├── [VDB from Polygons] → [VDB SDF]
    │   └── 计算河道距离场
    │
    ├── [VDB Topology to SDF] → [Volume VOP]
    │   └── 距离场 → 衰减曲线 → 河岸遮罩
    │
    ├── [Heightfield Layer] 河道地形修正
    │   └── 河道区域高度降低 + 河岸平滑过渡
    │
    ├── [PolyReduce] 生成碰撞体 LOD
    │
    └── [Output] 四路输出
        ├── Output 0: 河道 Mesh (FBX)
        ├── Output 1: Flowmap 纹理 (EXR)
        ├── Output 2: 河岸遮罩 (EXR/PNG)
        └── Output 3: 碰撞体 Mesh (FBX)
```

**Python 批量导出脚本（运行在 Houdini 中）：**

```python
import hou
import os
import json
import subprocess

def export_all_rivers(export_root, hda_node_path="/obj/RiverGenerator"):
    """遍历所有河流曲线节点，批量导出"""
    river_gen = hou.node(hda_node_path)
    if not river_gen:
        raise RuntimeError(f"HDA node not found: {hda_node_path}")

    # 获取所有河流曲线
    curves_node = river_gen.node("INPUT_CURVES")
    river_curves = curves_node.children()  # 每个子节点是一条河流

    export_manifest = {
        "version": "1.0.0",
        "rivers": []
    }

    for curve_node in river_curves:
        river_name = curve_node.name()  # e.g. "River_01"

        # 设置当前激活的河流
        river_gen.parm("active_river").set(curve_node.path())

        # 触发 Cook
        river_gen.cook()

        # 检查是否有错误
        errors = river_gen.errors()
        if errors:
            print(f"⚠️  {river_name} cook error: {errors}")
            continue

        # 导出路径
        mesh_path = os.path.join(export_root, "Meshes/Rivers", f"{river_name}_Mesh.fbx")
        flow_path = os.path.join(export_root, "Textures/Rivers", f"{river_name}_Flowmap.exr")
        mask_path = os.path.join(export_root, "Textures/Rivers", f"{river_name}_BankMask.png")
        collider_path = os.path.join(export_root, "Meshes/Rivers", f"{river_name}_Collider.fbx")

        os.makedirs(os.path.dirname(mesh_path), exist_ok=True)
        os.makedirs(os.path.dirname(flow_path), exist_ok=True)

        # 导出河道 Mesh
        mesh_geo = river_gen.node("OUT_MESH").geometry()
        mesh_geo.saveToFile(mesh_path)

        # 导出 Flowmap（EXR 格式保持精度）
        flow_geo = river_gen.node("OUT_FLOWMAP").geometry()
        flow_geo.saveToFile(flow_path)

        # 导出河岸遮罩
        mask_geo = river_gen.node("OUT_BANK_MASK").geometry()
        mask_geo.saveToFile(mask_path)

        # 导出碰撞体
        collider_geo = river_gen.node("OUT_COLLIDER").geometry()
        collider_geo.saveToFile(collider_path)

        # 记录到 manifest
        curve_points = []
        for pt in curve_node.geometry().points():
            curve_points.append([pt.position().x(), pt.position().y(), pt.position().z()])

        river_data = {
            "name": river_name,
            "curve_points": curve_points,
            "params": {
                "width": curve_node.parm("river_width").eval(),
                "depth": curve_node.parm("river_depth").eval(),
                "flow_speed": curve_node.parm("flow_speed").eval(),
                "bank_slope": curve_node.parm("bank_slope").eval(),
            },
            "files": {
                "mesh": os.path.relpath(mesh_path, export_root),
                "flowmap": os.path.relpath(flow_path, export_root),
                "bank_mask": os.path.relpath(mask_path, export_root),
                "collider": os.path.relpath(collider_path, export_root),
            }
        }
        export_manifest["rivers"].append(river_data)
        print(f"✅ Exported {river_name}")

    # 保存 manifest（版本控制用）
    manifest_path = os.path.join(export_root, "rivers_manifest.json")
    with open(manifest_path, "w") as f:
        json.dump(export_manifest, f, indent=2, ensure_ascii=False)

    print(f"\n🎉 All rivers exported. Manifest: {manifest_path}")
    return export_manifest


def export_modified_rivers(export_root, last_commit_hash):
    """增量导出：只导出 Git 中曲线数据有变化的河流"""
    # 获取上次导出后有变化的文件
    result = subprocess.run(
        ["git", "diff", "--name-only", last_commit_hash, "rivers/"],
        capture_output=True, text=True, cwd=export_root
    )
    changed_files = result.stdout.strip().split("\n")
    changed_rivers = []
    for f in changed_files:
        if f.endswith(".json") and "River_" in f:
            changed_rivers.append(f.split("/")[-1].replace(".json", ""))

    if not changed_rivers:
        print("No rivers modified since last export.")
        return

    print(f"🔄 Incremental export: {len(changed_rivers)} rivers modified")
    # 只导出变化的河流
    river_gen = hou.node("/obj/RiverGenerator")
    for river_name in changed_rivers:
        curve_node = river_gen.node(f"INPUT_CURVES/{river_name}")
        if curve_node:
            _export_single_river(curve_node, river_gen, export_root)
```

**Unity 导入自动化（AssetPostprocessor）：**

```csharp
using UnityEngine;
using UnityEditor;

public class RiverAssetPostprocessor : AssetPostprocessor
{
    static void OnPostprocessAllAssets(
        string[] importedAssets, string[] deletedAssets,
        string[] movedAssets, string[] movedFromAssetPaths)
    {
        foreach (string path in importedAssets)
        {
            // 河道 Flowmap 自动设置导入参数
            if (path.Contains("Textures/Rivers/") && path.EndsWith("_Flowmap.exr"))
            {
                var importer = AssetImporter.GetAtPath(path) as TextureImporter;
                if (importer != null)
                {
                    importer.textureType = TextureImporterType.Normal;
                    importer.textureCompression = TextureImporterCompression.Uncompressed;
                    importer.filterMode = FilterMode.Point;
                    importer.mipmapEnabled = false;
                    importer.SaveAndReimport();
                    Debug.Log($"🌊 Flowmap configured: {path}");
                }
            }

            // 河道 Mesh 自动加碰撞体
            if (path.Contains("Meshes/Rivers/") && path.EndsWith("_Collider.fbx"))
            {
                var importer = AssetImporter.GetAtPath(path) as ModelImporter;
                if (importer != null)
                {
                    importer.addCollider = true; // 自动加 Mesh Collider
                    importer.meshCompression = ModelImporterMeshCompression.Medium;
                    importer.SaveAndReimport();
                }
            }

            // 河道 Mesh 自动绑定水体材质
            if (path.Contains("Meshes/Rivers/") && path.EndsWith("_Mesh.fbx"))
            {
                var go = AssetDatabase.LoadAssetAtPath<GameObject>(path);
                if (go != null)
                {
                    var renderer = go.GetComponentInChildren<MeshRenderer>();
                    if (renderer != null)
                    {
                        var waterMat = AssetDatabase.FindAssets("t:Material Water_River_Shader")
                                      .Length > 0
                            ? AssetDatabase.LoadAssetAtPath<Material>(
                                AssetDatabase.GUIDToAssetPath(
                                    AssetDatabase.FindAssets("t:Material Water_River_Shader")[0]))
                            : null;
                        if (waterMat != null)
                        {
                            renderer.sharedMaterial = waterMat;
                            Debug.Log($"💧 Water material auto-bound: {path}");
                        }
                    }
                }
            }
        }
    }
}
```

**管线架构图：**

```
┌──────────────────────────────────────────────────────────┐
│                    美术工作流                              │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │ Houdini │───→│ River HDA    │───→│ Unity Preview   │   │
│  │ 画曲线  │    │ (自动生成)    │    │ (Houdini Engine)│   │
│  └─────────┘    └──────┬───────┘    └─────────────────┘   │
│                        │                                   │
│                 JSON 序列化（Git 版本控制）                  │
└────────────────────────┼─────────────────────────────────┘
                         │
┌────────────────────────┼─────────────────────────────────┐
│                  Python 批量导出                           │
│                        │                                   │
│  ┌─────────────────────┼─────────────────────────────┐    │
│  │              遍历所有河流                            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │    │
│  │  │ Cook HDA │→ │ Export   │→ │ Update Manifest  │ │    │
│  │  │ (生成)   │  │ (FBX/EXR)│  │ (JSON)           │ │    │
│  │  └──────────┘  └──────────┘  └──────────────────┘ │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────┼─────────────────────────────────┘
                         │
┌────────────────────────┼─────────────────────────────────┐
│               Unity 自动导入                               │
│                        │                                   │
│  ┌──────────┐  ┌───────────────┐  ┌──────────────────┐    │
│  │ Mesh     │  │ Flowmap EXR   │  │ Bank Mask PNG    │    │
│  │ +Collider│  │ (无压缩/Point)│  │ (单通道)          │    │
│  └──────────┘  └───────────────┘  └──────────────────┘    │
│       ↓               ↓                    ↓                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          RiverRenderer 组件（自动绑定）                 │  │
│  │  ┌──────────┐  ┌──────────────┐  ┌────────────────┐  │  │
│  │  │ Mesh     │  │ Flowmap      │  │ 水体 Shader    │  │  │
│  │  │ Filter   │  │ Material     │  │ (URP)          │  │  │
│  │  └──────────┘  └──────────────┘  └────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### ⚡ 实战经验

- **曲线质量决定一切**：美术画的曲线如果有锐角折线，Sweep 出来的河道会扭曲。在 HDA 内部加一个 `Smooth SOP` 自动平滑，美术感受不到但效果好很多
- **地形匹配是最大坑**：河流曲线投影到 Heightfield 时，如果地形和曲线高度差太大（河流穿过山崖），会出现河道悬空或穿地。加一个 `Heightfield Detect Issues` 节点标记问题区域，用红色 Display 标签提示美术
- **Flowmap 精度**：Flowmap 用 EXR（32bit float）而不是 PNG（8bit）。8bit 的 Flowmap 在河道弯曲处会出现马赛克状的流速方向错误
- **Houdini Engine for Unity 的性能**：实时 Cook 一条河 3-5 秒，30 条河 = 等待 2 分钟。美术编辑时用低精度预览（1/4 分辨率），最终导出时用全精度
- **命名规范自动化**：河流名称一旦确定就不要改（改名 = 所有导出文件路径变化 = Unity 引用断裂）。在 HDA 参数面板锁定名称参数
- **CI/CD 集成**：用 Houdini Headless（`hython`）在 CI 服务器上运行 Python 导出脚本，当美术提交曲线 JSON 时自动触发重新导出
- **版本兼容**：HDA 升级时，旧河流数据可能不兼容。在 JSON manifest 中存 HDA 版本号，升级时做一次批量迁移
- **碰撞体不要用河道原始 Mesh**：太复杂，碰撞检测会卡。用 PolyReduce 降到 10-20% 三角面，或者直接用手工 Box 拼接简化碰撞体

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么从曲线生成河道 | Sweep SOP / 放样 | 学 Houdini Sweep / Sweep2 |
| Flowmap 原理不清 | Flowmap 在水体渲染中的作用 | 学 Flowmap 水面 Shader 实现 |
| 不知道 HDA 怎么封装 | Houdini Digital Asset 体系 | 学 HDA 创建 + 参数暴露 |
| Python 导出脚本不会写 | Houdini Python API | 学 hou 模块（geometry/file I/O） |
| Unity 导入不会自动化 | AssetPostprocessor | 学 Unity Asset Pipeline |
| 版本控制方案说不清 | 程序化资产的版本管理 | 学 DCC 工具的 Git 最佳实践 |
| 不知道 CI/CD 怎么集成 | hython + CI Pipeline | 学 GitHub Actions / Jenkins + Houdini |

### 🔗 相关问题

- Houdini Engine for Unity 和直接导出 FBX 有什么区别？什么时候用哪个？
- 如果地形是运行时动态加载的（如 Tile Streaming），河流数据怎么适配？
- 多条河流汇聚到湖泊时，Flowmap 怎么处理交汇区域？
- HDA 升级后旧数据不兼容怎么办？设计一个数据迁移方案。
- 如果美术想在 Unity 中实时调河流走向（而不是回到 Houdini 改），能实现吗？
