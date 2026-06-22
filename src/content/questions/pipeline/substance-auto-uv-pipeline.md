---
title: "Substance + 自动展开 UV：如何搭建从高模到游戏资产的材质烘焙管线？"
category: "pipeline"
level: 2
tags: ["Substance Painter", "UV自动展开", "材质烘焙", "工具管线", "RizomUV", "低模生成", "资产流水线"]
hint: "核心是打通「高模→低模→UV展开→烘焙→材质绘制→引擎导入」全链路自动化，减少手工环节"
related: ["pipeline/houdini-vegetation-scatter", "technical-art/pbr-material-authoring", "pipeline/maya-lod-automation"]
---

## 参考答案

### 🎬 场景描述

面试官给你看美术团队的当前工作流：

> "我们项目角色资产制作流程太慢了，一个角色从高模到引擎内成品平均 3 天。主要瓶颈在：
> 1. 手动展 UV 要半天，美术抱怨枯燥
> 2. 高模到低模拓扑靠手工，不统一
> 3. Substance 烘焙后贴图命名不统一，导入引擎老出错
> 4. 没有统一的材质模板，每个美术各做各的
>
> 你来设计一条自动化管线，把 3 天压缩到 1 天。"

### ✅ 核心要点

1. **UV 展开自动化**：用 RizomUV CLI 或 Houdini Auto-UV 批处理替代手工展 UV
2. **低模生成标准化**：用 Maya/Houdini 的 Reduce + 规范约束，不依赖手工拓扑
3. **烘焙脚本化**：Substance Painter 的 Python API 或 Designer 的 Batch Tools
4. **命名规范 + 自动校验**：从文件名到贴图通道全链路统一，导入前脚本检查
5. **材质模板库**：预设 Smart Materials + Anchor 锚点系统，新资产 70% 复用

### 📖 深度展开

#### 解决思路（从最终效果倒推）

```
目标：3 天 → 1 天
  ↓ 时间花在哪了？
手动展 UV (4h) + 手动拓扑 (3h) + 手动烘焙 (2h) + 反复导入修错 (3h) + 绘制 (8h)
  ↓ UV 怎么自动化？
RizomUV 有 CLI 模式，可以脚本批量调用
  ↓ 低模怎么办？
不是所有资产都需要手工拓扑 → 用 ZRemesher / Maya Reduce + 规范约束
  ↓ 烘焙怎么自动化？
Substance Painter 有 Python API → 可以启动并配置烘焙参数
  ↓ 导入老出错？
命名规范 → 自动校验脚本 → 不合规则报错不让人工导入
  ↓ 材质绘制怎么提速？
建立 Smart Material 库 → 新资产 70% 区域用模板 → 只精修 30%
```

#### 知识点拆解（倒推树）

```
材质烘焙自动化管线
├── UV 展开
│   ├── RizomUV CLI（命令行批量展 UV）
│   ├── Houdini Auto-UV SOP（UV Flatten / UV Autoseam）
│   ├── Maya UV 工具链（Unfold3D 集成）
│   └── 规范检查：Texel Density / 重叠 / 紧凑度
├── 低模生成
│   ├── ZBrush ZRemesher（有机体）
│   ├── Maya Reduce（硬表面）
│   ├── Houdini PolyReduce（批量）
│   └── 质量检查：三角形数 / UV 翻转 / 法线
├── 烘焙自动化
│   ├── Substance Painter Python API
│   │   ├── 启动 SP 加载低模 + 高模
│   │   ├── 配置 Bake Maps（Normal/AO/Curvature/Position）
│   │   ├── 执行烘焙并保存
│   │   └── 导出预设（PBR MetalRough / SpecGloss）
│   ├── Substance Designer Batch Tools
│   │   ├── .sbs 文件批量编译
│   │   └── Command Line 渲染输出
│   └── Marmoset Toolbag CLI（备选方案）
├── 命名规范
│   ├── 资产名_LOD0_BaseColor.png
│   ├── 资产名_LOD0_Normal.png
│   ├── 资产名_LOD0_ORM.png (AO + Rough + Metal 合并)
│   └── 校验脚本：Python 正则匹配 + 报告
├── 引擎导入
│   ├── Unity：AssetPostprocessor 脚本
│   ├── Unreal：Import Pipeline（Texture / Material）
│   └── 自动设置：sRGB / Compression / Mipmap
└── 材质模板库
    ├── Smart Materials（SP 预设）
    ├── Anchor 锚点系统（统一修改入口）
    ├── 材质变体（Base → 做旧 → 风化）
    └── 按品类划分（金属/布料/皮革/木头）
```

#### 代码实现

**RizomUV CLI 批量展开 UV（Python）：**

