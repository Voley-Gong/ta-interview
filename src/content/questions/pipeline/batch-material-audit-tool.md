---
title: "美术提交了 2000+ 个材质球，其中大量参数不一致——怎么写一个批量材质审计与修复工具？"
category: "pipeline"
level: 3
tags: ["Python", "材质审计", "自动化工具", "Unity Editor Scripting", "批量处理", "工程规范"]
hint: "不是手动检查 2000 个材质——写工具自动扫描、分类、修复、生成报告，让美术只做最终确认"
related: ["pipeline/maya-lod-automation", "technical-art/shader-template-system"]
---

## 参考答案

### 🎬 场景描述

面试官说：「项目进入中期，美术团队 20 多人提交了 2000+ 个材质球。你发现很多问题：有的材质用了错的 shader、有的贴图通道对应错误、有的参数超出规范范围（比如 Smoothness = 1.0 导致全反光）。写一个工具，自动扫描所有材质、检测问题、生成报告、并能批量修复常见问题。」

这道题考的是 TA 的**工程化能力**——不是只会写 shader，而是能不能用代码解决团队级的工程问题。

### ✅ 核心要点

1. **批量扫描而非逐个检查**：递归遍历 Asset 目录，用 Editor Script 或 Python（如果用 Python for Unity）批量处理
2. **规则引擎设计**：把检测规则抽象为数据驱动的配置，而非硬编码
3. **分类报告**：按严重等级（Error/Warning/Info）和问题类型分类
4. **批量修复**：安全的自动修复（只修改明确无歧义的问题）+ 需人工确认的标记
5. **可集成 CI**：工具要能命令行运行，接入 CI/CD 检查每次美术提交

### 📖 深度展开

#### 解决思路（从问题倒推工具架构）

```
最终目标：2000+ 材质100%符合规范，新增材质自动检查
                ↑
倒推1：需要定义"什么是规范"→ 规则配置文件（JSON/YAML）
倒推2：需要扫描所有材质 → AssetDatabase.FindAssets 递归遍历
倒推3：需要自动修复常见问题 → 规则定义"期望值"和"修复动作"
倒推4：需要人确认复杂问题 → 生成可视化报告 + Unity Editor 窗口
倒推5：需要集成 CI → 命令行模式 + 退出码 + 报告文件
```

#### 知识点拆解（倒推树）

```
材质批量审计工具
├── 规则系统设计
│   ├── 规则数据结构
│   │   ├── 规则ID + 描述 + 严重等级
│   │   ├── 检测条件（shader名称匹配/属性值范围/贴图通道检查）
│   │   ├── 修复动作（设置默认值/替换shader/标记待人工）
│   │   └── 例外列表（白名单）
│   ├── 常见检测规则
│   │   ├── Shader 一致性检查（是否用项目标准 Shader）
│   │   ├── 属性值范围检查（如 Smoothness 0-0.95, Metallic 0-1）
│   │   ├── 贴图格式检查（是否用了压缩、分辨率是否在规范内）
│   │   ├── 贴图通道绑定检查（如 _MetallicGlossMap 是否正确赋值）
│   │   ├── 关键字检查（如 _EMISSION 是否启用但没有 emission 贴图）
│   │   └── 引用完整性（贴图是否存在、是否有 Missing 引用）
│   └── 规则配置文件格式（JSON/YAML，可版本控制）
├── 扫描引擎
│   ├── AssetDatabase.FindAssets("t:Material")
│   ├── 按目录/标签/Layer 过滤
│   ├── 多线程友好的扫描（EditorCoroutine 避免卡死主线程）
│   └── 增量扫描（基于 git diff 或 AssetDatabase 修改时间）
├── 修复引擎
│   ├── 安全修复（Safe Fix）
│   │   ├── Shader 替换（已知映射关系 A→B）
│   │   ├── 属性值 Clamp（Smoothness 1.0 → 0.95）
│   │   └── Missing 贴图清除
│   ├── 建议修复（Suggested Fix）
│   │   ├── 推荐正确的贴图分配
│   │   └── 需要美术确认后执行
│   └── 修复日志（每次修改记录 before/after，可回滚）
├── 报告系统
│   ├── Unity Editor 窗口（开发期间交互式修复）
│   ├── HTML/CSV 报告（CI 和美术可读）
│   ├── 问题分布统计（按类型、按目录、按提交者）
│   └── 严重等级筛选和导出
├── CI/CD 集成
│   ├── 命令行入口（-batchmode -executeMethod）
│   ├── 退出码规则（0=通过, 1=警告, 2=错误）
│   ├── PR 评论机器人（自动评论材质问题）
│   └── 阻断合并规则（Error 级别问题阻断 PR）
└── 美术工作流集成
    ├── 项目模板中预装
    ├── 美术快捷键：一键检查当前选中的材质
    ├── Material Preset 系统联动
    └── 培训文档：常见问题如何避免
```

