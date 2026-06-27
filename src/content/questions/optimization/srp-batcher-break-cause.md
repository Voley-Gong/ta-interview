---
title: "SRP Batcher 被神秘打断：Frame Debugger 每隔几个 Draw Call 就 Break，怎么排查？"
category: "optimization"
level: 3
tags: ["SRP Batcher", "URP", "性能优化", "Frame Debugger", "Draw Call", "Material"]
hint: "SRP Batcher 打断的三大元凶：Material Property Block（会打破 CBUFFER）、Shader 不兼容、渲染队列跳跃"
related: ["optimization/drawcall-500-to-100", "optimization/shader-variant-explosion", "technical-art/shader-template-system"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的 URP 项目做了 SRP Batcher 优化，理论上同材质的物体应该一次性提交。但开 Frame Debugger 一看，每 3-4 个 Draw Call 就出现一次 'SRP Batcher Break'，实际批次远远高于预期。项目里有角色、场景建筑、植被、UI，排查一下为什么 Batcher 不断被打断，给我一套系统化的排查流程。」

补充信息：
- 角色用了 MaterialPropertyBlock 做换色
- 植被用了 GPU Instancing
- 部分 Shader 是外包写的，不确定是否兼容 SRP Batcher
- 场景中有透明物体和不透明物体混排
- 目标是移动端，每个 Batcher Break 都是性能损失

### ✅ 核心要点

1. **SRP Batcher 的前提条件**：相同 Shader（不是相同 Material）、Shader 兼容 CBUFFER、渲染队列连续、不使用 MaterialPropertyBlock
2. **MaterialPropertyBlock 是头号杀手**：MPB 会改变 Per-Material CBUFFER 内容，直接打断 Batcher
3. **Shader 兼容性检查**：所有 Shader 必须使用 `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END` 包裹材质参数
4. **渲染状态变化导致 Break**：透明物体（Queue=Transparent）和不透明物体（Queue=Geometry）天然不能 Batch
5. **GPU Instancing 和 SRP Batcher 互斥**：开了 Instancing 的材质不走 SRP Batcher（有优先级规则）
6. **排查工具链**：Frame Debugger → SRP Batcher 兼容性检查 → Frame Timing → Profiler

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：消除所有不合理的 SRP Batcher Break
                ↑
Break 的原因有哪些？
  → A. Material Property Block（Per-Material 数据被修改）
  → B. Shader 不兼容（缺少 CBUFFER 或用了不支持的特性）
  → C. 渲染队列跳跃（Geometry → Transparent → Geometry）
  → D. GPU Instancing 冲突（Instancing 优先于 SRP Batcher）
  → E. RenderType / Layer 变化
  → F. 深度写入状态变化（ZWrite On/Off 切换）
                ↑
排查顺序（从最容易查到最隐蔽）：
  1. Frame Debugger 看每个 Break 的原因文字
  2. 检查所有 Material 是否勾选了 Enable GPU Instancing（关掉！除非确实需要）
  3. 检查所有 Shader 的 CBUFFER 兼容性
  4. 找出所有使用 MaterialPropertyBlock 的代码
  5. 检查渲染队列排序
```

#### 知识点拆解（倒推树）

```
SRP Batcher Break 排查
├── Break 原因分类
│   ├── A. Node Use Material Property Block（最常见）
│   │   ├── MPB 修改了 Per-Material CBUFFER
│   │   ├── 常见来源：角色换色、血条颜色、闪烁效果
│   │   ├── SRP Batcher 要求 Per-Material CBUFFER 不可在运行时修改
│   │   └── 解决：改用 CBUFFER 内的 instanced 属性 或拆分 Material
│   ├── B. Shader Not Compatible
│   │   ├── 缺少 CBUFFER_START(UnityPerMaterial)
│   │   ├── 使用了不被 SRP Batcher 支持的 Shader Feature
│   │   ├── Shader 中使用了 `uniform` 声明在 CBUFFER 外部
│   │   ├── 多 Pass Shader（每个 Pass 可能独立 Batch）
│   │   └── 解决：按 URP 规范重写 Shader
│   ├── C. Render Queue Jump
│   │   ├── Queue=Geometry(2000) 和 Queue=Transparent(3000) 交替
│   │   ├── 同 Queue 内但 ZTest/ZWrite 设置不同
│   │   └── 解决：按 Queue 排序渲染，同 Queue 内确保 Z 状态一致
│   ├── D. GPU Instancing Override
│   │   ├── Material 勾选了 Enable GPU Instancing
│   │   ├── URP 的优先级：GPU Instancing > SRP Batcher（当物体数量多时）
│   │   ├── 实际上 URP 会自动选择更优方案，但会造成 Batcher 不稳定
│   │   └── 解决：同类型物体要么全走 Instancing，要么全走 Batcher
│   ├── E. Cross Render Pass
│   │   ├── 不透明 Pass 和透明 Pass 是不同 Pass，天然 Break
│   │   ├── Shadow Pass 和 Main Pass 也是独立 Batch
│   │   ├── Renderer Feature 注入的额外 Pass 会打断
│   │   └── 解决：这是正常的，关注同 Pass 内的 Break
│   └── F. Stencil / Blend State Change
│       ├── Stencil Ref 值不同的 Material 不能 Batch
│       ├── Blend Mode（SrcAlpha / OneMinusSrcAlpha vs Additive）
│       └── 解决：分组排序，同状态连续渲染
│
├── 排查工具链
│   ├── Frame Debugger
│   │   ├── 逐 Draw Call 查看 Break 原因
│   │   ├── 关注 "Reason for break" 提示文字
│   │   └── 常见提示：
│   │       ├── "Node use Material Property Block"
│   │       ├── "Material has different CBUFFER"
│   │       ├── "Different Shader"
│   │       └── "Different Render Queue"
│   ├── SRP Batcher Compatibility Report
│   │   ├── Window > Analysis > SRP Batcher Information
│   │   ├── 可以看到哪些 Shader 不兼容及原因
│   │   └── 编译时会警告
│   ├── Profiler
│   │   ├── CPU Timeline 看 Render Loop 耗时
│   │   ├── Camera.Render → SetupCamera → RenderLoop.Draw
│   │   └── SRP Batcher Save/Restore 开销
│   └── 自定义统计脚本
│       ├── 遍历所有 Material 检查 Instancing 开关
│       ├── 检查所有 Shader 的 CBUFFER 声明
│       └── 统计运行时 MPB 使用情况
│
├── MPB 替代方案
│   ├── 方案 A：拆分 Material（最直接）
│   │   ├── 预创建 N 个 Material（如 10 种颜色）
│   │   ├── 运行时切换 Material 而非 MPB
│   │   ├── 优点：完全兼容 SRP Batcher
│   │   ├── 缺点：Material 数量增加，Draw Call 可能不降反升
│   │   └── 适用：颜色/外观种类有限的情况
│   ├── 方案 B：CBUFFER Instanced Properties
│   │   ├── 用 `UNITY_ACCESS_INSTANCED_PROP` 读取 Per-Instance 数据
│   │   ├── 需要 #pragma multi_compile_instancing
│   │   ├── 此时走 GPU Instancing 而非 SRP Batcher
│   │   ├── 优点：大量同 Mesh 物体性能最好
│   │   ├── 缺点：要求同 Mesh
│   │   └── 适用：植被、道具、重复物体
│   ├── 方案 C：Texture Lookup（推荐）
│   │   ├── 用一张小 Texture 存储所有角色的颜色数据
│   │   ├── 每个 Material 设置一个 UV offset 指向自己的颜色
│   │   ├── Material 本身完全相同 → Batcher 兼容
│   │   ├── 优点：无 MPB、无 Instancing 冲突、无限颜色组合
│   │   └── 适用：角色换色、装备外观
│   └── 方案 D：StructuredBuffer（高级）
│       ├── Per-Instance 数据存入 StructuredBuffer
│       ├── 通过 SV_InstanceID 索引
│       └── 需要 Compute Buffer 支持
│
└── URP SRP Batcher 工作原理
    ├── 传统渲染：每个 Draw Call 设置一次 Material 参数到 GPU
    ├── SRP Batcher：相同 Shader 的物体共享 CBUFFER 设置
    │   ├── Per-Camera CBUFFER（视图矩阵等）设一次
    │   ├── Per-Material CBUFFER 在 Shader 级别缓存
    │   └── 只需更新 Per-Object 的 World Matrix
    └── 性能提升来源
        ├── 减少 CPU 端的 SetPassCall / Material Setup
        ├── 减少 GPU 状态切换
        └── 移动端尤其明显（状态切换开销大）
```

#### 代码实现

**SRP Batcher 兼容 Shader 模板（URP）：**

```hlsl
Shader "Custom/SRPBatcherCompatible"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1,1,1,1)
        _Smoothness ("Smoothness", Range(0,1)) = 0.5
        // 注意：所有 Per-Material 参数必须在 CBUFFER 内
    }

    SubShader
    {
        Tags
        {
            "RenderType" = "Opaque"
            "RenderPipeline" = "UniversalPipeline"
            "Queue" = "Geometry"  // 统一 Queue
        }

        Pass
        {
            Name "ForwardLit"  // 有名字的 Pass 更容易被 Batcher 识别
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // ❌ 不要在这里加 multi_compile_instancing，除非确实需要 GPU Instancing
            // #pragma multi_compile_instancing  // 和 SRP Batcher 冲突

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            // ✅ SRP Batcher 要求：所有材质参数必须在 UnityPerMaterial CBUFFER 内
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                half4  _BaseColor;
                half   _Smoothness;
            CBUFFER_END

            // ✅ 纹理声明在 CBUFFER 外部（纹理对象不占 CBUFFER 空间）
            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv          : TEXCOORD0;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                half4 col = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv) * _BaseColor;
                return col;
            }
            ENDHLSL
        }
    }
}
```

**Texture Lookup 换色方案（替代 MPB）：**

```csharp
using UnityEngine;

