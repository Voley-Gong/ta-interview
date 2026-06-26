---
title: "角色冰冻特效：如何在 Shader 中实现冰晶蔓延 + 表面结晶 + 体积光折射的冰冻效果？"
category: "shader"
level: 3
tags: ["冰冻效果", "顶点位移", "折射", "Fresnel", "噪声纹理", "URP", "角色特效"]
hint: "三层叠加——冰冻遮罩控制蔓延范围 + 冰晶纹理叠加表面质感 + 折射扰动模拟冰的透光感"
related: ["shader/dissolve-effect", "shader/freeze-crystal-effect", "shader/hit-flash-damage-blink"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的 RPG 角色被冰系技能命中后，需要一个从脚到蔓延全身的冰冻效果——皮肤表面逐渐覆盖冰晶纹理，冰冻区域有轻微的体积膨胀（顶点外扩），表面有折射光泽。URP Shader 实现，给我方案。」

### ✅ 核心要点

1. **冰冻遮罩（Frost Mask）**：用一张噪声纹理 + 阈值参数控制冰冻从脚到上的蔓延方向
2. **冰晶表面纹理**：在冰冻区域叠加 Ice Normal Map + Detail Noise，产生结晶质感
3. **顶点位移膨胀**：冰冻区域顶点沿法线方向轻微外扩，模拟冰的体积感
4. **折射光泽**：冰冻区域用 Grab Pass / Color RT 做屏幕折射扰动，模拟冰的透光性
5. **边缘 Fresnel 冰光**：冰冻与非冰冻的边界处叠加 Fresnel 高光，做出冰霜蔓延的光感
6. **可控参数化**：_FrostAmount（0→1）驱动所有效果层级联动

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色从脚开始 → 逐渐覆盖冰晶 → 冰冻区域轻微膨胀 → 表面有折射光泽 → 边缘有冰霜光
                ↑
倒推1：需要控制「从脚到上蔓延」→ 用世界坐标 Y 轴高度 + 噪声扰动 做 Frost Mask
倒推2：需要「冰晶质感」→ 在 Frost 区域叠加 Ice Normal Map + Detail Noise
倒推3：需要「体积膨胀」→ 顶点着色器中，Frost 区域沿法线位移
倒推4：需要「折射光泽」→ 屏幕颜色采样 + UV 扰动（模拟冰面折射）
倒推5：需要「冰霜边缘光」→ Frost Mask 边缘（smoothstep 过渡带）叠加 Fresnel 冰蓝色光
倒推6：需要「可控触发」→ C# 脚本插值 _FrostAmount（0=正常，1=完全冰冻）
```

#### 知识点拆解（倒推树）

```
角色冰冻特效
├── 冰冻遮罩系统（Frost Mask）
│   ├── 基于世界 Y 高度的渐变（footY → headY 映射到 0→1）
│   ├── 噪声扰动使边缘不规则（Voronoi/Crackle Noise 模拟冰晶扩散路径）
│   ├── _FrostAmount 参数（0=无冰冻，1=完全冰冻）
│   └── smoothstep 控制边缘过渡宽度
├── 冰晶表面纹理（Ice Surface）
│   ├── Ice Normal Map（法线凹凸——冰面起伏感）
│   ├── Detail Noise Map（细节噪声——细小冰晶颗粒）
│   ├── Normal 混合（RNM/UDN 混合基础法线和冰面法线）
│   └── 冰面 Roughness/Metallic 覆盖（粗糙度降低，金属度=0）
├── 顶点位移膨胀（Vertex Displacement）
│   ├── 沿法线方向外扩（frostArea * _ExpandAmount * normalDir）
│   ├── 噪声驱动不均匀膨胀（让冰刺看起来随机生长）
│   └── 位移幅度极小（0.01-0.05 单位），避免穿模
├── 折射效果（Refraction）
│   ├── 方案A：URP 中用 Custom Screen Texture（_CameraOpaqueTexture）
│   ├── 方案B：Scene Color 节点（Shader Graph）
│   ├── UV 扰动：用 Ice Normal 的 RG 通道 * _RefractStrength 偏移采样
│   └── 色彩偏移：冰冻区域叠加淡蓝色 tint + 亮度微提升
├── 边缘冰光（Rim Frost）
│   ├── Fresnel 效应：pow(1 - dot(N, V), power) 计算边缘强度
│   ├── 仅在 Frost Mask 边缘带生效（smoothstep 过渡区间）
│   └── 冰蓝色 Emission 叠加
├── URP 实现方案
│   ├── Shader Graph vs 手写 HLSL（推荐 Shader Graph 快速出效果 + HLSL 优化）
│   ├── 需要开启 Opaque Texture（用于折射）
│   └── 透明混合 vs 不透明（推荐不透明 + 折射，避免透明排序问题）
└── C# 驱动
    ├── MaterialPropertyBlock 控制 _FrostAmount（不要 new Material）
    ├── DOTween / 协程做渐变动画（1秒内从0→0.8，留0.2给手动解除）
    └── 多角色材质统一参数更新（遍历 Renderer[]）
```

#### 代码实现

**手写 HLSL（URP 兼容，核心逻辑）：**

```hlsl
Shader "Custom/CharacterFrost"
{
    Properties
    {
        [Header(Base)]
        _BaseMap ("Base Map", 2D) = "white" {}
        _NormalMap ("Normal Map", 2D) = "bump" {}

        [Header(Frost)]
        _FrostAmount ("Frost Amount", Range(0, 1)) = 0
        _FrostNoise ("Frost Noise", 2D) = "white" {}
        _FrostSpread ("Frost Spread", Range(0.1, 1.0)) = 0.3
        _FootY ("Foot World Y", Float) = 0.0
        _HeadY ("Head World Y", Float) = 2.0

        [Header(Ice Surface)]
        _IceNormal ("Ice Normal Map", 2D) = "bump" {}
        _IceDetail ("Ice Detail Noise", 2D) = "white" {}
        _IceColor ("Ice Tint Color", Color) = (0.7, 0.85, 1.0, 1.0)
        _IceSmoothness ("Ice Smoothness", Range(0, 1)) = 0.9

        [Header(Vertex Expand)]
        _ExpandAmount ("Vertex Expand", Range(0, 0.1)) = 0.03

        [Header(Refraction)]
        _RefractStrength ("Refraction Strength", Range(0, 0.05)) = 0.015

        [Header(Rim Frost)]
        _RimColor ("Rim Frost Color", Color) = (0.6, 0.9, 1.0, 1.0)
        _RimPower ("Rim Power", Range(1, 10)) = 4.0
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode"="UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _IceNormal_ST;
                float4 _IceDetail_ST;
                float4 _FrostNoise_ST;
                float4 _IceColor;
                float4 _RimColor;
                float _FrostAmount;
                float _FrostSpread;
                float _FootY;
                float _HeadY;
                float _IceSmoothness;
                float _ExpandAmount;
                float _RefractStrength;
                float _RimPower;
            CBUFFER_END

            TEXTURE2D(_BaseMap);          SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NormalMap);         SAMPLER(sampler_NormalMap);
            TEXTURE2D(_FrostNoise);        SAMPLER(sampler_FrostNoise);
            TEXTURE2D(_IceNormal);         SAMPLER(sampler_IceNormal);
            TEXTURE2D(_IceDetail);         SAMPLER(sampler_IceDetail);
            TEXTURE2D(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);

            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float4 tangentOS  : TANGENT;
                float2 uv         : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv          : TEXCOORD0;
                float3 normalWS    : TEXCOORD1;
                float3 tangentWS   : TEXCOORD2;
                float3 bitangentWS : TEXCOORD3;
                float3 positionWS  : TEXCOORD4;
                float4 screenPos   : TEXCOORD5;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            // 计算 Frost Mask（0=未冰冻, 1=完全冰冻）
            float ComputeFrostMask(float3 positionWS, float2 uv, float frostAmount)
            {
                // 基于世界 Y 高度的从脚到头渐变
                float heightFactor = saturate((positionWS.y - _FootY) / max(0.001, _HeadY - _FootY));
                // frostAmount 控制冰冻顶部高度：frostAmount=0 → 全部未冰冻，=1 → 全部冰冻
                float heightGradient = saturate((1.0 - heightFactor) * (1.0 / max(0.01, frostAmount)));
                // 用噪声扰动边缘，模拟冰晶不规则扩散
                float noise = SAMPLE_TEXTURE2D_LOD(_FrostNoise, sampler_FrostNoise, uv * 3.0, 0).r;
                heightGradient = heightGradient + (noise - 0.5) * _FrostSpread;
                return saturate(heightGradient);
            }

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);

                // 先计算 WS position（位移前）
                float3 positionWS = TransformObjectToWorld(IN.positionOS.xyz);
                float3 normalWS = TransformObjectToWorldNormal(IN.normalOS);

                // 预计算 Frost Mask（使用 UV 采样噪声，世界 Y 做方向）
                float frostMask = ComputeFrostMask(positionWS, IN.uv, _FrostAmount);

                // 顶点沿法线膨胀（仅冰冻区域）
                float3 displacedWS = positionWS + normalWS * frostMask * _ExpandAmount;

                OUT.positionWS = displacedWS;
                OUT.positionHCS = TransformWorldToHClip(displacedWS);
                OUT.normalWS = normalWS;
                OUT.tangentWS = TransformObjectToWorldDir(IN.tangentOS.xyz);
                OUT.bitangentWS = cross(normalWS, OUT.tangentWS) * IN.tangentOS.w;
                OUT.uv = IN.uv;
                OUT.screenPos = ComputeScreenPos(OUT.positionHCS);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                // --- 基础纹理 ---
                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
                float3 baseNormalTS = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, IN.uv));

                // --- Frost Mask ---
                float frostMask = ComputeFrostMask(IN.positionWS, IN.uv, _FrostAmount);

                // --- 冰面法线（混合） ---
                float3 iceNormalTS = UnpackNormal(SAMPLE_TEXTURE2D(_IceNormal, sampler_IceNormal, IN.uv * 2.0));
                float3 iceDetailTS = UnpackNormal(SAMPLE_TEXTURE2D(_IceDetail, sampler_IceDetail, IN.uv * 5.0));
                // 细节法线混合
                iceNormalTS = normalize(float3(iceNormalTS.xy + iceDetailTS.xy * 0.5, iceNormalTS.z));
                // 基础法线和冰面法线混合（按 frostMask）
                float3 blendedNormalTS = normalize(lerp(baseNormalTS, iceNormalTS, frostMask));

                // TBN 矩阵 → 世界空间法线
                float3 normalWS = normalize(mul(blendedNormalTS, float3x3(
                    IN.tangentWS, IN.bitangentWS, IN.normalWS)));

                // --- URP 光照（简化版，只取主光） ---
                Light mainLight = GetMainLight();
                float3 lightDir = normalize(mainLight.direction);
                float NdotL = max(0, dot(normalWS, lightDir));
                float3 diffuse = baseColor.rgb * mainLight.color * NdotL;

                // 冰冻区域叠加冰色 tint + 高光
                float3 iceDiffuse = lerp(diffuse, diffuse * _IceColor.rgb + float3(0.1, 0.15, 0.2), frostMask);
                float specPower = lerp(1.0, _IceSmoothness, frostMask);
                float3 halfDir = normalize(lightDir + GetWorldSpaceNormalizeViewDir(IN.positionWS));
                float specIntensity = pow(max(0, dot(normalWS, halfDir)), specPower * 128) * frostMask;
                iceDiffuse += float3(0.8, 0.9, 1.0) * specIntensity;

                // --- 折射效果（用 Opaque Texture） ---
                float2 screenUV = IN.screenPos.xy / IN.screenPos.w;
                float2 refractOffset = iceNormalTS.rg * _RefractStrength * frostMask;
                float3 sceneColor = SAMPLE_TEXTURE2D(_CameraOpaqueTexture, sampler_CameraOpaqueTexture,
                                                      screenUV + refractOffset).rgb;
                // 冰冻区域用折射色替换
                float3 finalColor = lerp(iceDiffuse, sceneColor * _IceColor.rgb + float3(0.05, 0.1, 0.15), frostMask * 0.5);

                // --- Rim Frost 冰霜边缘光 ---
                float3 viewDir = GetWorldSpaceNormalizeViewDir(IN.positionWS);
                float rim = pow(1.0 - max(0, dot(normalWS, viewDir)), _RimPower);
                // 仅在 frost mask 边缘带增强
                float edgeBand = smoothstep(0.7, 1.0, frostMask) * smoothstep(1.0, 0.85, frostMask) * 4.0;
                finalColor += _RimColor.rgb * rim * edgeBand;

                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**C# 冰冻控制器：**

