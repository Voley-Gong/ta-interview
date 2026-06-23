---
title: "入职第一天：美术资产混乱怎么治？写一个一键检查工具"
category: "pipeline"
level: 2
tags: ["Unity编辑器扩展", "AssetPostprocessor", "美术规范", "自动化工具", "TA工具链"]
hint: "核心是 EditorWindow + AssetPostprocessor 双管齐下——Postprocessor 在导入时自动检查，EditorWindow 做批量扫描和报告"
related: ["technical-art/mobile-texture-compression", "pipeline/batch-material-audit-tool", "technical-art/lod-spec-and-qa"]
---

## 参考答案

### 🎬 场景描述

面试官说：「你入职后发现项目的美术资产非常混乱：贴图有没有压缩格式不一致的、模型面数超标没人管、Shader 引用丢失、材质命名毫无规范。美术总监要求你写一个工具，能一键扫描整个项目并生成报告，同时在新资产导入时自动检查。你怎么设计这个工具？」

（网易、字节、米哈游 TA 面试的高频题——考察 Unity 编辑器扩展能力和工程化思维）

### ✅ 核心要点

1. **双模式检查**：导入时检查（AssetPostprocessor）+ 手动批量检查（EditorWindow）
2. **规则模块化**：每条检查规则是独立模块（面向接口），方便扩展
3. **检查范围**：贴图（尺寸/压缩/通道）、模型（面数/LOD/UV）、材质（Shader/属性/引用）、动画（帧数/压缩）
4. **报告输出**：可视化窗口 + 导出 CSV/HTML + 一键定位到问题资产
5. **阻断 vs 警告**：严重问题阻断导入（报错），轻微问题警告（黄色标记）

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：一键扫描 → 发现所有不合规资产 → 分类报告 → 可定位修复
                ↑
倒推1：需要「知道什么是不合规的」→ 规则定义系统（Rule Interface）
倒推2：需要「扫描整个项目的资产」→ AssetDatabase.FindAssets + 遍历
倒推3：需要「每条规则对每个资产做判断」→ 批量检查器 + 进度条
倒推4：需要「可视化展示结果」→ EditorWindow + ListView + 分类着色
倒推5：需要「新资产导入时也检查」→ AssetPostprocessor.OnPostprocess 钩子
倒推6：需要「可配置」→ ScriptableObject 存储规则配置（阈值等）
```

#### 知识点拆解（倒推树）

```
美术资产检查工具
├── 规则系统（核心设计）
│   ├── IAssetCheckRule 接口（Check() → CheckResult）
│   ├── 规则实现：
│   │   ├── TextureSizeRule（最大尺寸限制，如 2048）
│   │   ├── TextureCompressionRule（平台压缩格式检查）
│   │   ├── TextureChannelRule（多余通道检测）
│   │   ├── MeshTriangleRule（面数上限，按 LOD 级别）
│   │   ├── MeshUVRule（UV 重叠/越界检测）
│   │   ├── MaterialShaderRule（Shader 白名单检查）
│   │   ├── MaterialMissingRefRule（引用丢失检测）
│   │   ├── AnimationClipRule（帧率/压缩检查）
│   │   └── NamingConventionRule（命名规范）
│   └── 规则配置：ScriptableObject (CheckRuleConfig)
├── 导入时检查（AssetPostprocessor）
│   ├── OnPostprocessTexture → Texture 规则
│   ├── OnPostprocessModel → Mesh 规则
│   ├── OnPostprocessMaterial → Material 规则
│   ├── 阻断模式：LogError（阻止导入）
│   └── 警告模式：LogWarning（允许导入但标记）
├── 批量检查（EditorWindow）
│   ├── 扫描范围选择（全部/指定文件夹/指定类型）
│   ├── 异步扫描 + 进度条（EditorUtility.DisplayProgressBar）
│   ├── 结果分类：Error / Warning / Info
│   ├── 点击跳转：Selection.activeObject = asset
│   └── 批量修复（部分规则支持自动修复，如压缩格式）
├── 报告导出
│   ├── CSV 导出（资产路径 + 违反规则 + 严重级别）
│   ├── HTML 报告（带样式和筛选）
│   └── 历史报告对比（追踪修复进度）
└── 性能优化
    ├── AssetDatabase 批量加载 vs 逐个加载
    ├── EditorCoroutine 异步扫描（不卡死编辑器）
    └── 缓存机制（hash → 上次检查结果）
