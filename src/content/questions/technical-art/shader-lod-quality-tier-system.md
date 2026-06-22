---
title: "移动端 Shader LOD 规范制定：如何为 5 档机型建立质量分级体系？"
category: "technical-art"
level: 3
tags: ["Shader LOD", "质量分级", "资产规范", "移动端优化", "multi_compile", "性能预算"]
hint: "核心是「Device Tier 分级 + Shader Variant 策略 + 降级规则明确」——不是拍脑袋调参数，而是建立可量化可维护的体系"
related: ["technical-art/mobile-texture-compression", "optimization/drawcall-500-to-100", "technical-art/shader-template-system"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们的游戏要同时上高通骁龙 8 Gen3、骁龙 778G、联发科天玑 8100、骁龙 680、以及 iPhone X。现在美术抱怨"高端机效果太好低端机跑不动，低端机适配后高端机效果又变差了"。你作为 TA 要设计一套 Shader 质量分级体系，让不同机型自动获得匹配的视觉效果和性能表现。你会怎么做？

这是米哈游、叠纸、网易等跨设备手游项目的 TA 核心面试题。考察的是**资产规范制定 + 引擎变体管理 + 性能量化 + 沟通协作**的全面能力。不是"会不会写 Shader"的问题，而是"能不能建立可落地的工程体系"。

### ✅ 核心要点

1. **机型分级标准**：基于 GPU 算力 + 内存带宽 + 芯片代际划分 3-5 档，不是简单看"高端/中端/低端"
2. **Shader Variant 策略**：用 `multi_compile` 控制效果层级，避免运行时动态分支
3. **效果差异可量化**：每个 Tier 的效果差异要明确（PBR→简化PBR→Blinn-Phong→Unlit），而不是模糊的"好/差"
4. **降级规则自动化**：启动时检测机型，自动选择 Tier，不需要美术手动配置
5. **性能预算配套**：每档 Tier 对应明确的帧率、Draw Call、Shader 复杂度（ALU 指令数）预算

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：一套 Shader 系统，5 档机型各自获得最佳"性能/效果"平衡
     ↓
Step 1：定义机型分级标准
  ├── 基准：GPU 浮点算力（GFLOPS）+ 内存带宽（GB/s）+ 渲染特性支持
  ├── Tier S：骁龙8 Gen3 / A17 Pro（算力 > 2000 GFLOPS）
  ├── Tier A：骁龙8 Gen1 / A15 / 天玑9000（算力 1000~2000）
  ├── Tier B：骁龙778G / 天玑8100 / A13（算力 500~1000）
  ├── Tier C：骁龙680 / 天玑700 / A11（算力 200~500）
  └── Tier D：骁龙4系 / A9（算力 < 200）
     ↓
Step 2：定义每档的视觉特性矩阵
  Tier S: PBR + SSR + SSS + 后处理全套 + 实时阴影
  Tier A: PBR + 简化SSR + 后处理（无TAA）
  Tier B: 简化PBR（关闭环境反射）+ 基础后处理
  Tier C: Blinn-Phong + 无后处理
  Tier D: Unlit/Lambert + 最低分辨率
     ↓
Step 3：Shader Variant 实现
     ↓
Step 4：性能验证 + 资产规范文档
```

#### 知识点拆解（倒推树）

```
Shader 质量分级体系
├── 机型识别与分级
│   ├── 需要理解：移动端 GPU 架构差异
│   │   ├── Mali (Mali-G715 vs Mali-G57) → 不同架构性能差异大
│   │   ├── Adreno (Adreno 750 vs 610) → driver overhead 差异
│   │   └── Apple GPU (A17 5-core vs A11 3-core)
│   ├── 需要理解：SystemInfo GPU 信息获取
│   │   └── SystemInfo.graphicsDeviceName / graphicsMemorySize
│   ├── 需要理解：GPU 算力基准数据库
│   │   └── 维护一份 GFLOPS / Bandwidth 对照表
│   └── 需要理解：iOS 和 Android 的不同分级策略
│       └── iOS 按芯片型号（A11~A17），Android 按 GPU 型号
│
├── Shader Variant 管理
│   ├── multi_compile vs shader_feature
│   │   ├── multi_compile_QUALITY_HIGH / _MEDIUM / _LOW
│   │   ├── multi_compile：所有变体都编译（变体多但不会缺）
│   │   └── shader_feature：按需编译（省构建大小但需管理）
│   ├── 变体数量控制（Variant Stripping）
│   │   └── IPipelineAsset.stripUnusedVariants
│   └── Shader Keywords 的优先级
│       └── QUALITY_ 级别应该作为全局 Keyword
│
├── 效果降级策略
│   ├── 光照模型降级
│   │   ├── Tier S/A: PBR (Cook-Torrance BRDF)
│   │   ├── Tier B: 简化 PBR（去掉 clear coat / sheen）
│   │   ├── Tier C: Blinn-Phong + Rim Light
│   │   └── Tier D: Lambert + 简化方向光
│   ├── 贴图通道降级
│   │   ├── Tier S: Albedo + Normal + Metallic + AO + Emission + Detail
│   │   ├── Tier A: Albedo + Normal + Metallic-Smoothness(AO合并)
│   │   ├── Tier B: Albedo + Normal（Metallic 用常数）
│   │   └── Tier C/D: Albedo only
│   ├── 后处理降级
│   │   ├── Tier S: Bloom + TAA + Color Grading + DOF + SSR
│   │   ├── Tier A: Bloom + FXAA + Color Grading + DOF
│   │   ├── Tier B: Bloom + Color Grading
│   │   └── Tier C/D: 无后处理
│   └── 分辨率降级
│       ├── Tier S: 1080p / 120fps
│       ├── Tier A: 1080p / 60fps
│       ├── Tier B: 900p / 60fps 或 1080p / 30fps
│       ├── Tier C: 720p / 30fps
│       └── Tier D: 540p / 30fps
│
└── 工程化落地
    ├── 资产规范文档
    │   └── 每档 Tier 的 Shader 规范、贴图规范、LOD 规范
    ├── CI/CD 验证
    │   └── 每次 Shader 变更自动编译所有 Variant 并检查 ALU 数
    ├── 性能监控
    │   └── 实机测试报告：每档 Tier 的帧率/GPU时长/内存
    └── 美术工作流
        └── 美术在编辑器中切换 Tier 预览效果
```

#### 代码实现

**机型识别与 Tier 判定：**

```csharp
using UnityEngine;
using System.Linq;

public enum DeviceTier { S, A, B, C, D }

public static class DeviceTierDetector {
    // GPU 算力基准数据库（GFLOPS）
    // 实际项目中应维护一份完整的 JSON 配置
    private static readonly string[] TierS_GPUs = {
        "Adreno 750", "Adreno 740",        // 骁龙8 Gen3/Gen2
        "Apple GPU 15", "Apple GPU 14",     // A17/A16
    };
    private static readonly string[] TierA_GPUs = {
        "Adreno 730", "Adreno 725",        // 骁龙8 Gen1/8+
        "Mali-G715", "Mali-G710",           // 天玑9000/Exynos
        "Apple GPU 12", "Apple GPU 11",     // A15/A14
    };
    private static readonly string[] TierB_GPUs = {
        "Adreno 642", "Adreno 620",        // 骁龙778G/765G
        "Mali-G610", "Mali-G510",           // 天玑8100
        "Apple GPU 10", "Apple GPU 9",      // A13/A12
    };
    
    private static DeviceTier? _cachedTier;
    
    public static DeviceTier Detect() {
        if (_cachedTier.HasValue) return _cachedTier.Value;
        
        string gpuName = SystemInfo.graphicsDeviceName;
        int gpuMemory = SystemInfo.graphicsMemorySize; // MB
        
        // iOS：直接按设备型号判断更准
        if (Application.platform == RuntimePlatform.IPhonePlayer) {
            _cachedTier = DetectIOSTier(SystemInfo.deviceModel);
            return _cachedTier.Value;
        }
        
        // Android：按 GPU 名称 + 内存
        DeviceTier tier;
        if (TierS_GPUs.Any(g => gpuName.Contains(g))) tier = DeviceTier.S;
        else if (TierA_GPUs.Any(g => gpuName.Contains(g))) tier = DeviceTier.A;
        else if (TierB_GPUs.Any(g => gpuName.Contains(g))) tier = DeviceTier.B;
        else if (gpuMemory >= 2048) tier = DeviceTier.C;
        else tier = DeviceTier.D;
        
        // 兜底：根据内存大小微调
        if (gpuMemory < 1024 && tier <= DeviceTier.C) tier = DeviceTier.D;
        
        _cachedTier = tier;
        Debug.Log($"[DeviceTier] GPU: {gpuName}, Memory: {gpuMemory}MB → Tier {tier}");
        return tier;
    }
    
    static DeviceTier DetectIOSTier(string deviceModel) {
        // iPhone14,5 = A15, iPhone15,2 = A16, iPhone16,1 = A17
        if (deviceModel.Contains("iPhone16")) return DeviceTier.S;
        if (deviceModel.Contains("iPhone15")) return DeviceTier.A;
        if (deviceModel.Contains("iPhone14")) return DeviceTier.A;
        if (deviceModel.Contains("iPhone13")) return DeviceTier.B;
        if (deviceModel.Contains("iPhone12")) return DeviceTier.B;
        if (deviceModel.Contains("iPhone11")) return DeviceTier.C;
        if (deviceModel.Contains("iPhone10")) return DeviceTier.C;
        if (deviceModel.Contains("iPhone X")) return DeviceTier.C;
        return DeviceTier.D; // 更老的设备
    }
    
    // 获取该 Tier 对应的 Shader Keyword
    public static string GetQualityKeyword() => Detect() switch {
        DeviceTier.S => "_QUALITY_ULTRA",
        DeviceTier.A => "_QUALITY_HIGH",
        DeviceTier.B => "_QUALITY_MEDIUM",
        DeviceTier.C => "_QUALITY_LOW",
        DeviceTier.D => "_QUALITY_MINIMUM",
        _ => "_QUALITY_MEDIUM",
    };
    
    // 获取渲染分辨率缩放
    public static float GetResolutionScale() => Detect() switch {
        DeviceTier.S => 1.0f,
        DeviceTier.A => 1.0f,
        DeviceTier.B => 0.85f,
        DeviceTier.C => 0.66f,
        DeviceTier.D => 0.5f,
        _ => 0.75f,
    };
}
```

**多 Tier Shader 模板（multi_compile 方式）：**

```hlsl
// TieredCharacter.shader — 支持 5 档质量的通用角色 Shader
Shader "TA/TieredCharacter" {
    Properties {
        _BaseMap ("Albedo", 2D) = "white" {}
        _NormalMap ("Normal", 2D) = "bump" {}
        _MetallicMap ("Metallic (R) Smoothness (A)", 2D) = "white" {}
        _AOMap ("Ambient Occlusion", 2D) = "white" {}
        _EmissionMap ("Emission", 2D) = "black" {}
        _BaseColor ("Base Color", Color) = (1,1,1,1)
        _Metallic ("Metallic", Range(0,1)) = 0.0
        _Smoothness ("Smoothness", Range(0,1)) = 0.5
    }
    
    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        LOD 100
        
        // ============================================================
        //  ULTRA / HIGH — 完整 PBR + Rim + Emission（Tier S/A）
        // ============================================================
        Pass {
            Name "ForwardUltra"
            Tags { "LightMode"="UniversalForward" }
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            // === multi_compile 控制 5 档质量 ===
            #pragma multi_compile_local _QUALITY_ULTRA _QUALITY_HIGH _QUALITY_MEDIUM _QUALITY_LOW _QUALITY_MINIMUM
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            
            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float4 tangentOS  : TANGENT;
                float2 uv         : TEXCOORD0;
            };
            
            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 normalWS   : TEXCOORD1;
                float3 tangentWS  : TEXCOORD2;
                float3 bitangentWS: TEXCOORD3;
                float3 positionWS : TEXCOORD4;
            };
            
            TEXTURE2D(_BaseMap);      SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NormalMap);    SAMPLER(sampler_NormalMap);
            TEXTURE2D(_MetallicMap);  SAMPLER(sampler_MetallicMap);
            
            #if defined(_QUALITY_ULTRA) || defined(_QUALITY_HIGH)
            TEXTURE2D(_AOMap);        SAMPLER(sampler_AOMap);
            TEXTURE2D(_EmissionMap);  SAMPLER(sampler_EmissionMap);
            #endif
            
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _BaseColor;
                float  _Metallic;
                float  _Smoothness;
            CBUFFER_END
            
            Varyings vert(Attributes input) {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                output.normalWS = TransformObjectToWorldNormal(input.normalOS);
                output.tangentWS = TransformObjectToWorldDir(input.tangentOS.xyz);
                output.bitangentWS = cross(output.normalWS, output.tangentWS) * input.tangentOS.w;
                output.positionWS = TransformObjectToWorld(input.positionOS.xyz);
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target {
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;
                
                // === TSN (Tangent Space Normal) ===
                half3 normalTS = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, input.uv));
                half3x3 TBN = half3x3(input.tangentWS, input.bitangentWS, input.normalWS);
                half3 normalWS = normalize(mul(normalTS, TBN));
                
                // === 表面参数 ===
                half metallic = _Metallic;
                half smoothness = _Smoothness;
                
                #if defined(_QUALITY_ULTRA) || defined(_QUALITY_HIGH)
                half4 metallicMap = SAMPLE_TEXTURE2D(_MetallicMap, sampler_MetallicMap, input.uv);
                metallic = metallicMap.r * _Metallic;
                smoothness = metallicMap.a * _Smoothness;
                half ao = SAMPLE_TEXTURE2D(_AOMap, sampler_AOMap, input.uv).r;
                #elif defined(_QUALITY_MEDIUM)
                half4 metallicMap = SAMPLE_TEXTURE2D(_MetallicMap, sampler_MetallicMap, input.uv);
                metallic = metallicMap.r * _Metallic;
                smoothness = metallicMap.a * _Smoothness;
                half ao = 1.0; // 关闭 AO 采样
                #else
                half ao = 1.0; // 最低档不采样额外贴图
                #endif
                
                // === 光照计算（按 Tier 分级） ===
                InputData lightingInput = (InputData)0;
                lightingInput.positionWS = input.positionWS;
                lightingInput.normalWS = normalWS;
                lightingInput.viewDirectionWS = SafeNormalize(GetCameraPositionWS() - input.positionWS);
                
                SurfaceData surface = (SurfaceData)0;
                surface.albedo = albedo.rgb;
                surface.metallic = metallic;
                surface.smoothness = smoothness;
                surface.occlusion = ao;
                
                #if defined(_QUALITY_ULTRA) || defined(_QUALITY_HIGH)
                // 完整 PBR + 额外光源
                surface.emission = SAMPLE_TEXTURE2D(_EmissionMap, sampler_EmissionMap, input.uv).rgb;
                half4 color = UniversalFragmentPBR(lightingInput, surface);
                // 额外逐像素光源
                uint additionalLightsCount = GetAdditionalLightsCount();
                for (uint i = 0; i < additionalLightsCount && i < 4; i++) {
                    Light light = GetAdditionalLight(i, input.positionWS);
                    color.rgb += LightingPhysicallyBased(
                        surface, lightingInput, light.direction, light.color, light.distanceAttenuation);
                }
                return color;
                
                #elif defined(_QUALITY_MEDIUM)
                // 简化 PBR（只有主光 + 1 个额外光）
                half4 color = UniversalFragmentPBR(lightingInput, surface);
                return color;
                
                #elif defined(_QUALITY_LOW)
                // Blinn-Phong（去掉 PBR 的复杂计算）
                Light mainLight = GetMainLight();
                half NdotL = saturate(dot(normalWS, mainLight.direction));
                half3 diffuse = albedo.rgb * mainLight.color * NdotL;
                half3 viewDir = lightingInput.viewDirectionWS;
                half3 halfDir = normalize(mainLight.direction + viewDir);
                half NdotH = saturate(dot(normalWS, halfDir));
                half spec = pow(NdotH, smoothness * 128.0) * smoothness;
                half3 ambient = SampleSH(normalWS) * albedo.rgb * 0.5;
                return half4(diffuse + spec.xxx + ambient, 1.0);
                
                #else // _QUALITY_MINIMUM
                // 最简单：Lambert + 环境光
                Light mainLight = GetMainLight();
                half NdotL = saturate(dot(normalWS, mainLight.direction));
                half3 diffuse = albedo.rgb * mainLight.color * NdotL;
                half3 ambient = SampleSH(normalWS) * albedo.rgb * 0.5;
                return half4(diffuse + ambient, 1.0);
                #endif
            }
            ENDHLSL
        }
    }
    
    // 不同 Tier 的 Fallback
    SubShader {
        // Tier D 极低端的 Fallback
        Tags { "RenderType"="Opaque" }
        LOD 50
        Pass {
            Tags { "LightMode"="UniversalForward" }
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // 极简：只有颜色 + 顶点光照
            // ...（省略极简实现）
            ENDCG
        }
    }
}
```

**Tier 配置与运行时切换：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

[CreateAssetMenu(fileName = "TierConfig", menuName = "TA/Tier Configuration")]
public class TierConfig : ScriptableObject {
    [Header("Tier S - Ultra")]
    public TierSettings tierS = new TierSettings {
        renderScale = 1.0f,
        shadowDistance = 150f,
        shadowCascadeCount = 4,
        maxAdditionalLights = 8,
        enablePostProcessing = true,
        enableSSR = true,
        enableTAA = true,
        textureQuality = 1.0f,
        maxLODLevel = 0,
    };
    
    [Header("Tier A - High")]
    public TierSettings tierA = new TierSettings {
        renderScale = 1.0f,
        shadowDistance = 100f,
        shadowCascadeCount = 2,
        maxAdditionalLights = 4,
        enablePostProcessing = true,
        enableSSR = false,
        enableTAA = true,
        textureQuality = 1.0f,
        maxLODLevel = 0,
    };
    
    [Header("Tier B - Medium")]
    public TierSettings tierB = new TierSettings {
        renderScale = 0.85f,
        shadowDistance = 60f,
        shadowCascadeCount = 1,
        maxAdditionalLights = 2,
        enablePostProcessing = true,
        enableSSR = false,
        enableTAA = false,
        textureQuality = 0.75f,
        maxLODLevel = 1,
    };
    
    [Header("Tier C - Low")]
    public TierSettings tierC = new TierSettings {
        renderScale = 0.66f,
        shadowDistance = 30f,
        shadowCascadeCount = 1,
        maxAdditionalLights = 0,
        enablePostProcessing = false,
        enableSSR = false,
        enableTAA = false,
        textureQuality = 0.5f,
        maxLODLevel = 2,
    };
    
    [Header("Tier D - Minimum")]
    public TierSettings tierD = new TierSettings {
        renderScale = 0.5f,
        shadowDistance = 0f,
        shadowCascadeCount = 0,
        maxAdditionalLights = 0,
        enablePostProcessing = false,
        enableSSR = false,
        enableTAA = false,
        textureQuality = 0.25f,
        maxLODLevel = 3,
    };
    
    public TierSettings GetSettings(DeviceTier tier) => tier switch {
        DeviceTier.S => tierS,
        DeviceTier.A => tierA,
        DeviceTier.B => tierB,
        DeviceTier.C => tierC,
        _ => tierD,
    };
    
    public void ApplyToPipeline(DeviceTier tier) {
        var settings = GetSettings(tier);
        var urpAsset = GraphicsSettings.currentRenderPipeline as UniversalRenderPipelineAsset;
        
        if (urpAsset != null) {
            urpAsset.renderScale = settings.renderScale;
            urpAsset.shadowDistance = settings.shadowDistance;
            urpAsset.shadowCascadeCount = settings.shadowCascadeCount;
            urpAsset.additionalLightsCount = settings.maxAdditionalLights;
            urpAsset.supportsHDR = settings.enablePostProcessing;
            // 注：部分属性需要重新创建 Pipeline Asset
        }
        
        // 全局 Shader Keyword
        Shader.EnableKeyword(GetQualityKeywordForTier(tier));
        
        // 全局 Texture Quality
        QualitySettings.masterTextureLimit = settings.maxLODLevel;
        
        // 全局 LOD Bias
        QualitySettings.lodBias = settings.textureQuality;
        
        Debug.Log($"[TierConfig] Applied Tier {tier} settings");
    }
    
    string GetQualityKeywordForTier(DeviceTier tier) => tier switch {
        DeviceTier.S => "_QUALITY_ULTRA",
        DeviceTier.A => "_QUALITY_HIGH",
        DeviceTier.B => "_QUALITY_MEDIUM",
        DeviceTier.C => "_QUALITY_LOW",
        _ => "_QUALITY_MINIMUM",
    };
}

[System.Serializable]
public struct TierSettings {
    [Range(0.25f, 1f)] public float renderScale;
    public float shadowDistance;
    [Range(0,4)] public int shadowCascadeCount;
    [Range(0,8)] public int maxAdditionalLights;
    public bool enablePostProcessing;
    public bool enableSSR;
    public bool enableTAA;
    [Range(0.25f, 1f)] public float textureQuality;
    [Range(0,3)] public int maxLODLevel;
}
```