/// <summary>
/// 用 Texture Lookup 替代 MaterialPropertyBlock，保持 SRP Batcher 兼容
/// 所有角色共用同一个 Material，通过 UV offset 区分颜色
/// </summary>
[RequireComponent(typeof(Renderer))]
public class TextureLookupColorSetter : MonoBehaviour
{
    [Header("颜色查找表")]
    [SerializeField] private Texture2D _colorLookupTexture; // 64x1 的小贴图，存64种颜色
    [SerializeField] private int _colorIndex = 0;           // 当前颜色索引

    // 所有角色共享的 Material（不要 new，用 Asset 共享）
    private static Material _sharedColorMaterial;

    private Renderer _renderer;

    void Start()
    {
        _renderer = GetComponent<Renderer>();

        // 所有角色用同一个 Material 实例
        if (_sharedColorMaterial == null)
        {
            _sharedColorMaterial = new Material(Shader.Find("Custom/TextureLookupColor"));
            _sharedColorMaterial.SetTexture("_ColorLUT", _colorLookupTexture);
        }
        _renderer.sharedMaterial = _sharedColorMaterial; // 用 sharedMaterial，不是 material!

        SetColor(_colorIndex);
    }

    /// <summary>
    /// 通过 UV offset 切换颜色，不使用 MPB
    /// </summary>
    public void SetColor(int index)
    {
        _colorIndex = index;
        float texWidth = _colorLookupTexture.width;
        float offsetX = (float)index / texWidth;

        // ❌ 错误做法：会创建 Material 实例，打断 Batcher
        // _renderer.material.SetTextureOffset("_MainTex", new Vector2(offsetX, 0));

        // ✅ 正确做法：所有角色共享 Material 的 offset 固定
        // 改为在 Vertex Shader 中用 InstanceID 索引
        // 或者：如果颜色种类有限，预创建多个 Material（每个一个 offset）
    }
}

