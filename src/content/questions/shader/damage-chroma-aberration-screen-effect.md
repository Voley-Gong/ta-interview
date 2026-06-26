---
title: "角色受重击时全屏色差撕裂+色调分离：面试官要求你现场写Shader"
category: shader
level: 3
tags: ["后处理", "色差", "色调分离", "URP", "伤害反馈"]
hint: "RGB通道偏移 + 色阶量化，核心在于用后处理传递打击感"
related: ["hit-flash-damage-blink", "radial-blur-hit-effect", "low-hp-screen-edge-pulse"]
---

## 参考答案

### 🎬 场景描述

面试官打开一段动作游戏战斗录像，指着角色被Boss重击的瞬间说：

> "这个受击瞬间，画面出现了色差撕裂和色调分离的效果，大概持续0.3秒然后恢复。我现在给你一个URP项目，你来实现这个后处理效果。要求：
> 1. 全屏RGB通道偏移产生色差
> 2. 画面颜色被量化成几个色阶，产生复古色调分离感
> 3. 效果强度随时间从强到弱（伤害冲击→恢复）
> 4. 移动端要能跑，不能上Compute Shader"

### ✅ 核心要点

1. **色差**：对场景渲染纹理分别用不同UV偏移采样R/G/B通道
2. **色调分离**：对颜色做 `floor(color * levels) / levels` 量化
3. **时间驱动**：用一个 0→1 的权重参数控制效果强度，由C#脚本在受击时设为1然后lerp回0
4. **URP集成**：通过 ScriptableRendererFeature + Blit 实现全屏后处理

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：受击瞬间画面色差撕裂+色调分离
        ↓ 倒推
需要两步：1) RGB通道偏移采样  2) 颜色量化
        ↓ 倒推
需要一个控制参数：intensity (0=正常, 1=最强)
        ↓ 倒推
需要C#脚本：受击事件 → 设intensity=1 → 每帧lerp衰减
        ↓ 倒推
需要URP后处理管线：Renderer Feature → Blit → 自定义Shader
        ↓ 倒推
必须掌握：URP Renderer Feature机制、Blit API、Shader采样
```

#### 知识点拆解（倒推树）

```
全屏色差+色调分离后处理
├── URP 后处理机制
│   ├── ScriptableRendererFeature 生命周期
│   ├── ScriptableRenderPass.Execute()
│   ├── Blitter.BlitCameraTexture() 用法
│   └── RenderTargetIdentifier 概念
├── Shader 实现
│   ├── RGB 通道分别采样（UV偏移）
│   ├── 颜色量化公式（posterization）
│   ├── _Intensity 参数驱动画面变化
│   └── 移动端精度注意事项（half vs float）
├── C# 控制逻辑
│   ├── 单例管理器监听受击事件
│   ├── 渐变衰减（Mathf.Lerp / AnimationCurve）
│   └── 通过 Shader.GlobalSetFloat 或 VolumeParameter 传值
└── 性能意识
    ├── 全屏后处理的带宽成本
    ├── 移动端是否可以降低分辨率
    └── 何时关闭效果（intensity < 0.01 时跳过Blit）
