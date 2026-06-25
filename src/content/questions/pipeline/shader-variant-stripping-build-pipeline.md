---
title: "包体暴涨到 2GB：Shader Variant 爆炸后如何用构建管线自动裁剪？"
category: "pipeline"
level: 3
tags: ["Shader变体", "构建管线", "包体优化", "Shader Stripping", "自动化工具"]
hint: "Shader 变体爆炸不是 Shader 问题——是管线问题。关键是用 IPreprocessShaders 接口在构建阶段自动裁剪无用变体"
related: ["optimization/shader-variant-explosion", "pipeline/shader-library-asset-bundle-hotfix", "optimization/mobile-package-size-2gb-to-500mb"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的 Unity 项目从上个版本开始包体从 800MB 暴涨到 2GB，排查后发现是 Shader Variant 数量从 2000 飙到 18000+。美术加了很多 Shader Keyword 组合，大部分变体在游戏中根本用不到。你作为 TA，设计一套自动化的 Shader 变体裁剪管线，在构建阶段自动剔除无用变体。」

（这个问题是叠纸、米哈游等以角色渲染为核心的公司的高频面试题——Shader 变体管理是 TA Pipeline 的核心能力之一。）

### ✅ 核心要点

1. **变体爆炸的根因**：`multi_compile` / `shader_feature` 的笛卡尔积——5 个 keyword 各 2 个状态 = 32 个变体，多个 Pass 叠加后指数增长
2. **收集实际使用变体**：用 ShaderVariantCollection 或运行时记录实际编译的变体
3. **构建时裁剪**：实现 `IPreprocessShaders` 接口，在 Build 阶段拦截并丢弃无用变体
4. **CI/CD 集成**：构建管线自动执行变体收集→裁剪→验证→打包
5. **防范机制**：Shader 提交规范 + 变体数量监控报警

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：Shader 变体从 18000+ 降到实际需要的 ~3000，包体回归正常
                    ↑
Step 1：变体来源分析
  ├── 哪些 Shader 贡献了最多变体？
  ├── multi_compile vs shader_feature 各占多少？
  └── 有多少变体在游戏中实际被触发？
     ↓
Step 2：实际使用变体收集
  ├── 方案 A：ShaderVariantCollection（手动维护，易遗漏）
  ├── 方案 B：运行时记录（PlayerConnection 回调，自动化）
  └── 方案 C：静态分析（扫描材质和场景引用）
     ↓
Step 3：构建时裁剪
  ├── 实现 IPreprocessShaders
  ├── 对比「实际使用」白名单，丢弃不在名单中的变体
  └── 输出裁剪日志（哪些被裁了、哪些被保留）
     ↓
Step 4：验证 & 持续监控
  ├── 构建后检查 Shader Variant 数量
  ├── CI 流水线变体数量 diff 报警
  └── 美术提交 Shader 时自动检查 keyword 必要性
```

#### 知识点拆解（倒推树）

```
Shader 变体裁剪管线
├── 变体爆炸原理
│   ├── multi_compile：全局 keyword，所有材质都编译所有变体
│   ├── shader_feature：仅材质使用的 keyword 组合才编译
│   ├── 变体数 = Π(每个keyword的状态数) × Pass 数
│   └── 常见爆炸源：fog / lightmap / shadow / LOD cross product
│
├── 变体收集策略
│   ├── 运行时收集（最准）
│   │   ├── Application.isEditor + Shader.keywordSpace (Unity 2021+)
│   │   ├── PlayerConnection 发送编译记录到 Editor
│   │   └── 覆盖测试：跑遍所有场景 + 切换所有画质档位
│   ├── 静态分析（补充）
│   │   ├── 扫描所有 Material 的 EnabledKeywords
│   │   ├── 扫描 ShaderVariantCollection
│   │   └── 扫描场景中的材质引用
│   └── 组合验证：运行时 ∪ 静态分析 = 安全白名单
│
├── IPreprocessShaders 裁剪实现
│   ├── 接口回调：OnProcessShader(shader, snippet, variant)
│   ├── 判定逻辑：variant.shaderKeywordSet ⊂ 白名单？
│   ├── 多 Pass 处理：每个 Pass 的变体独立裁剪
│   └── 日志输出：记录裁剪/保留统计
│
├── 构建管线集成
│   ├── Pre-build：更新白名单 → 序列化到 JSON
│   ├── Build：Unity 自动调用 IPreprocessShaders
│   ├── Post-build：验证变体数 → 生成报告
│   └── CI/CD：Jenkins/GitHub Actions 集成 + 报警
│
└── 预防机制
    ├── Shader 提交 Review：新增 keyword 必须说明理由
    ├── 变体预算：每个 Shader 的变体上限
    ├── 自动检测：pre-commit hook 检查变体增量
    └── 教育：美术理解 multi_compile 的代价
```

#### 代码实现

**核心裁剪器：IPreprocessShaders 实现**

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using UnityEditor.Build;
using UnityEditor.Build.Reporting;
using System.Collections.Generic;
using System.IO;
using System.Linq;

/// <summary>
/// 构建时自动裁剪未使用的 Shader 变体。
/// 白名单来源：运行时收集 + 静态分析（JSON 文件）。
/// </summary>
public class ShaderVariantStripper : IPreprocessShaders, IPreprocessBuild, IPostprocessBuild
{
    // IPreprocessShaders 回调优先级（越小越先执行）
    public int callbackOrder => 0;

    // 白名单：Shader名 -> 该Shader允许的变体集合
    private Dictionary<string, HashSet<string>> _variantWhitelist;
    
    // 裁剪统计
    private int _totalVariants = 0;
    private int _strippedVariants = 0;
    private Dictionary<string, (int total, int stripped)> _perShaderStats = new();

    public void OnPreprocessBuild(BuildReport report)
    {
        // 构建前加载白名单
        LoadVariantWhitelist();
        Debug.Log($"[ShaderStripper] Whitelist loaded: {_variantWhitelist.Count} shaders");
    }

    public void OnProcessShader(
        Shader shader, 
        ShaderSnippetData snippet, 
        IList<ShaderCompilerData> variantData)
    {
        string shaderName = shader.name;
        int originalCount = variantData.Count;
        _totalVariants += originalCount;

        if (!_variantWhitelist.ContainsKey(shaderName))
        {
            // 不在白名单中的 Shader → 保留全部（安全策略）
            // 如果确认某个 Shader 完全不用，可以在这里清空 variantData
            return;
        }

        var allowedKeywords = _variantWhitelist[shaderName];
        var toStrip = new List<int>();

        for (int i = variantData.Count - 1; i >= 0; i--)
        {
            var variant = variantData[i];
            var keywordSet = variant.shaderKeywordSet;
            
            // 获取此变体的所有 keyword
            var keywords = keywordSet.GetShaderKeywords();
            var keywordStrs = new List<string>();
            foreach (var kw in keywords)
            {
                keywordStrs.Add(kw.GetKeywordName());
            }
            
            // 构建变体签名
            string signature = string.Join("+", keywordStrs.OrderBy(s => s));
            
            // 如果不在白名单中 → 标记裁剪
            if (!allowedKeywords.Contains(signature))
            {
                toStrip.Add(i);
            }
        }

        // 执行裁剪（从后往前删除，避免索引错位）
        foreach (int idx in toStrip)
        {
            variantData.RemoveAt(idx);
        }

        _strippedVariants += toStrip.Count;

        // 记录统计
        if (!_perShaderStats.ContainsKey(shaderName))
            _perShaderStats[shaderName] = (0, 0);
        var stats = _perShaderStats[shaderName];
        _perShaderStats[shaderName] = (stats.Item1 + originalCount, stats.Item2 + toStrip.Count);

        if (toStrip.Count > 0)
        {
            Debug.Log($"[ShaderStripper] {shaderName}: stripped {toStrip.Count}/{originalCount} variants " +
                      $"(pass: {snippet.passType})");
        }
    }

    public void OnPostprocessBuild(BuildReport report)
    {
        // 构建完成后输出报告
        string reportPath = $"BuildReports/shader_stripping_{System.DateTime.Now:yyyyMMdd_HHmmss}.txt";
        Directory.CreateDirectory("BuildReports");

        using var writer = new StreamWriter(reportPath);
        writer.WriteLine("=== Shader Variant Stripping Report ===");
        writer.WriteLine($"Total variants: {_totalVariants}");
        writer.WriteLine($"Stripped: {_strippedVariants} ({(float)_strippedVariants / _totalVariants * 100:F1}%)");
        writer.WriteLine($"Kept: {_totalVariants - _strippedVariants}");
        writer.WriteLine();
        writer.WriteLine("=== Per-Shader Breakdown ===");
        
        foreach (var kv in _perShaderStats.OrderByDescending(x => x.Value.stripped))
        {
            var (total, stripped) = kv.Value;
            writer.WriteLine($"  {kv.Key}: {stripped}/{total} stripped");
        }

        Debug.Log($"[ShaderStripper] Report saved to {reportPath}");
        Debug.Log($"[ShaderStripper] Total: {_totalVariants}, Stripped: {_strippedVariants} " +
                  $"({(float)_strippedVariants / _totalVariants * 100:F1}%)");
    }

    private void LoadVariantWhitelist()
    {
        string path = "Assets/BuildConfig/shader_variant_whitelist.json";
        if (!File.Exists(path))
        {
            Debug.LogWarning("[ShaderStripper] Whitelist not found! No stripping will occur.");
            _variantWhitelist = new Dictionary<string, HashSet<string>>();
            return;
        }

        string json = File.ReadAllText(path);
        var data = JsonUtility.FromJson<VariantWhitelistData>(json);
        _variantWhitelist = new Dictionary<string, HashSet<string>>();

        foreach (var entry in data.entries)
        {
            _variantWhitelist[entry.shaderName] = new HashSet<string>(entry.variants);
        }
    }

    [System.Serializable]
    public class VariantWhitelistData
    {
        public List<WhitelistEntry> entries;
    }

    [System.Serializable]
    public class WhitelistEntry
    {
        public string shaderName;
        public List<string> variants;
    }
}
#endif
```

**运行时变体收集器（挂在测试场景中跑一遍）：**

```csharp
using UnityEngine;
using System.Collections.Generic;
using System.IO;
using System.Linq;

/// <summary>
/// 运行时收集实际编译的 Shader 变体。
/// 在测试场景中挂载此脚本，跑遍所有场景/角色后导出白名单。
/// </summary>
public class RuntimeVariantCollector : MonoBehaviour
{
    private Dictionary<string, HashSet<string>> _collected = new();
    
    void Update()
    {
        // Unity 2021.2+ 支持 Shader.keywordSpace
        // 遍历所有已加载的 Shader，记录实际使用的 keyword 组合
        foreach (var mat in Resources.FindObjectsOfTypeAll<Material>())
        {
            if (mat.shader == null) continue;
            
            string shaderName = mat.shader.name;
            if (!_collected.ContainsKey(shaderName))
                _collected[shaderName] = new HashSet<string>();
            
            // 获取材质启用的 keyword
            var keywords = mat.shaderKeywords;
            if (keywords.Length == 0)
            {
                _collected[shaderName].Add(""); // 无 keyword 的基础变体
            }
            else
            {
                var sorted = keywords.OrderBy(k => k).ToArray();
                _collected[shaderName].Add(string.Join("+", sorted));
            }
        }
    }

    void OnApplicationQuit()
    {
        ExportWhitelist();
    }

    /// <summary>
    /// 在 Editor 模式下也可以手动调用导出
    /// </summary>
    [ContextMenu("Export Whitelist")]
    public void ExportWhitelist()
    {
        var entries = new List<WhitelistEntry>();
        foreach (var kv in _collected)
        {
            entries.Add(new WhitelistEntry
            {
                shaderName = kv.Key,
                variants = kv.Value.ToList()
            });
        }

        var data = new VariantWhitelistData { entries = entries };
        string json = JsonUtility.ToJson(data, true);

        string outputPath = "Assets/BuildConfig/shader_variant_whitelist.json";
        Directory.CreateDirectory(Path.GetDirectoryName(outputPath)!);
        File.WriteAllText(outputPath, json);

        int totalVariants = _collected.Values.Sum(s => s.Count);
        Debug.Log($"[VariantCollector] Exported {totalVariants} variants across {_collected.Count} shaders to {outputPath}");
    }

    [System.Serializable]
    public class WhitelistEntry
    {
        public string shaderName;
        public List<string> variants;
    }

    [System.Serializable]
    public class VariantWhitelistData
    {
        public List<WhitelistEntry> entries;
    }
}
```

**CI 集成：变体数量 Diff 报告（GitHub Actions 示例）：**

```yaml
# .github/workflows/shader-variant-check.yml
name: Shader Variant Check
on: [pull_request]

jobs:
  variant-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 需要完整历史来做 diff
      
      - name: Run Variant Analysis
        run: |
          # 调用 Unity BatchMode 执行变体分析
          # 输出 JSON 格式的变体统计
          unity -batchmode -projectPath . -executeMethod VariantAnalysisTool.ExportCurrentVariantCount -quit
        
      - name: Compare with main branch
        run: |
          # 对比 main 分支的变体数
          python3 scripts/variant_diff.py \
            --before BuildReports/variants_main.json \
            --after BuildReports/variants_pr.json \
            --threshold 50  # 新增超过 50 个变体就报警
```

### ⚡ 实战经验

1. **不要一次性全裁**：第一次启用裁剪时，先用「只记录不裁剪」模式跑一周，确认白名单完整后再开启实际裁剪
2. **Fog / Lightmap / Shadow 是隐形杀手**：这些是 Unity 内置的 `multi_compile`，一个 `#pragma multi_compile_fog` 就翻倍变体。在 Project Settings > Graphics 中关闭不需要的内置 variant
3. **shader_feature 替代 multi_compile**：如果一个 keyword 只在少数材质中使用，用 `shader_feature`（按需编译）而非 `multi_compile`（全量编译）
4. **AssetBundle 与变体的关系**：如果 Shader 单独打 AB，但变体信息分散在其他 AB 的材质中，加载时可能缺少变体导致粉红。需要 ShaderVariantCollection 配合
5. **变体白名单要版本管理**：每次新增场景/角色/特效后，重新跑一遍收集器更新白名单

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道变体为什么会爆炸 | 不理解 keyword 笛卡尔积 | 手动计算几个 Shader 的变体数 |
| 裁剪后材质变粉红 | 白名单不全 | 补充运行时收集流程，覆盖所有场景 |
| 不知道在哪一步裁剪 | 不了解 Unity Build Pipeline | 学 IPreprocessBuild / IPreprocessShaders 接口 |
| CI 没法监控变体数 | 不熟悉 CI 集成 | 学习 Unity BatchMode + 脚本导出 |
| 美术经常引入新 keyword | 缺乏预防机制 | 建立 Shader Review 流程 + 变体预算 |

### 🔗 相关问题

- Shader Variant Collection 的最佳实践是什么？手动维护可靠吗？
- 如何在 Shader Graph 中控制变体数量？（SubGraph 的 keyword 传播问题）
- 不同画质档位（Low/Medium/High）如何用不同的 keyword 组合？
- AssetBundle 热更新时如何保证 Shader 变体一致性？
