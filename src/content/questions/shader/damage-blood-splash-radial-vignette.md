---
title: "受击红屏脉冲：怎么用 Shader 实现径向血溅扩散 + 色相偏移的全屏受伤反馈？"
category: "shader"
level: 3
tags: ["受击反馈", "后处理", "Vignette", "色相偏移", "全屏特效", "URP", "打击感"]
hint: "不是简单红屏——径向遮罩从受击方向扩散 + HSV 色相旋转 + 边缘暗角，三层叠加才有「重伤感」"
related: ["shader/hit-flash-damage-blink", "shader/low-hp-screen-edge-pulse", "shader/radial-blur-hit-effect"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做 ARPG，角色被 Boss 重击时要有一个全屏受伤反馈——不是简单整体闪红，而是从受击方向溅出一波血色扩散，同时画面色相偏移到暖色调，边缘加暗角。整个效果 0.5 秒结束。URP 下给我完整方案。」

### ✅ 核心要点

1. **径向遮罩扩散**：从屏幕上受击点向外扩散的环形遮罩，控制红色叠加区域
2. **HSV 色相偏移**：将整个画面 RGB→HSV，H 通道向红色偏移，S 通道拉高，营造「血色滤镜」
3. **动态暗角（Vignette）**：受击瞬间暗角半径收缩再弹回，模拟视线模糊收缩感
4. **时间轴编排**：扩散波、色相偏移、暗角三者的起止时间和峰值不同，错峰才有层次感
5. **URP Renderer Feature / Fullscreen Blit**：用 `Blit` 将全屏 Shader 作用于最终画面

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色被重击 → 从受击方向径向扩散血色 → 全屏色相偏暖 → 边缘暗角收缩 → 0.5s 恢复
                ↑
倒推1：三层效果叠加 → 径向血溅遮罩 × 色相偏移层 + Vignette 暗角层
倒推2：径向扩散 → 屏幕空间 UV 到受击点的距离 → step/threshold 控制环形波纹
倒推3：色相偏移 → RGB↔HSV 转换 → H 通道加偏移量 → 再转回 RGB
倒推4：暗角收缩 → Vignette 半径随时间变化（spring 曲线：快速收缩→弹回→稳定）
倒推5：全屏后处理 → URP 中用 Renderer Feature + Fullscreen Shader（Fullscreen Shader Graph 或手写 HLSL）
倒推6：受击方向传递 → C# 将世界空间击中点投影到屏幕空间 → 传给 Shader 作为 _HitScreenPos
```

#### 知识点拆解（倒推树）

```
受击红屏脉冲
├── 径向血溅扩散
│   ├── 屏幕空间距离计算（UV - _HitScreenPos → distance）
│   ├── 扩散波纹（distance vs _SpreadRadius → smoothstep 做软边缘环）
│   ├── 噪声扰动（用噪声纹理扭曲扩散边缘，避免完美圆形）
│   └── 扩散速度曲线（_SpreadRadius 随时间 expansion: ease-out）
├── HSV 色相偏移
│   ├── RGB → HSV 转换（HLSL 内联函数或查表）
│   ├── H 通道偏移（向红色 0°/360° 偏移，偏移量随时间衰减）
│   ├── S 通道增强（饱和度拉高 1.2~1.5x）
│   ├── V 通道微降（亮度暗 5~10%，配合暗角）
│   └── HSV → RGB 转回
├── 动态 Vignette
│   ├── 标准 Vignette 公式（1 - dist(uv, 0.5) * intensity）
│   ├── 半径动画（受击瞬间收缩→spring 回弹→恢复）
│   └── 颜色 tint（暗角不是黑色，是暗红色 #330000）
├── URP 全屏后处理集成
│   ├── Renderer Feature 配置（Blit Material → Fullscreen）
│   ├── Fullscreen Shader Graph（Unity 2022.2+ 原生支持）
│   ├── 手写 HLSL（URP 14 的 FullscreenPassRendererFeature）
│   └── _BlitTexture / _BlitScaleBias 参数理解
├── C# 事件驱动
│   ├── 受击事件 → 计算屏幕空间命中点 → 设置 Shader 全局参数
│   ├── 全局 MaterialPropertyBlock 或 Shader.SetGlobalVector
│   ├── 时间管理（Coroutine / DOTween 控制各层时间线）
│   └── 多次受击叠加（不要 reset，用 max blend 叠加强度）
└── 进阶打磨
    ├── 方向性血溅（不只是径向圆，沿受击方向椭圆形扩散）
    ├── 屏幕冻结帧（hitstop 0.05s 冻结 + 红屏同步触发）
    ├── 心跳脉冲（低血量时红屏周期性脉冲，不是一次性）
    └── 音效同步（红屏峰值时刻对齐音效 punch）
```

#### 代码实现

**1. URP Fullscreen Shader（HLSL）**

```hlsl
// DamageFeedbackFullscreen.shader
Shader "Hidden/DamageFeedbackFullscreen"
{
    Properties
    {
        _BlitTexture ("Source", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }
        Pass
        {
            Name "DamageFeedback"
            ZWrite Off ZTest Always Cull Off Blend Off

            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment DamageFeedbackFrag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"

            TEXTURE2D_X(_BlitTexture);
            SAMPLER(sampler_BlitTexture);

            // --- 受击参数（C# 设置）---
            float4 _HitParams;     // xy = 屏幕空间命中点(0~1), z = 强度(0~1), w = 扩散半径
            float4 _HSVShift;      // x = 色相偏移, y = 饱和度倍率, z = 亮度倍率, w = 暗角强度
            float _VignetteRadius; // 暗角半径（动态变化）

            // RGB ↔ HSV
            float3 RGBtoHSV(float3 c)
            {
                float4 K = float4(0.0, -1.0/3.0, 2.0/3.0, -1.0);
                float4 p = lerp(float4(c.bg, K.wz), float4(c.gb, K.xy), step(c.b, c.g));
                float4 q = lerp(float4(p.xyw, c.r), float4(c.r, p.yzx), step(p.x, c.r));
                float d = q.x - min(q.w, q.y);
                float e = 1.0e-10;
                return float3(abs(q.z + (q.w - q.y) / (6.0 * d + e)), d / (q.x + e), q.x);
            }

            float3 HSVtoRGB(float3 c)
            {
                float4 K = float4(1.0, 2.0/3.0, 1.0/3.0, 3.0);
                float3 p = abs(frac(c.xxx + K.xyz) * 6.0 - K.www);
                return c.z * lerp(K.xxx, saturate(p - K.xxx), c.y);
            }

            // 简易噪声
            float hash21(float2 p)
            {
                p = frac(p * float2(123.34, 345.45));
                p += dot(p, p + 34.345);
                return frac(p.x * p.y);
            }

            half4 DamageFeedbackFrag(VaryingsFullscreen input) : SV_Target
            {
                float2 uv = input.uv;
                half3 srcColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv).rgb;

                // --- 1. 径向血溅扩散 ---
                float2 hitPos = _HitParams.xy;
                float hitStrength = _HitParams.z;
                float spreadRadius = _HitParams.w;

                float distToHit = distance(uv, hitPos);
                // 考虑屏幕宽高比，使扩散是椭圆而非完美圆
                float aspectFix = _ScreenParams.x / _ScreenParams.y;
                float2 adjustedUV = float2((uv.x - hitPos.x) * aspectFix, uv.y - hitPos.y);
                float adjustedDist = length(adjustedUV);

                // 扩散波纹边缘
                float waveEdge = smoothstep(spreadRadius, spreadRadius - 0.05, adjustedDist);
                // 噪声扰动边缘
                float noise = hash21(uv * 200.0 + _Time.yy) * 0.03;
                waveEdge = smoothstep(spreadRadius + noise, spreadRadius - 0.05 + noise, adjustedDist);
                // 波纹衰减（从命中点向外强度递减）
                float distFade = saturate(1.0 - adjustedDist / (spreadRadius + 0.01));
                float bloodMask = waveEdge * distFade * hitStrength;

                // --- 2. HSV 色相偏移 ---
                float3 hsv = RGBtoHSV(srcColor);
                // 色相向红色（H=0 或 1）偏移，受击区域偏移更强
                float hueShift = _HSVShift.x * (0.3 + bloodMask * 0.7);
                hsv.r = frac(hsv.r + hueShift + 0.5); // +0.5 让色相往红色区域靠
                // 饱和度增强
                hsv.g *= _HSVShift.y;
                // 亮度微降
                hsv.b *= _HSVShift.z;
                float3 shiftedColor = HSVtoRGB(hsv);

                // 血色叠加（径向区域更红）
                float3 bloodColor = float3(0.6, 0.02, 0.02); // 暗红色
                float3 result = lerp(shiftedColor, bloodColor, bloodMask * 0.6);

                // --- 3. 动态暗角 ---
                float2 vCenter = float2(0.5, 0.5);
                float vDist = distance(uv, vCenter);
                float vignette = smoothstep(_VignetteRadius, _VignetteRadius - 0.3, vDist);
                float3 vColor = float3(0.2, 0.0, 0.0); // 暗红色暗角
                result = lerp(result, result * vColor * 2.0, (1.0 - vignette) * _HSVShift.w * hitStrength);

                return half4(result, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**2. C# 驱动脚本**

```csharp
using UnityEngine;
using System.Collections;

[RequireComponent(typeof(Camera))]
public class DamageFeedbackController : MonoBehaviour
{
    public Material damageFeedbackMat; // 引用上面的 shader 材质
    public float duration = 0.5f;

    private bool _isActive = false;
    private float _timer = 0f;
    private Vector4 _hitParams;
    private Vector4 _hsvShift;
    private float _vignetteRadius = 1.0f;

    void OnEnable()
    {
        // 全局 Shader 参数初始化
        Shader.SetGlobalVector("_HitParams", new Vector4(0.5f, 0.5f, 0f, 0f));
        Shader.SetGlobalVector("_HSVShift", Vector4.zero);
        Shader.SetGlobalFloat("_VignetteRadius", 1.0f);
    }

    // 外部调用：hitWorldPos 是世界空间受击点
    public void TriggerDamage(Vector3 hitWorldPos, float intensity = 1f)
    {
        Vector3 screenPos = Camera.main.WorldToViewportPoint(hitWorldPos);
        if (screenPos.z < 0) screenPos = new Vector3(0.5f, 0.5f, 0);

        _hitParams.x = screenPos.x;
        _hitParams.y = screenPos.y;
        _hitParams.z = intensity;
        _hitParams.w = 0f; // 扩散半径从0开始

        if (!_isActive)
        {
            StartCoroutine(DamageSequence());
        }
        else
        {
            // 已在播放中：叠加强度，不重新启动
            _hitParams.z = Mathf.Max(_hitParams.z, intensity);
        }
    }

    IEnumerator DamageSequence()
    {
        _isActive = true;
        _timer = 0f;
        float maxSpread = 1.2f;

        while (_timer < duration)
        {
            float t = _timer / duration;
            float easedT = 1f - Mathf.Pow(1f - t, 3f); // ease-out cubic

            // 扩散半径：0 → maxSpread
            _hitParams.w = Mathf.Lerp(0f, maxSpread, easedT);
            // 强度衰减：1 → 0
            _hitParams.z = Mathf.Lerp(_hitParams.z, 0f, easedT * 0.8f);

            // 色相偏移：前期强后期弱
            float hueAmount = Mathf.Sin(t * Mathf.PI) * 0.15f; // 先增后减
            _hsvShift = new Vector4(hueAmount, 1.3f, 0.92f, 0.6f * (1f - easedT));

            // 暗角半径：快速收缩→弹回
            if (t < 0.15f)
                _vignetteRadius = Mathf.Lerp(1.0f, 0.35f, t / 0.15f); // 收缩
            else
                _vignetteRadius = Mathf.Lerp(0.35f, 1.0f, (t - 0.15f) / 0.85f); // 弹回

            // 推送到 GPU
            Shader.SetGlobalVector("_HitParams", _hitParams);
            Shader.SetGlobalVector("_HSVShift", _hsvShift);
            Shader.SetGlobalFloat("_VignetteRadius", _vignetteRadius);

            _timer += Time.deltaTime;
            yield return null;
        }

        // 重置
        Shader.SetGlobalVector("_HitParams", new Vector4(0.5f, 0.5f, 0f, 0f));
        Shader.SetGlobalVector("_HSVShift", Vector4.zero);
        Shader.SetGlobalFloat("_VignetteRadius", 1.0f);
        _isActive = false;
    }
}
```

**3. URP Renderer Feature 配置（伪代码）**

```csharp
// 在 UniversalRendererData 中添加 FullscreenPassRendererFeature
// → Material: DamageFeedbackFullscreen
// → Pass Index: 0
// → Injection Point: Before Rendering Post-Processing 或 After Rendering Transparents
// → 当 _HitParams.z > 0 时才激活（通过 bool 控制 Render Pass Event）
```

### ⚡ 实战经验

- **别用 GrabPass**：URP 下没有 GrabPass，必须走 Renderer Feature + Blit。新手最容易踩的坑就是还在找 GrabPass
- **HSV 转换有精度损失**：两次矩阵转换会掉色，调试时建议加一个 bypass 开关对比原图和效果图
- **多光源场景下注意**：全屏后处理在 HDR pipeline 下会非线性放大红色，调参时要在实际渲染路径下测，别在 Scene 视窗测
- **性能注意**：全屏 shader 在移动端开销不小。中低端机可以降低分辨率渲染后处理 RT，或者直接用 UI 层叠加简化版红屏 + Vignette
- **方向感很关键**：扩散中心点要精准对齐受击方向。如果角色被从左侧打，但红屏从右侧扩散，会非常违和。击中点的屏幕投影必须准确
- **多次受击叠加处理**：不要每次 Trigger 都重置 timer，而是维持当前播放并刷新强度和方向，让连续受击有连贯感

### 🎯 能力体检清单

- [ ] URP 下全屏后处理的正确实现方式是什么？（Renderer Feature + Blit，不是 OnRenderImage）
- [ ] RGB ↔ HSV 转换的数学过程能手写吗？
- [ ] 径向遮罩扩散如何避免完美圆形？（噪声扰动 / 椭圆变形）
- [ ] 多层效果的时间编排：扩散、色相偏移、暗角三者的起止时间应该怎样错峰？
- [ ] 如果不用全屏 shader，纯 UI 层能做简化版吗？（Canvas + Image + radial mask）
- [ ] 在移动端如何降级？（降低 RT 分辨率 / 简化为 UI 叠加）
- [ ] Shader.SetGlobalVector 和 Material.SetVector 的区别是什么？全局参数会影响所有材质吗？

### 🔗 相关问题

- [角色受击闪白](shader/hit-flash-damage-blink.md) — 角色本体的受击反馈
- [低血量屏幕边缘脉冲](shader/low-hp-screen-edge-pulse.md) — 持续型全屏受伤反馈
- [径向模糊受击效果](shader/radial-blur-hit-effect.md) — 另一种全屏受击表现手法
- [自定义后处理 URP](rendering/custom-post-processing-urp.md) — URP 后处理架构基础