#### 代码实现

**核心：MaterialAuditor.cs — Unity Editor 工具**

```csharp
using UnityEngine;
using UnityEditor;
using System.Collections.Generic;
using System.IO;
using System.Text;
using System.Linq;

namespace TA.Pipeline
{
    /// <summary>
    /// 材质批量审计工具
    /// 用法1：菜单 TA Tools → Material Auditor 打开窗口
    /// 用法2：命令行 -executeMethod TA.Pipeline.MaterialAuditor.RunFromCommandLine
    /// </summary>
    public class MaterialAuditor : EditorWindow
    {
        // ============ 规则定义 ============
        
        [System.Serializable]
        public class MaterialRule
        {
            public string ruleId;
            public string description;
            public Severity severity;
            public CheckType checkType;
            public string shaderPattern;      // shader 名称匹配
            public string propertyName;       // 检查的属性名
            public float minValue;            // 属性值最小值
            public float maxValue;            // 属性值最大值
            public FixAction fixAction;       // 修复动作
            public float fixValue;            // 修复后的值
            
            public enum Severity { Info, Warning, Error }
            public enum CheckType 
            { 
                ShaderName,         // 检查 shader 名称
                PropertyRange,      // 检查属性值范围
                TextureMissing,     // 检查贴图是否缺失
                TextureFormat,      // 检查贴图格式
                KeywordCheck        // 检查 shader keyword
            }
            public enum FixAction 
            { 
                None,               // 不修复，仅报告
                ClampValue,         // 将值限制到范围内
                ReplaceShader,      // 替换 shader
                ClearTexture,       // 清除缺失的贴图引用
                SetKeyword          // 设置关键字
            }
        }
        
        // ============ 规则配置（实际项目用 JSON 文件加载） ============
        
        private static List<MaterialRule> DefaultRules = new List<MaterialRule>
        {
            new MaterialRule
            {
                ruleId = "R001",
                description = "Smoothness 不应为 1.0（完全光滑，导致全反射）",
                severity = MaterialRule.Severity.Warning,
                checkType = MaterialRule.CheckType.PropertyRange,
                propertyName = "_Smoothness",
                minValue = 0f, maxValue = 0.95f,
                fixAction = MaterialRule.FixAction.ClampValue,
                fixValue = 0.95f
            },
            new MaterialRule
            {
                ruleId = "R002",
                description = "Metallic 不应超过 1.0",
                severity = MaterialRule.Severity.Error,
                checkType = MaterialRule.CheckType.PropertyRange,
                propertyName = "_Metallic",
                minValue = 0f, maxValue = 1f,
                fixAction = MaterialRule.FixAction.ClampValue,
                fixValue = 1f
            },
            new MaterialRule
            {
                ruleId = "R003",
                description = "材质不应使用 Standard Shader（项目统一用 URP/Lit）",
                severity = MaterialRule.Severity.Error,
                checkType = MaterialRule.CheckType.ShaderName,
                shaderPattern = "Standard",
                fixAction = MaterialRule.FixAction.ReplaceShader
            },
            new MaterialRule
            {
                ruleId = "R004",
                description = "主贴图不应缺失",
                severity = MaterialRule.Severity.Error,
                checkType = MaterialRule.CheckType.TextureMissing,
                propertyName = "_BaseMap",
                fixAction = MaterialRule.FixAction.ClearTexture
            },
            new MaterialRule
            {
                ruleId = "R005",
                description = "Emission Color 不应为纯黑（等于没开 Emission）",
                severity = MaterialRule.Severity.Info,
                checkType = MaterialRule.CheckType.PropertyRange,
                propertyName = "_EmissionColor",
                minValue = -1f, maxValue = 999f, // 特殊：纯黑检测
                fixAction = MaterialRule.FixAction.None
            },
        };
        
        // ============ 扫描结果 ============
        
        public class MaterialIssue
        {
            public string materialPath;
            public string materialName;
            public string ruleId;
            public MaterialRule.Severity severity;
            public string description;
            public string currentValue;
            public string expectedValue;
            public bool canAutoFix;
            public bool fixed_; // 已修复
        }
        
        private List<MaterialIssue> _allIssues = new List<MaterialIssue>();
        private Vector2 _scrollPos;
        private MaterialRule.Severity _minSeverity = MaterialRule.Severity.Warning;
        private bool _showFixed = false;
        
        // ============ UI ============
        
        [MenuItem("TA Tools/Material Auditor")]
        public static void ShowWindow()
        {
            var window = GetWindow<MaterialAuditor>("Material Auditor");
            window.minSize = new Vector2(800, 500);
        }
        
        void OnGUI()
        {
            GUILayout.Label("Material Auditor", EditorStyles.boldLabel);
            
            EditorGUILayout.Space();
            
            using (new EditorGUILayout.HorizontalScope())
            {
                _minSeverity = (MaterialRule.Severity)EditorGUILayout.EnumPopup("Min Severity", _minSeverity);
                _showFixed = EditorGUILayout.Toggle("Show Fixed", _showFixed);
                
                if (GUILayout.Button("Scan All", GUILayout.Width(100)))
                {
                    ScanAllMaterials();
                }
                
                if (GUILayout.Button("Auto Fix Safe", GUILayout.Width(120)))
                {
                    AutoFixSafeIssues();
                }
                
                if (GUILayout.Button("Export Report", GUILayout.Width(100)))
                {
                    ExportReport();
                }
            }
            
            EditorGUILayout.Space();
            
            // 统计概览
            var filtered = GetFilteredIssues();
            DrawSummary(filtered);
            
            EditorGUILayout.Space();
            
            // 问题列表
            _scrollPos = EditorGUILayout.BeginScrollView(_scrollPos);
            foreach (var issue in filtered)
            {
                DrawIssueRow(issue);
            }
            EditorGUILayout.EndScrollView();
        }
        
        void DrawSummary(List<MaterialIssue> issues)
        {
            int errors = issues.Count(i => i.severity == MaterialRule.Severity.Error && !i.fixed_);
            int warnings = issues.Count(i => i.severity == MaterialRule.Severity.Warning && !i.fixed_);
            int infos = issues.Count(i => i.severity == MaterialRule.Severity.Info && !i.fixed_);
            int fixedCount = issues.Count(i => i.fixed_);
            
            EditorGUILayout.HelpBox(
                $"🔴 Errors: {errors}    🟡 Warnings: {warnings}    🔵 Info: {infos}    ✅ Fixed: {fixedCount}",
                MessageType.None
            );
        }
        
        void DrawIssueRow(MaterialIssue issue)
        {
            using (new EditorGUILayout.HorizontalScope("box"))
            {
                // 严重等级图标
                string icon = issue.severity switch
                {
                    MaterialRule.Severity.Error => "🔴",
                    MaterialRule.Severity.Warning => "🟡",
                    _ => "🔵"
                };
                
                GUILayout.Label(icon, GUILayout.Width(25));
                
                // 材质路径
                GUILayout.Label(issue.materialName, EditorStyles.boldLabel, GUILayout.Width(200));
                GUILayout.Label(issue.description, GUILayout.Width(300));
                GUILayout.Label($"当前: {issue.currentValue}", GUILayout.Width(150));
                
                if (issue.fixed_)
                {
                    GUILayout.Label("✅ Fixed", GUILayout.Width(60));
                }
                else if (issue.canAutoFix)
                {
                    if (GUILayout.Button("Fix", GUILayout.Width(50)))
                    {
                        FixIssue(issue);
                    }
                }
                
                if (GUILayout.Button("Select", GUILayout.Width(50)))
                {
                    Selection.activeObject = AssetDatabase.LoadAssetAtPath<Material>(issue.materialPath);
                }
            }
        }
        
        // ============ 扫描逻辑 ============
        
        void ScanAllMaterials()
        {
            _allIssues.Clear();
            
            string[] guids = AssetDatabase.FindAssets("t:Material", new[] { "Assets" });
            
            int scanned = 0;
            foreach (string guid in guids)
            {
                string path = AssetDatabase.GUIDToAssetPath(guid);
                Material mat = AssetDatabase.LoadAssetAtPath<Material>(path);
                
                if (mat == null) continue;
                
                CheckMaterial(mat, path);
                scanned++;
                
                if (scanned % 200 == 0)
                {
                    EditorUtility.DisplayProgressBar(
                        "Scanning Materials",
                        $"Scanned {scanned}/{guids.Length}",
                        (float)scanned / guids.Length
                    );
                }
            }
            
            EditorUtility.ClearProgressBar();
            Debug.Log($"[MaterialAuditor] Scanned {scanned} materials, found {_allIssues.Count} issues.");
        }
        
        void CheckMaterial(Material mat, string path)
        {
            foreach (var rule in DefaultRules)
            {
                switch (rule.checkType)
                {
                    case MaterialRule.CheckType.ShaderName:
                        CheckShaderName(mat, path, rule);
                        break;
                    case MaterialRule.CheckType.PropertyRange:
                        CheckPropertyRange(mat, path, rule);
                        break;
                    case MaterialRule.CheckType.TextureMissing:
                        CheckTextureMissing(mat, path, rule);
                        break;
                }
            }
        }
        
        void CheckShaderName(Material mat, string path, MaterialRule rule)
        {
            string shaderName = mat.shader ? mat.shader.name : "Missing";
            if (!string.IsNullOrEmpty(rule.shaderPattern) && shaderName.Contains(rule.shaderPattern))
            {
                _allIssues.Add(new MaterialIssue
                {
                    materialPath = path,
                    materialName = mat.name,
                    ruleId = rule.ruleId,
                    severity = rule.severity,
                    description = rule.description,
                    currentValue = shaderName,
                    expectedValue = "URP/Lit (或项目标准 Shader)",
                    canAutoFix = rule.fixAction == MaterialRule.FixAction.ReplaceShader
                });
            }
        }
        
        void CheckPropertyRange(Material mat, string path, MaterialRule rule)
        {
            if (!mat.HasProperty(rule.propertyName)) return;
            
            if (rule.propertyName == "_EmissionColor")
            {
                // 特殊检查：纯黑
                Color emi = mat.GetColor(rule.propertyName);
                if (emi.maxColorComponent < 0.01f)
                {
                    _allIssues.Add(new MaterialIssue
                    {
                        materialPath = path,
                        materialName = mat.name,
                        ruleId = rule.ruleId,
                        severity = rule.severity,
                        description = rule.description,
                        currentValue = $"({emi.r:F2}, {emi.g:F2}, {emi.b:F2})",
                        expectedValue = "非纯黑（或关闭 Emission keyword）",
                        canAutoFix = false
                    });
                }
                return;
            }
            
            float value = mat.GetFloat(rule.propertyName);
            if (value < rule.minValue || value > rule.maxValue)
            {
                _allIssues.Add(new MaterialIssue
                {
                    materialPath = path,
                    materialName = mat.name,
                    ruleId = rule.ruleId,
                    severity = rule.severity,
                    description = rule.description,
                    currentValue = value.ToString("F3"),
                    expectedValue = $"[{rule.minValue}, {rule.maxValue}]",
                    canAutoFix = rule.fixAction == MaterialRule.FixAction.ClampValue
                });
            }
        }
        
        void CheckTextureMissing(Material mat, string path, MaterialRule rule)
        {
            if (!mat.HasProperty(rule.propertyName)) return;
            
            Texture tex = mat.GetTexture(rule.propertyName);
            if (tex == null)
            {
                _allIssues.Add(new MaterialIssue
                {
                    materialPath = path,
                    materialName = mat.name,
                    ruleId = rule.ruleId,
                    severity = rule.severity,
                    description = rule.description,
                    currentValue = "NULL",
                    expectedValue = "有效贴图",
                    canAutoFix = rule.fixAction == MaterialRule.FixAction.ClearTexture
                });
            }
        }
        
        // ============ 修复逻辑 ============
        
        void AutoFixSafeIssues()
        {
            int fixedCount = 0;
            foreach (var issue in _allIssues.Where(i => i.canAutoFix && !i.fixed_))
            {
                FixIssue(issue);
                fixedCount++;
            }
            
            if (fixedCount > 0)
            {
                AssetDatabase.SaveAssets();
                Debug.Log($"[MaterialAuditor] Auto-fixed {fixedCount} issues.");
            }
        }
        
        void FixIssue(MaterialIssue issue)
        {
            Material mat = AssetDatabase.LoadAssetAtPath<Material>(issue.materialPath);
            if (mat == null) return;
            
            var rule = DefaultRules.First(r => r.ruleId == issue.ruleId);
            
            // 记录修改前的值（支持 Undo）
            Undo.RecordObject(mat, $"Fix: {rule.description}");
            
            switch (rule.fixAction)
            {
                case MaterialRule.FixAction.ClampValue:
                    float current = mat.GetFloat(rule.propertyName);
                    float clamped = Mathf.Clamp(current, rule.minValue, rule.maxValue);
                    mat.SetFloat(rule.propertyName, clamped);
                    break;
                    
                case MaterialRule.FixAction.ReplaceShader:
                    Shader newShader = Shader.Find("Universal Render Pipeline/Lit");
                    if (newShader != null) mat.shader = newShader;
                    break;
                    
                case MaterialRule.FixAction.ClearTexture:
                    mat.SetTexture(rule.propertyName, null);
                    break;
            }
            
            EditorUtility.SetDirty(mat);
            issue.fixed_ = true;
        }
        
        // ============ 报告导出 ============
        
        void ExportReport(string path = null)
        {
            path ??= $"MaterialAuditReport_{System.DateTime.Now:yyyyMMdd_HHmmss}.csv";
            
            var sb = new StringBuilder();
            sb.AppendLine("Severity,Material,Rule ID,Description,Current Value,Expected Value,Status");
            
            foreach (var issue in _allIssues)
            {
                sb.AppendLine(string.Join(",",
                    issue.severity.ToString(),
                    EscapeCsv(issue.materialName),
                    issue.ruleId,
                    EscapeCsv(issue.description),
                    EscapeCsv(issue.currentValue),
                    EscapeCsv(issue.expectedValue),
                    issue.fixed_ ? "Fixed" : "Open"
                ));
            }
            
            File.WriteAllText(path, sb.ToString());
            Debug.Log($"[MaterialAuditor] Report exported to: {path}");
            
            // 同时生成统计摘要
            GenerateSummaryReport();
        }
        
        void GenerateSummaryReport()
        {
            var bySeverity = _allIssues.GroupBy(i => i.severity)
                .Select(g => $"{g.Key}: {g.Count()}");
            var byRule = _allIssues.GroupBy(i => i.ruleId)
                .Select(g => $"{g.Key} ({g.First().description}): {g.Count()}");
            
            Debug.Log($"[MaterialAuditor] Summary:\n{string.Join("\n", bySeverity)}\n\nBy Rule:\n{string.Join("\n", byRule)}");
        }
        
        // ============ 过滤与辅助 ============
        
        List<MaterialIssue> GetFilteredIssues()
        {
            return _allIssues
                .Where(i => i.severity >= _minSeverity)
                .Where(i => _showFixed || !i.fixed_)
                .OrderByDescending(i => i.severity)
                .ToList();
        }
        
        string EscapeCsv(string input) => $"\"{input.Replace("\"", "\"\"")}\"";
        
        // ============ 命令行入口（CI 集成） ============
        
        /// <summary>
        /// CI 命令行入口
        /// unity -batchmode -projectPath . -executeMethod TA.Pipeline.MaterialAuditor.RunFromCommandLine -quit
        /// </summary>
        public static void RunFromCommandLine()
        {
            string[] args = System.Environment.GetCommandLineArgs();
            string reportPath = "MaterialAuditReport.csv";
            bool autoFix = false;
            
            for (int i = 0; i < args.Length; i++)
            {
                if (args[i] == "-reportPath" && i + 1 < args.Length)
                    reportPath = args[i + 1];
                if (args[i] == "-autoFix")
                    autoFix = true;
            }
            
            var auditor = CreateInstance<MaterialAuditor>();
            auditor.ScanAllMaterials();
            
            int errorCount = auditor._allIssues.Count(i => i.severity == MaterialRule.Severity.Error);
            
            if (autoFix)
            {
                auditor.AutoFixSafeIssues();
            }
            
            auditor.ExportReport(reportPath);
            
            // 退出码：有 Error = 2，有 Warning = 1，全通过 = 0
            int exitCode = errorCount > 0 ? 2 : 
                auditor._allIssues.Any(i => i.severity == MaterialRule.Severity.Warning) ? 1 : 0;
            
            Debug.Log($"[MaterialAuditor] CI scan complete. Exit code: {exitCode}");
            System.Environment.Exit(exitCode);
        }
    }
}
```

