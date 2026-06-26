---
title: "美术喊'参数太多了'：如何用 ScriptableObject 构建可配置的 Shader 材质模板系统？"
category: "pipeline"
level: 2
tags: ["ScriptableObject", "材质模板", "Shader 参数", "工具开发", "Unity", "工作流"]
hint: "核心：ScriptableObject 做参数容器 + Material Property Block 做运行时注入 + 编辑器面板做美术友好界面"
related: ["pipeline/shader-hot-reload-live-preview", "technical-art/shader-template-system", "pipeline/material-reference-audit-tool"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们项目有 30+ 个角色，每个角色有 4~6 套皮肤，每套皮肤有 15 个 Shader 参数需要调。美术现在直接在 Material Inspector 里调参数，经常出错，参数之间有依赖关系（比如'金属度>0.8 时粗糙度不能<0.3'）。你来做一套工具，让美术不用接触 Inspector 就能安全地调参数。」

### ✅ 核心要点

1. **ScriptableObject 做参数容器**：将 Shader 参数抽象为数据资产，可复用、可版本管理
2. **参数分组与约束**：逻辑分组（基础色/金属度/粗糙度/自发光）+ 值域约束 + 互斥规则
3. **MaterialPropertyBlock 运行时注入**：不创建新 Material 实例，GPU Instancing 友好
4. **自定义编辑器面板**：美术友好的 UI（Slider + Color Picker + 预设值），不碰 Inspector
5. **批量应用与回退**：一键应用到角色所有材质，参数变更支持 Undo
6. **版本管理友好**：ScriptableObject 是 YAML 文本，Git diff 可读

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：美术在自定义面板上安全调参，不碰 Inspector，参数有约束，可批量应用
                ↑
倒推1：不碰 Inspector → 需要自定义编辑器窗口
倒推2：参数有约束 → 需要参数描述文件（Schema）→ ScriptableObject
倒推3：参数可复用 → 每套皮肤 = 一个 ScriptableObject 资产
倒推4：不创建新 Material → 用 MaterialPropertyBlock 注入
倒推5：批量应用 → 遍历角色所有 Renderer，统一注入
倒推6：参数依赖规则 → ScriptableObject 的 OnValidate 做约束检查
倒推7：版本管理 → YAML 序列化，Git diff 友好
```

#### 知识点拆解（倒推树）

```
ScriptableObject 材质模板系统
├── 数据层：ScriptableObject 设计
│   ├── 参数定义类（名称/类型/范围/默认值/分组）
│   ├── 参数组（一组关联参数，如"PBR 基础"）
│   ├── 约束规则（值域/互斥/依赖）
│   └── 预设资产（Preset：某角色某皮肤的具体参数集）
├── 运行时：MaterialPropertyBlock 注入
│   ├── 为什么不用 Material 实例？（增加 SetPass Call / 内存）
│   ├── MPB 的限制（不能改变 Shader 关键字 / Render Queue）
│   ├── MPB 与 GPU Instancing 的关系
│   └── SRP Batcher 兼容性（MPB 会打断 SRP Batcher！）
├── 编辑器工具层
│   ├── 自定义 EditorWindow（美术面板）
│   ├── PropertyDrawer（每类参数的自定义绘制）
│   ├── 预设保存/加载（ScriptableObject CreateAsset）
│   └── 批量选择与应用（Selection + Undo）
├── 约束系统
│   ├── Attribute 标记（[Range] [Tooltip]）
│   ├── OnValidate() 中的运行时检查
│   ├── 自定义校验器（Validator Pattern）
│   └── 错误报告（Console + UI 标红）
├── 工作流集成
│   ├── 角色预制体关联（自动找到 Renderer 组件）
│   ├── 材质模板绑定（哪个 Slot 用哪个参数组）
│   ├── 一键导出/导入 JSON（与外部工具互通）
│   └── CI 校验（Build 前自动检查所有预设合法性）
└── 进阶
    ├── Shader Variant 收集（根据预设自动生成 Keyword 组合）
    ├── 性能预估（根据参数推断渲染开销）
    └── 自动截图（应用参数后自动截屏对比）
```

#### 代码实现

**1. 参数定义 ScriptableObject：**

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

// 单个参数的定义
[Serializable]
public class ShaderParamDef
{
    public string displayName;       // 美术看到的名字："主色调"
    public string shaderProperty;    // Shader 属性名："_BaseColor"
    public ParamType type;           // 参数类型
    public Vector2 range = new(0, 1); // 值域
    public string group = "基础";     // 分组
    [Tooltip("依赖条件，留空表示无依赖")]
    public string dependencyRule;    // 如 "Metallic > 0.8 → Smoothness Min = 0.3"

    public enum ParamType { Color, Float, Range, Texture, Vector, Keyword }
}

// 参数模板（定义结构）
[CreateAssetMenu(fileName = "MaterialTemplate", menuName = "TA/Material Template")]
public class MaterialTemplate : ScriptableObject
{
    public string shaderName;                    // 关联的 Shader
    public string templateName;                   // 模板名称
    public string description;                    // 描述
    public List<ShaderParamDef> paramDefs = new(); // 参数定义列表

    // 按分组获取参数
    public Dictionary<string, List<ShaderParamDef>> GetGroupedParams()
    {
        var dict = new Dictionary<string, List<ShaderParamDef>>();
        foreach (var def in paramDefs)
        {
            if (!dict.ContainsKey(def.group))
                dict[def.group] = new List<ShaderParamDef>();
            dict[def.group].Add(def);
        }
        return dict;
    }
}

// 参数预设（具体值实例，每个角色每套皮肤一个）
[CreateAssetMenu(fileName = "MaterialPreset", menuName = "TA/Material Preset")]
public class MaterialPreset : ScriptableObject
{
    public MaterialTemplate template;             // 引用的模板
    public string characterName;                  // 角色名
    public string skinName;                       // 皮肤名

    [Serializable]
    public class ParamValue
    {
        public string shaderProperty;
        public Color colorValue;
        public float floatValue;
        public Vector4 vectorValue;
        public Texture textureValue;
        public bool keywordEnabled;
    }

    public List<ParamValue> values = new();

    // 校验参数是否符合模板约束
    public List<string> Validate()
    {
        var errors = new List<string>();
        foreach (var val in values)
        {
            var def = template.paramDefs.Find(d => d.shaderProperty == val.shaderProperty);
            if (def == null) continue;

            // 值域检查
            if (def.type == ShaderParamDef.ParamType.Range)
            {
                if (val.floatValue < def.range.x || val.floatValue > def.range.y)
                    errors.Add($"{def.displayName} 超出范围 [{def.range.x}, {def.range.y}]");
            }

            // 依赖规则检查（简化版）
            if (!string.IsNullOrEmpty(def.dependencyRule))
            {
                // 解析 "Metallic > 0.8 → Smoothness Min = 0.3" 类规则
                // 实际项目中用更完整的规则引擎
            }
        }
        return errors;
    }
}
```

**2. 运行时 MaterialPropertyBlock 注入：**

```csharp
using UnityEngine;

public class MaterialPresetApplier : MonoBehaviour
{
    [SerializeField] private MaterialPreset preset;
    [SerializeField] private Renderer[] targetRenderers; // 角色身上的 Renderer

    private MaterialPropertyBlock _mpb;
    private static readonly int BaseColorID = Shader.PropertyToID("_BaseColor");
    private static readonly int MetallicID = Shader.PropertyToID("_Metallic");
    private static readonly int SmoothnessID = Shader.PropertyToID("_Smoothness");
    private static readonly int EmissionColorID = Shader.PropertyToID("_EmissionColor");

    void Awake()
    {
        _mpb = new MaterialPropertyBlock();
        ApplyPreset();
    }

    public void ApplyPreset()
    {
        if (preset == null || targetRenderers == null) return;

        foreach (var renderer in targetRenderers)
        {
            renderer.GetPropertyBlock(_mpb);

            foreach (var val in preset.values)
            {
                var def = preset.template.paramDefs.Find(
                    d => d.shaderProperty == val.shaderProperty);
                if (def == null) continue;

                switch (def.type)
                {
                    case ShaderParamDef.ParamType.Color:
                        _mpb.SetColor(val.shaderProperty, val.colorValue);
                        break;
                    case ShaderParamDef.ParamType.Float:
                    case ShaderParamDef.ParamType.Range:
                        _mpb.SetFloat(val.shaderProperty, val.floatValue);
                        break;
                    case ShaderParamDef.ParamType.Vector:
                        _mpb.SetVector(val.shaderProperty, val.vectorValue);
                        break;
                    case ShaderParamDef.ParamType.Texture:
                        _mpb.SetTexture(val.shaderProperty, val.textureValue);
                        break;
                }
            }

            renderer.SetPropertyBlock(_mpb);
        }
    }

    // 运行时动态切换皮肤
    public void SwitchPreset(MaterialPreset newPreset)
    {
        preset = newPreset;
        ApplyPreset();
    }
}
```

**3. 美术编辑器面板：**

```csharp
using UnityEditor;
using UnityEngine;

public class MaterialPresetEditorWindow : EditorWindow
{
    private MaterialPreset _currentPreset;
    private Vector2 _scrollPos;
    private Dictionary<string, bool> _groupFoldout = new();

    [MenuItem("TA Tools/Material Preset Editor")]
    static void Open() => GetWindow<MaterialPresetEditorWindow>("材质预设编辑器");

    void OnGUI()
    {
        _currentPreset = (MaterialPreset)EditorGUILayout.ObjectField(
            "当前预设", _currentPreset, typeof(MaterialPreset), false);

        if (_currentPreset == null || _currentPreset.template == null)
        {
            EditorGUILayout.HelpBox("请选择预设资产", MessageType.Info);
            return;
        }

        // 校验
        var errors = _currentPreset.Validate();
        if (errors.Count > 0)
        {
            EditorGUILayout.HelpBox(
                $"发现 {errors.Count} 个问题:\n" + string.Join("\n", errors),
                MessageType.Warning);
        }

        // 按分组绘制参数
        var grouped = _currentPreset.template.GetGroupedParams();
        _scrollPos = EditorGUILayout.BeginScrollView(_scrollPos);

        foreach (var kvp in grouped)
        {
            _groupFoldout.TryGetValue(kvp.Key, out bool foldout);
            _groupFoldout[kvp.Key] = EditorGUILayout.BeginFoldoutHeaderGroup(foldout, kvp.Key);

            if (_groupFoldout[kvp.Key])
            {
                foreach (var def in kvp.Value)
                {
                    DrawParamField(def, _currentPreset);
                }
            }
            EditorGUILayout.EndFoldoutHeaderGroup();
        }

        EditorGUILayout.EndScrollView();

        // 底部操作按钮
        EditorGUILayout.Space(10);
        using (new EditorGUILayout.HorizontalScope())
        {
            if (GUILayout.Button("应用到选中物体", GUILayout.Height(30)))
            {
                ApplyToSelection();
            }
            if (GUILayout.Button("保存预设", GUILayout.Height(30)))
            {
                EditorUtility.SetDirty(_currentPreset);
                AssetDatabase.SaveAssets();
            }
        }
    }

    void DrawParamField(ShaderParamDef def, MaterialPreset preset)
    {
        var val = preset.values.Find(v => v.shaderProperty == def.shaderProperty);
        if (val == null) return;

        using (new EditorGUILayout.HorizontalScope())
        {
            GUILayout.Label(def.displayName, GUILayout.Width(100));

            switch (def.type)
            {
                case ShaderParamDef.ParamType.Color:
                    val.colorValue = EditorGUILayout.ColorField(val.colorValue);
                    break;
                case ShaderParamDef.ParamType.Range:
                    val.floatValue = EditorGUILayout.Slider(
                        val.floatValue, def.range.x, def.range.y);
                    break;
                case ShaderParamDef.ParamType.Float:
                    val.floatValue = EditorGUILayout.FloatField(val.floatValue);
                    break;
            }
        }
    }

    void ApplyToSelection()
    {
        foreach (var go in Selection.gameObjects)
        {
            var applier = go.GetComponent<MaterialPresetApplier>();
            if (applier == null) applier = go.AddComponent<MaterialPresetApplier>();
            Undo.RecordObject(applier, "Apply Material Preset");
            // 通过反射或公共方法设置 preset 并应用
        }
    }
}
```

### ⚡ 实战经验

1. **MPB 打断 SRP Batcher 是一个大坑**：如果项目依赖 SRP Batcher 减少 CPU 开销，MPB 会让该 Renderer 退出 Batcher。权衡：用 Material 实例（影响 Instancing）还是 MPB（影响 SRP Batcher）需要 profile 决定。
2. **参数约束比你想的重要**：美术最容易犯的错误是金属度=1 + 粗糙度=0 → 全黑。加约束规则能省无数 debug 时间。
3. **预设资产的 Git 冲突处理**：ScriptableObject 是 YAML，但如果两人同时改同一预设会冲突。给每个角色/皮肤独立预设文件，减少冲突面。
4. **运行时切换皮肤用 MPB 最快**：比替换 Material 实例快 5~10 倍，因为不需要创建新材质资产。
5. **导出 JSON 给外部工具**：Substance Painter / Houdini 可以读 JSON 格式的参数预设，实现"外部调参 → 导入 Unity"的工作流。
6. **CI 中校验预设**：Build Pipeline 加一步检查所有预设的 `Validate()`，不符合规则的直接报错。防止美术提交非法参数。

### 🎯 能力体检清单

- [ ] ScriptableObject 和 MonoBehaviour 的区别是什么？为什么参数预设用 SO 而不是 MB？
- [ ] MaterialPropertyBlock 会影响 GPU Instancing 吗？会影响 SRP Batcher 吗？分别说明。
- [ ] 如果美术需要切换 Shader Keyword（如 `_EMISSION`），MPB 能做到吗？不能的话怎么办？
- [ ] 你的参数约束系统如何设计？硬编码 vs 数据驱动（规则引擎）的取舍是什么？
- [ ] 30 个角色 × 6 套皮肤 = 180 个预设资产，如何组织目录结构和命名规范？
- [ ] 如果策划要求"运行时玩家可以自由染色"，你的系统需要做什么改动？
- [ ] 这套系统如何和 Shader Variant Stripping 配合？（预设中用到的 Keyword 组合需要在 Build 时收集）

### 🔗 相关问题

- [Shader 热重载与实时预览：编辑器工具如何提升 Shader 迭代效率？](pipeline/shader-hot-reload-live-preview)
- [Shader 模板系统：如何让美术安全复用 Shader 而不出错？](technical-art/shader-template-system)
- [材质引用审计工具：项目中材质引用混乱如何排查？](pipeline/material-reference-audit-tool)
- [Shader Variant 构建裁剪：移动端如何精确控制 Shader 变体数量？](pipeline/shader-variant-stripping-build-pipeline)
