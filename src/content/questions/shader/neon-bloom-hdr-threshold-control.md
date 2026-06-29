---
title: "霓虹城市夜景Shader：如何控制Bloom阈值让发光物不糊成一团？"
category: "shader"
level: 3
tags: ["Bloom", "HDR", "ToneMapping", "后处理", "阈值控制", "URP", "发光特效"]
hint: "Bloom 的核心不是模糊——是 HDR 亮度阈值 + Tone Mapping 顺序，搞反了发光体会过曝"
related: ["shader/hologram-projection-effect", "rendering/custom-post-processing-urp", "shader/energy-shield-effect"]
---

## 参考答案

### 🎬 场景描述

面试官给你看一张赛博朋克风格的游戏截图——霓虹灯招牌、角色能量武器发光、UI 元素微微泛光，层次分明。然后又给你看你们项目的截图——所有发光物体糊成一坨，白色区域扩散到整个屏幕。然后问：

> "我们的 Bloom 效果总是控制不好——要么什么都不亮，要么整个画面过曝泛白。你是 TA，给我讲清楚 Bloom 的完整控制链路，以及怎样在 URP 里做一个可控的霓虹发光 Shader。"

这是网易盘古、腾讯天美、字节朝夕光年等做夜景/科幻风项目 TA 岗的高频实战题。

### ✅ 核心要点

1. **Bloom 不是模糊——是 HDR 亮度提取 + 多级模糊 + 合成**：核心控制点是亮度阈值（Threshold），不是模糊半径
2. **Tone Mapping 必须在 Bloom 之后**：HDR 空间提取亮度 → Bloom 扩散 → Tone Mapping 压回 LDR。顺序反了，Bloom 就废了
3. **发光 Shader 要输出 >1.0 的 HDR 颜色**：不靠 Bloom 后处理"猜"亮度，而是在 Shader 阶段就明确告诉管线"这里要亮"
4. **阈值不是二值开关——是软曲线**：硬阈值会产生锯齿边缘，用 smoothstep 或指数衰减做软提取
5. **多级降采样（Mip Chain）是性能关键**：URP 的 Bloom 用 6-8 级降采样纹理，每级模糊半径翻倍

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：霓虹灯清晰锐利，光晕自然扩散，不吞噬周围细节
     ↓ 倒推
合成阶段 = 原始 HDR 画面 + 处理后的 Bloom 贴图（Add 混合）
     ↓ 倒推
Bloom 贴图 = 多级模糊金字塔合成（从 1px 到 256px 的光晕层级）
     ↓ 倒推
多级降采样金字塔：
  ├── Mip 0: 原始分辨率 → 阈值提取（brightness > threshold 的像素）
  ├── Mip 1: 1/2 分辨率 → 水平+垂直高斯模糊
  ├── Mip 2: 1/4 分辨率 → 再模糊
  ├── Mip 3: 1/8 分辨率 → 大范围光晕
  ├── Mip 4-6: 更低分辨率 → 巨大范围扩散光
  └── 合成：从粗到细 upsample + lerp 权重
     ↓ 倒推
阈值提取（最关键的步骤）：
  ├── HDR 亮度计算：luminance = 0.2125R + 0.7154G + 0.0721B
  ├── 软阈值：smoothstep(threshold, threshold + knee, luminance)
  ├── 散色（Scatter）：亮度衰减 = (luminance - threshold) * intensity
  └── 颜色保留：输出 RGB * scatterAmount（而非灰度）
     ↓ 倒推
发光 Shader 输出：
  ├── 基础颜色 × 自发光倍率（Emission Multiplier > 1.0）
  ├── HDR Pipeline：渲染目标为 HDR（RGBAHalf 或 RGB9E5）
  └── Bloom Threshold 在 URP Volume 中控制截断点
     ↓ 倒推
Tone Mapping 位置（URP Volume 栈顺序）：
  HDR 画面 → Bloom → Color Adjustments → Tone Mapping → Output LDR
  注意：Tone Mapping 必须在 Bloom 之后！
