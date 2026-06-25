---
title: "美术做的PBR材质在PC预览完美，上手机后金属变塑料、颜色偏蓝——你怎么排查和解决？"
category: "technical-art"
level: 3
tags: ["PBR", "色彩空间", "Gamma/Linear", "移动端渲染", "贴图压缩", "Tonemapping"]
hint: "九成是色彩空间不一致+贴图压缩精度损失+Tonemapping差异，三者叠加导致跨平台材质不一致"
related: ["technical-art/pbr-material-authoring", "technical-art/mobile-texture-compression", "technical-art/mobile-normal-map-compression"]
---

## 参考答案

### 🎬 场景描述

面试官给你看两张对比图——左边是 Unity 编辑器里 PBR 材质球，金属高光锐利、反射清晰、色调准确；右边是真机截图（骁龙 888），同一个材质球看起来发蓝、金属感消失了（像塑料）、高光发散。

> "美术跟我说材质在手机上'不对'，但说不清楚哪里不对。你是 TA，给我一套系统性的排查流程，找出跨平台 PBR 差异的根因并修复。"

这是叠纸、米哈游等美术品质驱动的项目必考题——考验你对**色彩管线、移动端渲染差异、贴图精度**的综合理解。

### ✅ 核心要点

1. **色彩空间（Gamma vs Linear）是头号嫌疑犯**：URP 默认 Linear，但移动端某些渲染路径或后处理可能无意中做了 Gamma 校正
2. **贴图压缩精度损失**：ASTC 6×6 压缩会使粗糙度/金属度贴图的精度量化，导致菲涅尔和高光分布偏移
3. **Tonemapping 不一致**：PC 用 ACES，移动端为了性能可能用了更简单的 Reinhard 或直接 None
4. **环境光照差异**：PC 可能用了 IBL（环境反射探针），移动端降级为 SH（球谐光照），金属反射来源完全不同
5. **Shader 分支差异**：`#pragma multi_compile` 可能导致移动端走了简化路径

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
症状：手机上金属→塑料，颜色偏蓝
     ↓ 倒推
金属感来源 = 高反射率 + 低粗糙度 → 镜面反射占主导
手机上金属感消失 = 镜面反射出了问题
     ↓ 倒推
镜面反射的四个环节逐一排查：
  ├── 环节1：输入（贴图精度 → 压缩是否破坏了 Metalness/Roughness 值）
  ├── 环节2：计算（Shader 数学是否一致 → 色彩空间/光照模型差异）
  ├── 环节3：环境（IBL 反射探针是否降级 → 移动端可能没有 CubeMap 反射）
  └── 环节4：输出（Tonemapping → ACES vs None 的色彩映射差异）
     ↓ 倒推
系统性排查流程：
  Step 1：在 PC 和手机上截图同一帧，逐像素对比
  Step 2：检查色彩空间设置（Project Settings → Player → Color Space）
  Step 3：对比贴图压缩前后的像素差异（ASTC vs uncompressed）
  Step 4：检查 Shader 的 multi_compile 分支是否一致
  Step 5：验证 IBL Reflection Probe 是否在移动端被剔除
  Step 6：Tonemapping 设置对比
