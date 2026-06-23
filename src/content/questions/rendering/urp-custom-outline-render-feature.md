---
title: "URP 自定义描边方案：5 种实现路线与 Renderer Feature 实战"
category: "rendering"
level: 3
tags: ["URP", "描边", "Renderer Feature", "后处理", "法线拓展", "Stencil"]
hint: "描边不只是 Rim Light——法线外扩、后处理描边、Stencil 描边各有优劣，选错方案美术会骂你"
related: ["rendering/urp-renderer-feature", "shader/npr-outline-cartoon", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们的项目需要给场景中的"可交互对象"加描边提示——玩家靠近时物体出现金色描边，选中后变红。需要支持任意形状的物体（不只是角色），而且不能影响场景中其他物体。URP 下怎么做？

> 追问：如果同时还需要给角色加卡通渲染的永久描边呢？两套描边如何共存？

这是腾讯、米哈游、鹰角等做 NPR 渲染或游戏交互系统的 TA 常被问到的题——**表面是"描边怎么写"，实际考察的是 URP 渲染管线架构、Renderer Feature 机制、多 Pass 控制能力**。

### ✅ 核心要点

1. **描边方案矩阵**：法线外扩、屏幕后处理（法线/深度边缘检测）、Stencil 描边、 Rim Light 描边、Jump Flood 描边
2. **Renderer Feature 是 URP 的扩展点**：在渲染管线的特定时机插入自定义 Pass
3. **交互描边的核心是"选择性渲染"**：只给特定物体描边，不影响其他物体
4. **多套描边共存**：用 Stencil / Layer / Render Target 分离不同描边类型
5. **性能取舍**：不同方案对 GPU 的开销差异巨大，移动端需谨慎选择

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
需求拆解：
  A. 可交互物体描边（交互触发，任意物体）→ 需要灵活、可切换
  B. 角色永久卡通描边（始终存在）→ 需要稳定、与渲染一体化
                ↑
方案选型倒推：
  交互描边：
    → 需要动态控制（脚本开关）→ 排除 Rim Light（集成在角色 Shader 中，无法控制非角色物体）
    → 需要支持任意形状 → 排除纯法线外扩（需要修改物体 Shader）
    → 最佳选择：Stencil Buffer + 描边 Pass
      ↓
  角色卡通描边：
    → 需要稳定常驻 → 法线外扩最稳定（不依赖屏幕空间分辨率）
    → 或者屏幕后处理描边（对所有角色统一处理）
    → 最佳选择：法线外扩（内置在角色 Shader 中）
      ↓
共存策略：
  → 交互描边用 Renderer Feature + Stencil（屏幕空间）
  → 角色描边用法线外扩（模型空间）
  → 两套走不同管线阶段，互不干扰
```

#### 知识点拆解（倒推树）

```
URP 描边方案
├── 方案1：法线外扩描边（模型空间）
│   ├── 原理：两个 Pass，第一个 Pass 正常渲染，第二个 Pass 沿法线膨胀顶点+纯色
│   ├── 优点：描边宽度稳定（屏幕空间一致性好）、不受分辨率影响
│   ├── 缺点：需要修改物体 Shader（增加描边 Pass）、硬边模型法线问题
│   ├── 关键技术
│   │   ├── 顶点膨胀：`pos.xyz += normal * _OutlineWidth`
│   │   ├── 硬边处理：用 Smoothed Normal（自定义法线属性）或熔接后法线
│   │   ├── 宽度控制：`OutlineWidth * pos.w`（透视投影补偿）
│   │   └── Z-Prevent：`z = z * 0.9999`（避免被本体遮挡或穿透）
│   └── 适用：角色卡通渲染（Genshin/星穹铁道风格）
│
├── 方案2：后处理边缘检测（屏幕空间）
│   ├── 原理：对屏幕的 Normal/Depth/Color 做卷积（Sobel/Roberts/Prewitt）
│   ├── 优点：全屏统一、支持所有物体、无需修改模型 Shader
│   ├── 缺点：线条不够干净、细节处容易断裂、移动端有性能压力
│   ├── 关键技术
│   │   ├── Sobel 算子：3×3 卷积核检测梯度
│   │   ├── 深度+法线联合检测：减少内部边缘误检
│   │   └── 可交互物体标记：用 Stencil / Custom Depth 区分
│   └── 适用：通用描边（物体选中提示、X-Ray 透视描边）
│
├── 方案3：Stencil Buffer 描边（经典方案）
│   ├── 原理
│   │   Step 1：物体正常渲染时写入 Stencil = 1
│   │   Step 2：描边 Pass 渲染放大版的物体，只画 Stencil ≠ 1 的像素
│   │   → 放大版本的边缘正好勾勒出物体轮廓
│   ├── 优点：描边精度高、可控性强
│   ├── 缺点：需要额外的描边 Pass（可以用法线外扩 Pass）、多一个 DC
│   ├── URP 实现
│   │   ├── Renderer Feature：在 AfterRenderingOpaques 时机插入描边 Pass
│   │   ├── 描边 Pass：Cull Front（渲染背面）+ 法线外扩 + Stencil Test
│   │   └── Stencil 清理 Pass：描边后清除 Stencil 避免污染后续渲染
│   └── 适用：需要精确控制的描边（交互提示、选中高亮）
│
├── 方案4：Rim Light / Fresnel 描边（Shader 内置）
│   ├── 原理：`intensity = pow(1 - dot(N, V), power)` → 边缘叠加颜色
│   ├── 优点：零额外 Pass、性能最好
│   ├── 缺点：描边方向不可控（只能沿边缘亮）、无法做粗描边
│   └── 适用：简单的边缘高亮效果
│
├── 方案5：Jump Flood Algorithm 描边（高级）
│   ├── 原理：用 Jump Flood 算法对 Stencil Mask 做距离场扩散
│   ├── 优点：描边宽度完全可控（可做粗描边）、可以渐变描边
│   ├── 缺点：计算复杂度高（多 Pass 距离场传播）、移动端不友好
│   └── 适用：需要特殊描边效果（渐变描边、虚线描边、发光描边）
│
└── URP Renderer Feature 机制
    ├── 渲染管线插入点
    │   ├── BeforeRenderingPrepasses（阴影之前）
    │   ├── AfterRenderingOpaques（不透明之后）
    │   ├── AfterRenderingSky（天空盒之后）
    │   ├── BeforeRenderingPostProcessing（后处理之前）
    │   └── AfterRenderingTransparents（透明之后）
    ├── ScriptableRenderPass 生命周期
    │   ├── OnCameraSetup()：配置 RT、材质
    │   ├── Execute()：执行渲染命令
    │   └── OnCameraCleanup()：释放临时 RT
    └── 多 Pass 叠加
        ├── 每个 Renderer Feature 可以包含多个 Pass
        └── 多个 Feature 按顺序执行（注意顺序）
```

#### 代码实现

**方案3 实现：URP Renderer Feature + Stencil 描边（交互描边）**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

// ============================================================
// 交互描边 Renderer Feature
// 使用方法：在 URP Renderer Data 上添加此 Feature
// 配合 InteractionOutlineController.cs 控制开关
// ============================================================

public class InteractionOutlineFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class OutlineSettings
    {
        public LayerMask outlineLayer = 0;        // 描边物体的 Layer
        public Material outlineMaterial;           // 描边材质（法线外扩 + Stencil Test）
        public Color outlineColor = Color.yellow;  // 描边颜色
        [Range(0.001f, 0.1f)] public float outlineWidth = 0.02f;
        public bool renderInSceneView = true;
    }
    
    public OutlineSettings settings = new OutlineSettings();
    private OutlineRenderPass _pass;
    
    public override void Create()
    {
        _pass = new OutlineRenderPass(settings)
        {
            renderPassEvent = RenderPassEvent.AfterRenderingOpaques
        };
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.outlineMaterial == null) return;
        
        // 场景视图控制
        if (!settings.renderInSceneView && renderingData.cameraData.cameraType == CameraType.SceneView)
            return;
        
        renderer.EnqueuePass(_pass);
    }
    
    protected override void Dispose(bool disposing)
    {
        _pass?.Dispose();
    }
}

// ============================================================
// 描边渲染 Pass
// ============================================================
public class OutlineRenderPass : ScriptableRenderPass
{
    private InteractionOutlineFeature.OutlineSettings _settings;
    private FilteringSettings _filteringSettings;
    private RenderStateBlock _renderStateBlock;
    private StencilState _stencilState;
    
    // Stencil 参数
    private const int STENCIL_REF = 1;
    private const int STENCIL_MASK = 1;
    
    public OutlineRenderPass(InteractionOutlineFeature.OutlineSettings settings)
    {
        _settings = settings;
        _filteringSettings = new FilteringSettings(
            RenderQueueRange.opaque, 
            _settings.outlineLayer
        );
        
        // Stencil 状态：只画 Stencil ≠ 1 的像素（本体已经被标记为 1）
        _stencilState = new StencilState(
            compareFunction: CompareFunction.NotEqual,  // Stencil ≠ 1 时才渲染
            passOperation: StencilOp.Keep,
            failOperation: StencilOp.Keep,
            depthFailOperation: StencilOp.Keep
        );
        
        _renderStateBlock = new RenderStateBlock(RenderStateMask.Stencil)
        {
            stencilState = _stencilState,
            stencilReference = STENCIL_REF
        };
    }
    
    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("InteractionOutline");
        
        // 设置描边材质参数
        var block = new MaterialPropertyBlock();
        cmd.SetGlobalColor("_OutlineColor", _settings.outlineColor);
        cmd.SetGlobalFloat("_OutlineWidth", _settings.outlineWidth);
        
        // 用 Stencil Test 渲染描边物体
        // 前提：描边物体在不透明 Pass 中已经写了 Stencil = 1
        // 这个 Pass 渲染它们的背面（Cull Front），通过法线外扩产生轮廓
        var drawSettings = new DrawingSettings(
            new ShaderTagId("UniversalForward"), 
            new SortingSettings(renderingData.cameraData.camera)
        )
        {
            overrideMaterial = _settings.outlineMaterial,
            overrideMaterialPassIndex = 0
        };
        
        context.DrawRenderers(renderingData.cullResults, ref drawSettings, 
                              ref _filteringSettings, ref _renderStateBlock);
        
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
    
    public void Dispose() { }
}
```

**描边 Shader（配合 Stencil 的法线外扩）：**

```hlsl
Shader "Custom/InteractionOutline"
{
    Properties
    {
        _OutlineColor ("Outline Color", Color) = (1, 0.8, 0, 1)
        _OutlineWidth ("Outline Width", Range(0.001, 0.1)) = 0.02
        _ZOffset ("Z Offset", Range(0, 0.001)) = 0.0001
    }
    
    SubShader
    {
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Opaque"
            "Queue" = "Geometry+1"  // 在本体之后渲染
        }
        
        Pass
        {
            Name "Outline"
            Tags { "LightMode" = "UniversalForward" }
            
            Cull Front   // 渲染背面（外扩后的背面 = 轮廓）
            ZWrite Off
            ZTest LEqual
            ColorMask RGB
            Offset 1, 1  // 防止 Z-fighting
            
            Stencil
            {
                Ref 1
                Comp NotEqual  // 只画 Stencil ≠ 1 的像素（本体外面）
                Fail Keep
                ZFail Keep
            }
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            CBUFFER_START(UnityPerMaterial)
                float4 _OutlineColor;
                float  _OutlineWidth;
                float  _ZOffset;
            CBUFFER_END
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };
            
            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
            };
            
            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                
                // 法线外扩（模型空间）
                float3 pos = IN.positionOS.xyz + IN.normalOS * _OutlineWidth;
                
                OUT.positionHCS = TransformObjectToHClip(pos);
                
                // Z 偏移防止描边被本体遮挡
                OUT.positionHCS.z -= _ZOffset * OUT.positionHCS.w;
                
                return OUT;
            }
            
            half4 frag(Varyings IN) : SV_Target
            {
                return half4(_OutlineColor.rgb, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**交互描边控制器（C# 脚本）：**

```csharp
using UnityEngine;

// 挂在可交互物体上，控制描边开关
[RequireComponent(typeof(Renderer))]
public class InteractionOutlineController : MonoBehaviour
{
    [Header("描边配置")]
    public Color normalOutlineColor = new Color(1f, 0.8f, 0f, 1f);  // 默认金色
    public Color selectedOutlineColor = new Color(1f, 0.2f, 0.2f, 1f); // 选中红色
    public Color hoverOutlineColor = new Color(0.3f, 1f, 0.5f, 1f);   // 悬停绿色
    
    [Header("Stencil 配置")]
    [Tooltip("物体的材质需要写入 Stencil 值为 1")]
    public bool writeStencil = true;
    
    private Renderer _renderer;
    private MaterialPropertyBlock _mpb;
    private static readonly int OutlineColorID = Shader.PropertyToID("_OutlineColor");
    private static readonly int StencilRefID = Shader.PropertyToID("_StencilRef");
    
    public enum OutlineState { None, Hover, Selected }
    private OutlineState _state = OutlineState.None;
    
    void Awake()
    {
        _renderer = GetComponent<Renderer>();
        _mpb = new MaterialPropertyBlock();
    }
    
    public void SetState(OutlineState state)
    {
        if (_state == state) return;
        _state = state;
        UpdateOutline();
    }
    
    void UpdateOutline()
    {
        _renderer.GetPropertyBlock(_mpb);
        
        switch (_state)
        {
            case OutlineState.None:
                // 关闭 Stencil 写入，描边消失
                _mpb.SetFloat(StencilRefID, 0);
                break;
            case OutlineState.Hover:
                _mpb.SetFloat(StencilRefID, 1);
                Shader.SetGlobalColor(OutlineColorID, hoverOutlineColor);
                break;
            case OutlineState.Selected:
                _mpb.SetFloat(StencilRefID, 1);
                Shader.SetGlobalColor(OutlineColorID, selectedOutlineColor);
                break;
        }
        
        _renderer.SetPropertyBlock(_mpb);
    }
    
    // 示例：射线检测交互
    void OnMouseEnter() => SetState(OutlineState.Hover);
    void OnMouseExit() => SetState(OutlineState.None);
    void OnMouseDown() => SetState(OutlineState.Selected);
}
```

**5 种描边方案对比表**

| 方案 | 原理 | 精度 | 性能 | 适用场景 | 移动端 |
|------|------|------|------|----------|--------|
| 法线外扩 | 顶点沿法线膨胀 | 高 | 低（1 DC/物体） | 角色卡通渲染 | ✅ 推荐 |
| 后处理边缘检测 | Sobel 卷积 | 中 | 中（全屏 Pass） | 通用描边 | ⚠️ 谨慎 |
| Stencil 描边 | Stencil Test + 外扩 | 高 | 低（1 DC/物体） | 交互提示 | ✅ 推荐 |
| Rim Light | Fresnel 边缘亮 | 低 | 极低（0 额外 DC） | 简单高亮 | ✅ |
| Jump Flood | 距离场扩散 | 高 | 高（多 Pass） | 粗描边/特效 | ❌ |

### ⚡ 实战经验

1. **法线外扩的"硬边问题"**：机械模型（如箱子、建筑）有很多硬边缘，顶点法线不平滑会导致描边断裂。解决方案：在 DCC 软件中导出一套"平滑法线"存储在顶点色或 UV2 中，描边 Pass 用平滑法线外扩
2. **Stencil 值需要每帧清理**：URP 默认不会自动清 Stencil Buffer。如果你的描边物体被销毁/禁用，但 Stencil 值还在，后续渲染可能出 bug。在 Renderer Feature 的 Pass 开始前加一个 `cmd.ClearStencil()`
3. **后处理描边的"内部线条"问题**：Sobel 算子检测的不只是轮廓，还包括法线/深度突变的内部边缘（如衣服褶皱）。用 Stencil Mask 或法线阈值过滤掉不想要的内部描边
4. **多套描边共存的 Layer 分离**：交互描边用 Layer "Interactable"（Layer 8），角色卡通描边在角色 Shader 内部处理。两套描边走不同管线阶段，不会互相干扰
5. **URP 版本兼容性**：URP 12+（Unity 2021+）的 Renderer Feature API 与旧版有较大差异。面试中提到版本兼容性，说明你对实际项目踩坑有经验

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么只给特定物体描边 | Stencil Buffer / Custom Depth | 学 URP Stencil 写入与测试 |
| 法线外扩描边断裂 | 硬边模型法线问题 | 学平滑法线导出 + 熔融法线技术 |
| 不知道在哪插入描边 Pass | URP 渲染管线时机 | 学 RenderPassEvent 枚举值 |
| 多套描边互相干扰 | Layer / Stencil / RT 分离 | 学渲染目标的隔离策略 |
| 描边性能差 | 各方案性能差异 | 做 Profiler 分析 + Draw Call 统计 |
| 描边被遮挡或穿透 | Z-Test / Depth 逻辑 | 复习深度测试原理 + Z Bias |

### 🔗 相关问题

- 如何实现"穿透墙壁的角色描边"（X-Ray 描边）？（提示：用 Depth Test 比较 + 颜色叠加）
- 卡通渲染中的描边为什么有时粗细不均？（顶点法线不均匀 / 屏幕空间宽度 vs 世界空间宽度）
- 如何在 Shader Graph 中实现法线外扩描边？（Vertex Stage 的 Position Override）
- Jump Flood 描边具体怎么实现？适合什么场景？（距离场算法 + 多 Pass 传播）
