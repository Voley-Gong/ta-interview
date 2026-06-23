---
title: "URP下如何实现体积雾？面试官要你能落地而非只会说原理"
category: "rendering"
level: 3
tags: ["体积雾", "Volumetric Fog", "URP", "Ray Marching", "Froxeling", "屏幕空间"]
hint: "移动端用高度雾+噪声扰动近似，PC/主机用 Froxel 3D 纹理体积雾——关键是理解两者的性能边界"
related: ["rendering/custom-post-processing-urp", "optimization/gpu-bandwidth-optimization", "shader/screen-space-rain-droplet"]
---

## 参考答案

### 🎬 场景描述

面试官给你看两张游戏截图——一张是干瘪的纯像素雾（Unity 内置 Linear Fog），另一张是《对马岛之魂》那种有体积感、光线穿透、随高度衰减的雾。然后问：

> "我们的 URP 项目现在雾效很平，没有体积感。策划想要那种光柱穿过雾层的效果（God Rays）。你是 TA，给我一个能在 URP 下落地的体积雾方案，目标平台是 PS5/PC，但也要能降级到移动端。"

这是腾讯天美、网易雷火等 3A 项目 TA 岗的高频题，考察的是**对渲染管线全局理解 + 性能分级思维**。

### ✅ 核心要点

1. **体积雾本质**：光在大气中与微粒发生散射（Mie 散射为主），光路越长、散射越多、雾越浓——不是简单的深度 fade
2. **Froxel 体积雾是主流方案**：将视锥体划分为 3D 格子（Froxel），每格存雾的密度和散射光，是当前 3A 标准
3. **高度雾 + 屏幕空间噪声是移动端降级方案**：用指数高度雾函数 + 3D Noise 扰动 + Bloom 合成伪 God Rays
4. **光照交互是加分项**：体积雾必须能响应主光源（方向光穿透）、点光源（灯光照亮雾气）
5. **性能红线**：Froxel 体积雾在 PS5 约 1.5-2ms，移动端高度雾方案不超过 0.5ms

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：有体积感的雾，光柱穿透，高度衰减，雾中灯光有光晕
     ↓ 倒推
体积感来源 = 沿视线方向对每个像素做 Ray Marching，累积散射光
     ↓ 倒推
高效 Ray Marching = Froxel 体积雾
  ├── Step 1：将视锥体划分为 3D 纹理（如 160×90×64）
  ├── Step 2：每帧计算每个 Froxel 的雾密度（高度+噪声）
  ├── Step 3：注入光源散射（主光+点光）到每个 Froxel
  ├── Step 4：在光照 Pass 中对 Froxel 体积纹理做 3 次采样（前/中/后）
  └── Step 5：最终合成到画面
     ↓ 倒推
移动端降级 = 不做 Ray Marching，用后处理近似
  ├── 深度重建世界坐标 → 计算高度衰减
  ├── 3D Noise 采样 → 模拟体积感扰动
  └── 屏幕 God Rays 径向模糊 → 伪光柱
     ↓ 倒推
URP 集成 = Custom Renderer Feature + Compute Shader（PC）/ Fragment（移动端）
```

#### 知识点拆解（倒推树）

```
URP 体积雾
├── 核心原理
│   ├── 光散射物理模型（Mie/Rayleigh 散射公式简化）
│   ├── 指数高度雾函数：fogDensity = baseDensity * exp(-height * falloff)
│   ├── Beer-Lambert 定律：透射率 = exp(-σt × distance)
│   └── Ray Marching 累积积分（数值积分步长权衡）
├── Froxel 体积雾实现
│   ├── 视锥体分块（深度对数分布，近密远疏）
│   ├── 3D Render Texture 创建与格式选择（RGBA Half）
│   ├── Compute Shader 计算密度场与光照注入
│   ├── 光照注入：方向光、点光源列表传给 Compute
│   ├── 时间复用（Temporal Reprojection）降噪与稳定性
│   └── 采样合成：三线性采样 + 对齐深度
├── 移动端降级方案
│   ├── 全屏后处理：Custom Pass (Before Post Process)
│   ├── 深度重建世界坐标（_CameraDepthTexture → WS Position）
│   ├── 高度雾计算 + 3D Noise 动画
│   ├── 屏幕 God Rays 径向模糊（从屏幕空间光源方向）
│   └── 性能优化：半分辨率 RT + 双线性上采样
├── URP 工程实现
│   ├── ScriptableRendererFeature 注册 Render Pass
│   ├── Compute Shader vs Fragment Shader 平台分支
│   ├── 材质参数暴露（雾色、密度、高度衰减、噪声速度）
│   └── 多平台 Quality Settings 切换
└── 与后处理交互
    ├── Bloom 与雾的高光交互
    ├── 雾对 DOF（景深）的影响（先雾后景深 vs 先景深后雾）
    └── 色调映射后的雾颜色校正