```

#### 知识点拆解（倒推树）

```
PBR 跨平台一致性
├── 色彩管线（Color Pipeline）
│   ├── Linear vs Gamma 工作流
│   │   ├── Linear：物理正确，光照计算在线性空间
│   │   ├── Gamma 2.2 校正在输出时进行
│   │   └── 常见错误：sRGB 贴图未标记 → 双重 Gamma 校正
│   ├── 色彩空间转换链
│   │   ├── 贴图 sRGB → 引擎线性 → 后处理线性 → 输出 sRGB
│   │   └── 每一步错误都会导致色偏
│   └── Unity URP 的 Color Space 设置验证
│       ├── Player Settings → Color Space = Linear
│       └── 贴图的 sRGB 勾选规则（Base Color = sRGB，Normal/Metal/Rough = Linear）
│
├── 贴图精度问题
│   ├── PBR 贴图的通道规范
│   │   ├── BaseColor（sRGB）：颜色信息
│   │   ├── Normal（Linear）：法线偏移
│   │   ├── Metallic（Linear）：0 或 1，金属度
│   │   ├── Roughness（Linear）：0-1，粗糙度（最敏感的通道！）
│   │   └── AO（Linear）：环境遮蔽
│   ├── ASTC 压缩对 Roughness 的影响
│   │   ├── ASTC 6×6 → 每像素 0.89 bpp → 精度约 7-8 bit
│   │   ├── Roughness 精度不足 → 高光分布发散 → 金属感丢失
│   │   └── 解决方案：Roughness 用单独通道 + 无压缩或 ASTC 12×12 for RGBA
│   ├── 移动端 PBR 贴图打包策略
│   │   ├── 方案A：Metallic + Smoothness + AO + Detail 打包到 ORM 贴图
│   │   ├── 方案B：Roughness 单独一张 ASTC 4×4（高精度）
│   │   └── 方案C：用 LUT（查找表）替代高精度贴图
│   └── 法线贴图压缩的精度问题（BC5 vs ASTC）
│
├── 环境光照（IBL）差异
│   ├── Specular IBL（环境反射探针）
│   │   ├── PC：256×256×6 CubeMap + Prefiltered Mip Chain
│   │   ├── 移动端降级：64×64 CubeMap 或 SH9 替代
│   │   └── 没有反射探针 → 金属表面没有环境反射 → 看起来像哑光塑料
│   ├── Diffuse IBL（球谐光照 SH）
│   │   ├── PC：SH5（25 系数）或 SH3（9 系数）
│   │   ├── 移动端：SH3 或纯恒定环境光
│   │   └── SH 精度不足 → 暗部偏色（常见蓝色偏移）
│   └── Reflection Probe 的刷新频率与烘焙策略
│
├── Shader 分支差异
│   ├── #pragma multi_compile 的平台分支
│   │   ├── _SPECULARHIGHLIGHTS_OFF（移动端常见）
│   │   ├── _ENVIRONMENTREFLECTIONS_OFF
│   │   └── _GLOSSYREFLECTIONS_OFF
│   ├── URP 移动端默认关闭的特性
│   │   ├── 高品质反射 → 回退到低精度 CubeMap
│   │   ├── 区域光 → 不支持
│   │   └── 多光源阴影 → 1 盏或禁用
│   └── Shader Keyword 限制（移动端 256 keywords limit）
│
├── Tonemapping 差异
│   ├── ACES Filmic（电影级，PC/主机标准）
│   │   ├── 压缩高光，保留中间调
│   │   └── 整体色调偏暖（但正确）
│   ├── Reinhard（简化版）
│   │   ├── 高光压缩更少 → 金属高光发散
│   │   └── 色彩映射不自然
│   ├── None（无色调映射）
│   │   ├── 高光直接截断 → 金属高光区域全白
│   │   └── 色彩整体偏冷蓝（因为暗部没有暖色补偿）
│   └── 自定义 Tonemapping 曲线的跨平台一致性
│
└── 排查工具链
    ├── Frame Debug 逐 Draw Call 对比
    ├── RenderDoc 截取 PC 和移动端同一帧
    │   ├── 对比贴图采样值（Texture Viewer → 像素拾取）
    │   └── 对比 Shader 常量（Pipeline State → Constants）
    ├── 在引擎内画一个"校准球"
    │   ├── Metalness=1, Roughness=0.2 的纯金属球
    │   └── 对比不同平台的高光形状和亮度
    └── Build Settings 中 Shader Compiler 的差异（HLSLcc vs GLSL）