**规则配置文件示例（rules.json）**

```json
{
  "rules": [
    {
      "ruleId": "R001",
      "description": "Smoothness 不应为 1.0",
      "severity": "Warning",
      "checkType": "PropertyRange",
      "propertyName": "_Smoothness",
      "minValue": 0.0,
      "maxValue": 0.95,
      "fixAction": "ClampValue",
      "fixValue": 0.95
    },
    {
      "ruleId": "R003",
      "description": "不应使用 Standard Shader",
      "severity": "Error",
      "checkType": "ShaderName",
      "shaderPattern": "Standard",
      "fixAction": "ReplaceShader"
    },
    {
      "ruleId": "R006",
      "description": "法线贴图分辨率应与主贴图一致",
      "severity": "Warning",
      "checkType": "TextureFormat",
      "propertyName": "_BumpMap",
      "fixAction": "None"
    }
  ],
  "whitelist": [
    "Assets/Art/Effects/**",
    "Assets/ThirdParty/**"
  ]
}
```

#### 面试追问预演

**追问1：「2000 个材质扫描很慢，怎么优化性能？」**

> 主要策略：
> 1. **增量扫描**：用 `AssetDatabase.GetModifiedAssetPaths()` 或 git diff 只扫描本次变更的材质
> 2. **避免反复 LoadAsset**：`AssetDatabase.FindAssets` 返回 GUID 后，批量 Load，利用 Unity 内部的缓存
> 3. **EditorCoroutine 异步扫描**：每帧扫描 N 个材质，保持编辑器不卡死（而非 `for` 循环一口气跑完）
> 4. **子线程做校验逻辑**：虽然 Unity API 只能在主线程调用，但规则匹配逻辑（字符串比较、数值判断）可以在子线程做——主线程只负责加载和写入
> 5. **缓存扫描结果**：用 `ScriptableObject` 缓存上次扫描结果，只重新扫描修改过的文件