```

#### 知识点拆解（倒推树）

```
Bloom 全链路控制
├── HDR 渲染管线基础
│   ├── 渲染目标格式（RGBAHalf = 16bit float per channel）
│   ├── LDR vs HDR 的亮度范围差异（0-1 vs 0-∞）
│   └── 为什么 LDR 渲染没法做好 Bloom（信息已丢失）
├── 亮度提取（Threshold 控制）
│   ├── Luminance 计算公式（Rec.709 / Rec.601）
│   ├── 硬阈值 vs 软阈值（smoothstep / knee 参数）
│   ├── 颜色通道加权提取（绿色更敏感，蓝色不敏感）
│   └── ACES 曲线预映射（对高亮区域做对数压缩后再提取）
├── 多级降采样金字塔（Downsample Pyramid）
│   ├── 双线性降采样 vs 高斯降采样（性能 vs 质量取舍）
│   ├── 降采样次数选择（移动端 4 级，PC 6-8 级）
│   ├── Upsample 合成权重（Kawase 滤波 / 高斯权重混合）
│   └── 移动端优化：用 RG11B10F 替代 RGBAHalf 省带宽
├── 发光 Shader 编写
│   ├── Emission 属性在 Standard/URP Lit 中的工作原理
│   ├── 自定义 Shader 中输出 HDR 颜色（>1.0 的 RGB）
│   ├── _EmissionColor 强度控制（可用 C# 脚本动态调节）
│   └── 区域发光控制（Vertex Color / Mask Map 控制发光范围）
├── Tone Mapping 与 Bloom 的顺序关系
│   ├── ACES Filmic Tonemapping 的 S 曲线特性
│   ├── 为什么先 Tone Map 再 Bloom 会"暗部噪点也泛光"
│   ├── URP Volume Stack 的执行顺序如何配置
│   └── 移动端用简化 ACES（近似多项式版）
└── 调试与调参
    ├── Bloom Debug View（只显示 Bloom 贴图）
    ├── 节奏控制：不同区域的 Bloom 强度（Luma Mask）
    └── 抗闪烁：TAA + Bloom 的时序稳定性
```

#### 代码实现

**URP 自定义 Bloom 阈值提取 Shader（ShaderLab + HLSL）：**

```hlsl
// BloomPrefilter.shader — 阈值提取 + 软膝
// 这一步决定 Bloom 的"素材质量"

