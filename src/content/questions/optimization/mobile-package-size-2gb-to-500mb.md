---
title: "手游包体从 2GB 压到 500MB：你该怎么制定资产瘦身方案？"
category: "optimization"
level: 3
tags: ["包体优化", "Asset Bundle", "纹理压缩", "音频压缩", "资源管理"]
hint: "不是单纯压纹理——要从资产审计、冗余清理、按需加载、格式策略四个维度系统作战"
related: ["optimization/gpu-memory-budget", "optimization/shader-variant-explosion", "technical-art/mobile-texture-compression"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的一款二次元开放世界手游，首次包体已经膨胀到 2.3GB，渠道反馈安装转化率低于 15%。老板要求一个月内把首包压到 500MB 以内，剩余资源走边玩边下。你是 TA，给我一个完整的瘦身方案。」

### ✅ 核心要点

1. **资产审计先行**：不知道钱花在哪就不知道该砍哪里——先用工具扫描全量资产
2. **纹理是包体大户**：通常占 50%-70%，从分辨率、压缩格式、图集三个方向下手
3. **音频不可忽视**：BGM 和语音往往占 15%-25%，切换编码格式收益巨大
4. **冗余与重复资源**：项目迭代中积累的「废弃资产」可能占 10%-20%
5. **分包与按需下载**：首包只留「核心体验」所需的资产，其余走 CDN 边玩边下

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：首包 ≤ 500MB，玩家体验不降级
                ↑
倒推1：500MB 能装什么？→ 核心场景 + 首章角色 + UI + 基础特效
倒推2：哪些资产该移出首包？→ 后续章节、活动资源、高清CG、语音包
倒推3：留在首包的资产怎么压？→ 纹理降分辨率 + ASTC + 图集合并
倒推4：有没有浪费？→ 冗余资产清理（重复材质、未引用贴图、空引用）
倒推5：如何持续可控？→ 建立资产规范 + CI 自动检查 + 包体监控面板
```

#### 知识点拆解（倒推树）

```
包体瘦身 2GB → 500MB
├── 资产审计（Phase 1：摸底）
│   ├── Unity Asset Auditor / Build Report
│   ├── 按类型统计：纹理 / 网格 / 音频 / 动画 / Shader
│   ├── 冗余检测：重复资源（同名 / 同 Hash）
│   └── 依赖分析：未被引用的资产（孤儿资产）
├── 纹理优化（Phase 2：大头）
│   ├── 分辨率策略：角色 2048→1024，场景 1024→512，UI 适量保留
│   ├── 压缩格式：ETC2 → ASTC 6x6（质量更好体积更小）
│   ├── 通道合并：Splatmap / ORM 贴图合并（RGBA 四通道复用）
│   ├── Mipmap 策略：UI 纹理关闭 Mipmap（省 33%）
│   └── 图集合并：散碎小贴图 → Atlas（减少冗余 Padding）
├── 网格与动画（Phase 3）
│   ├── 网格压缩：Mesh Compression 开启
│   ├── LOD 策略：远距离用低模（减顶点数）
│   ├── 动画压缩：Keyframe Reduction / Curve Compression
│   └── 骨骼精简：非关键角色减少骨骼数
├── 音频优化（Phase 4）
│   ├── BGM：WAV → Vorbis（品质 70%），或平台原生格式
│   ├── 语音：单独打 Asset Bundle，按需下载
│   ├── 音效：短音效用 ADPCM 或留本地
│   └── 采样率降级：48kHz → 22kHz（语音可接受）
├── 分包策略（Phase 5：核心）
│   ├── 首包内容定义：新手引导 + 第一章 + 核心角色
│   ├── Asset Bundle 划分：按场景 / 章节 / 功能模块
│   ├── CDN 边玩边下：后台预加载下一场景
│   └── 热更新机制：版本号 + 增量更新（bsdiff/bspatch）
└── 持续管控（Phase 6）
    ├── CI 集成：每次构建输出包体报告
    ├── 资产准入规范：贴图最大尺寸、音频码率上限
    ├── 自动告警：包体超阈值 → 飞书/钉钉通知
    └── 资产 Review 流程：新增资产必须审核
```

#### 代码实现

**Unity 构建报告分析脚本（C#）：**

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections.Generic;
using System.IO;
using System.Linq;

#if UNITY_EDITOR
using UnityEditor;

public static class PackageSizeAuditor
{
    [MenuItem("Tools/TA/包体审计报告")]
    public static void GenerateReport()
    {
        // 获取所有构建中包含的资产
        var allAssets = AssetDatabase.FindAssets("")
            .Select(AssetDatabase.GUIDToAssetPath)
            .Where(path => !path.StartsWith("ProjectSettings/") &&
                          !path.StartsWith("Packages/"))
            .ToList();

        var report = new List<AssetEntry>();
        long totalSize = 0;

        foreach (var path in allAssets)
        {
            var fi = new FileInfo(path);
            if (!fi.Exists) continue;

            var size = fi.Length;
            totalSize += size;

            report.Add(new AssetEntry
            {
                Path = path,
                SizeKB = size / 1024f,
                Type = System.IO.Path.GetExtension(path),
                IsReferenced = CheckReferences(path)
            });
        }

        // 按大小排序，输出 Top 100
        var top = report.OrderByDescending(e => e.SizeKB).Take(100).ToList();
        var orphans = report.Where(e => !e.IsReferenced).OrderByDescending(e => e.SizeKB).ToList();
        var byType = report.GroupBy(e => e.Type)
            .Select(g => new { Type = g.Key, Count = g.Count(), TotalMB = g.Sum(e => e.SizeKB) / 1024f })
            .OrderByDescending(g => g.TotalMB).ToList();

        Debug.Log($"[包体审计] 总资产: {allAssets.Count}, 总大小: {totalSize / 1024f / 1024f:F1} MB");
        Debug.Log($"[包体审计] 孤儿资产: {orphans.Count} 个, 浪费: {orphans.Sum(e => e.SizeKB) / 1024f:F1} MB");

        // 导出 CSV
        ExportCSV(report, "BuildReport/asset_audit.csv");
        ExportOrphanReport(orphans, "BuildReport/orphan_assets.csv");
        ExportTypeSummary(byType, "BuildReport/type_summary.csv");

        AssetDatabase.Refresh();
    }

    static bool CheckReferences(string path)
    {
        // 检查是否被场景或其他资产引用
        var deps = AssetDatabase.GetDependencies(new[] { path });
        // 简化检查：如果在 Resources 或 StreamingAssets 中算被引用
        if (path.Contains("Resources/") || path.Contains("StreamingAssets/"))
            return true;
        // 检查是否被场景引用（需要更完整的依赖图遍历）
        // 此处简化，实际项目用更完善的工具
        return true;
    }

    static void ExportCSV(List<AssetEntry> entries, string path)
    {
        var dir = System.IO.Path.GetDirectoryName(path);
        if (dir != null) Directory.CreateDirectory(dir);
        File.WriteAllLines(path, entries.Select(e =>
            $"{e.Path},{e.SizeKB:F1},{e.Type},{e.IsReferenced}"));
    }

    static void ExportOrphanReport(List<AssetEntry> orphans, string path)
    {
        File.WriteAllLines(path, orphans.Select(e => $"{e.Path},{e.SizeKB:F1}"));
    }

    static void ExportTypeSummary(IEnumerable<dynamic> summary, string path)
    {
        var lines = new List<string> { "Type,Count,TotalMB" };
        foreach (var s in summary)
            lines.Add($"{s.Type},{s.Count},{s.TotalMB:F1}");
        File.WriteAllLines(path, lines);
    }

    struct AssetEntry
    {
        public string Path;
        public float SizeKB;
        public string Type;
        public bool IsReferenced;
    }
}
#endif
```

**Asset Bundle 分包配置示例：**

```csharp
using UnityEngine;
using UnityEditor;
using System.IO;
using System.Collections.Generic;

public static class BundleSplitStrategy
{
    // 分包规则定义
    public static readonly Dictionary<string, string[]> BundleRules = new()
    {
        // 首包（打进 APK/IPA）
        ["core"] = new[] {
            "Assets/Scenes/Boot",
            "Assets/Scenes/Login",
            "Assets/Scenes/Chapter1",
            "Assets/Characters/Starter",
            "Assets/UI/Core",
            "Assets/Shaders",          // Shader 始终在首包（避免变异体问题）
            "Assets/Audio/BGM_Menu",
        },
        // 边玩边下（CDN）
        ["chapter2"] = new[] { "Assets/Scenes/Chapter2", "Assets/Characters/Ch2" },
        ["chapter3"] = new[] { "Assets/Scenes/Chapter3", "Assets/Characters/Ch3" },
        ["voice_pack"] = new[] { "Assets/Audio/Voice" },   // 语音独立包
        ["event_assets"] = new[] { "Assets/Events" },       // 活动资源
    };

    [MenuItem("Tools/TA/生成分包配置")]
    public static void GenerateBundleConfig()
    {
        var config = new BundleConfig
        {
            version = PlayerSettings.bundleVersion,
            bundles = new List<BundleInfo>()
        };

        foreach (var kv in BundleRules)
        {
            long estimatedSize = 0;
            foreach (var folder in kv.Value)
            {
                var guids = AssetDatabase.FindAssets("", new[] { folder });
                foreach (var guid in guids)
                {
                    var path = AssetDatabase.GUIDToAssetPath(guid);
                    if (File.Exists(path))
                        estimatedSize += new FileInfo(path).Length;
                }
            }

            config.bundles.Add(new BundleInfo
            {
                name = kv.Key,
                isFirstPack = kv.Key == "core",
                sizeEstimateMB = estimatedSize / 1024f / 1024f,
                downloadPriority = kv.Key == "core" ? 0 :
                                   kv.Key.StartsWith("chapter") ? 1 : 2
            });
        }

        var json = JsonUtility.ToJson(config, true);
        File.WriteAllText("Assets/StreamingAssets/bundle_config.json", json);
        Debug.Log($"[分包配置] 首包预估: {config.bundles.Find(b => b.isFirstPack).sizeEstimateMB:F0} MB");
        AssetDatabase.Refresh();
    }

    [System.Serializable]
    class BundleConfig { public string version; public List<BundleInfo> bundles; }
    [System.Serializable]
    class BundleInfo { public string name; public bool isFirstPack; public float sizeEstimateMB; public int downloadPriority; }
}
```

**资产优化收益估算表：**

| 优化项 | 原始占比 | 预估收益 | 实施难度 | 风险 |
|--------|----------|----------|----------|------|
| 纹理 ASTC 压缩 | 55% (1.26GB) | -40% (~500MB) | 低 | 极低（ASTC 6x6 质量损失可忽略） |
| 冗余资产清理 | 15% (345MB) | -90% (~310MB) | 中 | 低（需确认引用关系） |
| 音频格式升级 | 12% (276MB) | -60% (~165MB) | 低 | 低（Vorbis 品质 70% 几乎无感） |
| 分辨率降级 | — | -20% (~200MB) | 中 | 中（需逐张 Review） |
| 分包拆分 | — | 首包 -60% | 高 | 中（需测试下载流程） |
| Mipmap 关闭 (UI) | 3% (69MB) | -33% (~23MB) | 低 | 无 |
| **合计预估** | **2.3GB** | **首包 ~450MB** | — | — |

### ⚡ 实战经验

- **先审计再动手**：见过团队上来就压纹理，结果发现 30% 的包体是废弃资产，白压了
- **ASTC 是移动端首选**：ETC2 是保底，ASTC 6x6 比 ETC2 小 20-30% 且质量更好，iOS/Android 全面支持
- **语音包独立**：中文配音动辄几百 MB，拆成独立 Bundle 让玩家选是否下载
- **Shader 不要打 AssetBundle**：Shader 变异体 + 异步加载 = 坑，放首包最安全
- **持续监控比一次性优化更重要**：在 CI 里加一个 `Build Report → 飞书通知` 的步骤，包体超阈值自动告警

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道包体大头在哪 | 资产审计工具 | Unity Build Report / Asset Auditor |
| 压了纹理但包体没小多少 | Build Pipeline 缓存 | 理解 Asset Bundle 打包机制 |
| ASTC 压缩后画质崩 | 压缩格式特性 | 学 ASTC block size 与质量关系 |
| 分包后游戏卡顿 | 下载时机设计 | 边玩边下预加载策略设计 |
| 孤儿资产不敢删 | 引用检测不完整 | Asset Database 依赖图分析 |
| 包体反复膨胀 | 缺乏持续管控 | CI 集成 + 资产准入规范 |

### 🔗 相关问题

- Asset Bundle 的依赖关系怎么设计才能避免重复打包？
- 如何在不降级视觉质量的前提下进一步压缩纹理？（提示：超分重建 + 按需加载高清贴图）
- 热更新补丁包怎么做增量？（bsdiff/bspatch 或 Unity 的差异构建）
- PC 端和移动端的包体策略有什么核心差异？
