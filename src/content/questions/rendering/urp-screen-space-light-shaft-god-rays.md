---
title: "URP手游体积光轴：如何在移动端实现屏幕空间God Rays（丁达尔效应）？"
category: "rendering"
level: 3
tags: ["体积光", "God Rays", "光轴", "屏幕空间", "URP", "后处理", "Radial Blur", "手游优化"]
hint: "核心思路：从太阳屏幕坐标做 Radial Blur 采样 + 深度遮罩剔除被遮挡区域，不要真的射线追踪"
related: ["rendering/urp-volumetric-fog", "rendering/custom-post-processing-urp", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的开放世界手游，早晨和黄昏的时候需要太阳光穿过树林、建筑产生光轴效果（丁达尔效应）。要求在 URP 下实现，移动端要能跑，不能掉帧。给我方案。」

这是腾讯天美、网易雷火等做开放世界项目的常见渲染面试题——考察你对后处理管线、屏幕空间技巧和移动端性能的平衡能力。

### ✅ 核心要点

1. **屏幕空间方案（Screen Space Light Shaft）**：不需要真正的体积积分，从太阳屏幕坐标出发做 Radial Blur 采样
2. **深度遮罩（Depth Occlusion）**：用 Scene Depth 采样判断像素是否被建筑/山体遮挡，被遮挡的区域不参与光轴
3. **URP Renderer Feature 注入**：在 Before Rendering Post Processing 时机插入自定义 Pass
4. **Radial Blur 算法**：以太阳屏幕坐标为中心，沿径向采样叠加颜色，采样数控制质量
5. **移动端适配**：半分辨率 Render Target + 采样数降到 12-16 次 + Bloom 联动

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：太阳光从天空穿过树木/建筑 → 形成放射状光束 → 近处被建筑遮挡的地面无光束
                ↑
倒推1：光束的「放射感」→ 以太阳屏幕坐标为中心做径向模糊（Radial Blur）
倒推2：光束只出现在「天空/透明区域」→ 需要深度判断：近处像素被遮挡就不渲染光束
倒推3：光束颜色 = 太阳颜色 × 散射强度 → 需要太阳方向、太阳颜色
倒推4：光束应该有「体积衰减」→ 离太阳越远光束越弱 → 距离衰减因子
倒推5：移动端要跑 → 半分辨率计算 + 降低采样数 + 和 Bloom 共享 RT
倒推6：只在黄昏/早晨出现 → C# 控制开关 + 太阳角度判断
```

#### 知识点拆解（倒推树）

```
屏幕空间 God Rays
├── URP Renderer Feature 搭建
│   ├── 自定义 ScriptableRendererFeature + ScriptableRenderPass
│   ├── 注入时机：BeforeRenderingPostProcessing（在后处理之前）
│   ├── Render Target 分配（半分辨率 _LightShaftTex）
│   └── Blit 链路：Scene Color → Light Shaft Pass → 合成回 Scene Color
├── Radial Blur 光轴算法
│   ├── 太阳屏幕坐标计算
│   │   ├── 世界空间太阳方向 → View Space → Clip Space → NDC → Screen UV
│   │   └── 太阳在屏幕外时 clamp 到边缘（光轴方向感不变）
│   ├── 径向采样
│   │   ├── 从当前像素 UV → 到太阳 UV 的方向向量
│   │   ├── 沿该方向做 N 步采样（逐步靠近太阳）
│   │   └── 每步采样累加颜色 × 衰减权重
│   └── 采样数 vs 质量 vs 性能
│       ├── PC：32 次采样，质量高
│       ├── 移动端高端：16 次（骁龙8 Gen2+）
│       └── 移动端低端：8-12 次 + 双 Pass 各 6 次
├── 深度遮罩（Depth Occlusion）
│   ├── 采样 Scene Depth（_CameraDepthTexture）
│   ├── 将采样位置的世界深度与太阳的深度比较
│   │   ├── 简化方案：比较像素深度和太阳深度——太阳很远（深度=1.0）
│   │   └── 近处物体（深度 < 1.0）→ 挡住了太阳 → 该方向无光轴
│   └── 构建 Depth Mask：depthFactor = step(linearDepth, sunDepthThreshold)
├── 光轴着色逻辑
│   ├── 光轴颜色 = 太阳色（_SunColor）× 散射强度
│   ├── 天空区域增强（亮度 > 阈值的像素贡献更多光轴）
│   ├── 距离衰减 = 1 - distance(pixelUV, sunUV) × _Decay
│   └── 密度控制：与大气散射浓度参数联动
├── 移动端优化策略
│   ├── 半分辨率 RT（Width/2 × Height/2）
│   ├── 减少采样数（min(8, N)）
│   ├── 使用 Bilinear 采样而非高权重滤波
│   ├── 仅在太阳高度角 < 30° 时开启（黄昏/早晨）
│   └── 与 Bloom Pass 共享 Blur 中间 RT（节省内存）
└── 与现有后处理的集成
    ├── 光轴在 Bloom 之前（光轴→Bloom→Color Grading）
    ├── 光轴输出写入自定义 RT → Blit 回 Camera Color
    └── Volume Profile 控制参数（全局/局部）
```

#### 代码实现

**URP Renderer Feature（C#）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class LightShaftFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        public int downsample = 2;              // 半分辨率
        public int sampleCount = 16;            // 移动端默认16
        public Material lightShaftMaterial;     // 光轴材质
    }

    public Settings settings = new Settings();
    private LightShaftPass _pass;

    public override void Create()
    {
        _pass = new LightShaftPass(settings)
        {
            renderPassEvent = settings.renderPassEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.lightShaftMaterial == null) return;
        // 仅在白天场景且有方向光时启用
        var sun = RenderSettings.sun;
        if (sun == null || !sun.gameObject.activeInHierarchy) return;

        _pass.Setup(renderer.cameraColorTargetHandle);
        renderer.EnqueuePass(_pass);
    }

    protected override void Dispose(bool disposing)
    {
        _pass?.Dispose();
    }
}

public class LightShaftPass : ScriptableRenderPass
{
    private LightShaftFeature.Settings _settings;
    private RTHandle _lightShaftRT;
    private Material _mat;
    private static readonly int SunScreenPosID = Shader.PropertyToID("_SunScreenPos");
    private static readonly int SampleCountID = Shader.PropertyToID("_SampleCount");
    private static readonly int SunColorID = Shader.PropertyToID("_SunColor");
    private static readonly int DecayID = Shader.PropertyToID("_Decay");
    private static readonly int DensityID = Shader.PropertyToID("_Density");
    private static readonly int LightShaftTexID = Shader.PropertyToID("_LightShaftTex");

    public LightShaftPass(LightShaftFeature.Settings settings)
    {
        _settings = settings;
        _mat = settings.lightShaftMaterial;
    }

    public void Setup(RTHandle cameraColor)
    {
        // 分配半分辨率 RT
        var desc = RenderingUtils.GetTransientRTDescriptor();
        // 在 OnCameraSetup 中分配
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        var cameraData = renderingData.cameraData;
        int w = cameraData.cameraTargetDescriptor.width / _settings.downsample;
        int h = cameraData.cameraTargetDescriptor.height / _settings.downsample;

        var desc = new RenderTextureDescriptor(w, h, RenderTextureFormat.ARGBHalf, 0);
        RenderingUtils.ReAllocateIfNeeded(ref _lightShaftRT, desc, name: "_LightShaftRT");
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        var cmd = CommandBufferPool.Get("LightShaft");
        var camera = renderingData.cameraData.camera;

        // 计算太阳屏幕坐标
        Light sun = RenderSettings.sun;
        if (sun == null) { CommandBufferPool.Release(cmd); return; }

        Vector3 sunDir = sun.transform.forward;
        Vector3 sunWorldPos = camera.transform.position - sunDir * 500f;
        Vector3 sunViewPos = camera.worldToCameraMatrix.MultiplyPoint(sunWorldPos);
        Vector4 sunClip = camera.projectionMatrix * new Vector4(sunViewPos.x, sunViewPos.y, sunViewPos.z, 1);
        Vector2 sunNDC = new Vector2(sunClip.x / sunClip.w, sunClip.y / sunClip.w);
        Vector2 sunScreenUV = new Vector2(sunNDC.x * 0.5f + 0.5f, sunNDC.y * 0.5f + 0.5f);

        // 太阳在屏幕外时 clamp
        sunScreenUV = new Vector2(
            Mathf.Clamp(sunScreenUV.x, -0.5f, 1.5f),
            Mathf.Clamp(sunScreenUV.y, -0.5f, 1.5f));

        cmd.SetGlobalVector(SunScreenPosID, sunScreenUV);
        cmd.SetGlobalFloat(SampleCountID, _settings.sampleCount);
        cmd.SetGlobalVector(SunColorID, sun.color * sun.intensity);
        cmd.SetGlobalFloat(DecayID, 0.95f);
        cmd.SetGlobalFloat(DensityID, 0.8f);

        // Blit: Camera Color → Light Shaft RT（半分辨率计算）
        Blitter.BlitCameraTexture(cmd, renderingData.cameraData.renderer.cameraColorTargetHandle, _lightShaftRT, _mat, 0);
        // Blit 回 Camera Color（自动上采样）
        Blitter.BlitCameraTexture(cmd, _lightShaftRT, renderingData.cameraData.renderer.cameraColorTargetHandle);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public void Dispose()
    {
        _lightShaftRT?.Release();
        _lightShaftRT = null;
    }
}
```

**光轴 Shader（HLSL，URP 兼容）：**

```hlsl
Shader "Hidden/LightShaft"
{
    Properties
    {
        _SunScreenPos ("Sun Screen Pos", Vector) = (0.5, 0.5, 0, 0)
        _SampleCount ("Sample Count", Float) = 16
        _SunColor ("Sun Color", Color) = (1, 0.9, 0.7, 1)
        _Decay ("Decay", Range(0.8, 1.0)) = 0.95
        _Density ("Density", Range(0.3, 2.0)) = 0.8
        _SunAngleThreshold ("Sun Angle Threshold", Range(0, 1)) = 0.3
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            Name "LightShaftPass"
            ZWrite Off ZTest Always Cull Off Blend Off

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #pragma multi_compile _ _USE_DEPTH_OCCLUSION

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _SunScreenPos;
                float _SampleCount;
                float4 _SunColor;
                float _Decay;
                float _Density;
                float _SunAngleThreshold;
            CBUFFER_END

            TEXTURE2D_X(_BlitTexture); SAMPLER(sampler_BlitTexture);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings Vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 Frag(Varyings IN) : SV_Target
            {
                float2 uv = IN.uv;
                float2 sunUV = _SunScreenPos.xy;

                // 当前像素到太阳的方向
                float2 dir = sunUV - uv;
                float dist = length(dir);
                dir = normalize(dir);

                // 径向采样
                half3 accumulation = 0;
                float weight = 1.0;
                float totalWeight = 0;

                [loop]
                for (int i = 0; i < (int)_SampleCount; i++)
                {
                    float t = (float)i / _SampleCount;
                    // 从当前像素向太阳方向逐步采样
                    float2 sampleUV = uv + dir * dist * t * _Density;

                    // 采样场景颜色
                    half3 sceneCol = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, sampleUV).rgb;

                    // 深度遮罩：被近处物体挡住的区域不贡献光轴
                    #if _USE_DEPTH_OCCLUSION
                        float rawDepth = SampleSceneDepth(sampleUV);
                        float linearDepth = LinearEyeDepth(rawDepth, _ZBufferParams);
                        // 太阳在远处（depth ≈ 1），近处物体 depth < 1
                        float depthMask = saturate((linearDepth - 50.0) / 100.0); // 50-150米过渡
                        sceneCol *= depthMask;
                    #endif

                    // 天空区域贡献增强（亮度高的像素）
                    float luminance = dot(sceneCol, half3(0.2126, 0.7152, 0.0722));
                    float skyBoost = smoothstep(0.4, 1.2, luminance);

                    accumulation += sceneCol * skyBoost * weight;
                    totalWeight += weight;
                    weight *= _Decay;
                }

                accumulation /= max(0.001, totalWeight);

                // 距离衰减：离太阳越远越弱
                float radialFalloff = saturate(1.0 - dist * 1.5);
                accumulation *= radialFalloff;

                // 光轴颜色叠加
                half3 lightShaftColor = accumulation * _SunColor.rgb * _SunColor.a;

                // 与原始场景色混合（Add 叠加）
                half3 originalColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv).rgb;
                half3 finalColor = originalColor + lightShaftColor * 0.6;

                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**太阳高度角控制（C#）：**

```csharp
using UnityEngine;

public class LightShaftController : MonoBehaviour
{
    [SerializeField] private VolumeProfile profile;
    [SerializeField] private Light sun;
    [SerializeField] private float maxSunAngle = 30f;  // 太阳高度角 < 30° 时开启
    [SerializeField] private AnimationCurve intensityByAngle;

    private LightShaftVolume _volumeComponent;

    void Update()
    {
        if (sun == null) return;

        // 计算太阳高度角
        float sunElevation = Vector3.Angle(sun.transform.forward, Vector3.up) - 90f;
        sunElevation = Mathf.Abs(sunElevation);

        // 角度 < maxSunAngle 时逐渐开启光轴
        float t = 1f - Mathf.Clamp01(sunElevation / maxSunAngle);
        float intensity = intensityByAngle.Evaluate(t);

        if (_volumeComponent != null)
        {
            _volumeComponent.intensity.Override(intensity);
            _volumeComponent.active = intensity > 0.01f;
        }
    }
}
```

**效果层级表：**

| 效果层 | 技术手段 | 视觉表现 | 性能开销 |
|--------|----------|----------|----------|
| 基础光轴 | Radial Blur 16 次 | 太阳方向的放射光束 | 低（半分辨率） |
| 深度遮罩 | Scene Depth 采样 | 建筑物下方无光束 | 中（每步加一次深度采样） |
| 天空增强 | 亮度阈值平滑 | 天空区域光束更亮 | 极低 |
| 距离衰减 | UV 距离衰减因子 | 离太阳越远越弱 | 极低 |
| 颜色控制 | _SunColor 参数 | 黄昏暖色/晨光冷色 | 极低 |

### ⚡ 实战经验

- **太阳在屏幕外的处理**：不要 discard！太阳在屏幕外时光轴方向仍然有效，把 sunUV clamp 到 [-0.5, 1.5] 范围，光束依然从屏幕边缘射入
- **深度遮罩的性能代价**：每次采样 Scene Depth 都是一次纹理读取。16 次采样 × 2 次（颜色+深度）= 32 次纹理读取，移动端 Mali GPU 上可能扛不住。方案：降采样数到 8，或先做一次低分辨率的深度 Mask 预计算
- **半分辨率伪影**：半分辨率 RT 做 Radial Blur 会出现阶梯感。解决方案：最后一个 Blit 回全分辨率时用 Bilinear 上采样 + 轻微的 Gaussian 模糊
- **与 Bloom 的关系**：光轴的亮度叠加如果太强会被 Bloom 放大成白屏。在管线中确保光轴 Pass 在 Bloom 之前执行，且光轴叠加强度不要超过 0.8
- **云层穿透效果**：要做出光线穿过云层的效果，可以将云的渲染结果（_CloudOpacityTex）作为遮罩，在云的缝隙处增强光轴强度
- **与真正体积雾的区别**：屏幕空间 God Rays 是伪体积效果——它不知道三维空间中的雾浓度分布。如果需要真正的体积效果（如雾气浓淡随高度变化），需要结合 Volumetric Fog

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么在 URP 注入自定义 Pass | ScriptableRendererFeature | 学 URP Renderer Feature 架构 |
| 太阳屏幕坐标算不对 | 投影变换链 | 学 World→View→Clip→NDC→Screen UV |
| 光轴穿过建筑物 | 深度遮罩缺失 | 学 Scene Depth 采样 + LinearEyeDepth |
| 移动端太卡 | 采样数过高 | 学半分辨率 RT + 采样数控制 |
| 光轴太硬不自然 | 缺少衰减 | 学距离衰减 + 衰减系数调参 |
| 光轴在 Bloom 后叠加导致白屏 | 渲染管线顺序 | 学 URP 后处理 Pass 顺序 |

### 🔗 相关问题

- 如何与 URP 体积雾（Volumetric Fog）联动，让光轴穿过雾气时增强散射？
- 如果太阳被山完全挡住，光轴应该消失——怎么实现平滑过渡？
- 在 Unreal Engine 中怎么做同样的屏幕空间 God Rays？和 Unity 的差异？
- 如何用 Compute Shader 优化光轴采样（并行化 Radial Blur）？