```

#### 代码实现

**规则接口定义：**

```csharp
using UnityEditor;
using UnityEngine;

public enum CheckSeverity { Info, Warning, Error }

public struct CheckResult
{
    public bool passed;
    public CheckSeverity severity;
    public string message;
    public Object asset;

    public static CheckResult Pass() => new CheckResult { passed = true };
    public static CheckResult Fail(CheckSeverity severity, string msg, Object asset = null) =>
        new CheckResult { passed = false, severity = severity, message = msg, asset = asset };
}

/// <summary>
/// 所有检查规则的统一接口
/// </summary>
public interface IAssetCheckRule
{
    string RuleName { get; }
    string TargetType { get; } // "Texture", "Mesh", "Material", "Animation"
    CheckResult Check(Object asset, CheckRuleConfig config);
    bool CanAutoFix { get; }
    void AutoFix(Object asset, CheckRuleConfig config);
}
```

**规则配置（ScriptableObject）：**

```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "AssetCheckConfig", menuName = "TA/Asset Check Config")]
public class CheckRuleConfig : ScriptableObject
{
    [Header("贴图规则")]
    public int maxTextureSize = 2048;
    public bool requireCompression = true;
    public bool checkRedundantAlpha = true;
    public string[] allowedTextureFormats = { "ASTC", "ETC2" };

    [Header("模型规则")]
    public int maxTrianglesLOD0 = 50000;
    public int maxTrianglesLOD1 = 20000;
    public int maxTrianglesLOD2 = 8000;
    public bool requireUV2 = true;

    [Header("材质规则")]
    public string[] shaderWhitelist = {
        "Universal Render Pipeline/Lit",
        "Universal Render Pipeline/Simple Lit",
        "Custom/CharacterStandard"
    };
    public bool checkMissingReferences = true;

    [Header("命名规范")]
    public string textureNamePattern = @"^T_[A-Z]\w+_[A-Z]{2,3}$";
    public string materialNamePattern = @"^M_[A-Z]\w+$";
    public string meshNamePattern = @"^SM_[A-Z]\w+_LOD\d$";

    [Header("导入阻断")]
    public bool blockImportOnError = true;
}
```

**贴图检查规则示例：**

```csharp
using UnityEditor;
using UnityEngine;

public class TextureSizeRule : IAssetCheckRule
{
    public string RuleName => "贴图尺寸检查";
    public string TargetType => "Texture";
    public bool CanAutoFix => false;

    public CheckResult Check(Object asset, CheckRuleConfig config)
    {
        var texture = asset as Texture2D;
        if (texture == null) return CheckResult.Pass();

        var importer = AssetImporter.GetAtPath(AssetDatabase.GetAssetPath(texture)) as TextureImporter;
        if (importer == null) return CheckResult.Pass();

        int maxSize = importer.maxTextureSize;
        if (maxSize > config.maxTextureSize)
        {
            return CheckResult.Fail(CheckSeverity.Error,
                $"贴图最大尺寸 {maxSize} 超过限制 {config.maxTextureSize}（路径: {importer.assetPath}）",
                texture);
        }

        return CheckResult.Pass();
    }

    public void AutoFix(Object asset, CheckRuleConfig config) { /* 不支持自动修复 */ }
}

public class TextureCompressionRule : IAssetCheckRule
{
    public string RuleName => "贴图压缩检查";
    public string TargetType => "Texture";
    public bool CanAutoFix => true;

