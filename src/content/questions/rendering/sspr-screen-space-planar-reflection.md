---
title: "URP 下屏幕空间平面反射（SSPR）：如何让水洼和湿润地面实时反射周围画面？"
category: "rendering"
level: 4
tags: ["SSPR", "屏幕空间反射", "Planar Reflection", "URP", "Renderer Feature", "后处理", "水面"]
hint: "核心是「对屏幕像素做平面镜像翻转后的重投影」——不需要真渲染一面镜子，而是在后处理阶段利用已有颜色缓冲"
related: ["rendering/urp-renderer-feature", "rendering/custom-post-processing-urp", "shader/water-caustics"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一个赛博朋克风格的城市游戏，雨后的地面需要能实时反射周围霓虹灯和角色。传统的 Planar Reflection 太贵（等于多渲染一遍场景），SSR 又只能反射屏幕几何体。你要怎么在 URP 下用屏幕空间平面反射（SSPR）实现这个效果？移动端也要能跑。」

### ✅ 核心要点

1. **屏幕空间重投影**：不额外渲染场景，而是对已有颜色帧做「平面镜像翻转 + 重投影采样」
2. **Hit Buffer 生成**：通过 Compute Shader 或 Fragment Pass 计算每个像素的反射 hit point
3. **模糊与扰动**：对反射结果做模糊（模拟粗糙表面）和 UV 扰动（模拟水波）
4. **高度遮罩驱动**：只在地面积水/湿润区域生效，通过 Height Mask 或 Roughness Map 控制
5. **性能控制**：半分辨率计算 + 时间复用 + 距离衰减，移动端可接受

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：雨后地面反射霓虹灯/角色，越远越模糊，有水波扭曲
              ↑
倒推1：反射颜色从哪来？→ 对当前帧颜色缓冲做平面镜像翻转
倒推2：每个屏幕像素对应哪个反射像素？→ Hit Buffer
        ├── 像素的世界坐标 (x, y, z)
        ├── 沿镜面反射方向追踪到地面平面
        └── 反射点投影回屏幕 → 对应的颜色UV
倒推3：怎么知道哪里有积水？→ Wetness Mask（材质或全局）
倒推4：怎么做出模糊和扭曲？→
        ├── 模糊：对反射采样做 blur（高斯 / 双 pass）
        └── 扭曲：用法线扰动反射UV
倒推5：怎么控制性能？→ 半分辨率 RT + 距离衰减 + Compute Shader
```

#### 知识点拆解（倒推树）

```
SSPR 屏幕空间平面反射
├── 理论基础
│   ├── 平面反射原理（Reflect = Eye → Plane → Symmetric Point）
│   ├── 屏幕空间重投影（世界坐标 → 镜像翻转 → 投影回屏幕）
│   └── 与 SSR 的区别
│       ├── SSR：基于法线反射，需要 G-Buffer Normal
│       └── SSPR：基于固定平面（通常是地面），只需高度
│
├── Hit Buffer 生成（核心 Pass）
│   ├── Fragment Shader 方案（移动端友好）
│   │   ├── 输入：深度缓冲 → 重建世界坐标
│   │   ├── 镜面翻转：worldPos.y = 2 * planeHeight - worldPos.y
│   │   └── 重投影：Project(viewProj, mirroredPos) → 屏幕UV
│   ├── Compute Shader 方案（高端机）
│   │   └── Hi-Z Tracing 加速（步进深度金字塔）
│   └── 屏幕外裁剪：反射点在屏幕外的区域标记 miss
│
├── 反射合成
│   ├── 颜色采样：Camera Opaque Texture @ hitUV
│   ├── 距离衰减：反射距离越远，强度越低
│   │   └── falloff = 1 - saturate(dist / maxDist)
│   ├── 模糊层
│   │   ├── 方案A：降采样 + 高斯模糊（双 pass H+V）
│   │   ├── 方案B：Poisson Disk 采样（移动端省带宽）
│   │   └── 方案C：Hi-Z Mipmap 采样（一次查询近似模糊）
│   └── UV 扰动（水面波纹）
│       └── 法线图驱动偏移：hitUV += normalMap.rg * distortion
│
├── 遮罩系统
│   ├── Wetness Mask（哪里有水）
│   │   ├── 方案A：材质 Roughness/Metallic 通道驱动
│   │   ├── 方案B：全局雨水量（Rain Wetness 全局参数）
│   │   └── 方案C：Decal 投影积水区域
│   ├── Fresnel 增强（掠射角反射更强）
│   │   └── reflectionStrength *= pow(1 - NdotV, 5)
│   └── 边缘消隐（屏幕外反射 → 淡出）
│
├── URP 集成
│   ├── Custom Renderer Feature
│   │   ├── Pass 1: RenderObjects → 渲染场景到 _CameraOpaqueTexture
│   │   ├── Pass 2: Blit → Compute Hit Buffer
│   │   ├── Pass 3: Blit → Blur反射图
│   │   └── Pass 4: Blit → 合成到最终画面
│   ├── RT 管理（半分辨率 + Format选择）
│   │   ├── Hit Buffer: ARGBHalf（精度需求）
│   │   └── Reflection Color: ARGB32（颜色，可低精度）
│   └── Shader关键词控制：_SSPR_QUALITY_LOW / MEDIUM / HIGH
│
└── 性能优化
    ├── 半分辨率渲染（Hit Buffer + Reflection Color 都用 0.5x）
    ├── 时间复用：上一帧的反射结果做时域滤波（类似 TAA）
    ├── 距离剔除：超过 N 米的像素直接 skip
    └── 分机型策略
        ├── 旗舰机：Compute Shader + Hi-Z + 全分辨率
        ├── 中端机：Fragment + 半分辨率 + Poisson Blur
        └── 低端机：降级为 Cubemap 假反射
```

#### 代码实现

**URP Renderer Feature — SSPR 设置：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class SSPRFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class SSPRSettings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        public float resolutionScale = 0.5f;       // Hit Buffer 分辨率缩放
        public float maxReflectionDistance = 50f;   // 最大反射距离
        public float planeHeight = 0f;              // 反射平面高度（Y轴）
        public float blurStrength = 1.5f;
        [Range(0, 4)] public int blurIterations = 2;
        public bool enableDistortion = true;
    }

    public SSPRSettings settings = new SSPRSettings();
    private SSPRPass _pass;

    public override void Create()
    {
        _pass = new SSPRPass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (renderingData.cameraData.cameraType == CameraType.Preview) return;
        renderer.EnqueuePass(_pass);
    }
}
```

**SSPR Pass — Hit Buffer 计算 + 合成：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class SSPRPass : ScriptableRenderPass
{
    private SSPRFeature.SSPRSettings _settings;
    private Material _ssprMaterial;
    private RTHandle _hitBuffer;
    private RTHandle _reflectionColor;
    private RTHandle _tempBlur;

    private static readonly int PlaneHeightID = Shader.PropertyToID("_PlaneHeight");
    private static readonly int MaxDistID = Shader.PropertyToID("_MaxReflectionDistance");
    private static readonly int BlurStrengthID = Shader.PropertyToID("_BlurStrength");
    private static readonly int HitBufferID = Shader.PropertyToID("_SSPRHitBuffer");
    private static readonly int ReflectionTexID = Shader.PropertyToID("_SSPRReflectionTex");

    public SSPRPass(SSPRFeature.SSPRSettings settings)
    {
        _settings = settings;
        renderPassEvent = settings.renderPassEvent;
        _ssprMaterial = CoreUtils.CreateEngineMaterial(Shader.Find("Hidden/SSPR"));
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        desc.width = (int)(desc.width * _settings.resolutionScale);
        desc.height = (int)(desc.height * _settings.resolutionScale);
        desc.colorFormat = RenderTextureFormat.ARGBHalf;
        desc.depthBufferBits = 0;

        RenderingUtils.ReAllocateIfNeeded(ref _hitBuffer, desc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_SSPRHitBuffer");
        desc.colorFormat = RenderTextureFormat.ARGB32;
        RenderingUtils.ReAllocateIfNeeded(ref _reflectionColor, desc, FilterMode.Bilinear, Texture.WrapMode.Clamp, name: "_SSPRReflectionTex");
        RenderingUtils.ReAllocateIfNeeded(ref _tempBlur, desc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_SSPRTempBlur");
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        var cmd = CommandBufferPool.Get("SSPR");

        // === Pass 1: 计算 Hit Buffer（每个像素的反射UV） ===
        _ssprMaterial.SetFloat(PlaneHeightID, _settings.planeHeight);
        _ssprMaterial.SetFloat(MaxDistID, _settings.maxReflectionDistance);
        Blit(cmd, renderingData.cameraData.renderer.cameraColorTargetHandle, _hitBuffer, _ssprMaterial, 0);

        // === Pass 2: 采样反射颜色 ===
        cmd.SetGlobalTexture(HitBufferID, _hitBuffer);
        Blit(cmd, renderingData.cameraData.renderer.cameraColorTargetHandle, _reflectionColor, _ssprMaterial, 1);

        // === Pass 3: 模糊（多次迭代） ===
        _ssprMaterial.SetFloat(BlurStrengthID, _settings.blurStrength);
        var src = _reflectionColor;
        for (int i = 0; i < _settings.blurIterations; i++)
        {
            Blit(cmd, src, _tempBlur, _ssprMaterial, 2); // Horizontal blur
            Blit(cmd, _tempBlur, src, _ssprMaterial, 3); // Vertical blur
        }

        // === Pass 4: 合成到最终画面 ===
        cmd.SetGlobalTexture(ReflectionTexID, _reflectionColor);
        Blit(cmd, renderingData.cameraData.renderer.cameraColorTargetHandle,
             renderingData.cameraData.renderer.cameraColorTargetHandle, _ssprMaterial, 4);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public override void OnCameraCleanup(CommandBuffer cmd)
    {
        // RTHandle 由 ReAllocateIfNeeded 管理
    }
}
```

**SSPR Shader — Hit Buffer Pass（HLSL）：**

```hlsl
Shader "Hidden/SSPR"
{
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }

        // === Pass 0: Hit Buffer 计算 ===
        Pass
        {
            Name "SSPRHitBuffer"

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag_hit

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            float _PlaneHeight;
            float _MaxReflectionDistance;

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 frag_hit(Varyings IN) : SV_Target
            {
                // 1. 重建世界坐标（从深度缓冲）
                float2 uv = IN.uv;
                #if UNITY_UV_STARTS_AT_TOP
                    uv.y = 1.0 - uv.y;
                #endif

                float depth = SampleSceneDepth(uv);
                float3 worldPos = ComputeWorldSpacePosition(uv, depth, UNITY_MATRIX_I_VP);

                // 2. 只处理平面以上的像素（水洼反射地面以上的物体）
                if (worldPos.y < _PlaneHeight) {
                    return half4(0, 0, 0, 0); // miss
                }

                // 3. 平面镜像翻转（沿 Y = _PlaneHeight 翻转）
                float3 mirroredPos = worldPos;
                mirroredPos.y = 2.0 * _PlaneHeight - worldPos.y;

                // 4. 反射距离裁剪
                float reflDist = abs(worldPos.y - _PlaneHeight) * 2.0;
                if (reflDist > _MaxReflectionDistance) {
                    return half4(0, 0, 0, 0); // 太远，跳过
                }

                // 5. 将镜像后的位置投影回屏幕
                float4 clipPos = mul(UNITY_MATRIX_VP, float4(mirroredPos, 1.0));
                float3 ndcPos = clipPos.xyz / clipPos.w;

                // 6. 检查是否在屏幕内
                bool offscreen = any(ndcPos.xy < -1.0) || any(ndcPos.xy > 1.0);
                if (offscreen || ndcPos.z < 0.0) {
                    return half4(0, 0, 0, 0); // miss
                }

                // 7. 输出：屏幕UV + 距离衰减因子
                float2 screenUV = ndcPos.xy * 0.5 + 0.5;
                float falloff = 1.0 - saturate(reflDist / _MaxReflectionDistance);

                return half4(screenUV.xy, reflDist, falloff);
            }
            ENDHLSL
        }

        // === Pass 1: 采样反射颜色 ===
        Pass
        {
            Name "SSPRSampleColor"

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag_sample

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_SSPRHitBuffer); SAMPLER(sampler_SSPRHitBuffer);
            TEXTURE2D_X(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings vert(Attributes IN) {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 frag_sample(Varyings IN) : SV_Target
            {
                half4 hitData = SAMPLE_TEXTURE2D(_SSPRHitBuffer, sampler_SSPRHitBuffer, IN.uv);
                
                // alpha = 0 表示 miss（屏幕外或距离太远）
                if (hitData.a < 0.01) discard;

                // 用 Hit Buffer 中的 UV 采样场景颜色
                float2 reflUV = hitData.xy;
                #if UNITY_UV_STARTS_AT_TOP
                    reflUV.y = 1.0 - reflUV.y;
                #endif

                half3 reflColor = SAMPLE_TEXTURE2D_X(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, reflUV).rgb;
                
                // 距离衰减写入 alpha 通道
                return half4(reflColor * hitData.a, hitData.a);
            }
            ENDHLSL
        }

        // === Pass 2: Horizontal Blur ===
        Pass
        {
            Name "SSPRBlurH"
            HLSLPROGRAM
            #pragma vertex vert_blur
            #pragma fragment frag_blurH
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_SSPRReflectionTex); SAMPLER(sampler_SSPRReflectionTex);
            float _BlurStrength;
            float4 _SSPRReflectionTex_TexelSize;

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings vert_blur(float4 positionOS : POSITION, float2 uv : TEXCOORD0) {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(positionOS.xyz);
                OUT.uv = uv;
                return OUT;
            }

            // 5-tap 高斯模糊（水平方向）
            static const float weights[5] = { 0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216 };
            
            half4 frag_blurH(Varyings IN) : SV_Target {
                half4 col = SAMPLE_TEXTURE2D(_SSPRReflectionTex, sampler_SSPRReflectionTex, IN.uv) * weights[0];
                float2 offset = float2(_SSPRReflectionTex_TexelSize.x * _BlurStrength, 0);
                
                [unroll]
                for (int i = 1; i < 5; i++) {
                    col += SAMPLE_TEXTURE2D(_SSPRReflectionTex, sampler_SSPRReflectionTex, IN.uv + offset * i) * weights[i];
                    col += SAMPLE_TEXTURE2D(_SSPRReflectionTex, sampler_SSPRReflectionTex, IN.uv - offset * i) * weights[i];
                }
                return col;
            }
            ENDHLSL
        }

        // === Pass 3: Vertical Blur（结构同上，offset.y）===
        // ...（省略，与 Pass 2 对称）

        // === Pass 4: 最终合成 ===
        Pass
        {
            Name "SSPRComposite"

            HLSLPROGRAM
            #pragma vertex vert_composite
            #pragma fragment frag_composite
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_SSPRReflectionTex); SAMPLER(sampler_SSPRReflectionTex);
            TEXTURE2D_X(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings vert_composite(float4 positionOS : POSITION, float2 uv : TEXCOORD0) {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(positionOS.xyz);
                OUT.uv = uv;
                return OUT;
            }

            half4 frag_composite(Varyings IN) : SV_Target {
                half4 sceneColor = SAMPLE_TEXTURE2D_X(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, IN.uv);
                half4 reflColor = SAMPLE_TEXTURE2D(_SSPRReflectionTex, sampler_SSPRReflectionTex, IN.uv);

                // alpha 混合：reflColor.a 是距离衰减
                // 这里可以用 Wetness Mask 替代固定 lerp
                float blendFactor = reflColor.a * 0.6; // 可调参数

                half3 finalColor = lerp(sceneColor.rgb, reflColor.rgb, blendFactor);
                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**方案对比表：SSPR vs 传统 Planar Reflection vs SSR**

| 方案 | 性能（移动端） | 反射质量 | 屏幕外反射 | 实现难度 | 适用场景 |
|------|---------------|---------|-----------|---------|---------|
| **传统 Planar Reflection** | 差（×2 Draw Call） | 最好 | 支持 | 中 | 高端PC/主机 |
| **SSPR（屏幕空间）** | 中（后处理开销） | 好（屏幕内有） | 不支持 | 高 | 手游主流方案 |
| **SSR（屏幕空间射线）** | 差（射线步进） | 好（曲面可用） | 不支持 | 高 | 非平面反射 |
| **Cubemap 假反射** | 最好 | 差（静态） | 支持 | 低 | 低端机降级 |
| **Hi-Z SSPR** | 较好（Compute） | 好 | 不支持 | 很高 | 旗舰机 |

### ⚡ 实战经验

1. **半分辨率是底线**：Hit Buffer + Reflection Color 都用 0.5x 分辨率，模糊后几乎看不出区别，但省 4 倍带宽
2. **闪烁问题**：屏幕外反射区域在移动时会出现闪烁（hit/miss 频繁切换），需要做时域平滑——保留上一帧 Hit Buffer 做 2 帧混合
3. **积水遮罩用 Decal**：不要用材质的 Roughness 驱动（需要改所有材质），用 Decal Projector 投影积水区域更灵活，可以动态生成雨后水洼
4. **性能分档**：低端机直接降级为预渲染 Cubemap 反射，中端机用 Fragment 方案 + Poisson Blur，旗舰机才上 Compute Shader

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么从深度重建世界坐标 | 深度缓冲原理 | 学逆投影变换（Inverse View-Projection） |
| 反射结果有黑边/破面 | 屏幕外裁剪处理 | 检查 NDC 边界、Hit Buffer miss 标记 |
| 反射在移动时闪烁 | 时域稳定性 | TAA 原理 + Hit Buffer 时域混合 |
| 不知道怎么在 URP 搭多 Pass | Renderer Feature 架构 | URP Render Pass 生命周期 + RTHandle 管理 |
| 移动端跑不动 | 带宽/ALU 预算 | 半分辨率 + Compute→Fragment 降级 + Blur 简化 |

### 🔗 相关问题

- SSR（屏幕空间反射）和 SSPR 有什么区别？什么时候用哪个？
- 如何用 Compute Shader 加速 SSPR 的 Hit Buffer 计算？
- 雨天全局湿润效果怎么做？（Wetness 系统 + Roughness 调制）
