---
title: "角色低血量时屏幕边缘呼吸红光：全屏后处理 Shader 实现"
category: "shader"
level: 3
tags: ["后处理", "全屏Shader", "游戏UI", "URP", "呼吸动画", "径向遮罩"]
hint: "核心考点：屏幕空间径向遮罩 + 时间驱动的呼吸曲线 + 颜色混合 + 性能控制"
related: ["shader/hit-flash-damage-blink", "rendering/custom-post-processing-urp", "shader/radial-blur-hit-effect"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们要做一个动作游戏的低血量反馈：当角色 HP 低于 30% 时，屏幕边缘出现红色光晕，以心跳节奏（约每秒1次）呼吸闪烁，越接近死亡频率越快。在 URP 下实现这个全屏后处理效果，要求移动端也能跑。」

### ✅ 核心要点

1. **径向遮罩生成**：用屏幕 UV 到中心的距离做 Mask，只让边缘有红光
2. **呼吸节奏驱动**：用 `_Time` 参数驱动正弦波，HP 越低频率越高
3. **颜色叠加方式**：红光与原图用 `lerp` 或 `screen` 混合模式叠加
4. **性能关键**：单 Pass 全屏 Shader，无额外 RT 分配，移动端可降分辨率
5. **URP 集成**：通过 `ScriptableRendererFeature` 注入 Volume Parameter 控制开关与参数

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：低血量时屏幕边缘红光呼吸闪烁
                ↑
倒推1：红光只在边缘 → 需要径向遮罩（中心透明、边缘不透明）
倒推2：呼吸节奏 → 需要时间驱动的亮度振荡（正弦波）
倒推3：HP越低频率越快 → 需要将HP比值映射为频率参数
倒推4：叠加到画面 → 需要全屏后处理 Pass
倒推5：可控开关与参数 → 需要 URP Volume 系统 + Renderer Feature
倒推6：移动端要跑 → 单 Pass，不需要额外 RT，可半分辨率
```

#### 知识点拆解（倒推树）

```
低血量边缘呼吸红光
├── 径向遮罩（Radial Mask）
│   ├── 屏幕UV到中心的距离：distance(uv, float2(0.5, 0.5))
│   ├── 距离归一化：max(dist) = 0.707（对角线），需除以 0.707
│   ├── 遮罩曲线：smoothstep(innerRadius, outerRadius, dist)
│   └── 椭圆修正：宽高比适配（uv.x *= aspectRatio）
├── 呼吸动画（Breathing Pulse）
│   ├── 时间驱动：sin(_Time.y * frequency)
│   ├── 映射到 0~1：(sin(x) + 1) * 0.5
│   ├── HP→频率映射：lerp(0.8, 2.5, 1 - hpRatio)
│   ├── 心跳双跳模式：用 abs(sin) 的二次方模拟"咚-咚"节奏
│   └── 平滑过渡：HP刚到30%时不要突然出现，用 saturate 渐入
├── 颜色混合
│   ├── 基础红色：float3(0.8, 0.05, 0.05)
│   ├── 强度调制：color * pulseIntensity * mask
│   ├── 混合模式选择：Screen/Add/Overlay 的区别
│   └── 最终输出：lerp(originalColor, originalColor + redGlow, mask * pulse)
├── URP 集成
│   ├── ScriptableRendererFeature + ScriptableRenderPass
│   ├── Volume Component 自定义参数（强度、颜色、频率、HP比值）
│   ├── Blit 到 Camera Color Target
│   └── renderPassEvent = AfterRenderingPostProcessing
└── 移动端优化
    ├── 半分辨率后处理 RT（RTHandle 降采样）
    ├── 避免 branch（用 step/saturate 替代 if）
    └── Shader 复杂度控制：ALU < 20 指令
```

#### 代码实现

**HLSL Shader（核心逻辑）：**

```hlsl
// LowHPBreathing.shader
Shader "Hidden/Custom/LowHPBreathing"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
    }
    
    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    
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
    
    TEXTURE2D(_MainTex);
    SAMPLER(sampler_MainTex);
    
    float _Intensity;       // 整体强度 0~1
    float _HpRatio;         // HP比值 0~1（1=满血）
    float _BreathSpeed;     // 呼吸基础频率
    float3 _GlowColor;      // 红光颜色
    float _EdgeSoftness;    // 边缘软化
    float _InnerRadius;     // 径向遮罩内半径
    
    Varyings Vert(Attributes input)
    {
        Varyings output;
        output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
        output.uv = input.uv;
        return output;
    }
    
    half4 Frag(Varyings input) : SV_Target
    {
        half4 sourceColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, input.uv);
        
        // === 1. 径向遮罩 ===
        float2 centeredUV = input.uv - 0.5;
        // 椭圆修正：适配宽高比
        centeredUV.x *= _ScreenParams.x / _ScreenParams.y;
        float dist = length(centeredUV);
        float radialMask = smoothstep(_InnerRadius, _InnerRadius + _EdgeSoftness, dist);
        
        // === 2. 呼吸动画 ===
        // HP越低，频率越快：30%HP→1x, 10%HP→2.5x
        float hpFactor = saturate((0.3 - _HpRatio) / 0.3); // 0~1
        float breathFreq = lerp(_BreathSpeed, _BreathSpeed * 3.0, hpFactor);
        
        // 心跳双跳模式：用 abs(sin) ^ 4 模拟"咚-咚...咚-咚"
        float pulse = abs(sin(_Time.y * breathFreq * 3.14159));
        pulse = pow(pulse, 3.0); // 锐化，让"跳"更明显
        
        // HP渐入：刚到30%时淡入，避免突然出现
        float hpFadeIn = smoothstep(0.3, 0.25, _HpRatio); // 30%~25%渐入
        float finalIntensity = pulse * radialMask * _Intensity * hpFadeIn;
        
        // === 3. 颜色叠加 ===
        half3 glowColor = half3(_GlowColor);
        half3 finalColor = lerp(sourceColor.rgb, 
                                sourceColor.rgb + glowColor * 1.5, 
                                finalIntensity);
        
        // 边缘轻微暗角，增强危险感
        float vignette = smoothstep(0.3, 0.7, dist);
        finalColor *= (1.0 - vignette * 0.3 * hpFadeIn);
        
        return half4(finalColor, sourceColor.a);
    }
    ENDHLSL
    
    SubShader
    {
        Tags { "RenderType" = "Opaque" }
        Pass
        {
            Name "LowHPBreathing"
            ZTest Always ZWrite Off Cull Off
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            ENDHLSL
        }
    }
}
```

**URP Renderer Feature（C# 控制层）：**

```csharp
// LowHPBreathingFeature.cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class LowHPBreathingFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingPostProcessing;
        public Material material;
        [Range(0f, 1f)] public float intensity = 0.6f;
        [Range(0f, 1f)] public float hpRatio = 1f;
        public Color glowColor = new Color(0.8f, 0.05f, 0.05f, 1f);
        [Range(0.5f, 5f)] public float breathSpeed = 1.2f;
        [Range(0.2f, 0.6f)] public float innerRadius = 0.35f;
        [Range(0.05f, 0.4f)] public float edgeSoftness = 0.2f;
    }
    
    public Settings settings = new Settings();
    LowHPBreathingPass _pass;
    
    public override void Create()
    {
        _pass = new LowHPBreathingPass(settings);
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer, 
        ref RenderingData renderingData)
    {
        // 只在Game/Scene相机渲染，且HP低于阈值时才注入
        if (settings.material != null && settings.hpRatio < 0.3f)
        {
            renderer.EnqueuePass(_pass);
        }
    }
}
```

**运行时更新（从游戏系统驱动）：**

```csharp
// HealthSystemBridge.cs
public class HealthSystemBridge : MonoBehaviour
{
    public LowHPBreathingFeature feature;
    public PlayerHealth playerHealth;
    
