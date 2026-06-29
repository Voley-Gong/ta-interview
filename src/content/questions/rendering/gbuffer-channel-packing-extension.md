---
title: "GBuffer 通道不够用：延迟管线要加自定义渲染通道怎么办？"
category: "rendering"
level: 4
tags: ["GBuffer", "延迟渲染", "MRT", "通道打包", "URP", "自定义Pass"]
hint: "GBuffer 通道数是固定的——想要多塞数据，要么压缩重打包，要么扩展 MRT，要么走 Deferred+Forward 混合"
related: ["rendering/deferred-multi-light", "rendering/deferred-rendering-transparency-solution", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的项目用的是延迟渲染，现在策划要加一个'涂装系统'——玩家可以在场景任意表面上喷涂颜色和图案，且这些涂装要受光照影响。但 GBuffer 的 4 个 MRT 通道已经全部分配完了（Albedo、Normal、SpecularSmoothness、Emission）。你怎么在不破坏现有渲染管线的前提下，把涂装数据加进去？」

这是腾讯天美、字节朝夕光年等做延迟管线项目的高频追问。考察的不只是 GBuffer 打包技巧，更是对整个延迟管线架构的理解深度。

### ✅ 核心要点

1. **GBuffer 通道预算**：传统延迟渲染 GBuffer 通常 4 个 MRT（Multiple Render Target），每个 RGBA8 = 32bit，总计 128bit/像素
2. **通道重打包**：分析现有通道精度冗余，压缩后腾出空间（如 Specular 和 Smoothness 合并到 RGB 两个通道）
3. **MRT 扩展**：增加 GBuffer5，但要注意带宽和兼容性成本
4. **混合管线方案**：涂装走单独的 Forward Pass 叠加，避免动 GBuffer
5. **移动端限制**：Adreno/Mali GPU 对 MRT 数量有限制（通常 ≤ 4），扩展方案需因地制宜

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：在延迟管线的 GBuffer 中加入「涂装颜色 + 涂装遮罩」，不破坏现有光照计算
     ↓
Step 1：评估现有 GBuffer 通道利用率
  ├── MRT0 (RGBA8): Albedo.rgb + AO.a → AO 只需 1bit，a 通道浪费了 7bit
  ├── MRT1 (RGBA8): Normal.rg + unused.ba → 法线用球面坐标只需 2 通道，ba 空闲！
  ├── MRT2 (RGBA8): Specular.r + Smoothness.g + Metallic.b + unused.a
  └── MRT3 (RGBA8): Emission.rgb + unused.a
     ↓
Step 2：方案比选
  ├── 方案A：通道重打包 → 腾出空间（免费，但需改所有 Lighting Shader）
  ├── 方案B：增加 GBuffer4 MRT → 最干净（但带宽 +25%，移动端可能不支持）
  ├── 方案C：Stencil Buffer 涂装标记 + 独立 Forward Pass（不动 GBuffer）
  └── 方案D：GBuffer 打包涂装遮罩 + 运行时查涂装纹理（混合方案）
     ↓
Step 3：选定方案并实现（本题选方案A + D 混合）
  ├── 重打包：Normal 从 2 通道压到 Octahedral 编码（仍 2 通道但释放精度空间）
  ├── 涂装遮罩存入 MRT1 的 .b 通道（0 = 无涂装，1 = 有涂装）
  ├── 涂装颜色存入 MRT3 的 .a 通道（作为 Index 查 LUT）
  └── Lighting Pass 中根据遮罩采样涂装 LUT 替换 Albedo
```

#### 知识点拆解（倒推树）

```
GBuffer 通道扩展
├── 延迟渲染架构理解
│   ├── GBuffer Pass → 几何数据写入 MRT
│   ├── Lighting Pass → 读取 GBuffer 计算光照
│   ├── MRT (Multiple Render Target) 机制
│   │   ├── GPU 同时输出多个 Render Target
│   │   ├── 带宽 = MRT数量 × 分辨率 × 通道位数
│   │   └── 移动端 TBDR 的 Tile 内存限制
│   └── GBuffer 通道分配策略
│       ├── 必需数据：Albedo, Normal, Material params (Roughness/Metallic/Specular)
│       ├── 可选数据：AO, Emission, Subsurface Scattering mask
│       └── 自定义数据：涂装、伤口、雪覆盖、湿度等 gameplay 数据
├── 通道打包技术
│   ├── 法线压缩
│   │   ├── 球面坐标：Normal.rg → 2通道，重建 z = sqrt(1 - x² - y²)
│   │   ├── Octahedral 编码：2通道更均匀的精度分布
│   │   └── Stereographic 投影：精度最优但解码成本高
│   ├── 材质参数合并
│   │   ├── Metallic + Roughness → 1 通道（Metallic 1bit + Roughness 7bit）
│   │   ├── Specular + Smoothness → 1 通道（各 4bit）
│   │   └── F0 + Roughness → 2 通道（标准 PBR 最小集）
│   └── Bit 打包
│       ├── 1 byte 塞 2 个 4bit 值：floor(x * 15) << 4 | floor(y * 15)
│       ├── 1 byte 塞 8 个 1bit flag：位掩码
│       └── 精度损失评估：4bit = 16 级梯度，肉眼是否可分辨？
├── MRT 扩展方案
│   ├── GBuffer5/6 添加
│   │   ├── URP/UE5 中配置额外 GBuffer RT
│   │   ├── 带宽成本：1080p × RGBA8 = 8MB/帧/MRT
│   │   ├── 移动端限制：Adreno 支持 ≤ 4 MRT，Mali ≤ 8 MRT
│   │   └── 带宽优化：RGBA16F → RGBA8 或 RGB565 降精度
│   └── 条件 MRT 写入
│       ├── 只在涂装区域写 GBuffer5（Pixel Shader 分支）
│       └── 利用 SV_Target5 可选写入（GPU 仍分配带宽）
├── 混合管线方案
│   ├── Stencil + Forward
│   │   ├── GBuffer Pass 用 Stencil 标记涂装区域
│   │   ├── 单独 Forward Pass 渲染涂装（只画被标记的像素）
│   │   └── 优势：不动 GBuffer，劣势：Forward 光照开销
│   ├── Light Pass 注入
│   │   ├── 在 Lighting Shader 中加一个 if(customMask) 分支
│   │   ├── 采样涂装贴图（Bindless 或 Texture Array）
│   │   └── 只改 Lighting Shader，不改 GBuffer 写入
│   └── Screen Space Decal
│       ├── 用 Decal System 投射涂装（DBuffer 方案）
│       ├── DBuffer = 独立的 Decal GBuffer
│       └── UE5 的 DBuffer Mesh Decals 就是这个思路
└── 性能评估
    ├── 带宽对比
    │   ├── 4 MRT (128bit) → 5 MRT (160bit)：+25% GBuffer 带宽
    │   ├── 移动端 TBDR：Tile 内 SRAM 膨胀 → L2 Cache 命中率下降
    │   └── 实测影响：骁龙8 Gen2 上 1080p 延迟渲染，+1 MRT ≈ -3fps
    ├── 重打包成本
    │   ├── 编码/解码 ALU 指令增加
    │   ├── 但带宽不变（MRT 数量没变）
    │   └── 净效果通常优于增加 MRT
    └── 内存占用
        ├── GBuffer RT 内存：4×4MB → 5×4MB = +4MB
        └── 移动端 4GB 设备上需要权衡
```

#### 代码实现

**方案 A：GBuffer 重打包（URP 12+，释放 MRT1.b 存涂装遮罩）**

```hlsl
// GBufferWrite.hlsl —— 修改 GBuffer Pass 的写入逻辑

#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Decal.hlsl"

// 涂装数据从 Custom Texture 采样
TEXTURE2D(_CustomPaintTex); SAMPLER(sampler_CustomPaintTex);

struct GBufferOut
{
    half4 gbuffer0 : SV_Target0; // Albedo + AO
    half4 gbuffer1 : SV_Target1; // Normal.xy + PaintMask + unused
    half4 gbuffer2 : SV_Target2; // Specular + Smoothness + Metallic
    half4 gbuffer3 : SV_Target3; // Emission + PaintColorIndex
};

// Octahedral 编码：法线 3→2 通道
float2 OctEncode(float3 n)
{
    float l1norm = abs(n.x) + abs(n.y) + abs(n.z);
    float2 res = n.xy / l1norm;
    if (n.z < 0)
        res = (1.0 - abs(res.yx)) * (res.xy >= 0 ? 1.0 : -1.0);
    return res;
}

GBufferOut WriteGBuffer(Varyings input)
{
    GBufferOut output;

    // 原有材质数据
    half3 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv).rgb;
    half  ao     = 1.0;
    half3 normal = NormalizeNormalPerPixel(input.normalWS);

    // === 涂装系统数据采样 ===
    half paintMask = 0;
    half paintColorIdx = 0;
    if (_PaintSystemEnabled)
    {
        half4 paintData = SAMPLE_TEXTURE2D(_CustomPaintTex, sampler_CustomPaintTex, input.uv * _PaintTexTiling);
        paintMask = paintData.r;       // 0 = 无涂装, 1 = 完全覆盖
        paintColorIdx = paintData.g;   // 颜色 LUT 索引 (0~1 映射到 LUT)
    }

    // === GBuffer 通道打包 ===
    output.gbuffer0 = half4(albedo, PackR8(ao));          // Albedo.rgb + AO(8bit)
    float2 octNormal = OctEncode(normal);                  // Octahedral 编码
    output.gbuffer1 = half4(octNormal, paintMask, 0);     // Normal.xy + PaintMask + unused
    output.gbuffer2 = half4(_Specular, _Smoothness, _Metallic, 0);
    output.gbuffer3 = half4(_Emission, paintColorIdx);    // Emission.rgb + PaintColorIndex

    return output;
}
```

**Lighting Pass 中读取涂装数据：**

```hlsl
// DeferredLighting.hlsl —— 修改 Lighting Pass

