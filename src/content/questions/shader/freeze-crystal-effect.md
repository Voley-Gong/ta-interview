---
title: "角色冰冻特效：如何用 Shader 实现冰晶蔓延 + 透射折射？"
category: "shader"
level: 3
tags: ["Shader", "冰冻效果", "顶点变形", "折射", "噪声", "战斗特效"]
hint: "冰冻的核心是「动态遮罩控制蔓延」+「冰晶法线扰动模拟折射」+「顶点外扩模拟冰刺」，三层效果叠加才有质感"
related: ["shader/dissolve-effect", "rendering/urp-renderer-feature", "shader/npr-outline-cartoon"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们做一个 ARPG 手游，角色被冰系技能击中时需要有"冰冻"效果——从脚到头逐渐覆盖冰晶，冰层有折射感，表面有冰刺凸起，破碎时有碎冰飞溅。性能要求移动端 60fps 同屏 10 个角色。你会怎么实现？

这是腾讯（王者荣耀）、网易（永劫无间手游）、米哈游等动作游戏项目的高频 Shader 面试题。考察的是**多层效果组合能力**——冰冻不是单一 Shader，而是顶点变形 + 表面纹理 + 折射 + 粒子的系统工程。

### ✅ 核心要点

1. **动态遮罩驱动蔓延**：用噪声纹理 + 时间参数控制冰冻从脚到头的扩散方向
2. **冰晶表面模拟**：Voronoi 噪声生成冰晶纹理 + 法线扰动模拟冰面凹凸
3. **折射假效果**：抓屏 Pass + 法线扰动 UV 采样背景，模拟冰层的透射折射
4. **顶点外扩冰刺**：沿法线方向根据遮罩强度外扩顶点，形成冰刺轮廓
5. **破碎联动**：遮罩消失时触发碎冰粒子，Shader 中用 Dissolve 算法控制消散

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
期望效果：冰冻蔓延 → 冰晶表面 → 折射感 → 冰刺轮廓 → 破碎消散
         ↓
效果分层拆解：
  Layer 1：蔓延遮罩（控制冰冻扩散方向与进度）
     ↓
  Layer 2：冰晶表面纹理（Voronoi 网格 + 法线扰动）
     ↓
  Layer 3：冰层折射（屏幕抓色 + UV 扰动）
     ↓
  Layer 4：顶点变形（沿法线外扩模拟冰刺）
     ↓
  Layer 5：消散破碎（Dissolve + 粒子联动）
```

#### 知识点拆解（倒推树）

```
冰冻特效
├── 蔓延遮罩系统
│   ├── 需要理解：如何控制效果在模型表面"生长"
│   │   ├── 方案A：基于 Y 轴高度的线性蔓延（简单但死板）
│   │   ├── 方案B：噪声纹理 + 阈值控制（有机但不定向）
│   │   └── 方案C：Y轴 + 噪声扰动（推荐：有方向感且边界自然）
│   ├── 核心参数：_FreezeProgress (0→1)
│   └── 边缘过渡：smoothstep + 噪声扰动制造冰尖形状
│
├── 冰晶表面纹理
│   ├── 需要理解：Voronoi 噪声的数学原理
│   │   └── F1 Voronoi：最近点距离 → 形成晶格状纹理
│   ├── 需要理解：法线扰动（Normal Perturbation）
│   │   └── 用 Voronoi 的梯度扰动表面法线 → 凹凸感
│   └── 冰的物理特性：高反射 + 低粗糙度 + 偏蓝色调
│
├── 折射效果（假折射）
│   ├── 需要理解：真正的折射需要 GrabPass / Opaque Texture
│   ├── URP 方案：_CameraOpaqueTexture + 法线扰动 UV
│   ├── 扰动强度 = 法线强度 × 冰层厚度遮罩
│   └── 性能注意：移动端 GrabPass 开销大，可以用降屏 + 模糊替代
│
├── 顶点外扩（冰刺）
│   ├── 需要理解：顶点着色器中沿法线膨胀
│   │   └── pos += normal * _IceSpikeAmount * freezeMask
│   ├── 需要理解：噪声控制外扩不均匀
│   │   └── 用 Simplex Noise 让某些区域凸出更多
│   └── 性能注意：顶点变形不能太剧烈，否则阴影会穿模
│
└── 破碎消散
    ├── Dissolve 算法：clip(noise - threshold)
    ├── 碎冰粒子：遮罩边缘触发 GPU 粒子
    └── 时间线控制：冰冻 → 维持 → 裂纹 → 碎裂（状态机）
```

#### 代码实现

**冰冻 Shader（URP 兼容，核心 Pass）：**

```hlsl
// FreezeEffect.shader — URP 兼容
Shader "TA/FreezeEffect" {
    Properties {
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1,1,1,1)
        
        // 冰冻参数
        _FreezeProgress ("Freeze Progress", Range(0,1)) = 0
        _FreezeColor ("Freeze Color", Color) = (0.7, 0.85, 1.0, 1.0)
        _FreezeHeight ("Freeze Start Height", Float) = 0
        
        // 冰晶纹理
        _IceNoiseTex ("Ice Noise Texture", 2D) = "white" {}
        _IceNoiseScale ("Ice Noise Scale", Float) = 5.0
        _CrystalIntensity ("Crystal Intensity", Range(0,1)) = 0.8
        
        // 折射
        _RefractionStrength ("Refraction Strength", Range(0,0.1)) = 0.02
        _CameraOpaqueTexture ("Camera Opaque", 2D) = "" {}
        
        // 冰刺
        _IceSpikeAmount ("Ice Spike Amount", Range(0,0.1)) = 0.03
        
        // 消散
        _DissolveProgress ("Dissolve Progress", Range(0,1)) = 0
        _DissolveEdgeWidth ("Dissolve Edge Width", Range(0,0.5)) = 0.05
        _DissolveEdgeColor ("Dissolve Edge Color", Color) = (0.5,0.8,1,1)
    }
    
    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        LOD 300
        
        Pass {
            Name "FreezeForward"
            Tags { "LightMode"="UniversalForward" }
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile _ _SCREEN_SPACE_OCCLUSION
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            
            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
            };
            
            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 normalWS   : TEXCOORD1;
                float3 positionWS : TEXCOORD2;
                float4 screenPos  : TEXCOORD3;
            };
            
            TEXTURE2D(_BaseMap);        SAMPLER(sampler_BaseMap);
            TEXTURE2D(_IceNoiseTex);    SAMPLER(sampler_IceNoiseTex);
            TEXTURE2D(_CameraOpaqueTexture); SAMPLER(sampler_CameraOpaqueTexture);
            
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _BaseColor;
                float  _FreezeProgress;
                float4 _FreezeColor;
                float  _FreezeHeight;
                float  _IceNoiseScale;
                float  _CrystalIntensity;
                float  _RefractionStrength;
                float  _IceSpikeAmount;
                float  _DissolveProgress;
                float  _DissolveEdgeWidth;
                float4 _DissolveEdgeColor;
            CBUFFER_END
            
            // Voronoi 噪声（简化版，移动端友好）
            float2 voronoi(float2 uv) {
                float2 i = floor(uv);
                float2 f = frac(uv);
                float minDist = 8.0;
                float2 closest = 0;
                
                [unroll]
                for (int x = -1; x <= 1; x++) {
                    [unroll]
                    for (int y = -1; y <= 1; y++) {
                        float2 neighbor = float2(x, y);
                        float2 point = frac(sin(float2(
                            dot(i + neighbor, float2(127.1, 311.7)),
                            dot(i + neighbor, float2(269.5, 183.3)
                        ))) * 43758.5453);
                        point = 0.5 + 0.5 * sin(point * 6.2831);
                        float2 diff = neighbor + point - f;
                        float dist = length(diff);
                        if (dist < minDist) {
                            minDist = dist;
                            closest = i + neighbor;
                        }
                    }
                }
                return float2(minDist, 0);
            }
            
            Varyings vert(Attributes input) {
                Varyings output;
                
                // 计算冰冻遮罩（基于高度 + 进度）
                float heightFactor = saturate(
                    (input.positionOS.y - _FreezeHeight) * 2.0 + 0.5
                );
                // 噪声扰动使边界更自然
                float noise = SAMPLE_TEXTURE2D_LOD(_IceNoiseTex, sampler_IceNoiseTex,
                    input.uv * _IceNoiseScale, 0).r;
                float freezeMask = smoothstep(
                    1.0 - _FreezeProgress - 0.1,
                    1.0 - _FreezeProgress + 0.1,
                    heightFactor + noise * 0.2
                );
                
                // 顶点外扩（冰刺）
                float3 posOS = input.positionOS.xyz;
                float spikeNoise = noise * 2.0 - 1.0;
                posOS += input.normalOS * _IceSpikeAmount * freezeMask * (0.5 + spikeNoise);
                
                output.positionCS = TransformObjectToHClip(posOS);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                output.normalWS = TransformObjectToWorldNormal(input.normalOS);
                output.positionWS = TransformObjectToWorld(posOS);
                output.screenPos = ComputeScreenPos(output.positionCS);
                
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target {
                // === 冰冻遮罩 ===
                float heightFactor = saturate(
                    (input.positionWS.y - _FreezeHeight) * 0.5 + 0.5
                );
                float iceNoise = SAMPLE_TEXTURE2D(_IceNoiseTex, sampler_IceNoiseTex,
                    input.uv * _IceNoiseScale).r;
                float freezeMask = smoothstep(
                    1.0 - _FreezeProgress - 0.1,
                    1.0 - _FreezeProgress + 0.1,
                    heightFactor + iceNoise * 0.2
                );
                
                // === 冰晶纹理 ===
                float2 voro = voronoi(input.uv * _IceNoiseScale * 2.0);
                float crystalPattern = voro.x; // 0~1
                float crystal = smoothstep(0.0, 0.3, crystalPattern) * _CrystalIntensity;
                
                // === 基础颜色混合 ===
                half4 baseCol = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;
                half4 iceCol = _FreezeColor;
                iceCol.rgb += crystal * 0.3; // 冰晶高光
                
                half4 finalCol = lerp(baseCol, iceCol, freezeMask);
                
                // === 假折射 ===
                float2 screenUV = input.screenPos.xy / input.screenPos.w;
                float2 refractionOffset = input.normalWS.xz * _RefractionStrength * freezeMask;
                half3 refractionCol = SAMPLE_TEXTURE2D(_CameraOpaqueTexture,
                    sampler_CameraOpaqueTexture, screenUV + refractionOffset).rgb;
                finalCol.rgb = lerp(finalCol.rgb, refractionCol, freezeMask * 0.4);
                
                // === 冰面法线扰动（简单光照增强） ===
                float3 perturbedNormal = normalize(input.normalWS +
                    float3(crystal * 0.3, 0, crystal * 0.3));
                Light mainLight = GetMainLight();
                half NdotL = saturate(dot(perturbedNormal, mainLight.direction));
                finalCol.rgb *= (0.7 + NdotL * 0.3 * freezeMask);
                
                // === Dissolve 消散 ===
                float dissolveThreshold = _DissolveProgress;
                float dissolveMask = iceNoise - dissolveThreshold;
                float edge = smoothstep(0.0, _DissolveEdgeWidth, dissolveMask);
                clip(dissolveMask);
                finalCol.rgb = lerp(_DissolveEdgeColor.rgb, finalCol.rgb, edge);
                
                return finalCol;
            }
            ENDHLSL
            }
        }
    }
    FallBack "Universal Render Pipeline/Lit"
}
```

**C# 状态机控制（战斗系统联动）：**

```csharp
using UnityEngine;
using System.Collections;

