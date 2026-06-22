---
title: "能量护盾效果：如何用 Shader 实现受击发光的六边形护盾？"
category: "shader"
level: 3
tags: ["Energy Shield", "六边形纹理", "Fresnel", "受击反馈", "URP", "Shader"]
hint: "核心是 Voronoi/六边形网格 + Fresnel 边缘 + 受击位置的脉冲扩散——护盾不只是透明球，要有空间结构感"
related: ["shader/dissolve-effect", "rendering/custom-post-processing-urp", "shader/npr-outline-cartoon"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们游戏角色需要一个能量护盾效果——护盾是半透明的，表面有六边形网格纹理。当被击中时，受击点要产生一圈向外扩散的能量波纹，波纹消退时护盾局部闪烁。整体要有科幻感的 Fresnel 边缘光。在 URP 下实现，给我完整方案。」

这是叠纸、鹰角、米哈游等二次元/科幻风格项目的高频面试题，考察 Shader 综合能力：程序化纹理、顶点/片元交互、动态参数传入、性能控制。

### ✅ 核心要点

1. **程序化六边形网格**：用 Voronoi 或数学方法生成六边形纹理，不需要美术画
2. **Fresnel 边缘光**：护盾球边缘亮度增强，营造能量场包裹感
3. **受击脉冲扩散**：世界空间受击坐标 → 距离场计算 → 波纹动画
4. **护盾血量映射**：整体透明度/脉冲频率与护盾值挂钩
5. **性能控制**：护盾是半透明叠加，Overdraw 是主要风险

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：半透明护盾球 → 六边形网格 → Fresnel 边缘 → 受击波纹扩散 → 血量影响整体表现
                ↑
倒推1：「半透明球」→ 独立 Mesh（球体/凸包）+ Alpha Blend
倒推2：「六边形网格」→ 程序化 UV 采样或运行时生成
倒推3：「Fresnel」→ viewDir · normal 反比
倒推4：「受击波纹」→ 受击点传入 Shader，distance(vertex, hitPoint) 做距离场波纹
倒推5：「血量映射」→ C# 传入 _ShieldHealth，影响整体 Alpha 和脉冲频率
倒推6：「多次受击叠加」→ 用 RT (Render Texture) 累积受击信息，采样回读
```

#### 知识点拆解（倒推树）

```
能量护盾效果
├── 六边形网格生成
│   ├── Voronoi 算法（最近点距离 = 网格线）
│   ├── 数学六边形（hexagonal tiling 公式）
│   ├── 网格线宽度控制（边缘 smoothstep）
│   └── 网格脉冲呼吸（时间驱动 sin 波）
├── Fresnel 边缘光
│   ├── viewDir 计算（WorldSpace View Dir）
│   ├── pow(1 - dot(N, V), power) 公式
│   ├── 边缘颜色与强度参数化
│   └── 与基础色 blend（加法/乘法选择）
├── 受击波纹系统
│   ├── 世界空间受击坐标传入（MaterialPropertyBlock）
│   ├── 距离场：distance(worldPos, hitWorldPos)
│   ├── 波纹动画：sin(dist - time * speed) * mask
│   ├── 多点受击：数组传入 or Render Texture 累积
│   └── 波纹衰减：exp(-dist * decay) 或 1/(1+dist²)
├── 护盾血量系统
│   ├── _ShieldHealth (0~1) 影响整体 Alpha
│   ├── 低血量时网格闪烁加强（高频脉冲）
│   ├── 低血量时边缘色变红（lerp(normalColor, dangerColor)）
│   └── 护盾破碎效果（配合 dissolve 思路）
├── URP 集成
│   ├── 透明队列排序（Queue = Transparent）
│   ├── SRP Batcher 兼容（CBUFFER）
│   ├── 渲染顺序：护盾在不透明物体之后
│   └── 深度写入控制（ZWrite Off for blend）
└── 性能优化
    ├── Overdraw 控制（护盾面积不宜过大）
    ├── 移动端简化（降精度、去多受击点）
    ├── 烘焙六边形纹理 vs 运行时计算（移动端烘焙）
    └── GPU Instancing + per-instance 参数
```

#### 代码实现

**核心 Shader（URP HLSL）：**

```hlsl
Shader "Custom/EnergyShield"
{
    Properties
    {
        _BaseColor ("Shield Base Color", Color) = (0.3, 0.7, 1.0, 0.3)
        _EdgeColor ("Edge Color", Color) = (0.5, 0.9, 1.0, 1.0)
        _HexColor ("Hex Grid Color", Color) = (0.6, 1.0, 1.0, 0.8)
        _FresnelPower ("Fresnel Power", Float) = 3.0
        _FresnelIntensity ("Fresnel Intensity", Float) = 1.5
        _HexScale ("Hex Grid Scale", Float) = 10.0
        _HexLineWeight ("Hex Line Weight", Range(0.001, 0.1)) = 0.02
        _PulseSpeed ("Pulse Speed", Float) = 2.0
        _HitColor ("Hit Ripple Color", Color) = (1.0, 1.0, 1.0, 1.0)
        _HitRadius ("Hit Ripple Radius", Float) = 0.3
        _HitSpeed ("Hit Ripple Speed", Float) = 3.0
        _HitDecay ("Hit Decay", Float) = 2.0
        _ShieldHealth ("Shield Health", Range(0, 1)) = 1.0
    }
    SubShader
    {
        Tags {
            "RenderType"="Transparent"
            "Queue"="Transparent"
            "RenderPipeline"="UniversalPipeline"
        }
        Pass
        {
            Blend SrcAlpha OneMinusSrcAlpha
            ZWrite Off
            Cull Front  // 渲染内表面，避免外表面遮挡

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float4 _EdgeColor;
                float4 _HexColor;
                float4 _HitColor;
                float _FresnelPower;
                float _FresnelIntensity;
                float _HexScale;
                float _HexLineWeight;
                float _PulseSpeed;
                float _HitRadius;
                float _HitSpeed;
                float _HitDecay;
                float _ShieldHealth;
            CBUFFER_END

            // 支持 4 个同时受击点
            float4 _HitPoints[4]; // xyz = world pos, w = elapsed time (0=inactive)
            float _TimeSinceHit;

            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float3 worldPos : TEXCOORD0;
                float3 normalWS : TEXCOORD1;
                float3 viewDirWS : TEXCOORD2;
                float2 uv : TEXCOORD3;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            // 程序化六边形网格
            // 参考: https://www.shadertoy.com/view/MdVyDw
            float hexDist(float2 p)
            {
                p = abs(p);
                float c = dot(p, normalize(float2(1.0, 1.732)));
                c = max(c, p.x);
                return c;
            }

            float4 hexCoords(float2 uv)
            {
                float2 r = float2(1.0, 1.732);
                float2 h = r * 0.5;
                float2 a = fmod(uv, r) - h;
                float2 b = fmod(uv - h, r) - h;
                float2 gv = dot(a,a) < dot(b,b) ? a : b;
                float x = atan2(gv.x, gv.y);
                float y = 0.5 - hexDist(gv);
                float4 col = float4(floor(uv - gv), x, y); // hex id + local coords
                return col;
            }

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);

                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.worldPos = TransformObjectToWorld(IN.positionOS.xyz);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                OUT.viewDirWS = GetWorldSpaceNormalizeViewDir(OUT.worldPos);
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);

                // === Fresnel 边缘光 ===
                float NdotV = saturate(dot(normalize(IN.normalWS), normalize(IN.viewDirWS)));
                float fresnel = pow(1.0 - NdotV, _FresnelPower) * _FresnelIntensity;

                // === 六边形网格 ===
                float2 hexUV = IN.uv * _HexScale;
                float4 hc = hexCoords(hexUV);
                // 网格线：y 接近 0 表示靠近六边形边缘
                float hexLine = smoothstep(0.0, _HexLineWeight, hc.w);
                hexLine = 1.0 - hexLine; // 反转：边缘=1

                // 呼吸脉冲
                float pulse = sin(_Time.y * _PulseSpeed + hc.x * 0.7 + hc.y * 0.5) * 0.5 + 0.5;
                hexLine *= 0.5 + pulse * 0.5;

                // === 受击波纹 ===
                float hitRipple = 0.0;
                for (int i = 0; i < 4; i++)
                {
                    if (_HitPoints[i].w > 0.0)
                    {
                        float dist = distance(IN.worldPos, _HitPoints[i].xyz);
                        float elapsed = _HitPoints[i].w;
                        float waveRadius = elapsed * _HitSpeed;
                        float waveWidth = _HitRadius;

                        // 波纹环带
                        float ring = 1.0 - saturate(abs(dist - waveRadius) / waveWidth);
                        // 时间衰减
                        float decay = exp(-elapsed * _HitDecay);
                        hitRipple += ring * decay;
                    }
                }
                hitRipple = saturate(hitRipple);

                // === 血量影响 ===
                float healthFactor = _ShieldHealth;
                // 低血量时增强脉冲
                float dangerPulse = (1.0 - healthFactor) * (sin(_Time.y * 8.0) * 0.5 + 0.5);
                // 低血量偏红
                float3 dangerColor = float3(1.0, 0.2, 0.1);

                // === 合成 ===
                float3 baseCol = _BaseColor.rgb;
                float3 edgeCol = _EdgeColor.rgb * fresnel;
                float3 hexCol = _HexColor.rgb * hexLine;
                float3 hitCol = _HitColor.rgb * hitRipple * 2.0;

                float3 finalCol = baseCol + edgeCol + hexCol + hitCol;

                // 低血量混入危险色
                finalCol = lerp(finalCol, finalCol * 0.5 + dangerColor * dangerPulse * 0.5,
                                1.0 - healthFactor);

                // 透明度
                float alpha = _BaseColor.a + fresnel * 0.5 + hexLine * _HexColor.a + hitRipple * 0.5;
                alpha *= saturate(healthFactor * 0.7 + 0.3); // 低血量变透明

                return half4(finalCol, saturate(alpha));
            }
            ENDHLSL
        }
    }
}
```

**C# 受击管理器：**

```csharp
using UnityEngine;

