---
title: "角色受击闪白：怎么用 Shader 实现可控的 Damage Blink 效果？"
category: "shader"
level: 1
tags: ["受击反馈", "闪白", "Hit Flash", "Shader", "URP", "打击感"]
hint: "核心是叠加一个白色遮罩并随时间衰减——但要不要区分材质区域？要不要加扰动？细节决定品质"
related: ["shader/dissolve-effect", "shader/radial-blur-hit-effect", "shader/energy-shield-effect"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们动作游戏角色被打到时要有一个闪白效果——全身瞬间变白然后快速恢复。但皮肤、金属、布料闪白的强度不一样，金属要更亮。URP 下给我方案。」

### ✅ 核心要点

1. **遮罩叠加法**：在最终颜色上 `lerp(baseColor, flashColor, flashAmount)`，最简单也最通用
2. **按材质区分强度**：用顶点色 / Mask 纹理的通道控制不同区域的闪白权重
3. **时间衰减曲线**：不是线性恢复，用 `ease-out` 或 `pow` 让闪白「啪地亮、缓缓收」
4. **Emission 联动**：闪白时同步拉高 Emission，配合 Bloom 后处理打出辉光感
5. **C# 触发 + MaterialPropertyBlock**：受击事件驱动，不产生额外材质实例

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色受击 → 全身闪白 → 不同材质强度不同 → 快速衰减恢复
                ↑
倒推1：需要「叠加白色」→ lerp(baseColor, _FlashColor, _FlashAmount)
倒推2：需要「材质区分」→ 顶点色 R 通道 / Mask 纹理 A 通道做权重
倒推3：需要「快速衰减」→ C# 协程插值 _FlashAmount: 1→0，用 ease-out 曲线
倒推4：需要「辉光联动」→ 闪白时 Emission = baseColor * flashAmount * boost
倒推5：需要「多角色独立」→ MaterialPropertyBlock per-renderer，不影响共享材质
```

#### 知识点拆解（倒推树）

```
角色受击闪白
├── Shader 核心
│   ├── lerp() 混合基础色与闪白色
│   ├── Mask 权重（顶点色 / 纹理通道 / shader_feature 分支）
│   ├── Emission 联动（闪白 = 自发光 = 配合 Bloom）
│   └── HDR 颜色（_FlashColor > 1.0 配合 Bloom 更亮）
├── URP 集成
│   ├── Shader Graph 搭建（Vertex Color → Lerp 节点）
│   ├── 手写 HLSL（适合需要 multi_compile 分支控制）
│   └── SRP Batcher 兼容（CBUFFER 包裹所有参数）
├── C# 驱动
│   ├── MaterialPropertyBlock（避免 new Material）
│   ├── 受击事件订阅（动画事件 / 碰撞回调 / Gameplay 信号）
│   └── 衰减曲线选择（Ease-out / Spring / Exponential）
└── 进阶效果
    ├── 受击方向指示（法线 · 击中方向 → 局部闪白）
    ├── 扰动叠加（闪白 + UV 抖动 = 「震感」）
    └── 多段受击叠加（不同颜色闪烁区分伤害类型）
```

#### 代码实现

**手写 HLSL（URP 兼容，支持材质区分）：**

```hlsl
Shader "Custom/HitFlash"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _MaskMap ("Material Mask (R=Skin, G=Metal, B=Cloth)", 2D) = "white" {}
        _FlashColor ("Flash Color", Color) = (1, 1, 1, 1)
        _FlashAmount ("Flash Amount", Range(0, 1)) = 0
        _MetalBoost ("Metal Flash Boost", Range(1, 5)) = 2.0
        _SkinReduce ("Skin Flash Reduce", Range(0, 1)) = 0.7
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
                float4 _FlashColor;
                float _FlashAmount;
                float _MetalBoost;
                float _SkinReduce;
            CBUFFER_END

            TEXTURE2D(_BaseMap);  SAMPLER(sampler_BaseMap);
            TEXTURE2D(_MaskMap);  SAMPLER(sampler_MaskMap);

            struct Attributes {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
                float4 color : COLOR; // 顶点色 R 作为区域 mask
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float4 vertexColor : TEXCOORD1;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                OUT.vertexColor = IN.color;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
                half3 mask = SAMPLE_TEXTURE2D(_MaskMap, sampler_MaskMap, IN.uv).rgb;

                // 计算材质区域权重：金属增强、皮肤减弱
                half materialWeight = 1.0;
                materialWeight = lerp(materialWeight, _SkinReduce, mask.r);  // 皮肤
                materialWeight = lerp(materialWeight, _MetalBoost, mask.g);  // 金属

                // 顶点色 R 通道作为额外权重（可选，美术可刷顶点色控制）
                materialWeight *= lerp(1.0, IN.vertexColor.r, step(0.01, IN.vertexColor.r));

                // 闪白混合
                half finalFlash = _FlashAmount * materialWeight;
                half3 finalColor = lerp(baseColor.rgb, _FlashColor.rgb, finalFlash);

                // Emission 联动：闪白时拉高自发光，配合 Bloom
                finalColor += _FlashColor.rgb * finalFlash * finalFlash * 2.0;

                return half4(finalColor, baseColor.a);
            }
            ENDHLSL
        }
    }
}
```

**C# 受击闪烁控制器：**

```csharp
using System.Collections;
using UnityEngine;