/// <summary>
/// 方案 B：预创建 Material 池（颜色种类有限时最简单）
/// </summary>
public class MaterialPoolColorSetter : MonoBehaviour
{
    [SerializeField] private Material[] _colorMaterials; // 预创建的 Material 数组
    [SerializeField] private int _colorIndex = 0;

    private Renderer _renderer;

    void Start()
    {
        _renderer = GetComponent<Renderer>();
        UpdateColor();
    }

    public void SetColor(int index)
    {
        _colorIndex = index;
        UpdateColor();
    }

    void UpdateColor()
    {
        // 切换 sharedMaterial，不创建实例 → SRP Batcher 友好
        // 相同 Material 的物体会被 Batcher 合并
        if (_colorIndex >= 0 && _colorIndex < _colorMaterials.Length)
        {
            _renderer.sharedMaterial = _colorMaterials[_colorIndex];
        }
    }
}
```

**自动化排查脚本（Editor Tool）：**

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using System.Collections.Generic;
using System.Text;

public class SRPBatcherAuditor : EditorWindow
{
    [MenuItem("Tools/TA/SRP Batcher Auditor")]
    static void ShowWindow() => GetWindow<SRPBatcherAuditor>();

    void OnGUI()
    {
        if (GUILayout.Button("Audit All Materials (GPU Instancing Check)"))
            AuditMaterialsInstancing();

        if (GUILayout.Button("Audit All Shaders (CBUFFER Compatibility)"))
            AuditShadersCBUFFER();

        if (GUILayout.Button("Find All MaterialPropertyBlock Usage"))
            FindMPBUsage();

        if (GUILayout.Button("Generate Full Report"))
            GenerateFullReport();
    }

    /// <summary>
    /// 找到所有勾选了 GPU Instancing 的 Material
    /// </summary>
    void AuditMaterialsInstancing()
    {
        var guids = AssetDatabase.FindAssets("t:Material");
        var offenders = new List<string>();

        foreach (var guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            var mat = AssetDatabase.LoadAssetAtPath<Material>(path);

            if (mat != null && mat.enableInstancing)
            {
                offenders.Add($"  ⚠️ {path} (GPU Instancing = ON)");
            }
        }

        Debug.Log($"[SRP Batcher Audit] GPU Instancing Materials ({offenders.Count}):\n" +
                  string.Join("\n", offenders));
    }

    /// <summary>
    /// 检查 Shader 是否包含 UnityPerMaterial CBUFFER
    /// </summary>
    void AuditShadersCBUFFER()
    {
        var guids = AssetDatabase.FindAssets("t:Shader");
        var offenders = new List<string>();

        foreach (var guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            var shader = Shader.Find(AssetDatabase.LoadAssetAtPath<Shader>(path)?.name);
            if (shader == null) continue;

            // 读取 Shader 源码检查 CBUFFER
            string shaderPath = AssetDatabase.GetAssetPath(shader);
            string source = System.IO.File.ReadAllText(shaderPath);

            bool hasCBUFFER = source.Contains("CBUFFER_START(UnityPerMaterial)");
            bool hasProperties = shader.GetPropertyCount() > 0;

            if (hasProperties && !hasCBUFFER)
            {
                offenders.Add($"  ❌ {shaderPath} — 缺少 CBUFFER_START(UnityPerMaterial)");
            }
        }

        Debug.Log($"[SRP Batcher Audit] Incompatible Shaders ({offenders.Count}):\n" +
                  string.Join("\n", offenders));
    }

    /// <summary>
    /// 在场景中搜索使用了 MPB 的 Renderer
    /// </summary>
    void FindMPBUsage()
    {
        var renderers = FindObjectsByType<Renderer>(FindObjectsSortMode.None);
        var offenders = new List<string>();

        foreach (var r in renderers)
        {
            var mpb = new MaterialPropertyBlock();
            r.GetPropertyBlock(mpb);

            if (mpb.GetFloat("_CustomValue", -999f) != -999f ||
                mpb.GetColor("_Color", Color.clear) != Color.clear ||
                mpb.GetVector("_CustomVector", Vector4.zero) != Vector4.zero)
            {
                offenders.Add($"  🔴 {r.gameObject.name} ({r.GetType().Name}) — 使用了 MaterialPropertyBlock");
            }
        }

        Debug.Log($"[SRP Batcher Audit] MPB Users ({offenders.Count}):\n" +
                  string.Join("\n", offenders));
    }

    void GenerateFullReport()
    {
        var sb = new StringBuilder();
        sb.AppendLine("=== SRP Batcher Full Audit Report ===");
        sb.AppendLine($"Date: {System.DateTime.Now}");
        sb.AppendLine();

        // 三个检查都跑一遍
        // ...（调用上面的方法并汇总）
        Debug.Log(sb.ToString());
    }
}
#endif
```

