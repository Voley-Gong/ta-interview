---
title: "美术说 Shader 调参太痛苦——如何用自定义 Inspector + Tool Window 打造一站式工作流？"
category: "pipeline"
level: 2
tags: ["Unity Editor", "自定义Inspector", "Tool Window", "EditorScripting", "美术工作流", "UI Toolkit"]
hint: "核心是减少美术在 Inspector、Material、Scene 之间的反复横跳——用 EditorWindow + MaterialPropertyDrawer 把所有参数集中到一个面板"
related: ["pipeline/shader-hot-reload-live-preview", "pipeline/scriptableobject-material-template-pipeline", "soft-skills/art-vague-feedback-data-driven-diagnosis"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们项目有 40+ 个自定义 Shader，每个 Shader 有 10-20 个参数。美术调材质时，经常在 Inspector 里找不到参数、不理解参数含义，或者不小心调了不该动的值。你作为 TA，怎么用 Unity Editor 工具改善这个工作流？」

### ✅ 核心要点

1. **问题核心**：原生 Material Inspector 对美术不友好——参数暴露无分组、无说明、无范围限制、无预设
2. **分层解决**：ShaderLab 属性标记（HUD/Label/Toggle）→ 自定义 ShaderGUI（分组 + 条件显示）→ 独立 EditorWindow（多材质批量编辑）
3. **关键技能**：ShaderGUI 继承、EditorWindow 生命周期、SerializedObject 修改、Undo/Redo 支持
4. **进阶**：UI Toolkit（原 UIElements）替代 IMGUI，实现响应式布局
5. **交付标准**：美术打开面板 → 直观看到所有参数 → 分组明确 → 误操作有保护 → 有预设保存

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：美术打开一个面板 → 清晰看到分组参数 → 调参 → 保存预设 → 一键应用到其他材质
                ↑
倒推1：需要自定义 UI → Unity ShaderGUI（Per-Material Inspector）
倒推2：参数需要分组 → FoldoutGroup / Header 标签
倒推3：部分参数需要条件显示 → 如开了 Emissive 才显示 Emission Color
倒推4：需要批量编辑 → 独立 EditorWindow（跨多个材质）
倒推5：需要预设管理 → ScriptableObject 保存材质参数快照
倒推6：需要误操作保护 → Undo.RegisterCompleteObjectUndo + 范围 clamp
```

#### 知识点拆解（倒推树）

```
Unity Editor 工具开发
├── Level 1: ShaderLab 属性标记（零成本）
│   ├── [HDR] 标记（颜色 > 1.0 的 HDR 颜色拾取器）
│   ├── [Toggle] / [ToggleUI]（开关 + Shader Keyword）
│   ├── [Enum]（下拉菜单 → Shader Keyword）
│   ├── [PowerSlider(x)]（非线性滑条）
│   ├── [Header("xxx")]（文本标题分隔）
│   ├── [Space(n)]（间距）
│   ├── [NoScaleOffset]（隐藏 Tiling/Offset）
│   └── [HideInInspector]（隐藏不暴露）
├── Level 2: 自定义 ShaderGUI（Per-Material Inspector）
│   ├── 继承 ShaderGUI，重写 OnGUI()
│   ├── MaterialProperty 查找与修改
│   ├── EditorGUI 折页组（Foldout）
│   ├── 条件显示（Keyword 状态控制子属性可见性）
│   ├── 范围保护 + Clamp
│   ├── Undo 支持（ShaderGUI 内置 Undo）
│   └── Preset 按钮保存/加载
├── Level 3: 独立 EditorWindow（批量编辑工具）
│   ├── EditorWindow.GetWindow<T>()
│   ├── Selection.activeObject / GetFiltered<Material>()
│   ├── 多材质同步修改（foreach + SerializedObject）
│   ├── 预设保存 → ScriptableObject MaterialPreset
│   ├── 预设对比（Diff 当前值 vs 预设值）
│   └── 拖拽 Material 到窗口 → 自动读取属性
├── Level 4: UI Toolkit（下一代 Editor UI）
│   ├── UXML（类似 HTML 的结构化布局）
│   ├── USS（类似 CSS 的样式表）
│   ├── C# QueryBuilder（rootVisualElement.Q<Label>("name")）
│   ├── 响应式绑定（SerializedObject → automatically reflect changes）
│   └── IMGUI vs UI Toolkit（旧 vs 新，2021+ 推荐 UI Toolkit）
├── Level 5: Scene View 联动
│   ├── Handles / Gizmos 辅助可视化参数
│   ├── OnSceneGUI 在编辑面板时同步 Scene 预览
│   └── 实时预览参数变化（Shader 的 hot-reload 不需要，材质属性即时生效）
└── 工程化交付
    ├── Editor 程序集隔离（Editor 文件夹 + asmdef）
    ├── 版本控制友好（预设是 .asset 文件）
    ├── 文档：每个参数配 tooltip 说明
    └── 美术验收：5 分钟教会美术使用
```

#### 代码实现

**Level 2: 自定义 ShaderGUI（分组 + 条件显示）：**

```csharp
using UnityEditor;
using UnityEngine;

public class CharacterShaderGUI : ShaderGUI
{
    private bool _showBase = true;
    private bool _showEmission = false;
    private bool _showRim = false;

    public override void OnGUI(MaterialEditor materialEditor, MaterialProperty[] properties)
    {
        Material material = materialEditor.target as Material;

        // ===== 基础属性组 =====
        _showBase = EditorGUILayout.BeginFoldoutHeaderGroup(_showBase, "🎨 基础颜色");
        if (_showBase)
        {
            MaterialProperty baseMap = FindProperty("_BaseMap", properties);
            MaterialProperty baseColor = FindProperty("_BaseColor", properties);
            materialEditor.TexturePropertySingleLine(
                new GUIContent("Base Map", "基础贴图"), baseMap, baseColor);
            materialEditor.TextureScaleOffsetProperty(baseMap);
        }
        EditorGUILayout.EndFoldoutHeaderGroup();

        // ===== Emission 组（条件显示） =====
        MaterialProperty emissionEnabled = FindProperty("_EmissionEnabled", properties);
        EditorGUILayout.PropertyField(emissionEnabled, new GUIContent("✨ 启用发光"));
        if (emissionEnabled.floatValue > 0.5f)
        {
            EditorGUI.indentLevel++;
            _showEmission = EditorGUILayout.BeginFoldoutHeaderGroup(_showEmission, "发光参数");
            if (_showEmission)
            {
                MaterialProperty emisMap = FindProperty("_EmissionMap", properties);
                MaterialProperty emisColor = FindProperty("_EmissionColor", properties);
                materialEditor.TexturePropertySingleLine(
                    new GUIContent("Emission Map"), emisMap, emisColor);
                MaterialProperty emisIntensity = FindProperty("_EmissionIntensity", properties);
                // 范围保护
                emisIntensity.floatValue = Mathf.Max(0, emisIntensity.floatValue);
                materialEditor.ShaderProperty(emisIntensity, "发光强度");
            }
            EditorGUILayout.EndFoldoutHeaderGroup();
            EditorGUI.indentLevel--;
        }

        // ===== Rim Light 组 =====
        _showRim = EditorGUILayout.BeginFoldoutHeaderGroup(_showRim, "💡 边缘光 (Rim)");
        if (_showRim)
        {
            MaterialProperty rimColor = FindProperty("_RimColor", properties);
            MaterialProperty rimPower = FindProperty("_RimPower", properties);
            materialEditor.ShaderProperty(rimColor, "Rim 颜色");
            materialEditor.ShaderProperty(rimPower, "Rim 范围");
        }
        EditorGUILayout.EndFoldoutHeaderGroup();

        // 提示信息
        EditorGUILayout.HelpBox(
            "💡 提示：\n" +
            "- 基础颜色和贴图建议从美术规范模板复制\n" +
            "- 发光强度超过 3.0 可能导致 Bloom 过曝\n" +
            "- Rim 范围建议 2.0~6.0",
            MessageType.Info);
    }
}
```

**Level 3: 批量材质编辑 EditorWindow：**

```csharp
using UnityEditor;
using UnityEngine;

public class MaterialBatchEditor : EditorWindow
{
    private Material _targetMaterial;
    private Vector2 _scrollPos;
    private MaterialPreset _currentPreset;

    [MenuItem("TA Tools/Material Batch Editor")]
    public static void Open() => GetWindow<MaterialBatchEditor>("材质批量编辑器");

    void OnGUI()
    {
        _targetMaterial = (Material)EditorGUILayout.ObjectField(
            "目标材质", _targetMaterial, typeof(Material), false);

        if (_targetMaterial == null) return;

        // 预设管理
        EditorGUILayout.Space(10);
        EditorGUILayout.LabelField("📋 预设管理", EditorStyles.boldLabel);
        _currentPreset = (MaterialPreset)EditorGUILayout.ObjectField(
            "当前预设", _currentPreset, typeof(MaterialPreset), false);

        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("保存当前参数为预设"))
            SavePreset();
        if (GUILayout.Button("应用预设到选中材质"))
            ApplyPresetToSelection();
        EditorGUILayout.EndHorizontal();

        // 批量操作
        EditorGUILayout.Space(10);
        EditorGUILayout.LabelField("🔧 批量操作", EditorStyles.boldLabel);
        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("应用预设到场景所有同 Shader 材质"))
            ApplyPresetToAllSameShader();
        if (GUILayout.Button("导出预设为 JSON"))
            ExportPresetJSON();
        EditorGUILayout.EndHorizontal();

        // 属性对比
        if (_currentPreset != null)
        {
            EditorGUILayout.Space(10);
            _scrollPos = EditorGUILayout.BeginScrollView(_scrollPos);
            DrawDiffTable();
            EditorGUILayout.EndScrollView();
        }
    }

    void SavePreset()
    {
        var preset = ScriptableObject.CreateInstance<MaterialPreset>();
        preset.shaderName = _targetMaterial.shader.name;
        preset.floatValues = new System.Collections.Generic.List<MaterialPreset.FloatEntry>();
        preset.colorValues = new System.Collections.Generic.List<MaterialPreset.ColorEntry>();

        // 序列化材质属性
        var so = new SerializedObject(_targetMaterial);
        var props = so.FindProperty("m_SavedProperties.m_Floats");
        for (int i = 0; i < props.arraySize; i++)
        {
            var prop = props.GetArrayElementAtIndex(i);
            preset.floatValues.Add(new MaterialPreset.FloatEntry
            {
                name = prop.FindPropertyRelative("first").stringValue,
                value = prop.FindPropertyRelative("second").floatValue
            });
        }
        // ... 类似处理 Colors 和 Textures

        AssetDatabase.CreateAsset(preset, $"Assets/MaterialPresets/{_targetMaterial.name}_Preset.asset");
        AssetDatabase.SaveAssets();
        Debug.Log($"预设已保存: {preset.name}");
    }

    void ApplyPresetToSelection()
    {
        var selected = Selection.GetFiltered<Material>(SelectionMode.Assets);
        foreach (var mat in selected)
        {
            Undo.RecordObject(mat, "Apply Preset");
            // ... 应用预设值
            EditorUtility.SetDirty(mat);
        }
        AssetDatabase.SaveAssets();
    }

    void ApplyPresetToAllSameShader()
    {
        string targetShader = _targetMaterial.shader.name;
        var allMaterials = AssetDatabase.FindAssets("t:Material");
        int count = 0;
        foreach (var guid in allMaterials)
        {
            var path = AssetDatabase.GUIDToAssetPath(guid);
            var mat = AssetDatabase.LoadAssetAtPath<Material>(path);
            if (mat != null && mat.shader.name == targetShader)
            {
                Undo.RecordObject(mat, "Batch Apply Preset");
                // ... 应用预设值
                EditorUtility.SetDirty(mat);
                count++;
            }
        }
        AssetDatabase.SaveAssets();
        Debug.Log($"已应用到 {count} 个同 Shader 材质");
    }

    void DrawDiffTable() { /* 对比表格绘制 */ }
    void ExportPresetJSON() { /* JSON 导出 */ }
}
```

**预设 ScriptableObject 数据结构：**

```csharp
using UnityEngine;
using System.Collections.Generic;

