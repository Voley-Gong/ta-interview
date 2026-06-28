---
title: "角色石化特效：如何用 Shader 实现从脚到头的石头蔓延+裂纹+碎裂？"
category: "shader"
level: 3
tags: ["Shader", "石化", "URP", "顶点色", "噪声遮罩", "法线扰动", "战斗特效"]
hint: "石化的核心是「遮罩驱动蔓延」+「岩石法线扰动改变光照」+「裂纹生成」+「颜色置换」，和冰冻/溶解同源但质感完全不同"
related: ["shader/freeze-crystal-effect", "shader/dissolve-effect", "shader/character-armor-shatter-effect"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们做一个二次元 ARPG，Boss 有一个"美杜莎之眼"技能——中了技能的角色会从脚开始逐渐变成石头，石化过程中颜色变为灰岩石质感、表面出现裂纹、动作越来越慢，最终完全石化后可以被击碎成碎石。请设计完整的 Shader 方案，移动端 URP，同屏可能 5 个角色同时石化。

这是叠纸、鹰角、米哈游等二次元项目的经典战斗特效面试题。和冰冻/溶解同属"遮罩蔓延系"，但石化效果的核心差异在于**质感置换**——不是简单的颜色混合，而是将光滑角色表面的法线和粗糙度完全替换为岩石质感。

### ✅ 核心要点

1. **三段式遮罩系统**：用顶点色 R 通道存储"可石化区域"、噪声纹理控制蔓延边缘、进度参数驱动整体覆盖
2. **法线置换（质感核心）**：石化区域用预烘焙的岩石法线贴图替换原始法线，彻底改变光照响应
3. **裂纹生成**：Voronoi/Cellular 噪声生成裂纹图案，在石化边缘和内部生成不同密度裂纹
4. **颜色与粗糙度置换**：基础 PBR 参数（albedo/smoothness/metallic）在石化区域整体替换为岩石参数
5. **动画减速联动**：石化进度通过 Shader 之外，由 C# 控制动画速度递减至 0

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
期望效果：角色脚部开始石化 → 蔓延至全身 → 灰岩石质感 → 裂纹 → 可击碎
         ↓
质感拆解（石化 ≠ 变灰，是完整的材质替换）：
  Layer 1：蔓延遮罩（控制石化扩散方向与进度）
     ↓
  Layer 2：岩石法线置换（改变表面光照 = 质感的核心）
     ↓
  Layer 3：裂纹生成（Voronoi 裂纹 → 内部细裂纹 + 边缘裂纹带）
     ↓
  Layer 4：颜色/粗糙度替换（albedo→灰褐色, smoothness→粗糙, metallic→0）
     ↓
  Layer 5：碎裂消散（击碎时 = 不规则碎块 + 石屑粒子）
```

#### 知识点拆解（倒推树）

```
石化特效
├── 蔓延遮罩系统
│   ├── 需要理解：遮罩的"有机蔓延"
│   │   ├── 方案A：纯 Y 轴高度（太死板，石化像水平面上升）
│   │   ├── 方案B：纯噪声阈值（没有方向感）
│   │   └── 方案C：Y轴 + 噪声扰动 + 顶点色权重（推荐）
│   │       └── 顶点色 R = 1 表示可石化区域，R = 0 表示免疫（如武器）
│   ├── 核心参数：_PetrifyProgress (0→1)
│   └── 边缘过渡：硬边 vs 渐变（石化边缘应该是参差不齐的）
│
├── 岩石法线置换（核心差异点）
│   │  ← 这是石化和冰冻/溶解的根本区别
│   ├── 需要理解：为什么不能只换颜色？
│   │   └── 光照由法线决定。光滑球体变灰色还是光滑的灰球
│   │       只有法线被替换为岩石凹凸法线才有"石头质感"
│   ├── 方案A：Detail Normal Map 叠加（简单但不够强烈）
│   ├── 方案B：完全替换法线（推荐，石化区域用岩石法线贴图）
│   │   └── normalWS = lerp(originalNormal, rockNormal, petrifyMask)
│   └── 方案C：程序化法线扰动（无贴图，用噪声梯度）
│       └── 移动端友好但质感不如预烘焙
│
├── 裂纹系统
│   ├── 需要理解：Cellular/Voronoi 噪声的裂纹提取
│   │   └── Cellular Noise F2-F1 → 边界处形成裂纹线
│   ├── 裂纹密度分层：
│   │   ├── 石化边缘：密集裂纹带（transition zone）
│   │   └── 石化内部：稀疏大裂纹（已稳定区域）
│   ├── 裂纹深度模拟：
│   │   └── 裂纹处法线向下凹陷 → 暗色 → 视觉深度
│   └── 时间演化：裂纹随时间缓慢"生长"（_CrackGrowth 参数）
│
├── PBR 参数置换
│   ├── Albedo：原色 → 灰褐色（采样岩石贴图或程序化颜色）
│   ├── Smoothness：原值 → 0.1~0.3（粗糙石头）
│   ├── Metallic：→ 0（非金属）
│   └── AO：石化区域增加接触阴影感
│
└── 碎裂系统
    ├── 与 Dissolve 的差异：
    │   ├── Dissolve = 像素级消散（边缘粒子化）
    │   └── 碎裂 = 块状分离（碎石飞溅）
    ├── 方案：顶点动画 + 碎块 Prefab
    │   ├── 简单版：Dissolve + 灰色碎屑粒子
    │   └── 高级版：模型分块（预先切割）+ 物理碎块
    └── 性能权衡：完整碎裂效果 = 高开销，移动端用简化版
```

#### 代码实现

**石化 Shader（URP，核心 Pass）：**

```hlsl
// PetrifyEffect.shader — URP 兼容
Shader "TA/PetrifyEffect" {
    Properties {
        [Header(Base)]
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1,1,1,1)
        _Smoothness ("Original Smoothness", Range(0,1)) = 0.5
        
        [Header(Petrify Progress)]
        _PetrifyProgress ("Petrify Progress", Range(0,1)) = 0
        _EdgeWidth ("Edge Width", Range(0.01, 0.3)) = 0.08
        _EdgeColor ("Edge Glow Color", Color) = (0.9, 0.7, 0.3, 1)
        
        [Header(Rock Surface)]
        _RockNormalTex ("Rock Normal Tex", 2D) = "bump" {}
        _RockNormalScale ("Rock Normal Scale", Float) = 3.0
        _RockAlbedoTex ("Rock Albedo Tex", 2D) = "gray" {}
        _RockColor ("Rock Tint Color", Color) = (0.55, 0.52, 0.48, 1)
        _RockSmoothness ("Rock Smoothness", Range(0,0.5)) = 0.15
        
        [Header(Cracks)]
        _CrackNoiseTex ("Crack Noise Tex", 2D) = "white" {}
        _CrackScale ("Crack Scale", Float) = 8.0
        _CrackWidth ("Crack Width", Range(0.01, 0.2)) = 0.04
        _CrackDepth ("Crack Depth", Range(0,1)) = 0.7
        _CrackGrowth ("Crack Growth", Range(0,1)) = 1.0
        
        [Header(Shatter)]
        _ShatterProgress ("Shatter Progress", Range(0,1)) = 0
        _ShatterNoiseScale ("Shatter Noise Scale", Float) = 4.0
    }
    
    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        LOD 300
        
        Pass {
            Name "PetrifyForward"
            Tags { "LightMode"="UniversalForward" }
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            
            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float4 tangentOS  : TANGENT;
                float2 uv         : TEXCOORD0;
                float4 vertexColor : COLOR; // R=petrify mask
            };
            
            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 normalWS   : TEXCOORD1;
                float3 tangentWS  : TEXCOORD2;
                float3 bitangentWS: TEXCOORD3;
                float3 positionWS : TEXCOORD4;
                float  petrifyMask: TEXCOORD5;
            };
            
            TEXTURE2D(_BaseMap);       SAMPLER(sampler_BaseMap);
            TEXTURE2D(_RockNormalTex); SAMPLER(sampler_RockNormalTex);
            TEXTURE2D(_RockAlbedoTex); SAMPLER(sampler_RockAlbedoTex);
            TEXTURE2D(_CrackNoiseTex); SAMPLER(sampler_CrackNoiseTex);
            
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _BaseColor;
                float  _Smoothness;
                float  _PetrifyProgress;
                float  _EdgeWidth;
                float4 _EdgeColor;
                float4 _RockNormalTex_ST;
                float  _RockNormalScale;
                float4 _RockAlbedoTex_ST;
                float4 _RockColor;
                float  _RockSmoothness;
                float4 _CrackNoiseTex_ST;
                float  _CrackScale;
                float  _CrackWidth;
                float  _CrackDepth;
                float  _CrackGrowth;
                float  _ShatterProgress;
                float  _ShatterNoiseScale;
            CBUFFER_END
            
            // 简化 Cellular 噪声（用于裂纹）
            float cellularNoise(float2 uv) {
                float2 i = floor(uv);
                float2 f = frac(uv);
                float minDist = 1.0;
                float secondDist = 1.0;
                
                [unroll]
                for (int x = -1; x <= 1; x++) {
                    [unroll]
                    for (int y = -1; y <= 1; y++) {
                        float2 neighbor = float2(x, y);
                        float2 hash = frac(sin(dot(i + neighbor,
                            float2(127.1, 311.7))) * 43758.5453);
                        hash = 0.5 + 0.5 * sin(hash * 6.2831);
                        float2 diff = neighbor + hash - f;
                        float dist = length(diff);
                        if (dist < minDist) {
                            secondDist = minDist;
                            minDist = dist;
                        } else if (dist < secondDist) {
                            secondDist = dist;
                        }
                    }
                }
                // F2-F1：边界处值大 = 裂纹
                return secondDist - minDist;
            }
            
            Varyings vert(Attributes input) {
                Varyings output = (Varyings)0;
                
                // 计算石化遮罩
                float heightFactor = saturate(input.positionOS.y * 0.5 + 0.5);
                float noiseVal = SAMPLE_TEXTURE2D_LOD(_CrackNoiseTex,
                    sampler_CrackNoiseTex, input.uv * 2.0, 0).r;
                
                float petrifyThreshold = 1.0 - _PetrifyProgress;
                float rawMask = (heightFactor + noiseVal * 0.15) - petrifyThreshold;
                float petrifyMask = smoothstep(-_EdgeWidth, _EdgeWidth, rawMask);
                
                // 顶点色 R 控制可石化区域
                petrifyMask *= input.vertexColor.r;
                
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                output.normalWS = TransformObjectToWorldNormal(input.normalOS);
                output.tangentWS = TransformObjectToWorldDir(input.tangentOS.xyz);
                output.bitangentWS = cross(output.normalWS, output.tangentWS) *
                    input.tangentOS.w * unity_WorldTransformParams.w;
                output.positionWS = TransformObjectToWorld(input.positionOS.xyz);
                output.petrifyMask = petrifyMask;
                
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target {
                // === 石化遮罩 ===
                float petrifyMask = input.petrifyMask;
                
                // 边缘过渡带
                float edgeBand = smoothstep(0.4, 0.5, petrifyMask) *
                                 smoothstep(0.6, 0.5, petrifyMask);
                
                // === 原始材质 ===
                half4 baseAlbedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;
                float origSmoothness = _Smoothness;
                
                // === 岩石法线置换 ===
                float2 rockUV = input.uv * _RockNormalScale;
                half3 rockNormalTS = UnpackNormal(
                    SAMPLE_TEXTURE2D(_RockNormalTex, sampler_RockNormalTex, rockUV));
                
                // 构建原始 TBN
                float3x3 tbnOrig = float3x3(
                    input.tangentWS, input.bitangentWS, input.normalWS);
                half3 normalOrigTS = half3(0, 0, 1); // 模型原始法线（TS）
                
                // 裂纹法线扰动
                float crackValue = cellularNoise(input.uv * _CrackScale);
                float crackMask = 1.0 - smoothstep(0.0, _CrackWidth, crackValue);
                crackMask *= _CrackGrowth * petrifyMask;
                
                // 裂纹处法线向下凹陷
                half3 crackNormalTS = half3(0, 0, -_CrackDepth);
                half3 blendedRockTS = normalize(
                    lerp(rockNormalTS, crackNormalTS, crackMask));
                
                // 法线置换：原始 → 岩石+裂纹
                half3 finalNormalTS = lerp(normalOrigTS, blendedRockTS, petrifyMask);
                float3 finalNormalWS = normalize(mul(finalNormalTS, tbnOrig));
                
                // === 岩石 Albedo ===
                half4 rockAlbedo = SAMPLE_TEXTURE2D(_RockAlbedoTex,
                    sampler_RockAlbedoTex, input.uv * 3.0) * _RockColor;
                
                // 裂纹处变暗
                rockAlbedo.rgb *= (1.0 - crackMask * 0.5);
                
                half3 finalAlbedo = lerp(baseAlbedo.rgb, rockAlbedo.rgb, petrifyMask);
                float finalSmoothness = lerp(origSmoothness, _RockSmoothness, petrifyMask);
                
                // 边缘高光（石化蔓延前沿）
                finalAlbedo += _EdgeColor.rgb * edgeBand * 2.0;
                
                // === 光照计算 ===
                InputData lightData = (InputData)0;
                lightData.positionWS = input.positionWS;
                lightData.normalWS = finalNormalWS;
                lightData.viewDirectionWS = normalize(
                    GetCameraPositionWS() - input.positionWS);
                
                SurfaceData surf = (SurfaceData)0;
                surf.albedo = finalAlbedo;
                surf.metallic = 0.0;
                surf.smoothness = finalSmoothness;
                surf.normalTS = finalNormalTS;
                surf.alpha = 1.0;
                
                half4 color = UniversalFragmentPBR(lightData, surf);
                
                // === 碎裂 Dissolve（击碎阶段） ===
                float shatterNoise = cellularNoise(input.uv * _ShatterNoiseScale);
                float shatterMask = shatterNoise - _ShatterProgress;
                clip(shatterMask + 0.01); // 留一点边缘
                
                // 碎裂边缘高光
                float shatterEdge = smoothstep(0.0, 0.1, shatterMask) -
                                    smoothstep(0.1, 0.2, shatterMask);
                color.rgb += _EdgeColor.rgb * shatterEdge;
                
                return color;
            }
            ENDHLSL
        }
    }
    FallBack "Universal Render Pipeline/Lit"
}
```

**C# 石化状态控制器（含动画减速）：**

```csharp
using UnityEngine;
using System.Collections;