**Frame Debugger Break 原因速查表：**

| Frame Debugger 提示 | 含义 | 解决方案 |
|---------------------|------|----------|
| "Node use Material Property Block" | 使用了 MPB | 改用 Material 池或 Texture Lookup |
| "Material has different keywords" | Shader Keyword 组合不同 | 统一 Keyword 或按 Keyword 分组排序 |
| "Different Shader" | 根本不是同一个 Shader | 按 Shader 分组渲染 |
| "Different Render Queue" | 渲染队列不同 | 检查 Material 的 Queue 设置 |
| "Not SRP batcher compatible shader" | Shader 缺少 CBUFFER | 添加 `CBUFFER_START(UnityPerMaterial)` |
| "Render Pass Change" | 跨渲染 Pass | 正常现象（不透明→透明→后处理） |
| "GPU Instancing" | 走了 Instancing 而非 Batcher | 关闭 Instancing 或接受 Instancing 路径 |

### ⚡ 实战经验

1. **`renderer.material` 和 `renderer.sharedMaterial` 的陷阱**：`.material` 会创建独立实例（自动 MPB 效果），`.sharedMaterial` 才是共享引用。全项目统一用 `.sharedMaterial`，需要换色用 Material 池
2. **SRP Batcher 和 GPU Instancing 不是非此即彼**：URP 内部有优先级判断——如果物体数量少（< ~20）走 Batcher 更优，物体多走 Instancing 更优。但混用时 Batcher 不稳定，建议同类型物体走同一条路径
3. **Shadow Pass 的 Batcher 是独立计算的**：即使 Shadow Pass 打断了 Batcher，不透明 Pass 仍可以正常 Batch。不要因为 Shadow Pass 的 Break 就放弃优化
4. **URP 的 Sorting 有坑**：默认按距离排序，但同距离的物体顺序不确定。可以在 Renderer 上加 `Sorting Group` 组件强制分组排序
5. **真实案例**：一个项目 400 个 Draw Call，其中 300 次是 SRP Batcher Break。排查后发现是 150 个角色各自用了 MPB 设置血条颜色。改为 Material 池（5 种颜色阶段 × 1 个 Material = 5 个 Material），Draw Call 降到 180
6. **移动端 SRP Batcher 收益更大**：移动 GPU 的状态切换开销比 PC 大得多（TBR 架构），SRP Batcher 减少的不仅是 CPU 开销，还有 GPU Tile 切换

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不理解 SRP Batcher 的工作原理 | CBUFFER / Per-Material 数据管理 | 学 URP SRP Batcher 源码、CBUFFER 机制 |
| 不知道 MPB 会打断 Batcher | MPB 与 CBUFFER 的冲突 | 学 MaterialPropertyBlock 实现原理 |
| Shader 报 Not Compatible | CBUFFER 声明规范 | 按 URP 规范重写 Shader（CBUFFER_START/END） |
| 不知道怎么排查 Break | Frame Debugger 使用 | 逐 Draw Call 查看 Break 原因并分类 |
| 换色需求和 Batcher 冲突 | 替代方案设计 | 学 Material 池 / Texture Lookup / Instanced Property |
| Instancing 和 Batcher 混乱 | URP 渲染路径优先级 | 学 URP 的 Instancing vs Batcher 选择逻辑 |

### 🔗 相关问题

- [Draw Call 从 500 降到 100](../optimization/drawcall-500-to-100.md)：SRP Batcher 是降 Draw Call 开销的手段之一，但不是唯一
- [Shader Variant 爆炸](../optimization/shader-variant-explosion.md)：Shader 变体过多也会导致 Batcher 不稳定（不同 Variant = 不同 Shader）
- [Shader 模板系统](../technical-art/shader-template-system.md)：统一 Shader 规范是 Batcher 友好的基础
- 面试追问：如果一个场景有 1000 个相同 Mesh 的石头，你用 GPU Instancing 还是 SRP Batcher？（提示：Instancing，因为同 Mesh 大量重复时 Instancing 远优于 Batcher）
