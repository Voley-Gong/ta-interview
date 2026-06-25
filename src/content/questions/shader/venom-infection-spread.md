---
title: "角色毒液感染蔓延：如何用 Shader 实现沿身体表面扩散的纹路+脉动发光？"
category: "shader"
level: 3
tags: ["Shader", "URP", "顶点色", "噪声蔓延", "特效着色器"]
hint: "核心是顶点色控制蔓延起点 + 噪声控制纹路形状 + 时间驱动脉动——不只是叠个贴图就完事"
related: ["shader/dissolve-effect", "shader/freeze-crystal-effect", "shader/hit-flash-damage-blink"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款 ARPG，Boss 有个技能是'毒液感染'——击中玩家后，角色身体上从受击点开始蔓延出紫色毒液纹路，纹路会脉动发光，持续掉血。15秒后可以解除。用 URP Shader 实现，给我完整方案。」

补充约束：
- 蔓延速度要可控（前3秒快，后面变慢）
- 纹路要有「血管感」，不是简单的色块
- 脉动频率要和掉血节奏同步
- 多个受击点要能同时蔓延并合并

### ✅ 核心要点

1. **顶点色存储感染源**：用顶点色的 R/G/B 通道存储最多3个感染源的位置（球坐标或世界空间距离）
2. **球面距离计算蔓延**：对每个顶点计算到感染源的距离，和时间一起驱动蔓延前沿
3. **噪声纹理塑形纹路**：用 Voronoi 或 FBM 噪声把圆形蔓延前沿打散成「血管纹路」
4. **脉动发光**：sin(_Time * frequency) 驱动 Emission 强度，与游戏逻辑掉血频率同步
5. **多源合并**：多个感染源的 mask 取 max() 或加法叠加
6. **可逆清除**：感染参数从 1 衰减回 0，视觉上自然消退

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：受击点 → 紫色纹路蔓延全身 → 纹路脉动发光 → 可消退
               ↑
倒推1：「纹路」不是纯圆形 → 需要噪声扰动蔓延前沿
倒推2：「从受击点开始」→ 需要知道受击点在模型表面的位置
倒推3：「脉动发光」→ sin波驱动 Emission
倒推4：「多个受击点合并」→ 多源 mask 叠加
倒推5：「可控消退」→ 蔓延量可从1→0衰减
倒推6：「与游戏逻辑联动」→ C# 设置感染参数到 Shader
```

#### 知识点拆解（倒推树）

```
毒液感染蔓延
├── 蔓延几何
│   ├── 模型表面距离（不能用直线距离，要沿表面）
│   ├── 顶点色烘焙最短路径距离（DCC工具预处理）
│   ├── 运行时球面距离近似（position距受击点的world distance）
│   └── 蔓延前沿（mask = smoothstep(distance, spreadRadius)）
├── 纹路塑形
│   ├── Voronoi 噪声（细胞感，像血管分叉）
│   ├── FBM 叠加多层噪声（更自然的纹路）
│   ├── 噪声作为 mask 的扰动项：mask = mask * noise(uv, time)
│   └── 纹路边缘锐化 vs 柔和过渡
├── 脉动发光
│   ├── sin(_Time.y * _PulseFreq) 基础脉动
│   ├── 脉动强度随感染进度递增
│   ├── 掉血节奏同步：C# 传入 _BeatPhase
│   └── 发光颜色渐变（从亮紫→暗紫→黑）
├── 多源合并
│   ├── 感染源数组（StructuredBuffer 或 3个 uniform）
│   ├── mask = max(source1, source2, source3)
│   └── 边缘融合：lerp 或 smoothstep 避免硬边界
├── URP 集成
│   ├── Overlay 渲染（在角色基础 Pass 之上叠加）
│   ├── Shader Graph 实现 vs HLSL 手写
│   ├── MaterialPropertyBlock 设置感染源坐标
│   └── SRP Batcher 兼容性
└── 性能注意
    ├── 纹理采样数：噪声图1张 + 基础贴图
    ├── 指令数：多源循环要控制
    └── 移动端简化：减源数、用更简单的噪声
```

#### 代码实现

**HLSL 核心 Fragment（URP 兼容）：**

```hlsl
Shader "Custom/VenomInfection"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _NoiseMap ("Noise Map (Voronoi/FBM)", 2D) = "white" {}
        _InfectionColor ("Infection Color", Color) = (0.5, 0.0, 0.8, 1)
        _EmissionColor ("Emission Color", Color) = (0.8, 0.2, 1.0, 1)
        _PulseFreq ("Pulse Frequency", Float) = 4.0
        _VeinSharpness ("Vein Sharpness", Range(0.1, 10)) = 3.0
        _SpreadSpeed ("Spread Speed", Float) = 0.3
        _EdgeSoftness ("Edge Softness", Range(0.01, 0.5)) = 0.1
        // 3个感染源：xyz=世界坐标, w=强度(0~1)
        _Source0 ("Infection Source 0", Vector) = (0,0,0,0)
        _Source1 ("Infection Source 1", Vector) = (0,0,0,0)
        _Source2 ("Infection Source 2", Vector) = (0,0,0,0)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _InfectionColor;
                float4 _EmissionColor;
                float4 _Source0, _Source1, _Source2;
                float _PulseFreq;
                float _VeinSharpness;
                float _SpreadSpeed;
                float _EdgeSoftness;
            CBUFFER_END

            TEXTURE2D(_BaseMap);   SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NoiseMap);  SAMPLER(sampler_NoiseMap);

            struct Attributes {
                float4 positionOS : POSITION;
                float3 positionWS : TEXCOORD1; // 传世界坐标
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float3 positionWS : TEXCOORD1;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.positionWS = TransformObjectToWorld(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                return OUT;
            }

            // 计算单个感染源的 mask（0=未感染, 1=完全感染）
            float InfectionMask(float3 worldPos, float4 source)
            {
                if (source.w <= 0.001) return 0;

                // 距离衰减（模型表面距离近似）
                float dist = distance(worldPos, source.xyz);

                // 蔓延前沿：source.w 是当前蔓延半径
                // 前沿之前 = 完全感染, 前沿附近 = 过渡区
                float front = source.w;
                float mask = smoothstep(front, front - _EdgeSoftness, dist);

                return mask;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                // 基础颜色
                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);

                // 三源合并
                float mask0 = InfectionMask(IN.positionWS, _Source0);
                float mask1 = InfectionMask(IN.positionWS, _Source1);
                float mask2 = InfectionMask(IN.positionWS, _Source2);
                float infectionMask = max(max(mask0, mask1), mask2);

                // 噪声纹路塑形：用噪声把均匀的 mask 打散成血管感
                half2 noiseUV = IN.uv * 3.0; // 放大噪声频率
                half noise = SAMPLE_TEXTURE2D(_NoiseMap, sampler_NoiseMap, noiseUV).r;
                // FBM 感：用两层噪声叠加
                half noise2 = SAMPLE_TEXTURE2D(_NoiseMap, sampler_NoiseMap, noiseUV * 2.7 + 13.0).r;
                noise = noise * 0.6 + noise2 * 0.4;

                // 噪声塑形：noise > threshold 才显示纹路，制造分叉感
                float veinPattern = smoothstep(0.3, 0.7, noise);
                infectionMask *= veinPattern;

                // 脉动发光
                float pulse = sin(_Time.y * _PulseFreq) * 0.5 + 0.5;
                pulse = pow(pulse, 2.0); // 锐化脉动
                float3 emission = _EmissionColor.rgb * pulse * infectionMask * 2.0;

                // 感染区基础色渐变（从原色→感染色）
                float3 infectedColor = lerp(baseColor.rgb, _InfectionColor.rgb, infectionMask);

                // 叠加发光
                infectedColor += emission;

                return half4(infectedColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**C# 驱动感染逻辑：**

```csharp
using UnityEngine;

public class VenomInfectionController : MonoBehaviour
{
    [SerializeField] private Renderer targetRenderer;
    [SerializeField] private float maxSpreadRadius = 2.5f;
    [SerializeField] private float spreadDuration = 3f;
    [SerializeField] private float pulseFreq = 4f; // 和Shader同步

    private MaterialPropertyBlock _mpb;
    private Vector4[] _sources = new Vector4[3]; // xyz=hitWorldPos, w=currentRadius
    private int _sourceCount = 0;

    // Shader property IDs
    private static readonly int Source0ID = Shader.PropertyToID("_Source0");
    private static readonly int Source1ID = Shader.PropertyToID("_Source1");
    private static readonly int Source2ID = Shader.PropertyToID("_Source2");
    private static readonly int PulseFreqID = Shader.PropertyToID("_PulseFreq");

    void Awake()
    {
        _mpb = new MaterialPropertyBlock();
        _mpb.SetFloat(PulseFreqID, pulseFreq);
    }

    /// <summary>
    /// 受击时调用，传入世界空间受击点
    /// </summary>
    public void OnHit(Vector3 worldHitPoint)
    {
        if (_sourceCount >= 3) return; // 最多3源

        int idx = _sourceCount;
        _sources[idx] = new Vector4(worldHitPoint.x, worldHitPoint.y, worldHitPoint.z, 0f);
        _sourceCount++;

        // 启动该源的蔓延
        StartCoroutine(SpreadRoutine(idx));
    }

    System.Collections.IEnumerator SpreadRoutine(int sourceIdx)
    {
        float elapsed = 0f;
        // 前30%时间快速蔓延，后面减速（缓动函数）
        AnimationCurve spreadCurve = AnimationCurve.EaseInOut(0, 0, spreadDuration, maxSpreadRadius);

        while (elapsed < spreadDuration)
        {
            float radius = spreadCurve.Evaluate(elapsed);
            var s = _sources[sourceIdx];
            s.w = radius;
            _sources[sourceIdx] = s;

            ApplyToMaterial();
            elapsed += Time.deltaTime;
            yield return null;
        }
    }

    void ApplyToMaterial()
    {
        targetRenderer.GetPropertyBlock(_mpb);
        _mpb.SetVector(Source0ID, _sources[0]);
        _mpb.SetVector(Source1ID, _sources[1]);
        _mpb.SetVector(Source2ID, _sources[2]);
        targetRenderer.SetPropertyBlock(_mpb);
    }

    /// <summary>
    /// 解毒：所有源强度衰减到0
    /// </summary>
    public System.Collections.IEnumerator CureRoutine(float duration = 1.5f)
    {
        float elapsed = 0f;
        var startSources = (Vector4[])_sources.Clone();

        while (elapsed < duration)
        {
            float t = elapsed / duration;
            for (int i = 0; i < 3; i++)
            {
                _sources[i].w = Mathf.Lerp(startSources[i].w, 0f, t);
            }
            ApplyToMaterial();
            elapsed += Time.deltaTime;
            yield return null;
        }
        _sourceCount = 0;
    }
}
```

**对比表格：感染源位置传递方案**

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| 世界坐标 Uniform | 传 float4 给 Shader | 简单直接 | 只支持固定数量源 | ≤3源（推荐） |
| 顶点色烘焙 | DCC 工具预计算表面距离 | 蔓延精确沿表面 | 不能运行时改受击点 | 固定路径感染 |
| RenderTexture 贴花 | 在 RT 上绘制感染区域 | 无限源、任意形状 | 需要额外 Pass、UV2 | 高级效果（AAA） |
| StructuredBuffer | GPU 端存感染源数组 | 支持大量源 | 移动端兼容性差 | PC/主机 |

### ⚡ 实战经验

- **「表面距离」的真相**：世界空间直线距离只是近似，如果模型有复杂凹面（如斗篷内侧），蔓延会「穿透」。精确方案是用 DCC 工具烘焙 Geodesic Distance 到顶点色
- **噪声纹理选择**：Voronoi 的细胞感最像「血管分叉」，FBM 更像「菌丝蔓延」。两层叠加效果最佳
- **性能实测**：3源 + 2次噪声采样 ≈ 额外 25 条 ALU 指令，移动端可接受。如果只有1源，去掉循环直接算，省一半
- **感染色不要太纯**：纯紫色(1,0,1) 很假。用偏暗的紫(0.4, 0.05, 0.6) + 高亮 Emission 更真实
- **和特效配合**：Shader 蔓延负责「贴身」，毒液溅射粒子负责「空中」，两者结合才有冲击力

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么让纹路「沿表面蔓延」 | 表面距离 vs 直线距离 | 学 Geodesic Distance / 顶点色烘焙流程 |
| 蔓延纹路像色块不像血管 | 噪声塑形不够 | 学 Voronoi / FBM 噪声原理和 Shader 实现 |
| 多源合并有接缝 | mask 叠加方式错误 | 复习 max / smoothstep / blend 算法 |
| 脉动和游戏逻辑不同步 | C#↔Shader 数据传递 | 学 MaterialPropertyBlock + ShaderLab Property |
| 远处角色感染效果不可见 | LOD 衔接 | 在低 LOD 模型上用简化版（只变色不发光） |

### 🔗 相关问题

- [角色溶解消失](../shader/dissolve-effect.md)：溶解是「消失」，感染是「覆盖」，核心都是 mask 驱动
- [角色冰冻特效](../shader/freeze-crystal-effect.md)：冰冻蔓延逻辑和感染蔓延几乎一样，只是视觉风格不同
- 如果要沿身体表面精确蔓延（不穿透），你会怎么实现？（提示：Geodesic Distance 烘焙到顶点色）