[System.Serializable]
public class MaterialPreset : ScriptableObject
{
    public string shaderName;
    public string description;
    public List<FloatEntry> floatValues;
    public List<ColorEntry> colorValues;
    public List<TextureEntry> textureValues;

    [System.Serializable] public struct FloatEntry   { public string name; public float value; }
    [System.Serializable] public struct ColorEntry   { public string name; public Color value; }
    [System.Serializable] public struct TextureEntry { public string name; public Texture value; }
}
```

**方案对比：**

| 方案 | 开发成本 | 美术体验 | 维护成本 | 推荐度 |
|------|----------|----------|----------|--------|
| ShaderLab 属性标记 | ⭐ | 一般（比默认好） | 低 | 入门必做 |
| 自定义 ShaderGUI | ⭐⭐ | 好（分组+说明） | 中 | 中型项目推荐 |
| EditorWindow 批量工具 | ⭐⭐⭐ | 优秀（批量+预设） | 中高 | 大型项目推荐 |
| UI Toolkit 重写 | ⭐⭐⭐⭐ | 最佳（响应式+可扩展） | 高 | 新项目/长期维护 |

### ⚡ 实战经验

- **从 ShaderGUI 开始**：80% 的改善只需要一个 ShaderGUI——分组 + 中文 Label + HelpBox 提示，美术立刻觉得专业
- **预设系统的价值**：美术调好一个"标准角色皮肤"材质后保存预设，后续所有角色一键应用——减少重复劳动 90%
- **UI Toolkit 是趋势**：Unity 6+ 强烈推荐 UI Toolkit 做 Editor 工具，IMGUI 逐步被边缘化。但短期内 IMGUI 仍然是最快出活的方案
- **Tooltip 不能省**：每个参数都加 tooltip 说明含义和建议范围——减少美术提问 50%
- **版本兼容陷阱**：ShaderGUI 需要适配多个 Shader 版本——用 `material.HasProperty()` 和 `material.IsKeywordEnabled()` 做防御性检查，避免老材质报错
- **真实案例**：某项目引入自定义 ShaderGUI + 预设系统后，美术调材质的平均时间从 15 分钟降到 3 分钟，Shader 相关的问题咨询减少了 70%

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么改 Material Inspector | ShaderGUI 机制 | 学 UnityEditor.ShaderGUI 基类和 API |
| 分组写出来了但条件显示不会 | MaterialProperty + Keyword | 学 material.GetFloat / IsKeywordEnabled |
| 批量编辑做不出 Undo | Undo 系统 | 学 Undo.RecordObject / RegisterCompleteObjectUndo |
| EditorWindow 打开了但不知道怎么选材质 | Selection API | 学 Selection.GetFiltered / activeObject |
| 想用 UI Toolkit 但不会 | UXML / USS | 学 UIElements 基础 + QueryBuilder |

### 🔗 相关问题

- 如何让美术在 Scene 视图中直接拖拽调整 Shader 参数？（提示：OnSceneGUI + Handles）
- 材质预设如何做版本控制？不同 Shader 版本的预设兼容性怎么处理？
- 如何实现一个通用的"任意 Shader 自动生成 Inspector"工具？（提示：ShaderUtil.GetPropertyCount + 动态绘制）