half3 CustomLighting(float2 uv, float3 lightDir, float3 viewDir)
{
    // 读取 GBuffer
    half4 gbuffer0 = SAMPLE_TEXTURE2D(_GBuffer0, sampler_GBuffer0, uv);
    half4 gbuffer1 = SAMPLE_TEXTURE2D(_GBuffer1, sampler_GBuffer1, uv);
    half4 gbuffer2 = SAMPLE_TEXTURE2D(_GBuffer2, sampler_GBuffer2, uv);
    half4 gbuffer3 = SAMPLE_TEXTURE2D(_GBuffer3, sampler_GBuffer3, uv);

    // 解码法线
    float3 normal = OctDecode(gbuffer1.xy);

    // 检查涂装遮罩
    half paintMask = gbuffer1.z;
    half3 albedo = gbuffer0.rgb;

    if (paintMask > 0.01)
    {
        // 用涂装颜色 LUT 替换 Albedo
        half paintColorIdx = gbuffer3.a;
        half3 paintColor = SAMPLE_TEXTURE2D(_PaintColorLUT, sampler_PaintColorLUT, float2(paintColorIdx, 0.5)).rgb;
        albedo = lerp(albedo, paintColor, paintMask);
    }

    // 标准 PBR 光照计算（使用解码后的数据）
    half3 brdf = BRDF_Lambert(albedo) + BRDF_Specular(gbuffer2.rgb, gbuffer2.a, normal, lightDir, viewDir);
    return brdf;
}

