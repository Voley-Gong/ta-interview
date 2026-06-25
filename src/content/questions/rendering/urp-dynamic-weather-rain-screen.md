---
title: "URP动态天气系统：屏幕空间雨幕渲染怎么做？面试官要我现场画架构"
category: "rendering"
level: 3
tags: ["URP", "RendererFeature", "天气系统", "后处理", "雨幕", "ScreenSpace", "RenderObjects"]
hint: "雨幕=全屏后处理Pass(雨滴法线扰动+运动模糊)+世界空间雨滴粒子+地面涟漪Shader三层混合，全部在Renderer Feature中编排"
related: ["rendering/urp-renderer-feature", "rendering/custom-post-processing-urp", "rendering/depth-based-screen-distortion", "shader/screen-space-rain-droplet"]
---

## 参考答案

### 🎬 场景描述

面试官在白板上画了一个空的游戏场景，然后说：

> "我们要做一个动态天气系统，支持晴天→多云→小雨→暴雨→雪的实时切换。现在是 URP 项目，移动端。先不做体积云和闪电，第一期的核心需求是：**屏幕空间的雨幕效果**——镜头前有雨滴飞过、地面有水花涟漪、画面整体有湿润感的色调变化。你作为 TA，给我渲染架构设计，包括需要几个 Pass、每个 Pass 做什么、用什么 Blit。"

这是网易、腾讯做开放世界 / MMO 项目时 **TA 岗的经典系统设计题**。考察的是 URP Renderer Feature 编排能力、后处理架构设计、以及移动端性能预算的把控。

### ✅ 核心要点

1. **三层混合架构**：屏幕空间雨滴层 + 地面交互层（涟漪/水花）+ 画面湿润色调层
2. **Renderer Feature 编排**：用 1-2 个 `ScriptableRendererFeature` 编排所有 Pass，不侵入现有渲染管线
3. **雨滴用后处理实现**：全屏 Quad 上的噪声动画 + 法线扰动 UV，比真实粒子省 100 倍性能
4. **地面涟漪用 Shader 修改**：在地面材质的 Shader 中叠加雨滴交互（高度场扰动），而非后处理
5. **性能预算**：移动端全屏 Pass 不能超过 2ms（GLES3 基准），雨滴层必须用半分辨率 RT

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：雨天场景，镜头前雨滴飞过 + 地面涟漪扩散 + 画面偏冷湿润
                ↑
倒推1：三个视觉层需要独立处理
       ├── 屏幕雨滴 → 后处理全屏 Pass（不需要 3D 几何体）
       ├── 地面涟漪 → 地面材质 Shader 内部处理（需要世界坐标）
       └── 湿润色调 → 后处理 Color Grading / LUT 切换
倒推2：屏幕雨滴用后处理实现
       ├── 需要一张全屏 RT 存储雨滴法线/运动方向
       ├── 用噪声纹理 + 时间动画生成雨滴轨迹
       └── 对 Scene Color 做 UV 偏移采样（模拟雨滴折射）
倒推3：地面涟漪需要世界空间判断
       ├── 只在「地面」材质上触发（Layer 判断）
       ├── 涟漪是周期性扩散的环形纹理
       └── 需要一个 Renderer Feature 在 AfterRenderingOpaques 注入
倒推4：湿润色调
       ├── 晴天 LUT → 雨天 LUT 的权重插值
       └── 用 URP 的 Volume + Color Adjustments 控制
倒推5：移动端性能
       ├── 雨滴 RT 用 1/2 分辨率
       ├── 地面涟漪用一张可平铺的 Ring 贴图（不要运行时生成）
       └── 总 GPU 预算控制在 2.5ms 以内
