---
title: "美术资产规范化：如何做一个「一键检查并修复」的编辑器工具？"
category: "pipeline"
level: 3
tags: ["Unity Editor", "AssetPostprocessor", "自动化", "美术规范", "工具开发"]
hint: "AssetPostprocessor 预处理 + 编辑器批量检查 + 一键修复——三层防线杜绝不规范资产"
related: ["pipeline/unity-asset-checker-tool", "pipeline/batch-material-audit-tool", "technical-art/lod-spec-and-qa"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们团队 20 个美术，每周导入上百个模型/贴图/材质，经常出现：贴图没有压缩、模型没生成 LOD、材质引用丢失、命名不规范等问题。现在靠人工 QA，耗时且漏检率高。你作为 TA，设计一套自动化的美术资产规范检查+修复工具。」

### ✅ 核心要点

1. **AssetPostprocessor 拦截**：资产导入时自动检查/修改，问题根本不进项目
2. **批量扫描工具**：已导入的存量资产可以一键扫描，生成违规报告
3. **一键修复**：常见问题（贴图压缩、LOD 生成、命名）可以自动批量修复
4. **规范配置化**：规则不写死在代码里，用 ScriptableObject 或 JSON 配置
5. **CI 集成**：Pre-commit / Build 前自动跑检查，阻断不合规资产

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：不规范的资产根本进不了项目 + 存量问题一键修复
                ↑
倒推1：「预防」导入时拦截 → AssetPostprocessor.OnPostprocessModel / OnPostprocessTexture
倒推2：「检测」已导入的批量扫描 → EditorWindow + AssetDatabase.FindAssets
倒推3：「修复」常见问题自动修正 → 按规则配置批量修改 + Reimport
倒推4：「规范配置」→ ScriptableObject 定义规则（命名/压缩/LOD 参数）
倒推5：「拦截」CI/Build 前检查 → 命令行模式执行 + 失败退出码
```

#### 知识点拆解（倒推树）

```
美术资产规范工具
├── 导入拦截层（AssetPostprocessor）
│   ├── OnPreprocessTexture（贴图导入前修改设置）
│   ├── OnPostprocessTexture（贴图导入后验证）
│   ├── OnPostprocessModel（模型导入后检查/生成 LOD）
│   ├── OnPostprocessMaterial（材质检查）
│   └── AssetImporter.GetAtPath()（运行时修改导入设置）
├── 批量检查层（EditorWindow）
│   ├── AssetDatabase.FindAssets("t:Texture2D")
│   ├── 按规则遍历每个资产
│   ├── 违规报告生成（树状展示 + 导出 CSV）
│   └── 过滤器（按目录/类型/标签筛选）
├── 自动修复层
│   ├── 贴图：设置压缩格式 / maxSize / mipMap
│   ├── 模型：生成 LOD / 设置 Mesh Compression
│   ├── 材质：检查引用 / 重定向 Shader
│   ├── 命名：按规则重命名 + 更新引用
│   └── 修复后 AssetDatabase.ImportAsset() 刷新
├── 规则配置（ScriptableObject）
│   ├── TextureRule（maxSize, format, compression, sRGB）
│   ├── ModelRule（LOD levels, mesh compression, read/write）
│   ├── NamingRule（前缀/后缀/分隔符/大小写）
│   └── MaterialRule（shader 白名单，贴图槽位检查）
└── CI 集成
    ├── 命令行模式 -batchmode -executeMethod
    ├── 检查失败 → 返回非零退出码 → CI 阻断
    └── HTML/JSON 报告输出
```

#### 代码实现

**1. 规则配置（ScriptableObject）：**

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

[CreateAssetMenu(fileName = "ArtAssetRules", menuName = "TA/Art Asset Rules")]
public class ArtAssetRules : ScriptableObject
{
    [Header("贴图规则")]
    public List<TextureRule> textureRules = new List<TextureRule>();

    [Header("模型规则")]
    public List<ModelRule> modelRules = new List<ModelRule>();

    [Header("命名规则")]
    public List<NamingRule> namingRules = new List<NamingRule>();

    // 匹配规则（按目录前缀）
    public TextureRule GetTextureRule(string assetPath)
    {
        foreach (var rule in textureRules)
        {
            if (assetPath.StartsWith(rule.directoryPrefix, StringComparison.OrdinalIgnoreCase))
                return rule;
        }
        return null;
    }
}

[Serializable]
public class TextureRule
{
    public string name;
    [Tooltip("匹配的目录前缀，如 Assets/Art/Characters/")]
    public string directoryPrefix;
    public int maxSize = 2048;
    public TextureImporterFormat androidFormat = TextureImporterFormat.ASTC_6x6;
    public TextureImporterFormat iosFormat = TextureImporterFormat.ASTC_6x6;
    public bool generateMipMaps = true;
    public bool sRGB = true;
    public bool crunchedCompression = true;
}

[Serializable]
public class ModelRule
{
    public string name;
    public string directoryPrefix;
    public bool generateLOD = true;
    public int[] lodScreenRelativeHeights = new int[] { 0.5f, 0.2f, 0.01f }
        .Select(v => Mathf.RoundToInt(v * 100)).ToArray();
    public bool readable = false;
    public ModelImporterMeshCompression meshCompression = ModelImporterMeshCompression.Medium;
}

[Serializable]
public class NamingRule
{
    public string name;
    public string directoryPrefix;
    public string requiredPrefix;     // 如 "T_" for textures, "M_" for materials
    public string allowedDelimiter = "_";
    public bool requireLowerCase = false;
}
```

**2. AssetPostprocessor 拦截：**

```csharp
using UnityEditor;
using UnityEngine;

public class ArtAssetPostprocessor : AssetPostprocessor
{
    private ArtAssetRules _rules;

    ArtAssetRules GetRules()
    {
        if (_rules == null)
        {
            _rules = AssetDatabase.LoadAssetAtPath<ArtAssetRules>(
                "Assets/Settings/ArtAssetRules.asset"
            );
        }
        return _rules;
    }

    void OnPreprocessTexture()
    {
        var rules = GetRules();
        if (rules == null) return;

        var rule = rules.GetTextureRule(assetPath);
        if (rule == null) return;

        var importer = (TextureImporter)assetImporter;

        // 通用设置
        importer.maxTextureSize = rule.maxSize;
        importer.textureCompression = TextureImporterCompression.Compressed;
        importer.crunchedCompression = rule.crunchedCompression;
        importer.mipmapEnabled = rule.generateMipMaps;
        importer.sRGBTexture = rule.sRGB;

        // 平台特定
        var android = importer.GetPlatformTextureSettings("Android");
        android.overridden = true;
        android.format = rule.androidFormat;
        android.maxTextureSize = rule.maxSize;
        importer.SetPlatformTextureSettings(android);

        var ios = importer.GetPlatformTextureSettings("iPhone");
        ios.overridden = true;
        ios.format = rule.iosFormat;
        ios.maxTextureSize = rule.maxSize;
        importer.SetPlatformTextureSettings(ios);

        // 法线贴图检测（后缀 _n / _normal）
        if (assetPath.ToLower().Contains("_n.") || assetPath.ToLower().Contains("_normal."))
        {
            importer.textureType = TextureImporterType.NormalMap;
        }

        Debug.Log($"[ArtAsset] Texture preprocessed: {assetPath}", this);
    }

    void OnPostprocessModel(GameObject model)
    {
        var rules = GetRules();
        if (rules == null) return;

        foreach (var modelRule in rules.modelRules)
        {
            if (!assetPath.StartsWith(modelRule.directoryPrefix)) continue;

            var importer = (ModelImporter)assetImporter;
            importer.isReadable = modelRule.readable;
            importer.meshCompression = modelRule.meshCompression;

            // LOD 自动生成
            if (modelRule.generateLOD && importerLODCount(importer) == 0)
            {
                // Unity 2020+ 支持 LOD Generator
                Debug.Log($"[ArtAsset] Model needs LOD generation: {assetPath}");
                // 标记为需要 LOD 处理，稍后由 LOD 工具批量处理
            }
        }
    }

    int importerLODCount(ModelImporter importer)
    {
        // 简化判断：如果模型只有一个 LOD level
        return importer.lodCount > 1 ? importer.lodCount : 0;
    }
}
```

**3. 批量检查 + 修复窗口：**

```csharp
using System.Collections.Generic;
using System.Linq;
using System.Text;
using UnityEditor;
using UnityEngine;

public class ArtAssetCheckerWindow : EditorWindow
{
    private ArtAssetRules _rules;
    private Vector2 _scrollPos;
    private List<AssetViolation> _violations = new List<AssetViolation>();
    private bool _autoFixOnScan = false;

    [MenuItem("Tools/TA/Art Asset Checker")]
    static void Open() => GetWindow<ArtAssetCheckerWindow>("Art Asset Checker");

    void OnEnable()
    {
        _rules = AssetDatabase.LoadAssetAtPath<ArtAssetRules>(
            "Assets/Settings/ArtAssetRules.asset"
        );
    }

    void OnGUI()
    {
        EditorGUILayout.Space();
        _rules = (ArtAssetRules)EditorGUILayout.ObjectField(
            "Rules", _rules, typeof(ArtAssetRules), false);
        _autoFixOnScan = EditorGUILayout.Toggle("Auto Fix", _autoFixOnScan);

        EditorGUILayout.Space();
        if (GUILayout.Button("Scan All Assets", GUILayout.Height(30)))
        {
            ScanAllAssets();
        }

        if (GUILayout.Button("Fix All Violations", GUILayout.Height(30)))
        {
            FixAllViolations();
        }

        if (GUILayout.Button("Export Report (CSV)", GUILayout.Height(20)))
        {
            ExportReport();
        }

        EditorGUILayout.Space();
        EditorGUILayout.LabelField($"Violations: {_violations.Count}", EditorStyles.boldLabel);

        _scrollPos = EditorGUILayout.BeginScrollView(_scrollPos);
        foreach (var v in _violations)
        {
            DrawViolation(v);
        }
        EditorGUILayout.EndScrollView();
    }

    void DrawViolation(AssetViolation v)
    {
        EditorGUILayout.BeginHorizontal(GUI.skin.box);
        EditorGUI.DrawRect(GUILayoutUtility.GetRect(4, 30), GetSeverityColor(v.severity));

        EditorGUILayout.BeginVertical();
        EditorGUILayout.LabelField(v.assetPath, EditorStyles.boldLabel);
        EditorGUILayout.LabelField(v.message);

        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("Select", GUILayout.Width(60)))
            Selection.activeObject = AssetDatabase.LoadAssetAtPath<Object>(v.assetPath);
        if (GUILayout.Button("Fix", GUILayout.Width(60)))
        {
            FixViolation(v);
            _violations.Remove(v);
        }
        EditorGUILayout.EndHorizontal();
        EditorGUILayout.EndVertical();

        EditorGUILayout.EndHorizontal();
    }

    void ScanAllAssets()
    {
        _violations.Clear();

        // 扫描贴图
        var textureGuids = AssetDatabase.FindAssets("t:Texture2D", new[] { "Assets/Art" });
        foreach (var guid in textureGuids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            CheckTexture(path);
        }

        // 扫描模型
        var modelGuids = AssetDatabase.FindAssets("t:Model", new[] { "Assets/Art" });
        foreach (var guid in modelGuids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            CheckModel(path);
        }

        // 扫描命名
        CheckNamingConventions();

        Debug.Log($"[ArtAssetChecker] Scan complete: {_violations.Count} violations found.");
    }

    void CheckTexture(string path)
    {
        var importer = AssetImporter.GetAtPath(path) as TextureImporter;
        if (importer == null || _rules == null) return;

        var rule = _rules.GetTextureRule(path);
        if (rule == null) return;

        var android = importer.GetPlatformTextureSettings("Android");
        if (android.format != rule.androidFormat)
        {
            _violations.Add(new AssetViolation
            {
                assetPath = path,
                message = $"Android format is {android.format}, expected {rule.androidFormat}",
                severity = Severity.Error,
                type = ViolationType.TextureFormat
            });
        }

        if (importer.maxTextureSize > rule.maxSize)
        {
            _violations.Add(new AssetViolation
            {
                assetPath = path,
                message = $"Max size {importer.maxTextureSize} > limit {rule.maxSize}",
                severity = Severity.Warning,
                type = ViolationType.TextureSize
            });
        }
    }

    void CheckModel(string path)
    {
        var importer = AssetImporter.GetAtPath(path) as ModelImporter;
        if (importer == null) return;

        if (importer.isReadable)
        {
            _violations.Add(new AssetViolation
            {
                assetPath = path,
                message = "Read/Write enabled — wastes memory",
                severity = Severity.Warning,
                type = ViolationType.ModelReadable
            });
        }
    }

    void CheckNamingConventions()
    {
        if (_rules == null) return;
        var guids = AssetDatabase.FindAssets("", new[] { "Assets/Art" });

        foreach (var guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            string fileName = System.IO.Path.GetFileNameWithoutExtension(path);

            foreach (var rule in _rules.namingRules)
            {
                if (!path.StartsWith(rule.directoryPrefix)) continue;
                if (!string.IsNullOrEmpty(rule.requiredPrefix) &&
                    !fileName.StartsWith(rule.requiredPrefix))
                {
                    _violations.Add(new AssetViolation
                    {
                        assetPath = path,
                        message = $"Expected prefix '{rule.requiredPrefix}', got '{fileName}'",
                        severity = Severity.Warning,
                        type = ViolationType.Naming
                    });
                }
            }
        }
    }

    void FixAllViolations()
    {
        int fixedCount = 0;
        for (int i = _violations.Count - 1; i >= 0; i--)
        {
            if (FixViolation(_violations[i]))
            {
                _violations.RemoveAt(i);
                fixedCount++;
            }
        }
        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
        Debug.Log($"[ArtAssetChecker] Fixed {fixedCount} violations.");
    }

    bool FixViolation(AssetViolation v)
    {
        switch (v.type)
        {
            case ViolationType.TextureFormat:
            {
                var importer = AssetImporter.GetAtPath(v.assetPath) as TextureImporter;
                var rule = _rules.GetTextureRule(v.assetPath);
                if (importer != null && rule != null)
                {
                    var android = importer.GetPlatformTextureSettings("Android");
                    android.format = rule.androidFormat;
                    importer.SetPlatformTextureSettings(android);
                    importer.SaveAndReimport();
                    return true;
                }
                break;
            }
            case ViolationType.TextureSize:
            {
                var importer = AssetImporter.GetAtPath(v.assetPath) as TextureImporter;
                var rule = _rules.GetTextureRule(v.assetPath);
                if (importer != null && rule != null)
                {
                    importer.maxTextureSize = rule.maxSize;
                    importer.SaveAndReimport();
                    return true;
                }
                break;
            }
            case ViolationType.ModelReadable:
            {
                var importer = AssetImporter.GetAtPath(v.assetPath) as ModelImporter;
                if (importer != null)
                {
                    importer.isReadable = false;
                    importer.SaveAndReimport();
                    return true;
                }
                break;
            }
        }
        return false;
    }

    void ExportReport()
    {
        var sb = new StringBuilder();
        sb.AppendLine("Asset Path,Type,Severity,Message");
        foreach (var v in _violations)
        {
            sb.AppendLine($"\"{v.assetPath}\",{v.type},{v.severity},\"{v.message}\"");
        }
        string path = System.IO.Path.Combine(System.IO.Path.GetTempPath(), "art_asset_report.csv");
        System.IO.File.WriteAllText(path, sb.ToString());
        EditorUtility.RevealInFinder(path);
    }

    Color GetSeverityColor(Severity s) => s switch
    {
        Severity.Error => Color.red,
        Severity.Warning => Color.yellow,
        _ => Color.white
    };
}

// 数据结构
public enum Severity { Error, Warning, Info }
public enum ViolationType { TextureFormat, TextureSize, ModelReadable, Naming, MissingLOD }

[System.Serializable]
public class AssetViolation
{
    public string assetPath;
    public string message;
    public Severity severity;
    public ViolationType type;
}
```

**工具分层架构：**

| 层 | 职责 | 技术 | 触发时机 |
|----|------|------|----------|
| 拦截层 | 阻止不规范资产进入 | AssetPostprocessor | 导入时自动 |
| 检查层 | 批量扫描存量资产 | EditorWindow + AssetDatabase | 手动 / 定期 |
| 修复层 | 自动修正常见问题 | Importer API + Reimport | 手动 / 按需 |
| 规则层 | 定义规范 | ScriptableObject 配置 | 编辑器可视化编辑 |
| CI 层 | 阻断不合规提交 | -batchmode + 退出码 | Pre-commit / Build |

### ⚡ 实战经验

- **AssetPostprocessor 的坑**：`OnPostprocessTexture` 中修改纹理格式已经来不及（纹理已导入），要在 `OnPreprocessTexture` 中设置
- **性能优化**：`AssetDatabase.FindAssets` 搜索全项目很慢，限定搜索目录 `"Assets/Art"` 可大幅加速
- **规则版本化**：ArtAssetRules.asset 要放在 Git 里，但每个美术的本地 Unity 可能覆盖设置。建议加一个 `EditorPrefs` 存上次检查时间，做增量检查
- **多平台规则差异**：安卓用 ASTC_6x6，iOS 也用 ASTC，但 PC Standalone 可能需要 BC7。规则配置要支持 per-platform
- **CI 命令行模式**：`-batchmode -nographics -executeMethod ArtAssetCheckerWindow.BatchCheckAndExit`，在退出码中返回违规数，CI 据此阻断
- **美术沟通**：工具做好后，最重要的不是代码而是沟通。给美术开一次培训会，讲解规则和自动修复流程，比改代码有效 10 倍

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么在导入时自动修改资产 | AssetPostprocessor | 学 OnPreprocess / OnPostprocess 系列回调 |
| 检查工具太慢 | AssetDatabase API 性能 | 学 FindAssets 过滤 + 增量检查策略 |
| 修复后没生效 | Importer 保存机制 | 理解 `SaveAndReimport()` 的调用时机 |
| 规则写死在代码里，改规则要发版 | 配置化思维 | 学 ScriptableObject 做配置文件 |
| CI 集成不知道怎么搞 | Unity 命令行模式 | 学 `-batchmode -executeMethod` 工作流 |

### 🔗 相关问题

- 如何把这个工具扩展到支持 Addressables 资产的检查？
- 如果美术用的是 Substance Painter 导出的 PBR 贴图集，怎么自动检查贴图槽位对应关系？
- 如何与 Perforce / Git 做 Pre-commit hook 集成，在提交前就拦截不合规资产？