public class ShieldHitManager : MonoBehaviour
{
    [SerializeField] private Renderer shieldRenderer;
    [SerializeField] private float rippleLifetime = 2f;

    private MaterialPropertyBlock _mpb;
    private Vector4[] _hitPoints = new Vector4[4]; // xyz=pos, w=elapsed
    private float[] _hitTimers = new float[4];
    private int _nextSlot = 0;

    private static readonly int HitPointsID = Shader.PropertyToID("_HitPoints");
    private static readonly int ShieldHealthID = Shader.PropertyToID("_ShieldHealth");

    void Awake() => _mpb = new MaterialPropertyBlock();

    void Update()
    {
        bool dirty = false;
        for (int i = 0; i < 4; i++)
        {
            if (_hitTimers[i] > 0)
            {
                _hitTimers[i] -= Time.deltaTime;
                _hitPoints[i].w = _hitTimers[i] > 0 ? rippleLifetime - _hitTimers[i] : 0;
                dirty = true;
            }
        }
        if (dirty)
        {
            shieldRenderer.GetPropertyBlock(_mpb);
            _mpb.SetVectorArray(HitPointsID, _hitPoints);
            shieldRenderer.SetPropertyBlock(_mpb);
        }
    }

    /// <summary>
    /// 外部调用：世界空间受击位置
    /// </summary>
    public void RegisterHit(Vector3 worldHitPos)
    {
        _hitPoints[_nextSlot] = new Vector4(worldHitPos.x, worldHitPos.y, worldHitPos.z, 0.01f);
        _hitTimers[_nextSlot] = rippleLifetime;
        _nextSlot = (_nextSlot + 1) % 4;
    }

