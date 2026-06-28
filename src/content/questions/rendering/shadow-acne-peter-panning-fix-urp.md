---
title: "URP 自定义阴影出现 Peter Panning（阴影悬浮），如何系统性定位和修复？"
category: rendering
level: 3
tags: ["URP", "阴影", "Shadow Map", "Peter Panning", "Shadow Acne", "深度偏移", "渲染管线"]
hint: "Depth Bias 和 Normal Bias 的调参不是玄学，理解 Shadow Map 精度分布才能根治"
related: ["urp-renderer-feature", "custom-screen-space-shadow-soften", "planar-projection-shadow", "character-ground-contact-soft-shadow"]
---

## 参考答案

### 🎬 场景描述

面试官问：你们项目用 URP 自定义渲染管线，QA 反馈角色脚下"阴影飘起来了"（Peter Panning 现象），离地面有明显的缝隙。同时远处的建筑还有阴影锯齿（Shadow Acne）。你被要求修复这两个问题，但不能增加阴影贴图分辨率（性能预算已满）。你怎么做？

追问：如果 Depth Bias 和 Normal Bias 都调到极限还是不够，你有什么备选方案？

### ✅ 核心要点

- **Peter Panning 和 Shadow Acne 是一对矛盾**——加大 Bias 消除 Acne 就会产生 Peter Panning
- **理解 Bias 本质**——Depth Bias 是在深度空间偏移，Normal Bias 是在法线方向缩放
- **精度分布不均匀是根因**——远离光源的阴影区域精度更差，更容易出问题
- **系统性解决方案**——不能只调 Bias 参数，需要从 Shadow Map 精度分配、级联配置、接收面法线入手
- **备选方案**——Receiver Plane Depth Bias、Per-Pixel Bias、烘培静态阴影

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
症状：Peter Panning（阴影悬浮） + 远处 Shadow Acne（阴影条纹）
    ↓ 倒推：为什么会这样？
根因1：Depth Bias 过大 → 阴影采样点被推到表面以下 → 阴影"脱离"物体
根因2：Shadow Map 精度不足 → 远处深度量化误差大 → Acne
根因3：级联阴影（CSM）边界过渡处精度跳变 → 近处没问题远处出问题
    ↓ 倒推解决方案链路：
Level 1：正确调节 Depth Bias 和 Normal Bias（基础）
Level 2：优化 CSM 级联分割比（提升远处精度）
Level 3：自定义 Per-Triangle / Per-Pixel Bias（进阶）
Level 4：Receiver Plane Depth Bias（高级，最精确）
```

#### 知识点拆解（倒推树）

```
阴影偏移问题
├── 问题分类与诊断
│   ├── Shadow Acne（阴影条纹/自阴影）
│   │   ├── 原因：Shadow Map 深度量化误差
│   │   ├── 表现：表面出现平行条纹状阴影
│   │   └── 解决：增大 Depth Bias 或 Slope-Scale Bias
│   ├── Peter Panning（阴影悬浮）
│   │   ├── 原因：Depth Bias 过大，阴影脱离物体底部
│   │   ├── 表现：物体与阴影之间有可见缝隙
│   │   └── 解决：减小 Depth Bias 或改用 Normal Bias
│   ├── Shadow Loops / 重复阴影
│   │   ├── 原因：CSM 级联重叠
│   │   └── 解决：调整级联分割比 + Blend 过渡
│   └── Light Bleeding / 光泄漏
│       ├── 原因：多光源阴影叠加时 Bias 叠加
│       └── 解决：Per-Light Bias 独立配置
├── Shadow Map 精度分析
│   ├── 正交投影 vs 透视投影的深度分布
│   │   ├── 正交投影：深度均匀分布（Dir Light 常用）
│   │   └── 透视投影：近处精度高远处精度低
│   ├── 深度精度：16bit vs 24bit vs 32bit float
│   │   └── 移动端通常用 16bit → 精度更紧张
│   ├── Shadow Map 分辨率：1024 → 2048 → 4096
│   │   └── 不能加分辨率时怎么提升精度？
│   └── 级联阴影（CSM）精度分配
│       ├── 分割策略：Logarithmic / Uniform / Practical Split
│       ├── 级联数：2 / 3 / 4（移动端通常 2 级）
│       └── 级联 Blend：Split 颜色 / PCF 过渡
├── Bias 参数详解
│   ├── Depth Bias（深度偏移）
│   │   ├── 原理：Shadow Map 深度值 + bias 后再比较
│   │   ├── 适用：解决平面上的 Acne
│   │   └── 风险：过大 → Peter Panning
│   ├── Normal Bias（法线偏移）
│   │   ├── 原理：将采样点沿法线方向缩放
│   │   ├── 适用：解决斜面上的 Acne
│   │   └── 优势：不会产生 Peter Panning（偏移在表面上方）
│   ├── Slope-Scale Bias（斜率偏移）
│   │   ├── 原理：斜面越陡 Bias 越大
│   │   └── URP 中对应：受光方向与法线夹角
│   └── Receiver Plane Depth Bias
│       ├── 原理：基于接收面斜率计算精确偏移
│       └── 优势：最精确，几乎不产生副作用
├── URP 阴影系统
│   ├── UniversalRenderPipelineAsset → Shadows 设置
│   ├── Shadow Resolution / Distance / Cascade Count
│   ├── Main Light Shadow Cast Pass / Caster Shader
│   ├── 阴影自定义：Render Feature 注入自定义 Pass
│   └── 移动端限制：Max 2 Cascades, 16-bit depth
└── 高级方案
    ├── Per-Pixel Depth Bias（根据像素法线和距离）
    ├── Variance Shadow Map（VSM）—— 消除 Acne 但有 Light Leaking
    ├── Moment Shadow Map（MSM）—— 比 VSM 更精确
    └── 烘焙静态阴影 + 实时动态阴影混合
