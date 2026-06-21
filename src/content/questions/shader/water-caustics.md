---
title: "如何用Shader实现逼真的水面焦散效果？"
category: "shader"
level: 2
tags: ["Shader", "焦散", "水面", "UV动画", "程序化纹理"]
hint: "焦散的本质是光线折射汇聚形成的明暗纹理，用程序化噪声+UV偏移模拟"
related: ["shader/dissolve-effect", "rendering/sss-skin-rendering"]
---

## 参考答案

### 🎬 场景描述

面试官给你一段参考视频——阳光下泳池底部那种波光粼粼的光斑在不停流动。然后说：

> "我们需要在手游项目中实现水面焦散效果。性能预算很紧，不能用 GrabPass，移动端要跑满60帧。你给我一个方案。"

### ✅ 核心要点

1. **焦散本质**：光线经水面折射后，在池底形成明暗交错的光斑纹理——不是贴图动画，是光线汇聚/发散的数学模拟
2. **程序化生成优先**：移动端不能用 GrabPass / 后处理，必须用程序化纹理（Procedural）在 Fragment 阶段直接计算
3. **噪声叠加是灵魂**：单层正弦波太假，至少两层不同频率的噪声做干涉叠加才接近真实
4. **UV 动画驱动流动**：时间变量驱动 UV 偏移，两层噪声以不同方向+速度滚动产生有机感
5. **性能红线**：控制数学运算次数（Sin/Cos/Pow 很贵），移动端 Fragment 指令预算 ≤ 30 ALU

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：池底有流动的光斑纹理
     ↓ 倒推
光斑 = 明暗变化的亮度图
     ↓ 倒推
亮度图 = 两层波纹叠加后的结果
     ↓ 倒推
波纹 = 用 UV 坐标做正弦/噪声计算
     ↓ 倒推
流动 = Time 驱动 UV 滚动（两层不同方向+速度）
     ↓ 倒推
贴到池底 = 池底 Mesh 的世界坐标 → UV → 采样焦散
```

#### 知识点拆解（倒推树）

```
水面焦散效果
├── 数学基础
│   ├── 正弦波叠加（两层不同频率/方向）
│   ├── Power 函数控制锐度（焦散光斑的"锐利感"）
│   └── UV 坐标系与世界坐标的映射
├── Shader 实现
│   ├── Vertex Shader：传递世界坐标到 Fragment
│   ├── Fragment Shader：程序化计算焦散亮度
│   └── 时间变量驱动动画（_Time.y 或自定义）
├── 渲染管线
│   ├── 池底 Mesh 的材质需要支持世界坐标
│   ├── 半透明混合：焦散颜色 × 焦散强度 叠加到底色
│   └── 深度考量：浅水区强、深水区弱（需要 Depth 采样或顶点高度衰减）
└── 性能约束
    ├── 移动端：避免 tex2D 采样（纯数学计算）
    ├── 指令优化：用 mad 指令合并乘加
    └── LOD 策略：远处降低噪声层数
