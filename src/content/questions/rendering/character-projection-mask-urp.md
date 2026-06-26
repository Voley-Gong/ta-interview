---
title: "URP下实现角色脚底投影遮罩：如何让角色只在自己脚下显示阴影？"
category: "rendering"
level: 3
tags: ["URP", "Renderer Feature", "投影遮罩", "Stencil", "角色阴影", "Render Texture"]
hint: "不是关闭全局光照——是用 Stencil 标记角色区域，只在角色脚下渲染一张独立的俯视深度图作为遮罩"
related: ["rendering/urp-renderer-feature", "rendering/planar-projection-shadow", "rendering/custom-screen-space-shadow-soften", "rendering/stencil-selection-outline-urp"]
---

## 参考答案

### 🎬 场景描述

面试官（渲染组长级别）说：

> "我们做一个俯视角 ARPG 手游，URP 管线下，角色身上需要有一个'脚底光圈'效果——但这个光圈只在自己脚下显示，不能影响地面其他角色。全局阴影太贵了，而且多角色叠在一起会乱。你给我一个方案，要求：
> 1. 每个角色脚下有一个柔化投影圆盘
> 2. 投影只显示在自己的范围内，不互相干扰
> 3. 移动端性能可控，不能每个角色单独开一张 Render Texture"

这是米哈游《原神》项目组、腾讯天美工作室的真实面试题。核心矛盾是 **角色局部阴影控制 vs 移动端性能预算**。

### ✅ 核心要点

1. **不是关闭全局 Shadow，而是做一层"角色专属投影遮罩"**
2. **Stencil 方案**：用 Stencil Buffer 标记地面区域，只在角色脚下的范围内渲染投影
3. **屏幕空间方案**：将所有角色的投影绘制到一张共享的 RT 上，通过 UV 偏移区分角色
4. **顶点着色器方案（最省）**：在角色模型底部直接生成一个几何体（圆形/椭圆形 mesh），用 Falloff 贴图控制透明度
5. **混合方案最实用**：Vertex Offset 圆盘 mesh + SDF 软边 + Stencil 遮挡

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：每个角色脚下有独立柔化投影，不互相干扰，移动端性能可控
     ↓ 倒推
"不互相干扰" = 每个角色投影有独立的遮罩区域
     ↓ 倒推
"性能可控" = 不能每角色一张 RT（100角色 = 100张RT = 内存爆炸）
     ↓ 倒推
方案A：每角色一个脚底 Mesh + Shader（几何体方案，最轻量）
方案B：单张 Scene-Level RT + Stencil 区分（屏幕空间方案，中等开销）
方案C：Deferred Decal（G-Buffer 阶段贴花，URP 不直接支持需自定义）
     ↓ 倒推选择
移动端首选 → 方案A（几何体方案）+ 方案B的 Stencil 思路用于遮挡
```

#### 知识点拆解（倒推树）

```
角色脚底投影遮罩
├── URP Render Pipeline
│   ├── URP Asset 配置（Shadow 距离、Cascade）
│   ├── Renderer Feature 机制（何时插入自定义 Pass）
│   └── Render Pass 执行顺序
├── Stencil Buffer
│   ├── Stencil Ref 值分配（地面=1, 角色=2, 投影=3）
│   ├── Compare Function（Equal / NotEqual / Always）
│   └── Stencil 操作（Keep / Replace / Zero）
├── 投影几何体
│   ├── 脚底圆盘 Mesh 自动生成（C# Editor 脚本）
│   ├── Vertex Shader 偏移（贴地、法线对齐）
│   └── 像素 SDF 软边（smoothstep + distance field）
├── 遮挡处理
│   ├── ZTest（投影被墙挡住时不显示）
│   ├── Stencil 遮罩（角色 A 投影不覆盖角色 B）
│   └── 高度衰减（跳起时投影变淡变大）
└── 性能控制
    ├── Batch 合批（所有投影用一个 Material）
    ├── GPU Instancing（相同 Mesh + Material）
    └── LOD 策略（远处角色投影简化或关闭）
```

#### 代码实现

**方案A：脚底 Mesh + SDF 软边 Shader（推荐移动端）**

C# 脚本自动生成脚底圆盘：

```csharp
using UnityEngine;
using UnityEditor;

[RequireComponent(typeof(CharacterController))]
public class CharacterFootShadow : MonoBehaviour
{
    [Header("投影参数")]
    [SerializeField] private float _radius = 0.8f;
    [SerializeField] private float _heightOffset = 0.02f;
    [SerializeField] private Material _shadowMat;
    
    private GameObject _shadowQuad;
    private MeshRenderer _shadowRenderer;
    
    void Awake()
    {
        // 生成脚底圆盘 Mesh
        _shadowQuad = GameObject.CreatePrimitive(PrimitiveType.Quad);
        _shadowQuad.name = "FootShadow";
        Destroy(_shadowQuad.GetComponent<Collider>());
        
        _shadowRenderer = _shadowQuad.GetComponent<MeshRenderer>();
        _shadowRenderer.material = _shadowMat;
        _shadowRenderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.Off;
        _shadowRenderer.receiveShadows = false;
        
        _shadowQuad.transform.SetParent(transform);
    }
    
