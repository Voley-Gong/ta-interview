---
title: "美术每次手动导出贴图耗时且出错？Substance Painter 批量导出自动化管线"
category: pipeline
level: 2
tags: ["Substance Painter", "自动化导出", "Export Preset", "Python", "纹理管线", "PyShade"]
hint: "用 SP 的 Python API + 自定义 Export Preset 实现「一键导出 → 重命名 → 压缩 → 入库」全流程"
related: ["substance-auto-uv-pipeline", "batch-texture-compression-tool", "scriptableobject-material-template-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官说："我们项目角色资产量大，美术在 Substance Painter 里画完贴图后，每次都要手动设置导出格式、命名、Packing Channel，然后手动拖进 Unity/Unreal。经常出现：导出格式选错、命名不统一、Channel Packing 拼错、忘了压缩就直接入库。你来设计一套自动化导出管线解决这个问题。"

### ✅ 核心要点

- **问题本质**：人工导出步骤多、易出错、无法批量处理
- **解决方向**：Substance Painter Python API + Export Preset + 后处理脚本
- **技术栈**：SP Python API（`substance_painter` 模块）→ Export Preset（`.spexp`）→ Python 后处理（重命名 + Channel Packing）→ CI 入库
- **价值**：一键导出 + 统一规范 + 零人为错误

### 📖 深度展开

#### 解决思路（从效果倒推实现）

**最终效果**：美术在 SP 中点击一个按钮（或按快捷键），所有贴图自动按项目规范导出到指定目录、命名正确、通道拼合完成、压缩完成、自动复制到引擎资源目录。

**倒推链**：
1. 要一键完成 → SP Python 插件提供 UI 按钮 + 调用 SP 导出 API
2. 导出格式统一 → 预定义 Export Preset（.spexp 文件），规定输出哪些贴图、什么格式
3. 命名规范 → Python 后处理脚本重命名文件
4. Channel Packing → 导出时用 SP 的 Preset 自动拼合（如 ORM = Occlusion/ Roughness/ Metallic）
5. 压缩 → 后处理脚本调用压缩工具（如 ISPC Texture Compressor / nvcompress）
6. 入库 → 脚本复制到引擎 Asset 目录，触发 Meta 生成

#### 知识点拆解（倒推树）

```
Substance Painter 自动化导出管线
├── 1. SP Export Preset（.spexp）
│   ├── 定义输出贴图列表（BaseColor, Normal, ORM, Emissive...）
│   ├── Channel Packing 规则（R=AO, G=Roughness, B=Metallic）
│   ├── 输出格式（PNG / EXR / TIFF）
│   └── 分辨率设置（2K / 4K）
├── 2. SP Python API
│   ├── substance_painter.export 模块
│   ├── export_project_textures() 函数
│   ├── 自定义 Export Path / Naming Convention
│   └── 插件 UI（QToolBar / QAction）
├── 3. 后处理脚本
│   ├── 文件重命名（按项目规范）
│   ├── 格式转换（PNG → TGA / DDS）
│   ├── Channel Packing 后验（如果 SP 没拼好）
│   └── 压缩（ASTC / BC7 / ETC2）
├── 4. 引擎入库
│   ├── 复制到 Unity Assets/ 或 Unreal Content/
│   ├── 触发 Asset Import（Unity: AssetDatabase.ImportAsset）
│   └── Meta 文件 / Import Settings 自动配置
└── 5. 批量处理
    ├── 批量打开 .spp 文件 → 导出 → 关闭
    ├── 命令行模式（substance_painter --batch-export）
    └── CI/CD 集成（提交 .spp → 自动导出入库）
```

#### 代码实现

**SP Python 插件：一键导出**：

```python
# substance_painter_auto_export.py
# 放置在 SP 插件目录或通过 Plugin Manager 加载

import substance_painter as sp
import substance_painter.export as sp_export
import substance_painter.ui as sp_ui
import os
import shutil
import json

# ===== 配置 =====
CONFIG = {
    "export_root": "D:/Project/Art/Textures/Characters/",
    "engine_asset_path": "D:/Project/Unity/Assets/Art/Characters/",
    "texture_size": "2048",
    "compression_format": "ASTC",  # or BC7, ETC2
    "naming_convention": "{mesh}_{material}_{texture_type}_v{version}",
}

# Export Preset 配置（等价于 .spexp 文件内容）
def build_export_preset():
    return sp_export.ExportConfig(
        export_shader_params=False,
        export_path=CONFIG["export_root"],
        export_list=[
            # ORM Packed (R=AO, G=Roughness, B=Metallic)
            sp_export.ResourceUsage(
                labels=["AO", "Roughness", "Metallic"],
                file_name="{meshName}_ORM",
                channels=["R", "G", "B"],
            ),
            # Base Color
            sp_export.ResourceUsage(
                labels=["BaseColor"],
                file_name="{meshName}_BaseColor",
            ),
            # Normal (OpenGL)
            sp_export.ResourceUsage(
                labels=["Normal"],
                file_name="{meshName}_Normal",
            ),
            # Emissive
            sp_export.ResourceUsage(
                labels=["Emissive"],
                file_name="{meshName}_Emissive",
            ),
        ],
        export_parameters=[
            sp_export.ExportParameter(
                parameter=sp_export.ExportProperties.PixelSize,
                value=int(CONFIG["texture_size"]),
            ),
        ],
        default_export_preset="Unity 5 (Metallic)",  # 使用内置 Preset 作为基础
    )


def on_export_clicked():
    """一键导出按钮回调"""
    # 1. 获取当前项目信息
    project_name = sp.project.ProjectName()
    mesh_name = project_name.replace(".spp", "")

    print(f"[AutoExport] Starting export for: {mesh_name}")

    # 2. 执行 SP 导出
    export_config = build_export_preset()
    export_path = CONFIG["export_root"]

    try:
        result = sp_export.export_project_textures(
            export_config
        )
        print(f"[AutoExport] SP Export result: {result}")
    except Exception as e:
        sp_ui.MessageBox.critical(
            "Export Failed",
            f"Export error: {str(e)}"
        )
        return

    # 3. 后处理：重命名 + 压缩 + 入库
    post_process_exported_textures(export_path, mesh_name)

    # 4. 通知美术
    sp_ui.MessageBox.information(
        "Export Complete",
        f"Textures exported and processed to:\n{CONFIG['engine_asset_path']}"
    )


def post_process_exported_textures(export_dir, mesh_name):
    """后处理：重命名 → 压缩 → 复制到引擎"""
    # --- 重命名 ---
    # SP 导出可能有额外后缀，统一规范化
    expected_files = [
        f"{mesh_name}_ORM.png",
        f"{mesh_name}_BaseColor.png",
        f"{mesh_name}_Normal.png",
        f"{mesh_name}_Emissive.png",
    ]

    for f in os.listdir(export_dir):
        f_path = os.path.join(export_dir, f)
        if not os.path.isfile(f_path):
            continue

        # 统一为小写 + 下划线
        new_name = f.lower().replace(" ", "_").replace("-", "_")
        if new_name != f:
            os.rename(f_path, os.path.join(export_dir, new_name))
            print(f"[AutoExport] Renamed: {f} → {new_name}")

    # --- 压缩（调用外部工具示例）---
    # 假设使用 ISPC Texture Compressor
    compressor_path = "D:/Tools/ispc_texture_compressor/ispc_texcomp.exe"
    for f in expected_files:
        f_path = os.path.join(export_dir, f)
        if not os.path.isfile(f_path):
            continue

        if "Normal" in f or "_N." in f:
            fmt = "BC5"  # 法线贴图用 BC5
        elif "ORM" in f:
            fmt = "BC7"  # ORM 用 BC7
        else:
            fmt = CONFIG["compression_format"]

        compressed_path = f_path.replace(".png", ".dds")
        cmd = f'"{compressor_path}" -f {fmt} -i "{f_path}" -o "{compressed_path}"'
        os.system(cmd)
        print(f"[AutoExport] Compressed: {f} → {fmt}")

    # --- 复制到引擎 ---
    engine_dir = CONFIG["engine_asset_path"]
    os.makedirs(engine_dir, exist_ok=True)

    for f in os.listdir(export_dir):
        if f.endswith((".dds", ".png")):
            src = os.path.join(export_dir, f)
            dst = os.path.join(engine_dir, f)
            shutil.copy2(src, dst)
            print(f"[AutoExport] Copied to engine: {f}")

    print("[AutoExport] Post-processing complete!")


# ===== 插件初始化 =====
def start_plugin():
    """SP 启动时加载插件"""
    # 创建工具栏按钮
    toolbar = sp_ui.get_main_toolbar()

    export_action = sp_ui.QAction("🚀 Auto Export", None)
    export_action.triggered.connect(on_export_clicked)
    toolbar.addAction(export_action)

    print("[AutoExport] Plugin loaded successfully!")


def close_plugin():
    """SP 关闭时清理"""
    print("[AutoExport] Plugin unloaded.")


# 注册插件
sp.plugin.register_plugin(
    start=start_plugin,
    close=close_plugin,
    name="Auto Export Pipeline",
    version="1.0.0",
)
```

**批量命令行导出（CI/CD 场景）**：

```bash
#!/bin/bash
# batch_export_spp.sh
# 批量打开 .spp 文件并执行导出脚本

SUBSTANCE_PAINTER="/c/Program Files/Adobe/Adobe Substance 3D Painter/painter.exe"
SP_PLUGIN_DIR="$HOME/Documents/Adobe/Adobe Substance 3D Painter/plugins/auto_export"

ART_DIR="/d/Project/Art/Source_SP"
EXPORT_DIR="/d/Project/Art/Textures/Characters"

# 遍历所有 .spp 文件
for spp_file in "$ART_DIR"/*.spp; do
    filename=$(basename "$spp_file" .spp)
    echo "Processing: $filename"

    # 使用 SP 命令行模式打开并导出
    "$SUBSTANCE_PAINTER" \
        --mesh "$(find "$ART_DIR" -name "${filename}.fbx" | head -1)" \
        --export-path "$EXPORT_DIR" \
        --export-preset "Project_Custom_Preset.spexp" \
        "$spp_file" \
        --enable-plugin "$SP_PLUGIN_DIR/auto_export.py" \
        --quit

    echo "Done: $filename"
done

echo "Batch export complete!"
```

**Export Preset（.spexp）关键结构**：

```json
{
  "name": "Project Custom Export",
  "maps": [
    {
      "name": "{meshName}_ORM",
      "channels": [
        {"source": "ao",        "dest": "R"},
        {"source": "roughness", "dest": "G"},
        {"source": "metallic",  "dest": "B"}
      ],
      "format": "png",
      "bitDepth": 8
    },
    {
      "name": "{meshName}_Normal",
      "channels": [
        {"source": "normal", "dest": "RGB"}
      ],
      "format": "png",
      "bitDepth": 8,
      "colorSpace": "linear"
    },
    {
      "name": "{meshName}_BaseColor",
      "channels": [
        {"source": "baseColor", "dest": "RGB"}
      ],
      "format": "png",
      "bitDepth": 8,
      "colorSpace": "sRGB"
    }
  ],
  "parameters": {
    "pixelSize": [2048, 2048],
    "padding": 16,
    "dithering": true
  }
}
```

### ⚡ 实战经验

1. **Export Preset 是核心**：不要让美术每次手动选输出贴图，预定义好项目标准的 .spexp 文件，放到版本控制里，所有人共用
2. **Channel Packing 在导出时做，不要后处理拼**：SP 的 Export Preset 原生支持通道拼合，比后处理用 Python/Pillow 拼更可靠
3. **版本号管理**：在命名规范中加入版本号 `_v01`，每次导出版本递增。结合版本控制可以追溯每次贴图变更
4. **法线贴图的颜色空间陷阱**：Normal/ORM 必须是 Linear，BaseColor/Emissive 是 sRGB。Export Preset 里一定要设对，否则导入引擎后亮度不对
5. **SP 10.0+ Python API 有变更**：Adobe 收购后 API 有 breaking change，从 `substance_painter` 模块迁移，注意版本兼容
6. **CI/CD 集成**：SP 支持命令行模式 `--export-path` + `--export-preset`，不需要 GUI 就能批量导出，适合 Jenkins/GitLab CI 跑夜间构建

### 🎯 能力体检清单

- [ ] 能解释 Channel Packing 的原理以及为什么能省贴图采样/内存
- [ ] 知道 Substance Painter 的 Export Preset 怎么配置和共享
- [ ] 了解 SP Python API 的基本使用（插件加载、export 调用）
- [ ] 能设计批量导出的命令行流程
- [ ] 面试追问："如果美术改了贴图但忘了重新导出怎么办？" → CI 钩子检测 .spp 文件变更自动触发导出，或用 SP 的 Live Link 直连引擎
- [ ] 面试追问："贴图导入引擎后 Import Settings 怎么自动化？" → Unity 用 `AssetPostprocessor`，Unreal 用 Import Task + Blueprint
- [ ] 理解法线贴图用 BC5 / ASTC 6x6 而非 BC7 的原因（法线 XY 分量精度需求）

### 🔗 相关问题

- [Substance 自动 UV 管线](../pipeline/substance-auto-uv-pipeline.md)
- [批量贴图压缩工具](../pipeline/batch-texture-compression-tool.md)
- [ScriptableObject 材质模板管线](../pipeline/scriptableobject-material-template-pipeline.md)