Shader "Hidden/Custom/BloomPrefilter"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline"}
        Cull Off ZWrite Off ZTest Always

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            TEXTURE2D(_MainTex); SAMPLER(sampler_MainTex);
            float _Threshold;    // 亮度阈值，通常 0.8-1.5
            float _Knee;         // 软过渡范围，通常 0.2-0.5
            float _Intensity;    // Bloom 整体强度

            v2f vert(float3 pos : POSITION, float2 uv : TEXCOORD0)
            {
                v2f o;
                o.pos = TransformObjectToHClip(pos);
                o.uv = uv;
                return o;
            }

            // 软阈值提取函数
            float3 Prefilter(float3 hdrColor, float threshold, float knee)
            {
                float luma = dot(hdrColor, float3(0.2125, 0.7154, 0.0721));

                // 软膝曲线：threshold 以下完全截断，threshold 到 threshold+knee 之间平滑过渡
                float kneeStart = threshold;
                float kneeEnd = threshold + knee;

                // 使用 smoothstep 做平滑过渡
                float softness = smoothstep(kneeStart, kneeEnd, luma);

                // 亮度贡献 = 超过阈值的部分
                float contribution = max(luma - threshold, 0.0) / max(luma, 0.00001);
                contribution *= softness; // 平滑边缘

                // 颜色保留（不转为灰度，保持色彩信息）
                return hdrColor * contribution * _Intensity;
            }

            float4 frag(v2f i) : SV_Target
            {
                float3 col = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv).rgb;

                // 防止 NaN/Inf 污染 Bloom（移动端某些 GPU 会出 NaN）
                col = max(col, 0.0);

                float3 bloomColor = Prefilter(col, _Threshold, _Knee);
                return float4(bloomColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**URP Volume Component（C# 侧控制参数）：**

```csharp
using System;
using UnityEngine;
using UnityEngine.Rendering;

[Serializable, VolumeComponentMenu("Custom/Neon Bloom")]
public class NeonBloomComponent : VolumeComponent, IPostProcessComponent
{
    [Header("阈值控制")]
    [Tooltip("HDR 亮度阈值，超过此值才会被 Bloom 提取")]
    public ClampedFloatParameter threshold = new ClampedFloatParameter(1.0f, 0f, 5f);

    [Tooltip("软过渡范围，值越大边缘越柔和")]
    public ClampedFloatParameter knee = new ClampedFloatParameter(0.3f, 0f, 1f);

    [Header("强度控制")]
    public ClampedFloatParameter intensity = new ClampedFloatParameter(1.0f, 0f, 5f);

    [Header("降采样层级（移动端建议 4，PC 建议 6-8）")]
    public ClampedIntParameter maxIterations = new ClampedIntParameter(6, 2, 8);

    public bool IsActive() => intensity.value > 0f && maxIterations.value > 0;
}
```

**霓虹灯 Shader（输出 HDR 颜色，配合 Bloom 使用）：**

```hlsl
Shader "Custom/NeonSign"
{
    Properties
    {
        _BaseMap ("Base Texture", 2D) = "white" {}
        _EmissionMap ("Emission Mask", 2D) = "white" {}
        _EmissionColor ("Emission Color", Color) = (1, 0.2, 0.8, 1)
        [HDR] _NeonColor ("Neon HDR Color", Color) = (2.0, 0.4, 1.6, 1)
        _PulseSpeed ("Pulse Speed", Float) = 4.0
        _PulseAmount ("Pulse Amount", Range(0, 1)) = 0.2
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline"}

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            TEXTURE2D(_BaseMap);        SAMPLER(sampler_BaseMap);
            TEXTURE2D(_EmissionMap);    SAMPLER(sampler_EmissionMap);
            float4 _EmissionColor;
            float4 _NeonColor;  // HDR 颜色，分量 > 1.0
            float _PulseSpeed;
            float _PulseAmount;

            v2f vert(float3 pos : POSITION, float2 uv : TEXCOORD0)
            {
                v2f o;
                o.pos = TransformObjectToHClip(pos);
                o.uv = uv;
                return o;
            }

            float4 frag(v2f i) : SV_Target
            {
                float3 baseCol = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.uv).rgb;
                float emissionMask = SAMPLE_TEXTURE2D(_EmissionMap, sampler_EmissionMap, i.uv).r;

                // 呼吸脉冲（霓虹灯闪烁效果）
                float pulse = 1.0 + sin(_Time.y * _PulseSpeed) * _PulseAmount;

                // 关键：霓虹颜色乘以 pulse 后保持 HDR（>1.0）
                // Bloom 系统会自动提取这些 >1.0 的像素
                float3 neonEmission = _NeonColor.rgb * emissionMask * pulse;

                // 最终输出：基础色 + HDR 自发光
                float3 finalCol = baseCol + neonEmission;

                return float4(finalCol, 1.0);
            }
            ENDHLSL
        }
    }
}
```

### ⚡ 实战经验

**坑 1：LDR 渲染管线做 Bloom 是伪 Bloom**
> 项目初期为了省性能开了 LDR 渲染，结果 Bloom 效果完全不灵——因为 LDR 下颜色被钳到 0-1，阈值提取只能在 0-1 范围内切，所有亮度差异被压平。HDR 是 Bloom 的前提，移动端至少用 RGBAHalf（RGBA16F）。

**坑 2：Tone Mapping 顺序搞反**
> 美术反馈"暗部噪点也在泛光"，排查发现 URP Volume Stack 中 Tone Mapping 排在 Bloom 前面。Tone Mapping 把暗部噪点拉亮到阈值以上，Bloom 就把它们也提取了。正确顺序：HDR → Bloom → Tone Mapping → LDR Output。

**坑 3：Bloom 半径太大导致"屏幕泛白"**
> 美术把 Bloom Intensity 调到 3.0 + 降采样层级只有 2 级，导致光晕半径不够大但强度过高，看起来像加了一层白纱。正确做法：保持 Intensity 适中（0.5-1.5），增加降采样层级（6-8 级），让大半径光晕通过 Mip Chain 自然产生。

**坑 4：移动端 RG11B10F 格式的精度坑**
> 为了省带宽用了 RG11B10F（11-bit float），但 11-bit float 的精度在大面积平滑渐变处会出现 banding（色带）条纹。解法：加轻微噪声 dither（在 Prefilter Pass 中加 dither noise），用 0.5 LSB 的蓝噪声打破色带。

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明 |
|--------|-----------|-----------|
| Bloom 的完整管线步骤是什么？（从 HDR 画面到最终合成） | 理解 Bloom 全链路 | 只知道"模糊一下加回去" |
| 为什么 Tone Mapping 必须在 Bloom 之后？ | 理解 HDR/LDR 与 Bloom 的关系 | 缺少管线顺序知识 |
| HDR 渲染目标有哪些格式？移动端用哪个？ | 理解渲染目标格式选型 | 缺少 HDR 格式工程经验 |
| 软阈值（Soft Threshold / Knee）为什么比硬阈值好？ | 理解阈值提取质量 | 缺少 Bloom 调参经验 |
| 多级降采样金字塔为什么比直接大半径模糊高效？ | 理解 Bloom 性能优化 | 缺少后处理性能思维 |
| Shader 中输出 >1.0 的颜色为什么不会被截断？ | 理解 HDR 管线 | 缺少渲染管线格式认知 |
| 移动端 Bloom 的预算大约是多少 ms？ | 有平台性能直觉 | 缺少移动端性能基线 |

### 🔗 相关问题

- [URP 自定义后处理管线](../rendering/custom-post-processing-urp) — Bloom 是后处理管线的一部分
- [全息投影 Shader 效果](../shader/hologram-projection-effect) — 同样依赖 Bloom 的发光控制
- [能量护盾 Shader](../shader/energy-shield-effect) — 边缘 Fresnel 发光需要 Bloom 配合