```

#### 知识点拆解（倒推树）

```
URP 动态天气雨幕
├── URP Renderer Feature 架构
│   ├── ScriptableRendererFeature 生命周期（Create / AddToPassList）
│   ├── ScriptableRenderPass 注入点（AfterRenderingOpaques / BeforeRenderingPostProcessing）
│   ├── Blit 链设计（Source RT → 处理 → Target RT）
│   ├── Render Target 分配（RTHandles / TemporaryRT）
│   └── 多 Pass 编排顺序与依赖关系
├── 屏幕空间雨滴
│   ├── 雨滴生成：噪声纹理 + UV Scroll（模拟下落运动）
│   ├── 雨滴法线扰动：对 SceneColor 做 ddx/ddy 法线扰动
│   ├── 运动模糊：速度场（Velocity Field）或方向性 Blur
│   ├── 雨滴拖尾：径向模糊 / 方向性拖尾
│   └── 深度 aware：用 Scene Depth 排除背景（远处雨滴更小更慢）
├── 地面涟漪交互
│   ├── Layer Mask 判断（只影响 Ground Layer）
│   ├── 涟漪生成：时间驱动的周期性环形纹理
│   ├── 涟漪法线：扰动地面法线 → 影响高光形状
│   ├── 水花粒子（可选）：粒子系统在地面碰撞点触发
│   └── 湿滑反射：地面 Roughness 降低（SSR 或 CubeMap 反射增强）
├── 湿润色调
│   ├── Volume 权重插值（_WeatherAmount 0→1）
│   ├── Color Filter / White Balance / Tone Mapping 联动
│   └── 雨天 LUT 预烘焙 vs 运行时计算
├── 天气状态机
│   ├── 天气参数集（WeatherProfile: rainIntensity, windDir, fogDensity...）
│   ├── 状态切换插值（晴天→雨天 5 秒过渡）
│   └── 全局变量传递（Global Shader Properties / CBUFFER）
└── 性能预算
    ├── 雨滴 Pass：1.0ms（半分辨率 RT）
    ├── 涟漪（在地面 Shader 内）：0.3ms
    ├── 色调 LUT 切换：0.1ms（硬件 LUT 单元）
    ├── 体积雨粒子（可选）：1.5ms
    └── CPU 开销：天气参数更新 + MaterialPropertyBlock（<0.1ms）
```

#### 代码实现

**Renderer Feature 架构：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class DynamicWeatherFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class WeatherSettings
    {
        public RenderPassEvent passEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        public Material rainScreenMaterial;
        public Material wetToneMaterial;
        public int downsample = 2; // 半分辨率
    }

    public WeatherSettings settings = new WeatherSettings();
    private RainScreenPass _rainPass;

    public override void Create()
    {
        _rainPass = new RainScreenPass(settings)
        {
            renderPassEvent = settings.passEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.rainScreenMaterial == null) return;
        // 只在天气激活时注入
        if (WeatherSystem.Instance == null || !WeatherSystem.Instance.IsWeatherActive) return;

        _rainPass.Setup(renderer.cameraColorTargetHandle);
        renderer.EnqueuePass(_rainPass);
    }

    protected override void Dispose(bool disposing)
    {
        _rainPass?.Dispose();
    }
}

public class RainScreenPass : ScriptableRenderPass
{
    private DynamicWeatherFeature.WeatherSettings _settings;
    private RTHandle _rainRT;
    private Material _blitMaterial;
    private static readonly int RainTexID = Shader.PropertyToID("_RainTempRT");

    public RainScreenPass(DynamicWeatherFeature.WeatherSettings settings)
    {
        _settings = settings;
        _blitMaterial = settings.rainScreenMaterial;
    }

    public void Setup(RTHandle cameraColor)
    {
        // 在 Execute 中使用
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        // 分配半分辨率 RT
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        desc.width /= _settings.downsample;
        desc.height /= _settings.downsample;
        desc.depthBufferBits = 0;
        RenderingUtils.ReAllocateIfNeeded(ref _rainRT, desc, name: "_RainTempRT");
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("RainScreenEffect");

        // Pass 1: 生成雨滴法线扰动图（半分辨率）
        cmd.SetRenderTarget(_rainRT);
        cmd.DrawMesh(RenderingUtils.fullscreenMesh, Matrix4x4.identity, _blitMaterial, 0, 0);

        // Pass 2: 将雨滴图叠加回场景颜色（双线性上采样）
        cmd.SetRenderTarget(renderingData.cameraData.renderer.cameraColorTargetHandle);
        cmd.DrawMesh(RenderingUtils.fullscreenMesh, Matrix4x4.identity, _blitMaterial, 0, 1);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public void Dispose()
    {
        _rainRT?.Release();
    }
}
```