```csharp
using System.Collections;
using UnityEngine;

public class FrostEffectController : MonoBehaviour
{
    [SerializeField] private Renderer[] targetRenderers;
    [SerializeField] private float frostDuration = 0.8f;
    [SerializeField] private float maxFrost = 0.85f;

    private MaterialPropertyBlock _mpb;
    private static readonly int FrostAmountID = Shader.PropertyToID("_FrostAmount");
    private static readonly int FootYID = Shader.PropertyToID("_FootY");
    private static readonly int HeadYID = Shader.PropertyToID("_HeadY");

    void Awake()
    {
        _mpb = new MaterialPropertyBlock();
        UpdateFootHeadY();
    }

    void UpdateFootHeadY()
    {
        var bounds = new Bounds();
        foreach (var r in targetRenderers)
            bounds.Encapsulate(r.bounds);

        foreach (var r in targetRenderers)
        {
            r.GetPropertyBlock(_mpb);
            _mpb.SetFloat(FootYID, bounds.min.y);
            _mpb.SetFloat(HeadYID, bounds.max.y);
            r.SetPropertyBlock(_mpb);
        }
    }

    public void ApplyFrost()
    {
        StartCoroutine(FrostRoutine(maxFrost));
    }

    public void RemoveFrost()
    {
        StartCoroutine(FrostRoutine(0f));
    }

    IEnumerator FrostRoutine(float target)
    {
        float startValue = 0f;
        // 读取当前值（简化版，可扩展为实际读取）
        float elapsed = 0f;
        while (elapsed < frostDuration)
        {
            float t = elapsed / frostDuration;
            float value = Mathf.Lerp(startValue, target, t);

            foreach (var r in targetRenderers)
            {
                r.GetPropertyBlock(_mpb);
                _mpb.SetFloat(FrostAmountID, value);
                r.SetPropertyBlock(_mpb);
            }

            elapsed += Time.deltaTime;
            yield return null;
        }
    }
}
```

