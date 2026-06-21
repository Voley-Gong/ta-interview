---
title: "角色溶解消失：如何用 Shader 实现可控的 burn-out 效果？"
category: "shader"
level: 2
tags: ["Dissolve", "噪声纹理", "AlphaTest", "Shader", "URP"]
hint: "核心是噪声纹理阈值 + 边缘发光带——别只用 discard，要做出灼烧边缘的质感"
related: ["shader/npr-outline-cartoon", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们游戏中角色死亡时需要一个溶解消失的效果——身体从边缘开始逐渐变成飞散的灰烬，溶解边缘要有一圈灼烧的发光感。URP 下用 Shader 实现，给我方案。」

### ✅ 核心要点

1. **噪声纹理驱动**：用 Noise Map（Simplex/Perlin）控制溶解的时空顺序
2. **阈值渐进裁剪**：`clip(noise - threshold)` 实现 AlphaTest 式的逐步消失
3. **边缘发光带**：在 threshold 附近一小段范围内叠加 Emission，模拟灼烧
4. **可控参数化**：溶解进度、边缘宽度、灼烧颜色暴露给材质或 C# 脚本
5. **粒子配合**：纯 Shader 溶解只是「消失」，加粒子才是「飞散」

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色从边缘开始 → 逐步消失 → 边缘有灼烧发光 → 灰烬飞散
                ↑
倒推1：需要控制「哪里先消失」→ 噪声纹理 R 通道
倒推2：需要控制「何时消失」→ _DissolveAmount 参数 (0→1)
倒推3：需要「灼烧感」→ 在 threshold ± _EdgeWidth 范围叠加 Emission
倒推4：需要「灰烬飞散」→ 粒子系统 + 角色骨骼挂点
倒推5：需要「可控触发」→ C# 脚本协程插值 _DissolveAmount
```

#### 知识点拆解（倒推树）

```
角色溶解消失
├── Shader 核心
│   ├── Noise 纹理采样（UV 缩放/偏移控制密度）
│   ├── clip() / AlphaTest（硬边裁剪 vs alpha blend 对比）
│   ├── smoothstep() 控制边缘过渡（避免硬边）
│   └── Emission 边缘增强（Step 1：检测边缘 → Step 2：叠加颜色）
├── URP 集成
│   ├── Shader Graph 搭建 vs 手写 HLSL（各适用场景）
│   ├── 材质属性暴露：_DissolveAmount, _EdgeWidth, _BurnColor
│   └── SRP Batcher 兼容性检查（CBUFFER 块）
├── C# 驱动
│   ├── MaterialPropertyBlock（不要 new Material 实例）
│   ├── 协程/DOTween 插值控制
│   └── 粒子系统参数联动
└── 性能注意
    ├── Overdraw：半透明边缘叠加可能产生 overdraw
    ├── 纹理采样：Noise 纹理尽量复用，不要每角色一张
    └── GPU Instancing：dissolve 参数需 per-instance
```

#### 代码实现

**手写 HLSL（URP 兼容）：**

```hlsl
Shader "Custom/Dissolve"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _NoiseMap ("Noise Map", 2D) = "white" {}
        _DissolveAmount ("Dissolve Amount", Range(0, 1)) = 0
        _EdgeWidth ("Edge Width", Range(0.01, 0.3)) = 0.1
        _BurnColor ("Burn Color", Color) = (1, 0.5, 0, 1)
        _NoiseScale ("Noise Scale", Float) = 1.0
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
                float4 _BurnColor;
                float _DissolveAmount;
                float _EdgeWidth;
                float _NoiseScale;
            CBUFFER_END

            TEXTURE2D(_BaseMap);   SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NoiseMap);  SAMPLER(sampler_NoiseMap);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                // 采样噪声纹理（用 UV * scale 控制密度）
                float2 noiseUV = IN.uv * _NoiseScale;
                half noiseValue = SAMPLE_TEXTURE2D(_NoiseMap, sampler_NoiseMap, noiseUV).r;

                // 阈值裁剪：noiseValue < dissolveAmount 的区域被 discard
                half dissolveThreshold = _DissolveAmount;
                clip(noiseValue - dissolveThreshold);

                // 边缘发光：在裁剪边缘附近（noiseValue 接近 threshold）
                half edgeFactor = 1.0 - saturate((noiseValue - dissolveThreshold) / _EdgeWidth);
                edgeFactor = smoothstep(0.0, 1.0, edgeFactor);

                // 基础颜色
                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);

                // 叠加灼烧色
                baseColor.rgb += _BurnColor.rgb * edgeFactor * 3.0;

                return baseColor;
            }
            ENDHLSL
        }
    }
}
```

**C# 触发溶解：**

```csharp
using System.Collections;
using UnityEngine;

public class DissolveController : MonoBehaviour
{
    [SerializeField] private Renderer[] renderers;
    [SerializeField] private float duration = 2f;

    private MaterialPropertyBlock _mpb;
    private static readonly int DissolveAmountID = Shader.PropertyToID("_DissolveAmount");

    void Awake() => _mpb = new MaterialPropertyBlock();

    public void StartDissolve()
    {
        StartCoroutine(DissolveRoutine());
    }

    IEnumerator DissolveRoutine()
    {
        float elapsed = 0f;
        while (elapsed < duration)
        {
            float t = elapsed / duration;
            foreach (var r in renderers)
            {
                r.GetPropertyBlock(_mpb);
                _mpb.SetFloat(DissolveAmountID, t);
                r.SetPropertyBlock(_mpb);
            }
            elapsed += Time.deltaTime;
            yield return null;
        }
    }
}
```

**对比表格：Dissolve 实现方案**

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| clip() AlphaTest | 简单高效，无需排序 | 硬边，无半透明 | 卡通/像素风格 |
| Alpha Blend 渐隐 | 边缘柔和 | 需要排序，有 Overdraw | 写实风格 |
| clip + 边缘 Emission | 视觉效果最佳 | 需要噪声纹理 | 主流方案（推荐） |
| 后处理像素溶解 | 全屏统一 | 算力高，需 RT | 过场动画 |

### ⚡ 实战经验

- **噪声纹理选择**：Simplex/Perlin 效果好但需要外部生成，运行时用 `tex2D` 采样的噪声图（64x64 可平铺）最省事
- **边缘宽度动态调整**：随 _DissolveAmount 递增时调大 _EdgeWidth，灼烧带会从细到粗，更有层次感
- **多角色复用**：用不同 UV 偏移采样同一张噪声图，避免每个角色一张纹理
- **SRP Batcher**：把所有参数放进 `CBUFFER_START(UnityPerMaterial)`，否则合批会断裂

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么控制「哪里先消失」 | 噪声纹理原理 | 学 Perlin/Simplex noise 生成 |
| 边缘灼烧做不出来 | smoothstep / step 函数 | 复习 CG 数学函数（smoothstep, lerp, saturate） |
| 多角色 dissolve 合批断裂 | SRP Batcher 兼容性 | 学 URP CBUFFER 规范 |
| 粒子和 Shader 不同步 | 粒子系统参数联动 | 学习 Particle System API + 事件回调 |
| 溶解后角色 Collider 还在 | 组件生命周期 | 理解 GameObject 销毁 vs 激活控制 |

### 🔗 相关问题

- 角色复活时的「反向溶解」怎么实现？（提示：反转阈值方向 + 从核心到边缘）
- 如何用 Compute Shader 实现更高级的像素飞散效果？
- Shader Graph 中如何实现 dissolve？手写和 Graph 的性能差异？
