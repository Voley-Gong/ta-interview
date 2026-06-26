---
title: "水墨渲染：如何在 URP 中实现角色受击时的水墨晕染扩散效果？"
category: "shader"
level: 3
tags: ["水墨渲染", "NPR", "后处理", "噪声", "URP", "中国风", "受击特效", "边缘检测"]
hint: "三层结构——边缘检测做水墨线条 + 噪声扰动做晕染扩散 + 宣纸纹理叠加做纸感"
related: ["shader/dissolve-effect", "shader/hit-flash-damage-blink", "shader/radial-blur-hit-effect"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的国风动作游戏，角色受击时不要传统的红色闪屏，而是要做一个水墨溅射晕染的效果——从受击点开始，墨色向外扩散，边缘有毛笔飞白的质感，扩散过程有浓淡变化，最后像宣纸吸墨一样自然消散。全屏后处理或角色贴图级别都可以，给我方案。」

### ✅ 核心要点

1. **墨迹扩散遮罩**：从受击点（屏幕坐标）向外扩散的径向噪声遮罩，控制墨迹覆盖范围
2. **水墨边缘（毛笔飞白）**：用噪声调制扩散边缘，产生不规则的笔触质感，不是光滑的圆
3. **浓淡渐变**：扩散中心浓墨（高 alpha），边缘淡墨（低 alpha），叠加多层噪声做浓淡变化
4. **宣纸底纹**：叠加宣纸纹理（Paper Normal / Paper Diffuse），让墨迹看起来是渗透到纸里的
5. **边缘描线**：Sobel 边缘检测在角色轮廓处加深墨色，模拟国画的白描线条
6. **消散控制**：墨迹不是消失而是「被纸吸收」——alpha 从中心向边缘逐渐降低，配合噪声扰动

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：受击点 → 墨色溅射 → 毛笔飞白边缘 → 浓淡渐变 → 宣纸渗透感 → 自然消散
                ↑
倒推1：需要「从受击点向外扩散」→ 屏幕空间径向遮罩，中心=受击点的屏幕坐标
倒推2：需要「毛笔飞白」→ 扩散边缘不能光滑 → 用噪声+阈值化产生不规则边缘
倒推3：需要「浓淡变化」→ 多层噪声叠加（Domain Warp），模拟墨色不均匀
倒推4：需要「宣纸质感」→ 叠加宣纸法线纹理，墨迹区域的法线微扰产生纸面凹凸
倒推5：需要「白描线条」→ Sobel 边缘检测角色轮廓 → 轮廓线叠加墨色
倒推6：需要「自然消散」→ _InkProgress 参数 1→0 反向驱动，配合噪声使消散不规则
```

#### 知识点拆解（倒推树）

```
水墨晕染特效
├── 墨迹扩散遮罩（Ink Spread Mask）
│   ├── 受击点屏幕坐标（C# 传入 _HitScreenPos）
│   ├── 径向距离计算（distance(uv, _HitScreenPos)）
│   ├── 扩散进度控制（_InkProgress 0→1，驱动扩散半径）
│   └── 扩散速度调制（不同方向扩散速度不同 → 各向异性扩散）
├── 毛笔飞白边缘（Brush Stroke Edge）
│   ├── 噪声调制（Simplex Noise / Curl Noise 扰动扩散半径）
│   ├── 阈值化处理（step / smoothstep 产生硬边 → 模拟笔触断续）
│   ├── 方向性拉伸（沿受击方向的运动模糊 → 笔触方向感）
│   └── 多层叠加（大噪声做轮廓 + 小噪声做细节飞白）
├── 浓淡渐变（Ink Gradient）
│   ├── 中心浓墨（径向遮罩中心 alpha=1）
│   ├── 过渡区域（smoothstep 控制浓→淡过渡）
│   ├── Domain Warp（噪声扰动 UV → 墨色边界不规则）
│   └── 多次叠墨（模拟毛笔多次蘸墨 → 叠加 2-3 层不同噪声参数）
├── 宣纸纹理（Xuan Paper Texture）
│   ├── 宣纸法线贴图（Paper Normal → 纸面纤维凹凸）
│   ├── 纸面颜色（微黄/米白色基底）
│   ├── 墨迹渗透（墨色区域纸张颜色被「染色」→ color lerp）
│   └── 纸张纹理混合（Overlay / Soft Light 混合模式）
├── 边缘描线（Outline / 白描）
│   ├── Sobel 边缘检测（对场景深度/法线做 Sobel → 角色轮廓）
│   ├── 描线粗细调制（距离相机越近线越粗 → 透视感）
│   ├── 描线颜色（墨色，浓淡跟随光照变化）
│   └── 只在墨迹区域描线（避免全局描线影响风格）
├── 消散控制（Dissipation）
│   ├── _InkProgress 反向（1→0，从满墨到消失）
│   ├── 噪声驱动不规则消散（某些区域先消失 → 被纸吸收的感觉）
│   ├── 时间衰减曲线（ease-out → 开始消散快，后面慢慢淡出）
│   └── 消散尾声的扩散（最后阶段墨迹继续轻微扩散 → 模拟纸面毛细作用）
└── 后处理集成（URP Post-Processing）
    ├── Custom Renderer Feature（在 After Rendering 注入 Full Screen Pass）
    ├── Render Target 管理（墨迹积累 RT → 多次受击叠加）
    ├── 混合模式（墨迹 RT 与场景颜色做 Multiply/Overlay 混合）
    └── 多点受击支持（数组 _HitScreenPos[8] → 最多同时 8 个受击点）
```

#### 代码实现

**1. URP 后处理 Shader：水墨晕染**

```hlsl
Shader "Hidden/InkWash/InkSpread"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
        _PaperTex ("宣纸纹理", 2D) = "white" {}
        _PaperNormal ("宣纸法线", 2D) = "bump" {}
        _NoiseTex ("噪声纹理", 2D) = "white" {}
        _InkColor ("墨色", Color) = (0.05, 0.05, 0.08, 1.0)
        _PaperColor ("纸色", Color) = (0.95, 0.92, 0.85, 1.0)
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }

        Pass
        {
            Name "InkWashPost"

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_local _ _EDGE_DETECT _INK_ONLY _FULL_INK_WASH

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareNormalsTexture.hlsl"

            TEXTURE2D(_MainTex);       SAMPLER(sampler_MainTex);
            TEXTURE2D(_PaperTex);      SAMPLER(sampler_PaperTex);
            TEXTURE2D(_PaperNormal);   SAMPLER(sampler_PaperNormal);
            TEXTURE2D(_NoiseTex);      SAMPLER(sampler_NoiseTex);
            TEXTURE2D(_InkAccumRT);    SAMPLER(sampler_InkAccumRT); // 墨迹积累 RT

            // 从 CBUFFER 读取参数
            CBUFFER_START(UnityPerMaterial)
                float4 _MainTex_TexelSize;
                float4 _HitScreenPos[8];     // xy=屏幕坐标, z=强度, w=时间
                float _InkProgress;          // 整体消散进度 0→1
                float _InkSpreadRadius;      // 扩散半径
                float _BrushNoiseScale;      // 飞白噪声频率
                float _BrushNoiseIntensity;  // 飞白噪声强度
                float _EdgeThreshold;        // 边缘检测阈值
                float _PaperStrength;        // 宣纸纹理强度
                half4 _InkColor;
                half4 _PaperColor;
            CBUFFER_END

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
            };

            Varyings vert(Attributes input) {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                return output;
            }

            // ====== Simplex Noise (2D) ======
            float2 hash2(float2 p) {
                p = float2(dot(p, float2(127.1, 311.7)),
                           dot(p, float2(269.5, 183.3)));
                return -1.0 + 2.0 * frac(sin(p) * 43758.5453123);
            }

            float snoise(float2 p) {
                const float K1 = 0.366025404;
                const float K2 = 0.211324865;
                float2 i = floor(p + (p.x + p.y) * K1);
                float2 a = p - i + (i.x + i.y) * K2;
                float m = step(a.y, a.x);
                float2 o = float2(m, 1.0 - m);
                float2 b = a - o + K2;
                float2 c = a - 1.0 + 2.0 * K2;
                float3 h = max(0.5 - float3(dot(a,a), dot(b,b), dot(c,c)), 0.0);
                float3 n = h * h * h * h * float3(
                    dot(a, hash2(i)),
                    dot(b, hash2(i + o)),
                    dot(c, hash2(i + 1.0))
                );
                return dot(n, 1.0);
            }

            // ====== Domain Warp（多层噪声扰动） ======
            float domainWarp(float2 uv, float time) {
                float2 q = float2(snoise(uv + time * 0.1),
                                  snoise(uv + float2(5.2, 1.3)));
                float2 r = float2(snoise(uv + 4.0 * q + float2(1.7, 9.2) + time * 0.15),
                                  snoise(uv + 4.0 * q + float2(8.3, 2.8) + time * 0.12));
                return snoise(uv + 4.0 * r);
            }

            // ====== 计算单个受击点的墨迹遮罩 ======
            float computeInkMask(float2 uv, float4 hitInfo) {
                float2 hitPos = hitInfo.xy;
                float intensity = hitInfo.z;
                float age = hitInfo.w;

                // 径向距离
                float dist = distance(uv, hitPos);
                float radius = _InkSpreadRadius * (0.3 + age * 0.7); // 随时间扩散

                // 基础径向遮罩
                float radial = 1.0 - smoothstep(0.0, radius, dist);

                // ===== 飞白噪声：让边缘不规则 =====
                float2 noiseUV = uv * _BrushNoiseScale;
                float brushNoise = snoise(noiseUV) * 0.5 + 0.5;
                brushNoise = pow(brushNoise, 2.0); // 加大对比度

                // Domain Warp 让墨迹边界产生扭曲
                float warp = domainWarp(noiseUV * 0.5, age * 0.5);
                float warpedRadius = radius * (1.0 + warp * _BrushNoiseIntensity);

                float inkMask = 1.0 - smoothstep(0.0, warpedRadius, dist);

                // 飞白：在墨迹边缘叠加噪声，产生断续感
                float edgeBand = smoothstep(0.3, 0.8, inkMask) * (1.0 - smoothstep(0.8, 1.0, inkMask));
                inkMask *= (1.0 - edgeBand * (1.0 - brushNoise) * _BrushNoiseIntensity);

                // 浓淡：中心浓，边缘淡
                inkMask = pow(inkMask, 1.5);

                return inkMask * intensity;
            }

            // ====== Sobel 边缘检测 ======
            float sobelEdge(float2 uv) {
                float2 texel = _MainTex_TexelSize.xy;

                // 采样深度（用深度做边缘检测更稳定）
                float d = SampleSceneDepth(uv);
                float d_l = SampleSceneDepth(uv + float2(-texel.x, 0));
                float d_r = SampleSceneDepth(uv + float2( texel.x, 0));
                float d_u = SampleSceneDepth(uv + float2(0,  texel.y));
                float d_d = SampleSceneDepth(uv + float2(0, -texel.y));

                float gx = (d_r - d_l);
                float gy = (d_u - d_d);
                float edge = sqrt(gx * gx + gy * gy);

                return smoothstep(_EdgeThreshold, _EdgeThreshold * 2.0, edge);
            }

            half4 frag(Varyings input) : SV_Target {
                float2 uv = input.uv;

                // 1. 采样场景颜色
                half3 sceneCol = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).rgb;

                // 2. 采样墨迹积累 RT（之前帧积累的墨迹）
                float accumInk = SAMPLE_TEXTURE2D(_InkAccumRT, sampler_InkAccumRT, uv).r;

                // 3. 计算本帧各受击点的墨迹
                float frameInk = 0;
                [unroll] for (int i = 0; i < 8; i++) {
                    frameInk += computeInkMask(uv, _HitScreenPos[i]);
                }
                frameInk = saturate(frameInk);

                // 4. 合并墨迹（积累 + 本帧）
                float totalInk = saturate(accumInk * 0.92 + frameInk); // 0.92 = 每帧衰减

                // 5. 消散控制
                totalInk *= (1.0 - _InkProgress * 0.8); // _InkProgress 0→1 逐渐消散

                // 6. 飞白噪声细节
                float detail = snoise(uv * 30.0) * 0.15 + 0.85;
                totalInk *= detail;

                // 7. 宣纸纹理混合
                half3 paperCol = SAMPLE_TEXTURE2D(_PaperTex, sampler_PaperTex, uv).rgb;
                half3 paperNormal = UnpackNormal(SAMPLE_TEXTURE2D(_PaperNormal, sampler_PaperNormal, uv));
                paperCol = lerp(paperCol, _PaperColor.rgb, 0.5);

                // 8. 边缘检测（白描）
                float edge = sobelEdge(uv);

                // 9. 最终合成
                // 墨迹区域：纸色 × (1 - ink) + 墨色 × ink
                half3 inkCol = lerp(paperCol, _InkColor.rgb, totalInk);

                // 边缘描线：在墨迹区域内加深边缘
                inkCol = lerp(inkCol, _InkColor.rgb * 0.5, edge * totalInk * 0.6);

                // 宣纸法线影响（微弱的光照变化）
                float paperLight = dot(paperNormal, float3(0.3, 0.3, 1.0)) * 0.1 + 0.9;
                inkCol *= paperLight;

                // 10. 与场景颜色混合
                // 受击墨迹是叠加在场景上的 → Multiply 混合感
                half3 finalCol = lerp(sceneCol, inkCol, saturate(totalInk + edge * 0.3));

                // 宣纸纸感叠加（全局微弱叠加）
                finalCol = lerp(finalCol, finalCol * paperCol * 1.2, _PaperStrength * 0.1);

                return half4(finalCol, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**2. C# 控制脚本：受击墨迹触发**

```csharp
using UnityEngine;
using System.Collections.Generic;

public class InkWashEffectController : MonoBehaviour
{
    [Header("引用")]
    public Material inkWashMaterial;      // 上面 Shader 的材质
    public RenderTexture inkAccumRT;      // 墨迹积累 RT

    [Header("参数")]
    public float spreadRadius = 0.3f;
    public float brushNoiseScale = 15f;
    public float brushNoiseIntensity = 0.3f;
    public float edgeThreshold = 0.0002f;
    public float inkDuration = 1.5f;       // 墨迹持续时间
    public float dissipateDelay = 0.5f;    // 开始消散的延迟

    private Vector4[] hitPositions = new Vector4[8];
    private List<int> activeSlots = new List<int>();

    /// <summary>
    /// 触发水墨受击效果
    /// </summary>
    /// <param name="worldPos">受击世界坐标</param>
    /// <param name="intensity">墨迹强度 0-1</param>
    public void TriggerInk(Vector3 worldPos, float intensity = 1f)
    {
        int slot = FindFreeSlot();
        if (slot < 0) return;

        // 世界坐标 → 屏幕坐标
        Vector3 screenPos = Camera.main.WorldToScreenPoint(worldPos);
        screenPos.y = Screen.height - screenPos.y; // UV 翻转
        screenPos.x /= Screen.width;
        screenPos.y /= Screen.height;

        hitPositions[slot] = new Vector4(screenPos.x, screenPos.y, intensity, 0f);
        activeSlots.Add(slot);

        // 启动协程更新时间和消散
        StartCoroutine(UpdateInkSlot(slot));
    }

    private System.Collections.IEnumerator UpdateInkSlot(int slot)
    {
        float elapsed = 0f;

        while (elapsed < inkDuration)
        {
            elapsed += Time.deltaTime;

            var h = hitPositions[slot];
            h.w = elapsed / inkDuration; // age 归一化

            // 延迟后开始消散
            float dissipationProgress = Mathf.Max(0,
                (elapsed - dissipateDelay) / (inkDuration - dissipateDelay));
            inkWashMaterial.SetFloat("_InkProgress", dissipationProgress);

            hitPositions[slot] = h;

            if (elapsed >= inkDuration)
            {
                hitPositions[slot] = Vector4.zero;
                activeSlots.Remove(slot);
                yield break;
            }

            yield return null;
        }
    }

    private void OnRenderImage(RenderTexture src, RenderTexture dst)
    {
        // 更新 Material 参数
        inkWashMaterial.SetVectorArray("_HitScreenPos", hitPositions);
        inkWashMaterial.SetFloat("_InkSpreadRadius", spreadRadius);
        inkWashMaterial.SetFloat("_BrushNoiseScale", brushNoiseScale);
        inkWashMaterial.SetFloat("_BrushNoiseIntensity", brushNoiseIntensity);
        inkWashMaterial.SetFloat("_EdgeThreshold", edgeThreshold);

        // 双 Pass 渲染：Pass1 更新墨迹 RT，Pass2 合成到屏幕
        // Pass 1: 墨迹积累
        Graphics.Blit(inkAccumRT, inkAccumRT, inkWashMaterial, 0);

        // Pass 2: 合成
        Graphics.SetRenderTarget(inkAccumRT);
        Graphics.Blit(src, dst, inkWashMaterial, 0);
    }

    private int FindFreeSlot()
    {
        for (int i = 0; i < 8; i++)
            if (!activeSlots.Contains(i))
                return i;
        return -1;
    }
}
```

### ⚡ 实战经验

> **踩坑1：噪声函数太贵**
> Simplex Noise 在片元着色器中每个像素调用 3 次（Domain Warp）非常昂贵。手机上用预渲染的 Noise LUT（256×256 的 RGBA 噪声图），UV 平铺采样，性能提升 5x+。
>
> **踩坑2：多点受击的 VRAM 压力**
> 墨迹积累 RT 需要全屏分辨率，在 1080p 手机上 = 8MB（R8 格式）。如果游戏支持 4 个角色同时受击，考虑用 1/2 分辨率的墨迹 RT，再 Upsample 到全屏，视觉差异不大。
>
> **踩坑3：消散不自然**
> 匀速消散看起来像淡入淡出，不像「纸吸墨」。解法：消散进度用 `easeOutCubic(t)` 而不是线性，并且在消散后期继续增大扩散半径（墨迹在变淡的同时还在缓慢扩散）。
>
> **踩坑4：与角色渲染风格冲突**
> 如果游戏是写实 PBR 风格，水墨后处理会很违和。水墨效果适合 NPR / 国风 / 低多边形风格的游戏。如果是写实风格，考虑只在 UI 层做水墨转场，而不是 3D 场景中。
>
> **踩坑5：边缘检测抖动**
> Sobel 用深度做边缘检测，相机移动时远处边缘会闪烁。用 `step(threshold, edge)` 代替 `smoothstep` 做硬切，配合 TAA 可以减轻抖动。

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明盲区在 |
|--------|-----------|----------------|
| Domain Warp 是什么？为什么水墨需要它？ | 噪声扰动UV产生不规则边界 | 程序化噪声进阶 |
| 飞白效果用什么噪声参数实现？ | 高频噪声 + 高对比度 + 阈值化 | 噪声调参经验 |
| 墨迹积累 RT 为什么要用 R8 格式？ | 只存标量遮罩，省带宽 | RT 格式优化 |
| 多点受击怎么管理？ | 固定槽数组 + 超时回收 | 引擎资源管理 |
| 宣纸法线贴图怎么制作？ | 程序化生成或扫描真实宣纸 | 美术资源管线 |
| 为什么用深度做边缘检测而不是颜色？ | 深度更稳定，不受光照变化影响 | 后处理基础 |
| 如何让墨迹只在角色身上出现，不影响背景？ | 需要角色 Stencil 或深度区间判定 | 渲染管线高级 |
| 全屏后处理 vs 角色贴图级水墨，各自适用场景？ | 后处理=受击特效，贴图级=持久水墨材质 | 方案选型能力 |

### 🔗 相关问题

- [受击红闪特效](../shader/hit-flash-damage-blink) — 传统受击特效方案对比
- [溶解消失效果](../shader/dissolve-effect) — 噪声遮罩驱动的另一种应用
- [受击径向模糊](../shader/radial-blur-hit-effect) — 后处理类受击效果
- [卡通渲染 Outline](../shader/npr-outline-cartoon) — 白描线条的另一种实现思路