**追问2：「美术说你的工具把他们有意为之的参数（比如特殊材质就是需要 Smoothness=1）误报为错误。怎么办？」**

> 这是工具落地时的经典问题：
> 1. **白名单机制**：支持 per-material 或 per-directory 的例外规则
> 2. **美术审批流程**：误报项美术可以"标记为忽略"（Issue → Acknowledge → 记录到 ignore list）
> 3. **规则可配置而非硬编码**：所有规则从 JSON 加载，美术 lead 可以调整阈值
> 4. **分级策略**：特殊材质用自定义 Shader 或 Material Property 来标识自己（如 `_IsSpecial = 1`），扫描器跳过这些材质
> 5. **核心原则**：工具是辅助不是监工，要赢得美术信任而不是产生对立

**追问3：「这个工具怎么和 git pre-commit hook 集成？」**

> Unity 的限制是材质是 `.mat` 文件（YAML 格式），其实可以直接用 Python 脚本解析，不需要启动 Unity：
> ```python
> # pre_commit_check.py — 解析 .mat 文件检查关键属性
> import yaml, glob, sys
> 
> errors = []
> for mat_path in glob.glob("**/*.mat", recursive=True):
>     with open(mat_path) as f:
>         data = yaml.safe_load(f)
>     # 检查 m_Shader 字段
>     shader_name = data.get('MonoBehaviour', {}).get('m_Shader', {})
>     # ... 检查逻辑
> ```
> 
> 完整方案：
> - **Pre-commit**（轻量）：Python 脚本检查 `.mat` 文件的 shader 名称和关键字属性，<1s
> - **Pre-merge / CI**（完整）：Unity batchmode 运行完整 MaterialAuditor，检查所有规则
> - **定期全量**：每周跑一次完整扫描，生成趋势报告

