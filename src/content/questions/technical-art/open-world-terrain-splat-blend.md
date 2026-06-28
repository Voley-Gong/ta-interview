---
title: "开放世界地形材质 Blend：SplatMap 权重图工作流怎么搭？"
category: "technical-art"
level: 3
tags: ["地形", "SplatMap", "材质混合", "权重图", "开放世界", "Workflow"]
hint: "核心不是 Shader 写法——是权重图的生成规范、通道分配、接缝处理和性能预算，四层材质混合在移动端要慎用"
related: ["technical-art/mobile-texture-compression", "technical-art/shader-lod-quality-tier-system", "pipeline/houdini-terrain-river-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们正在做一款开放世界手游，地形面积 4km×4km。美术需要在地形上绘制草地、泥土、岩石、雪地四种材质，并且根据海拔高度自动过渡（低处草地→山坡泥土→高处岩石→山顶雪地）。你作为 TA 来设计整套地形材质 Blend 工作流，包括权重图规范、Shader 方案和性能预算。移动端要跑到 60fps。」

### ✅ 核心要点

1. **SplatMap（权重图）是核心**：RGBA 四通道对应四种材质的权重，每像素权重之和 = 1.0
2. **海拔自动过渡**：Shader 中根据世界坐标 Y 值 lerp 权重，再叠加美术手动绘制的 SplatMap
3. **材质层数控制**：移动端最多 4 层 Blend（一张 RGBA SplatMap），PC/主机可 8 层（两张）
4. **纹理复用策略**：每种材质的 Diffuse/Normal 用 Tiling + Offset 重复平铺，不需要超大纹理
5. **性能预算**：地形 Shader 是全屏成本的，移动端控制在 2ms 以内（简化法线、减少采样次数）

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：4km×4km 地形 → 四种材质自然过渡 → 海拔自动分布 → 美术可手动微调 → 移动端 60fps
                ↑
倒推1：需要「多种材质在同一表面混合」→ SplatMap 权重图（RGBA = 4层权重）
倒推2：需要「自然过渡不生硬」→ 权重图分辨率够高 + 高斯模糊过渡带
倒推3：需要「海拔自动分布」→ Shader 中 heightFactor = smoothstep(yLow, yHigh, worldPos.y)
      再与 SplatMap 权重做加权混合
倒推4：需要「美术可手动绘制」→ Unity Terrain Paint / Houdini Height Field / World Machine
倒推5：需要「大面积不重复」→ 每层材质 Tiling 平铺 + 细节法线打破重复感
倒推6：需要「移动端跑得动」→ 采样次数优化 + SplatMap 分块 Streaming + 低分辨率权重图
```

#### 知识点拆解（倒推树）

```
地形材质 Blend 系统
├── SplatMap 规范设计
│   ├── 分辨率：通常地形分块 256×256 或 512×512 per patch
│   ├── 通道分配：R=草地, G=泥土, B=岩石, A=雪地（可配置）
│   ├── 权重归一化：RGBA 和为 1.0（美术绘制时自动归一化）
│   ├── 多 SplatMap：4 层不够时叠加第二张（移动端慎用）
│   └── 分块 Streaming：按玩家位置加载邻近 patch 的 SplatMap
├── Shader 方案
│   ├── 顶点着色器：传入 worldPos.y 用于海拔计算
│   ├── 片元着色器核心：
│   │   ├── 采样 SplatMap 得到 4 权重
│   │   ├── 海拔权重混合：heightMask × splatWeight
│   │   ├── 4 层材质各采样 Albedo + Normal（共 8 次 + 1 次 Splat = 9 次采样）
│   │   └── 加权混合：finalColor = Σ(weight_i × layerColor_i)
│   ├── 斜坡优化：基于 slope angle（法线 Y 分量）自动切换岩石
│   └── 细节增强：Detail Map 叠加（近距离才显示，远距离 LOD 切换）
├── 美术绘制工作流
│   ├── 方案A：Unity Terrain 内置 Paint Texture（实时反馈好）
│   ├── 方案B：Houdini Height Field（程序化生成 → 导出 SplatMap）
│   │   ├── Height Field Pattern → 噪声分布
│   │   ├── Height Field Mask by Feature → 海拔/坡度遮罩
│   │   └── Height Field Remap → 权重归一化导出
│   ├── 方案C：World Machine / Gaea（专业地形软件）
│   │   ├── 选择器（Selector）按海拔/坡度生成遮罩
│   │   └── 导出 SplatMap 作为 Unity Terrain Layer 权重
│   └── 混合方案：程序化 70% + 美术手绘微调 30%
├── 性能预算（移动端）
│   ├── 采样次数：SplatMap(1) + 4×Albedo(4) + 4×Normal(4) = 9 次 → 压力大
│   ├── 优化方案：
│   │   ├── 方案1：双材质混合替代四层（采样次数 5 次）
│   │   ├── 方案2：预合并 — 离线将 SplatMap 烘焙到 BaseMap（牺牲灵活性）
│   │   ├── 方案3：SplatMap 半分辨率（权重不需要像素级精度）
│   │   └── 方案4：远处切换为预烘焙的 Simple Lit（LOD 策略）
│   ├── 带宽估算：9 次采样 × 4 bytes × 1280×720 ≈ 33MB/frame → 可接受
│   └── 目标：地形 Shader < 2ms (Adreno 650 @ 1080p)
└── 接缝与边界处理
    ├── Patch 间接缝：SplatMap 边缘做 1-2 像素 padding blur
    ├── 远距离雾化：远处地形混合到雾色，掩盖 SplatMap 精度不足
    └── 水面交界：岸边材质 Blend 到湿沙效果（基于高度判断）
```

#### 代码实现

**地形 Blend Shader（HLSL for URP）：**

```hlsl
// terrain_splat_blend.shader
Shader "Custom/TerrainSplatBlend"
{
    Properties
    {
        _SplatMap ("Splat Map (RGBA)", 2D) = "white" {}
        
        // 四层材质
        _GrassAlbedo ("Grass Albedo", 2D) = "white" {}
        _GrassNormal ("Grass Normal", 2D) = "bump" {}
        _GrassTiling ("Grass Tiling", Float) = 10
        
        _DirtAlbedo ("Dirt Albedo", 2D) = "white" {}
        _DirtNormal ("Dirt Normal", 2D) = "bump" {}
        _DirtTiling ("Dirt Tiling", Float) = 8
        
        _RockAlbedo ("Rock Albedo", 2D) = "white" {}
        _RockNormal ("Rock Normal", 2D) = "bump" {}
        _RockTiling ("Rock Tiling", Float) = 6
        
        _SnowAlbedo ("Snow Albedo", 2D) = "white" {}
        _SnowNormal ("Snow Normal", 2D) = "bump" {}
        _SnowTiling ("Snow Tiling", Float) = 12
        
        // 海拔过渡参数
        _GrassMaxHeight ("Grass Max Height", Float) = 50
        _RockMinHeight ("Rock Min Height", Float) = 80
        _SnowMinHeight ("Snow Min Height", Float) = 150
        _HeightBlendRange ("Height Blend Range", Float) = 20
        
        // 坡度控制
        _RockMaxSlope ("Rock Max Slope (0=flat, 1=vertical)", Float) = 0.4
    }

    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

    struct Attributes
    {
        float4 positionOS : POSITION;
        float3 normalOS : NORMAL;
        float2 uv : TEXCOORD0;
        float2 uvSplat : TEXCOORD1;  // SplatMap 专用 UV
    };

    struct Varyings
    {
        float4 positionCS : SV_POSITION;
        float3 normalWS : NORMAL;
        float3 positionWS : TEXCOORD0;
        float2 uv : TEXCOORD1;
        float2 uvSplat : TEXCOORD2;
    };

    // SplatMap
    TEXTURE2D(_SplatMap); SAMPLER(sampler_SplatMap);

    // 四层材质纹理
    TEXTURE2D(_GrassAlbedo);  SAMPLER(sampler_GrassAlbedo);
    TEXTURE2D(_GrassNormal);  SAMPLER(sampler_GrassNormal);
    TEXTURE2D(_DirtAlbedo);   SAMPLER(sampler_DirtAlbedo);
    TEXTURE2D(_DirtNormal);   SAMPLER(sampler_DirtNormal);
    TEXTURE2D(_RockAlbedo);   SAMPLER(sampler_RockAlbedo);
    TEXTURE2D(_RockNormal);   SAMPLER(sampler_RockNormal);
    TEXTURE2D(_SnowAlbedo);   SAMPLER(sampler_SnowAlbedo);
    TEXTURE2D(_SnowNormal);   SAMPLER(sampler_SnowNormal);

    float _GrassTiling, _DirtTiling, _RockTiling, _SnowTiling;
    float _GrassMaxHeight, _RockMinHeight, _SnowMinHeight, _HeightBlendRange;
    float _RockMaxSlope;

    // 采样一层材质的 Albedo + Normal
    void SampleLayer(float2 uv, float tiling,
                     TEXTURE2D_PARAM(albedoTex, samplerA),
                     TEXTURE2D_PARAM(normalTex, samplerN),
                     out half3 albedo, out half3 normalTS)
    {
        float2 tiledUV = uv * tiling;
        albedo = SAMPLE_TEXTURE2D(albedoTex, samplerA, tiledUV).rgb;
        normalTS = UnpackNormal(SAMPLE_TEXTURE2D(normalTex, samplerN, tiledUV));
    }

    // 海拔权重计算
    float4 GetHeightWeights(float worldY, float slopeY)
    {
        float4 w = 0;

        // smoothstep 生成各层海拔遮罩
        w.r = 1.0 - smoothstep(_GrassMaxHeight - _HeightBlendRange,
                               _GrassMaxHeight + _HeightBlendRange, worldY);
        w.g = smoothstep(_GrassMaxHeight - _HeightBlendRange,
                         _GrassMaxHeight + _HeightBlendRange, worldY)
            * (1.0 - smoothstep(_RockMinHeight - _HeightBlendRange,
                               _RockMinHeight + _HeightBlendRange, worldY));
        w.b = smoothstep(_RockMinHeight - _HeightBlendRange,
                         _RockMinHeight + _HeightBlendRange, worldY)
            * (1.0 - smoothstep(_SnowMinHeight - _HeightBlendRange,
                               _SnowMinHeight + _HeightBlendRange, worldY));
        w.a = smoothstep(_SnowMinHeight - _HeightBlendRange,
                         _SnowMinHeight + _HeightBlendRange, worldY);

        // 坡度增强：陡坡自动偏向岩石
        if (slopeY < _RockMaxSlope)
        {
            float rockBoost = smoothstep(_RockMaxSlope, 0.0, slopeY) * 0.7;
            w.b += rockBoost;
        }

        // 归一化
        float sum = dot(w, 1.0);
        return w / max(sum, 0.0001);
    }

    half4 FragTerrain(Varyings i) : SV_Target
    {
        // 1. 采样 SplatMap（美术手绘权重）
        float4 splat = SAMPLE_TEXTURE2D(_SplatMap, sampler_SplatMap, i.uvSplat);

        // 2. 海拔 + 坡度自动权重
        float slopeY = abs(i.normalWS.y);  // 平地≈1, 陡坡≈0
        float4 heightW = GetHeightWeights(i.positionWS.y, slopeY);

        // 3. 最终权重 = SplatMap × 海拔权重（两者叠加）
        float4 weights = splat * heightW;
        weights /= max(dot(weights, 1.0), 0.0001);

        // 4. 采样四层材质（此处简化，实际可用循环）
        half3 grassAlb, grassNrm;
        half3 dirtAlb, dirtNrm;
        half3 rockAlb, rockNrm;
        half3 snowAlb, snowNrm;

        SampleLayer(i.uv, _GrassTiling,
            TEXTURE2D_ARGS(_GrassAlbedo, sampler_GrassAlbedo),
            TEXTURE2D_ARGS(_GrassNormal, sampler_GrassNormal),
            grassAlb, grassNrm);
        SampleLayer(i.uv, _DirtTiling,
            TEXTURE2D_ARGS(_DirtAlbedo, sampler_DirtAlbedo),
            TEXTURE2D_ARGS(_DirtNormal, sampler_DirtNormal),
            dirtAlb, dirtNrm);
        SampleLayer(i.uv, _RockTiling,
            TEXTURE2D_ARGS(_RockAlbedo, sampler_RockAlbedo),
            TEXTURE2D_ARGS(_RockNormal, sampler_RockNormal),
            rockAlb, rockNrm);
        SampleLayer(i.uv, _SnowTiling,
            TEXTURE2D_ARGS(_SnowAlbedo, sampler_SnowAlbedo),
            TEXTURE2D_ARGS(_SnowNormal, sampler_SnowNormal),
            snowAlb, snowNrm);

        // 5. 加权混合
        half3 albedo = grassAlb * weights.r
                     + dirtAlb  * weights.g
                     + rockAlb  * weights.b
                     + snowAlb  * weights.a;

        half3 normalTS = grassNrm * weights.r
                       + dirtNrm  * weights.g
                       + rockNrm  * weights.b
                       + snowNrm  * weights.a;

        // 6. 标准 URP 光照
        InputData inputData = (InputData)0;
        inputData.positionWS = i.positionWS;
        inputData.normalWS = TransformTangentToWorld(normalTS,
            half3x3(float3(1,0,0), float3(0,1,0), float3(0,0,1))); // 简化
        inputData.normalWS = normalize(inputData.normalWS);

        SurfaceData surface = (SurfaceData)0;
        surface.albedo = albedo;
        surface.occlusion = 1.0;
        surface.smoothness = 0.5;

        return UniversalFragment_BlinnPhong(inputData, surface);
    }
    ENDHLSL

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        LOD 100
        Pass
        {
            Name "TerrainSplat"
            HLSLPROGRAM
            #pragma vertex UnlitPassVert
            #pragma fragment FragTerrain
            ENDHLSL
        }
    }
}
```

### ⚡ 实战经验

1. **SplatMap 分辨率是体验和性能的平衡点**：太低则材质边界锯齿严重，太高则内存爆炸。实践中 1 texel/m²（每平方米一个权重像素）是甜点
2. **海拔自动分布 + 手绘微调 = 最佳实践**：纯手绘 4km² 地形要 2 周，程序化生成 + 手绘重点区域只要 2 天
3. **Triplanar Mapping 解决岩石拉伸**：陡坡上的岩石材质如果用普通 UV 会严重拉伸，改用三平面投影（X/Y/Z 三方向投影取最佳）
4. **移动端务实方案**：远处不跑 4 层 Blend——离玩家 50m 外的地形切到预烘焙 Simple Lit，近处才跑完整 Blend
5. **草地的特殊性**：草地通常不只用材质——叠加 GPU Instancing 草丛 mesh（ billboard / cross mesh），材质只管地面底色
6. **Houdini Height Field 是核武器**：用 Houdini 做完程序化地形后，用 `Height Field Convert` + `Height Field Output` 直接导出 SplatMap，流程远超 Unity 内置 Paint
7. **边界软化技巧**：在 Houdini 中用 `Height Field Blur` 对 SplatMap 做轻度模糊（1-2 像素），可以消除材质间的硬边

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能解释 SplatMap RGBA 四通道的权重归一化原理 | → 材质混合盲区：复习权重混合数学 |
| 知道为什么四层 Blend 在移动端有性能压力 | → 移动 GPU 盲区：纹理采样带宽估算 |
| 能用 Houdini Height Field 生成海拔分层遮罩 | → Houdini 盲区：Height Field 节点链 |
| 理解 Triplanar Mapping 解决什么问题 | → UV 投影盲区：三平面投影原理 |
| 能设计 SplatMap 分块 Streaming 加载策略 | → 开放世界盲区：资源 Streaming 分块 |

### 🔗 相关问题

- [technical-art/mobile-texture-compression](../technical-art/mobile-texture-compression.md) — 移动端贴图压缩方案选型
- [technical-art/shader-lod-quality-tier-system](../technical-art/shader-lod-quality-tier-system.md) — Shader LOD 与画质分级系统
- [pipeline/houdini-terrain-river-pipeline](../pipeline/houdini-terrain-river-pipeline.md) — Houdini 地形河流生成管线