```

#### 代码实现

**1. URP 阴影参数调优工具（Editor）：**

```csharp
using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering.Universal;

public class ShadowBiasTuner : EditorWindow
{
    private Light directionalLight;
    private float depthBias = 1.0f;
    private float normalBias = 1.0f;
    private float nearPlane = 0.2f;

    [MenuItem("TA Tools/Shadow Bias Tuner")]
    static void Init() => GetWindow<ShadowBiasTuner>("Shadow Bias Tuner");

    void OnEnable()
    {
        directionalLight = FindObjectsByType<Light>(
            FindObjectsSortMode.None).FirstOrDefault(l => l.type == LightType.Directional);
        if (directionalLight)
        {
            depthBias = directionalLight.shadowBias;
            // NormalBias 在 URP 中通过 Additional Light Data
            var additionalData = directionalLight.GetComponent<UniversalAdditionalLightData>();
            if (additionalData) normalBias = additionalData.shadowNormalBias;
        }
    }

    void OnGUI()
    {
        GUILayout.Label("URP 阴影 Bias 调参工具", EditorStyles.boldLabel);
        GUILayout.Space(5);

        directionalLight = (Light)EditorGUILayout.ObjectField(
            "方向光", directionalLight, typeof(Light), true);

        EditorGUI.BeginChangeCheck();
        depthBias = EditorGUILayout.Slider(
            "Depth Bias", depthBias, 0f, 10f);
        normalBias = EditorGUILayout.Slider(
            "Normal Bias", normalBias, 0f, 10f);
        nearPlane = EditorGUILayout.Slider(
            "Shadow Near Plane", nearPlane, 0.01f, 5f);
        if (EditorGUI.EndChangeCheck() && directionalLight)
        {
            Undo.RecordObject(directionalLight, "Adjust Shadow Bias");
            directionalLight.shadowBias = depthBias;
            var additionalData = directionalLight.GetComponent<UniversalAdditionalLightData>();
            if (additionalData)
            {
                additionalData.shadowNormalBias = normalBias;
                additionalData.shadowNearPlane = nearPlane;
            }
            EditorUtility.SetDirty(directionalLight);
        }

        GUILayout.Space(10);
        GUILayout.Label("诊断提示:", EditorStyles.boldLabel);
        if (depthBias > 3f)
            EditorGUILayout.HelpBox(
                "⚠️ Depth Bias 过大！可能导致 Peter Panning（阴影悬浮）。\n" +
                "建议：减小 Depth Bias，改用 Normal Bias 消除 Acne。",
                MessageType.Warning);
        if (depthBias < 0.5f && normalBias < 0.5f)
            EditorGUILayout.HelpBox(
                "⚠️ 两个 Bias 都很小，可能出现 Shadow Acne（阴影条纹）。\n" +
                "建议：先增大 Normal Bias 到 1.0-2.0，再微调 Depth Bias。",
                MessageType.Warning);

        GUILayout.Space(5);
        if (GUILayout.Button("输出 CSM 级联精度分析"))
            AnalyzeCascadePrecision();
    }

