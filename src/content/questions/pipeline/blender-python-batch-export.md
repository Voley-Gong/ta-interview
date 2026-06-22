---
title: "美术交了 200 个 FBX 要全部导出为 Prefab 并配置材质：怎么写 Blender Python 脚本一键搞定？"
category: "pipeline"
level: 2
tags: ["Blender Python", "bpy", "批量导出", "FBX", "Unity Prefab", "材质自动配置", "工具链"]
hint: "核心考点：bpy 操作 API + 导出规则配置 + 批量处理框架——TA 的基本功是把重复劳动变成一键工具"
related: ["pipeline/maya-lod-automation", "pipeline/batch-material-audit-tool", "technical-art/shader-template-system"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们项目用 Blender 建模，Unity 做客户端。美术刚交付了一批新地图道具——200 个 .blend 文件，每个文件里有一个到多个 Mesh。

你需要把它们全部：
1. 导出为 FBX（Unity 可用格式）
2. 在 Unity 里导入并生成 Prefab
3. 自动关联材质（材质名和 FBX 名有对应关系）
4. 输出到对应的文件夹（props/outdoor/、props/indoor/ 等）

美术说手工弄要一个星期。你怎么用脚本把这个流程自动化？给我讲讲你的思路和核心代码。」

追问一：如果有些 Blender 文件里有命名不规范的问题（比如中文命名、空格、特殊字符），你的脚本怎么处理？

追问二：导出的 FBX 在 Unity 里材质显示不对（变成了白色），可能是什么原因？

### ✅ 核心要点

1. **Blender Python (bpy) API**：掌握 `bpy.data`、`bpy.ops.export_scene`、`bpy.context` 三大核心模块
2. **批量处理框架**：遍历目录 → 打开文件 → 处理 → 导出 → 关闭，要有日志和错误恢复
3. **FBX 导出配置**：坐标轴变换（Blender Z-up → Unity Y-up）、材质模式、嵌入纹理选项
4. **Unity 侧自动化**：`AssetPostprocessor` 在导入时自动配置材质和 Prefab 生成
5. **命名规范与容错**：文件名清洗规则、异常处理、跳过已处理文件（断点续传）

### 📖 深度展开

#### 解决思路（从最终交付倒推工具链）

```
最终目标：Unity 项目里出现 200 个配好材质的 Prefab，分好类
              ↑
倒推1：Unity 怎么自动建 Prefab？→ AssetPostprocessor 监听 FBX 导入
倒推2：FBX 从哪来？→ Blender 脚本批量导出
倒推3：导出前要处理什么？→ 只导出需要的 Mesh，清理无用数据
倒推4：材质怎么自动关联？→ 命名规则匹配 + Unity 材质模板
倒推5：分类怎么实现？→ Blender 文件名/目录结构 → Unity 子目录映射
倒推6：万一中断了怎么办？→ 日志记录 + 跳过已完成文件
```

#### 知识点拆解（倒推树）

```
Blender 批量 FBX 导出 + Unity 自动 Prefab 工具链
├── Blender 脚本层
│   ├── bpy 文件操作（bpy.ops.wm.open_mainfile / save）
│   ├── 遍历场景 Mesh 对象（bpy.data.objects, filter by type=='MESH'）
│   ├── 只选择要导出的对象（bpy.ops.object.select_set）
│   ├── FBX 导出参数（axis_forward='-Z', axis_up='Y', path_mode='COPY'）
│   ├── 材质导出模式（'USE_MATERIAL_NODE_TREE' for Principled BSDF）
│   └── 文件名清洗（中文→拼音 or 编号，空格→下划线）
├── 目录遍历与批量框架
│   ├── os.walk 遍历 .blend 文件
│   ├── 断点续传（已完成列表写入 log.json）
│   ├── 异常恢复（单个文件出错不中断整体流程）
│   └── 进度输出（当前/总数/预计剩余时间）
├── Unity 导入自动化
│   ├── AssetPostprocessor.OnPostprocessModel（FBX 导入回调）
│   ├── 自动创建材质（Shader 根据命名规则选择模板）
│   ├── 自动生成 Prefab（PrefabUtility.SaveAsPrefabAsset）
│   └── 材质关联策略（按名称匹配 or 按 Material Slot 顺序）
├── 命名规范与容错
│   ├── 文件名规则定义（小写 + 下划线 + 编号）
│   ├── 不规范命名的自动修正（regex replace）
│   ├── 重名冲突处理（追加序号 suffix）
│   └── 导入失败告警（输出错误日志 + 通知美术）
└── 质量校验
    ├── 导出后文件完整性检查（文件大小 > 0？Mesh 数量 > 0？）
    ├── Unity 导入后材质检查（是否有粉色材质 = Shader 缺失）
    └── 生成报告（成功数/失败数/警告列表）
```

#### 代码实现

**Blender 批量导出脚本（batch_export_fbx.py）：**

```python
"""
Blender 批量 FBX 导出脚本
使用方式: blender --background --python batch_export_fbx.py -- --input_dir ./props --output_dir ./fbx_output
"""

import bpy
import os
import sys
import json
import re
import time
from pathlib import Path

# 解析命令行参数
argv = sys.argv
if "--" in argv:
    argv = argv[argv.index("--") + 1:]
else:
    argv = []

input_dir = argv[0] if len(argv) > 0 else "./props"
output_dir = argv[1] if len(argv) > 1 else "./fbx_output"
log_file = os.path.join(output_dir, "export_log.json")

# ========== 配置 ==========
FBX_EXPORT_PRESET = {
    "axis_forward": "-Z",
    "axis_up": "Y",           # Unity 是 Y-up
    "use_selection": True,
    "object_types": {"MESH"},
    "use_mesh_modifiers": True,
    "path_mode": "COPY",       # 复制纹理到 FBX 同目录
    "embed_textures": False,
    "batch_mode": "OFF",
    "mesh_smooth_type": "FACE",
}

# 分类规则：文件名关键词 → 子目录
CATEGORY_MAP = {
    "outdoor": "props/outdoor",
    "indoor": "props/indoor",
    "furniture": "props/furniture",
    "light": "props/lighting",
    "plant": "props/nature",
    "tree": "props/nature",
    "stone": "props/nature",
}

# 文件名清洗规则
def sanitize_name(name: str) -> str:
    """清洗文件名：中文空格等不规范字符"""
    # 替换空格为下划线
    name = name.replace(" ", "_")
    # 移除特殊字符（保留中文、字母、数字、下划线、减号）
    name = re.sub(r'[^\w\u4e00-\u9fff\-]', '', name)
    # 转小写
    name = name.lower()
    # 多个连续下划线合并
    name = re.sub(r'_+', '_', name)
    return name

def get_category(filename: str) -> str:
    """根据文件名关键词判断分类目录"""
    lower = filename.lower()
    for keyword, subdir in CATEGORY_MAP.items():
        if keyword in lower:
            return subdir
    return "props/misc"

def export_single_blend(blend_path: str, output_base: str) -> dict:
    """导出单个 .blend 文件中的所有 Mesh"""
    results = {"file": blend_path, "exported": [], "errors": []}

    try:
        # 打开文件
        bpy.ops.wm.open_mainfile(filepath=blend_path)

        # 获取所有 Mesh 对象
        meshes = [obj for obj in bpy.data.objects if obj.type == 'MESH']
        if not meshes:
            results["errors"].append("No mesh objects found")
            return results

        # 确定输出目录
        filename = Path(blend_path).stem
        category = get_category(filename)
        out_dir = os.path.join(output_base, category)
        os.makedirs(out_dir, exist_ok=True)

        # 选择所有 Mesh（清除当前选择）
        bpy.ops.object.select_all(action='DESELECT')
        for obj in meshes:
            obj.select_set(True)

        # 导出 FBX
        clean_name = sanitize_name(filename)
        fbx_path = os.path.join(out_dir, f"{clean_name}.fbx")
        bpy.ops.export_scene.fbx(filepath=fbx_path, **FBX_EXPORT_PRESET)

        results["exported"].append({
            "fbx": fbx_path,
            "mesh_count": len(meshes),
            "mesh_names": [m.name for m in meshes],
        })

    except Exception as e:
        results["errors"].append(str(e))

    return results

def batch_export(input_dir: str, output_dir: str):
    """批量导出入口"""
    os.makedirs(output_dir, exist_ok=True)

    # 加载已完成的日志（断点续传）
    completed = set()
    if os.path.exists(log_file):
        with open(log_file, "r") as f:
            for entry in json.load(f).get("completed", []):
                completed.add(entry)

    # 收集所有 .blend 文件
    blend_files = []
    for root, dirs, files in os.walk(input_dir):
        for f in files:
            if f.endswith(".blend") and not f.endswith(".blend1"):
                blend_files.append(os.path.join(root, f))

    total = len(blend_files)
    print(f"\n{'='*60}")
    print(f"批量导出任务: {total} 个 .blend 文件")
    print(f"输入目录: {input_dir}")
    print(f"输出目录: {output_dir}")
    print(f"{'='*60}\n")

    all_results = []
    success_count = 0
    fail_count = 0
    start_time = time.time()

    for i, blend_path in enumerate(blend_files):
        # 跳过已完成
        if blend_path in completed:
            print(f"[{i+1}/{total}] SKIP (already done): {blend_path}")
            continue

        print(f"[{i+1}/{total}] Processing: {blend_path}")
        result = export_single_blend(blend_path, output_dir)
        all_results.append(result)

        if result["errors"]:
            fail_count += 1
            print(f"  ❌ ERRORS: {result['errors']}")
        else:
            success_count += 1
            completed.add(blend_path)
            print(f"  ✅ Exported {result['exported'][0]['mesh_count']} mesh(es)")

        # 每处理 10 个文件保存一次日志（防中断丢失）
        if (i + 1) % 10 == 0:
            elapsed = time.time() - start_time
            rate = (i + 1) / elapsed
            eta = (total - i - 1) / rate
            print(f"  ⏱️ Progress: {i+1}/{total} | Rate: {rate:.1f} files/s | ETA: {eta:.0f}s")

            with open(log_file, "w") as f:
                json.dump({"completed": list(completed)}, f)

    # 最终日志
    elapsed = time.time() - start_time
    with open(log_file, "w") as f:
        json.dump({
            "completed": list(completed),
            "results": all_results,
            "summary": {
                "total": total,
                "success": success_count,
                "fail": fail_count,
                "elapsed_seconds": elapsed,
            }
        }, f, ensure_ascii=False, indent=2)

    print(f"\n{'='*60}")
    print(f"✅ 完成: {success_count} 成功, {fail_count} 失败, 耗时 {elapsed:.1f}s")
    print(f"{'='*60}")

# 运行
batch_export(input_dir, output_dir)
```

**Unity 侧自动导入脚本（FBXPostprocessor.cs）：**

```csharp
using UnityEngine;
using UnityEditor;

public class FBXPostprocessor : AssetPostprocessor
{
    static readonly string PROPS_PATH = "Assets/Art/Props";

    // FBX 导入后回调
    void OnPostprocessModel(GameObject root)
    {
        if (!assetPath.StartsWith(PROPS_PATH)) return;

        // 自动配置材质
        var renderer = root.GetComponentInChildren<Renderer>();
        if (renderer != null)
        {
            foreach (var mat in renderer.sharedMaterials)
            {
                if (mat == null) continue;

                // 根据材质名匹配 Shader 模板
                string matName = mat.name.ToLower();

                if (matName.Contains("opaque"))
                    mat.shader = Shader.Find("Universal Render Pipeline/Lit");
                else if (matName.Contains("transparent") || matName.Contains("glass"))
                    mat.shader = Shader.Find("Universal Render Pipeline/Simple Lit");
                else if (matName.Contains("foliage") || matName.Contains("grass"))
                    mat.shader = Shader.Find("Custom/Nature/Foliage");

                // 设置材质关键词
                mat.SetKeyword("_ALPHATEST_ON", matName.Contains("cutout"));
            }
        }
    }

    // 所有资源导入完成后回调
    static void OnPostprocessAllAssets(string[] importedAssets, string[] deletedAssets,
                                        string[] movedAssets, string[] movedFromAssetPaths)
    {
        foreach (string path in importedAssets)
        {
            if (!path.EndsWith(".fbx")) continue;
            if (!path.StartsWith(PROPS_PATH)) continue;

            // 自动生成 Prefab
            string prefabDir = path.Replace(".fbx", ".prefab");
            GameObject fbxRoot = AssetDatabase.LoadAssetAtPath<GameObject>(path);

            if (fbxRoot != null && !AssetDatabase.LoadAssetAtPath<GameObject>(prefabDir))
            {
                PrefabUtility.SaveAsPrefabAsset(fbxRoot, prefabDir);
                Debug.Log($"[AutoPrefab] Created prefab: {prefabDir}");
            }
        }
    }
}
```

**FBX 导出参数对比表：**

| 参数 | Blender 默认 | Unity 推荐值 | 原因 |
|------|------------|------------|------|
| axis_up | Z | Y | Unity 是 Y-up |
| axis_forward | -Y | -Z | 统一朝向 |
| mesh_smooth_type | OFF | FACE | 避免平滑组丢失 |
| path_mode | AUTO | COPY | 纹理跟随 FBX |
| embed_textures | True | False | Unity 重新导入纹理更可控 |
| use_mesh_modifiers | False | True | 烘焙修改器结果 |
| object_types | ALL | {MESH} | 只导出网格，灯光相机不要 |

### ⚡ 实战经验

- **Blender Headless 模式**：`blender --background --python script.py` 可以无界面运行，CI/CD 里也能用。比 GUI 模式快 30%+
- **FBX 材质粉色问题**：Unity 导入 FBX 时如果材质变粉，通常是 Shader 找不到。检查：① FBX 材质模式是否用了 Principled BSDF ② Unity 默认材质是否设为 URP/Lit（不是 Standard）
- **命名规范前置**：在脚本跑之前，先让美术确认命名规则。跑完发现命名不对再改，所有输出都要重来
- **Blender 版本兼容**：bpy API 在 2.8 → 3.x → 4.x 有变化（比如 `bpy.ops.export_scene.fbx` 参数名微调），脚本要注明测试版本
- **批量任务做日志**：200 个文件总会有几个出问题。日志要详细到文件名 + 错误原因，方便定位修复后重新跑那几个

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 bpy 怎么批量操作 | Blender Python API 不熟 | 学 bpy.data / bpy.ops 核心模块 |
| FBX 导入 Unity 朝向不对 | 坐标系变换理解不足 | 学 Blender/Unity 坐标系差异 |
| 材质变粉色不会排查 | 材质管线理解不足 | 学 Unity 材质导入流程和 Shader 匹配 |
| 200 个文件跑到一半挂了 | 缺少容错和断点续传 | 学 try/except + 日志 checkpoint |
| 不知道 Unity 能自动处理 | AssetPostprocessor 不熟 | 学 Unity 资产导入回调 API |

### 🔗 相关问题

- 如果美术用的是 Maya 而不是 Blender，你的方案需要改什么？（提示：Maya 用 Python + cmds/OpenMaya API）
- 如何把这个工具做成带 UI 的 Blender 插件，让美术自己点击操作？
- 如果需要导出 LOD 信息（每个 Mesh 带 LOD Group），FBX 怎么组织？Unity 怎么接收？