    public CheckResult Check(Object asset, CheckRuleConfig config)
    {
        var texture = asset as Texture2D;
        if (texture == null) return CheckResult.Pass();

        string path = AssetDatabase.GetAssetPath(texture);
        var importer = AssetImporter.GetAtPath(path) as TextureImporter;

        // 检查 Android 平台压缩格式
        var androidSettings = importer.GetPlatformTextureSettings("Android");
        if (config.requireCompression && androidSettings.overridden)
        {
            string format = androidSettings.format.ToString();
            bool valid = false;
            foreach (var allowed in config.allowedTextureFormats)
            {
                if (format.Contains(allowed)) { valid = true; break; }
            }
            if (!valid)
            {
                return CheckResult.Fail(CheckSeverity.Warning,
                    $"Android 贴图格式 {format} 不在允许列表 [{string.Join(", ", config.allowedTextureFormats)}]",
                    texture);
            }
        }

        return CheckResult.Pass();
    }

    public void AutoFix(Object asset, CheckRuleConfig config)
    {
        var texture = asset as Texture2D;
        string path = AssetDatabase.GetAssetPath(texture);
        var importer = AssetImporter.GetAtPath(path) as TextureImporter;

        var settings = importer.GetPlatformTextureSettings("Android");
        settings.overridden = true;
        settings.format = TextureImporterFormat.ASTC_6x6;
        importer.SetPlatformTextureSettings(settings);
        importer.SaveAndReimport();
    }
}
```

**AssetPostprocessor（导入时自动检查）：**

```csharp
using UnityEditor;
using UnityEngine;

public class AssetImportChecker : AssetPostprocessor
{
    private static CheckRuleConfig _config;

    private static CheckRuleConfig Config
    {
        get
        {
            if (_config == null)
            {
                var guids = AssetDatabase.FindAssets("t:CheckRuleConfig");
                if (guids.Length > 0)
                    _config = AssetDatabase.LoadAssetAtPath<CheckRuleConfig>(
                        AssetDatabase.GUIDToAssetPath(guids[0]));
            }
            return _config;
        }
    }

    void OnPostprocessTexture(Texture2D texture)
    {
        if (Config == null) return;

        var rules = new IAssetCheckRule[] {
            new TextureSizeRule(),
            new TextureCompressionRule(),
            new TextureChannelRule()
        };

        foreach (var rule in rules)
        {
            var result = rule.Check(texture, Config);
            if (!result.passed)
            {
                if (result.severity == CheckSeverity.Error && Config.blockImportOnError)
                {
                    LogError($"[AssetCheck] {rule.RuleName}: {result.message}");
                }
                else
                {
                    LogWarning($"[AssetCheck] {rule.RuleName}: {result.message}");
                }
            }
        }
    }

    void OnPostprocessModel(GameObject model)
    {
        if (Config == null) return;

        var rules = new IAssetCheckRule[] {
            new MeshTriangleRule(),
            new MeshUVRule()
        };

        string path = assetPath;
        var importer = AssetImporter.GetAtPath(path) as ModelImporter;

        foreach (var rule in rules)
        {
            // 注意：OnPostprocessModel 时 Mesh 数据已可用
            var meshes = model.GetComponentsInChildren<MeshFilter>();
            foreach (var mf in meshes)
            {
                if (mf.sharedMesh != null)
                {
                    var result = rule.Check(mf.sharedMesh, Config);
                    if (!result.passed)
                    {
                        if (result.severity == CheckSeverity.Error)
                            LogError($"[AssetCheck] {rule.RuleName}: {result.message}");
                        else
                            LogWarning($"[AssetCheck] {rule.RuleName}: {result.message}");
                    }
                }
            }
        }
    }
}
```

**EditorWindow（批量扫描与报告）：**

```csharp
using UnityEditor;
using UnityEngine;
using System.Collections.Generic;
using System.IO;
using System.Text;

public class AssetCheckerWindow : EditorWindow
{
    private CheckRuleConfig _config;
    private List<CheckResult> _results = new List<CheckResult>();
    private Vector2 _scrollPos;
    private CheckSeverity _minSeverity = CheckSeverity.Warning;