```

#### 代码实现

**PC/主机：Froxel 体积雾 Compute Shader 核心**

```hlsl
// VolumetricFogFroxlize.compute
#pragma kernel CSMain

RWTexture3D<float4> _FogVolume;     // 3D 体积纹理
Texture2D<float> _CameraDepthTexture;
float4x4 _InvViewProjMatrix;
float3 _LightDir;
float3 _LightColor;
float _FogDensity;
float _FogHeightFalloff;
float _Time;

// 采样雾密度（高度 + 噪声）
float SampleFogDensity(float3 worldPos)
{
    float heightFactor = exp(-worldPos.y * _FogHeightFalloff);
    float noise = tex3D(_FogNoise, worldPos * 0.01 + _Time * 0.1).r;
    return _FogDensity * heightFactor * (0.6 + 0.4 * noise);
}

// 光沿视线方向散射（Henyey-Greenstein 相位函数简化）
float PhaseFunction(float cosTheta)
{
    float g = 0.6; // 前向散射偏移
    return (1 - g*g) / pow(1 + g*g - 2*g*cosTheta, 1.5) * 0.25;
}

[numthreads(8, 8, 1)]
void CSMain(uint3 id : SV_DispatchThreadID)
{
    // Froxel 坐标 → 世界坐标（通过逆矩阵）
    float2 uv = (id.xy + 0.5) / _FogVolumeSize.xy;
    float depth = DepthSliceToLinearEye(id.z); // 对数深度分布
    float3 worldPos = UVDepthToWorld(uv, depth, _InvViewProjMatrix);

    // 计算雾密度和散射
    float density = SampleFogDensity(worldPos);

    // 主光散射贡献
    float NdotL = dot(normalize(_LightDir), normalize(-_CameraForward));
    float scattering = PhaseFunction(NdotL) * density;

    // 点光源贡献（循环遍历场景点光列表）
    float3 pointLightContribution = 0;
    for (int i = 0; i < _PointLightCount; i++)
    {
        float3 toLight = _PointLights[i].position - worldPos;
        float dist = length(toLight);
        float atten = 1.0 / (1.0 + dist * dist);
        pointLightContribution += _PointLights[i].color * atten * density;
    }

    float3 totalColor = _LightColor * scattering + pointLightContribution;

    _FogVolume[id] = float4(totalColor, density);
}
```

**移动端降级方案：屏幕空间高度雾 + God Rays**

```hlsl
// MobileVolumetricFog.shader (Full Screen Pass)
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

float _FogDensity;
float _FogHeightFalloff;
float3 _FogColor;
float3 _SunDir;       // 屏幕空间太阳方向
float4x4 _InvViewProj;

// 3D Noise（程序化，无需纹理）
float hash(float3 p) {
    p = frac(p * 0.3183099 + 0.1);
    p *= 17.0;
    return frac(p.x * p.y * p.z * (p.x + p.y + p.z));
}

float noise3D(float3 p) {
    float3 i = floor(p);
    float3 f = frac(p);
    f = f * f * (3.0 - 2.0 * f);
    return lerp(
        lerp(lerp(hash(i), hash(i + float3(1,0,0)), f.x),
             lerp(hash(i + float3(0,1,0)), hash(i + float3(1,1,0)), f.x), f.y),
        lerp(lerp(hash(i + float3(0,0,1)), hash(i + float3(1,0,1)), f.x),
             lerp(hash(i + float3(0,1,1)), hash(i + float3(1,1,1)), f.x), f.y), f.z);
}