    void AnalyzeCascadePrecision()
    {
        var urpAsset = UniversalRenderPipeline.asset;
        if (urpAsset == null) return;

        int cascadeCount = urpAsset.shadowCascadeCount;
        float shadowDist = urpAsset.shadowDistance;
        float shadowRes = urpAsset.shadowAtlasResolution;

        Debug.Log("===== CSM 级联精度分析 =====");
        Debug.Log($"级联数: {cascadeCount}");
        Debug.Log($"阴影距离: {shadowDist}m");
        Debug.Log($"Shadow Map 分辨率: {shadowRes}x{shadowRes}");

        // 计算每级联的世界空间精度
        // texelSize ≈ cascadeRange / shadowResolution
        for (int i = 0; i < cascadeCount; i++)
        {
            float cascadeSplit = GetCascadeSplit(urpAsset, i, cascadeCount);
            float cascadeRange = cascadeSplit * shadowDist;
            float texelWorldSize = cascadeRange / shadowRes;

            // Depth 精度（假设正交投影 + 16bit）
            float nearClip = nearPlane;
            float farClip = cascadeRange;
            float depthPrecision = (farClip - nearClip) / 65535f; // 16-bit

            Debug.Log($"级联 {i}: " +
                      $"范围 0-{cascadeRange:F1}m | " +
                      $"每纹素 {texelWorldSize * 100:F2}cm | " +
                      $"深度精度 {depthPrecision * 1000:F2}mm");
        }

        Debug.Log("建议:");
        Debug.Log("- 每纹素 < 2cm 时精度通常够用");
        Debug.Log("- 深度精度 > 5mm 时容易出 Acne");
        Debug.Log("- 如果远处级联精度不够，考虑调整分割比偏向近处");
    }

    float GetCascadeSplit(UniversalRenderPipelineAsset asset, int idx, int count)
    {
        // URP 默认分割比
        if (count == 2) return idx == 0 ? 0.25f : 1f;
        if (count == 3) return idx == 0 ? 0.1f : idx == 1 ? 0.5f : 1f;
        if (count == 4) return idx == 0 ? 0.067f : idx == 1 ? 0.2f : idx == 2 ? 0.467f : 1f;
        return 1f;
    }
}
```

**2. 自定义 Per-Pixel Depth Bias Shader（URP）：**

```hlsl
// 自定义阴影接收 Shader，包含 Per-Pixel Bias 计算
// 解决固定 Bias 无法适应所有场景的问题

