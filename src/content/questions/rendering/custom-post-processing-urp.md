---
title: "URP 后处理管线自定义：如何串联 Bloom + Color Grading + TAA？"
category: "rendering"
level: 3
tags: ["URP", "后处理", "Bloom", "Color Grading", "TAA", "Renderer Feature", "Render Target"]
hint: "核心难点在于 RT 的生命周期管理和 Pass 顺序——TAA 必须在 Bloom 之前，Color Grading 在 Bloom 之后"
related: ["rendering/urp-renderer-feature", "optimization/gpu-bandwidth-optimization", "rendering/forward-plus-cluster"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们项目用 URP，美术觉得默认的后处理效果不够好——Bloom 过渡不自然、颜色偏灰、移动端有锯齿。你现在需要自己写一套后处理管线：TAA 抗锯齿 → 自定义 Bloom → Color Grading（LUT），并且要在移动端跑到 60fps。你会怎么设计？

这是网易、腾讯、字节等使用 URP 开发手游的高频面试题。考察的不是"会不会调参数"，而是**对 URP 后处理管线的深度理解 + RT 管理 + 性能控制**的综合能力。

### ✅ 核心要点

1. **Pass 顺序是灵魂**：TAA → (Tonemapping) → Bloom → Color Grading，顺序错了效果全毁
2. **Render Target 管理**：每个 Pass 的输入/输出 RT 要精确管理，移动端 MSAA Resolve 和格式选择直接影响带宽
3. **自定义 Bloom（Mip Chain）**：URP 默认 Bloom 用降采样 Mip Chain，自定义可以实现更好的光晕控制
4. **TAA 的 History Buffer**：需要跨帧的 History RT，处理动态物体 Ghosting 问题
5. **移动端性能取舍**：RT 格式（RGB111110 vs RGBA32）、降采样策略、Pass 合并

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：TAA + Bloom + Color Grading 的自定义后处理管线，移动端 60fps
     ↓
关键决策：
  Q1：Pass 顺序怎么排？
      → TAA 最先（需要原始几何颜色）→ Bloom（需要 HDR 线性空间）→ Color Grading（LDR 最终调色）
     ↓
  Q2：RT 格式怎么选？
      → TAA 前：HDR (R16G16B16A16 或 RGB111110)
      → Bloom 中间：HDR 降采样
      → Color Grading 后：LDR (RGBA32)
     ↓
  Q3：移动端怎么省带宽？
      → TAA 在半分辨率运行
      → Bloom 的 Mip Chain 用 1/4 → 1/8 → 1/16
      → 合并 Tone Mapping 到 Bloom 的最后一步
     ↓
  Q4：TAA 的 History RT 怎么管理？
      → 用 RTHandle 系统，跨帧保留
      → 需要处理摄像机抖动（Jitter）和速度矢量（Motion Vector）
```

#### 知识点拆解（倒推树）

```
自定义后处理管线
├── URP Renderer Feature 架构
│   ├── 需要理解：ScriptableRendererFeature / ScriptableRenderPass
│   ├── 需要理解：RenderPassEvent 注入点
│   │   ├── BeforeRenderingPostProcessing
│   │   └── AfterRenderingPostProcessing
│   └── 需要理解：Blit 操作与 RT 切换
│       └── Blitter.BlitTexture (URP 14+)
│
├── TAA（Temporal Anti-Aliasing）
│   ├── 需要理解：Jitter（Halton 序列摄像机偏移）
│   ├── 需要理解：Motion Vector Pass（速度矢量）
│   │   └── 需要前一帧的 ViewProjection 矩阵
│   ├── 需要理解：History Buffer（上一帧的颜色）
│   │   └── 跨帧 RT：RTHandle 系统
│   ├── 需要理解：Neighborhood Clipping（A-SVGF 或 Variance Clipping）
│   │   └── 解决 Ghosting（动态物体拖影）
│   └── 性能：可以在 1/2 分辨率运行
│
├── Bloom（Mip Chain Bloom）
│   ├── 需要理解：降采样 Mip Chain
│   │   ├── Step 1：从 Source RT 逐级降采样（每级 ÷2）
│   │   ├── Step 2：每级做高斯模糊
│   │   └── Step 3：逐级上采样叠加（Prefilter + Upsample）
│   ├── 需要理解：Threshold 控制光晕来源
│   │   └── Soft Threshold（ Knee）避免硬边
│   ├── 需要理解：HDR 空间的 Bloom
│   │   └── 必须在 Tone Mapping 之前（线性 HDR 空间）
│   └── 自定义增强：Lens Dirt（污渍纹理驱动光晕形状）
│
├── Color Grading（LUT）
│   ├── 需要理解：Tonemapping（ACES / Neutral / Reinhard）
│   │   └── 在 HDR → LDR 转换时应用
│   ├── 需要理解：LUT（Look-Up Table）调色
│   │   ├── 3D LUT（32×32×32）覆盖全色域
│   │   └── 引擎中用 2D Strip LUT（如 1024×32）节省采样
│   ├── 需要理解：Color Space（Linear vs sRGB）
│   │   └── LUT 调色在 Linear 空间做，最后转 sRGB
│   └── 自定义：多 LUT 混合（白天/夜晚/地下城切换）
│
└── RT 管理与性能
    ├── RTHandle 系统（URP 14+）
    │   ├── 跨帧 RT（TAA History Buffer）
    │   └── 自动 Resolution Scaling
    ├── 移动端格式选择
    │   ├── HDR：RGB111110Float（无 Alpha）或 R16G16B16A16
    │   └── LDR：RGBA8
    ├── 降采样策略
    │   ├── TAA：1/2 分辨率
    │   ├── Bloom Mip：1/4 → 1/16
    │   └── 最终合成：全分辨率
    └── Pass 合并
        ├── Tone Mapping + Color Grading → 一个 Pass
        └── Bloom Upsample + 最终 Blit → 一个 Pass
```

#### 代码实现

**自定义后处理 Renderer Feature：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class CustomPostProcessFeature : ScriptableRendererFeature {
    public enum PostProcessOrder { TAA_Bloom_Grading, Bloom_TAA_Grading }
    
    [Header("Settings")]
    public PostProcessOrder order = PostProcessOrder.TAA_Bloom_Grading;
    public bool halfResTAA = true;
    public bool enableBloom = true;
    public bool enableColorGrading = true;
    
    private TAAPass _taaPass;
    private BloomPass _bloomPass;
    private ColorGradingPass _gradingPass;
    
    public override void Create() {
        _taaPass = new TAAPass(halfResTAA);
        _bloomPass = new BloomPass();
        _gradingPass = new ColorGradingPass();
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer,
        ref RenderingData renderingData) {
        // 注入到后处理开始前
        if (order == PostProcessOrder.TAA_Bloom_Grading) {
            renderer.EnqueuePass(_taaPass);
            if (enableBloom) renderer.EnqueuePass(_bloomPass);
            if (enableColorGrading) renderer.EnqueuePass(_gradingPass);
        }
    }
    
    // URP 16+ 资源释放
    public override void SetupRenderPasses(RenderGraph renderGraph,
        ref RenderingData renderingData) {
        // 在 RenderGraph 模式下注册 Pass
    }
}
```

**TAA Pass（核心逻辑）：**

```csharp
using UnityEngine.Rendering.Universal;

public class TAAPass : ScriptableRenderPass {
    private readonly bool _halfRes;
    private Material _taaMaterial;
    private RTHandle _historyRT;
    private bool _historyReady = false;
    
    // Halton 序列抖动
    private static readonly float[] HaltonX = {
        0.5f, 0.25f, 0.75f, 0.125f, 0.625f, 0.375f, 0.875f, 0.0625f
    };
    private static readonly float[] HaltonY = {
        0.5f, 0.75f, 0.25f, 0.625f, 0.125f, 0.875f, 0.375f, 0.9375f
    };
    private int _frameIndex = 0;
    
    // 前一帧矩阵（用于 Motion Vector）
    private Matrix4x4 _prevViewProj;
    
    public TAAPass(bool halfRes) {
        _halfRes = halfRes;
        renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        
        var shader = Shader.Find("Hidden/Custom/TAA");
        _taaMaterial = CoreUtils.CreateEngineMaterial(shader);
        
        // 分配 History RT
        var desc = new RenderTextureDescriptor(
            halfRes ? Screen.width / 2 : Screen.width,
            halfRes ? Screen.height / 2 : Screen.height,
            RenderTextureFormat.RGB111110Float, 0
        );
        RenderingUtils.ReAllocateIfNeeded(ref _historyRT, desc, "HistoryRT_TAA");
    }
    
    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData) {
        // 设置 Jitter（投影矩阵偏移）
        int idx = _frameIndex % 8;
        float jitterX = (HaltonX[idx] - 0.5f) / (_halfRes ? Screen.width / 2f : Screen.width);
        float jitterY = (HaltonY[idx] - 0.5f) / (_halfRes ? Screen.height / 2f : Screen.height);
        
        var camera = renderingData.cameraData.camera;
        var proj = camera.nonJitteredProjectionMatrix;
        proj.m02 += jitterX * 2;
        proj.m12 += jitterY * 2;
        camera.projectionMatrix = proj;
        
        // 配置 Pass 的输入输出
        ConfigureInput(ScriptableRenderPassInput.Depth | ScriptableRenderPassInput.MotionVectors);
    }
    
    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
        var cmd = CommandBufferPool.Get("CustomTAA");
        
        int w = _halfRes ? Screen.width / 2 : Screen.width;
        int h = _halfRes ? Screen.height / 2 : Screen.height;
        
        // 获取当前帧颜色
        var cameraColor = renderingData.cameraData.renderer.cameraColorTargetHandle;
        
        // TAA Shader 参数
        _taaMaterial.SetTexture("_HistoryTex", _historyRT);
        _taaMaterial.SetFloat("_BlendFactor", 0.1f); // 历史帧混合权重
        
        // 速度矢量（URP 自动生成，如果开启 Motion Vector）
        // _taaMaterial.SetTexture("_MotionVectorTex", ...);
        
        // 执行 TAA：Blit 当前帧 → 临时 RT
        var tempRT = RTHandles.Alloc(w, h, 1, DepthBits.None,
            RenderTextureFormat.RGB111110Float, "TAA_Temp");
        
        Blitter.BlitCameraTexture(cmd, cameraColor, tempRT, _taaMaterial, 0);
        
        // 更新 History RT
        Blitter.BlitCameraTexture(cmd, tempRT, _historyRT);
        
        // 输出回 CameraColor
        Blitter.BlitCameraTexture(cmd, tempRT, cameraColor);
        
        RTHandles.Release(tempRT);
        _frameIndex++;
        
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```

**自定义 Bloom Shader（Mip Chain Bloom）：**

```hlsl
// CustomBloom.shader — 降采样 + 上采样 Bloom
Shader "Hidden/Custom/Bloom" {
    Properties { _MainTex ("Source", 2D) = "white" {} }
    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        
        // === Pass 0: 降采样（Prefilter + Downsample）===
        Pass {
            Name "BloomDown"
            ZTest Always ZWrite Off Cull Off
            
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment BloomDownFrag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            TEXTURE2D(_MainTex); SAMPLER(sampler_MainTex);
            float4 _MainTex_TexelSize;
            float _Threshold;
            float _Knee;
            
            float Luminance(float3 c) { return dot(c, float3(0.2126, 0.7152, 0.0722)); }
            
            // Soft Threshold（Knee 曲线）
            float3 Prefilter(float3 color) {
                float brightness = Luminance(color);
                float soft = brightness - _Threshold + _Knee;
                soft = clamp(soft, 0, 2 * _Knee);
                soft = soft * soft / (4 * _Knee + 1e-4);
                float contribution = max(soft, brightness - _Threshold);
                contribution /= max(brightness, 1e-4);
                return color * contribution;
            }
            
            half4 BloomDownFrag(Varyings input) : SV_Target {
                float2 uv = input.uv;
                float2 texelSize = _MainTex_TexelSize.xy;
                
                // 3×3 高斯降采样
                float3 color = 0;
                float weights[9] = {
                    0.0625, 0.125, 0.0625,
                    0.125,  0.25,  0.125,
                    0.0625, 0.125, 0.0625
                };
                int idx = 0;
                for (int x = -1; x <= 1; x++) {
                    for (int y = -1; y <= 1; y++) {
                        float2 offset = float2(x, y) * texelSize;
                        color += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex,
                            uv + offset).rgb * weights[idx++];
                    }
                }
                
                // 第一步需要 Prefilter
                return half4(Prefilter(color), 1.0);
            }
            ENDHLSL
        }
        
        // === Pass 1: 上采样叠加（Upsample + Composite）===
        Pass {
            Name "BloomUp"
            ZTest Always ZWrite Off Cull Off
            
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment BloomUpFrag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            TEXTURE2D(_MainTex);    SAMPLER(sampler_MainTex);    // 上一级 Mip
            TEXTURE2D(_BloomHigh);  SAMPLER(sampler_BloomHigh);  // 当前累积 Bloom
            float4 _MainTex_TexelSize;
            float _BloomIntensity;
            float _Scatter; // 散射度（0.5~1.0）
            
            half4 BloomUpFrag(Varyings input) : SV_Target {
                float2 uv = input.uv;
                
                // 双线性上采样 + 加权混合
                float3 lowRes = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).rgb;
                float3 highRes = SAMPLE_TEXTURE2D(_BloomHigh, sampler_BloomHigh, uv).rgb;
                
                // Scatter 控制光晕扩散范围
                float3 result = lerp(highRes, lowRes, _Scatter);
                return half4(result * _BloomIntensity, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**Color Grading Pass（LUT 调色）：**

```hlsl
// ColorGrading.shader — Tonemapping + LUT
Shader "Hidden/Custom/ColorGrading" {
    Properties {
        _MainTex ("Source", 2D) = "white" {}
        _LUTTex ("LUT Texture", 2D) = "white" {}
        _LUTIntensity ("LUT Intensity", Range(0,1)) = 1.0
        _Exposure ("Exposure", Float) = 0.0
    }
    SubShader {
        Pass {
            Name "ColorGrading"
            ZTest Always ZWrite Off Cull Off
            
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment GradingFrag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            TEXTURE2D(_MainTex); SAMPLER(sampler_MainTex);
            TEXTURE2D(_LUTTex);   SAMPLER(sampler_LUTTex);
            float _LUTIntensity;
            float _Exposure;
            
            // ACES Tonemapping（电影级色调映射）
            float3 ACESFilm(float3 x) {
                float a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14;
                return saturate((x * (a * x + b)) / (x * (c * x + d) + e));
            }
            
            // 从 2D Strip LUT 中采样 3D LUT 效果
            // Strip 格式：width = 32 * 32 = 1024, height = 32
            float3 SampleLUT(float3 color) {
                color = saturate(color);
                float maxColor = 31.0;
                
                // 将 RGB 映射到 Strip 坐标
                int slice = (int)(color.b * maxColor);
                float2 uv;
                uv.x = (color.r * maxColor + slice + 0.5) / (32.0 * 32.0);
                uv.y = (color.g * maxColor + 0.5) / 32.0;
                
                return SAMPLE_TEXTURE2D(_LUTTex, sampler_LUTTex, uv).rgb;
            }
            
            half4 GradingFrag(Varyings input) : SV_Target {
                float3 hdrColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, input.uv).rgb;
                
                // Step 1: Exposure
                hdrColor *= exp2(_Exposure);
                
                // Step 2: Tonemapping（HDR → LDR）
                float3 ldrColor = ACESFilm(hdrColor);
                
                // Step 3: LUT 调色
                float3 graded = SampleLUT(ldrColor);
                float3 finalColor = lerp(ldrColor, graded, _LUTIntensity);
                
                // Step 4: sRGB 输出
                finalColor = pow(finalColor, 1.0 / 2.2);
                
                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**后处理管线流程图：**

```
Camera Render
     ↓
[Geometry Pass] → HDR Color RT (R16G16B16A16)
     ↓
[TAA Pass]（Jitter + History Blend + Motion Vector）
     ↓ Tone Mapping 可以合并到 Bloom 最后一步
[Bloom Pass]
     ├── Downsample Pass 1: Full → 1/2
     ├── Downsample Pass 2: 1/2 → 1/4
     ├── Downsample Pass 3: 1/4 → 1/8
     ├── Downsample Pass 4: 1/8 → 1/16
     ├── Upsample Pass: 1/16 → 1/8 → 1/4 → 1/2 → Full
     └── Composite: Bloom 叠加到原图
     ↓
[Color Grading Pass]（ACES Tonemapping + LUT）
     ↓
Final Output → 显示器
```

### ⚯ 实战经验

1. **TAA 的 Ghosting 是头号问题**：动态物体（尤其是 UI、粒子）会产生拖影。解决方案是在 TAA Shader 中做 Velocity-based clipping——当像素速度过大时降低历史帧权重。对于 UI，要在 TAA 之后再渲染（URP 的 Overlay Camera 机制）
2. **Bloom 的 Threshold 要用 Knee 曲线**：硬阈值会导致 Bloom 边界生硬。用 ACES 推荐的 Soft Knee 曲线，在阈值附近平滑过渡，光晕更自然
3. **移动端 RT 格式的选择**：HDR 中间 RT 用 `RGB111110Float`（11F→11F→10F，5字节）而非 `R16G16B16A16`（8字节），带宽省 37.5%，精度对于后处理足够
4. **RenderGraph 是未来方向**：URP 16+ 推荐用 RenderGraph API 替代直接的 RTHandle 管理。RenderGraph 自动做 Pass 合并和 RT 复用，能进一步降低显存

### 🎯 能力体检清单

- [ ] **如果不知道 Pass 顺序为什么重要** → 你需要补：HDR/LDR 空间、Tonemapping 时机、Bloom 必须在 HDR 线性空间
- [ ] **如果不会写 Renderer Feature** → 你需要补：URP 的 ScriptableRendererFeature 机制、RenderPassEvent 注入点、Blit 操作
- [ ] **如果不懂 TAA 原理** → 你需要补：Jitter 抖动、Motion Vector、History Buffer、Neighborhood Clipping
- [ ] **如果不知道 Bloom Mip Chain** → 你需要补：降采样金字塔、高斯模糊分离轴、Prefilter 和 Upsample 的区别
- [ ] **如果不会做性能优化** → 你需要补：移动端 TBDR 架构、RT 带宽计算、降采样策略、Pass 合并

### 🔗 相关问题

- URP 的 RenderGraph 系统和传统的 RTHandle 管理有什么区别？迁移策略是什么？
- 移动端 TAA 的替代方案有哪些？（FXAA / SMAA 对比）
- 如何实现 LUT 的实时编辑和预览？（DCC 工具 → 引擎管线）