**5 档质量对比表：**

| 特性 | Tier S (Ultra) | Tier A (High) | Tier B (Medium) | Tier C (Low) | Tier D (Minimum) |
|------|:---:|:---:|:---:|:---:|:---:|
| 光照模型 | PBR 完整 | PBR | 简化 PBR | Blinn-Phong | Lambert |
| 主光源 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 额外光源 | 8 个 | 4 个 | 2 个 | 0 | 0 |
| 法线贴图 | ✅ | ✅ | ✅ | ✅ | ❌ |
| Metallic/Smooth | ✅ | ✅ | ✅ | 常数 | ❌ |
| AO 贴图 | ✅ | ✅ | ❌ | ❌ | ❌ |
| Emission | ✅ | ✅ | ❌ | ❌ | ❌ |
| 阴影 | 4级级联 | 2级级联 | 1级 | 简单 | ❌ |
| 后处理 | 全套 | 大部分 | 基础 | ❌ | ❌ |
| 目标帧率 | 120fps | 60fps | 60fps | 30fps | 30fps |
| 渲染分辨率 | 1080p | 1080p | 900p | 720p | 540p |
| 目标 ALU | <500 | <350 | <200 | <100 | <50 |

### ⚯ 实战经验

1. **不要用 `if-else` 做 Shader 降级**：GPU 的动态分支开销巨大。必须用 `#if defined() / #elif` 预处理宏在编译期生成不同的 Shader Variant，让编译器裁剪掉不需要的代码
2. **变体爆炸是最大风险**：5 档质量 × 其他 multi_compile（FOG, LIGHTMAP, SHADOWS 等）可能导致数千个变体。用 `shader_feature_local` 替代 `multi_compile` 可以减少变体数，配合 `IPipelineAsset.stripUnusedVariants` 在构建时裁剪
3. **Tier 检测要做缓存**：`SystemInfo.graphicsDeviceName` 在某些机型上返回的字符串不规范。维护一份 GPU 型号 → Tier 的映射表（JSON），并且做字符串模糊匹配兜底
4. **美术必须参与 Tier 设计**：不要 TA 一个人定降级规则——"Normal Map 关掉后角色还能看吗？"这类问题需要美术参与评估。在编辑器里做 Tier 预览切换工具，让美术实时看到每档效果

### 🎯 能力体检清单

- [ ] **如果不知道怎么判断机型 Tier** → 你需要补：移动 GPU 架构知识、SystemInfo API、GPU 性能基准数据库
- [ ] **如果混淆 multi_compile 和 shader_feature** → 你需要补：Shader 编译系统、变体管理、Variant Stripping
- [ ] **如果不懂 PBR 到 Blinn-Phong 的降级** → 你需要补：BRDF 模型对比、PBR 参数的物理含义、不同光照模型的视觉效果差异
- [ ] **如果不会做性能预算** → 你需要补：GPU Profiler 使用、ALU/Memory Bandwidth/Texture Fetch 三类开销的量化方法
- [ ] **如果不会写资产规范** → 你需要补：TA 文档撰写能力、与美术/程序的协作流程、规范的可执行性设计

### 🔗 相关问题

- 如何在 CI/CD 中自动检测 Shader 变体数量超标？（Variant Stripping + Build Report）
- 不同 GPU 架构（Mali vs Adreno vs Apple）对同一 Shader 的性能差异有多大？
- 如何做跨 Tier 的 QA 验证？（真机自动化测试 + 性能回溯报告）