public class HitFlashController : MonoBehaviour
{
    [SerializeField] private Renderer[] renderers;
    [SerializeField] private float flashDuration = 0.15f;
    [SerializeField] private AnimationCurve flashCurve = AnimationCurve.EaseInOut(0, 1, 1, 0);

    private MaterialPropertyBlock _mpb;
    private static readonly int FlashAmountID = Shader.PropertyToID("_FlashAmount");
    private Coroutine _flashRoutine;

    void Awake() => _mpb = new MaterialPropertyBlock();

    /// <summary>
    /// 外部调用：受击事件触发闪白
    /// </summary>
    public void TriggerFlash()
    {
        if (_flashRoutine != null) StopCoroutine(_flashRoutine);
        _flashRoutine = StartCoroutine(FlashRoutine());
    }

    /// <summary>
    /// 支持叠加：多段受击时刷新但不打断当前衰减
    /// </summary>
    public void TriggerFlash(float intensity)
    {
        if (_flashRoutine != null) StopCoroutine(_flashRoutine);
        _flashRoutine = StartCoroutine(FlashRoutine(intensity));
    }

    IEnumerator FlashRoutine(float startIntensity = 1f)
    {
        float elapsed = 0f;
        while (elapsed < flashDuration)
        {
            float t = elapsed / flashDuration;
            // Ease-out 曲线：快速亮起后缓缓消退
            float flashValue = startIntensity * flashCurve.Evaluate(t);

            foreach (var r in renderers)
            {
                r.GetPropertyBlock(_mpb);
                _mpb.SetFloat(FlashAmountID, flashValue);
                r.SetPropertyBlock(_mpb);
            }
            elapsed += Time.deltaTime;
            yield return null;
        }

        // 确保归零
        foreach (var r in renderers)
        {
            r.GetPropertyBlock(_mpb);
            _mpb.SetFloat(FlashAmountID, 0f);
            r.SetPropertyBlock(_mpb);
        }
    }
}
```

**Shader Graph 搭建路径（无代码方案）：**

```
[Sample BaseMap] ─────────────────┐
                                   ├──→ [Lerp] ──→ [Color] ──→ Fragment Base Color
[FlashAmount] × [MaterialWeight] ──┘
                                   ──→ [Multiply] ──→ [Emission] ──→ Fragment Emission
[Sample MaskMap G] ──→ [MetalBoost]              ↑
[FlashAmount] ──────────────────────────────────┘
```

**对比表格：闪白实现方案**

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 全色 lerp 闪白 | 最简单，1 行代码 | 无材质区分，效果单调 | 2D 游戏 / 低精度需求 |
| Mask 纹理区分 | 材质表现丰富 | 需要额外 Mask 贴图 | 3D 角色游戏（推荐） |
| 顶点色区分 | 不增加纹理采样 | 需要美术刷顶点色 | 卡通渲染 / 低面数模型 |
| 后处理全屏闪白 | 统一管理 | 无法区分角色 | UI / 受伤红屏 |
| 替换材质（闪白材质） | 可做复杂效果 | 材质切换卡顿 / DC+1 | 不推荐 |

### ⚡ 实战经验

- **衰减曲线是灵魂**：线性恢复看起来「廉价」，用 `AnimationCurve` 或 `pow(t, 2)` 的 ease-out 衰减会大幅提升打击感
- **闪白时间窗口**：0.1s~0.2s 最佳，超过 0.3s 就像「发光怪」不像「被打」
- **Bloom 联动**：闪白时 Emission 拉高 + 后处理 Bloom = 辉光效果，比纯颜色叠加好看一个档次
- **多段受击**：用「打断重启」策略（新触发直接 StopCoroutine + 重开），避免闪白值叠加到 >1
- **Boss 特例**：Boss 可以用红色/紫色闪白区分普通受击和「破防」事件
- **性能要点**：`MaterialPropertyBlock` 是必须的，不然每次 `new Material` 直接内存泄漏

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 闪白看起来很「扁」，没有层次 | 不懂 Mask / 材质区分 | 学 Shader 中多通道 Mask 的运用 |
| 闪白恢复很生硬 | 衰减曲线选择 | 复习 AnimationCurve / 缓动函数 |
| 多角色闪白互相影响 | MaterialPropertyBlock 原理 | 学 URP 下 per-instance 数据传递 |
| 闪白配合 Bloom 不好看 | HDR 颜色 / Emission | 学 URP Post Processing 的 Bloom 阈值 |
| Boss 闪白和普通怪一样 | 事件系统 / 参数化设计 | 学游戏伤害系统中 event-driven 设计 |

### 🔗 相关问题

- [角色溶解消失](dissolve-effect)：受击死亡 → 闪白 → 溶解的完整链条怎么做？
- [径向模糊打击感](radial-blur-hit-effect)：角色闪白 + 镜头径向模糊 = 完整打击反馈
- [能量护盾效果](energy-shield-effect)：护盾受击的局部闪白怎么实现（不是全身）？
