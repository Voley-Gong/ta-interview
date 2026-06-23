---
title: "动作游戏打击瞬间的径向模糊和速度线怎么做？面试官要即写Shader"
category: "shader"
level: 3
tags: ["径向模糊", "速度线", "打击感", "后处理", "URP", "ScreenUV", "动作游戏"]
hint: "打击感=径向模糊+速度线+顿帧——后处理三连，关键是中心偏移和屏幕UV方向控制"
related: ["rendering/custom-post-processing-urp", "shader/screen-space-rain-droplet", "rendering/motion-vectors-velocity", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官打开一段《鬼泣5》的战斗视频，指着但丁挥剑的瞬间说：

> "看到没有？挥剑的一瞬间画面中心有径向模糊，边缘有速度线，还有一个极短的顿帧。这就是打击感。我们项目是个 ARPG，策划也要这种效果。你在 URP 下给我实现：1）以角色为中心的径向模糊；2）沿运动方向的速度线；3）能被技能事件触发。先写核心 Shader，再说后处理集成。"

这是腾讯天美（怪猎手游）、网易雷火、字节朝夕光年等动作游戏项目的 **TA Shader 高频题**。考察的是后处理 Shader 功底 + 对"打击感"视觉构成的理解。

### ✅ 核心要点

1. **径向模糊本质**：以画面某点为中心，沿径向方向多次采样叠加——不是方向模糊（Directional Blur）
2. **速度线本质**：沿运动方向拉伸高频细节，用亮度阈值提取边缘后做方向模糊 + 径向遮罩
3. **中心点动态控制**：模糊中心要跟随角色屏幕位置（通过 `Camera.WorldToScreenPoint` 传入）
4. **触发控制**：通过一个全局参数（`_HitIntensity`）在顿帧瞬间从 0→1→0 快速变化
5. **性能意识**：后处理是全屏操作，移动端要做降采样 + 采样次数控制

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：挥剑瞬间画面中心径向模糊 + 边缘速度线 + 整体震屏
          ↓ 倒推
径向模糊 = 以角色屏幕坐标为中心，沿径向方向采 N 个样本做加权平均
          ↓ 倒推
速度线 = 提取画面高频细节（Sobel/亮度差），沿运动方向做方向模糊
          ↓ 倒推
运动方向 = 径向方向（从中心向外）就是天然的"速度方向"
          ↓ 倒推
触发 = C# 在攻击命中帧设置 _HitIntensity = 1，100ms 后 lerp 回 0
          ↓ 倒推
URP 集成 = ScriptableRendererFeature 注入全屏后处理 Pass
```

#### 知识点拆解（倒推树）

```
打击感后处理
├── 径向模糊（Radial Blur）
│   ├── 原理：uv = centerUV + (uv - centerUV) * scale，scale 从 1→1+offset
│   ├── 采样策略（8~16 次采样，等间距或偏移递增）
│   ├── 中心点控制（_BlurCenter.xy = 角色屏幕坐标）
│   ├── 衰减遮罩（距离中心越远模糊越强 或 越弱，看风格）
│   └── 与原画面混合（lerp(original, blurred, _HitIntensity)）
├── 速度线（Speed Lines / Motion Lines）
│   ├── 高频细节提取（亮度阈值 / Sobel 边缘检测）
│   ├── 方向模糊（沿径向方向用多次采样拉伸）
│   ├── 密度控制（阈值越高线越少越锐利）
│   ├── 径向遮罩（只在边缘出现，中心清晰）
│   └── 动画感（线条长度随 _HitIntensity 变化）
├── URP 后处理集成
│   ├── ScriptableRendererFeature + ScriptableRenderPass
│   ├── Blit 到临时 RenderTexture
│   ├── 材质参数通过 ShaderProperty 全局设置
│   └── RenderPassEvent 注入时机（After Rendering Post Processing）
├── C# 触发系统
│   ├── 攻击事件 → 角色屏幕坐标 → Shader 全局参数
│   ├── Coroutine 控制 _HitIntensity 的上升和衰减曲线
│   ├── 顿帧配合（Time.timeScale 短暂降低）
│   └── 震屏配合（Camera positional noise）
└── 性能优化
    ├── 降采样（半分辨率做模糊，再 upsample）
    ├── 采样次数动态调整（移动端 8 次，PC 16 次）
    ├── 分帧策略（不需要逐像素全精度）
    └── 仅在 _HitIntensity > 0 时执行后处理
```

#### 代码实现

**1. 径向模糊 + 速度线 Shader（URP/HLSL）**

```hlsl
Shader "Hidden/HitEffect"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }
        Cull Off ZWrite Off ZTest Always

        Pass
        {
            Name "HitEffect"

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #pragma multi_compile_local _ _QUALITY_LOW _QUALITY_HIGH

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
            };

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            float4 _MainTex_TexelSize;

            // === 全局参数（由 C# 设置）===
            float  _HitIntensity;      // 0~1，打击强度
            float2 _BlurCenter;        // 径向模糊中心（屏幕UV，0~1）
            float  _BlurStrength;      // 模糊强度（0.001~0.05）
            float  _SpeedLineThreshold;// 速度线亮度阈值（0.5~0.9）
            float  _SpeedLineIntensity;// 速度线强度

            Varyings Vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                return output;
            }

            /// 获取径向方向（从中心指向当前像素）
            inline float2 GetRadialDir(float2 uv, float2 center)
            {
                float2 dir = uv - center;
                // 处理屏幕宽高比，避免椭圆变形
                dir.x *= _MainTex_TexelSize.z / _MainTex_TexelSize.w; // aspectRatio
                return normalize(dir);
            }

            /// 径向距离（归一化）
            inline float GetRadialDist(float2 uv, float2 center)
            {
                float2 dir = uv - center;
                dir.x *= _MainTex_TexelSize.z / _MainTex_TexelSize.w;
                return length(dir);
            }

            // ========================================================
            // 径向模糊（Radial Blur）
            // ========================================================
            half3 RadialBlur(float2 uv, float2 center, float strength, float intensity)
            {
                float2 radialDir = GetRadialDir(uv, center);
                radialDir = normalize(uv - center); // 用原始 UV 方向采样

#if defined(_QUALITY_LOW)
                #define SAMPLE_COUNT 6
#else
                #define SAMPLE_COUNT 12
#endif

                half3 color = half3(0, 0, 0);
                float totalWeight = 0;

                // [0, 1] 范围采样，越靠近中心权重越高
                UNITY_UNROLL
                for (int i = 0; i < SAMPLE_COUNT; i++)
                {
                    float t = (float)i / (SAMPLE_COUNT - 1);       // 0~1
                    float offset = (t - 0.5) * strength * intensity; // 中心对称偏移

                    float2 sampleUV = uv + radialDir * offset;
                    float weight = 1.0 - abs(t - 0.5) * 2.0;  // 三角权重
                    weight = max(weight, 0.1);

                    color += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, sampleUV).rgb * weight;
                    totalWeight += weight;
                }

                return color / totalWeight;
            }

            // ========================================================
            // 速度线（Speed Lines）
            // ========================================================
            half3 SpeedLines(float2 uv, float2 center, float threshold, float intensity)
            {
                float2 radialDir = normalize(uv - center);
                float radialDist = GetRadialDist(uv, center);

                // 1. 提取高频细节：当前像素与径向方向上邻居的亮度差
                float2 texel = _MainTex_TexelSize.xy;
                float2 step = radialDir * texel * 2.0;

                half3 col     = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).rgb;
                half3 colPrev = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv - step * 4).rgb;
                half3 colNext = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv + step * 4).rgb;

                float lum     = dot(col, half3(0.299, 0.587, 0.114));
                float lumPrev = dot(colPrev, half3(0.299, 0.587, 0.114));
                float lumNext = dot(colNext, half3(0.299, 0.587, 0.114));

                // 亮度差异 = 高频信息
                float detail = abs(lum - lumPrev) + abs(lum - lumNext);

                // 2. 阈值化：只保留高对比度区域
                float lineMask = smoothstep(threshold * 0.5, threshold, detail);

                // 3. 沿径向方向拉伸（多次采样做方向模糊）
                half3 lineColor = half3(0, 0, 0);
                float lineSamples = 6;
                float stretchLen = 0.08 * intensity;

                UNITY_UNROLL
                for (int j = 0; j < 6; j++)
                {
                    float s = (float)j / (lineSamples - 1) - 0.5;
                    float2 sampleUV = uv + radialDir * s * stretchLen;
                    lineColor += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, sampleUV).rgb;
                }
                lineColor /= lineSamples;

                // 4. 径向遮罩：中心清晰，边缘出现速度线
                float radialMask = smoothstep(0.15, 0.5, radialDist);

                // 5. 与原图混合
                half3 result = lerp(col, lineColor, lineMask * radialMask * intensity);
                return result;
            }

            // ========================================================
            // 主函数
            // ========================================================
            half4 Frag(Varyings input) : SV_Target
            {
                float2 uv = input.uv;
                half3 originalColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).rgb;

                // 如果没有打击效果，直接返回原图
                // C# 端在 _HitIntensity <= 0 时不注入此 Pass，这里是安全检查
                float hitLerp = saturate(_HitIntensity);

                // === 径向模糊 ===
                half3 blurredColor = RadialBlur(uv, _BlurCenter, _BlurStrength, hitLerp);

                // === 速度线 ===
                half3 lineColor = SpeedLines(uv, _BlurCenter, _SpeedLineThreshold,
                                             _SpeedLineIntensity * hitLerp);

                // === 混合 ===
                // 先把速度线结果叠加到原图，再做径向模糊混合
                half3 speedLineResult = lerp(originalColor, lineColor, 0.7 * hitLerp);

                // 最终：原图 ↔ (速度线 + 径向模糊)
                half3 finalColor = lerp(originalColor, speedLineResult, hitLerp);
                finalColor = lerp(finalColor, blurredColor, 0.6 * hitLerp);

                // 打击瞬间提亮（punch flash）
                finalColor += hitLerp * 0.15;

                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**2. URP Renderer Feature（C# 触发与集成）**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class HitEffectFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        public Shader shader;
        [Range(6, 16)] public int sampleCount = 12;
        public bool downsample = true;
    }

    public Settings settings = new Settings();
    private HitEffectPass _pass;

    public override void Create()
    {
        if (settings.shader == null)
            settings.shader = Shader.Find("Hidden/HitEffect");

        _pass = new HitEffectPass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer,
        ref RenderingData renderingData)
    {
        // 只有 _HitIntensity > 0 时才执行
        if (HitEffectController.Intensity > 0.01f && _pass != null)
        {
            _pass.Setup(renderer.cameraColorTargetHandle);
            renderer.EnqueuePass(_pass);
        }
    }

    public override void SetupForRendering() => _pass?.Init(settings);
}
```

**3. 打击效果触发控制器**

```csharp
using System.Collections;
using UnityEngine;