### ⚡ 实战经验

1. **工具是给美术用的，不是给 TA 自己用的**。如果美术觉得工具难用、误报多，他们就不会用，工具形同虚设。设计 UI 时要站在美术视角：清晰的严重等级、一键修复、快速跳转到材质。

2. **规则一定要数据驱动**。我见过太多项目把检查规则硬编码在工具里，每次美术 lead 要调阈值都要找 TA 改代码。改成 JSON 配置后，美术 lead 自己改，TA 不再是瓶颈。

3. **修复一定要可回滚**。批量修复前用 `Undo.RecordObject`，或者更好的——先生成修复报告，美术确认后再批量执行。一旦批量修改了 2000 个材质发现有误，没有回滚机制就是灾难。

4. **CI 集成是终极形态**。当工具可以命令行运行后，接入 CI 的 PR 检查流程：美术提交材质 → CI 自动运行审计 → 如果有 Error 级别问题，PR 被阻断并自动评论。这比任何人工审核都高效。

5. **趋势比快照更有价值**。除了"当前有多少问题"，更要看"问题数量随时间的变化趋势"。如果每周问题数在下降，说明工具在发挥作用；如果持平，说明有美术不看报告。

### 🎯 能力体检清单

- [ ] **我能不能写 Unity Editor Script？** → 不能说明 Unity 工具开发能力不足
- [ ] **我有没有设计过数据驱动的规则系统？** → 没有说明工程抽象能力需加强
- [ ] **我能不能让工具命令行运行并集成 CI？** → 不能说明 DevOps 意识不足
- [ ] **我有没有考虑美术的使用体验？** → 没有说明只站在技术视角，缺少产品思维
- [ ] **我能不能用 Python 或其他脚本解析 Unity 资源？** → 不能说明跨工具链能力有限
- [ ] **面对 2000+ 资产，我有没有增量扫描的意识？** → 没有说明性能思维有盲区

### 🔗 相关问题

- [Maya 脚本自动化 LOD：怎么让美术不用手动生成？](../pipeline/maya-lod-automation.md)
- [Shader 模板系统：如何让美术自助调参而不炸引擎？](../technical-art/shader-template-system.md)
- [LOD 规范制定与质量验收：怎么管 20 个美术的模型质量？](../technical-art/lod-spec-and-qa.md)