```

#### 代码实现

**核心 Shader（HLSL / URP 兼容）：**

```hlsl
Shader "TA/CausticsMobile"
{
    Properties
    {
        _BaseColor   ("池底基色", Color) = (0.15, 0.35, 0.5, 1)
        _CausticColor("焦散颜色", Color) = (0.8, 0.95, 1.0, 1)
        _CausticIntensity ("焦散强度", Range(0, 3)) = 1.2
        _CausticSharpness ("焦散锐度", Range(1, 8)) = 3.0
        _Speed1 ("噪声层1速度", Float) = 0.8
        _Speed2 ("噪声层2速度", Float) = 1.1
        _Scale  ("焦散缩放", Range(0.1, 5)) = 1.0
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        LOD 100

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float4 _CausticColor;
                float  _CausticIntensity;
                float  _CausticSharpness;
                float  _Speed1;
                float  _Speed2;
                float  _Scale;
            CBUFFER_END

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 worldPos   : TEXCOORD1;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = IN.uv;
                OUT.worldPos = TransformObjectToWorld(IN.positionOS.xyz);
                return OUT;
            }

            // ─── 核心焦散计算（程序化，无纹理采样） ───
            float CausticWeight(float2 uv, float time, float scale, float speed)
            {
                // 按缩放拉伸 UV
                float2 p = uv * scale;

                // 两层正弦波，不同方向不同速度 → 干涉叠加
                float2 dir1 = float2(1.0, 0.6);
                float2 dir2 = float2(-0.7, 1.0);

                float w1 = sin(dot(p, normalize(dir1)) + time * speed);
                float w2 = sin(dot(p, normalize(dir2)) + time * speed * 1.3);

                // 两层叠加后取绝对值 → 形成尖锐光斑
                float interference = abs(w1 + w2);

                // Power 提高对比度（锐化光斑边缘）
                return pow(interference, _CausticSharpness);
            }

            half4 frag(Varyings IN) : SV_Target
            {
                float t = _Time.y;
                float scale = _Scale;

                // 两层焦散叠加（不同频率、不同速度）
                float c1 = CausticWeight(IN.worldPos.xz, t, scale * 1.0, _Speed1);
                float c2 = CausticWeight(IN.worldPos.xz, t, scale * 2.1, _Speed2);

                // 叠加并限幅
                float caustic = saturate(c1 * 0.6 + c2 * 0.4);
                caustic *= _CausticIntensity;

                // 混合：底色 + 焦散亮部
                float3 finalColor = _BaseColor.rgb + _CausticColor.rgb * caustic;

                return half4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**焦散层数 vs 性能 vs 效果对比：**

| 方案 | 噪声层数 | ALU 估算 | 效果 | 适用场景 |
|------|---------|----------|------|---------|
| 极简版 | 1层 | ~8 | 勉强像水波纹 | 低端机（Android 4核） |
| **推荐版** | **2层** | **~18** | **光斑清晰自然** | **主流手游标准** |
| 高清版 | 3层叠加 | ~28 | 非常细腻 | 高端机/PC |
| 超高清版 | 3层+纹理扰动 | ~35+ | 电影级 | PC/PS5 |

#### 深度衰减（加分项）

面试官可能追问深水区怎么办：

```hlsl
// 在 frag 中加入深度衰减
float depth = ComputeSceneDepth(IN.positionCS);  // 需要深度纹理
float depthFade = saturate(1.0 - depth * _DepthFalloff);
caustic *= depthFade;  // 深处焦散减弱
```

### ⚡ 实战经验

- **不要用 RenderTexture + GrabPass 做焦散**：移动端 GrabPass 是性能杀手，中途带宽翻倍。纯数学计算反而更快
- **UV 方向别只用一个方向**：两层波纹必须不同方向（如 30° 和 -70°），否则叠加后看起来像单向条纹
- **Power 值是视觉灵魂**：Power=1 时像云雾，Power=3~5 才有焦散光斑的"锐利感"。让美术调这个值
- **池底 Mesh 需要足够分段**：如果池底只有两个三角形，Vertex Shader 传过去的世界坐标插值后精度不够，焦散会变形

### 🎯 能力体检清单

| 卡住的环节 | 说明你缺失的知识点 | 补习建议 |
|-----------|-------------------|---------|
| 不知道怎么产生光斑纹理 | 不理解程序化噪声/正弦波叠加 | 学习 Shader 中的程序化纹理生成（Value Noise、Perlin Noise） |
| UV 不流动 | 忘记时间变量驱动 | 复习 `_Time` / `UNITY_ZCBUFFER_TIME` 的使用 |
| 焦散看起来像条纹 | 两层波方向太接近 | 理解波叠加干涉原理，方向至少差 60° |
| 性能不达标 | Fragment 指令过多 | 学习移动端 Shader 优化（ALU 预算、mad 合并） |
| 深水区效果不对 | 缺少深度衰减 | 补习深度纹理采样和线性化 |

### 🔗 相关问题

- 如果水面本身也需要 Shader（波纹+反射+折射），整个水面渲染方案怎么设计？
- 延迟渲染下焦散效果怎么实现？（提示：G-Buffer 阶段写入还是 Lighting 阶段？）
- 如何用 Houdini 烘焙焦散动画到纹理序列，在超低端机上播放？