**冰冻效果层级表：**

| 效果层 | 技术手段 | 视觉表现 | 性能开销 |
|--------|----------|----------|----------|
| 蔓延遮罩 | 世界Y高度 + 噪声阈值 | 从脚到头的不规则扩散 | 极低（1次噪声采样） |
| 冰晶表面 | Ice Normal + Detail Noise 混合 | 冰晶凹凸质感 | 低（2次法线采样） |
| 顶点膨胀 | 法线方向位移 | 冰的体积感 | 低（顶点位移） |
| 折射光泽 | Opaque Texture UV 扰动 | 冰的透光性 | 中（1次屏幕采样） |
| 边缘冰光 | Fresnel + 边缘带检测 | 冰霜蔓延的光晕 | 极低 |

### ⚡ 实战经验

- **从脚到头的方向用世界 Y 还是物体本地 Y？** 如果角色会倒地（比如冰冻时倒下），用骨骼根节点的本地 Y。世界 Y 在角色倒地后会变成从头到脚蔓延，效果不对
- **折射不要太强**：_RefractStrength 超过 0.03 就会看起来像玻璃而不是冰。冰的折射应该是「微微扭曲」而非「明显变形」
- **噪声纹理选择**：Voronoi/Cellular Noise 效果最好——冰晶天然是六边形/蜂窝结构。Perlin Noise 太圆滑，没有冰的锐利感
- **多角色材质复用**：用 MaterialPropertyBlock 控制每角色的 _FrostAmount，不要实例化 Material。_FootY/_HeadY 也需要 per-instance 设置
- **与粒子配合**：冰冻完成瞬间，在角色关节处生成冰碎粒子爆发——Shader 负责「冻」，粒子负责「碎」
- **解冻效果**：反向插值 _FrostAmount 即可，但解冻时可以加一层水滴流动画（用流动噪声 + UV 滚动）

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么控制蔓延方向 | 世界坐标 vs 物体坐标 | 学 Object Space / World Space / Tangent Space 变换 |
| 冰面看起来像塑料不像冰 | 冰的 PBR 属性 | 学冰的 IOR（折射率1.31）、Roughness 范围 |
| 法线混合后表面抖动 | 法线混合算法 | 学 RNM（Reoriented Normal Mapping）或 Whiteout 混合 |
| 顶点位移穿模 | 位移幅度控制 | 学位移安全范围 + 邻接顶点平滑 |
| 折射效果在移动端不工作 | Opaque Texture 支持 | 学 URP Opaque Texture 设置 + 移动端兼容性 |
| 多角色同时冰冻性能差 | 材质实例管理 | 学 MaterialPropertyBlock + GPU Instancing |

### 🔗 相关问题

- 如何实现「局部冰冻」（只有手臂被冰冻）？（提示：用顶点色的 R 通道标记冰冻区域）
- 冰冻效果如何与现有的角色换装系统共存？（提示：材质模板系统 + Frost Override Shader）
- 如何做「冰冻破碎」效果？（提示：冰冻遮罩达到阈值后触发破碎粒子 + 模型碎块）
- 在 Shader Graph 中如何搭建这套冰冻系统？和手写 HLSL 的性能差异？
