---
title: "骁龙888发热掉帧：Adreno GPU Tile-Based渲染带宽优化怎么做？"
category: "optimization"
level: 4
tags: ["Adreno", "Tile-Based", "GPU带宽", "移动端优化", "骁龙", "带宽优化", "GLES"]
hint: "不是降分辨率那么简单——核心是理解 Tile-Based Rendering 的机制，减少 Tile 切换、避免 FBO 频繁切换、合并 Surface"
related: ["optimization/gpu-bandwidth-optimization", "optimization/mobile-overheating-gpu-analysis", "optimization/drawcall-500-to-100", "rendering/urp-volumetric-fog"]
---

## 参考答案

### 🎬 场景描述

面试官（技术总监 / 引擎负责人）说：

> "我们的游戏在骁龙888上跑 15 分钟就开始降频掉帧，GPU 频率从 840MHz 掉到 400MHz，帧率从 60 掉到 30。用 Snapdragon Profiler 一看，带宽占了 18GB/s，其中 70% 是 Tile 切换和 FBO 切换导致的。你作为 TA，给我一个带宽优化方案，目标把带宽压到 10GB/s 以内。
>
> 当前已知问题：
> 1. 角色渲染用了 4 张全屏 RT（法线/位置/SSS/SSAO）
> 2. 全屏后处理链有 7 个 Pass（Bloom、Color Grading、DOF、Motion Blur、TAA、Vignette、Chromatic Aberration）
> 3. UI 用了独立的 Render Texture，每帧从 CPU 拷贝"

这是高通骁龙平台优化面试题，考察的是 **Tile-Based GPU 架构理解 + 移动端带宽优化** 的深度。

### ✅ 核心要点

1. **Adreno GPU 是 Tile-Based Deferred Rendering（TBDR）**：不是 Desktop GPU，优化策略完全不同
2. **Tile 切换是最大带宽杀手**：每次 Render Target 切换，Tile 数据要在 GMEM ↔ System Memory 之间搬运
3. **GMEM 大小决定 Tile 分辨率**：Adreno 650 的 GMEM 为 1MB，1080p RGBA8 一个 Tile 只能放 256×256
4. **核心优化原则**：减少 Render Target 数量 → 减少 Pass 切换 → 减少 Tile 搬运
5. **具体手段**：RT 合并、MRT（多目标渲染）、减少 FBO 切换、降分辨率渲染、剔除无用后处理

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：带宽从 18GB/s 降到 10GB/s 以内
     ↓ 倒推
18GB/s 中 70% 来自 Tile 切换 → 需要砍掉 Tile 切换次数
     ↓ 倒推
Tile 切换 = Render Target 切换 → 当前有 4张 G-Buffer + 7个后处理 = 11次全屏切换
     ↓ 倒推
三层优化策略：
  Layer 1：砍 Pass（后处理从 7 个砍到 4 个）
  Layer 2：合 Pass（G-Buffer 4张合为 MRT 1 Pass）
  Layer 3：降规格（半分辨率 RT、合并 LDR Pass）
     ↓ 倒推
最终效果：Tile 切换从 11次→4次，带宽预计 8-9 GB/s
```

#### 知识点拆解（倒推树）

```
Adreno TBDR 带宽优化
├── Adreno GPU 架构
│   ├── TBDR 执行流程（Vertex → Tiling → Rasterize per Tile → Resolve）
│   ├── GMEM（内部显存）：1MB（650）/ 2MB（730），决定 Tile Size
│   ├── Tile Resolve / Load 操作（带宽来源）
│   └── Flexible Fast Clear（GMEM Clear 的正确使用）
├── Render Target 管理
│   ├── MRT（Multiple Render Targets）：一次 Draw 写多张 RT
│   ├── RT 分辨率策略：半分辨率 → 双线性上采样
│   ├── RT 格式选择：RGBA16F → RGBA8 / RGB9E5（HDR场景）
│   └── Vulkan Input Attachment（Tile 内直接读取，零带宽）
├── 后处理链优化
│   ├── Pass 合并：Bloom + Vignette + Chromatic Aberrance → 一个 Shader
│   ├── LDR vs HDR Pass 分离：HDR 阶段用 RGBA16F，LDR 阶段用 RGBA8
│   └── 计算 Shader vs 像素 Shader：移动端 CS 不一定更快
├── Snapdragon Profiler 使用
│   ├── Render Stage 视图：看每个 Stage 耗时
│   ├── Bandwidth 视图：GMEM ↔ DDR 的搬运量
│   └── 性能瓶颈定位：VS Bound / FS Bound / Texture Bound / Bandwidth Bound
└── 平台特性
    ├── Adreno GPU Turbo Boost：持续高频后会降频
    ├── binning 模式（Direct Render vs Bin Render）
    └── gpu_perfcounter 读取（L2 Cache Miss Rate）