```

#### 代码实现

**Shader（核心部分）：**

```hlsl
// DamageChromaAberration.shader
Shader "Hidden/DamageChromaAberration"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            float _Intensity;        // 0~1
            float _ChromaAmount;     // 色差偏移量，如 0.005
            float _PosterizeLevels;  // 色阶数，如 6

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings Vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                return output;
            }

            half4 Frag(Varyings input) : SV_Target
            {
                float2 uv = input.uv;
                float2 dir = float2(1.0, 0.0); // 水平方向偏移

                // 色差：R右偏、B左偏、G不动
                half r = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv + dir * _ChromaAmount * _Intensity).r;
                half g = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).g;
                half b = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv - dir * _ChromaAmount * _Intensity).b;
                half3 color = half3(r, g, b);

                // 色调分离（posterization）
                float levels = lerp(256.0, _PosterizeLevels, _Intensity); // 正常时256级=无变化
                color = floor(color * levels) / levels;

                // 效果强度混合
                half3 original = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).rgb;
                color = lerp(original, color, _Intensity);

                return half4(color, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**C# Renderer Feature（注册后处理）：**

```csharp
// DamageChromaAberrationFeature.cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class DamageChromaAberrationFeature : ScriptableRendererFeature
{
    public Shader shader;
    public float intensity;

    private Material _material;
    private DamageChromaPass _pass;

    public override void Create()
    {
        if (shader == null)
            shader = Shader.Find("Hidden/DamageChromaAberration");
        _material = CoreUtils.CreateEngineMaterial(shader);
        _pass = new DamageChromaPass(_material)
        {
            renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (_material == null || intensity < 0.01f) return; // 跳过无效帧
        _pass.Setup(intensity);
        renderer.EnqueuePass(_pass);
    }

    public void SetIntensity(float value) => intensity = value;
}

class DamageChromaPass : ScriptableRenderPass
{
    private Material _material;
    private RTHandle _tempTexture;

    public DamageChromaPass(Material mat) => _material = mat;

    public void Setup(float intensity)
    {
        _material.SetFloat("_Intensity", intensity);
        _material.SetFloat("_ChromaAmount", 0.008f);
        _material.SetFloat("_PosterizeLevels", 5f);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        var cmd = CommandBufferPool.Get("DamageChromaAberration");
        var source = renderingData.cameraData.renderer.cameraColorTargetHandle;

        // Blit：source → temp → source
        Blitter.BlitCameraTexture(cmd, source, _tempTexture, _material, 0);
        Blitter.BlitCameraTexture(cmd, _tempTexture, source);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```

**C# 控制逻辑（受击驱动）：**

```csharp
// DamageEffectController.cs
using UnityEngine;

public class DamageEffectController : MonoBehaviour
{
    public DamageChromaAberrationFeature feature;
    public float duration = 0.35f;
    public AnimationCurve falloff = AnimationCurve.EaseInOut(0, 1, 1, 0);

    private float _timer = -1f;

    void OnEnable()
    {
        Health.OnHeavyHit += TriggerEffect;
    }

    void OnDisable()
    {
        Health.OnHeavyHit -= TriggerEffect;
    }

    void TriggerEffect()
    {
        _timer = 0f;
    }

    void Update()
    {
        if (_timer < 0f) return;

        _timer += Time.deltaTime;
        float t = Mathf.Clamp01(_timer / duration);
        float intensity = falloff.Evaluate(t);
        feature.SetIntensity(intensity);

        if (t >= 1f)
        {
            _timer = -1f;
            feature.SetIntensity(0f);
        }
    }
}
```

### ⚡ 实战经验

1. **偏移方向不止水平**：高级做法是根据屏幕中心做径向偏移（`dir = normalize(uv - 0.5)`），打击感更强
2. **色阶不是越少越好**：移动端上 _PosterizeLevels 低于4会导致暗部色带（banding）太明显，建议5-8
3. **务必加 `intensity < 0.01` 跳过**：虽然不Blit，但URP Pass的Setup本身也有开销，在无效果帧应该完全跳过 `EnqueuePass`
4. **不要每帧 CreateEngineMaterial**：在 `Create()` 里创建一次，复用 material
5. **Volume 方案 vs 单例方案**：正式项目推荐用 Volume + VolumeParameter，可以和其他后处理一起管理优先级

### 🎯 能力体检清单

| 如果答不上来... | 说明盲区在 |
|---|---|
| 不知道怎么在URP里加自定义后处理 | URP Renderer Feature 机制 |
| 色差效果只想到后处理，想不到用RGB通道分离采样 | Shader 采样原理 |
| 不知道怎么让效果"逐渐恢复" | 缺乏时间驱动的参数管理思维 |
| 色调分离写不出公式 | 颜色量化数学基础 |
| 没提到移动端带宽问题 | 移动端GPU性能意识 |

### 🔗 相关问题

- [受击闪光：角色受伤害时为什么不能只改材质颜色？](hit-flash-damage-blink)
- [受击径向模糊：打击感的核心不在角色身上在屏幕上](radial-blur-hit-effect)
- [低血量屏幕边缘脉冲效果](low-hp-screen-edge-pulse)