public enum FreezeState { Normal, Freezing, Frozen, Cracking, Shattered }

public class FreezeEffectController : MonoBehaviour {
    [Header("References")]
    public Material targetMaterial;
    public ParticleSystem shatterParticles;
    
    [Header("Timing")]
    public float freezeDuration = 0.8f;
    public float frozenHoldDuration = 2.0f;
    public float shatterDuration = 0.5f;
    
    private FreezeState _state = FreezeState.Normal;
    private Coroutine _freezeRoutine;
    
    // 属性 ID 缓存
    static readonly int FreezeProgressID = Shader.PropertyToID("_FreezeProgress");
    static readonly int DissolveProgressID = Shader.PropertyToID("_DissolveProgress");
    
    public void TriggerFreeze() {
        if (_freezeRoutine != null) StopCoroutine(_freezeRoutine);
        _freezeRoutine = StartCoroutine(FreezeSequence());
    }
    
    IEnumerator FreezeSequence() {
        // Phase 1: 冰冻蔓延
        _state = FreezeState.Freezing;
        float t = 0;
        while (t < freezeDuration) {
            t += Time.deltaTime;
            targetMaterial.SetFloat(FreezeProgressID, t / freezeDuration);
            yield return null;
        }
        targetMaterial.SetFloat(FreezeProgressID, 1f);
        
        // Phase 2: 冰冻维持
        _state = FreezeState.Frozen;
        yield return new WaitForSeconds(frozenHoldDuration);
        
        // Phase 3: 裂纹 → 破碎
        _state = FreezeState.Cracking;
        if (shatterParticles != null) shatterParticles.Play();
        
        t = 0;
        while (t < shatterDuration) {
            t += Time.deltaTime;
            targetMaterial.SetFloat(DissolveProgressID, t / shatterDuration);
            yield return null;
        }
        
        // Phase 4: 完全消散
        _state = FreezeState.Shattered;
        targetMaterial.SetFloat(FreezeProgressID, 0f);
        targetMaterial.SetFloat(DissolveProgressID, 0f);
    }
    
