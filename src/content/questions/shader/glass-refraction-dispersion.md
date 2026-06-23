---
title: "玻璃折射与色散：如何用 Shader 实现高级感的透明材质？"
category: "shader"
level: 3
tags: ["玻璃", "折射", "色散", "GrabTexture", "URP", "透明渲染"]
hint: "玻璃的灵魂不是透明——是折射偏移 + 色散分离 + 菲涅尔边缘 + 双面厚度感知"
related: ["shader/dissolve-effect", "rendering/urp-renderer-feature", "shader/energy-shield-effect"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们项目中有一个水晶展示柜的场景，里面放了各种宝石和玻璃器皿。美术反馈现在的玻璃看起来像塑料——透明但没有折射，没有色散，没有厚度感。你需要在 URP 下写一个玻璃 Shader，要求：
> 1. 能看到背后的物体但有折射偏移
> 2. 边缘有菲涅尔反射
> 3. 厚的地方有轻微的色散（chromatic aberration）
> 4. 性能要在移动端可接受

这是米哈游、叠纸等注重画面品质的公司高频考察的 Shader 题——表面是"写玻璃"，实际考察的是：屏幕空间折射、菲涅尔、色散、透明物体渲染管线的综合理解。

### ✅ 核心要点

1. **屏幕空间折射（SSR-Refraction）**：用 Opaque Texture 抓取屏幕颜色，用法线偏移 UV 实现折射
2. **菲涅尔边缘**：`pow(1 - dot(N, V), power)` 控制边缘反射强度，让玻璃边缘"亮起来"
3. **色散分离**：RGB 通道用不同偏移量采样，模拟棱镜分光
4. **厚度感知**：利用顶点色或顶点法线 AO 估计厚度，厚处色散更强
5. **透明排序与渲染队列**：玻璃必须在所有不透明物体之后渲染，需要正确设置 Queue 和 RenderType

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：玻璃看起来有"实感"——透过它看东西是弯的，边缘在发光，厚的地方有彩虹边
                ↑
倒推1：折射偏移从哪来 → 需要屏幕背后的像素 + 法线驱动的 UV 偏移
倒推2：色散怎么实现   → R/G/B 三个通道用不同偏移量采样屏幕纹理
倒推3：边缘为什么亮   → 菲涅尔效应：掠射角反射率趋近 100%
倒推4：厚度怎么感知   → 顶点色 / 折射强度随法线与视角的夹角变化
倒推5：性能怎么保证   → 移动端降低采样次数 / 用 RT 分辨率减半 / 去掉色散
```

#### 知识点拆解（倒推树）

```
玻璃 Shader
├── 屏幕空间折射
│   ├── URP Opaque Texture（_CameraOpaqueTexture）
│   │   ├── 开启：URP Asset → Opaque Texture = true
│   │   └── 原理：不透明物体渲染后，拷贝屏幕到一张 RT
│   ├── 折射 UV 偏移
│   │   ├── 用屏幕 UV + 法线 XY 分量 × 强度
│   │   └── 偏移量受厚度 / 距离调制
│   └── 精度问题
│       ├── 半透明物体不会被抓进 Opaque Texture（只在 Transparent 之前抓）
│       └── 多层玻璃叠加时只有第一层有折射背景
│
├── 色散（Chromatic Aberration）
│   ├── 原理：不同波长光折射率不同 → RGB 偏移量不同
│   ├── 实现：R 通道 offset × 0.8, G × 1.0, B × 1.2
│   ├── 控制参数：色散强度（dispersion = 0 时无色散）
│   └── 厚度调制：厚的区域色散更强
│       └── 厚度来源：顶点色 R 通道 / Ambient Occlusion / 手绘 Mask
│
├── 菲涅尔反射
│   ├── Schlick 近似：R(θ) = R0 + (1 - R0)(1 - cosθ)^5
│   ├── 快速近似：fresnel = pow(1 - max(dot(N, V), 0), _FresnelPower)
│   ├── 叠加方式：lerp(refractionColor, reflectionColor, fresnel)
│   └── 反射颜色来源
│       ├── 环境贴图（Cubemap / Reflection Probe）
│       └── 或简单纯色 tint（移动端方案）
│
├── 透明渲染管线
│   ├── RenderQueue = Transparent（3000+）
│   ├── ZWrite Off（玻璃不写入深度，否则后面物体被裁掉）
│   ├── 双面渲染（Cull Off）—— 玻璃正面和背面都要渲染
│   │   └── 背面法线翻转：Varyings 中 `normal = face ? normal : -normal`
│   └── 排序问题
│       ├── 引擎按距离排序透明物体
│       └── 多层玻璃可能排序错误 → 方案：Ray Tracing / Order Independent Transparency
│
└── 移动端优化
    ├── 折射采样降分辨率（Half RT）
    ├── 色散用 2 次采样代替 3 次（RG 合并 / B 单独）
    ├── 菲涅尔用简化公式（避免 pow，用乘法级数近似）
    └── 考虑用预烘焙环境反射替代实时 Reflection Probe
```

#### 代码实现

**URP 玻璃 Shader（完整手写 HLSL）：**

```hlsl
Shader "Custom/GlassRefraction"
{
    Properties
    {
        [Header(Refraction)]
        _OpaqueTex ("Opaque Texture", 2D) = "white" {}
        _RefractionStrength ("Refraction Strength", Range(0, 0.1)) = 0.02
        _Dispersion ("Dispersion (Chromatic Aberration)", Range(0, 1)) = 0.3
        
        [Header(Fresnel)]
        _FresnelPower ("Fresnel Power", Range(0.5, 8)) = 3.0
        _FresnelColor ("Fresnel Color", Color) = (1, 1, 1, 1)
        _FresnelIntensity ("Fresnel Intensity", Range(0, 3)) = 1.0
        
        [Header(Surface)]
        _TintColor ("Tint Color", Color) = (0.9, 0.95, 1.0, 0.8)
        _Smoothness ("Smoothness", Range(0, 1)) = 0.95
        
        [Header(Thickness)]
        _ThicknessMap ("Thickness Map (Vertex Color R)", 2D) = "white" {}
        _ThicknessInfluence ("Thickness Influence", Range(0, 2)) = 1.0
        
        [Header(Advanced)]
        [Toggle(_DOUBLE_SIDED)] _DoubleSided ("Double Sided", Float) = 1
    }
    
    SubShader
    {
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Transparent"
            "Queue" = "Transparent"
        }
        
        Pass
        {
            Name "GlassRefraction"
            Tags { "LightMode" = "UniversalForward" }
            
            ZWrite Off
            Cull Off  // 双面渲染
            Blend SrcAlpha OneMinusSrcAlpha
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            #pragma shader_feature_local _DOUBLE_SIDED
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            // URP 的屏幕不透明纹理
            TEXTURE2D(_CameraOpaqueTexture);
            SAMPLER(sampler_CameraOpaqueTexture);
            
            TEXTURE2D(_ThicknessMap);
            SAMPLER(sampler_ThicknessMap);
            
            CBUFFER_START(UnityPerMaterial)
                float4 _ThicknessMap_ST;
                float4 _FresnelColor;
                float4 _TintColor;
                float  _RefractionStrength;
                float  _Dispersion;
                float  _FresnelPower;
                float  _FresnelIntensity;
                float  _Smoothness;
                float  _ThicknessInfluence;
            CBUFFER_END
            
            struct Attributes
            {
                float4 positionOS  : POSITION;
                float3 normalOS    : NORMAL;
                float2 uv          : TEXCOORD0;
                float4 vertexColor : COLOR;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };
            
            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float2 uv          : TEXCOORD0;
                float3 normalWS    : TEXCOORD1;
                float3 viewDirWS   : TEXCOORD2;
                float4 screenPos   : TEXCOORD3;
                float  thickness   : TEXCOORD4;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };
            
            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);
                
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _ThicknessMap);
                
                VertexPositionInputs posInputs = GetVertexPositionInputs(IN.positionOS.xyz);
                VertexNormalInputs normInputs = GetVertexNormalInputs(IN.normalOS);
                
                OUT.normalWS = normInputs.normalWS;
                OUT.viewDirWS = GetCameraPositionWS() - posInputs.positionWS;
                OUT.screenPos = ComputeScreenPos(OUT.positionHCS);
                
                // 厚度：优先用顶点色 R，没有则采样厚度纹理
                OUT.thickness = IN.vertexColor.r > 0.01 ? IN.vertexColor.r : 
                                SAMPLE_TEXTURE2D_LOD(_ThicknessMap, sampler_ThicknessMap, OUT.uv, 0).r;
                
                return OUT;
            }
            
            half4 frag(Varyings IN, bool isFrontFace : SV_IsFrontFace) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);
                
                float3 N = normalize(IN.normalWS);
                float3 V = normalize(IN.viewDirWS);
                
                #if defined(_DOUBLE_SIDED)
                    // 背面法线翻转
                    if (!isFrontFace) N = -N;
                #endif
                
                // === 屏幕空间折射 ===
                float2 screenUV = IN.screenPos.xy / IN.screenPos.w;
                
                // 法线驱动的折射偏移（屏幕空间）
                float2 refractionOffset = N.xz * _RefractionStrength * IN.thickness * _ThicknessInfluence;
                // 注意：用 xz 还是 xy 取决于玻璃的朝向，一般水平面用 xz
                // 更通用做法：把法线投影到屏幕空间
                // float3 screenNormal = mul((float3x3)UNITY_MATRIX_VP, float4(N, 0)).xyz;
                // refractionOffset = screenNormal.xy * _RefractionStrength;
                
                // === 色散（RGB 分离采样） ===
                float dispersionAmount = _Dispersion * IN.thickness;
                
                // R 通道：偏移量较小
                float2 uvR = screenUV + refractionOffset * (1.0 - dispersionAmount * 0.5);
                // G 通道：基准偏移
                float2 uvG = screenUV + refractionOffset;
                // B 通道：偏移量较大
                float2 uvB = screenUV + refractionOffset * (1.0 + dispersionAmount * 0.5);
                
                half refractionR = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, uvR).r;
                half refractionG = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, uvG).g;
                half refractionB = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, uvB).b;
                
                half3 refractionColor = half3(refractionR, refractionG, refractionB);
                
                // === 菲涅尔 ===
                float NdotV = saturate(dot(N, V));
                float fresnel = pow(1.0 - NdotV, _FresnelPower);
                // Schlick 近似（更精确）
                // float R0 = 0.04; // 玻璃的菲涅尔0度反射率约 4%
                // float fresnel = R0 + (1.0 - R0) * pow(1.0 - NdotV, 5.0);
                
                // === 混合 ===
                half3 finalColor = refractionColor * _TintColor.rgb;
                finalColor = lerp(finalColor, _FresnelColor.rgb, fresnel * _FresnelIntensity);
                
                // Alpha：边缘越透明，中心越不透明（厚度感）
                half alpha = lerp(_TintColor.a, 1.0, fresnel * 0.5);
                alpha *= saturate(IN.thickness * 2.0); // 太薄的地方接近透明
                
                return half4(finalColor, alpha);
            }
            ENDHLSL
        }
    }
    
    // 需要回退到不透明物体，确保阴影正确
    FallBack "Universal Render Pipeline/Lit"
}
```

**URP Renderer Feature — 开启 Opaque Texture：**

```csharp
// 在 URP Asset 上确保：
// Opaque Texture = ✅ true
// Opaque Texture Depth = 0 (默认即可)
// 这会让 URP 在不透明物体渲染完成后，自动拷贝屏幕到 _CameraOpaqueTexture