```

#### 代码实现

**1. 校准用的"跨平台材质检测球" Shader：**

```hlsl
// pbr_calibration_sphere.shader
// 用于在所有平台上输出完全一致的 PBR 理想结果
Shader "Custom/PBR_Calibration"
{
    Properties
    {
        _BaseColor   ("Base Color", Color) = (0.8, 0.2, 0.1, 1)
        _Metallic    ("Metallic", Range(0,1)) = 1.0
        _Roughness   ("Roughness", Range(0,1)) = 0.15
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes { float4 posOS : POSITION; float3 nOS : NORMAL; };
            struct Varyings  { float4 posCS : SV_POSITION; float3 nWS : TEXCOORD0; };

            half4 _BaseColor;
            half _Metallic;
            half _Roughness;

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.posCS = TransformObjectToHClip(IN.posOS.xyz);
                OUT.nWS = TransformObjectToWorldNormal(IN.nOS);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                // 用纯常量材质，排除贴图干扰
                half3 albedo = _BaseColor.rgb;
                half metallic = _Metallic;
                half roughness = _Roughness;
                half3 N = normalize(IN.nWS);

                // 标准 PBR BRDF
                Light mainLight = GetMainLight();
                half3 L = mainLight.direction;
                half3 V = GetWorldSpaceNormalizeViewDir(IN.posCS);
                half3 H = normalize(L + V);

                half NdotL = saturate(dot(N, L));
                half NdotV = saturate(dot(N, V));
                half NdotH = saturate(dot(N, H));
                half VdotH = saturate(dot(V, H));

                // Specular BRDF (Cook-Torrance)
                half a = roughness * roughness;
                half a2 = a * a;
                half d = (NdotH * a2 - NdotH) * NdotH + 1.0;
                half D = a2 / max(3.14159 * d * d, 1e-4);

                half k = (roughness + 1) * (roughness + 1) / 8.0;
                half G = NdotV / (NdotV * (1-k) + k);

                half3 F0 = lerp(0.04, albedo, metallic);
                half3 F = F0 + (1.0 - F0) * pow(1.0 - VdotH, 5.0);

                half3 specular = (D * G * F) / max(4.0 * NdotL * NdotV, 1e-4);
                half3 kd = (1.0 - F) * (1.0 - metallic);
                half3 diffuse = kd * albedo / 3.14159;

                half3 color = (diffuse + specular) * mainLight.color * NdotL;

                // 环境光（确保 IBL 一致）
                half3 ambient = SampleSH(N) * albedo * (1.0 - metallic);
                color += ambient;

                return half4(color, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**2. 跨平台一致性检测脚本：**

```csharp
// PBRConsistencyChecker.cs
// 自动在 PC 和手机上渲染同一场景，截图并逐像素对比
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;

public class PBRConsistencyChecker : EditorWindow
{
    [MenuItem("TA Tools/PBR Consistency Check")]
    static void Capture()
    {
        // 确保用相同色彩空间渲染
        Debug.Log($"Color Space: {QualitySettings.activeColorSpace}");
        Debug.Log($"Color Gamut: {PlayerSettings.colorSpace}");

        // 渲染校准球并截图
        var cam = Camera.main;
        var rt = new RenderTexture(512, 512, 24, RenderTextureFormat.ARGB32);
        cam.targetTexture = rt;
        cam.Render();

        // 读取像素并保存
        RenderTexture.active = rt;
        var tex = new Texture2D(512, 512, TextureFormat.RGB24, false);
        tex.ReadPixels(new Rect(0, 0, 512, 512), 0, 0);
        tex.Apply();

        // 保存到文件，用于跨平台对比
        var bytes = tex.EncodeToPNG();
        System.IO.File.WriteAllBytes(
            $"PBR_Check_{Application.platform}_{System.DateTime.Now:HHmmss}.png", bytes);

        // 输出关键像素值用于日志对比
        Color centerPixel = tex.GetPixel(256, 256);
        Debug.Log($"Center pixel RGB: ({centerPixel.r:F4}, {centerPixel.g:F4}, {centerPixel.b:F4})");
    }
}
#endif
```

### ⚡ 实战经验

| 常见症状 | 根因 | 解决方案 |
|---------|------|---------|
| 金属变塑料 | 移动端 IBL 被关闭或降级 | 确保反射探针未被 QualitySettings 禁用；提供至少 128px CubeMap |
| 整体偏蓝 | Tonemapping=None 或 Reinhard | 统一使用 ACES Tonemapping，或自定义统一 LUT |
| 高光发散 | Roughness 贴图 ASTC 压缩精度不够 | Roughness 单独通道用 ASTC 4×4，或合并到 RGBA 但用高精度模式 |
| 暗部偏色 | SH 环境光精度不足 | 从 SH3 升级到 SH5，或用 Gradient Environment 代替 |
| 全体偏暗 | sRGB 设置错误 | 检查 BaseColor 贴图是否标记为 sRGB；Normal/Metal/Rough 是否标记为 Linear |

> **血泪经验**：某项目角色皮肤在 PC 上质感极好，上手机后面部金属高光完全消失。排查三天发现：URP Asset 的 Mobile Quality 等级自动关闭了 `_ENVIRONMENTREFLECTIONS` 关键字。**解决方案：在 URP Asset 中手动勾选 Reflection Probe，或写自定义 Shader 强制启用 IBL 采样。**

### 🎯 能力体检清单

- [ ] 能否解释 sRGB 和 Linear 贴图在采样时的区别（GPU 硬件自动做 Gamma 解码）？
- [ ] 知道为什么 Roughness 是 PBR 中最敏感的通道（它控制高光分布形状）？
- [ ] 能否在 RenderDoc 中对比 PC 和移动端的 Pixel Shader 输出，找出色彩分歧的具体步骤？
- [ ] 知道 Unity URP 的 Quality 等级如何自动修改 Shader Keyword？
- [ ] 是否做过 ACES vs Reinhard 的视觉差异评估？
- [ ] 理解为什么要用"校准球"而不是直接对比复杂材质？

### 🔗 相关问题

- [PBR 材质规范制定](../technical-art/pbr-material-authoring.md)
- [移动端贴图压缩方案](../technical-art/mobile-texture-compression.md)
- [移动端法线贴图压缩](../technical-art/mobile-normal-map-compression.md)
