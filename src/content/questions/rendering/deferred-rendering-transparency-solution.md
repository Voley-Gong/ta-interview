---
title: "GBuffer暴雷：延迟渲染下透明物体到底怎么渲染？面试官要你说出3种方案"
category: "rendering"
level: 3
tags: ["延迟渲染", "透明渲染", "Forward+ ", "GBuffer", "混合管线", "URP", "UE5"]
hint: "延迟渲染写不了透明——GBuffer是不透明的。3种主流解法：深度剥离/独立透明Pass/混合管线(Forward+过渡)"
related: ["rendering/deferred-multi-light", "rendering/oit-transparency-order-independent", "rendering/forward-plus-cluster"]
---

## 参考答案

### 🎬 场景描述

面试官画了一个场景：一个角色站在玻璃橱窗前，橱窗里有透明展品，头顶有半透明粒子特效，远处有体积光。然后说：

> "我们的项目用的是延迟渲染（Deferred Rendering）。但现在有个问题：玻璃、水体、粒子特效这些半透明物体全部渲染不正确。GBuffer 只存了不透明信息，透明物体的光照完全丢了。你给我一套完整的解决方案。"

追问：
- "如果项目是 URP 呢？URP 的 Deferred 支持透明吗？"
- "手游上用哪种方案最省？"
- "如果玻璃既要折射又要高光反射，你的方案扛得住吗？"

### ✅ 核心要点

1. **延迟渲染的本质矛盾**：GBuffer 存储的是不透明表面属性（Albedo/Normal/Specular），透明物体无法写入 GBuffer（后面的像素会被覆盖）
2. **方案一：独立透明 Pass（最常用）**：不透明走 Deferred，透明物体走单独的 Forward Pass，在光照计算之后合成
3. **方案二：深度剥离（质量最高）**：多层深度剥离（DPS / WBOIT），逐层渲染透明表面
4. **方案三：混合管线（最现代）**：Forward+ 混合 Deferred，透明物体自动走 Forward+ Cluster 光照
5. **URP 的特殊性**：URP 2022+ 支持 Deferred 路径，但透明物体自动回退到 Forward——这是引擎内建行为，不是 bug

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：延迟渲染场景中，透明物体正确显示折射/反射/光照
               ↑
倒推1：为什么透明物体在 Deferred 下不工作？
       ├── GBuffer 是不透明的——写入即覆盖，没有"半透明"概念
       ├── 光照 Pass 读取 GBuffer 做计算——透明物体根本不在 GBuffer 里
       └── 透明物体需要的光照信息（折射背景、透过色）GBuffer 无法表达
倒推2：怎么让透明物体获得正确光照？
       ├── 方案A：透明物体单独走 Forward（手动算光照）
       ├── 方案B：多层深度剥离逐层合成
       └── 方案C：把 Deferred 改成混合管线，透明区域自动切换
倒推3：渲染顺序怎么安排？
       ├── Phase 1: 不透明 → GBuffer（Deferred 路径）
       ├── Phase 2: 光照计算（基于 GBuffer）
       ├── Phase 3: 透明物体合成（Forward 路径，叠加在光照结果上）
       └── Phase 4: 后处理
倒推4：性能怎么控制？
       ├── 透明物体数量要少（每个都是额外 Draw Call）
       ├── 透明材质的 Shader 要简化（不能像 Deferred 那样依赖 GBuffer）
       └── 手游极端情况：只用方案A，限制透明物体 ≤ 20 个