    [MenuItem("TA/美术资产检查工具")]
    static void Open() => GetWindow<AssetCheckerWindow>("Asset Checker");

    void OnEnable()
    {
        var guids = AssetDatabase.FindAssets("t:CheckRuleConfig");
        if (guids.Length > 0)
            _config = AssetDatabase.LoadAssetAtPath<CheckRuleConfig>(AssetDatabase.GUIDToAssetPath(guids[0]));
    }

    void OnGUI()
    {
        EditorGUILayout.Space();
        _config = (CheckRuleConfig)EditorGUILayout.ObjectField("规则配置", _config, typeof(CheckRuleConfig), false);
        _minSeverity = (CheckSeverity)EditorGUILayout.EnumPopup("最低显示级别", _minSeverity);

        EditorGUILayout.Space();

        using (new EditorGUILayout.HorizontalScope())
        {
            if (GUILayout.Button("扫描全部资产", GUILayout.Height(30)))
                ScanAll();
            if (GUILayout.Button("扫描选中文件夹", GUILayout.Height(30)))
                ScanSelected();
            if (GUILayout.Button("导出报告 (CSV)", GUILayout.Height(30)))
                ExportCSV();
            if (GUILayout.Button("清空", GUILayout.Height(30)))
                _results.Clear();
        }

        EditorGUILayout.Space();

        // 统计信息
        int errors = _results.FindAll(r => r.severity == CheckSeverity.Error).Count;
        int warnings = _results.FindAll(r => r.severity == CheckSeverity.Warning).Count;
        EditorGUILayout.LabelField($"结果: <color=red>{errors} 错误</color>  <color=yellow>{warnings} 警告</color>", new GUIStyle("richText"));

        EditorGUILayout.Space();

        // 结果列表
        _scrollPos = EditorGUILayout.BeginScrollView(_scrollPos);
        foreach (var result in _results)
        {
            if ((int)result.severity < (int)_minSeverity) continue;

            Color oldColor = GUI.color;
            GUI.color = result.severity == CheckSeverity.Error ? new Color(1f, 0.6f, 0.6f) : new Color(1f, 0.9f, 0.5f);

            using (new EditorGUILayout.HorizontalScope("box"))
            {
                EditorGUILayout.LabelField(result.severity.ToString(), GUILayout.Width(60));
                EditorGUILayout.LabelField(result.message, GUILayout.ExpandWidth(true));

                if (result.asset != null && GUILayout.Button("定位", GUILayout.Width(40)))
                {
                    EditorGUIUtility.PingObject(result.asset);
                    Selection.activeObject = result.asset;
                }
            }
            GUI.color = oldColor;
        }
        EditorGUILayout.EndScrollView();
    }

    void ScanAll()
    {
        _results.Clear();
        var guids = AssetDatabase.FindAssets("t:Texture t:Model t:Material", new[] { "Assets" });
        RunScan(guids);
    }

    void ScanSelected()
    {
        string[] folders = new string[Selection.objects.Length];
        for (int i = 0; i < Selection.objects.Length; i++)
            folders[i] = AssetDatabase.GetAssetPath(Selection.objects[i]);

        _results.Clear();
        var guids = AssetDatabase.FindAssets("t:Texture t:Model t:Material", folders);
        RunScan(guids);
    }

    void RunScan(string[] guids)
    {
        var rules = new List<IAssetCheckRule> {
            new TextureSizeRule(),
            new TextureCompressionRule(),
            new MeshTriangleRule(),
            new MeshUVRule(),
            new MaterialShaderRule(),
            new MaterialMissingRefRule(),
            new NamingConventionRule()
        };

        for (int i = 0; i < guids.Length; i++)
        {
            string path = AssetDatabase.GUIDToAssetPath(guids[i]);
            Object asset = AssetDatabase.LoadMainAssetAtPath(path);

            if (EditorUtility.DisplayCancelableProgressBar("扫描中...", path, (float)i / guids.Length))
                break;

            foreach (var rule in rules)
            {
                var result = rule.Check(asset, _config);
                if (!result.passed)
                    _results.Add(result);
            }
        }

        EditorUtility.ClearProgressBar();
        Debug.Log($"[AssetChecker] 扫描完成，发现 {_results.Count} 个问题");
    }

