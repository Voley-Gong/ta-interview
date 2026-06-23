---
title: "面试官问：DXR 实时光线追踪在手游上可行吗？PC 端混合渲染管线怎么搭？"
category: "rendering"
level: 4
tags: ["Ray Tracing", "DXR", "Vulkan RT", "混合渲染", "Reflection", "Shadow", "Denoiser", "UE5 Lumen"]
hint: "核心：RT 不是全替换光栅化，而是混合管线——光栅化做 G-Buffer，RT 做 Reflections/Shadow/GI，Denoiser 降噪"
related: ["rendering/sss-skin-rendering", "rendering/deferred-multi-light", "rendering/gpu-driven-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官说：「你提到熟悉现代渲染管线。那请教几个光线追踪的问题：

第一，DXR 的 RT 核心 和 Compute Shader 通用计算有什么区别？为什么 RT 核心能加速光线追踪？

第二，如果我们做一个 PC 端 3A 级别的游戏，你想用光追替换哪些光栅化效果？怎么搭一个混合渲染管线？

第三，现在手机端（骁龙 8 Gen 3 / Apple A17 Pro）能跑光追了吗？如果要做手游光追，你会选择只做哪个功能？」

追问一：光追反射和 SSR（Screen Space Reflection）相比，优势和代价分别是什么？

追问二：BVH（Bounding Volume Hierarchy）是什么？为什么光线追踪需要它？更新动态物体的 BVH 有什么开销？

### ✅ 核心要点

1. **RT 核心本质**：硬件加速 Ray-Triangle 相交测试 + BVH 遍历，把 CPU/GPU 通用计算需要成百上千条指令的操作压缩到一个时钟周期
2. **混合管线哲学**：光栅化做主画面（G-Buffer），RT 在此基础上叠加 Reflections / Shadows / GI / AO
3. **BVH 是加速结构**：把场景组织成层次包围盒树，光线遍历时快速跳过大块无关三角形
4. **Denoiser 必不可少**：每像素只追踪 1-4 根光线 → 噪声极大 → 需要 时域+空域 降噪（SVGF / OIDN）
5. **手游现状**：骁龙 8 Gen 3 支持 Vulkan RT，但性能预算只够做 **反射** 或 **软阴影** 单一功能；Apple A17 Pro 支持 Metal RT

### 📖 深度展开

#### 解决思路（从最终画面倒推管线）

```
最终效果：角色站在水坑前，水面反射出身后建筑物，反射清晰无噪声，60fps
              ↑
倒推1：反射怎么做的？→ 光线追踪（从 G-Buffer 的世界坐标发射反射光线）
倒推2：每像素多少根光线？→ 1-4 SPP（Samples Per Pixel），多了跑不动
倒推3：1 SPP 噪成这样能用？→ Denoiser 时域+空域降噪，利用上一帧累积
倒推4：光线怎么快速找到相交三角形？→ BVH 加速结构
倒推5：BVH 怎么来的？→ 离线构建静态 BVH + 运行时更新动态物体 BVH（Refit / Rebuild）
倒推6：GPU 怎么发射光线？→ DXR 的 TraceRay() / Vulkan 的 traceRayEXT()
倒推7：RT 核心做了什么？→ 硬件遍历 BVH + Ray-Triangle 交集测试
```

#### 知识点拆解（倒推树）

```
实时光线追踪混合管线
├── 硬件基础
│   ├── RT Core（NVIDIA Turing+ / AMD RDNA2+ / Intel Xe+）
│   │   ├── Ray-Triangle 交集测试单元（固定功能硬件）
│   │   ├── BVH 遍历单元（Box Intersection 测试）
│   │   └── 与 Compute Shader 的区别（CS 是通用计算，RT Core 是固定功能加速）
│   ├── 移动端 RT 支持
│   │   ├── 骁龙 8 Gen 2/3：Adreno GPU 支持 Vulkan RT（性能有限）
│   │   ├── Apple A17 Pro：Metal Ray Tracing API（RE4 重制版已搭载）
│   │   └── Mali-G715+：Valhall 架构支持 Vulkan RT
│   └── API 层
│       ├── DXR 1.0 / 1.1（DirectX 12）
│       ├── Vulkan VK_KHR_ray_tracing_pipeline
│       └── Metal 3 Ray Tracing（Apple 生态）
├── BVH 加速结构
│   ├── 构建（离线 / 运行时）
│   │   ├── TLAS（Top-Level AS）：场景级，包围 Instance
│   │   ├── BLAS（Bottom-Level AS）：Mesh 级，包围三角形
│   │   └── 构建 quality（Fast Build vs Fast Trace 权衡）
│   ├── 动态更新
│   │   ├── Refit（更新位置，不重建拓扑）→ 角色动画用
│   │   ├── Rebuild（重建整棵树）→ 形变大的物体
│   │   └── 更新频率策略（静态只建一次，动态每帧 Refit）
│   └── 内存开销（BVH 节点 ≈ 原始三角形数据的 1.5-2x）
├── 混合渲染管线
│   ├── Phase 1: 光栅化 G-Buffer（Albedo / Normal / Roughness / Depth）
│   ├── Phase 2: RT Passes（可选择叠加）
│   │   ├── RT Reflections（从 G-Buffer 发射反射光线）
│   │   ├── RT Shadows（从光源发射阴影光线）
│   │   ├── RT GI（从 G-Buffer 发射漫反射弹射光线 → 半球采样）
│   │   └── RT AO（短距离半球采样，比 GI 便宜）
│   ├── Phase 3: Denoiser（时域累积 + 空域滤波）
│   │   ├── SVGF（Spatiotemporal Variance-Guided Filtering）
│   │   ├── 历史帧复用（上一帧结果投影到当前帧）
│   │   └── Disocclusion 检测（物体移动露出后面 → 历史无效 → 重置累积）
│   └── Phase 4: 合成 Upscale（DLSS / FSR / XeSS）
├── RT 反射 vs SSR 对比
│   ├── SSR：只反射屏幕内可见物体 → 屏幕外信息丢失
│   ├── RT 反射：反射全场景 → 无视屏幕边界
│   ├── SSR 代价：低（只读 G-Buffer + Depth）
│   ├── RT 反射代价：高（每像素发射光线，TraceRay 很贵）
│   └── 实用策略：Roughness < 0.3 用 RT，其余 fallback SSR
└── 性能预算（参考值）
    ├── PC（RTX 3060）：RT Reflections 4ms + RT Shadows 2ms + RT GI 6ms = ~12ms
    ├── PC（RTX 4090）：全功能 RT ~6ms
    └── 手游（骁龙 8 Gen 3）：RT Reflections ~5-8ms（只能选一个 RT 功能）
```

#### 代码实现

**DXR 1.1 简化示例：RT 反射 ClosestHit Shader**

```hlsl
// === DXR ClosestHit Shader: 反射光线 ===

// 全局加速结构
RaytracingAccelerationStructure g_sceneBVH : register(t0);
Texture2D<float4> g_gbufferAlbedo    : register(t1);
Texture2D<float4> g_gbufferNormal    : register(t2);
Texture2D<float>  g_gbufferDepth     : register(t3);
Texture2D<float>  g_gbufferRoughness : register(t4);

RWTexture2D<float4> g_reflectionOutput : register(u0);

cbuffer Constants : register(b0)
{
    float4x4 _InvViewProj;
    float4x4 _ViewProj;
    float2   _ScreenSize;
    uint     _FrameCount; // 用于时序抖动
};

struct RayPayload
{
    float3 color;
    float  hitDistance;
    bool   hitSky;
};

[shader("closesthit")]
void ClosestHitShader(inout RayPayload payload, Attributes attr)
{
    // 获取命中三角形数据
    uint instanceID = InstanceID();
    uint primitiveIndex = PrimitiveIndex();

    // 简化：从实例缓冲中取材质颜色
    float3 albedo = g_instanceColors[instanceID];
    payload.color = albedo;
    payload.hitDistance = RayTCurrent();
    payload.hitSky = false;
}

[shader("miss")]
void MissShader(inout RayPayload payload)
{
    // 未命中 → 采样天空盒
    payload.color = g_skybox.SampleLevel(g_linearClamp, Direction().xyz, 0).rgb;
    payload.hitSky = true;
    payload.hitDistance = 1000.0;
}

// === 主渲染 Pass：从 G-Buffer 发射反射光线 ===
[shader("raygeneration")]
void RayGenShader()
{
    uint2 pixel = DispatchRaysIndex().xy;
    float2 uv = (pixel + 0.5) / _ScreenSize;

    // 1. 从 G-Buffer 重建世界坐标
    float depth = g_gbufferDepth[pixel];
    if (depth >= 1.0) // 天空
    {
        g_reflectionOutput[pixel] = float4(0, 0, 0, 0);
        return;
    }

    float4 clipPos = float4(uv * 2.0 - 1.0, depth, 1.0);
    float4 worldPos = mul(_InvViewProj, clipPos);
    worldPos /= worldPos.w;

    // 2. 获取法线和粗糙度
    float3 normal = normalize(g_gbufferNormal[pixel].xyz * 2.0 - 1.0);
    float  roughness = g_gbufferRoughness[pixel];

    // Roughness 太高 → 不做 RT 反射，fallback 到 SSR / 不反射
    if (roughness > 0.6)
    {
        g_reflectionOutput[pixel] = float4(0, 0, 0, 0);
        return;
    }

    // 3. 计算反射方向
    float3 viewDir = normalize(_CameraPos.xyz - worldPos.xyz);
    float3 reflectDir = reflect(-viewDir, normal);

    // 时序抖动（用于时序降噪累积）
    float2 jitter = HaltonSequence(_FrameCount % 16);
    reflectDir = normalize(reflectDir + (jitter.x - 0.5) * roughness * 0.3);

    // 4. 发射光线
    RayDesc ray;
    ray.Origin = worldPos.xyz + normal * 0.01; // 偏移避免自交
    ray.Direction = reflectDir;
    ray.TMin = 0.01;
    ray.TMax = 100.0;

    RayPayload payload;
    TraceRay(g_sceneBVH,
             RAY_FLAG_CULL_BACK_FACING_TRIANGLES,
             0xFF, // InstanceMask
             0, 1, 0, // HitGroup index offsets
             ray, payload);

    // 5. 写入反射颜色（后续 Denoiser Pass 降噪）
    float3 finalColor = payload.color;
    // Fresnel 衰减
    float fresnel = pow(1.0 - saturate(dot(viewDir, normal)), 5.0);
    finalColor *= lerp(0.04, 1.0, fresnel); // Schlick Fresnel

    g_reflectionOutput[pixel] = float4(finalColor, 1.0);
}
```

**SVGF 降噪核心思路（伪代码）：**

```hlsl
// SVGF: Spatiotemporal Variance-Guided Filtering
// 核心思想：时域累积 + 空域双边滤波

// Step 1: 时域累积（历史帧复用）
float4 SVGF_TemporalAccumulation(uint2 pixel, float4 currentColor)
{
    // 获取上一帧对应像素（使用 Motion Vector 做重投影）
    float2 prevUV = uv - motionVector;
    float4 historyColor = historyBuffer.Sample(prevUV);

    // Disocclusion 检测：如果当前位置深度/法线和历史差异大 → 减少历史权重
    float depthDiff = abs(currentDepth - historyDepth);
    float normalDot = dot(currentNormal, historyNormal);
    bool disoccluded = depthDiff > 0.1 || normalDot < 0.8;

    float alpha = disoccluded ? 0.2 : 0.9; // 历史权重
    return lerp(currentColor, historyColor, alpha);
}

// Step 2: 空域滤波（基于方差的边缘保持滤波）
float4 SVGF_SpatialFilter(uint2 pixel)
{
    float variance = computeVariance(pixel, 5); // 5x5 方差
    float centerDepth = depthBuffer[pixel];
    float3 centerNormal = normalBuffer[pixel];

    float4 sum = 0;
    float totalWeight = 0;

    [loop] for (int x = -2; x <= 2; x++)
    {
        [loop] for (int y = -2; y <= 2; y++)
        {
            uint2 neighbor = pixel + int2(x, y);

            // 深度权重（拒绝不同深度的像素）
            float depthW = exp(-abs(depthBuffer[neighbor] - centerDepth) / sigmaDepth);

            // 法线权重（拒绝不同朝向的像素）
            float3 nN = normalBuffer[neighbor];
            float normalW = pow(max(dot(centerNormal, nN), 0), sigmaNormal);

            // 方差权重（方差大 → 噪声多 → 滤波更强）
            float varW = 1.0 / (1.0 + variance);

            float w = depthW * normalW * varW;
            sum += colorBuffer[neighbor] * w;
            totalWeight += w;
        }
    }
    return sum / totalWeight;
}
```

**PC vs 手游 RT 性能预算对比表：**

| 功能 | PC（RTX 3060） | PC（RTX 4090） | 手游（骁龙 8 Gen 3） | 手游（A17 Pro） |
|------|---------------|---------------|---------------------|----------------|
| RT Reflections | 3-4 ms | 1-2 ms | 5-8 ms | 4-6 ms |
| RT Shadows | 2-3 ms | 0.5-1 ms | 3-5 ms | 2-4 ms |
| RT GI (1 bounce) | 5-8 ms | 2-3 ms | 不推荐 | 不推荐 |
| RT AO | 1-2 ms | 0.5 ms | 2-3 ms | 1-2 ms |
| BVH 更新（动态） | 0.5-1 ms | 0.2 ms | 1-2 ms | 0.5-1 ms |
| Denoiser | 1-2 ms | 0.5 ms | 1-2 ms | 1 ms |
| **总预算建议** | **8-12 ms** | **4-6 ms** | **≤ 8 ms（选一个）** | **≤ 6 ms** |

### ⚡ 实战经验

- **不要全用 RT**：即使是 RTX 4090，也没有游戏把所有效果全 RT 化。混合管线是标准做法——RT 反射 + 光栅化阴影 + Lumen GI
- **RT 反射的 Roughness Cutoff 很重要**：Roughness > 0.3 的表面反射极其模糊，SSR 效果差不多但便宜 10 倍。设置 cutoff 是最有效的性能优化
- **BVH 内存是大头**：一个大型场景的 BVH 可能占几百 MB GPU 内存。手游上需要严格控制场景规模或只对角色和近景物体建 BVH
- **Denoiser 质量决定画面质量**：1 SPP 的原始 RT 画面全是雪花噪点，完全不能看。SVGF 类降噪是必须的，且需要处理 Disocclusion（角色走动时上一帧历史失效）
- **手游 RT 的实际落地**：目前最成熟的是 RT 反射（原神移动端在 A17 Pro 上试过），其次是 RT 阴影。GI 在移动端完全不建议
- **Debugging 建议**：PIX / RenderDoc 的 RT 可视化是调试神器，可以单步追踪每根光线的 BVH 遍历路径

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 说不清 RT Core 和 Compute Shader 的区别 | 不理解固定功能加速 | 学习 GPU 硬件架构（RT Core 硬件单元） |
| 不知道 BVH 是什么 | 加速结构概念缺失 | 学习 BVH 构建、遍历、TLAS/BLAS 分层 |
| Denoiser 说不清楚 | 时域累积原理不熟 | 学习 SVGF 论文 + 时域复用原理 |
| 不知道手游能不能跑 RT | 移动端硬件认知落后 | 调研骁龙 8 Gen 3 / A17 Pro RT 支持 |
| 混合管线搭不出来 | 渲染管线全局视角缺失 | 研究 Control / Cyberpunk / RE4 的 RT 管线拆解 |

### 🔗 相关问题

- UE5 Lumen 的 GI 和 DXR RT GI 是同一回事吗？（提示：Lumen 是 Software RT + Screen Space GI 混合方案，不完全是硬件 RT）
- 如果不用硬件 RT，能不能用 Compute Shader 模拟光线追踪？代价是什么？
- DLSS 3 的 Frame Generation 在 RT 场景下为什么特别有效？（提示：RT 是 GPU 瓶颈，Frame Generation 缓解 GPU 压力）
