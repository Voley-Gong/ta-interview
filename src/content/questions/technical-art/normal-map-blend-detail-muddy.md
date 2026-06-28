---
title: "角色皮肤细节丢失：法线贴图叠加后变'糊'怎么救？"
category: "technical-art"
level: 3
tags: ["法线贴图", "细节叠加", "Normal Blend", "UDN", "Reoriented Normal", "皮肤纹理"]
hint: "简单 slerp/overlay 混合两张法线会破坏正常向量——需要 Reoriented Normal Blending 或 UDN 方法保持光照正确"
related: ["technical-art/pbr-material-authoring", "technical-art/mobile-normal-map-compression", "shader/sss-skin-rendering"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们角色的皮肤，美术给了一张主法线贴图（大面积肌肉结构），又给了一张细节法线贴图（毛孔/细纹）。美术直接在 Shader Graph 里用 Lerp 叠加，结果近看全是'糊'的——细节法线完全被主法线淹没，或者叠加后光照出现奇怪的亮斑。你怎么解决这个问题？」

### ✅ 核心要点

1. **法线不能简单 Lerp**：两张法线贴图直接线性混合（Lerp）会破坏单位向量长度，导致光照计算异常
2. **细节法线需要'贴附'在主法线表面上**：细节法线的扰动方向应该相对于主法线所在的切面，而非世界空间
3. **主流方案**：UDN Blend（廉价近似）、Reoriented Normal Blending（RNV，精确重定向）、Partial Derivative Blending（偏导数混合）
4. **强度控制**：细节法线的贡献需要可调节的 `Detail Normal Strength`，且不能破坏整体光照方向

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：主法线提供大结构 + 细节法线提供毛孔纹理 → 光照正确、细节清晰
                ↑
倒推1：为什么 Lerp 会'糊'？
      → Lerp(N1, N2, t) 不保证结果向量是单位长度
      → 归一化后细节方向被主法线'拖偏'，细节信号被稀释
倒推2：正确的思路是什么？
      → 细节法线应该被'重定向'：把细节法线当作在主法线定义的切面上的扰动
      → 即：主法线 = '地表'，细节法线 = '地表上的凹凸'
倒推3：如何数学实现'重定向'？
      → 方法A (UDN)：将细节法线偏移到主法线方向（近似但便宜）
      → 方法B (RNV)：完整旋转矩阵重定向（精确但稍贵）
倒推4：如何在 Shader 中实现并暴露美术可调参数？
```

#### 知识点拆解（倒推树）

```
法线贴图混合
├── 为什么 Lerp 是错的
│   ├── 法线是方向向量，不是颜色——线性混合数学上无意义
│   ├── 混合后 |N| ≠ 1 → normalize 后细节损失
│   └── Overlay/Screen 等颜色混合模式更离谱——它们假设 RGB 是颜色
├── UDN Blend（Unreal Developer Network 方法）
│   ├── 公式：N_detail_reoriented = normalize(float3(N_detail.xy * intensity + N_main.xy, N_main.z))
│   ├── 原理：把细节法线的 XY 扰动量叠加到主法线的 XY 上，Z 保持主法线
│   ├── 优点：极其廉价（1次加法 + 1次归一化）
│   ├── 缺点：主法线接近水平时失真（z → 0 时方向退化）
│   └── 适用：手游、大多数情况"够用"
├── Reoriented Normal Blending (RNV)
│   ├── 论文：Colin Barré-Brisebois & Stephen Hill, SIGGRAPH 2012
│   ├── 核心思路：构建旋转矩阵，将细节法线从切线空间重定向到主法线定义的局部坐标系
│   ├── 公式（简化）：
│   │     t = float3(N_main.xy, N_main.z + 1.0)
│   │     u = float3(-N_detail.xy, N_detail.z)
│   │     N = t * dot(t, u) - u * t.z
│   │     return normalize(N)
│   ├── 优点：物理正确，任意角度都准确
│   ├── 缺点：比 UDN 多几次运算（移动端影响微小）
│   └── 适用：高品质角色、近距离特写
├── Partial Derivative Blending
│   ├── 将法线转换为高度偏导数（∂h/∂x, ∂h/∂y），在偏导数域相加
│   ├── 最数学正确的方法，但实现复杂
│   └── 实际项目极少使用（RNV 已足够好）
├── Detail Normal Strength 控制
│   ├── 不能直接缩放细节法线值——会破坏方向
│   ├── 正确做法：缩放细节法线的 XY 分量后再混合
│   │     N_detail_adjusted.xy *= strength;
│   │     N_detail_adjusted = normalize(N_detail_adjusted);
│   └── 或在 UDN 公式中直接控制叠加系数
├── 贴图采样优化
│   ├── 细节法线用平铺采样（tiling）→ 毛孔重复感用随机扰动打破
│   ├── 采样频率：细节法线需要更高 mipmap bias，否则远处 aliasing
│   └── 移动端：细节法线用 BC5 (2-channel) 压缩，省带宽
└── 与 SSS 皮肤的配合
    ├── 细节法线影响 specular 入射角度 → 直接影响皮肤油光感
    ├── 毛孔法线 + SSS scatter = 真实皮肤质感的关键组合
    └── 近距离特写时，毛孔法线对高光分布至关重要
```

#### 代码实现

**UDN 方法（最常用，移动端首选）：**

```hlsl
// UDN Normal Blend —— 廉价但实用
float3 BlendNormalUDN(float3 n1, float3 n2, float detailStrength)
{
    // n1 = 主法线（大结构），n2 = 细节法线（毛孔/细纹）
    // 细节法线的 XY 扰动叠加到主法线
    float3 blended;
    blended.xy = n1.xy + n2.xy * detailStrength;
    blended.z = n1.z;
    return normalize(blended);
}
```

**Reoriented Normal Blending（高品质角色首选）：**

```hlsl
// RNV Normal Blend —— 精确重定向
// Reference: "Blending in Detail" (Hill & Barré-Brisebois, SIGGRAPH 2012)
float3 BlendNormalRNV(float3 n1, float3 n2, float detailStrength)
{
    // n1 = 主法线（大结构），n2 = 细节法线（毛孔/细纹）

    // 调节细节法线强度：缩放 XY 分量
    n2.xy *= detailStrength;
    n2 = normalize(n2);

    // 构建重定向向量
    float3 t = float3(n1.xy, n1.z + 1.0);
    float3 u = float3(-n2.xy, n2.z);

    // 点积投影
    float3 result = t * dot(t, u) - u * t.z;
    return normalize(result);
}
```

**Unity URP Shader Graph 集成（Custom Function 节点）：**

```hlsl
// 在 Shader Graph 中用 Custom Function 节点接入
// 输入：MainNormal (Vector3), DetailNormal (Vector3), DetailStrength (Float)
// 输出：BlendedNormal (Vector3)

void BlendDetailNormal_float(
    float3 MainNormal,
    float3 DetailNormal,
    float DetailStrength,
    out float3 Out)
{
    // 采样后需要 UnpackNormal，这里假设已解包为 [-1,1] 范围
    float3 t = float3(MainNormal.xy, MainNormal.z + 1.0);
    float3 u = float3(-DetailNormal.xy * DetailStrength,
                       DetailNormal.z);
    float3 n = t * dot(t, u) - u * t.z;
    Out = normalize(n);
}
```

**完整 Shader 片段（URP HLSL）：**

```hlsl
// 角色皮肤法线混合片段
half3 SampleSkinNormal(float2 uv, float2 detailUV)
{
    // 主法线：大结构（肌肉、骨骼走势）
    half3 mainNormal = UnpackNormalScale(
        SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, uv), _BumpScale);

    // 细节法线：毛孔纹理（高频平铺）
    half3 detailNormal = UnpackNormalScale(
        SAMPLE_TEXTURE2D(_DetailBumpMap, sampler_DetailBumpMap, detailUV * _DetailTiling),
        _DetailBumpScale);

    // 使用 RNV 混合
    return BlendNormalRNV(mainNormal, detailNormal, _DetailNormalStrength);
}