// Octahedral 解码
float3 OctDecode(float2 f)
{
    float3 n = float3(f.x, f.y, 1.0 - abs(f.x) - abs(f.y));
    float t = saturate(-n.z);
    n.xy += (n.x >= 0 ? -t : t);
    n.xy += (n.y >= 0 ? -t : t);
    return normalize(n);
}
```

**方案 B：GBuffer5 扩展（URP 自定义 GBuffer）：**

```csharp
// ExtendedGBufferFeature.cs —— URP 中增加 GBuffer5
public class ExtendedGBufferFeature : ScriptableRendererFeature
{
    private ExtendedGBufferPass _pass;
    private RTHandle _gbuffer5Handle;

    public override void Create()
    {
        _pass = new ExtendedGBufferPass
        {
            renderPassEvent = RenderPassEvent.AfterRenderingGbuffer
        };
    }

    public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
    {
        // 在 GBuffer Pass 之后、Lighting Pass 之前渲染涂装数据
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        desc.depthBufferBits = 0;
        desc.colorFormat = GraphicsFormat.R8G8B8A8_UNorm;

        RenderingUtils.ReAllocateIfNeeded(ref _gbuffer5Handle, desc, name: "_GBuffer5");
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        renderer.EnqueuePass(_pass);
    }
}

