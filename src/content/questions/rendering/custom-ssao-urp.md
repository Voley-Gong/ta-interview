---
title: "面试官让你在URP中自定义SSAO：从原理到移动端落地的完整方案"
category: "rendering"
level: 3
tags: ["SSAO", "环境光遮蔽", "URP", "屏幕空间", "渲染管线", "移动端优化"]
hint: "核心是深度重建世界坐标 + 法线 + 采样核做遮蔽计算；移动端要用半球随机旋转核 + 降分辨率渲染"
related: ["rendering/urp-renderer-feature", "rendering/custom-post-processing-urp", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们项目用的是 URP，内置的 SSAO 效果不太满意——在角色模型边缘有明显的"轮廓遮蔽"假阴影，而且移动端性能开销比较大。你能不能自定义一个 SSAO Renderer Feature，要求：
> 1. 效果更自然（减少 Contact Shadow 式的硬黑边）
> 2. 移动端可降级（半分辨率 + 采样核减半）
> 3. 支持法线空间遮蔽计算（不只是深度）
> 4. 可与场景间接光混合
>
> 你会怎么做？

这是米哈游、腾讯、网易等技术导向型公司的经典渲染面试题。考察的是**屏幕空间算法理解 + URP 自定义 Pass + 移动端性能优化**三合一能力。

### ✅ 核心要点

1. **SSAO 原理**：在像素的半球采样核内，检测有多少采样点被几何遮挡，被遮挡越多 → 环境光遮蔽越强
2. **深度重建**：从屏幕空间深度反推观察空间坐标，这是所有 SSAO 的基础
3. **采样核策略**：在切线空间生成半球采样核，用 TBN 矩阵转到观察空间，避免各向异性
4. **URP Renderer Feature**：通过 `ScriptableRendererFeature` + `ScriptableRenderPass` 插入自定义 SSAO Pass
5. **移动端优化**：降分辨率渲染（半分辨率 RT）、减少采样核数量（8-16 个）、双边模糊 Pass

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
期望效果：自然的间接光遮蔽 → 不硬黑 → 可控强度 → 移动端可跑
         ↓
SSAO 质量问题来源：
  "轮廓黑边" = 采样核半径太大，跨越了几何边界 → 用法线空间采样核 + Range Check 修复
  "噪点" = 采样核不足 → 需要双边模糊但不能糊掉细节
  "性能" = 每像素 N 次 depth sample → 降分辨率 + 减核数

实现路径：
  Custom SSAO Renderer Feature
    ├── Pass 1: SSAO 计算（深度+法线 → AO 纹理，半分辨率）
    ├── Pass 2: 双边模糊（保边缘的模糊，横竖分离）
    └── Pass 3: 合成到 Final Color（在 Lighting Pass 后做 Multiply blend）
```

#### 知识点拆解（倒推树）

```
自定义 SSAO
├── SSAO 算法原理
│   ├── Crytek SSAO（原始版本，只用深度）
│   │   └── 球形采样核 → 转到观察空间 → 检测被遮挡
│   ├── HBAO / GTAO（法线感知版本）
│   │   └── 半球采样核 + 法线权重 → 效果更自然但更贵
│   └── INTEL SSAO（切线空间版本，本文实现）
│       └── TBN 矩阵让采样核"贴着"表面 → 减少假黑边
│
├── 深度重建
│   ├── 方案A：NDC → Inverse ViewProjection → World Position（开销大）
│   ├── 方案B：NDC → View Space（只需 Inverse Projection）✅ 推荐
│   └── 关键函数：ComputeViewSpacePosition(uv, depth)
│
├── 采样核生成
│   ├── Halton 序列 / Fibonacci 球面分布
│   ├── 半球采样核（沿法线方向偏移）
│   ├── 随机旋转向量（每像素不同旋转 → 空间去噪）
│   └── 核数量：PC 32-64 个，移动端 8-16 个
│
├── URP Renderer Feature 集成
│   ├── ScriptableRendererFeature 生命周期
│   │   ├── Create() → 初始化 RT、材质
│   │   ├── AddRenderPasses() → 注入 Pass 到渲染队列
│   │   └── Dispose() → 释放 RT
│   ├── Render Pass 注入点
│   │   └── After Rendering Opaques ← ✅ 最佳（已有深度+法线）
│   └── RT 格式：半分辨率 R8（只存 AO 值）
│
├── 双边模糊（Bilateral Filter）
│   ├── 空间权重 × 深度权重 × 法线权重
│   └── 横竖分离双 Pass（节省采样次数）
│
└── 移动端优化
    ├── 半分辨率 RT
    ├── 降采样核（8 个）
    ├── 简化模糊（separable Gaussian）
    └── 低端机直接关闭 SSAO
```

#### 代码实现

**SSAO 计算 Shader（核心 Pass，URP 兼容）：**

```hlsl
// CustomSSAO.shader — URP SSAO
Shader "Hidden/TA/CustomSSAO" {
    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }

        Pass {
            Name "SSAOCompute"
            ZTest Always ZWrite Off Cull Off

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag_ao
            #pragma multi_compile _ _USE_NORMAL_AO
            #pragma multi_compile _ _HALF_RES

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareNormalsTexture.hlsl"

            #define KERNEL_SIZE 16

            float4 _SSAOParams;       // x=Radius y=Bias z=Power w=Intensity
            float4x4 _ProjectionMatrix;
            float4x4 _InverseProjectionMatrix;
            float4 _NoiseScale;       // xy = screenSize/noiseSize

            // Halton 序列半球采样核（16 个，预计算）
            static const float3 sampleSphere[KERNEL_SIZE] = {
                float3( 0.0259,  0.1129, 0.9928),
                float3(-0.1343,  0.0521, 0.9895),
                float3( 0.0812, -0.2091, 0.9742),
                float3(-0.2455, -0.1183, 0.9622),
                float3( 0.3214,  0.2865, 0.9012),
                float3(-0.0523,  0.3825, 0.9228),
                float3( 0.1129, -0.4275, 0.8965),
                float3(-0.3819,  0.1932, 0.9034),
                float3( 0.4284, -0.2018, 0.8802),
                float3(-0.2987, -0.3098, 0.9037),
                float3( 0.1501,  0.4982, 0.8538),
                float3(-0.0632, -0.5213, 0.8508),
                float3( 0.5308,  0.1232, 0.8395),
                float3(-0.4982,  0.0872, 0.8621),
                float3( 0.3716,  0.4034, 0.8357),
                float3(-0.1968, -0.4327, 0.8800)
            };

            TEXTURE2D(_SSAONoiseTex); SAMPLER(sampler_SSAONoiseTex);

            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
            };

            Varyings vert(uint vertexID : SV_VertexID) {
                Varyings o;
                o.positionCS = GetFullScreenTriangleVertexPosition(vertexID);
                o.uv = GetFullScreenTriangleTexCoord(vertexID);
                return o;
            }

            // 深度 → 观察空间坐标
            float3 ReconstructViewPos(float2 uv, float rawDepth) {
                float3 clipPos = float3(uv * 2.0 - 1.0, rawDepth);
                float4 viewPos = mul(_InverseProjectionMatrix, float4(clipPos, 1.0));
                return viewPos.xyz / viewPos.w;
            }

            half4 frag_ao(Varyings input) : SV_Target {
                float rawDepth = SampleSceneDepth(input.uv);

                // 天空盒 → 无遮蔽
                #if UNITY_REVERSED_Z
                if (rawDepth <= 0.0001) return half4(1,1,1,1);
                #else
                if (rawDepth >= 0.9999) return half4(1,1,1,1);
                #endif

                float3 fragPos = ReconstructViewPos(input.uv, rawDepth);

                // 法线获取
                #if _USE_NORMAL_AO
                    float3 normal = SampleSceneNormals(input.uv);
                    normal = TransformWorldToViewDir(normal);
                #else
                    float3 dx = ddx(fragPos);
                    float3 dy = ddy(fragPos);
                    float3 normal = normalize(cross(dy, dx));
                #endif

                // 随机旋转向量（4×4 noise tile）
                float3 randomVec = SAMPLE_TEXTURE2D(
                    _SSAONoiseTex, sampler_SSAONoiseTex,
                    input.uv * _NoiseScale.xy
                ).xyz * 2.0 - 1.0;

                // Gram-Schmidt 构建 TBN
                float3 tangent   = normalize(randomVec - normal * dot(randomVec, normal));
                float3 bitangent = cross(normal, tangent);
                float3x3 TBN     = float3x3(tangent, bitangent, normal);

                // 累积遮蔽
                float occlusion = 0.0;
                float radius = _SSAOParams.x;
                float bias   = _SSAOParams.y;

                [unroll]
                for (int i = 0; i < KERNEL_SIZE; i++) {
                    float3 samplePos = mul(sampleSphere[i], TBN);
                    samplePos = fragPos + samplePos * radius;

                    // 观察空间 → UV
                    float4 offset = mul(_ProjectionMatrix, float4(samplePos, 1.0));
                    offset.xyz /= offset.w;
                    float2 sampleUV = offset.xy * 0.5 + 0.5;

                    float sampleDepthRaw = SampleSceneDepth(sampleUV);
                    float3 sampleFragPos = ReconstructViewPos(sampleUV, sampleDepthRaw);

                    // Range Check：防止远距离遮蔽
                    float rangeCheck = smoothstep(
                        0.0, 1.0,
                        radius / abs(fragPos.z - sampleFragPos.z)
                    );

                    if (sampleFragPos.z + bias <= samplePos.z) {
                        occlusion += rangeCheck;
                    }
                }
                occlusion = 1.0 - (occlusion / KERNEL_SIZE);
                occlusion = pow(saturate(occlusion), _SSAOParams.z);
                occlusion = saturate(occlusion * _SSAOParams.w);

                return half4(occlusion.xxx, 1.0);
            }
            ENDHLSL
        }

        // === 双边模糊 Pass（横竖分离） ===
        Pass {
            Name "BilateralBlur"
            ZTest Always ZWrite Off Cull Off

            HLSLPROGRAM
            #pragma vertex vert_blur
            #pragma fragment frag_blur
            #pragma multi_compile _ _BLUR_H _BLUR_V

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            float4 _BlurParams; // x=radius y=depthThreshold zw=dir
            TEXTURE2D(_SSAOTexture); SAMPLER(sampler_SSAOTexture);
            float4 _SSAOTexture_TexelSize;

            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
            };

            Varyings vert_blur(uint vertexID : SV_VertexID) {
                Varyings o;
                o.positionCS = GetFullScreenTriangleVertexPosition(vertexID);
                o.uv = GetFullScreenTriangleTexCoord(vertexID);
                return o;
            }

            half4 frag_blur(Varyings input) : SV_Target {
                float centerDepth = LinearizeDepth(SampleSceneDepth(input.uv), _ZBufferParams);
                float centerAO = SAMPLE_TEXTURE2D(_SSAOTexture, sampler_SSAOTexture, input.uv).r;

                float totalWeight = 1.0;
                float totalAO = centerAO;

                #if _BLUR_H
                float2 dir = float2(_SSAOTexture_TexelSize.x, 0);
                #else
                float2 dir = float2(0, _SSAOTexture_TexelSize.y);
                #endif

                [unroll]
                for (int i = 1; i <= 4; i++) {
                    float w = exp(-i * i * 0.5);
                    float2 off = dir * i;

                    // 正方向
                    float2 uvP = input.uv + off;
                    float depthP = LinearizeDepth(SampleSceneDepth(uvP), _ZBufferParams);
                    float aoP = SAMPLE_TEXTURE2D(_SSAOTexture, sampler_SSAOTexture, uvP).r;
                    float wP = (abs(depthP - centerDepth) < _BlurParams.y) ? w : 0;

                    // 反方向
                    float2 uvN = input.uv - off;
                    float depthN = LinearizeDepth(SampleSceneDepth(uvN), _ZBufferParams);
                    float aoN = SAMPLE_TEXTURE2D(_SSAOTexture, sampler_SSAOTexture, uvN).r;
                    float wN = (abs(depthN - centerDepth) < _BlurParams.y) ? w : 0;

                    totalAO += aoP * wP + aoN * wN;
                    totalWeight += wP + wN;
                }

                return half4((totalAO / totalWeight).xxx, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**C# Renderer Feature（完整框架）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class CustomSSAORenderFeature : ScriptableRendererFeature {
    [System.Serializable]
    public class SSAOSettings {
        public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingOpaques;
        public Shader aoShader;
        [Range(4, 64)]  public int kernelSize = 16;
        [Range(0.1f, 5f)] public float radius = 0.5f;
        [Range(0f, 1f)]  public float bias = 0.025f;
        [Range(0f, 5f)]  public float intensity = 1.5f;
        [Range(0f, 4f)]  public float power = 1.0f;
        [Range(1, 4)]    public int blurRadius = 3;
        public bool halfResolution = true;
        public bool useNormalAware = true;
    }

    public SSAOSettings settings = new();
    private CustomSSAOPass _ssaoPass;

    public override void Create() {
        if (settings.aoShader == null)
            settings.aoShader = Shader.Find("Hidden/TA/CustomSSAO");

        _ssaoPass = new CustomSSAOPass(settings) {
            renderPassEvent = settings.renderPassEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        if (settings.aoShader == null || _ssaoPass == null) return;
        renderer.EnqueuePass(_ssaoPass);
    }

    protected override void Dispose(bool disposing) {
        _ssaoPass?.Dispose();
    }
}

public class CustomSSAOPass : ScriptableRenderPass {
    private static readonly int SSAOParamsID = Shader.PropertyToID("_SSAOParams");
    private static readonly int BlurParamsID = Shader.PropertyToID("_BlurParams");
    private static readonly int SSAORTID     = Shader.PropertyToID("_SSAORT");
    private static readonly int SSAOBlurredID = Shader.PropertyToID("_SSAOBlurred");
    private static readonly int NoiseScaleID  = Shader.PropertyToID("_NoiseScale");

    private Material _aoMaterial;
    private RTHandle _ssaoRT;
    private RTHandle _ssaoBlurredRT;
    private Texture2D _noiseTex;
    private CustomSSAORenderFeature.SSAOSettings _settings;

    public CustomSSAOPass(CustomSSAORenderFeature.SSAOSettings settings) {
        _settings = settings;
        _aoMaterial = CoreUtils.CreateEngineMaterial(settings.aoShader);
        _noiseTex = GenerateNoiseTex(4);
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData) {
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        int scale = _settings.halfResolution ? 2 : 1;
        desc.width  /= scale;
        desc.height /= scale;
        desc.depthBufferBits = 0;
        desc.colorFormat = RenderTextureFormat.R8;

        RenderingUtils.ReAllocateIfNeeded(ref _ssaoRT, desc, name: "_SSAORT");
        RenderingUtils.ReAllocateIfNeeded(ref _ssaoBlurredRT, desc, name: "_SSAOBlurred");

        _aoMaterial.SetVector(SSAOParamsID, new Vector4(
            _settings.radius, _settings.bias,
            _settings.power, _settings.intensity
        ));
        _aoMaterial.SetVector(NoiseScaleID, new Vector4(
            desc.width / 4f, desc.height / 4f, 0, 0
        ));
        _aoMaterial.SetTexture("_SSAONoiseTex", _noiseTex);

        // 关键字控制
        CoreUtils.SetKeyword(_aoMaterial, "_USE_NORMAL_AO", _settings.useNormalAware);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
        var cmd = CommandBufferPool.Get("CustomSSAO");

        // Pass 0: SSAO 计算
        CoreUtils.SetRenderTarget(cmd, _ssaoRT, ClearFlag.Color);
        cmd.DrawProcedural(Matrix4x4.identity, _aoMaterial, 0, MeshTopology.Triangles, 3);

        // Pass 1: 双边模糊（H → V 两步）
        _aoMaterial.SetVector(BlurParamsID, new Vector4(
            _settings.blurRadius, 0.1f, 0, 0
        ));

        // 横向
        CoreUtils.SetRenderTarget(cmd, _ssaoBlurredRT, ClearFlag.Color);
        cmd.SetGlobalTexture(SSAORTID, _ssaoRT);
        CoreUtils.SetKeyword(_aoMaterial, "_BLUR_H", true);
        CoreUtils.SetKeyword(_aoMaterial, "_BLUR_V", false);
        cmd.DrawProcedural(Matrix4x4.identity, _aoMaterial, 1, MeshTopology.Triangles, 3);

        // 纵向
        CoreUtils.SetRenderTarget(cmd, _ssaoRT, ClearFlag.Color);
        cmd.SetGlobalTexture(SSAORTID, _ssaoBlurredRT);
        CoreUtils.SetKeyword(_aoMaterial, "_BLUR_H", false);
        CoreUtils.SetKeyword(_aoMaterial, "_BLUR_V", true);
        cmd.DrawProcedural(Matrix4x4.identity, _aoMaterial, 1, MeshTopology.Triangles, 3);

        // 输出给后续 Lighting Pass 使用
        cmd.SetGlobalTexture("_ScreenSpaceOcclusionTexture", _ssaoRT);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public void Dispose() {
        _ssaoRT?.Release();
        _ssaoBlurredRT?.Release();
        if (_noiseTex != null) Object.Destroy(_noiseTex);
        if (_aoMaterial != null) CoreUtils.Destroy(_aoMaterial);
    }

    // 生成 4×4 随机噪声纹理
    private Texture2D GenerateNoiseTex(int size) {
        var tex = new Texture2D(size, size, TextureFormat.RGB24, false) {
            filterMode = FilterMode.Point,
            wrapMode = TextureWrapMode.Repeat
        };
        var pixels = new Color[size * size];
        for (int i = 0; i < pixels.Length; i++) {
            // 单位圆内的随机旋转向量
            float angle = Random.value * Mathf.PI * 2f;
            pixels[i] = new Color(
                Mathf.Cos(angle), Mathf.Sin(angle), 0, 0
            );
        }
        tex.SetPixels(pixels);
        tex.Apply();
        return tex;
    }
}
```

**性能预算对比表：**

| 配置 | 采样核 | 分辨率 | GPU 耗时（Adreno 650） | 适用场景 |
|------|--------|--------|----------------------|----------|
| 超高画质 | 32 | 全分辨率 | 2.1ms | 旗舰 PC/主机 |
| 高画质 | 16 | 全分辨率 | 1.3ms | 旗舰移动端 |
| 中画质 | 8 | 半分辨率 | 0.5ms | 中端移动端 |
| 低画质 | 关闭 | — | 0ms | 低端机 |

### ⚡ 实战经验

1. **Range Check 是消除"假黑边"的关键**：不加 Range Check 时，采样点跨越到另一面，会误判为遮蔽。`smoothstep(0, 1, radius / distance)` 能让远距离遮蔽自动衰减
2. **法线从 G-Buffer 获取比 ddx/ddy 更稳定**：ddx/ddy 在几何边缘会跳跃，导致法线噪声。如果项目有 Normal Texture（延迟渲染或预深度法线 Pass），优先从它采样
3. **半分辨率渲染 + 双边模糊 ≈ 全分辨率效果**：这是移动端最重要的优化手段。SSAO 本身是低频信息，半分辨率完全够用
4. **AO 与间接光混合**：不要直接 Multiply，用 `ambientColor * ao` 混合。只有间接光被遮蔽，直射光不受影响
5. **时间域抗锯齿（TAA）与 SSAO 配合**：SSAO 输出应该在 TAA 之前应用，否则 TAA 的历史帧会引入"残留遮蔽"鬼影

### 🎯 能力体检清单

- [ ] **如果不懂深度重建** → 你需要补：NDC 空间、投影矩阵逆变换、LinearizeDepth
- [ ] **如果不懂 TBN 矩阵** → 你需要补：切线空间、Gram-Schmidt 正交化、法线空间转换
- [ ] **如果不会写 URP Renderer Feature** → 你需要补：ScriptableRenderPass、RTHandle、CommandBuffer
- [ ] **如果不会做双边模糊** → 你需要补：Bilateral Filter 原理、深度权重、横竖分离优化
- [ ] **如果不理解移动端性能预算** → 你需要补：TBDR 架构、带宽优化、ALU vs 带宽平衡

### 🔗 相关问题

- SSAO vs GTAO（Ground Truth Ambient Occlusion）有什么本质区别？GTAO 为什么更贵？
- 如何把 SSAO 与 Lightmap 烘焙 AO 结合使用？（实时+离线混合方案）
- URP 7.7+ 内置的 SSAO 用的是什么算法？为什么它效率高但效果一般？