    void ExportCSV()
    {
        string path = EditorUtility.SaveFilePanel("导出报告", "", "asset_report", "csv");
        if (string.IsNullOrEmpty(path)) return;

        var sb = new StringBuilder();
        sb.AppendLine("Severity,Rule,Asset Path,Message");
        foreach (var r in _results)
        {
            string assetPath = r.asset != null ? AssetDatabase.GetAssetPath(r.asset) : "N/A";
            sb.AppendLine($"{r.severity},\"{r.message}\",\"{assetPath}\",\"{r.message}\"");
        }
        File.WriteAllText(path, sb.ToString());
        EditorUtility.RevealInFinder(path);
    }
}
```

**对比表：检查策略**

| 策略 | 时机 | 优点 | 缺点 | 推荐度 |
|------|------|------|------|--------|
| AssetPostprocessor | 资产导入时 | 第一时间拦截，防止不合规资产入库 | 无法覆盖已存在资产 | ★★★★ |
| EditorWindow 手动扫描 | 随时 | 全量扫描，生成报告 | 依赖人工触发 | ★★★★ |
| Pre-Commit Hook | Git 提交前 | 源头阻断 | 需要额外 CI 配置 | ★★★ |
| CI/CD 定时检查 | 每日/每次合并 | 趋势追踪，全量覆盖 | 反馈延迟 | ★★★★ |

### ⚡ 实战经验

- **规则配置 ScriptableObject 是灵魂**：不同项目规则不同（写实 vs 卡通，PC vs 移动），配置必须可编辑、可版本管理
- **阻断 vs 警告策略**：严重问题（如 Shader 引用丢失）阻断导入，风格问题（如命名规范）只警告。过于严格会导致美术绕过检查
- **进度条必须用 Cancelable 版本**：`DisplayCancelableProgressBar`，否则扫描大项目（数万资产）时编辑器直接卡死
- **AssetPostprocessor 性能注意**：OnPostprocess 是同步的，规则不要太重。复杂的全量检查留给手动扫描
- **PingObject 比选中更好用**：报告中点击「定位」用 `EditorGUIUtility.PingObject` 高亮但不改变选中状态，美术可以连续查看多个
- **CI 集成**：核心扫描逻辑放在非 Editor 依赖的 DLL 中，配合 Unity 的 `-batchmode -executeMethod` 在 CI 上跑无头检查
- **修复优先级排序**：Error 级别按「影响面」排序（Shader 错误影响所有用该材质的物体 > 单张贴图尺寸超标）

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么在导入时拦截 | AssetPostprocessor 机制 | 学习 Unity 导入管线生命周期 |
| 规则写死了，换个项目不能用 | 面向接口设计 | 复习 C# 接口/策略模式 |
| 扫描大项目卡死编辑器 | 异步编程 / EditorCoroutine | 学习 Unity Editor 异步模式 |
| 不知道怎么读取贴图压缩设置 | TextureImporter API | 学习 AssetImporter 子类 |
| 面试官追问：怎么让美术无法绕过检查 | CI/CD 集成 | 学习 Unity batchmode + Jenkins/GitHub Actions |
| 报告没人看 | 可视化和追踪 | 学习 Jira/飞书集成，自动提工单 |

### 🔗 相关问题

- 如何把这个检查工具集成到 CI/CD 流程中？（提示：Unity batchmode + Python 脚本解析日志）
- 贴图的冗余通道怎么自动检测？（提示：读取像素数据，分析 RGB 通道是否都为 1 或 0）
- 如何做 Shader 的白名单管理？新增 Shader 上线流程怎么走？
- 如果项目有 10 万个资产，扫描太慢怎么办？（提示：增量扫描 + hash 缓存 + 并行）