    /// <summary>
    /// 设置护盾血量 (0~1)
    /// </summary>
    public void SetHealth(float health)
    {
        shieldRenderer.GetPropertyBlock(_mpb);
        _mpb.SetFloat(ShieldHealthID, Mathf.Clamp01(health));
        shieldRenderer.SetPropertyBlock(_mpb);
    }
}
```

**方案对比：多受击点管理**

| 方案 | 受击数上限 | 性能 | 实现复杂度 | 适用场景 |
|------|-----------|------|-----------|----------|
| Shader 数组（本文） | 4~8 | 低 | 中 | 大多数游戏 |
| Render Texture 累积 | 无限 | 中（需额外 Pass） | 高 | 高端科幻项目 |
| GrabScreen + 后处理 | 无限 | 高 | 很高 | 3A 单机 |
| 顶点色存储受击 | 顶点数 | 低 | 中 | 低面数护盾 |

### ⚡ 实战经验

- **Cull Front 还是 Cull Back**：渲染护盾内表面（Cull Front）可以避免护盾被外部物体遮挡时消失，同时内外都能看到效果
- **六边形纹理 vs 程序化**：移动端建议烘焙六边形到纹理，减少 ALU 开销；PC/主机端程序化更灵活
- **受击坐标空间**：传入世界空间坐标最直观，但护盾移动时波纹会「跟随」——如果需要世界固定，考虑用护盾局部空间
- **多受击点循环**：Shader 中 for 循环次数必须是编译期常量，`[unroll]` 确保展开
- **深度写入**：ZWrite Off 避免护盾挡住角色面部——但也会导致护盾之间的排序问题，需要 Sort Layer 辅助

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么做六边形网格 | Voronoi / hexagonal tiling | 学程序化纹理生成（ShaderToy） |
| Fresnel 做出来没效果 | 法线空间 / viewDir 计算 | 复习 World/View/Tangent 空间变换 |
| 多受击点无法同时显示 | Shader 数组 / 常量限制 | 学 GPU 常量缓冲区限制与 RT 方案 |
| 护盾遮挡角色面部 | 深度测试 / 渲染队列 | 学 URP 透明排序与 ZWrite |
| 性能掉帧 | Overdraw / ALU 开销 | 用 Frame Debugger / RenderDoc 分析 |
| 护盾移动时波纹跟随 | 坐标空间选择 | 理解世界空间 vs 局部空间 |

### 🔗 相关问题

- 护盾破碎时如何配合粒子效果实现碎片飞散？（提示：结合 dissolve + 粒子爆破）
- 如何用 Render Texture 实现无限受击点？（提示：在 RT 上画白点累积，Shader 采样回读）
- 多层护盾（如三层不同颜色）如何管理渲染顺序？（提示：Stencil Buffer 或 Pass 分离）
