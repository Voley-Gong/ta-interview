---
title: "手游UI卡顿：Canvas重建与UGUI性能优化实战"
category: "optimization"
level: 3
tags: ["UGUI", "Canvas", "UI性能", "Rebuild", "Batching", "DOTS"]
hint: "UI卡顿的元凶往往不是GPU——Canvas的 Vertex Rebuild 和 SetPass 切断才是CPU侧的隐形杀手"
related: ["optimization/drawcall-500-to-100", "optimization/mobile-package-size-2gb-to-500mb", "soft-skills/art-quality-vs-performance-tradeoff"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的一款二次元手游，战斗中 UI 帧率不稳定。Profiler 显示每帧 CPU 耗时 12ms，其中 UI Rendering 占了 5ms。用 Frame Debugger 发现 Canvas Batch 数量多达 80+。你接手后怎么优化？」

这是网易、米哈游等做重度 UI 手游的常见面试题。UGUI 看起来简单，但性能陷阱极深——考察的是对 UGUI 源码级机制的理解。

### ✅ 核心要点

1. **Canvas 分割策略**：动静分离——频繁变化的元素（血条、伤害数字）单独一个 Canvas
2. **Rebuild 成本控制**：避免每帧 `SetVerticesDirty` / `SetLayoutDirty`，用 `Canvas.willRenderCanvases` 理解重建流程
3. **Batch 中断排查**：Texture 不同、Material 不同、层级穿插都会打断合批
4. **Overdraw 削减**：不可见的 UI 元素要 Disable 而不是设 Alpha=0
5. **TextMeshPro 优化**：动态字体 Atlas 溢出、字图集预生成

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：UI CPU 耗时 5ms → 1.5ms，Batch 数 80 → 20
     ↓
Step 1：Profile 定位——到底是 Rebuild 还是 Batch 中断？
  ├── Profiler > UI > Canvas.SendWillRenderCanvases（Rebuild 耗时）
  ├── Profiler > UI > Canvas.BuildBatch（合批耗时）
  └── Frame Debugger > 检查每批被打断的原因
     ↓
Step 2：Rebuild 优化
  ├── 哪些元素在每帧 dirty？（血条 Slider、Text 内容更新）
  └── 把高频更新元素移到独立子 Canvas
     ↓
Step 3：Batch 优化
  ├── 合并图集（角色头像、装备图标、技能图标分别打 Atlas）
  ├── 消除层级穿插（同一图集的元素要连续排列）
  └── 减少 Material 切换（TMP 字体统一）
     ↓
Step 4：Overdraw 优化
  ├── 全屏半透明背景面板 → 用不透明 + Mask
  └── 隐藏 UI 用 Canvas.enabled = false（不要只设 Alpha）
```

#### 知识点拆解（倒推树）

```
UGUI 性能优化
├── Canvas 机制理解
│   ├── Canvas 的渲染流程
│   │   ├── willRenderCanvases 事件（每帧检查 Dirty）
│   │   ├── Vertex Rebuild（重新生成网格）
│   │   └── Batch Build（合并网格提交 GPU）
│   ├── Sub-Canvas 分割原则
│   │   ├── 静态层（背景、边框、固定文字）
│   │   ├── 动态层（血条、计时器、飘字）
│   │   └── 弹窗层（独立渲染顺序）
│   └── Rendering Mode 选择
│       ├── Screen Space - Overlay（最贵，始终在最上）
│       ├── Screen Space - Camera（推荐，可控制渲染顺序）
│       └── World Space（3D UI，需注意 LOD）
├── Rebuild 优化
│   ├── Dirty Flag 机制
│   │   ├── SetVerticesDirty → 触发顶点重建
│   │   ├── SetLayoutDirty → 触发布局重建（更贵！）
│   │   └── SetMaterialDirty → 触发材质更新
│   ├── 高频更新优化
│   │   ├── 血条用 Image.fillAmount 而非 Scale
│   │   ├── Text 用 SetText 而非 string 拼接
│   │   └── 列表用对象池 + SetActive 复用
│   └── Layout Group 避免
│       ├── Vertical/Horizontal/Grid LayoutGroup 每帧重建
│       └── 替代方案：手动计算位置 + 锚点
├── Batch 优化
│   ├── 合批条件
│   │   ├── 同一张 Texture（图集）
│   │   ├── 同一个 Material
│   │   ├── 层级连续（无穿插）
│   │   └── 无 Mask/RectMask2D 隔断
│   ├── 图集策略
│   │   ├── Sprite Atlas（Unity 内置）
│   │   ├── 图集分类：UI Common / Role Icon / Item / Skill
│   │   └── 动态加载图集的 IncludeInBuild 控制
│   └── Mask 优化
│       ├── RectMask2D（软裁剪，会打断 Batch）
│       ├── Mask（Stencil 裁剪，更贵）
│       └── 替代方案：UV 裁剪在 Shader 中实现
├── TextMeshPro 优化
│   ├── 动态字体 Atlas
│   │   ├── Atlas 溢出 → 重新生成全量 Atlas（卡顿！）
│   │   ├── 预生成常用字集（CommonChars.txt）
│   │   └── 多 Atlas 分组（常用字 / 稀有字）
│   └── TMPro vs Text
│       ├── TMPro 网格更复杂（SDF 描边/阴影）
│       └── 静态文字可 Freeze（禁用 Rebuild）
└── 进阶方案
    ├── DOTS UI（ECS 化 UI，帧率稳定但生态不成熟）
    ├── 自研 UI 框架（对 Mesh 合批完全可控）
    └── GPU Driven UI（Compute Shader 构建 Mesh）
```

#### 代码实现

**Canvas 分割工具（自动检测高频 dirty 元素）：**

```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Collections.Generic;

#if UNITY_EDITOR
using UnityEditor;

public class UICanvasAnalyzer : EditorWindow
{
    [MenuItem("Tools/TA/UI Canvas Analyzer")]
    static void Open() => GetWindow<UICanvasAnalyzer>("UI Canvas Analyzer");

    void OnGUI()
    {
        if (GUILayout.Button("分析选中 Canvas 的 Batch 结构"))
        {
            AnalyzeCanvas(Selection.activeGameObject?.GetComponent<Canvas>());
        }
        if (GUILayout.Button("查找高频 Dirty 元素（需要正在运行）"))
        {
            FindDirtyElements();
        }
    }

    void AnalyzeCanvas(Canvas root)
    {
        if (root == null) { Debug.LogError("请选中带 Canvas 的对象"); return; }

        var renderers = root.GetComponentsInChildren<Graphic>(true);
        var texGroups = new Dictionary<Texture, List<Graphic>>();

        foreach (var g in renderers)
        {
            if (g.mainTexture == null) continue;
            if (!texGroups.ContainsKey(g.mainTexture))
                texGroups[g.mainTexture] = new List<Graphic>();
            texGroups[g.mainTexture].Add(g);
        }

        Debug.Log($"[Canvas分析] {root.name}");
        Debug.Log($"  Graphic 总数: {renderers.Length}");
        Debug.Log($"  Texture 分组数: {texGroups.Count}（每个分组内部可合批）");

        foreach (var kv in texGroups)
        {
            // 检查同纹理的元素是否被其他纹理的元素穿插
            var elements = kv.Value;
            int minSibling = int.MaxValue, maxSibling = 0;
            foreach (var e in elements)
            {
                minSibling = Mathf.Min(minSibling, e.transform.GetSiblingIndex());
                maxSibling = Mathf.Max(maxSibling, e.transform.GetSiblingIndex());
            }
            bool hasInterleave = false;
            foreach (var g in renderers)
            {
                if (g.mainTexture != kv.Key)
                {
                    int idx = g.transform.GetSiblingIndex();
                    if (idx > minSibling && idx < maxSibling)
                    {
                        hasInterleave = true;
                        Debug.LogWarning($"  ⚠️ 穿插: {g.name} (tex={g.mainTexture?.name}) 穿插了 {kv.Key.name} 的范围 [{minSibling}~{maxSibling}]");
                        break;
                    }
                }
            }
            if (!hasInterleave)
                Debug.Log($"  ✅ {kv.Key.name}: {elements.Count} 个元素，无穿插");
        }
    }

    void FindDirtyElements()
    {
        // 利用 CanvasUpdateRegistry 监控
        var canvas = FindObjectOfType<Canvas>();
        if (canvas == null) return;

        // 遍历所有 Graphic，检查 dirty 标志
        var graphics = FindObjectsOfType<Graphic>();
        var dirtyList = new List<string>();

        foreach (var g in graphics)
        {
            // 检查是否在当前帧被标记为 dirty
            // 通过反射访问内部 dirty 标志
            var type = typeof(Graphic);
            var verticesDirty = type.GetField("m_VerticesDirty", 
                System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);
            var layoutDirty = type.GetField("m_LayoutDirty",
                System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);

            bool vDirty = verticesDirty != null && (bool)verticesDirty.GetValue(g);
            bool lDirty = layoutDirty != null && (bool)layoutDirty.GetValue(g);

            if (vDirty || lDirty)
            {
                dirtyList.Add($"  {GetPath(g.transform)}: VertDirty={vDirty} LayoutDirty={lDirty}");
            }
        }

        if (dirtyList.Count > 0)
        {
            Debug.Log($"[Dirty 检测] {dirtyList.Count} 个元素处于 Dirty 状态：");
            dirtyList.ForEach(System.Action<string>.Invoke);
        }
        else
        {
            Debug.Log("[Dirty 检测] 当前无 Dirty 元素");
        }
    }

    string GetPath(Transform t)
    {
        string path = t.name;
        while (t.parent != null)
        {
            t = t.parent;
            path = t.name + "/" + path;
        }
        return path;
    }
}
#endif
```

**血条优化对比（错误 vs 正确）：**

```csharp
// ❌ 错误做法：用 Scale 改血条 → 每帧 SetVerticesDirty + 布局重建
public class BadHealthBar : MonoBehaviour
{
    public RectTransform fill;
    public void SetHealth(float ratio)
    {
        fill.localScale = new Vector3(ratio, 1, 1); // 触发 Layout 重建！
    }
}

// ✅ 正确做法 1：用 Image.fillAmount → 只更新 UV，不触发 Layout
public class GoodHealthBar : MonoBehaviour
{
    public Image fill;
    public void SetHealth(float ratio)
    {
        fill.fillAmount = ratio; // 只改 UV，轻量
    }
}

// ✅ 正确做法 2：用 MaterialPropertyBlock 直接传值 → 完全绕过 Rebuild
public class BestHealthBar : MonoBehaviour
{
    public Image fill;
    private MaterialPropertyBlock _mpb;
    private static readonly int FillID = Shader.PropertyToID("_FillAmount");

    void Awake()
    {
        _mpb = new MaterialPropertyBlock();
        // 给 Image 使用支持 _FillAmount 的自定义 Shader
    }

    public void SetHealth(float ratio)
    {
        // 完全不触发 UGUI 的 Rebuild 流程
        fill.SetPropertyBlock(_mpb);
        _mpb.SetFloat(FillID, ratio);
    }
}
```

**优化方案对比表：**

| 优化手段 | 效果 | 实施难度 | 风险 | 适用场景 |
|----------|------|----------|------|----------|
| Canvas 动静分离 | Rebuild 降 60% | 低 | 低 | 所有项目 |
| 去除 LayoutGroup | Layout 耗时降 80% | 中 | 中（需手动布局） | 列表/背包 UI |
| 图集合并 | Batch 数降 50% | 低 | 低 | 图标类 UI |
| RectMask→UV裁剪 | Batch 数降 30% | 高 | 中 | 滚动列表 |
| TMP 字集预生成 | Atlas 重建降 95% | 中 | 低 | 中文 UI |
| GPU Driven UI | CPU UI 耗时降 90% | 极高 | 高 | 旗舰项目 |

### ⚡ 实战经验

- **80/20 法则**：Canvas 分割 + 图集合并通常能解决 80% 的 UI 性能问题，不要一上来就考虑自研框架
- **血条优化**：同屏 100 个血条用 Scale 方式 = 100 次顶点重建，换成 fillAmount 几乎零开销
- **ScrollRect 列表**：不要给每行单独的 LayoutGroup，用 TMPro + 手动锚点排列 + 对象池
- **隐性 Batch 杀手**：组件的 disabled 不会阻断 Batch，但 `CanvasRenderer.cull = true` 的元素不会被合批（它虽然不渲染但占层级位）
- **Overlay 模式特殊处理**：Screen Space - Overlay 的 UI 不受光照影响但始终在最后渲染，Camera 模式更适合 3D UI 混排场景

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 Canvas 为什么会 Rebuild | UGUI 源码：CanvasUpdateRegistry | 读 UGUI 源码的 BuildBatch 流程 |
| Batch 数降不下来 | 合批打断条件 | Frame Debugger 逐批排查打断原因 |
| LayoutGroup 性能差 | ILayoutElement 的 CalcSizes | 理解 LayoutGroup 的两次 Pass（Calc + Set） |
| TMP 字体卡顿 | Dynamic Atlas 溢出机制 | 学习 TMP Atlas Generation 设置 |
| 不知道 Overdraw 有多严重 | GPU Overdraw 可视化 | Scene View > Overdraw 模式 + Render Doc |

### 🔗 相关问题

- 如何用 XLua / Puerts 驱动 UGUI 实现热更新的 UI 系统？
- DOTS 模式下 UI 怎么做？有什么成熟的 ECS UI 方案？
- 如果 UI Draw Call 不是瓶颈但 GPU 仍然高，可能是 Overdraw——怎么检测和优化？