```python
# auto_uv_pipeline.py
# 批量 UV 展开：遍历指定目录下的 .fbx 文件，调用 RizomUV 展 UV

import subprocess
import os
import shutil

RIZOMUV_PATH = r"C:\Program Files\Rizom-Lab\RizomUV VS 2022\rizomuv.exe"
INPUT_DIR = r"D:\assets\high_to_low"
OUTPUT_DIR = r"D:\assets\uv_done"
TEMP_DIR = r"D:\assets\temp"

def batch_unwrap_uv():
    """遍历所有 FBX，批量展 UV"""
    fbx_files = [f for f in os.listdir(INPUT_DIR) if f.endswith('.fbx')]

    for fbx in fbx_files:
        input_path = os.path.join(INPUT_DIR, fbx)
        output_path = os.path.join(OUTPUT_DIR, fbx)
        temp_path = os.path.join(TEMP_DIR, fbx)

        # 复制到临时目录
        shutil.copy2(input_path, temp_path)

        # RizomUV CLI 调用
        # --silent: 无 GUI 运行
        # --runlua: 执行 Lua 脚本控制展 UV 参数
        lua_script = f"""
        rs.LoadFile("{temp_path}")
        rs.Unfold()
        rs.Pack()
        rs.Optimize()
        rs.SaveFile("{output_path}")
        """

        lua_path = os.path.join(TEMP_DIR, fbx.replace('.fbx', '_uv.lua'))
        with open(lua_path, 'w') as f:
            f.write(lua_script)

        cmd = [
            RIZOMUV_PATH,
            "--silent",
            f"--runlua:{lua_path}",
            "--quit"  # 执行完退出
        ]

        result = subprocess.run(cmd, capture_output=True, timeout=120)
        if result.returncode != 0:
            print(f"❌ UV 展开失败: {fbx}")
            print(result.stderr.decode())
        else:
            print(f"✅ UV 展开完成: {fbx}")

def validate_uv_quality(fbx_path):
    """UV 质量检查：重叠 / Texel Density / 利用率"""
    # 用 Maya/Python 打开检查
    import maya.cmds as cmds
    cmds.file(fbx_path, force=True, open=True)

    issues = []

    meshes = cmds.ls(type='mesh')
    for mesh in meshes:
        uvs = cmds.polyEvaluate(mesh, uv=True)
        # 检查 UV 利用率（0-1 空间填充比例）
        uvs_in_range = cmds.polyEvaluate(mesh, uvArea=True)
        if uvs_in_range < 0.5:
            issues.append(f"{mesh}: UV 利用率 {uvs_in_range:.1%}，太低")

        # 检查 UV 重叠
        overlapping = cmds.polyUVOverlapping(mesh, checkAllUvs=True)
        if overlapping:
            issues.append(f"{mesh}: 有 {len(overlapping)} 处 UV 重叠")

    return issues

if __name__ == "__main__":
    batch_unwrap_uv()
```

**Substance Painter 烘焙自动化（Python API）：**

```python
# sp_bake_pipeline.py
# 在 Substance Painter 内运行的 Python 脚本
# 通过命令行启动：Substance Painter.exe --mesh model.fbx --script sp_bake_pipeline.py

import substance_painter.ui as spui
import substance_painter.textureset as ts
import substance_painter.baking as baking
import substance_painter.export as export
import os

def setup_bake_config(high_mesh_path, output_size=2048):
    """配置烘焙参数"""
    config = {
        "mapSize": output_size,
        "antialiasing": True,
        "keepExistingBakedMaps": False,
        "mesh": {
            "highPoly": [
                {"name": "high", "path": high_mesh_path}
            ]
        },
        "maps": [
            {"type": "Normal", "format": "png", "bitDepth": 16},
            {"type": "AmbientOcclusion", "format": "png", "bitDepth": 8},
            {"type": "Curvature", "format": "png", "bitDepth": 8},
            {"type": "Position", "format": "png", "bitDepth": 8},
            {"type": "Thickness", "format": "png", "bitDepth": 8},
            {"type": "WorldSpaceNormal", "format": "png", "bitDepth": 16},
        ]
    }
    return config

def bake_maps(config):
    """执行烘焙"""
    print("🔥 开始烘焙...")
    baking.bake(config)
    print("✅ 烘焙完成")

def export_textures(output_dir, asset_name):
    """按命名规范导出贴图"""
    export_config = {
        "preset": "Unity5Standard",  # 或 Unreal4 / Custom
        "exports": [
            {
                "destPath": os.path.join(output_dir, f"{asset_name}_LOD0"),
                "fileName": "{textureMap}_{assetName}",
                "parameters": {
                    "bitDepth": 8,
                    "dithering": True,
                }
            }
        ]
    }
    export.export_project(export_config)
    print(f"📦 导出完成: {output_dir}/{asset_name}_LOD0")

# 执行
if __name__ == "__main__":
    high_mesh = os.environ.get("HIGH_MESH_PATH", "high.fbx")
    asset_name = os.environ.get("ASSET_NAME", "hero_armor")

    config = setup_bake_config(high_mesh, output_size=2048)
    bake_maps(config)
    export_textures(r"D:\assets\exported", asset_name)
```