```

#### 代码实现

**Step 1：后处理 Pass 合并（最大收益）**

```hlsl
// 合并 Bloom + Vignette + Chromatic Aberrance + Color Grading 为单 Pass
// 文件：PostProcessCombined.shader

Shader "Hidden/PostProcess/CombinedFinal"
{
    SubShader
    {
        Pass
        {
            Name "CombinedFinal"
            
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragCombined
            #pragma multi_compile_local _ _BLOOM_ON
            #pragma multi_compile_local _ _VIGNETTE_ON
            #pragma multi_compile_local _ _CHROMATIC_ON
            #pragma multi_compile_local _ _COLOR_GRADING_ON
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"
            
            TEXTURE2D_X(_MainTex);          SAMPLER(sampler_MainTex);
            TEXTURE2D_X(_BloomTex);         SAMPLER(sampler_BloomTex);
            TEXTURE2D(_LUTTex);             SAMPLER(sampler_LUTTex);
            
            float4 _MainTex_TexelSize;
            float _BloomIntensity;
            float _VignetteIntensity;
            float _ChromaticAberration;
            float _ColorGradingContribution;
            
            half4 FragCombined(Varyings input) : SV_Target
            {
                float2 uv = input.uv;
                half4 color = SAMPLE_TEXTURE2D_X(_MainTex, sampler_MainTex, uv);
                
                // 1. Bloom（采半分辨率 Bloom RT）
                #if _BLOOM_ON
                half3 bloom = SAMPLE_TEXTURE2D_X(_BloomTex, sampler_BloomTex, uv).rgb;
                color.rgb += bloom * _BloomIntensity;
                #endif
                
                // 2. Chromatic Aberration（径向 UV 偏移，放在 LDR 阶段更省）
                #if _CHROMATIC_ON
                float2 caDir = uv - 0.5;
                float caDist = length(caDir);
                float2 caOffset = caDir * _ChromaticAberration * caDist;
                color.r = SAMPLE_TEXTURE2D_X(_MainTex, sampler_MainTex, uv + caOffset).r;
                color.b = SAMPLE_TEXTURE2D_X(_MainTex, sampler_MainTex, uv - caOffset).b;
                #endif
                
                // 3. Color Grading（LUT 采样）
                #if _COLOR_GRADING_ON
                half3 graded = ApplyLUT(color.rgb, _LUTTex, sampler_LUTTex);
                color.rgb = lerp(color.rgb, graded, _ColorGradingContribution);
                #endif
                
                // 4. Vignette（最后一步，暗角）
                #if _VIGNETTE_ON
                float vignette = smoothstep(1.0, 0.3, caDist * 2.0);
                color.rgb *= lerp(1.0, vignette, _VignetteIntensity);
                #endif
                
                return color;
            }
            ENDHLSL
        }
    }
}
```

```csharp
// URP Renderer Feature: 注册合并后的单 Pass 后处理
public class CombinedPostProcessFeature : ScriptableRendererFeature
{
    class CombinedPostPass : ScriptableRenderPass
    {
        private Material _combinedMat;
        private RTHandle _source;
        private RTHandle _tempTarget;
        
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.depthBufferBits = 0;
            RenderingUtils.ReAllocateIfNeeded(ref _tempTarget, desc);
            _source = renderingData.cameraData.renderer.cameraColorTargetHandle;
        }
        
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("CombinedPostProcess");
            
            // Blit to temp
            Blitter.BlitCameraTexture(cmd, _source, _tempTarget);
            
