---
title: "角色头发渲染：如何实现飘逸的各向异性高光与头发着色？"
category: "shader"
level: 3
tags: ["头发渲染", "各向异性BRDF", "Kajiya-Kay", "Marschner", "护发精油高光", "Shift Map", "卡通渲染"]
hint: "核心是理解头发高光「沿发丝方向拉伸」的各向异性特性，掌握 Kajiya-Kay 或 Marschner 模型，配合 Shift Map 做出双高光带"
related: ["shader/npr-outline-cartoon", "rendering/sss-skin-rendering", "shader/dissolve-effect"]
---

## 参考答案

### 🎬 场景描述

面试官打开一段游戏角色演示视频，指着角色的头发说：

> "我们项目角色头发目前用的是普通 Blinn-Phong 高光，看起来像塑料。美术反馈说想要那种'护发精油'一样的丝滑高光，沿着发丝方向流动。你来做一个头发 Shader 方案，要求：
> 1. 高光要沿着发丝方向拉伸，不是圆形光斑
> 2. 要有两条高光带（一条主高光偏白色，一条副高光可以带点发色）
> 3. 能通过 Shift Map 控制高光偏移，避免高光完全对称显得假
> 4. 移动端要能跑，不能用太复杂的模型"

### ✅ 核心要点

1. **头发高光本质是各向异性反射**：发丝近似圆柱体，高光沿发丝方向拉伸而非均匀扩散
2. **Kajiya-Kay 模型是移动端首选**：用切线方向计算各向异性高光，性能友好
3. **双高光带 = 主高光 + 副高光**：不同 Offset 和 Intensity 组合出层次感
4. **Shift Map 控制高光流向**：打破对称性，让每缕头发高光位置不同
5. **AO / Rim / 边缘透明**：发根暗、发梢透、边缘逆光，三者缺一不可

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标效果：丝滑流动的头发高光
  ↓ 为什么现在的 Blinn-Phong 不行？
Blinn-Phong 产生圆形高光斑，因为 N·H 是各向同性的
  ↓ 头发高光为什么要拉伸？
发丝是细长圆柱体，反射光沿发丝方向集中分布
  ↓ 如何实现各向异性？
用发丝切线方向 T 替代法线 N 来计算高光
  ↓ Kajiya-Kay 怎么算？
sin(T, H) 的平方 → 高光沿切线方向拉伸
  ↓ 为什么要双高光？
真实头发有角质层鳞片结构 → 产生主反射 + 次级漫反射散射
  ↓ Shift Map 怎么用？
每缕头发给不同偏移量 → 打破均匀性 → 看起来自然
```

#### 知识点拆解（倒推树）

```
头发着色
├── 各向异性 BRDF
│   ├── Kajiya-Kay 模型（切线·半角向量）
│   ├── Marschner 模型（M/R/TRT 三条路径，精度高但贵）
│   └── 经验近似（GGX-Anisotropic 改造）
├── 切线系统
│   ├── 切线从 UV 方向推导（通常 V 方向 = 发丝方向）
│   ├── Shift Map（R 通道存偏移量）
│   └── Flow Map（控制发丝流向，适合非平行头发）
├── 双高光带
│   ├── Primary Specular：窄、强、偏白色
│   ├── Secondary Specular：宽、弱、带发色染色
│   └── 不同 Noise/Shift 打散
├── 基础着色
│   ├── Diffuse：Wrap Lighting 或 Half-Lambert（避免发根太黑）
│   ├── 发根 AO：Vertex Color G 或贴图控制
│   └── 边缘逆光：Fresnel Rim Light
├── 透明处理
│   ├── Alpha Test（Cutout）：性能好但边缘硬
│   ├── Alpha Blend：边缘柔和但有排序问题
│   └── OIT / Stencil 投影（高级方案）
└── 性能策略
    ├── 移动端：Kajiya-Kay + 1 张 Shift Map
    └── 高端机：Marschner 简化版 + 多 Pass
```

#### 代码实现

**Kajiya-Kay 各向异性高光核心（HLSL / URP）：**

```hlsl
// 头发各向异性高光 - Kajiya-Kay 模型
// 输入：世界空间切线 T、视线方向 V、光源方向 L
float3 HairSpecularKajiyaKay(float3 T, float3 L, float3 V,
                              float shift, float exponent, float strength)
{
    // 1. 应用 Shift 偏移切线
    // shift 来自 Shift Map，范围 [-0.5, 0.5]
    float3 shiftedT = normalize(T + float3(shift, 0, shift));

    // 2. 计算半角向量
    float3 H = normalize(L + V);

    // 3. Kajiya-Kay: dot(T, H) → sin(θ) → sin²(θ)
    //    各向异性关键：用切线而非法线
    float dotTH = dot(shiftedT, H);
    float sinTH = sqrt(max(0, 1.0 - dotTH * dotTH));

    // 4. 高光分布：pow(sinθ, exponent)
    //    exponent 越大高光越锐利（通常 8~64）
    float specular = pow(sinTH, exponent) * strength;

    return specular;
}

