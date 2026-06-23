---
title: "角色全息投影：如何用 Shader 实现科幻感全息通讯效果？"
category: "shader"
level: 2
tags: ["Hologram", "Fresnel", "扫描线", "故障艺术", "URP", "Shader"]
hint: "Fresnel 边缘光 + 扫描线 + Glitch 偏移——三层效果叠加才是全息感的来源"
related: ["shader/dissolve-effect", "shader/energy-shield-effect", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们正在做一款科幻手游，剧情过场中需要一个角色全息投影通讯的效果——半透明、有水平扫描线、边缘有菲涅尔发光、偶尔有信号干扰的横向撕裂感。URP 下实现，要可以在角色身上直接渲染，不要后处理方案。」

### ✅ 核心要点

1. **菲涅尔边缘光**：视角越掠射越亮，形成全息的「轮廓发光」错觉
2. **扫描线纹理**：用 Sin 函数或纹理采样生成水平等距条纹，叠加到最终颜色
3. **Glitch 偏移**：时间驱动的随机 UV 偏移，模拟信号干扰
4. **半透明 + Additive**：Blend Mode 选 Additive 或 SrcAlpha One，制造发光体感
5. **参数化控制**：扫描线密度、Glitch 强度、信号强度暴露给美术调节

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：半透明角色 + 扫描线 + 边缘发光 + 偶尔 Glitch 撕裂
                ↑
倒推1：「半透明 + 发光感」→ Additive Blend + Fresnel
倒推2：「扫描线」→ UV.y * 频率 → sin → step 阈值 → 条纹 mask
倒推3：「Glitch 撕裂」→ 时间分段噪声 → UV.x 偏移（只在某些时间段触发）
倒推4：「信号干扰噪点」→ 随机噪点 mask 叠加
倒推5：「角色可见度」→ Alpha 控制整体不透明度（信号弱时闪烁）
```

#### 知识点拆解（倒推树）

```
角色全息投影
├── Shader 核心层
│   ├── Fresnel 效果（dot(normalDir, viewDir) → 1 - dot → pow）
│   ├── 扫描线生成（UV.y 数学方案 vs 纹理采样方案）
│   ├── Glitch 偏移（时间噪声 → UV 偏移 → 行错位）
│   ├── 噪点干扰（随机值叠加或 tex2D 噪声纹理）
│   └── Blend Mode（Additive vs AlphaBlend 的视觉差异）
├── URP 集成
│   ├── Surface Shader 不适用，必须 Unlit（自己控 Blend）
│   ├── Render Queue 3000（Transparent）
│   ├── SRP Batcher 兼容（CBUFFER）
│   └── 多 Pass 问题：URP 默认单 Pass，半透明不写深度
├── 表现进阶
│   ├── 信号闪烁（Sin 调制 Alpha 或随机时间窗口）
│   ├── 传导线效果（顶部到底部的扫描波纹，UV.y - time → step）
│   └── 颜色方案：青色系 / 蓝色系（全息经典配色）
└── 性能注意
    ├── 移动端 Additive 的 Overdraw 成本
    ├── 噪声纹理 vs 数学噪声的性能取舍
    └── Glitch 偏移的频率不宜太高（每秒 2-3 次足够）
```

#### 代码实现

**URP Unlit Hologram Shader（手写 HLSL）：**

```hlsl
Shader "Custom/Hologram"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _HoloColor ("Hologram Color", Color) = (0.3, 0.8, 1.0, 1.0)
        _FresnelPower ("Fresnel Power", Range(0.5, 8)) = 3.0
        _FresnelIntensity ("Fresnel Intensity", Range(0, 3)) = 1.5
        _ScanlineFreq ("Scanline Frequency", Range(50, 500)) = 200
        _ScanlineSpeed ("Scanline Speed", Range(0, 10)) = 2.0
        _ScanlineIntensity ("Scanline Intensity", Range(0, 1)) = 0.4
        _GlitchSpeed ("Glitch Speed", Range(0, 20)) = 8.0
        _GlitchIntensity ("Glitch Intensity", Range(0, 0.1)) = 0.02
        _OverallAlpha ("Overall Alpha", Range(0, 1)) = 0.8
    }
    SubShader
    {
        Tags
        {
            "RenderType"="Transparent"
            "RenderPipeline"="UniversalPipeline"
            "Queue"="Transparent"
        }

        Blend SrcAlpha One
        ZWrite Off
        Cull Back

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _HoloColor;
                float _FresnelPower;
                float _FresnelIntensity;
                float _ScanlineFreq;
                float _ScanlineSpeed;
                float _ScanlineIntensity;
                float _GlitchSpeed;
                float _GlitchIntensity;
                float _OverallAlpha;
            CBUFFER_END

            TEXTURE2D(_BaseMap); SAMPLER(sampler_BaseMap);

            // 简单随机数
            float rand(float2 co)
            {
                return frac(sin(dot(co.xy, float2(12.9898, 78.233))) * 43758.5453);
            }

            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float3 normalWS    : TEXCOORD1;
                float3 viewDirWS   : TEXCOORD2;
                float2 uv          : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);

                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                float3 positionWS = TransformObjectToWorld(IN.positionOS.xyz);
                OUT.viewDirWS = GetCameraPositionWS() - positionWS;
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);

                // Glitch：顶点阶段偏移 UV.x（按时间分段触发）
                float glitchTime = floor(_Time.y * _GlitchSpeed);
                float glitchRand = rand(float2(glitchTime, 0));
                if (glitchRand > 0.7) // 30% 时间段有 glitch
                {
                    OUT.uv.x += (rand(float2(glitchTime, IN.uv.y * 10)) - 0.5) * _GlitchIntensity * 10;
                }

                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                // === 1. 基础颜色 ===
                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);

                // === 2. Fresnel 边缘光 ===
                float3 normalDir = normalize(IN.normalWS);
                float3 viewDir = normalize(IN.viewDirWS);
                float fresnel = 1.0 - saturate(dot(normalDir, viewDir));
                fresnel = pow(fresnel, _FresnelPower);
                float fresnelTerm = fresnel * _FresnelIntensity;

                // === 3. 扫描线 ===
                float scanlineUV = IN.uv.y * _ScanlineFreq - _Time.y * _ScanlineSpeed;
                float scanline = sin(scanlineUV);
                scanline = smoothstep(0.0, 0.3, scanline); // 软化
                float scanlineTerm = lerp(1.0 - _ScanlineIntensity, 1.0, scanline);

                // === 4. 传导波纹（从上到下扫描） ===
                float sweep = frac(IN.uv.y - _Time.y * 0.3);
                sweep = pow(1.0 - sweep, 10.0); // 尖锐的高亮带
                float sweepTerm = sweep * 0.5;

                // === 5. 信号闪烁 ===
                float flicker = rand(float2(floor(_Time.y * 15), 0));
                flicker = lerp(0.85, 1.0, flicker);

                // === 6. 组合 ===
                half3 finalColor = _HoloColor.rgb;
                finalColor *= baseColor.rgb;
                finalColor += _HoloColor.rgb * fresnelTerm;   // Fresnel 叠加
                finalColor += _HoloColor.rgb * sweepTerm;     // 传导波纹
                finalColor *= scanlineTerm * flicker;

                float finalAlpha = _OverallAlpha * (0.3 + fresnelTerm * 0.7) * flicker;

                return half4(finalColor, finalAlpha);
            }
            ENDHLSL
        }
    }
}
```

**效果分层对比：**

| 效果层 | 实现方式 | 去掉后的影响 | 性能成本 |
|--------|----------|------------|----------|
| Fresnel 边缘光 | `1-dot(N,V)` → pow | 没有「全息轮廓感」 | 极低（数学运算） |
| 扫描线 | UV.y × freq → sin → step | 没有科幻感 | 极低 |
| Glitch 偏移 | 顶点阶段随机 UV.x 偏移 | 没有信号干扰感 | 低（少量随机计算） |
| 传导波纹 | UV.y - time → pow 尖锐带 | 缺少「扫描中」的动感 | 极低 |
| 信号闪烁 | 随机 Alpha 调制 | 太稳定，不真实 | 极低 |
| 噪点叠加 | 噪声纹理采样 | 缺少信号噪点质感 | 低（一次纹理采样） |

### ⚡ 实战经验

- **Additive vs AlphaBlend**：Additive 在暗背景上效果极佳，但亮场景下会「消失」。可以加一个关键字切换 Blend Mode
- **Glitch 频率控制**：不要连续 Glitch，用 `floor(time * speed)` 做时间段离散化，每段独立判断是否触发，更真实
- **扫描线方向**：水平扫描线最经典，但等距灰色横纹容易显得「土」。用 smoothstep 软化 + 加一条高亮传导带，层次感立刻提升
- **移动端性能**：这套效果全是数学运算 + 一次纹理采样，移动端完全跑得动。注意 Additive 的 Overdraw，全息角色不要超过 2 个同屏
- **颜色方案**：经典全息配色是青色 `(0.3, 0.8, 1.0)` 和红色 `(1.0, 0.3, 0.2)`（警告/反派），让美术在材质面板调

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么做边缘发光 | Fresnel 效果原理 | 复习 `dot(N, V)` → `1 - dot` → pow |
| 扫描线效果生硬（条纹太硬） | smoothstep 软化 | 练习 smoothstep / step / lerp 组合 |
| Glitch 效果不自然（一直闪） | 随机数与时间分段 | 学 floor + rand + 阈值触发的模式 |
| Additive 在亮场景看不见 | Blend Mode 理解 | 对比 Additive / Multiply / Overlay 的适用场景 |
| 不知道怎么让角色半透明但仍发光 | Blend + Alpha 关系 | 理解 SrcAlpha One 的合成公式 |

### 🔗 相关问题

- 如果要用后处理全屏 Hologram（比如 UI 全屏扫描），该怎么改？（提示：ScriptableRendererFeature + 全屏 RT）
- 全息效果中如何叠加角色的「线框模式」？（提示：Barycentric 坐标或 Geometry Shader 生成线框）
- 如何在全息角色上叠加文字信息（如名字、血条）？（提示：Shader 中采样 Text Texture 或 Camera Stencil）