    public void ForceShatter() {
        if (_state == FreezeState.Frozen && _freezeRoutine != null) {
            StopCoroutine(_freezeRoutine);
            _freezeRoutine = StartCoroutine(ShatterImmediate());
        }
    }
    
    IEnumerator ShatterImmediate() {
        if (shatterParticles != null) shatterParticles.Play();
        float t = 0;
        while (t < 0.3f) {
            t += Time.deltaTime;
            targetMaterial.SetFloat(DissolveProgressID, t / 0.3f);
            yield return null;
        }
    }
}
```

**效果分层与性能预算（移动端）：**

| 效果层 | 技术方案 | 性能开销 | 移动端取舍 |
|--------|----------|----------|------------|
| 蔓延遮罩 | 高度+噪声遮罩 | 极低 | ✅ 必须保留 |
| 冰晶纹理 | Voronoi 噪声 | 中（ALU） | ⚠️ 可简化为预烘焙纹理 |
| 折射 | GrabPass/Opaque Texture | 高（带宽） | ❌ 低端机关闭，用高光替代 |
| 冰刺变形 | 顶点外扩 | 低 | ✅ 保留但控制幅度 |
| Dissolve | 噪声+clip | 低 | ✅ 保留 |

### ⚯ 实战经验

1. **移动端折射的廉价替代**：真正的折射在移动端带宽消耗太大。用 Fresnel + Cubemap 反射 + 蓝色调偏移可以"骗"过玩家的眼睛，效果接近但性能好 10 倍
2. **Voronoi 可以预烘焙**：实时计算 Voronoi 在移动端 ALU 开销不小。可以把 Voronoi 纹理预烘焙到一张 RGBA 贴图里，运行时只需要采样，配合法线扰动效果几乎一样
3. **冰冻状态和动画系统联动**：冰冻蔓延时角色应播放颤抖动画、冻结时切到 Idle 冻结帧、破碎时触发 Ragdoll——这些不是 Shader 的事，是状态机驱动的美术效果
4. **同屏 10 个角色的性能策略**：冰冻 Shader 的 LOD 策略——近距离角色用完整版（折射+冰刺），中距离去掉折射，远距离只保留颜色变化+简单遮罩

### 🎯 能力体检清单

- [ ] **如果不知道怎么控制蔓延方向** → 你需要补：UV/空间坐标映射、smoothstep 的灵活运用、噪声纹理作为遮罩
- [ ] **如果不会做冰晶纹理** → 你需要补：Voronoi 噪声原理、程序化纹理生成、法线扰动技术
- [ ] **如果不懂折射实现** → 你需要补：URP 的 Opaque Texture 机制、屏幕空间 UV、法线扰动采样
- [ ] **如果不会做顶点变形** → 你需要补：顶点着色器基础、法线方向膨胀、变形后的阴影问题
- [ ] **如果不懂性能分级** → 你需要补：移动端 GPU 架构（TBDR）、Shader LOD、GPU 变体管理

### 🔗 相关问题

- 角色燃烧效果（Dissolve + 火焰粒子）和冰冻有什么共通的设计模式？
- 如何在战斗系统中管理"状态特效"的优先级和叠加规则？
- 移动端 Voronoi 噪声的替代方案有哪些？（预烘焙 vs SDF vs Simplex）