// ---- 完整 Fragment 片段 ----
float4 Frag(Varyings IN) : SV_Target
{
    float3 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv).rgb;
    float alpha  = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv).a;

    // Alpha Test
    clip(alpha - _Cutoff);

    // Shift Map：R 通道 = 主高光偏移，G 通道 = 副高光偏移
    float2 shiftMap = SAMPLE_TEXTURE2D(_ShiftMap, sampler_ShiftMap, IN.uv).rg;
    shiftMap = (shiftMap - 0.5) * _ShiftAmount; // [-_ShiftAmount, _ShiftAmount]

    // 发丝切线：从 UV 的 V 方向推导，转换到世界空间
    float3 tangentWS = normalize(IN.tangentWS);
    // 如果用 Flow Map，这里用 Flow 方向替换 tangentWS

    // 光照参数
    float3 L = normalize(_MainLightPosition.xyz);
    float3 V = normalize(GetWorldSpaceViewDir(IN.positionWS));
    float3 N = normalize(IN.normalWS);

    // 基础漫反射 - Wrap Lighting 让暗部不死黑
    float NdotL = dot(N, L);
    float diffuse = max(0, (NdotL + _WrapFactor) / (1 + _WrapFactor)) * _DiffuseStrength;
    float3 hairColor = albedo * diffuse * _MainLightColor.rgb;

    // 双高光带
    // 主高光：窄、强、偏白色
    float3 specPrimary = HairSpecularKajiyaKay(
        tangentWS, L, V,
        shiftMap.r + _PrimaryShift,   // 偏移
        _PrimaryExponent,              // 锐度（通常 16~32）
        _PrimaryStrength               // 强度
    ) * _SpecularColor1;  // 偏白色

    // 副高光：宽、弱、带发色
    float3 specSecondary = HairSpecularKajiyaKay(
        tangentWS, L, V,
        shiftMap.g + _SecondaryShift, // 与主高光错开
        _SecondaryExponent,            // 更钝（通常 4~8）
        _SecondaryStrength             // 更弱
    ) * _SpecularColor2 * albedo;     // 染上发色

    // 边缘光 Fresnel
    float fresnel = pow(1.0 - max(0, dot(N, V)), 3) * _RimStrength;
    float3 rimLight = fresnel * _RimColor;

    // 发根 AO（Vertex Color G 通道或贴图）
    float rootAO = IN.color.g; // 0=发根 1=发梢
    hairColor *= lerp(0.3, 1.0, rootAO);

    // 合成
    float3 finalColor = hairColor + specPrimary + specSecondary + rimLight;

    return float4(finalColor, alpha);
}
```

**Shift Map 与 Flow Map 对比：**

| 方案 | 分辨率 | 作用 | 适用场景 |
|------|--------|------|----------|
| Shift Map | 256² ~ 512² | 控制高光偏移量 | 标准方案，通用 |
| Flow Map | 256² ~ 512² | 控制发丝流向向量 | 卷发、非平行发型 |
| DerivMap | 512² | 直接存切线导数 | 高精度需求 |
| Vertex Flow | 逐顶点 | 低频流向 | 性能极限场景 |

**Kajiya-Kay vs Marschner 对比：**

| 维度 | Kajiya-Kay | Marschner |
|------|------------|-----------|
| 物理精度 | 近似（经验模型） | 高（基于毛发测量数据） |
| 高光条数 | 需手动做双高光 | 天然 M + TRT 两条 |
| 计算量 | 低（1 个 sin²） | 高（多次散射积分） |
| 移动端可行性 | ✅ 首选 | ❌ 通常不推荐 |
| 主机/PC | 可用 | ✅ 推荐 |
| 调参难度 | 简单 | 复杂（需理解 R, TT, TRT） |

### ⚡ 实战经验

1. **Shift Map 是灵魂**：没有 Shift Map 的头发高光会形成一条完美直线，美术一眼就觉得假。用噪点生成 Shift Map，打碎高光带即可提升一个档次
2. **副高光染色很重要**：副高光用发色染色后，整体头发质感从"塑料"变"毛发"，这是很多 TA 忽略的细节
3. **移动端别用 Alpha Blend**：头发面片数量多，半透明排序开销巨大。用 Alpha Test + MSAA 或 Stencil 投影方案更实际
4. **Dither 透明过渡**：角色近距离时 Alpha Test 边缘锯齿明显，用 Dither（抖动）做透明过渡可以大幅改善，且性能开销几乎为零

### 🎯 能力体检清单

| 卡住的环节 | 盲区在哪 | 补习建议 |
|------------|----------|----------|
| 不知道为什么要用切线替代法线 | 各向异性 vs 各向同性反射模型 | 先理解圆柱体光照模型，再看 Kajiya-Kay 论文 |
| 双高光带做不出来 | 不理解头发散射结构 | 研究 Marschner 论文的 M/R/TRT 路径 |
| 高光是一条死直线 | 没用 Shift Map / Noise | 在 Substance 中生成 Noise 贴图作为 Shift Map |
| 移动端帧率暴跌 | 可能用了 Alpha Blend 叠多层 | 改用 Alpha Test，减少 Overdraw |
| 不知道怎么从 UV 推导切线 | 切线空间基础知识薄弱 | 复习切线空间构建，理解 TBN 矩阵 |

### 🔗 相关问题

- [卡通渲染 Outline 方案](shader/npr-outline-cartoon.md) — 头发描边的特殊处理
- [SSS 皮肤渲染](rendering/sss-skin-rendering.md) — 与头发一起构成角色渲染核心
- 移动端头发透明排序有哪些方案？（延展：OIT、Stencil、Dither）