```

#### 知识点拆解（倒推树）

```
延迟渲染下的透明物体方案
├── 问题根因
│   ├── GBuffer 结构（Albedo/Normal/Material RT 不支持 Alpha 混合）
│   ├── 光照 Pass 依赖 GBuffer（透明物体缺失光照输入）
│   └── Depth Pre-pass 只写不透明深度（透明物体不参与 Z-prepass）
├── 方案一：独立透明 Forward Pass（业界标准）
│   ├── 渲染流程
│   │   ├── 1. Deferred 几何体 Pass → 填充 GBuffer
│   │   ├── 2. Deferred 光照 Pass → 输出最终不透明场景
│   │   ├── 3. 把不透明场景 Copy 到 FrameBuffer
│   │   ├── 4. 透明物体 Forward Pass（场景颜色作为背景）
│   │   │   ├── 自己计算光照（Forward lighting）
│   │   │   ├── 折射：采样不透明场景颜色
│   │   │   └── 混合：SrcAlpha / OneMinusSrcAlpha
│   │   └── 5. 后处理
│   ├── 光照一致性
│   │   ├── 主光源：和 Deferred 用同一个方向/颜色参数
│   │   ├── 多光源：Forward 中遍历影响该像素的点光源
│   │   └── 阴影：复用 Deferred 的 Shadow Map
│   ├── 优点：实现简单、性能可控、引擎兼容性好
│   ├── 缺点：透明物体的光照是近似（不走 GBuffer 完整路径）
│   └── 适用：90% 的项目（包括UE5、Unity URP默认方案）
├── 方案二：深度剥离（高质量离线方案）
│   ├── 单层深度剥离（Depth Peeling）
│   │   ├── 每次渲染最前面的透明层
│   │   ├── 逐层合成（从后往前）
│   │   └── N 层透明 = N 个 Pass（开销大）
│   ├── WBOIT（Weighted Blended OIT）
│   │   ├── 不需要排序，单 Pass 近似
│   │   ├── 用权重函数模拟正确的混合顺序
│   │   └── 质量不如真排序但性能极好
│   └── 适用：玻璃器皿、大量重叠半透明（很少在手游用）
├── 方案三：混合管线 Forward+ / Deferred
│   ├── Forward+ 核心
│   │   ├── 计算 Cluster（分块）的光源列表
│   │   ├── 不透明走 Deferred（写入 GBuffer）
│   │   ├── 透明走 Forward+（复用 Cluster 光源加速）
│   │   └── 两种路径共享 Cluster 数据
│   ├── 优点：透明物体也能享受多光源加速
│   ├── 缺点：管线复杂度高，调试困难
│   └── 适用：AAA PC/主机（CryEngine SVOGI、UE5 Lumen 混合）
├── URP 特殊处理
│   ├── URP Deferred 模式下透明物体的自动行为
│   │   ├── 不透明物体 → Deferred GBuffer
│   │   ├── 透明物体 → 自动 Forward 渲染
│   │   └── 引擎内部完成路径切换，用户无感
│   ├── URP Renderer Feature 扩展
│   │   ├── 自定义透明 Pass（ScriptableRenderPass）
│   │   ├── 注入位置：After Rendering Transparents
│   │   └── 可访问 GBuffer RT 做折射采样
│   └── URP 限制
│       ├── GBuffer 格式有限（URP 的 GBuffer 比 UE5 简化）
│       └── 透明物体的 SSR/SSAO 无法从 GBuffer 直接获取
└── 性能优化
    ├── 透明物体数量管控（引擎级批处理）
    ├── 透明面片合并（粒子 → 单个 Mesh）
    ├── 降低透明 Shader 复杂度（砍折射、用预计算 LUT）
    └── 分平台策略
        ├── PC/主机：方案一 + 折射反射全套
        ├── 高配手游：方案一 + 简化折射
        └── 低配手游：强制 Forward（不用 Deferred）
```

#### 代码实现

**方案一：URP 自定义透明 Pass（核心逻辑）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class CustomTransparentForwardPass : ScriptableRenderPass
{
    private RTHandle _colorTarget;
    private RTHandle _gbufferColor; // 延迟渲染后的不透明场景颜色
    private FilteringSettings _filteringSettings;
    private List<ShaderTagId> _shaderTagIdList;

    public CustomTransparentForwardPass()
    {
        renderPassEvent = RenderPassEvent.AfterRenderingOpaques;
        _filteringSettings = new FilteringSettings(RenderQueueRange.transparent);
        _shaderTagIdList = new List<ShaderTagId>
        {
            new ShaderTagId("UniversalForward"),
            new ShaderTagId("SRPDefaultUnlit"),
        };
    }

    public void Setup(RTHandle colorTarget, RTHandle gbufferColor)
    {
        _colorTarget = colorTarget;
        _gbufferColor = gbufferColor;
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        // 把延迟渲染结果作为透明 Pass 的背景
        ConfigureTarget(_colorTarget);
        ConfigureInput(TextureInput.Color);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("CustomTransparentForward");

        // 透明物体使用 Forward 路径渲染
        // 引擎会自动处理 SrcAlpha blend
        using (new ProfilingScope(cmd, new ProfilingSampler("Transparent Forward Pass")))
        {
            // 设置混合状态
            cmd.EnableShaderKeyword("CUSTOM_TRANSPARENCY_PASS");

            var drawSettings = CreateDrawingSettings(
                _shaderTagIdList, ref renderingData,
                SortingCriteria.CommonTransparent
            );

            context.DrawRenderers(
                renderingData.cullResults,
                ref drawSettings,
                ref _filteringSettings
            );
        }

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```

