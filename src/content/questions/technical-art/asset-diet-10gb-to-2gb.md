---
title: "10GB 美术资源压到 2GB 打进包：你作为 TA 怎么制定资产瘦身方案？"
category: "technical-art"
level: 4
tags: ["资产优化", "包体优化", "纹理压缩", "模型减面", "音频压缩", "实战项目"]
hint: "不是无脑压纹理——先审计分类、按类型分配预算、用工具批量处理、最后验证效果"
related: ["technical-art/mobile-texture-compression", "optimization/mobile-package-size-2gb-to-500mb", "technical-art/mobile-texture-pipeline-strategy"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们项目要上移动端，目前美术资源总计 10GB，要求打出来的包不超过 2GB（含代码和音视频）。你作为 TA，给我一套完整的资产瘦身方案——从审计到执行到验收。」

追问约束：
- 角色模型 200+ 个，场景资产 1000+ 件
- 纹理有 PNG 和 TGA 混用，部分 4K
- 音频有 WAV 原始文件直接放进来的
- 美术团队 20 人，你方案的执行不能全靠手动

### ✅ 核心要点

1. **审计先行**：用工具扫描全部资源，按类型/大小排序，找出 Top N 大头
2. **预算分配**：2GB 总预算 → 按模块分配（角色/场景/UI/音频/动画）
3. **纹理瘦身**：最大收益项——4K→2K/1K、格式转 ASTC、无用通道剥离
4. **模型瘦身**：LOD 链建立、减面、顶点色烘焙细节
5. **音频瘦身**：WAV→OGG，按使用场景分采样率
6. **自动化流水线**：写 Editor 工具批量处理，不靠手动改
7. **验收回归**：瘦身后逐模块对比截图/录屏，确保视觉品质可接受

### 📖 深度展开

#### 解决思路（从目标倒推步骤）

```
目标：10GB → 2GB（压到 20%）
              ↑
倒推1：10GB 里是什么？→ 审计分类（纹理占比? 模型? 音频?）
倒推2：每类压多少？→ 按预算分配（纹理3GB→0.8GB, 模型2GB→0.5GB...）
倒推3：怎么压？→ 每类有标准手段（ASTC/LOD/OGG）
倒推4：怎么批量执行？→ 自动化工具
倒推5：压完效果能接受吗？→ 验收流程
倒推6：如何防止美术再传大文件？→ 导入规范 + CI 检查
```

#### 知识点拆解（倒推树）

```
资产瘦身方案
├── 审计阶段
│   ├── Editor 工具：遍历 AssetDatabase，统计各类型占用
│   ├── 分类维度：纹理 / 模型 / 音频 / 动画 / 特效 / UI
│   ├── 排序：按文件大小 Top 50/Top 100
│   └── 识别异常：4K 纹理、未压缩音频、超高面模型
├── 预算分配
│   ├── 总预算 2GB 分配策略
│   │   ├── 纹理：40%（0.8GB）
│   │   ├── 模型：20%（0.4GB）
│   │   ├── 音频：15%（0.3GB）
│   │   ├── 动画：10%（0.2GB）
│   │   ├── UI + 特效：10%（0.2GB）
│   │   └── 代码 + 库：5%（0.1GB）
│   └── 按模块拆分：角色X%、场景Y%、UI Z%
├── 纹理瘦身（最大收益）
│   ├── 尺寸限制：角色 1K、场景 2K、UI 1K、特效 512
│   ├── 格式转换：ASTC 6x6（移动端标准）
│   │   ├── 法线贴图用 ASTC + 两通道（AG 编码）
│   │   └── UI 用 ASTC 8x8 或 ETC2
│   ├── 通道合并：ORM（Occlusion-Roughness-Metallic）合一
│   ├── Mipmap 策略：UI/字体关闭，3D 开启
│   ├── 无用通道剥离：Alpha 通道没用就删
│   └── Texture Packer / Atlas 合并小图
├── 模型瘦身
│   ├── LOD 链建立（LOD0: 15K, LOD1: 8K, LOD2: 3K, LOD3: 1K）
│   ├── 减面工具（Simplygon / Mesh Decimation）
│   ├── 顶点色烘焙细节（取代高分辨率法线）
│   ├── Mesh 压缩格式（Unity Mesh Compression）
│   └── Rig 减骨（非主角 30→20 骨骼）
├── 音频瘦身
│   ├── WAV → OGG Vorbis
│   ├── 采样率分级：BGM 44.1kHz / SFX 22kHz / Voice 16kHz
│   ├── 声道：BGM 立体声，SFX 单声道
│   ├── 按重要性分质量等级
│   └── Streaming vs 加载到内存（长音效 Streaming）
├── 动画瘦身
│   ├── Animation Compression（Key Reduction）
│   ├── 去除无用曲线（只保留关键骨骼）
│   ├── 动画融合 / 复用（走路→跑步 blend）
│   └── GPU Skinning 预算
├── 自动化工具
│   ├── AssetPostprocessor：导入时自动检查格式
│   ├── Editor Script：批量处理已有资源
│   ├── CI 检查：打包前扫描超标资产
│   └── 报告生成：Excel/HTML 看板
└── 验收流程
    ├── Before/After 截图对比（同机位同光照）
    ├── 重点检查：法线/粗糙度压缩后是否有瑕疵
    ├── 美术确认签字
    └── 持续监控：每版本包体报表
```

#### 代码实现

**审计工具——扫描全部资产并输出报告：**

```csharp
using UnityEditor;
using UnityEngine;
using System.IO;
using System.Collections.Generic;
using System.Text;

public class AssetAuditTool : EditorWindow
{
    [MenuItem("TA/资产审计报告")]
    static void ShowWindow() => GetWindow<AssetAuditTool>("资产审计");

    void OnGUI()
    {
        if (GUILayout.Button("生成审计报告 (CSV)"))
            GenerateAuditReport();
    }

    struct AssetInfo
    {
        public string path;
        public string type;
        public long sizeBytes;
    }

    static void GenerateAuditReport()
    {
        var assetPaths = AssetDatabase.GetAllAssetPaths();
        var entries = new List<AssetInfo>();

        foreach (var path in assetPaths)
        {
            if (!path.StartsWith("Assets/")) continue;

            string fullPath = Path.Combine(Application.dataPath, "..", path);
            if (!File.Exists(fullPath)) continue;

            var fi = new FileInfo(fullPath);
            string ext = Path.GetExtension(path).ToLower();

            string type = ext switch
            {
                ".png" or ".tga" or ".jpg" or ".psd" or ".exr" => "Texture",
                ".fbx" or ".obj" or ".blend" or ".maya" => "Model",
                ".wav" or ".mp3" or ".ogg" => "Audio",
                ".anim" or ".controller" => "Animation",
                ".mat" => "Material",
                ".prefab" => "Prefab",
                ".shader" or ".shadergraph" => "Shader",
                _ => "Other"
            };

            entries.Add(new AssetInfo { path = path, type = type, sizeBytes = fi.Length });
        }

        // 按类型汇总
        var byType = new Dictionary<string, long>();
        foreach (var e in entries)
        {
            if (!byType.ContainsKey(e.type)) byType[e.type] = 0;
            byType[e.type] += e.sizeBytes;
        }

        // 输出 CSV
        string csvPath = Path.Combine(Application.dataPath, "..", "asset_audit_report.csv");
        var sb = new StringBuilder();
        sb.AppendLine("Path,Type,Size(MB)");
        entries.Sort((a, b) => b.sizeBytes.CompareTo(a.sizeBytes));
        foreach (var e in entries)
            sb.AppendLine($"{e.path},{e.type},{e.sizeBytes / 1024f / 1024f:F2}");

        // 汇总
        sb.AppendLine();
        sb.AppendLine("--- SUMMARY ---");
        foreach (var kv in byType)
            sb.AppendLine($"TOTAL,{kv.Key},{kv.Value / 1024f / 1024f:F2}");

        File.WriteAllText(csvPath, sb.ToString());
        EditorUtility.OpenWithDefaultApp(csvPath);
        Debug.Log($"审计报告已生成: {csvPath}");
    }
}
```

**批量纹理处理工具——按规范压缩：**

```csharp
using UnityEditor;
using UnityEngine;

public class BatchTextureProcessor : EditorWindow
{
    [MenuItem("TA/批量纹理压缩")]
    static void ShowWindow() => GetWindow<BatchTextureProcessor>("批量纹理压缩");

    void OnGUI()
    {
        if (GUILayout.Button("执行：所有角色纹理 → ASTC 6x6, MaxSize 1024"))
            ProcessTextures("Assets/Art/Characters", 1024, TextureImporterFormat.ASTC_RGBA_6x6);

        if (GUILayout.Button("执行：所有场景纹理 → ASTC 6x6, MaxSize 2048"))
            ProcessTextures("Assets/Art/Environments", 2048, TextureImporterFormat.ASTC_RGBA_6x6);

        if (GUILayout.Button("执行：所有UI纹理 → ASTC 8x8, MaxSize 1024, 无Mipmap"))
            ProcessUITextures("Assets/Art/UI");
    }

    static void ProcessTextures(string folder, int maxSize, TextureImporterFormat format)
    {
        var guids = AssetDatabase.FindAssets("t:Texture2D", new[] { folder });
        int total = guids.Length;

        for (int i = 0; i < total; i++)
        {
            string path = AssetDatabase.GUIDToAssetPath(guids[i]);
            var importer = AssetImporter.GetAtPath(path) as TextureImporter;
            if (importer == null) continue;

            bool changed = false;

            // 尺寸限制
            if (importer.maxTextureSize > maxSize)
            {
                importer.maxTextureSize = maxSize;
                changed = true;
            }

            // Android 平台
            var androidSettings = importer.GetPlatformTextureSettings("Android");
            if (!androidSettings.overridden || androidSettings.format != format)
            {
                androidSettings.overridden = true;
                androidSettings.format = format;
                androidSettings.maxTextureSize = maxSize;
                importer.SetPlatformTextureSettings(androidSettings);
                changed = true;
            }

            // iOS 平台
            var iosSettings = importer.GetPlatformTextureSettings("iPhone");
            if (!iosSettings.overridden || iosSettings.format != format)
            {
                iosSettings.overridden = true;
                iosSettings.format = format;
                iosSettings.maxTextureSize = maxSize;
                importer.SetPlatformTextureSettings(iosSettings);
                changed = true;
            }

            if (changed)
            {
                importer.SaveAndReimport();
            }

            if (i % 50 == 0)
                EditorUtility.DisplayProgressBar("处理纹理", path, (float)i / total);
        }

        EditorUtility.ClearProgressBar();
        Debug.Log($"处理完成: {folder} ({total} 张纹理)");
    }

    static void ProcessUITextures(string folder)
    {
        var guids = AssetDatabase.FindAssets("t:Texture2D", new[] { folder });
        foreach (var guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            var importer = AssetImporter.GetAtPath(path) as TextureImporter;
            if (importer == null) continue;

            // UI 不需要 Mipmap
            importer.mipmapEnabled = false;

            // Android
            var android = importer.GetPlatformTextureSettings("Android");
            android.overridden = true;
            android.format = TextureImporterFormat.ASTC_RGBA_8x8;
            android.maxTextureSize = 1024;
            importer.SetPlatformTextureSettings(android);

            // iOS
            var ios = importer.GetPlatformTextureSettings("iPhone");
            ios.overridden = true;
            ios.format = TextureImporterFormat.ASTC_RGBA_8x8;
            ios.maxTextureSize = 1024;
            importer.SetPlatformTextureSettings(ios);

            importer.SaveAndReimport();
        }
        Debug.Log($"UI 纹理处理完成: {folder}");
    }
}
```

**对比表格：各类资产压缩策略与预期收益**

| 资产类型 | 原始占比 | 压缩手段 | 预期压缩率 | 风险点 |
|----------|----------|----------|-----------|--------|
| 纹理 (4K PNG) | ~40% | ASTC 6x6 + MaxSize | 80-90% ↓ | 法线/粗糙度可能有 banding |
| 模型 (高面) | ~20% | LOD + 减面 + 压缩 | 50-60% ↓ | 轮廓品质下降 |
| 音频 (WAV) | ~15% | OGG + 分级采样率 | 85-90% ↓ | BGM 有微妙细节损失 |
| 动画 | ~10% | Key Reduction + 去冗余曲线 | 40-50% ↓ | 手指/面部动画可能穿模 |
| 特效 | ~5% | 粒子贴图合并 + 减粒子 | 30-40% ↓ | 效果层次感降低 |

### ⚡ 实战经验

- **先审计再动手**：上来就改格式是灾难。一定要先跑审计报告，知道10GB里什么占大头。通常是纹理 40-50%、音频 15-20%、模型 15-20%
- **ASTC 6x6 是移动端甜点**：4x4 质量好但太大，8x8 太糊，6x6 是品质/大小的平衡点。法线贴图考虑 5x5 或 4x4
- **音频是隐形大户**：WAV 文件经常被美术忽略。10分钟BGM的WAV约100MB，转OGG 80kbps只有6MB——**单这一项就可能省下 80%+ 音频体积**
- **别忘记 Sprite Atlas**：散碎的 UI PNG 合成 Atlas 后，DrawCall 和包体双赢
- **CI 防线**：瘦身做完不是结束，要写 CI 脚本检查新导入资源是否超限（如：单张纹理 > 2048 或未设 ASTC → 报警）
- **美术沟通**：不要自己偷偷改完让美术猜。提前约定规范，每类资产给美术一份「这是压缩后效果截图」，让他们提前接受

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道从哪里开始 | 缺乏系统性方法论 | 学资产审计流程、按类型分类统计 |
| 纹理压完出现色带 | 不理解 ASTC 压缩原理 | 学 ASTC block size vs 质量 trade-off |
| 模型减面后轮廓丑 | LOD 和减面策略不对 | 学 Simplygon / Mesh Decimation 原理 |
| 瘦身后包体没怎么变 | 漏了 StreamingAssets 或 Addressables | 学 Unity 资产打包机制和 Build Report |
| 美术反复返工 | 没有前置规范 | 学 AssetPostprocessor + 导入预设 |

### 🔗 相关问题

- [移动端贴图压缩方案选型](../technical-art/mobile-texture-compression.md)：纹理压缩的基础知识
- [手游包体从2GB降到500MB](../optimization/mobile-package-size-2gb-to-500mb.md)：更极端的包体优化案例
- 面试追问：如果瘦身到2GB后，运行时内存峰值还是超了（手机闪退），你怎么继续优化？（提示：资产按需加载、Streaming、纹理池化）