/// <summary>
/// 挂在角色或战斗管理器上，攻击命中时触发后处理效果
/// 用法：在动画事件（Animation Event）或碰撞检测中调用 TriggerHit()
/// </summary>
public class HitEffectController : MonoBehaviour
{
    public static float Intensity { get; private set; }

    [Header("径向模糊参数")]
    [Range(0.01f, 0.1f)] public float blurStrength = 0.03f;

    [Header("速度线参数")]
    [Range(0.3f, 0.95f)] public float speedLineThreshold = 0.7f;
    [Range(0f, 2f)] public float speedLineIntensity = 1.2f;

    [Header("时序控制")]
    public float hitRiseTime = 0.03f;     // 从 0 到 1 的时间（极快）
    public float hitHoldTime = 0.06f;     // 保持 1 的时间（顿帧持续）
    public float hitDecayTime = 0.2f;     // 从 1 衰减到 0 的时间
    public AnimationCurve decayCurve = AnimationCurve.EaseInOut(0, 1, 1, 0);

    private Camera _mainCamera;
    private static readonly int BlurCenterID = Shader.PropertyToID("_BlurCenter");
    private static readonly int BlurStrengthID = Shader.PropertyToID("_BlurStrength");
    private static readonly int HitIntensityID = Shader.PropertyToID("_HitIntensity");
    private static readonly int SpeedLineThresholdID = Shader.PropertyToID("_SpeedLineThreshold");
    private static readonly int SpeedLineIntensityID = Shader.PropertyToID("_SpeedLineIntensity");

