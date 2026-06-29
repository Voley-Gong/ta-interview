---
title: "运行时 Shader 编译卡顿：如何建立 Shader 变体收集与预热管线？"
category: "pipeline"
level: 3
tags: ["Shader Variant", "预热", "卡顿", "构建管线", "Shader Variant Collection", "运行时"]
hint: "玩家第一次看到某个特效时卡了 200ms——这是 Shader 编译延迟，不是性能问题，是预热管线缺失"
related: ["optimization/shader-variant-explosion", "pipeline/shader-variant-stripping-build-pipeline", "optimization/loading-stall-hitch-spike"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的手游上线后收到大量玩家反馈：每次进入新场景或第一次释放大招时会卡顿 200-500ms。Profiler 抓到是 Shader.CompileGPUProgram。项目用了 URP + Shader Graph，构建后有 12000+ 个 Shader 变体。你怎么解决这个运行时 Shader 编译卡顿问题？」

这是米哈游、网易雷火等重度项目必问的工程题。Shader 变体管理是 TA/渲染程序的核心能力——不只是理论，更是管线工程。

### ✅ 核心要点

1. **根因定位**：Shader 编译发生在 GPU Program 第一次被使用时（JIT 编译），不是构建时
2. **Shader Variant Collection (SVC)**：Unity 的官方方案——预收集变体，在 Loading Screen 期间预热
3. **变体收集策略**：不能手动枚举 12000 个变体，需要从运行时抓取实际使用的变体
4. **预热时机**：在 Loading Screen / 启动画面中异步预编译，不能在战斗中触发
5. **构建管线裁剪**：结合 `IPreprocessShaders` 接口在构建时剥离不需要的变体

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：消灭运行时 Shader.CompileGPUProgram 卡顿
     ↓
Step 1：确认卡顿来源
  ├── Profiler > GPU > Shader.CompileGPUProgram
  ├── 确认是哪些 Shader 的哪些变体在编译
  └── 典型触发点：新材质首次渲染、Shader Keyword 切换、渲染路径切换
     ↓
Step 2：收集实际需要的变体
  ├── 方案A：Editor 模式下跑完所有场景/角色，导出 SVC
  ├── 方案B：运行时 Hook Material.SetPass / ShaderCompiler，记录使用的变体
  └── 方案C：静态分析——解析材质引用的 Shader + Keyword 组合
     ↓
Step 3：构建时裁剪不需要的变体
  ├── IPreprocessShaders 接口 + 变体白名单
  ├── 只保留 SVC 中列出的变体
  └── 12000 → 3000（减少 75%）
     ↓
Step 4：运行时预热
  ├── Loading Screen 期间：Shader.WarmupAllShaders()（粗暴但有效）
  ├── 精确预热：ShaderVariantCollection.WarmUp()
  └── 分帧预热：避免 Loading Screen 本身卡死
     ↓
Step 5：CI 门禁
  ├── 构建后检查：APK 内 Shader 变体数 < 阈值
  ├── 新增 Shader 自动加入 SVC 收集流程
  └── 预热覆盖率检测：实际渲染使用的变体 ⊂ SVC 变体
```

#### 知识点拆解（倒推树）

```
Shader 变体预热管线
├── Shader 编译机制理解
│   ├── JIT 编译（运行时）
│   │   ├── 第一次使用该变体时编译
│   │   ├── 编译成本：1-500ms / 变体（取决于复杂度）
│   │   └── 多关键词组合爆炸：N keywords → 2^N 变体
│   ├── AOT 编译（构建时）
│   │   ├── 部分平台支持 Precompile
│   │   ├── 移动端：Shader 编译为目标平台 SPIR-V / DXBC
│   │   └── 构建时间增长（需权衡）
│   └── 变体来源
│       ├── Shader Keywords（multi_compile / shader_feature）
│       ├── Shader Graph 的内置 Keywords（7+ 个）
│       ├── URP/HDRP 自带的 Keywords（几十个）
│       └── 材质属性 → Keyword 映射
├── 变体收集方法
│   ├── Editor 运行收集
│   │   ├── Edit > Graphics Settings > Shader Loading > Save to Asset
│   │   ├── 遍历所有场景 + 预览所有材质 → 导出 SVC
│   │   ├── 缺点：无法覆盖动态生成的 Keyword 组合
│   │   └── 缺点：人工遍历容易遗漏
│   ├── 运行时 Hook 收集
│   │   ├── ShaderUtil.RegisterShaderCompilerDelegate（内部 API）
│   │   ├── 自定义 ShaderProcessor：Hook 编译回调记录变体
│   │   ├── 输出变体日志 → 导入 Editor 生成 SVC
│   │   └── 优点：100% 覆盖实际使用场景
│   └── 静态分析
│       ├── 解析 .mat 文件提取 Keyword 组合
│       ├── 解析 .shader 文件提取 multi_compile / shader_feature
│       ├── 脚本分析：C# 中 EnableKeyword 调用的参数
│       └── 缺点：无法预测运行时的动态组合
├── 预热策略
│   ├── Shader.WarmupAllShaders()
│   │   ├── 编译当前加载的所有 Shader 的所有变体
│   │   ├── 优点：最简单
│   │   ├── 缺点：编译大量不需要的变体（内存 + 时间浪费）
│   │   └── 适用：变体数 < 1000 的小项目
│   ├── ShaderVariantCollection.WarmUp()
│   │   ├── 只编译 SVC 中列出的变体
│   │   ├── 优点：精确控制
│   │   ├── 缺点：需要完整的 SVC
│   │   └── 适用：中大型项目首选
│   └── 分帧预热
│       ├── 每帧编译 N 个变体（N=10~50）
│       ├── 搭配 Loading Screen 的进度条
│       ├── 预估总时间 = 变体数 × 平均编译时间 / 每帧数量
│       └── 超时保护：如果预热超过 5 秒，降低每帧数量
├── 构建时裁剪
│   ├── IPreprocessShaders 接口
│   │   ├── OnProcessShader(shader, snippet, customization)
│   │   ├── 返回 true = 保留，false = 剥离
│   │   ├── 结合编译符号：#if UNITY_ANDROID 等
│   │   └── 注册：[InitializeOnLoad] 自动注册
│   ├── 常见裁剪规则
│   │   ├── 平台裁剪：移动端去掉 PC-only 变体（如 Ray Tracing）
│   │   ├── 质量裁剪：Low 设备去掉 High Quality 变体
│   │   ├── 功能裁剪：确认不用的功能关键词（如 _ADDITIONAL_LIGHT_SHADOWS）
│   │   └── 场景裁剪：只保留当前场景引用的 Shader
│   └── 裁剪效果度量
│       ├── 编译前：变体数、APK 中 .shader 大小
│       ├── 编译后：实际变体数、Shader 内存占用
│       └── 工具：Shader Variant Information 工具
└── CI/CD 集成
    ├── 构建报告
    │   ├── 变体数对比（本次 vs 上次）
    │   ├── 变体增量告警（新增 > 100 触发 Review）
    │   └── 预热覆盖率 = SVC 变体 / 实际使用变体
    ├── 自动化 SVC 更新
    │   ├── 每次构建后跑一遍全场景预览
    │   ├── 自动导出 SVC 并提交
    │   └── Code Review 确认变体增长合理性
    └── 回归检测
        ├── 灰度包收到卡顿报告 → 检查是否有新变体未预热
        └── Profiler 自动跑场景 → 检测 CompileGPUProgram 事件
```

#### 代码实现

**变体收集器（运行时 Hook + 导出）：**

```csharp
using System.Collections.Generic;
using System.IO;
using UnityEngine;
using UnityEngine.Rendering;

#if UNITY_EDITOR
using UnityEditor;
using UnityEditor.Rendering;

public class ShaderVariantCollector : MonoBehaviour
{
    [Header("收集设置")]
    public Camera captureCamera;
    public List<Material> additionalMaterials = new();
    public int maxFrames = 300; // 收集帧数

    private HashSet<string> _usedVariants = new();
    private int _frameCount = 0;

    void Start()
    {
        // Hook 到渲染回调
        RenderPipelineManager.endFrameRendering += OnEndFrame;
    }

    void OnEndFrame(ScriptableRenderContext ctx, Camera[] cameras)
    {
        _frameCount++;

        // 遍历所有可见材质，记录使用的 Keyword 组合
        var renderers = FindObjectsByType<Renderer>(FindObjectsSortMode.None);
        foreach (var renderer in renderers)
        {
            var mats = renderer.sharedMaterials;
            foreach (var mat in mats)
            {
                if (mat == null || mat.shader == null) continue;

                // 获取当前激活的 Keywords
                var keywords = mat.shaderKeywords;
                if (keywords.Length == 0) continue;

                // 生成变体标识
                string variantKey = $"{mat.shader.name}|{string.Join(",", keywords)}";
                _usedVariants.Add(variantKey);
            }
        }

        if (_frameCount >= maxFrames)
        {
            ExportVariantLog();
            RenderPipelineManager.endFrameRendering -= OnEndFrame;
            Debug.Log($"[变体收集] 完成！共 {_usedVariants.Count} 个变体");
        }
    }

    void ExportVariantLog()
    {
        string path = Path.Combine(Application.persistentDataPath, "shader_variants.txt");
        using var writer = new StreamWriter(path);
        writer.WriteLine($"# Shader Variant Collection Log");
        writer.WriteLine($"# Collected: {_usedVariants.Count} variants in {_frameCount} frames");
        writer.WriteLine($"# Date: {System.DateTime.Now:yyyy-MM-dd HH:mm:ss}");
        writer.WriteLine();

        var sorted = new List<string>(_usedVariants);
        sorted.Sort();
        foreach (var v in sorted)
        {
            writer.WriteLine(v);
        }

        Debug.Log($"[变体收集] 日志已导出到: {path}");
    }

    void OnDestroy()
    {
        RenderPipelineManager.endFrameRendering -= OnEndFrame;
    }
}
#endif
```

**SVC 生成工具（Editor 脚本，从日志生成 SVC）：**

```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;
using System.IO;
using System.Collections.Generic;

public class SVCGenerator : EditorWindow
{
    [MenuItem("Tools/TA/Shader Variant Collection Generator")]
    static void Open() => GetWindow<SVCGenerator>("SVC Generator");

    void OnGUI()
    {
        if (GUILayout.Button("从变体日志生成 SVC"))
        {
            string logPath = EditorUtility.OpenFilePanel(
                "选择变体日志", Application.persistentDataPath, "txt");
            if (string.IsNullOrEmpty(logPath)) return;

            GenerateSVC(logPath);
        }

        if (GUILayout.Button("分析当前项目所有材质的变体"))
        {
            AnalyzeAllMaterials();
        }
    }

    void GenerateSVC(string logPath)
    {
        var lines = File.ReadAllLines(logPath);
        var variants = new List<ShaderVariantCollection.ShaderVariant>();
        var byShader = new Dictionary<Shader, List<string[]>>();

        foreach (var line in lines)
        {
            if (string.IsNullOrEmpty(line) || line.StartsWith("#")) continue;

            var parts = line.Split('|');
            if (parts.Length != 2) continue;

            var shader = Shader.Find(parts[0]);
            if (shader == null) continue;

            var keywords = parts[1].Split(',');
            if (!byShader.ContainsKey(shader))
                byShader[shader] = new List<string[]>();
            byShader[shader].Add(keywords);
        }

        // 创建 SVC Asset
        var svc = new ShaderVariantCollection();
        foreach (var kv in byShader)
        {
            foreach (var keywords in kv.Value)
            {
                // 尝试创建 ShaderVariant（不是所有组合都有效）
                try
                {
                    var variant = new ShaderVariantCollection.ShaderVariant(kv.Key, PassType.ForwardBase, keywords);
                    if (svc.Add(variant))
                    {
                        // 成功添加
                    }
                }
                catch (System.Exception e)
                {
                    Debug.LogWarning($"[SVC] 跳过无效变体: {kv.Key.name} | {string.Join(",", keywords)} | {e.Message}");
                }
            }
        }

        // 保存
        string savePath = EditorUtility.SaveFilePanelInProject(
            "保存 SVC", "RuntimeVariants", "shadervariants", "保存 Shader Variant Collection");
        if (!string.IsNullOrEmpty(savePath))
        {
            AssetDatabase.CreateAsset(svc, savePath);
            AssetDatabase.SaveAssets();
            Debug.Log($"[SVC] 生成完成: {savePath} ({svc.variantCount} 个变体)");
        }
    }

    void AnalyzeAllMaterials()
    {
        var matGuids = AssetDatabase.FindAssets("t:Material");
        var keywordSet = new HashSet<string>();
        int totalVariants = 0;

        foreach (var guid in matGuids)
        {
            var path = AssetDatabase.GUIDToAssetPath(guid);
            var mat = AssetDatabase.LoadAssetAtPath<Material>(path);
            if (mat == null || mat.shader == null) continue;

            // 统计该 Shader 可能的变体数
            var shader = mat.shader;
            int keywordCount = shader.keywordSpace.keywordCount;
            // 材质启用的 keywords
            var activeKeywords = mat.shaderKeywords;

            foreach (var kw in activeKeywords)
            {
                keywordSet.Add($"{shader.name}:{kw}");
            }

            totalVariants += (int)Mathf.Pow(2, Mathf.Min(keywordCount, 20)); // 上限保护
        }

        Debug.Log($"[变体分析] 材质总数: {matGuids.Length}");
        Debug.Log($"[变体分析] 理论变体数（上限）: {totalVariants}");
        Debug.Log($"[变体分析] 实际启用的 Keyword: {keywordSet.Count}");
    }
}
#endif
```

**分帧预热（Loading Screen 集成）：**

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;

public class ShaderWarmupManager : MonoBehaviour
{
    public ShaderVariantCollection runtimeVariants;
    public int variantsPerFrame = 20;     // 每帧编译变体数
    public float maxWarmupTime = 8f;      // 最大预热时间（秒）

    private List<ShaderVariantCollection.ShaderVariant> _pendingVariants;
    private float _startTime;

    public IEnumerator WarmupRoutine(System.Action<float> onProgress, System.Action onComplete)
    {
        if (runtimeVariants == null || runtimeVariants.variantCount == 0)
        {
            onComplete?.Invoke();
            yield break;
        }

        _pendingVariants = new List<ShaderVariantCollection.ShaderVariant>(runtimeVariants.variants);
        _startTime = Time.realtimeSinceStartup;

        int total = _pendingVariants.Count;
        int processed = 0;

        while (_pendingVariants.Count > 0)
        {
            int batch = Mathf.Min(variantsPerFrame, _pendingVariants.Count);

            for (int i = 0; i < batch; i++)
            {
                // WarmUp 是同步的——逐个编译
                var svc = new ShaderVariantCollection();
                svc.Add(_pendingVariants[0]);
                svc.WarmUp();
                _pendingVariants.RemoveAt(0);
                processed++;
            }

            float progress = (float)processed / total;
            onProgress?.Invoke(progress);

            // 超时保护
            if (Time.realtimeSinceStartup - _startTime > maxWarmupTime)
            {
                Debug.LogWarning($"[预热] 超时！已编译 {processed}/{total} 个变体，跳过剩余 {_pendingVariants.Count} 个");
                break;
            }

            yield return null; // 等待下一帧
        }

        Debug.Log($"[预热] 完成！编译 {processed}/{total} 个变体，耗时 {Time.realtimeSinceStartup - _startTime:F2}s");
        onComplete?.Invoke();
    }
}

// 使用示例：Loading Screen 调用
// var warmup = FindObjectOfType<ShaderWarmupManager>();
// StartCoroutine(warmup.WarmupRoutine(
//     progress => loadingBar.fillAmount = progress,
//     () => SceneManager.LoadScene(nextScene)
// ));
```

**构建时变体裁剪（IPreprocessShaders）：**

```csharp
#if UNITY_EDITOR
using UnityEditor.Build;
using UnityEditor.Rendering;
using UnityEngine;

class ShaderVariantStripper : IPreprocessShadersWithReport
{
    public int callbackOrder => 1000;

    // 从构建时注入的白名单读取
    private HashSet<string> _whitelist;

    public void OnPreprocessShaders(Shader shader, ShaderSnippetData snippet,
        IList<ShaderCompilerData> data)
    {
        LoadWhitelist();

        var toRemove = new List<ShaderCompilerData>();

        foreach (var entry in data)
        {
            // 获取该变体的 Keyword 组合
            var keywords = entry.shaderKeywordSet.GetShaderKeywords();

            // 裁剪规则 1：移动端去掉 PC-only 关键词
            #if UNITY_ANDROID || UNITY_IOS
            foreach (var kw in keywords)
            {
                if (kw.name.Contains("RAY_TRACING") ||
                    kw.name.Contains("STORE_SCENECOLOR") ||
                    kw.name.Contains("SCREEN_SPACE_GLOBAL_ILLUMINATION"))
                {
                    toRemove.Add(entry);
                    break;
                }
            }
            #endif

            // 裁剪规则 2：白名单外的变体剥离
            if (_whitelist != null && _whitelist.Count > 0)
            {
                string key = $"{shader.name}_{string.Join("_", GetKeywordNames(keywords))}";
                if (!_whitelist.Contains(key))
                {
                    toRemove.Add(entry);
                }
            }
        }

        foreach (var entry in toRemove)
            data.Remove(entry);

        Debug.Log($"[变体裁剪] {shader.name}: 移除 {toRemove.Count} 个变体");
    }

    void LoadWhitelist()
    {
        if (_whitelist != null) return;
        _whitelist = new HashSet<string>();
        var svc = AssetDatabase.LoadAssetAtPath<ShaderVariantCollection>(
            "Assets/Settings/RuntimeVariants.shadervariants");
        if (svc == null) return;

        foreach (var variant in svc.variants)
        {
            var names = new List<string>();
            foreach (var kw in variant.keywords)
                names.Add(kw);
            _whitelist.Add($"{variant.shader.name}_{string.Join("_", names)}");
        }
    }

    string[] GetKeywordNames(ShaderKeyword[] keywords)
    {
        var names = new string[keywords.Length];
        for (int i = 0; i < keywords.Length; i++)
            names[i] = keywords[i].name;
        return names;
    }
}
#endif
```

### ⚡ 实战经验

1. **Shader Graph 是变体炸弹**：每个 Shader Graph 默认带 7+ 个内置 Keyword，每多一个 Keyword 变体数翻倍。项目中大量使用 Shader Graph 时，变体数会指数级膨胀
2. **SVC 覆盖率 > 95% 才够用**：灰度测试时收集玩家设备的实际变体使用日志，和 SVC 做对比，覆盖率低于 95% 就有卡顿风险
3. **Loading Screen 是预热黄金期**：3 秒的 Loading Screen 可以编译约 300-500 个变体（骁龙8 Gen2），足够覆盖一个场景的全部需求
4. **WarmupAllShaders 不是洪水猛兽**：对于变体数 < 2000 的项目，启动时一次性 WarmupAllShaders 反而比精心设计的 SVC 更可靠（只是启动慢 2-3 秒）
5. **Apple 平台特殊处理**：iOS 上 Metal Shader 编译比 Android Vulkan 更快，同样的变体预热时间可以翻倍变体数

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能解释 Shader 变体为什么会爆炸（2^N） | → Shader 编译基础盲区 |
| 知道 ShaderVariantCollection 的用途和使用方式 | → Unity 渲染管线盲区 |
| 能写 IPreprocessShaders 裁剪不需要的变体 | → 构建管线扩展盲区 |
| 知道 JIT 编译发生在什么时机 | → GPU 驱动模型盲区 |
| 能设计分帧预热避免 Loading 卡死 | → 工程设计能力盲区 |

### 🔗 相关问题

- [optimization/shader-variant-explosion](../optimization/shader-variant-explosion.md) — Shader 变体爆炸后的瘦身方案
- [pipeline/shader-variant-stripping-build-pipeline](../pipeline/shader-variant-stripping-build-pipeline.md) — 构建管线变体裁剪
- [optimization/loading-stall-hitch-spike](../optimization/loading-stall-hitch-spike.md) — Loading 卡顿定位与消灭