Shader "Custom/URP/ShadowReceiverWithPerPixelBias"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1,1,1,1)
        _AcneRemovalBias ("Acne Removal Bias", Range(0, 0.01)) = 0.001
    }

    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }

        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE
            #pragma multi_compile _ _SHADOWS_SOFT

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS  : SV_POSITION;
                float3 normalWS    : TEXCOORD0;
                float3 positionWS  : TEXCOORD1;
                float2 uv          : TEXCOORD2;
                float4 shadowCoord : TEXCOORD3;
            };

            sampler2D _BaseMap;
            float4 _BaseMap_ST;
            half4 _BaseColor;
            float _AcneRemovalBias;

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.positionWS = TransformObjectToWorld(IN.positionOS.xyz);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);

                // 计算阴影坐标（使用 Cascade）
                VertexPositionInputs posInputs = GetVertexPositionInputs(IN.positionOS.xyz);
                OUT.shadowCoord = TransformWorldToShadowCoord(OUT.positionWS);

                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                half3 normalWS = normalize(IN.normalWS);
                half3 baseColor = tex2D(_BaseMap, IN.uv).rgb * _BaseColor.rgb;

                // === Per-Pixel Depth Bias 计算 ===
                // 根据法线与光源方向的夹角动态计算 Bias
                Light mainLight = GetMainLight(IN.shadowCoord);
                float3 lightDir = normalize(mainLight.direction);
                float NdotL = dot(normalWS, lightDir);

                // 斜面 Bias：NdotL 越小（越斜），需要的 Bias 越大
                // 公式：bias = baseBias / max(NdotL, 0.001)
                float perPixelBias = _AcneRemovalBias / max(NdotL, 0.01);

                // 沿法线方向偏移采样点（避免 Peter Panning）
                float3 biasedPosWS = IN.positionWS + normalWS * perPixelBias;
                float4 biasedShadowCoord = TransformWorldToShadowCoord(biasedPosWS);

                // 重新获取光照信息（使用偏移后的坐标）
                mainLight = GetMainLight(biasedShadowCoord);

                // 计算最终阴影
                half shadowAttenuation = mainLight.shadowAttenuation;

                // 简单 Lambert + 阴影
                half3 diffuse = baseColor * mainLight.color * saturate(NdotL) * shadowAttenuation;
                half3 ambient = half3(0.3, 0.3, 0.3) * baseColor;

                return half4(diffuse + ambient, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**3. Receiver Plane Depth Bias 实现（最精确方案）：**

```hlsl
// 在 Shadow Caster Pass 中计算基于接收面斜率的 Bias
// 这是图形学论文中的标准做法

// 原理：对于平坦接收面，Shadow Map 的深度导数是固定的
// 利用屏幕空间深度导数 ddx/ddy 计算精确的 Bias

half CalculateReceiverPlaneBias(float3 shadowPos, float2 shadowMapTexelSize)
{
    // 计算屏幕空间深度梯度
    float2 depthGradient;
    depthGradient.x = ddx(shadowPos.z);
    depthGradient.y = ddy(shadowPos.z);

    // 接收面在 Shadow Map 空间的斜率
    // 理想 Bias = dot(texelOffset, depthGradient)
    float2 texelOffset = float2(0.5, 0.5); // 中心采样

    float receiverBias = dot(texelOffset * shadowMapTexelSize * 
                             float2(shadowPos.w, shadowPos.w), 
                             depthGradient);

    // 加上固定的安全余量
    receiverBias += shadowMapTexelSize.x * 2.0; // 2 texel 安全余量

    return receiverBias;
}

// 在阴影采样函数中使用
half SampleShadowWithReceiverBias(
    TEXTURE2D_SHADOW_PARAM(_ShadowMap, sampler_ShadowMap),
    float3 shadowCoord,
    float2 shadowMapTexelSize)
{
    // 计算接收面偏移
    half bias = CalculateReceiverPlaneBias(shadowCoord, shadowMapTexelSize);

    // 应用偏移后再采样
    float3 biasedCoord = shadowCoord;
    biasedCoord.z -= bias; // 注意方向

    return SAMPLE_TEXTURE2D_SHADOW(_ShadowMap, sampler_ShadowMap, biasedCoord);
}
```

### ⚡ 实战经验

**Bias 调参速查表（URP 方向光）：**

| 场景类型 | Depth Bias | Normal Bias | 备注 |
|---------|-----------|-------------|------|
| 室内（短距离） | 0.5-1.0 | 0.5-1.0 | 精度充足，小 Bias 即可 |
| 室外近处（角色） | 1.0-2.0 | 1.0-2.0 | 注意角色脚部接触阴影 |
| 室外远处（建筑） | 2.0-5.0 | 1.5-3.0 | 需要 CSM 保证精度 |
| 大场景（开放世界） | 3.0-8.0 | 2.0-5.0 | 必须配合自定义 Bias |
| 移动端（16-bit depth） | ×1.5-2.0 倍 | ×1.5-2.0 倍 | 精度更差需要更大 Bias |

**常见陷阱：**
- **同时有 Acne 和 Peter Panning**：这说明 Bias 参数在临界点附近，不能靠调参解决，需要提升 Shadow Map 精度（CSM 分割优化或 Receiver Plane Bias）
- **CSM 级联交界处闪烁**：级联 Blend 区域设置过窄，通常需要 5-10% 的过渡区
- **移动端比 PC 严重**：移动端通常使用 16-bit depth texture，精度只有 PC 的一半，需要更大 Bias
- **阴影跟随角色移动时抖动**：Shadow Map 的世界空间位置没有做稳定化（Snapping to Texel Grid），导致每帧采样偏移

### 🎯 能力体检清单

- [ ] 你能解释 Depth Bias 和 Normal Bias 的区别以及各自的副作用吗？
- [ ] 你知道 Peter Panning 和 Shadow Acne 产生的根本原因吗？
- [ ] 你会计算给定 Shadow Map 分辨率和距离下的世界空间精度吗？
- [ ] 你了解 CSM 的三种分割策略（Logarithmic / Uniform / Practical Split）的优缺点吗？
- [ ] 你能实现一个基于屏幕空间导数的 Receiver Plane Depth Bias 吗？
- [ ] 你知道移动端 URP 阴影系统的限制（最大级联数、depth format）吗？
- [ ] 你理解为什么 Normal Bias 不会产生 Peter Panning 吗？

### 🔗 相关问题

- [URP 自定义 Renderer Feature 实战](../rendering/urp-renderer-feature.md)
- [自定义屏幕空间阴影柔化](../rendering/custom-screen-space-shadow-soften.md)
- [角色接地软阴影实现](../rendering/character-ground-contact-soft-shadow.md)
- [平面投影阴影方案](../rendering/planar-projection-shadow.md)