    void Start()
    {
        _mainCamera = Camera.main;
        Intensity = 0;
    }

    /// <summary>
    /// 在攻击命中帧调用（Animation Event 或 OnTriggerEnter）
    /// </summary>
    public void TriggerHit(Vector3 worldHitPosition, float power = 1f)
    {
        StopAllCoroutines();
        StartCoroutine(HitRoutine(worldHitPosition, power));
    }

    IEnumerator HitRoutine(Vector3 hitWorldPos, float power)
    {
        // === 1. 计算模糊中心：把命中点的世界坐标转到屏幕 UV ===
        Vector3 screenPos = _mainCamera.WorldToScreenPoint(hitWorldPos);
        Vector2 blurCenter = new Vector2(
            screenPos.x / Screen.width,
            screenPos.y / Screen.height
        );
        Shader.SetGlobalVector(BlurCenterID, blurCenter);
        Shader.SetGlobalFloat(BlurStrengthID, blurStrength * power);
        Shader.SetGlobalFloat(SpeedLineThresholdID, speedLineThreshold);
        Shader.SetGlobalFloat(SpeedLineIntensityID, speedLineIntensity * power);

        // === 2. 顿帧（TimeScale 降低）===
        float originalTimeScale = Time.timeScale;
        Time.timeScale = 0.1f;

        // === 3. 强度上升（0→1）===
        float elapsed = 0;
        while (elapsed < hitRiseTime)
        {
            // 顿帧期间用 unscaledTime 推进
            elapsed += Time.unscaledDeltaTime;
            Intensity = Mathf.Clamp01(elapsed / hitRiseTime) * power;
            Shader.SetGlobalFloat(HitIntensityID, Intensity);
            yield return null;
        }
        Intensity = power;
        Shader.SetGlobalFloat(HitIntensityID, Intensity);

        // === 4. 保持（hold）===
        float holdEnd = Time.unscaledTime + hitHoldTime;
        while (Time.unscaledTime < holdEnd)
        {
            yield return null; // 仍然在顿帧
        }

        // === 5. 恢复 timeScale ===
        Time.timeScale = originalTimeScale;

        // === 6. 强度衰减（1→0）===
        elapsed = 0;
        while (elapsed < hitDecayTime)
        {
            elapsed += Time.unscaledDeltaTime;
            float t = elapsed / hitDecayTime;
            Intensity = decayCurve.Evaluate(t) * power;
            Shader.SetGlobalFloat(HitIntensityID, Intensity);
            yield return null;
        }

        Intensity = 0;
        Shader.SetGlobalFloat(HitIntensityID, 0);
    }

