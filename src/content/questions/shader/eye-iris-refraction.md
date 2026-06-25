---
title: "角色眼睛通透感：如何用 Shader 实现虹膜折射与瞳孔缩放？"
category: "shader"
level: 3
tags: ["眼睛渲染", "折射", "虹膜", "瞳孔", "NPR", "角色Shader"]
hint: "眼睛通透感的核心是模拟光线进入角膜后的折射路径——不是贴一张贴图就完事，而是角膜→虹膜→瞳孔的三层结构"
related: ["shader/hair-anisotropic-lighting", "rendering/sss-skin-rendering", "technical-art/face-shadow-shimmering"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的二次元角色眼睛看起来很'死'，像贴了一张纸片。想要实现类似《原神》/《崩坏：星穹铁道》那种有通透感、有深度、瞳孔还能随光照缩放的眼睛渲染。URP 下用 Shader 实现，说说你的方案。」

### ✅ 核心要点

1. **三层结构建模**：角膜（Cornea）→ 虹膜（Iris）→ 瞳孔（Pupil），不是单层贴图
2. **折射模拟**：光线进入角膜发生折射，偏移后采样虹膜贴图，产生「看进眼睛里」的深度感
3. **视差修正**：根据视角动态调整采样偏移，侧面看时偏移大、正面看时偏移小
4. **瞳孔缩放**：用径向 UV 缩放模拟虹膜收缩/放大，可由光照强度或情绪参数驱动
5. **高光与反射**：角膜表面的镜面高光（基于 BDRF）+ 环境反射（cubemap/matcap）
6. **边缘暗化**：虹膜外缘的 limbal ring，增加眼球的体积感

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：眼睛有「通透感」+ 深度感 + 高光 + 瞳孔动态缩放
                ↑
倒推1：「通透感」来自 → 角膜透明 + 虹膜清晰可见 + 折射偏移
倒推2：「深度感」来自 → 视差/折射让虹膜看起来在角膜「下面」
倒推3：「高光」来自 → 角膜表面的 Specular BRDF + 环境反射
倒推4：「瞳孔缩放」来自 → 径向 UV scale，lerp 两个状态
倒推5：「边缘暗化」(limbal ring) 来自 → 虹膜边缘的径向遮罩
```

#### 知识点拆解（倒推树）

```
角色眼睛渲染
├── 眼球结构理解
│   ├── 角膜（透明外壳，光线折射发生处）
│   ├── 虹膜（彩色部分，决定眼睛颜色）
│   ├── 瞳孔（黑色孔洞，控制进光量）
│   └── 巩膜（眼白部分）
│
├── 折射模拟（核心难点）
│   ├── 方案 A：Parallax Mapping（视差偏移）
│   │   └── 简单但无真实折射，适合移动端
│   ├── 方案 B：基于 Snell 定律的近似折射
│   │   └── mid-tier 方案，用 refract() 函数
│   ├── 方案 C：预烘焙环境贴图查找
│   │   └── 高端方案，但需要额外 RT
│   └── 方案 D：二次元简化版——UV 偏移 + mask
│       └── 主流手游方案，性能友好
│
├── 瞳孔动态系统
│   ├── 瞳孔 UV 中心定位（模型 UV 规范要求）
│   ├── 径向 scale：lerp(pupilScaleMin, pupilScaleMax, lightFactor)
│   └── 平滑过渡：smoothstep + 时间插值
│
├── 视觉增强
│   ├── Limbal Ring：虹膜外缘暗化（径向 gradient）
│   ├── 角膜高光：Blinn-Phong / GGX + matcap 补光
│   ├── 次表面散射近似：眼白部分轻微 SSS 假色
│   └── 阴影接收：上眼睑投射阴影到眼球
│
└── 美术规范
    ├── UV 布局：虹膜居中在 UV 空间的固定区域
    ├── 贴图清单：Iris Color Map、Pupil Mask、AO Map、Normal Map
    └── 模型要求：眼球与眼睑的间隙控制（避免穿模）
```

#### 代码实现

**二次元风格眼睛 Shader（URP HLSL）：**

```hlsl
Shader "Character/EyeAnime"
{
    Properties
    {
        [Header(Iris)]
        _IrisMap ("Iris Map (RGB: Color, A: Pupil Mask)", 2D) = "white" {}
        _IrisColor ("Iris Tint", Color) = (1,1,1,1)
        _IrisRadius ("Iris Radius", Range(0.1, 0.5)) = 0.3
        
        [Header(Pupil)]
        _PupilScale ("Pupil Scale", Range(0.3, 2.0)) = 1.0
        _PupilDarkness ("Pupil Darkness", Range(0, 1)) = 0.95
        
        [Header(Refraction)]
        _RefractionStrength ("Refraction Strength", Range(0, 0.05)) = 0.02
        
        [Header(Specular)]
        _SpecularColor ("Specular Color", Color) = (1,1,1,1)
        _SpecularPower ("Specular Power", Range(1, 200)) = 60
        
        [Header(Limbal Ring)]
        _LimbalRingColor ("Limbal Ring Color", Color) = (0.2, 0.15, 0.1, 1)
        _LimbalRingWidth ("Limbal Ring Width", Range(0.01, 0.15)) = 0.05
        
        [Header(Sclera)]
        _ScleraColor ("Sclera (White) Color", Color) = (0.9, 0.88, 0.85, 1)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" "Queue"="Geometry" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _IrisMap_ST;
                float4 _IrisColor;
                float4 _SpecularColor;
                float4 _LimbalRingColor;
                float4 _ScleraColor;
                float _IrisRadius;
                float _PupilScale;
                float _PupilDarkness;
                float _RefractionStrength;
                float _SpecularPower;
                float _LimbalRingWidth;
            CBUFFER_END

            TEXTURE2D(_IrisMap); SAMPLER(sampler_IrisMap);

            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float3 normalWS    : NORMAL;
                float3 viewDirWS   : TEXCOORD1;
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
                OUT.viewDirWS = GetWorldSpaceNormalizeViewDir(TransformObjectToWorld(IN.positionOS.xyz));
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                float2 uv = IN.uv;
                
                // === 1. 确定虹膜/瞳孔区域（假设虹膜中心在 UV (0.5, 0.5)）===
                float2 irisCenter = float2(0.5, 0.5);
                float2 toCenter = uv - irisCenter;
                float distFromCenter = length(toCenter);
                
                // 虹膜遮罩
                half irisMask = smoothstep(_IrisRadius, _IrisRadius - 0.02, distFromCenter);
                // 巩膜遮罩（眼白）
                half scleraMask = 1.0 - irisMask;
                
                // === 2. 折射偏移（视角依赖的 UV 偏移）===
                float3 normalWS = normalize(IN.normalWS);
                float3 viewDirWS = normalize(IN.viewDirWS);
                float NdotV = saturate(dot(normalWS, viewDirWS));
                
                // 视角越斜，折射偏移越大（模拟 Snell 定律的简化版）
                float2 refractionOffset = normalize(toCenter) * _RefractionStrength * (1.0 - NdotV);
                float2 irisUV = uv + refractionOffset;
                
                // === 3. 瞳孔缩放（径向 UV scale）===
                float2 pupilUV = (irisUV - irisCenter) / _PupilScale + irisCenter;
                // 限制瞳孔 UV 不超出虹膜范围
                pupilUV = clamp(pupilUV, irisCenter - _IrisRadius, irisCenter + _IrisRadius);
                
                // === 4. 采样虹膜贴图 ===
                half4 irisSample = SAMPLE_TEXTURE2D(_IrisMap, sampler_IrisMap, pupilUV);
                half3 irisColor = irisSample.rgb * _IrisColor.rgb;
                
                // 瞳孔区域压暗
                half pupilMask = irisSample.a; // Alpha 通道作为瞳孔遮罩
                irisColor = lerp(irisColor, irisColor * (1.0 - _PupilDarkness), pupilMask);
                
                // === 5. Limbal Ring（虹膜外缘暗化）===
                float limbalFactor = smoothstep(_IrisRadius - _LimbalRingWidth, _IrisRadius, distFromCenter);
                irisColor = lerp(irisColor, _LimbalRingColor.rgb, limbalFactor * irisMask);
                
                // === 6. 巩膜颜色 ===
                half3 scleraColor = _ScleraColor.rgb;
                // 巩膜轻微的阴影（上眼睑 AO 近似）
                // 可通过顶点色的 AO 通道或预烘焙 AO 贴图实现
                
                // === 7. 合并虹膜和巩膜 ===
                half3 finalColor = lerp(scleraColor, irisColor, irisMask);
                
                // === 8. 角膜高光 ===
                Light mainLight = GetMainLight();
                float3 halfVec = normalize(mainLight.direction + viewDirWS);
                float NdotH = saturate(dot(normalWS, halfVec));
                float specular = pow(NdotH, _SpecularPower);
                finalColor += _SpecularColor.rgb * specular;
                
                // === 9. 基础环境光 ===
                half3 ambient = SampleSH(normalWS) * 0.5;
                finalColor += ambient * irisMask * 0.3;
                
                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**C# 瞳孔缩放驱动（光照响应版）：**

```csharp
using UnityEngine;

public class EyePupilController : MonoBehaviour
{
    [Header("References")]
    [SerializeField] private Renderer eyeRenderer;
    
    [Header("Pupil Settings")]
    [SerializeField] private float minPupilScale = 0.5f;   // 强光下瞳孔缩小
    [SerializeField] private float maxPupilScale = 1.8f;   // 暗光下瞳孔放大
    [SerializeField] private float responseSpeed = 5f;
    
    [Header("Light Settings")]
    [SerializeField] private Light sceneLight;
    [SerializeField] private float lightThreshold = 1.5f;
    
    private MaterialPropertyBlock _mpb;
    private static readonly int PupilScaleID = Shader.PropertyToID("_PupilScale");
    private float _currentScale = 1f;

    void Awake() => _mpb = new MaterialPropertyBlock();

    void Update()
    {
        // 根据光照强度计算目标瞳孔大小
        float lightIntensity = sceneLight != null ? sceneLight.intensity : 1f;
        float targetScale = lightIntensity > lightThreshold ? minPupilScale : maxPupilScale;
        
        // 情绪参数可以叠加（惊讶→放大，愤怒→缩小）
        // targetScale *= emotionFactor;
        
        _currentScale = Mathf.Lerp(_currentScale, targetScale, Time.deltaTime * responseSpeed);
        
        eyeRenderer.GetPropertyBlock(_mpb);
        _mpb.SetFloat(PupilScaleID, _currentScale);
        eyeRenderer.SetPropertyBlock(_mpb);
    }
}
```

**方案对比表：眼睛折射实现**

| 方案 | 视觉效果 | 性能开销 | 适用平台 | 复杂度 |
|------|----------|----------|----------|--------|
| 单层贴图（无折射） | ★★ 纸片感 | 极低 | 全平台 | 低 |
| UV 偏移模拟折射 | ★★★ 有深度感 | 低 | 手游 | 低 |
| Parallax Mapping | ★★★★ 较好 | 中 | 中高端 | 中 |
| Snell 折射近似 | ★★★★★ 写实 | 中高 | 主机/PC | 高 |
| Pre-integrated 查找表 | ★★★★★ 写实 | 中 | 主机/PC | 高（预处理） |

### ⚡ 实战经验

1. **UV 布局是前提**：虹膜必须在 UV 空间居中且为圆形，否则折射偏移会偏。和建模师提前约定 UV 规范
2. **二次元不做物理折射**：手游角色眼睛不需要正确的 Snell 定律，UV 偏移 + 阈值控制就够了，关键是美术效果
3. **Limbal Ring 是灵魂**：虹膜外缘那一圈暗色，是让眼睛看起来「真实」而不是「贴纸」的关键。很多新手忽略这个
4. **高光位置很重要**：高光应该在角膜表面而非虹膜上。如果高光跟着虹膜走，看起来会很诡异
5. **眼睑阴影**：上眼睑在眼球上投射的阴影，用一个 AO 贴图或顶点色就能解决，但效果提升巨大

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 眼睛看起来像贴纸 | 不理解折射/视差原理 | 学 Parallax Mapping 和折射近似 |
| 不知道怎么让瞳孔缩放 | UV 变换不熟 | 练习径向 UV scale 和 mask 混合 |
| 角膜高光位置不对 | 不理解高光在角膜还是虹膜 | 复习分层渲染思路 |
| 边缘没有体积感 | 忽略了 Limbal Ring | 学习眼球解剖结构和观察真实眼球照片 |
| 多角色眼睛不可复用 | UV 规范未统一 | 制定眼球 UV 布局标准 |

### 🔗 相关问题

- 头发各向异性高光和眼睛高光如何在同一角色脸上协调？
- 如何在 Shader Graph 中搭建眼睛折射节点网络？
- 写实风格眼睛需要 Ray Marching 吗？性能如何？
- 角色闭眼/眨眼动画如何与眼睛 Shader 联动？