    void Update()
    {
        float hpRatio = (float)playerHealth.CurrentHP / playerHealth.MaxHP;
        feature.settings.hpRatio = hpRatio;
        
        // 死亡时关闭
        if (hpRatio <= 0f)
            feature.SetActive(false);
    }
}
```

### ⚡ 实战经验

1. **心跳节奏比纯正弦波更有代入感**：真实的"咚-咚"双拍用 `pow(abs(sin(x)), 3)` 效果远好于简单 `sin`。很多新手直接用 sin 做呼吸，看起来像"呼吸机"而不是"心跳"
2. **HP 阈值过渡要做渐入**：30% HP 刚好触发的瞬间，如果红光突然满强度弹出，玩家会被吓到。用 `smoothstep(0.3, 0.25, hpRatio)` 做 5% 的渐入区间，体验自然很多
3. **暗角+红光配合效果翻倍**：单独红光呼吸还差点意思，加上随HP降低逐渐加深的暗角（vignette），"危险迫近"的感觉一下就出来了
4. **移动端用半分辨率 RT**：全屏后处理在手机上是最常见的性能杀手之一。低血量效果不需要像素精度，半分辨率完全看不出区别，ALU 负担降为 1/4
5. **注意不要和 Tone Mapping 冲突**：如果项目用了 ACES 或其他 Tone Mapper，红光叠加要在 Tone Mapping 之后做，否则颜色会被 Tone Mapper 吃掉

### 🎯 能力体检清单

- [ ] **如果不会做径向遮罩** → 你需要补：屏幕UV坐标系、距离场计算、smoothstep 函数
- [ ] **如果呼吸节奏看起来不对** → 你需要补：时间驱动动画、三角函数在Shader中的使用、动画曲线感觉
- [ ] **如果不知道怎么在URP中集成** → 你需要补：ScriptableRendererFeature 生命周期、Volume 参数系统、Blit 操作
- [ ] **如果在移动端性能不达标** → 你需要补：全屏Shader优化策略、RT降采样、GPU ALU 预算管理
- [ ] **如果颜色效果被Tone Mapping吃掉** → 你需要补：后处理管线顺序、Tone Mapping 原理、HDR vs LDR 工作流

### 🔗 相关问题

- 受到攻击时的屏幕一闪白（Hit Flash）如何与此效果叠加？(后处理链组合)
- 如何在非URP项目（Built-in管线）中实现同样的效果？(OnRenderImage + CommandBuffer)
- 如何让红光方向指向伤害来源方向？(方向遮罩 + 受击方向追踪)