    void LateUpdate()
    {
        // Raycast 贴地对齐
        if (Physics.Raycast(transform.position + Vector3.up * 0.1f, 
                           Vector3.down, out var hit, 2f, 
                           LayerMask.GetMask("Ground")))
        {
            _shadowQuad.transform.position = hit.point + Vector3.up * _heightOffset;
            // 对齐地面法线
            _shadowQuad.transform.rotation = Quaternion.FromToRotation(
                Vector3.up, hit.normal);
        }
        
        // 高度衰减：跳起时投影变大变淡
        float height = transform.position.y - (_shadowQuad.transform.position.y - _heightOffset);
        float fade = Mathf.Clamp01(1f - height / 3f);
        _shadowMat?.SetFloat("_Fade", fade);
        _shadowMat?.SetFloat("_Scale", 1f + height * 0.3f);
    }
}
```

SDF 软边投影 Shader：

```hlsl
Shader "Custom/FootShadowSDF"
{
    Properties
    {
        _Radius ("投影半径", Float) = 0.8
        _Softness ("边缘柔化", Range(0, 1)) = 0.3
        _Opacity ("不透明度", Range(0, 1)) = 0.5
        _Fade ("高度衰减", Range(0, 1)) = 1.0
        _Scale ("缩放", Float) = 1.0
    }
    
    SubShader
    {
        Tags 
        { 
            "RenderType" = "Transparent" 
            "Queue" = "Transparent-1"
            "RenderPipeline" = "UniversalPipeline"
        }
        
        Blend SrcAlpha OneMinusSrcAlpha
        ZWrite Off
        ZTest LEqual
        
        // Stencil: 只在地面上绘制（地面 Ref=1）
        Stencil
        {
            Ref 1
            Comp Equal
            Pass Keep
        }
        
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            CBUFFER_START(UnityPerMaterial)
                float _Radius;
                float _Softness;
                float _Opacity;
                float _Fade;
                float _Scale;
            CBUFFER_END
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };
            
            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };
            
            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target
            {
                // SDF 圆盘
                float2 centeredUV = input.uv - 0.5;
                float dist = length(centeredUV) * 2.0; // [0, 1] within circle
                
                // 距离场软边
                float shadow = 1.0 - smoothstep(
                    1.0 - _Softness, 
                    1.0, 
                    dist / _Scale);
                
                // 应用不透明度和高度衰减
                shadow *= _Opacity * _Fade;
                
                return half4(0, 0, 0, shadow);
            }
            ENDHLSL
        }
    }
}
```

**方案B：URP Renderer Feature + 共享 RT（适用于角色密集场景）**

```csharp
// URP Renderer Feature: 将所有角色投影绘制到一张下视角 RT
public class CharacterProjectionFeature : ScriptableRendererFeature
{
    class ProjectionPass : ScriptableRenderPass
    {
        private RTHandle _projectionRT;
        private const string k_ProjectionRTName = "_CharacterProjectionMap";
        
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.depthBufferBits = 0;
            desc.colorFormat = RenderTextureFormat.R8;
            desc.width /= 2;  // 半分辨率省性能
            desc.height /= 2;
            
            RenderingUtils.ReAllocateIfNeeded(ref _projectionRT, desc, 
                FilterMode.Bilinear, TextureWrapMode.Clamp, name: k_ProjectionRTName);
            
            // 清除为白色（无投影）
            cmd.SetRenderTarget(_projectionRT);
            cmd.ClearRenderTarget(false, true, Color.white);
        }
        
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            // 筛选所有带 CharacterProjectionMarker 的物体
            var sortingCriteria = SortingCriteria.CommonTransparent;
            var drawingSettings = CreateDrawingSettings(
                new ShaderTagId("CharacterProjection"), 
                ref renderingData, sortingCriteria);
            
            var filteringSettings = new FilteringSettings(
                RenderQueueRange.transparent, 
                LayerMask.GetMask("Character"));
            
            context.DrawRenderers(renderingData.cullResults, ref drawingSettings, 
                ref filteringSettings);
        }
    }
}
```

### ⚡ 实战经验

1. **移动端首选方案A（几何体方案）**：100 个角色 = 100 个 Quad，只有 1 个 Draw Call（GPU Instancing），几乎零开销
2. **Stencil 遮罩是防穿模的关键**：没加 Stencil 时，角色 A 的投影会"飞"到角色 B 身上
3. **高度衰减曲线要调**：跳起时投影不是线性变淡，用 `1 - (h/maxH)^2` 的曲线更自然
4. **地形坡度处理**：上坡时投影要贴合法线，否则会"浮空"——用 Raycast 法线对齐
5. **多光源下投影颜色**：不是纯黑色，应该是光源颜色的互补色——但移动端一般直接用半透黑就够了

### 🎯 能力体检清单

- [ ] 能否说出 URP 中 Stencil 的配置位置和作用？
- [ ] 方案A 和方案B 的性能差异在哪里？什么场景选哪个？
- [ ] 如果角色站在悬崖边，投影如何正确显示而不悬空？
- [ ] GPU Instancing 对投影 Mesh 的要求是什么？为什么不能实时缩放 Transform？
- [ ] 半分辨率 RT 方案中，UV 采样如何避免边缘锯齿？

### 🔗 相关问题

- [URP 自定义 Renderer Feature](rendering/urp-renderer-feature.md)
- [平面投影阴影](rendering/planar-projection-shadow.md)
- [URP 模板选择描边](rendering/stencil-selection-outline-urp.md)
- [自定义屏幕空间阴影柔化](rendering/custom-screen-space-shadow-soften.md)