public enum PetrifyState { Normal, Petrifying, Petrified, Shattering }

public class PetrifyEffectController : MonoBehaviour {
    [Header("References")]
    public Material petrifyMaterial;
    public Animator characterAnimator;
    public ParticleSystem stoneDebrisParticles;
    
    [Header("Timing")]
    public float petrifyDuration = 1.5f;
    public float petrifiedHoldDuration = 3.0f;
    public float shatterDuration = 0.8f;
    
    [Header("Animation")]
    public AnimationCurve animSlowdownCurve = AnimationCurve.EaseInOut(0, 1, 1, 0);
    
    private PetrifyState _state = PetrifyState.Normal;
    private float _originalAnimSpeed = 1.0f;
    
    static readonly int ProgressID = Shader.PropertyToID("_PetrifyProgress");
    static readonly int ShatterID = Shader.PropertyToID("_ShatterProgress");
    static readonly int CrackGrowthID = Shader.PropertyToID("_CrackGrowth");
    
    public void TriggerPetrify() {
        StartCoroutine(PetrifySequence());
    }
    
    IEnumerator PetrifySequence() {
        // === Phase 1: 石化蔓延 + 动画减速 ===
        _state = PetrifyState.Petrifying;
        if (characterAnimator != null)
            _originalAnimSpeed = characterAnimator.speed;
        
        float t = 0;
        while (t < petrifyDuration) {
            t += Time.deltaTime;
            float progress = t / petrifyDuration;
            
            petrifyMaterial.SetFloat(ProgressID, progress);
            
            // 动画同步减速
            if (characterAnimator != null) {
                characterAnimator.speed = _originalAnimSpeed * animSlowdownCurve.Evaluate(progress);
            }
            
            // 裂纹逐渐生长
            petrifyMaterial.SetFloat(CrackGrowthID, Mathf.Lerp(0.3f, 1.0f, progress));
            
            yield return null;
        }
        
        // === Phase 2: 完全石化 ===
        _state = PetrifyState.Petrified;
        petrifyMaterial.SetFloat(ProgressID, 1f);
        if (characterAnimator != null) characterAnimator.speed = 0f;
        
        yield return new WaitForSeconds(petrifiedHoldDuration);
        
        // === Phase 3: 击碎 ===
        _state = PetrifyState.Shattering;
        if (stoneDebrisParticles != null) stoneDebrisParticles.Play();
        
        t = 0;
        while (t < shatterDuration) {
            t += Time.deltaTime;
            petrifyMaterial.SetFloat(ShatterID, t / shatterDuration);
            yield return null;
        }
        
        // 重置（或销毁）
        ResetEffect();
    }
    