**雨滴屏幕空间 Shader（HLSL）：**

```hlsl
Shader "Hidden/RainScreen"
{
    Properties { }
    SubShader
    {
        // Pass 0: 生成雨滴扰动图
        Pass
        {
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment FragRain

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            TEXTURE2D(_NoiseTex); SAMPLER(sampler_NoiseTex);
            TEXTURE2D(_CameraDepthTexture); SAMPLER(sampler_CameraDepthTexture);

            float _RainIntensity;
            float _RainSpeed;
            float2 _WindDir;
            float _TimeParam;

            half4 FragRain(Varyings IN) : SV_Target
            {
                float2 uv = IN.uv;

                // 两层噪声叠加：层1=大雨滴，层2=细雨
                float2 uv1 = uv * float2(1.0, 3.0) + float2(_WindDir.x * 0.1, -_TimeParam * _RainSpeed);
                float2 uv2 = uv * float2(2.0, 6.0) + float2(_WindDir.x * 0.2, -_TimeParam * _RainSpeed * 1.5);

                half rain1 = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, uv1).r;
                half rain2 = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, uv2).r;
                half rain = (rain1 * 0.7 + rain2 * 0.3) * _RainIntensity;

                // 雨滴拖尾方向（沿风向+重力）
                float2 streakDir = normalize(float2(_WindDir.x, -1.0));
                float streak = pow(rain, 3.0);

                // 输出 RG = 法线扰动方向，B = 强度
                return half4(streakDir * streak, streak, 1.0);
            }
            ENDHLSL
        }

        // Pass 1: 叠加到场景颜色
        Pass
        {
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment FragComposite

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            TEXTURE2D(_RainTempRT); SAMPLER(sampler_RainTempRT);
            TEXTURE2D(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);

            float _RainRefractionStrength;
            float _WetDarken;

            half4 FragComposite(Varyings IN) : SV_Target
            {
                half4 rainData = SAMPLE_TEXTURE2D(_RainTempRT, sampler_RainTempRT, IN.uv);
                half2 distortion = rainData.rg * _RainRefractionStrength;

                // 偏移采样场景颜色（雨滴折射）
                half3 sceneColor = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, IN.uv + distortion).rgb;

                // 湿润暗化
                sceneColor *= (1.0 - _WetDarken * rainData.b);

                // 雨滴高光（边缘亮线）
                float rainEdge = smoothstep(0.3, 0.5, rainData.b);
                sceneColor += rainEdge * 0.15;

                return half4(sceneColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**地面涟漪 Shader 片段（嵌入地面材质）：**

```hlsl
// 在地面 Pixel Shader 中叠加
TEXTURE2D(_RippleTex); SAMPLER(sampler_RippleTex);
float _RainIntensity;
float3 _WorldPos;

