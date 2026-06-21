---
title: "延迟渲染下几百个动态光源怎么不炸？G-Buffer打包与Light Culling策略"
category: "rendering"
level: 3
tags: ["延迟渲染", "G-Buffer", "多光源", "Light Culling", "Tiled", "Cluster", "URP", "性能优化"]
hint: "核心考点：G-Buffer位深分配 + Tile/Cluster Light Culling + 前向 vs 延迟的取舍——不是所有平台都适合 Deferred"
related: ["rendering/urp-renderer-feature", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们正在做一个开放世界手游，夜城市场景里有大量霓虹灯、车灯、火把等动态点光源，同时存在几十盏。你负责渲染方案选型。
>
> 1. 你选 Forward 还是 Deferred？为什么？
> 2. 如果选 Deferred，G-Buffer 怎么打包？移动端 MRT 限制怎么处理？
> 3. 几百个光源的 Light Loop 怎么不把 GPU 拖垮？

### ✅ 核心要点

1. **管线选型决策**：Forward vs Deferred vs Forward+ 各自的光源复杂度拐点
2. **G-Buffer 打包策略**：有限 MRT 下如何塞下 Albedo / Normal / Roughness / Metallic / Emission / AO
3. **Light Culling 算法**：Tiled Forward+（屏幕空间分块）vs Cluster（三维分簇）的选择
4. **移动端适配**：GLES 3.0 的 MRT 限制、带宽爆炸问题的应对
5. **混合管线方案**：不透明用 Deferred，半透明回退 Forward

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：几百个动态光源，60fps 移动端
  ↓
瓶颈分析：Forward 的 per-light draw call 会爆炸（N光源 × M物体 = NM 次）
  ↓
方案选择：Deferred 渲染 → 光照计算在屏幕空间，与场景复杂度解耦
  ↓
新瓶颈：G-Buffer 带宽 + Light Loop 计算
  ↓
优化 1：G-Buffer 紧凑打包（少 RT、少位深）
优化 2：Light Culling（每像素只算影响它的少量光源）
  ↓
半透明问题：Deferred 天然不支持半透明 → 混合管线
```

#### 知识点拆解（倒推树）

```
几百个动态光源渲染
├── 管线选型
│   ├── Forward：O(N×M) draw call，光源越多越慢
│   ├── Deferred：O(N_pixel) 光照计算，与物体数无关
│   ├── Forward+ / Tiled Forward：分块做光源剔除，两全其美
│   └── 决策依据：光源数量、平台 MRT 支持、半透明比例、G-Buffer 带宽
│
├── G-Buffer 打包（Deferred 的核心难点）
│   ├── MRT 数量限制：PC 最多 8 张，移动端 GLES 3.0 实际 4 张
│   ├── 位深分配策略
│   │   ├── Albedo：RGB8（足够）
│   │   ├── Normal：RG16F 或 Octahedron 编码（省空间）
│   │   ├── Material（Roughness/Metallic/AO）：RGBA8 打包
│   │   └── Emission：RGB9E5 共享指数（HDR 省空间）
│   └── 带宽估算：4 张 RT × 1920×1080 × 4B = ~33MB/帧（仅 G-Buffer pass）
│
├── Light Culling
│   ├── Tiled（屏幕空间 2D 分块）
│   │   ├── 16×16 或 32×32 像素一个 Tile
│   │   ├── Compute Shader 计算每 Tile 的光源列表
│   │   └── 问题：深度跨度大的 Tile 光源数仍多
│   ├── Cluster（3D 分簇，Z 方向切片）
│   │   ├── 解决深度不连续问题
│   │   ├── 每簇内光源数大幅减少
│   │   └── 代表实现：Cluster Forward Shading（Olsson et al. 2012）
│   └── 移动端注意：无 Compute Shader 时用 CPU 预计算 / Fragment 直算
│
└── 半透明回退
    ├── Deferred 无法处理半透明（G-Buffer 只有最近层）
    ├── 方案：半透明物体单独 Forward pass
    └── 注意排序和光源传递
```

#### 代码实现

**G-Buffer 打包（URP Deferred 风格伪代码）：**

```hlsl
// G-Buffer 0: Albedo (RGB) + AO (A)
// G-Buffer 1: Normal XY (RG16F) + Roughness (B) + Metallic (A)
// G-Buffer 2: Emission (RGB9E5)

struct GBufferOutput {
    half4 buffer0 : SV_Target0;
    half4 buffer1 : SV_Target1;
    half4 buffer2 : SV_Target2;
};

GBufferOutput EncodeGBuffer(half3 albedo, half3 normalWS, 
                             half roughness, half metallic, half ao, half3 emission) {
    GBufferOutput o;
    o.buffer0 = half4(albedo, ao);
    
    // Octahedron 编码法线，省到 2 个通道（可选方案）
    half2 octNormal = PackNormalOctahedron(normalWS);
    o.buffer1 = half4(octNormal, roughness, metallic);
    
    // RGB9E5 共享指数编码 HDR 颜色
    o.buffer2 = half4(EncodeRGB9E5(emission), 1);
    
    return o;
}

// Light Pass 中解码
half3 DecodeGBufferForLighting(GBufferOutput gbuf) {
    half3 albedo = gbuf.buffer0.rgb;
    half ao = gbuf.buffer0.a;
    half3 normalWS = UnpackNormalOctahedron(gbuf.buffer1.rg);
    half roughness = gbuf.buffer1.b;
    half metallic = gbuf.buffer1.a;
    half3 emission = DecodeRGB9E5(gbuf.buffer2.rgb);
    // ... BRDF 计算
}
```

**Tiled Light Culling（Compute Shader）：**

```hlsl
[numthreads(TILE_SIZE, TILE_SIZE, 1)]
void CSMain(uint3 dispatchThreadId : SV_DispatchThreadID,
            uint3 groupId : SV_GroupID,
            uint groupIndex : SV_GroupIndex) {
    uint tileIndex = groupId.y * tileCountX + groupId.x;
    
    // Step 1: 计算 Tile 的 min/max depth（通过 GroupShared 内存 reduction）
    float minDepth = 1.0, maxDepth = 0.0;
    float depth = depthBuffer.Load(int3(dispatchThreadId.xy, 0)).r;
    // ... InterlockedMin/Max reduction
    
    // Step 2: 遍历所有光源，判断是否与 Tile AABB 相交
    uint lightCount = 0;
    uint lightIndices[MAX_LIGHTS_PER_TILE];
    
    for (uint i = 0; i < totalLightCount; i++) {
        Light light = lights[i];
        if (SphereIntersectsAABB(light.position, light.range, 
                                  tileAABB)) {
            lightIndices[lightCount++] = i;
        }
    }
    
    // Step 3: 写入 light index list
    if (groupIndex == 0) {
        tileLightCount[tileIndex] = lightCount;
        // 写入 lightIndices 到全局 buffer
    }
}
```

**管线对比表：**

| 维度 | Forward | Deferred | Forward+/Cluster |
|------|---------|----------|-------------------|
| 光照复杂度 | O(N×M) | O(N_pixel) | O(N_tile) |
| 半透明支持 | ✅ 原生 | ❌ 需回退 | ✅ 原生 |
| MSAA | ✅ 原生 | ❌ 困难 | ✅ 可行 |
| 移动端友好 | ✅ | ⚠️ MRT限制 | ⚠️ CS限制 |
| G-Buffer带宽 | 无 | 高 | 中 |
| 材质多样性 | ✅ 随意 | ⚠️ 统一光照模型 | ✅ 灵活 |
| 适用场景 | 少光源 | 多光源+统一材质 | 多光源+多样材质 |

### ⚡ 实战经验

- **移动端 Deferred 要三思**：GLES 3.0 的 4 RT MRT 在很多设备上性能惩罚巨大，带宽可能是 Forward 的 3-5 倍。实测发现中端安卓机 Deferred 比 Forward+ 慢 40%。高端机才能利用 Deferred 的优势
- **Forward+ / Cluster Forward 是手游最优解**：既保留 Forward 的材质灵活性和半透明支持，又通过 Light Culling 解决多光源问题。原神就是 Cluster Forward 方案
- **G-Buffer 打包是艺术**：正常值用 RGBA8，HDR 用 RGB9E5，法线用 Octahedron 编码可以从 RG16F 省到 RGBA8 的 2 个通道。每省 1 张 RT 就省巨大带宽
- **光源优先级裁剪**：实际项目中不需要每帧都精确 cull 所有光源。可以做光源层级（关键光/次要光/氛围光），按距离和重要性分帧剔除

### 🎯 能力体检清单

| 卡住的环节 | 盲区诊断 | 学习建议 |
|------------|----------|----------|
| 不知道选 Forward 还是 Deferred | 缺乏管线选型全局视角 | 对比三种管线的光源复杂度公式，做一次完整选型分析 |
| G-Buffer 不知道怎么塞 | 不熟悉数据打包编码 | 研究 GDC 2011 "Stable SSAO in Battlefield3" 的 G-Buffer 打包方案 |
| Light Culling 写不出来 | Compute Shader 基础不足 | 先写 CPU 版本理解算法，再迁移到 CS |
| 半透明处理不清楚 | 混合管线概念缺失 | 研究 UE5 的 Deferred + Forward 半透明 pass 实现 |
| 移动端方案选错 | 不了解移动端硬件特性 | 对比骁龙 8 Gen 系列的带宽实测数据，理解为什么移动端谨慎用 Deferred |

### 🔗 相关问题

- [URP 自定义 Renderer Feature](rendering/urp-renderer-feature) — URP 下如何扩展渲染管线
- [GPU 带宽优化](optimization/gpu-bandwidth-optimization) — G-Buffer 带宽是 Deferred 的最大开销
- Cluster Forward 的 Z 切片策略如何设计？（深度分布非线性，近处密远处疏）
