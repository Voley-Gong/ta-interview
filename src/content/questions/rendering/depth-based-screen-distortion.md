---
title: "热浪扭曲、透明物体折射：如何在 URP 中正确实现屏幕空间扭曲效果？"
category: "rendering"
level: 3
tags: ["ScreenDistortion", "URP", "折射", "后处理", "热浪", "SceneColor"]
hint: "URP 没有 Built-in 的 GrabPass——你需要用 RendererFeature + _CameraOpaqueTexture 来做屏幕采样偏移"
related: ["rendering/urp-renderer-feature", "shader/dissolve-effect", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们场景中有几个需求：1）沙漠地图的热浪空气扭曲；2）角色技能造成的空间扭曲；3）透明水晶体的折射效果。URP 管线下，你打算怎么统一处理这些屏幕空间扭曲需求？给我一个技术方案。」

（叠纸、鹰角等做风格化渲染的公司经常考察这个——核心是 URP 下如何优雅地获取场景颜色并做 UV 偏移）

### ✅ 核心要点

1. **URP 获取场景颜色**：开启 `Opaque Texture` 或用 Custom Pass 提前 Blit，通过 `_CameraOpaqueTexture` 采样
2. **深度还原世界位置**：扭曲需要知道「扭曲了多远」，需要 `DecodeDepth` 重建世界坐标做深度感知
3. **扭曲源分类**：热浪（噪声驱动）、折射体（法线驱动）、技能扭曲（径向偏移）
4. **RendererFeature 统一管理**：用一个 Distortion Pass 收集所有扭曲贡献，一次性应用，避免多次 Blit
5. **半透明排序问题**：扭曲效果在 Transparent 阶段，需要正确的渲染顺序

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：热浪扭曲背景 / 水晶折射背后物体 / 技能造成空间扭曲
                ↑
倒推1：需要采样「当前像素背后的场景颜色」→ 需要场景颜色 RT
倒推2：URP 默认没有 GrabPass → 方案 A：开 Opaque Texture  方案 B：Custom Pass Blit
倒推3：扭曲 = UV 偏移 → sceneColor.Sample(uv + distortionOffset)
倒推4：偏移量来源 → 热浪用噪声 / 折射用法线 / 技能用径向遮罩
倒推5：远处扭曲弱、近处强（或反之）→ 需要深度感知（重建世界坐标）
倒推6：多个扭曲源重叠 → 用 distortion accumulation buffer 统一收集
```

#### 知识点拆解（倒推树）

```
屏幕空间扭曲效果
├── 获取场景颜色（3 种方案）
│   ├── 方案A：URP Opaque Texture（最简单，开销小）
│   │   ├── Pipeline Asset → Opaque Texture = true
│   │   ├── Shader 中声明 TEXTURE2D(_CameraOpaqueTexture)
│   │   └── 缺点：只有不透明物体颜色，半透明物体之后才渲染
│   ├── 方案B：RendererFeature Blit 到自定义 RT
│   │   ├── 在 BeforeRenderingTransparents 注入
│   │   ├── Blit cameraColor → _SceneColorCopy
│   │   └── 优点：可控制时机（如包含/不包含天空盒）
│   └── 方案C：ColorTexture 中间纹理（全管线获取）
│       └── URP 17+ 提供 _CameraColorTexture
├── 扭曲计算
│   ├── 热浪扭曲
│   │   ├── 滚动噪声纹理（2 层不同速度的 Perlin Noise）
│   │   ├── noise → uv offset（控制扭曲强度）
│   │   └── 距离衰减：远处扭曲弱（用深度值反比缩放）
│   ├── 折射体（水晶/玻璃/水）
│   │   ├── 从物体法线计算折射方向（Snell's Law 近似）
│   │   ├── normal → uv offset（强度参数 _RefractionIntensity）
│   │   └── 色散：R/G/B 通道用不同偏移量（模拟棱镜分光）
│   └── 技能空间扭曲
│       ├── 径向扭曲：以技能中心为圆心做径向 UV 偏移
│       ├── 时变波形：sin(distance - time) 产生波纹传播
│       └── 边缘衰减：扭曲强度随距离中心增大而减弱
├── 深度感知
│   ├── 采样 _CameraDepthTexture
│   ├── 重建世界坐标：ComputeWorldSpacePosition(uv, depth, UNITY_MATRIX_I_VP)
│   └── 距离 → 扭曲强度映射
├── 扭曲累积（多源重叠）
│   ├── 方案A：Distortion Buffer（RGBA：RG=offset, A=intensity）
│   │   └── 所有扭曲体写入同一张 buffer → 最终一次性应用
│   ├── 方案B：逐物体 blend（additive）
│   │   └── 每个物体各自采样+扭曲，但多次 Blit 有开销
│   └── 推荐方案A：一次 Blit 完成
└── 性能优化
    ├── 半分辨率扭曲 buffer → 双线性上采样
    ├── 只在有扭曲体的区域处理（stencil mark 或 bounding box）
    └── 合理的 RT 格式（RGHalf 足够存 offset）
```

#### 代码实现

**URP RendererFeature（场景颜色捕获 + 扭曲统一应用）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class ScreenDistortionFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.BeforeRenderingTransparents;
        public Material distortionMaterial;
        public bool halfResolution = true;
    }

    public Settings settings = new Settings();
    private ScreenDistortionPass _pass;

    public override void Create()
    {
        _pass = new ScreenDistortionPass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.distortionMaterial == null) return;
        _pass.Setup(renderer.cameraColorTargetHandle, renderer.cameraDepthTargetHandle);
        renderer.EnqueuePass(_pass);
    }

    class ScreenDistortionPass : ScriptableRenderPass
    {
        private Settings _settings;
        private RTHandle _distortionBuffer;
        private Material _material;
        private RTHandle _cameraColor;
        private RTHandle _cameraDepth;

        public ScreenDistortionPass(Settings settings)
        {
            _settings = settings;
            _material = settings.distortionMaterial;
            renderPassEvent = settings.renderPassEvent;
        }

        public void Setup(RTHandle color, RTHandle depth)
        {
            _cameraColor = color;
            _cameraDepth = depth;
        }

        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            if (_settings.halfResolution)
            {
                desc.width /= 2;
                desc.height /= 2;
            }
            desc.colorFormat = RenderTextureFormat.RGHalf;
            RenderingUtils.ReAllocateIfNeeded(ref _distortionBuffer, desc, name: "_DistortionBuffer");
        }

        public override void Execute(ScriptableRenderContext ctx, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get("ScreenDistortion");
            // 这里扭曲体的材质已经在 Transparent 阶段写入了 _DistortionBuffer
            // 此 Pass 读取 distortion buffer，应用到场景
            Blitter.BlitCameraTexture(cmd, _cameraColor, _cameraColor, _material, 0);
            ctx.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

**热浪扭曲 Shader：**

```hlsl
Shader "Custom/HeatHaze"
{
    Properties
    {
        _NoiseTex ("Noise Texture", 2D) = "white" {}
        _DistortionStrength ("Distortion Strength", Range(0, 0.05)) = 0.01
        _NoiseSpeed ("Noise Speed", Float) = 1.0
        _NoiseScale ("Noise Scale", Float) = 1.0
    }
    SubShader
    {
        Tags
        {
            "RenderType"="Transparent"
            "RenderPipeline"="UniversalPipeline"
            "Queue"="Transparent"
        }

        Pass
        {
            Blend DstColor Zero, Zero One // 只写扭曲 buffer 不改颜色
            ZWrite Off
            Cull Off

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _NoiseTex_ST;
                float _DistortionStrength;
                float _NoiseSpeed;
                float _NoiseScale;
            CBUFFER_END

            TEXTURE2D(_NoiseTex); SAMPLER(sampler_NoiseTex);
            TEXTURE2D(_CameraDepthTexture); SAMPLER(sampler_CameraDepthTexture);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float2 screenUV : TEXCOORD1;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _NoiseTex);
                OUT.screenUV = OUT.positionHCS.xy / OUT.positionHCS.w;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                // 两层噪声叠加（模拟空气流动的随机性）
                float2 uv1 = IN.uv * _NoiseScale + float2(0, _Time.y * _NoiseSpeed * 0.3);
                float2 uv2 = IN.uv * _NoiseScale * 1.3 + float2(_Time.y * _NoiseSpeed * 0.2, 0);
                half noise1 = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, uv1).r;
                half noise2 = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, uv2).r;

                float2 distortion = (noise1 - 0.5) * 2.0 * _DistortionStrength;
                distortion += (noise2 - 0.5) * 1.5 * _DistortionStrength;

                // 深度衰减：远处扭曲弱
                float depth = SAMPLE_TEXTURE2D(_CameraDepthTexture, sampler_CameraDepthTexture, IN.screenUV).r;
                float linearDepth = LinearEyeDepth(depth, _ZBufferParams);
                float falloff = saturate(1.0 / (linearDepth * 0.05 + 1.0));

                distortion *= falloff;

                // 输出到扭曲 buffer（RG = offset）
                return half4(distortion, 0, 1);
            }
            ENDHLSL
        }
    }
}
```

**水晶折射 Shader（带色散）：**

```hlsl
Shader "Custom/CrystalRefraction"
{
    Properties
    {
        _BumpMap ("Normal Map", 2D) = "bump" {}
        _RefractionStrength ("Refraction Strength", Range(0, 0.1)) = 0.02
        _ChromaticAberration ("Chromatic Aberration", Range(0, 0.01)) = 0.002
        _Tint ("Tint Color", Color) = (0.9, 0.95, 1.0, 1.0)
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BumpMap_ST;
                float _RefractionStrength;
                float _ChromaticAberration;
                float4 _Tint;
            CBUFFER_END

            TEXTURE2D(_BumpMap); SAMPLER(sampler_BumpMap);
            TEXTURE2D(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
                float3 normalOS : NORMAL;
                float4 tangentOS : TANGENT;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float4 screenPos : TEXCOORD1;
                float3 normalWS : TEXCOORD2;
                float3 tangentWS : TEXCOORD3;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BumpMap);
                OUT.screenPos = ComputeScreenPos(OUT.positionHCS);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                OUT.tangentWS = TransformObjectToWorldDir(IN.tangentOS.xyz);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                // 屏幕坐标
                float2 screenUV = IN.screenPos.xy / IN.screenPos.w;

                // 采样法线贴图（切线空间 → 屏幕空间方向）
                half3 normalTS = UnpackNormal(SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, IN.uv));
                float2 normalScreen = normalTS.xy * _RefractionStrength;

                // 色散：R/G/B 分别用不同偏移量
                float2 offsetR = normalScreen * (1.0 + _ChromaticAberration * 2.0);
                float2 offsetG = normalScreen;
                float2 offsetB = normalScreen * (1.0 - _ChromaticAberration * 2.0);

                half r = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, screenUV + offsetR).r;
                half g = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, screenUV + offsetG).g;
                half b = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, screenUV + offsetB).b;

                half3 color = half3(r, g, b) * _Tint.rgb;

                // 菲涅尔边缘增亮
                float3 viewDir = normalize(GetWorldSpaceViewDir(IN.positionHCS));
                float fresnel = pow(1.0 - saturate(dot(IN.normalWS, viewDir)), 3.0);
                color += fresnel * 0.3;

                return half4(color, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**对比表：URP 获取场景颜色方案**

| 方案 | 实现复杂度 | 性能 | 灵活性 | 推荐度 |
|------|-----------|------|--------|--------|
| Opaque Texture（内置） | ★ | ★★★★ | ★★ | 移动端首选 |
| Custom Blit Pass | ★★★ | ★★★ | ★★★★ | 高端需求 |
| _CameraColorTexture (URP 17+) | ★ | ★★★★ | ★★★ | 新项目推荐 |
| RenderTargetHandle 中间 RT | ★★ | ★★★ | ★★★★ | 兼容性好 |

### ⚡ 实战经验

- **Opaque Texture 的坑**：开启后 URP 会在不透明渲染完成后自动 Blit 一次到 `_CameraOpaqueTexture`，有 ~0.3ms 开销，移动端要注意
- **半透明物体无法被折射**：Opaque Texture 只包含不透明物体。如果需要折射半透明物体，需要用 Custom Pass 在 Transparent 阶段做额外 Blit
- **色散让水晶更真实**：RGB 通道用不同偏移量采样，成本几乎为零但效果显著提升
- **热浪深度衰减不能省**：不衰减的话远处扭曲和近处一样强，看起来很假。用 `1/(depth * k + 1)` 做简单衰减即可
- **多扭曲源冲突**：如果同时有热浪和水晶，各自的 offset 可能互相覆盖。用 Distortion Buffer 统一收集（additive blend），最后一次性应用
- **性能红线**：半分辨率扭曲 buffer + 全分辨率应用采样，在骁龙 888 上 1080p 约 0.8ms

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 URP 怎么拿场景颜色 | URP 无 GrabPass | 学习 Opaque Texture + Custom Pass |
| 扭曲看起来像「贴图滑动」不真实 | 缺少深度感知 | 学习 Depth Buffer 重建世界坐标 |
| 水晶没有色散效果 | 色散原理（Chromatic Aberration） | 理解不同波长折射率差异 |
| 多个扭曲体重叠时效果错乱 | 半透明混合 / Distortion Buffer | 学习 RT blend mode + 累积方案 |
| 不知道在哪里注入 Pass | URP Render Pass Event 时机 | 复习 URP 渲染阶段顺序 |
| 热浪在远处也扭曲，很假 | 距离衰减 | 理解屏幕空间效果与深度的关系 |

### 🔗 相关问题

- 角色半透明斗篷需要折射背后的场景，和水晶折射有什么不同？（提示：排序问题 + 是否需要深度写入）
- 如何用 Compute Shader 加速热浪噪声计算？（提示：分 tile 计算噪声 → 共享内存优化）
- 后处理中的 Lens Distortion（镜头畸变）和屏幕空间扭曲有什么联系和区别？