// 在 Shader 中声明 GBuffer5
// half4 gbuffer5 : SV_Target5;
```

**方案对比表：**

| 方案 | 带宽变化 | 侵入性 | 移动端兼容 | 推荐场景 |
|------|----------|--------|------------|----------|
| A: 通道重打包 | 0% | 高（改所有 Shader） | 全兼容 | 移动端首选 |
| B: 增加 GBuffer5 | +25% | 低（独立 Pass） | 部分不支持 | PC/主机端 |
| C: Stencil + Forward | +少量 Forward | 中（加 Pass） | 全兼容 | 快速原型 |
| D: DBuffer Decal | +1 DBuffer | 中（Decal 系统） | 全兼容 | UE5 项目 |

### ⚡ 实战经验

1. **先审计再动刀**：拿到 GBuffer 布局图后，用 RenderDoc 查看每个通道的实际值分布——你会发现很多通道精度严重浪费（比如 AO 几乎只有 0/1 两个值）
2. **Octahedral 编码是利器**：比球面坐标精度更均匀，解码速度接近，已经成为业界标准（UE5、Unity HDRP 都在用）
3. **移动端别轻易加 MRT**：Adreno 5xx/6xx 对 5+ MRT 支持有性能悬崖，实测 Mali-G78 上 5 MRT 比 4 MRT 慢 15-20%
4. **涂装系统的实战建议**：如果涂装区域有限（贴花式），用 Stencil + Forward 叠加最省事；如果需要大面积覆盖（如雪覆盖全局），走 GBuffer 打包更高效
5. **Debug 可视化**：写一个 Shader 把各 GBuffer 通道可视化出来（类似 Unity Frame Debugger 的 GBuffer 预览），调打包精度时必不可少

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能画出 4 个 GBuffer MRT 的通道布局 | → 延迟渲染基础盲区 |
| 理解 Octahedral 编码为什么优于球面坐标 | → 法线压缩技术盲区 |
| 能解释 MRT 带宽计算公式 | → GPU 架构盲区 |
| 知道移动端 TBDR 对 MRT 数量的限制 | → 移动 GPU 架构盲区 |
| 能在 URP 中新增一个 GBuffer Pass | → URP 扩展能力盲区 |

### 🔗 相关问题

- [rendering/deferred-multi-light](../rendering/deferred-multi-light.md) — 延迟渲染下的多光源裁剪策略
- [rendering/deferred-rendering-transparency-solution](../rendering/deferred-rendering-transparency-solution.md) — 延迟渲染透明物体方案
- [rendering/custom-post-processing-urp](../rendering/custom-post-processing-urp.md) — URP 自定义后处理管线