**透明玻璃 Shader（Forward 路径，折射采样不透明场景）：**

```hlsl
Shader "Custom/DeferredTransparentGlass"
{
    Properties
    {
        _BaseColor ("Glass Tint", Color) = (0.9, 0.95, 1.0, 0.3)
        _Smoothness ("Smoothness", Range(0, 1)) = 0.95
        _RefractionStrength ("Refraction Strength", Range(0, 0.1)) = 0.02
        _FresnelPower ("Fresnel Power", Range(0.5, 8)) = 3.0
        _SceneColor ("_CameraOpaqueTexture", 2D) = "" {} // URP 不透明场景纹理
    }
    SubShader
    {
        Tags
        {
            "RenderType" = "Transparent"
            "Queue" = "Transparent"
            "RenderPipeline" = "UniversalPipeline"
        }

        Pass
        {
            Name "ForwardTransparent"
            Blend SrcAlpha OneMinusSrcAlpha
            ZWrite Off
            Cull Back

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS
            #pragma multi_compile_fragment _ _CAMERA_OPAQUE_TEXTURE

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            TEXTURE2D(_CameraOpaqueTexture);
            SAMPLER(sampler_CameraOpaqueTexture);

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float _Smoothness;
                float _RefractionStrength;
                float _FresnelPower;
            CBUFFER_END

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float3 normalWS    : TEXCOORD0;
                float3 viewDirWS   : TEXCOORD1;
                float2 uv          : TEXCOORD2;
                float4 screenPos   : TEXCOORD3;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                OUT.viewDirWS = GetCameraPositionWS() - TransformObjectToWorld(IN.positionOS.xyz);
                OUT.uv = IN.uv;
                OUT.screenPos = ComputeScreenPos(OUT.positionHCS);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                float3 normalWS = normalize(IN.normalWS);
                float3 viewDirWS = normalize(IN.viewDirWS);

                // === 折射：偏移采样不透明场景颜色 ===
                float2 screenUV = IN.screenPos.xy / IN.screenPos.w;
                float2 refractOffset = normalWS.xy * _RefractionStrength;
                float3 sceneColor = SAMPLE_TEXTURE2D(
                    _CameraOpaqueTexture, sampler_CameraOpaqueTexture,
                    screenUV + refractOffset
                ).rgb;

                // === 菲涅尔反射 ===
                float fresnel = pow(1.0 - saturate(dot(normalWS, viewDirWS)), _FresnelPower);

                // === 光照（Forward 路径，手动计算） ===
                Light mainLight = GetMainLight();
                float3 lightDir = normalize(mainLight.direction);
                float NdotL = saturate(dot(normalWS, lightDir));

                // 高光（简单 Blinn-Phong 近似）
                float3 halfDir = normalize(lightDir + viewDirWS);
                float spec = pow(saturate(dot(normalWS, halfDir)), _Smoothness * 256);

                // === 最终颜色合成 ===
                float3 finalColor = sceneColor;          // 折射背景
                finalColor = lerp(finalColor, _BaseColor.rgb, _BaseColor.a);  // 玻璃色
                finalColor += mainLight.color * spec * _Smoothness;           // 高光
                finalColor += fresnel * float3(1, 1, 1) * 0.5;               // 菲涅尔边缘

                float alpha = _BaseColor.a + fresnel * 0.5;
                alpha = saturate(alpha);

                return half4(finalColor, alpha);
            }
            ENDHLSL
        }
    }
}
```