half4 FragFog(Varyings input) : SV_Target
{
    // 1. 深度重建世界坐标
    float depth = SampleSceneDepth(input.uv);
    #if UNITY_REVERSED_Z
        depth = 1.0 - depth;
    #endif
    float3 worldPos = ComputeWorldSpacePosition(input.uv, depth, UNITY_MATRIX_I_VP);

    // 2. 指数高度雾
    float heightFactor = exp(-worldPos.y * _FogHeightFalloff);
    float noise = noise3D(worldPos * 0.5 + _Time.y * 0.3);
    float fogAmount = _FogDensity * heightFactor * (0.7 + 0.3 * noise);

    // 3. 遮挡系数（基于深度的距离雾）
    float viewDist = length(worldPos - _WorldSpaceCameraPos);
    float occlusion = 1.0 - exp(-fogAmount * viewDist * 0.01);

    return half4(_FogColor * occlusion, occlusion);
}
```

**God Rays 径向模糊（屏幕空间光柱）**

```hlsl
// GodRays.shader — 在雾之后，Bloom 之前执行
// 从屏幕空间太阳投影点做径向采样
half4 FragGodRays(Varyings input) : SV_Target
{
    float2 sunPos = _SunScreenPos.xy; // 太阳在屏幕空间的投影 NDC 坐标
    float2 dir = input.uv - sunPos;
    dir *= _Density; // 采样密度

    half4 color = 0;
    float totalWeight = 0;

    // 径向采样 16 次
    [unroll]
    for (int i = 0; i < 16; i++)
    {
        float2 offset = dir * (i / 16.0);
        // 采样雾颜色纹理（需要雾最亮的部分 → 光柱来源）
        half4 sample = SAMPLE_TEXTURE2D(_FogTexture, sampler_FogTexture, input.uv - offset);
        sample.rgb *= _SunColor * _GodRayIntensity;
        color += sample;
        totalWeight += 1.0;
    }

    return color / totalWeight;
}
```

### ⚡ 实战经验

1. **Froxel 精度陷阱**：深度分块用对数分布（近处密集、远处稀疏），线性分布会导致远处雾抖动
2. **Temporal 抖动是必须的**：每帧偏移采样位置 + 历史帧混合，否则 Froxel 格子边界可见
3. **移动端 God Rays 性价比极高**：只需要 1 个全屏径向模糊 Pass，0.2ms 换来的氛围感远超投入
4. **雾和半透明物体的渲染顺序**：体积雾应在半透明之前合成，否则半透明粒子（烟雾、火焰）会被雾错误遮挡
5. **不要在 Shader 里算 exp**：移动端 Mali GPU 对 exp 运算极慢，预计算到 LUT 纹理采样更快
6. **调试技巧**：用 `Visualize Froxel Grid` 调试模式，确认分块是否正确覆盖视锥
7. **与天气系统联动**：雨天的雾密度 ×1.5~2.0，颜色偏灰蓝；晴天的雾密度 ×0.3，颜色偏暖

### 🎯 能力体检清单

- [ ] 能否手写指数高度雾函数？理解 σt（消散系数）和Beer-Lambert定律？
- [ ] Froxel 3D 纹理的分辨率如何确定？太大/太小各有什么问题？
- [ ] 时间复用（Temporal Reprojection）的具体实现？如果场景中有快速移动的物体，历史帧数据如何失效？
- [ ] 如何在体积雾中支持探照灯（Spot Light）锥体？需要哪些额外数据？
- [ ] 移动端如果不能用 Compute Shader，如何用 Fragment Shader 模拟体积雾？性能瓶颈在哪？
- [ ] 体积雾与 SSR、DOF 等后处理效果的执行顺序如何安排？为什么？

### 🔗 相关问题

- [URP 自定义后处理 Pass](../rendering/custom-post-processing-urp.md)
- [GPU 带宽优化](../optimization/gpu-bandwidth-optimization.md)
- [屏幕空间雨滴效果](../shader/screen-space-rain-droplet.md)