// 在 fragment shader 中：
// half3 finalNormal = SampleSkinNormal(i.uv, i.uv);
// 然后 normal.z 应该被主法线主导（近距离观察仍能看到毛孔细节）
```

### ⚡ 实战经验

1. **UDN 在 99% 的手游场景够用**：性能差异在移动端几乎不可感知，但 UDN 的实现简单程度对美术和 TA 友好得多
2. **细节法线强度是美术手感参数**：通常 0.3-0.7 范围，超过 0.8 会出现"砂纸感"，低于 0.2 等于没叠
3. **细节法线平铺会产生重复感**：用两套不同频率的细节法线交替平铺，或加世界空间 triplanar 扰动
4. **远距离要关掉细节法线**：5 米外细节法线贡献低于 1 像素，反而引起 shimmering——用 distance-based lerp 淡出
5. **移动端 BC5 压缩**：主法线用 BC5（2 通道，质量好），细节法线可以考虑 BC7 或 ASTC 4×4，取决于平台
6. **与法线贴图压缩配合**：BC5/ASTC 压缩本身会引入误差，混合后误差叠加——压缩率不要设太激进

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能解释为什么 Lerp 两个法线贴图是错误的 | → 向量数学盲区：法线不是颜色，是单位方向向量 |
| 能手写 UDN 或 RNV 混合公式 | → 法线混合盲区：阅读"Blending in Detail"论文 |
| 知道 Detail Normal Strength 应该调节 XY 而非整体缩放 | → Shader 数学盲区：直接缩放法线值会破坏方向 |
| 能在 URP Shader Graph 中集成 Custom Function 实现法线混合 | → 工具链盲区：练习 Custom Function 节点用法 |
| 知道远距离为什么要淡出细节法线 | → 渲染管线盲区：理解 mipmap 和 shimmering 的关系 |

### 🔗 相关问题

- [technical-art/pbr-material-authoring](../technical-art/pbr-material-authoring.md) — PBR 材质制作流程
- [technical-art/mobile-normal-map-compression](../technical-art/mobile-normal-map-compression.md) — 移动端法线贴图压缩方案
- [shader/sss-skin-rendering](../rendering/sss-skin-rendering.md) — SSS 皮肤渲染中法线的作用