half3 ApplyRainRipple(half3 albedo, half3 normalWS, float2 uv)
{
    if (_RainIntensity < 0.01) return albedo;

    // 三层涟漪以不同速度/偏移平铺，打破重复感
    float2 uv1 = uv * 8.0 + _Time.y * 0.3;
    float2 uv2 = uv * 12.0 + _Time.y * 0.5 + float2(0.3, 0.7);
    float2 uv3 = uv * 5.0 - _Time.y * 0.2;

    half ripple1 = SAMPLE_TEXTURE2D(_RippleTex, sampler_RippleTex, uv1).r;
    half ripple2 = SAMPLE_TEXTURE2D(_RippleTex, sampler_RippleTex, uv2).r;
    half ripple3 = SAMPLE_TEXTURE2D(_RippleTex, sampler_RippleTex, uv3).r;

    half ripple = (ripple1 + ripple2 + ripple3) * _RainIntensity * 0.33;

    // 涟漪法线扰动
    half2 rippleNormal = float2(cos(ripple * 6.28), sin(ripple * 6.28)) * 0.1;
    normalWS.rg += rippleNormal;

    // 湿滑：降低 Roughness
    half wetness = saturate(_RainIntensity);
    albedo *= lerp(1.0, 0.7, wetness); // 湿润变暗

    return albedo;
}
```

**架构图（白板表达）：**

```
URP 渲染管线注入点
├── AfterRenderingOpaques
│   └── [WeatherFeature Pass 0] 生成雨滴扰动 RT (1/2 res)
├── AfterRenderingOpaques + 1
│   └── [WeatherFeature Pass 1] 雨滴 Blit 回 Scene Color
├── 地面 Shader 内部（不额外 Pass）
│   └── 涟漪 + 湿滑暗化（在 Base Shader 的 frag 中直接计算）
├── BeforeRenderingPostProcessing
│   └── [Volume: Color Adjustments] 湿润色调 LUT 插值
└── AfterRenderingPostProcessing
    └── 正常输出到屏幕
```

### ⚡ 实战经验

- **半分辨率 RT 是关键**：全分辨率雨滴 Pass 在中端机上要 4ms+，半分辨率 + 双线性上采样只需 0.8ms，视觉差异几乎不可见
- **深度剔除很重要**：不排除天空盒，雨滴会出现在天空中——雨滴 Pass 一定要采样 `_CameraDepthTexture` 做 depth aware
- **风向统一**：全局风向参数 `_WindDir` 要同时影响雨滴屏幕层、地面涟漪移动方向、树木摆动，否则视觉不统一
- **天气过渡插值**：晴天→雨天切换时，所有参数（_RainIntensity, _WetDarken, Volume weight）同时做 3-5 秒的 smoothstep 插值，避免突变
- **移动端 ASTC 压缩**：噪声纹理和涟漪纹理用 ASTC 4x4 压缩，每张 256x256 足够——内存增加不到 50KB
- **降级策略**：低端机（GLES3 < OpenGL 3.1）关闭雨滴层，只保留色调变化 + 地面暗化，通过 Quality Settings 控制
- **体积雨 vs 屏幕雨**：体积雨（3D 粒子）效果更好但 2-3ms，屏幕雨只需 1ms。面试时主动提出这个 tradeoff 会加分

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不会写 Renderer Feature | URP 自定义 Pass 机制 | 学 ScriptableRendererFeature + ScriptableRenderPass |
| 不知道怎么在 Pass 间传 RT | RTHandle 和临时 RT 管理 | 学 URP 2022+ 的 RTHandles API |
| 雨滴效果做得「像贴图在动」 | 缺少 depth-aware 和 motion blur | 学 Scene Depth 采样 + 方向性模糊 |
| 地面涟漪和雨滴不同步 | 全局参数传递 | 学 Global Shader Properties + WeatherManager |
| 不会做性能预算分配 | 移动端 GPU 性能分析 | 学 Snapdragon Profiler / Xcode GPU Capture |
| 不知道天气怎么切换 | Volume System / 天气状态机 | 学 URP Volume + ScriptableVolumeFeature |

### 🔗 相关问题

- 如果要做体积雨（3D 粒子），性能预算怎么分配？和屏幕雨的混合比例？
- 雪天天气怎么实现？和雨天架构有什么区别？（提示：积雪需要高度场遮罩 + 法线扰动）
- 天气系统如何和网络同步？多人游戏下雨天如何保证一致性？
- URP 2022+ 的 RTHandles 和旧版 TemporaryRT 有什么区别？迁移要注意什么？