    void ResetEffect() {
        petrifyMaterial.SetFloat(ProgressID, 0f);
        petrifyMaterial.SetFloat(ShatterID, 0f);
        petrifyMaterial.SetFloat(CrackGrowthID, 0f);
        if (characterAnimator != null)
            characterAnimator.speed = _originalAnimSpeed;
        _state = PetrifyState.Normal;
    }
    
    // 外部触发：攻击石化目标 → 直接碎裂
    public void ShatterImmediate() {
        if (_state == PetrifyState.Petrified) {
            StopAllCoroutines();
            StartCoroutine(ShatterOnly());
        }
    }
    
    IEnumerator ShatterOnly() {
        _state = PetrifyState.Shattering;
        if (stoneDebrisParticles != null) stoneDebrisParticles.Play();
        
        float t = 0;
        while (t < 0.5f) {
            t += Time.deltaTime;
            petrifyMaterial.SetFloat(ShatterID, t / 0.5f);
            yield return null;
        }
        ResetEffect();
    }
}
```

**石化 vs 冰冻 vs 溶解 对比表：**

| 维度 | 石化 | 冰冻 | 溶解 |
|------|------|------|------|
| 蔓延遮罩 | ✅ Y轴+噪声+顶点色 | ✅ Y轴+噪声 | ✅ 噪声+进度 |
| 颜色置换 | → 灰褐色 | → 冰蓝色 | → 无（消失） |
| **法线置换** | ✅ **岩石法线替换** | ✅ 冰晶法线扰动 | ❌ 无 |
| 顶点变形 | 可选（轻微膨胀） | ✅ 冰刺外扩 | ❌ 无 |
| 折射 | ❌ 不透明 | ✅ 冰层折射 | ❌ 无 |
| 消散方式 | **块状碎裂** | 冰晶飞溅 | 像素粒子化 |
| 动画联动 | ✅ **减速至停止** | ✅ 冻结帧 | ❌ 无 |
| 核心差异 | **质感替换** | **光学效果** | **透明度过渡** |

### ⚡ 实战经验

1. **法线置换是石化效果的灵魂**：很多新手只做颜色变灰，结果看起来像"灰色滤镜"。真正的石化质感来自法线——当角色的光滑法线被替换为粗糙岩石法线时，光照立刻产生石头般的凹凸感。这是石化 vs 冰冻的根本区别
2. **岩石法线贴图的选择**：不要用通用噪声法线，去 Substance Painter / Designer 里生成真正的岩石材质法线，或者从 Quixel Bridge 下载岩石扫描数据。法线的频率和深度决定了石头的"年龄感"——高频小凹凸像砂岩，低频大凹凸像花岗岩
3. **裂纹的时间演化**：裂纹不应该在石化的一瞬间全部出现。蔓延前沿裂纹密集（新鲜的石头在"生长"），已石化区域裂纹随时间扩展。用 `_CrackGrowth` 参数从 0.3 → 1.0 渐变
4. **动画减速是叙事关键**：石化不仅是视觉变化，更是一种"剥夺感"。动画从正常 → 颤抖 → 缓慢 → 冻结的过程让玩家感受到"角色在被慢慢困住"。用 AnimationCurve 控制比线性减速更有戏剧性
5. **同屏 5 个角色的性能策略**：岩石法线贴图采样 + Cellular 噪声计算在移动端不便宜。低端机降级方案：①预烘焙裂纹到顶点色/UV2 ②法线置换简化为 Detail Normal ③关闭裂纹时间演化

### 🎯 能力体检清单

- [ ] **如果只做了颜色变化没有法线置换** → 你需要补：法线在 PBR 光照中的角色、Detail Normal 叠加 vs 法线完全替换、TBN 矩阵的构建与使用
- [ ] **如果裂纹看起来像随机噪点而非石头裂纹** → 你需要补：Cellular/Voronoi 噪声（F2-F1 提取边界）、裂纹的几何特征（分叉、宽窄变化）、程序化裂纹 vs 预烘焙裂纹的取舍
- [ ] **如果不会联动动画系统** → 你需要补：Animator.speed 属性、AnimationCurve 的运行时求值、状态机与 Shader 参数的同步策略
- [ ] **如果碎裂效果和溶解看起来一样** → 你需要补：Dissolve vs Shatter 的视觉差异（均匀消散 vs 块状分离）、碎裂的物理模拟思路（刚体碎块 vs Shader 假碎裂）
- [ ] **如果不懂移动端性能分级** → 你需要补：Shader 变体管理（多版本 Shader LOD）、ALU 预算估算、移动端纹理采样次数限制

### 🔗 相关问题

- 石化和冰冻同属"遮罩蔓延系"，如何设计一个通用的「状态特效框架」来复用遮罩逻辑？
- 如果 Boss 同时施放冰冻和石化，两种效果如何叠加？优先级怎么定？
- 如何在不增加 Draw Call 的前提下，让石化效果支持同屏 10 个不同进度的角色？（MaterialPropertyBlock vs GPU Instancing with per-instance data）