**方案对比表：**

| 方案 | 渲染开销 | 视觉质量 | 实现复杂度 | 手游适用性 | PC/主机 |
|------|----------|----------|------------|------------|---------|
| 独立透明 Forward Pass | ⭐⭐ 低 | ⭐⭐⭐ 中 | ⭐ 简单 | ✅ 首选 | ✅ 标配 |
| 深度剥离（N层） | ⭐⭐⭐⭐ 很高 | ⭐⭐⭐⭐⭐ 最高 | ⭐⭐⭐ 复杂 | ❌ 不用 | ⚠️ 仅过场动画 |
| WBOIT（单层近似） | ⭐⭐ 低 | ⭐⭐⭐ 中 | ⭐⭐ 中等 | ⚠️ 少用 | ⚠️ 辅助 |
| Forward+ 混合 | ⭐⭐⭐ 中 | ⭐⭐⭐⭐ 高 | ⭐⭐⭐⭐ 很复杂 | ❌ 太重 | ✅ AAA标配 |

### ⚡ 实战经验

- **UE5 的默认行为**：UE5 的 Deferred Shader 默认就是不透明 Deferred + 透明 Forward，引擎自动处理，美术无感知。面试时说"我了解 UE5 的混合渲染路径"即可
- **URP 的坑**：URP Deferred 模式下，`_CameraOpaqueTexture` 开关必须打开，否则透明物体的折射采样不到不透明场景。这是一个 Render Pipeline Asset 上的设置项，默认关闭
- **半透明排序灾难**：Forward 透明 Pass 中，多个重叠的半透明物体会因为排序错误产生伪影。如果项目中树木叶子和玻璃重叠，出现闪烁，99% 是 RenderQueue 排序问题
- **手游实测数据**：中端机型上，50 个透明 Draw Call（简单 Alpha Blend）≈ 2ms。如果有折射采样（多一次纹理采样）≈ 3.5ms。所以手游透明物体要严格控制在 30 个以内
- **体积光 / 体积雾的特殊处理**：这些本身就是在 Forward 透明 Pass 之后做 Ray March 的，不属于传统透明物体。URP 的 Volumetric Fog 就是在 Transparents 之后注入的
- **Deferred 下透明物体的 SSAO**：GBuffer 里没有透明物体的 Normal，所以 SSAO 只算不透明物体。透明物体边缘没有 AO，视觉上会"飘"。解决方案：在透明 Shader 里手算近似 AO（基于场景深度的简单遮蔽）

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道为什么 Deferred 不能渲染透明 | GBuffer 是不透明的，Alpha 混合和 GBuffer 互斥 | 复习 GBuffer 结构和 Alpha Blending 原理 |
| 说不清楚独立透明 Pass 的流程 | 透明物体需要单独 Forward 光照 | 学 URP/UE5 的混合渲染路径 |
| 透明物体光照和不透明不一致 | Forward 光照参数和 Deferred 不共享 | 学跨 Pass 光照一致性方案 |
| 不知道 WBOIT / Depth Peeling | OIT（Order Independent Transparency） | 学 WBOIT 权重函数 + DPS 算法 |
| 不知道 Forward+ 怎么解决透明 | Forward+ Cluster 分块光照 | 学 Forward+ 和 Cluster 光照 |
| 手游上透明物体掉帧严重 | 透明物体 = Forward = 无 SRP Batcher | 学透明物体批处理 + 数量管控 |

### 🔗 相关问题

- [延迟渲染下的多光源](../rendering/deferred-multi-light.md)：多光源在 Deferred 下很高效，但透明物体的多光源怎么办？
- [OIT 顺序无关透明度](../rendering/oit-transparency-order-independent.md)：WBOIT 和 Depth Peeling 的底层原理
- [Forward+ Cluster](../rendering/forward-plus-cluster.md)：Forward+ 是 Deferred 和 Forward 的最佳折中？
- 如果面试官追问"UE5 Lumen + 透明物体怎么交互"，你怎么回答？