            // Blit back with combined material（所有后处理一步完成）
            Blitter.BlitCameraTexture(cmd, _tempTarget, _source, _combinedMat, 0);
            
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

**Step 2：G-Buffer MRT 合并**

```hlsl
// 移动端 G-Buffer：将 4 张 RT 合并为 1 Pass MRT（写入 2 张 RT）
// 方案：紧凑编码，将 Normal + Roughness + Metallic 编码到 RGBA8

struct GBufferOutput
{
    half4 RT0 : SV_Target0;  // Albedo.rgb + AO.a
    half4 RT1 : SV_Target1;  // Normal.rg (Octahedral) + Roughness.b + Metallic.a
    // 原来 4 张 RT → 2 张，Tile 切换减半
};

GBufferOutput FragGBuffer(Varyings input)
{
    GBufferOutput output;
    
    // Albedo + AO
    half3 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv).rgb;
    half ao = SAMPLE_TEXTURE2D(_AOMap, sampler_AOMap, input.uv).r;
    output.RT0 = half4(albedo, ao);
    
    // Normal（八面体编码：vec3 → vec2，省一个通道）
    half3 normalWS = normalize(input.normalWS);
    half2 octNormal = PackNormalOctahedron(normalWS);
    
    // Roughness + Metallic
    half roughness = SAMPLE_TEXTURE2D(_RoughnessMap, sampler_RoughnessMap, input.uv).r;
    half metallic = SAMPLE_TEXTURE2D(_MetallicMap, sampler_MetallicMap, input.uv).r;
    output.RT1 = half4(octNormal, roughness, metallic);
    
    return output;
}
```

**Step 3：性能预算表**

```
优化前 Tile 切换分析（1080p）：
┌──────────────────────┬────────────┬──────────────┐
│ Pass                 │ RT 格式    │ 带宽估算     │
├──────────────────────┼────────────┼──────────────┤
│ G-Buffer (4张分开)    │ RGBA16F×4  │ 4.2 GB/s    │
│ SSAO                  │ R8         │ 0.3 GB/s    │
│ SSS Preintegrate      │ RGBA16F    │ 2.1 GB/s    │
│ Bloom (降采样×3)       │ RGBA16F×3  │ 3.6 GB/s    │
│ DOF (近/远)           │ RGBA16F×2  │ 2.8 GB/s    │
│ Motion Blur           │ RGBA16F    │ 1.4 GB/s    │
│ TAA                   │ RGBA16F    │ 1.4 GB/s    │
│ Color Grading         │ RGBA8      │ 0.7 GB/s    │
│ Vignette + CA         │ RGBA8      │ 0.7 GB/s    │
│ UI Copy               │ RGBA8      │ 0.7 GB/s    │
├──────────────────────┼────────────┼──────────────┤
│ 总计                  │            │ ~18.0 GB/s  │
└──────────────────────┴────────────┴──────────────┘

优化后：
┌──────────────────────┬────────────┬──────────────┐
│ Pass                 │ RT 格式    │ 带宽估算     │
├──────────────────────┼────────────┼──────────────┤
│ G-Buffer (MRT 2张)   │ RGBA8×2    │ 1.8 GB/s    │
│ SSAO (半分辨率)       │ R8         │ 0.15 GB/s   │
│ Bloom (半分辨率)      │ RGBA8×1    │ 0.7 GB/s    │
│ DOF+MotionBlur+TAA    │ RGBA8×1    │ 0.7 GB/s    │
│ Combined Final       │ RGBA8×1    │ 0.7 GB/s    │
│ UI (直接渲染，无拷贝)  │ -          │ 0 GB/s      │
├──────────────────────┼────────────┼──────────────┤
│ 总计                  │            │ ~4.1 GB/s   │
└──────────────────────┴────────────┴──────────────┘

带宽降幅：18 → 4.1 GB/s（-77%）
```

### ⚡ 实战经验

1. **最大收益来自"砍 Pass"而非"降参数"**：合并 3 个后处理 Pass 为 1 个，比调参数收益大 10 倍
2. **MRT 在 Adreno 上有原生加速**：GMEM 内的 Tile 数据可以同时写多个 RT，零额外带宽
3. **RGBA16F → RGBA8 是巨大的带宽节省**：HDR 场景用 RGB9E5（5bit 指数 + 9bit 尾数）几乎看不出差别
4. **UI 不要用 Render Texture 拷贝**：直接在 Swapchain 上画 UI，省一次全屏 Blit
5. **Flexible Fast Clear**：Adreno 支持 Tile 级别的 Fast Clear，确保每帧第一个操作是 Clear（而非 Load）
6. **Snapdragon Profiler 的 "Bandwidth" 视图比 "Frame Duration" 更重要**：带宽才是移动端发热的根本原因
7. **降频是热保护机制**：带宽降下来后，GPU 不需要全频运行，发热自然减少

### 🎯 能力体检清单

- [ ] 能否画出 Adreno TBDR 的完整渲染流水线（Vertex → Bin → Render per Tile → Resolve）？
- [ ] GMEM 大小如何计算 Tile 尺寸？如果 GMEM 是 1MB，RGBA8 格式，1080p 下 Tile 是多大？
- [ ] 为什么在移动端 Vulkan 上 Input Attachment 比 sampling 纹理更省带宽？
- [ ] MRT 在 Adreno 上是否总是更优？有什么限制条件？
- [ ] 如果不做后处理合并，还有哪些手段可以降低带宽？

### 🔗 相关问题

- [GPU 带宽优化](optimization/gpu-bandwidth-optimization.md)
- [移动端 GPU 发热分析](optimization/mobile-overheating-gpu-analysis.md)
- [Draw Call 从 500 降到 100](optimization/drawcall-500-to-100.md)
- [URP 体积雾](rendering/urp-volumetric-fog.md)