    void OnDestroy()
    {
        Intensity = 0;
        Shader.SetGlobalFloat(HitIntensityID, 0);
        Time.timeScale = 1f;
    }
}
```

### ⚡ 实战经验

**打击感的"黄金公式"：顿帧 > 模糊 > 震屏**

很多新人 TA 一上来就搞复杂的 Shader，但打击感的核心是**时序**：
- **顿帧（Hit Stop）**：命中瞬间 timeScale 降到 0.05~0.15，持续 40~100ms——这一下"卡住"才是打击感的灵魂
- **径向模糊**：在顿帧恢复过程中叠加，强化"冲击扩散"的感觉
- **震屏（Screen Shake）**：配合模糊一起出现，幅度不要太大（2~5 像素偏移即可）

三者时序：顿帧 → （恢复时）径向模糊+速度线 → 衰减回正常

**移动端降级策略**

- 径向模糊采样数降到 4~6 次
- 速度线直接用径向遮罩 + 纹理动画替代（预生成的速度线纹理沿径向方向滚动）
- 降采样到半分辨率做后处理再 upsample
- 低端机只保留顿帧 + 简单震屏，砍掉所有后处理

**中心点的陷阱**

`Camera.WorldToScreenPoint` 返回的是像素坐标（0~Screen.width），要除以屏幕分辨率转成 UV。另外，如果角色在屏幕外（z < 0 或超出视锥），需要 clamp 中心点到屏幕边缘，否则会出现极端的模糊方向，画面会炸。

```csharp
// 安全的屏幕坐标转换
Vector3 screenPos = _mainCamera.WorldToScreenPoint(hitWorldPos);
if (screenPos.z < 0) screenPos = _mainCamera.WorldToScreenPoint(transform.position);
Vector2 blurCenter = new Vector2(
    Mathf.Clamp01(screenPos.x / Screen.width),
    Mathf.Clamp01(screenPos.y / Screen.height)
);
```

**面试加分：与运动模糊（Motion Blur）的区别**

面试官可能追问："这和 Motion Blur 有什么区别？"——
- **Motion Blur** 基于物体实际运动速度（速度缓冲 / Motion Vectors），是物理驱动的
- **打击感径向模糊** 是艺术化的冲击效果，不依赖物体运动，是事件驱动的
- 两者可以叠加，但打击感模糊有自己的时序曲线（瞬间爆发 → 衰减），Motion Blur 是持续性的

### 🎯 能力体检清单

| 检查项 | 如果你答不上来… |
|--------|----------------|
| 能解释径向模糊和方向模糊的区别？ | → 后处理基础概念不清 |
| 知道怎么把世界坐标转到屏幕 UV？ | → 坐标空间转换有盲区 |
| 能在 URP 中写自定义 RendererFeature？ | → URP 管线扩展能力不足 |
| 知道顿帧在打击感中的作用？ | → 游戏手感（Game Feel）理解不够 |
| 会用 Animation Event 触发 Shader 参数变化？ | → Animation 系统集成不熟 |
| 能说出移动端如何降级后处理？ | → 性能分级经验不足 |
| 知道 timeScale 降低后怎么用 unscaledDeltaTime？ | → Unity 时间机制不熟 |
| 能解释速度线的提取原理（高频细节）？ | → 图像处理基础薄弱 |

### 🔗 相关问题

- [URP 自定义后处理：从零到一实现一个完整的后处理 Pass](rendering/custom-post-processing-urp)
- [屏幕空间雨滴效果：水滴在屏幕上流淌](shader/screen-space-rain-droplet)
- [Motion Vectors / Velocity Texture：运动模糊与 TAA 的基础](rendering/motion-vectors-velocity)
- [GPU 带宽优化：移动端后处理的带宽管理](optimization/gpu-bandwidth-optimization)