**贴图命名规范自动校验脚本：**

```python
# validate_textures.py
# 在导入引擎前检查贴图命名是否合规

import re
import os
import sys

PATTERN = re.compile(
    r'^(?P<asset>[A-Z][a-z0-9_]+)_LOD(?P<lod>\d)_(?P<map>BaseColor|Normal|ORM|Emissive|Metallic|Roughness)\.(?P<ext>png|tga)$'
)

REQUIRED_MAPS = {"BaseColor", "Normal", "ORM"}

def validate_directory(directory):
    """校验目录下的贴图命名"""
    files = [f for f in os.listdir(directory) if f.endswith(('.png', '.tga'))]

    # 按资产分组
    assets = {}
    for f in files:
        m = PATTERN.match(f)
        if not m:
            print(f"❌ 命名不合规: {f}")
            continue
        asset_key = f"{m.group('asset')}_LOD{m.group('lod')}"
        assets.setdefault(asset_key, []).append(m.group('map'))

    # 检查每个资产是否有必需贴图
    errors = []
    for asset_key, maps in assets.items():
        missing = REQUIRED_MAPS - set(maps)
        if missing:
            errors.append(f"❌ {asset_key} 缺少贴图: {missing}")
        else:
            print(f"✅ {asset_key} 贴图完整: {maps}")

    if errors:
        print("\n🚨 校验失败，请修正后再导入引擎:")
        for e in errors:
            print(e)
        sys.exit(1)
    else:
        print("\n🎉 所有贴图校验通过!")

if __name__ == "__main__":
    validate_directory(sys.argv[1])
```

**管线流程图：**

```
高模（ZBrush .ztl）
  ↓ ZRemesher / 手工拓扑
低模（Maya .ma）
  ↓ auto_uv_pipeline.py
低模 + UV（.fbx）
  ↓ Substance Painter Python API
烘焙贴图（Normal/AO/Curvature/Thickness）
  ↓ Smart Material 模板库
材质绘制（70% 模板 + 30% 手工）
  ↓ export + validate_textures.py
贴图包（命名规范校验）
  ↓ Unity AssetPostprocessor
引擎资产（自动配置 Texture Importer）

时间统计：
  手工流程：高模(4h) + 拓扑(3h) + UV(4h) + 烘焙(2h) + 绘制(8h) + 导入修错(3h) = 24h
  自动管线：高模(4h) + 拓扑(1h) + UV(0.5h) + 烘焙(0.5h) + 绘制(4h) + 导入(0.5h) = 10.5h
  效率提升：~56%
```

### ⚡ 实战经验

1. **UV 自动展不是万能的**：硬表面机械类资产的 UV 切线需要人工指定接缝位置，全自动展 UV 在硬表面效果差。策略：有机体（角色/生物）全自动，硬表面（武器/建筑）半自动 + 接缝预设
2. **Substance Painter 启动慢是瓶颈**：SP 启动 + 加载高模要 30s~1min，批量处理时考虑用 Substance Designer 的 Batch Tools 替代，无需 GUI
3. **ORM 合并贴图省内存**：AO（R）+ Roughness（G）+ Metallic（B）合并为一张贴图，减少 2 张贴图的带宽和内存，移动端尤其重要
4. **Texel Density 校验必须加**：不同资产的 UV 比例不一致会导致游戏内分辨率差异明显，管线中必须加入 Texel Density 校验环节（通常 1024px/m 角色 / 512px/m 环境）

### 🎯 能力体检清单

| 卡住的环节 | 盲区在哪 | 补习建议 |
|------------|----------|----------|
| 不知道 RizomUV 有 CLI | 缺乏工具链调研 | 看 RizomUV 官方文档 CLI 章节，或研究 Houdini UV Flatten |
| Substance 烘焙不会脚本化 | 不了解 SP Python API | 官方文档有完整 API 参考，先跑通官方示例 |
| 命名规范执行不下去 | 没有自动化校验 | 写校验脚本，挂在 Git Hook 或 CI 上强制执行 |
| UV 展开质量不稳定 | 不理解 UV 接缝策略 | 学习 UV Seams 原则：隐藏面、对称线、密度均匀 |
| Texel Density 不知道怎么控 | UV 空间分配概念缺失 | 用 Texel Density Checker 插件（Maya/Blender 都有） |

### 🔗 相关问题

- [Houdini 批量植被生成](pipeline/houdini-vegetation-scatter.md) — 另一类管线自动化思路
- [PBR 材质编写](technical-art/pbr-material-authoring.md) — 材质管线的基础知识
- Substance Smart Material 的 Anchor 系统怎么用？（延展：非破坏式材质编辑流程）