// 如果需要更高控制，可以自定义 Renderer Feature：
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class GlassRefractionFeature : ScriptableRendererFeature
{
    class GlassPass : ScriptableRenderPass
    {
        private RenderTargetIdentifier _cameraColor;
        private int _opaqueTextureID = Shader.PropertyToID("_CameraOpaqueTexture");
        
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            _cameraColor = renderingData.cameraColorTarget;
        }
        
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get("GlassRefraction");
            
            // 将当前颜色缓冲拷贝到 _CameraOpaqueTexture
            // 这样透明物体可以在之后采样它
            cmd.GetTemporaryRT(_opaqueTextureID, 
                renderingData.cameraData.cameraTargetDescriptor);
            cmd.CopyTexture(_cameraColor, _opaqueTextureID);
            
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
    
    private GlassPass _pass;
    
    public override void Create()
    {
        _pass = new GlassPass
        {
            renderPassEvent = RenderPassEvent.BeforeRenderingTransparents
        };
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        renderer.EnqueuePass(_pass);
    }
}
```

**方案对比表：玻璃实现技术路线**

| 方案 | 折射来源 | 色散 | 菲涅尔 | 性能 | 适用平台 |
|------|----------|------|--------|------|----------|
| Opaque Texture + UV 偏移 | ✅ | ✅ | ✅ | 中 | PC/Mobile |
| GrabPass（内置管线） | ✅ | ✅ | ✅ | 低（管线 stall） | PC |
| Planar Reflection | ✅ | ❌ | ✅ | 高 | PC/主机 |
| Cubemap 假反射 | ❌ | ❌ | ✅ | 极低 | Mobile |
| Ray Tracing 折射 | ✅ 真实 | ✅ 真实 | ✅ | 极高 | RTX |
| Screen Space Ray Marching | ✅ | ✅ | ✅ | 高 | PC |

### ⚡ 实战经验

1. **不透明纹理是 URP 折射的基石**：一定要在 URP Asset 中开启 `Opaque Texture`，否则 `_CameraOpaqueTexture` 采样出来是黑的——这是新手最常见的坑
2. **色散宁弱勿强**：色散强度超过 0.5 就会显得"科幻"而非"写实"。真实玻璃的色散非常微妙，只有在边缘和厚度变化处才肉眼可见
3. **折射偏移方向问题**：屏幕空间法线偏移用 `N.xy` 还是 `N.xz` 取决于玻璃面的朝向。通用做法是将法线变换到 View Space 再取 xy
4. **多层玻璃的叠加问题**：因为 Opaque Texture 在所有透明物体之前抓取，第二层玻璃采样到的是第一层玻璃之前的画面，而非第一层玻璃本身。这个问题在实时渲染中很难解决（需要 OIT）
5. **性能分级**：PC 端可以全开（3 次采样 + 色散 + 菲涅尔），移动端建议关闭色散（1 次采样）+ 简化菲涅尔

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 折射背景是黑的 | URP Opaque Texture 机制 | 学 URP Asset 配置 + Renderer Feature |
| 不知道色散怎么实现 | 光的波长与折射率关系 | 学色差原理 + 多通道采样技巧 |
| 菲涅尔效果不自然 | Schlick 近似 / 菲涅尔公式 | 复习光学基础 + 着色器数学函数 |
| 多层玻璃渲染错误 | 透明物体排序原理 | 学 OIT / Depth Peeling / Weighted Blended OIT |
| 移动端太卡 | 采样次数 / 带宽优化 | 学移动端 GPU 架构 + 性能预算 |
| 法线偏移方向不对 | 屏幕空间 vs 世界空间 | 学空间变换矩阵 + View Space 投影 |

### 🔗 相关问题

- 玻璃上的水滴怎么和玻璃 Shader 联动？（水滴作为装饰纹理，折射强度叠加）
- 冰块和玻璃的 Shader 有什么区别？（冰块需要次表面散射 + 更强的色散）
- 如何在 Shader Graph 中实现同样的效果？什么情况下手写更好？
- 延迟渲染管线下，透明物体的折射怎么处理？（G-Buffer 不含透明信息 → Forward Pass 特殊处理）
