---
title: "远山为什么发蓝？大世界大气散射渲染从原理到落地"
category: "rendering"
level: 3
tags: ["大气散射", "Rayleigh散射", "Mie散射", "天空盒", "大气渲染", "大世界", "URP"]
hint: "远山发蓝不是贴图——是大气 Rayleigh 散射的效果，蓝光波长短被散射更多；理解物理模型才能做出正确的天空和雾"
related: ["rendering/urp-volumetric-fog", "rendering/volumetric-cloud-rendering", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官给你看两张开放世界游戏截图——一张是《塞尔达：旷野之息》，远山层次分明，从近处的翠绿到远处的蓝紫色，天空有自然的渐变和光晕；另一张是你们项目的大世界，远山和近山颜色一样，天空盒是贴死的纯渐变，完全flat。然后问：

> "我们的大世界缺乏空间纵深感，远处的山看起来和近处一样。美术调了好几次雾色和天空盒都不对。你是 TA，给我一个物理化的大气渲染方案——要能做出那种'空气感'，在 URP 下可落地，移动端也能跑。"

这是腾讯天美、网易雷火、米哈游等大世界项目 TA 岗的高频题，考察的是**物理化天空 + 大气散射 + 与现有雾效融合的全链路理解**。

### ✅ 核心要点

1. **远山发蓝是物理现象，不是美术风格**：大气中氮氧分子对短波长（蓝光）散射更强（Rayleigh 散射），远处物体经过更厚的大气层，蓝光被散射到视线中
2. **两种散射要分开处理**：Rayleigh（分子级，蓝天来源）和 Mie（气溶胶级，雾/霾来源），物理模型不同
3. **天空不是天空盒——是体积**：正确的天空是从摄像机射线穿过大气层的积分结果，预烘焙 CubeMap 只是移动端降级方案
4. **大气散射和场景雾必须统一**：很多项目天空用物理散射、场景雾用 Unity Linear Fog，两者颜色不匹配，远景出现明显接缝
5. **移动端可用分析模型 + 查找表**：不用做完整的 Ray Marching，用解析公式（Precomputed Atmospheric Scattering 的多项式拟合）+ LUT 查找

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：远山蓝紫色渐变，天空物理渐变，日出日落颜色变化自然
     ↓ 倒推
大气散射 = 物体原色 × 透射率（Transmittance）+ 散射光（In-scattering）
     ↓ 倒推
透射率：光线从物体到摄像机，穿过大气层时被吸收/散射掉了多少
  ├── Rayleigh 透射率：exp(-β_r × density_r × distance)
  └── Mie 透射率：exp(-β_m × density_m × distance)
     ↓ 倒推
散射光：沿途大气分子把太阳光散射到摄像机方向的光量
  ├── 相位函数描述散射方向性：
  │   ├── Rayleigh 相位函数：(3/(16π)) × (1 + cos²θ)
  │   └── Mie 相位函数：Henyel-Greenstein（g 参数控制前向/后向）
  ├── 太阳方向 × 大气密度 × 相位函数 → 每段路径的散射贡献
  └── 沿视线积分 → 最终 in-scattering 颜色
     ↓ 倒推
大气密度随高度衰减：
  ├── 密度 = exp(-height / scaleHeight)
  ├── Rayleigh scale height ≈ 8.4km（分子散射层）
  └── Mie scale height ≈ 1.2km（气溶胶层，更低更薄）
     ↓ 倒推
工程实现方案分级：
  ├── PC/主机：Bruneton & Precomputed Atmospheric Scattering（完整积分）
  ├── 移动端方案 A：Hosek-Wilkie 分析模型（解析公式，无 LUT 查找）
  ├── 移动端方案 B：预计算 Transmittance LUT + 散射 LUT（运行时 2 纹理采样）
  └── 超低端：CubeMap 天空 + 高度雾近似（无物理散射）
```

#### 知识点拆解（倒推树）

```
大气散射渲染
├── 物理基础
│   ├── Rayleigh 散射（分子散射）
│   │   ├── 波长依赖性 ∝ 1/λ⁴（蓝光散射为红光 16 倍）
│   │   ├── 相位函数：各向同性（前后散射相等）
│   │   └── Scale Height ≈ 8.4km
│   ├── Mie 散射（气溶胶散射）
│   │   ├── 波长依赖性弱（白/灰雾来源）
│   │   ├── 相位函数：Henyel-Greenstein（g>0 前向散射为主）
│   │   └── Scale Height ≈ 1.2km
│   └── 臭氧吸收（日落时绿色被吸收 → 红橙色天空）
├── 实现方案
│   ├── 方案 A：Precomputed A.T. Scattering (Bruneton 2008)
│   │   ├── 预计算 Transmittance LUT（2D 纹理：高度 × 视角天顶角）
│   │   ├── 预计算 Single/Multi-scatter LUT（4D → 2D 参数化）
│   │   ├── 运行时：采样 LUT + 太阳方向 → 天空颜色
│   │   └── 优点：物理正确；缺点：预计算耗时，LUT 占内存
│   ├── 方案 B：Hosek-Wilkie 分析模型
│   │   ├── 解析公式拟合（无 LUT，纯 ALU）
│   │   ├── 添加彩虹/光环等高级大气现象
│   │   └── 移动端友好：~40 ALU 指令
│   └── 方案 C：简化高度雾近似（移动端保底）
│       ├── 透射率 = exp(-β × height_factor × distance)
│       ├── 散射 = sun_color × haze_density × phase_function
│       └── 总颜色 = lerp(object_color, sky_tint, 1 - transmittance)
├── 天空盒渲染
│   ├── 全屏三角形 + 大气散射 Shader（最现代的做法）
│   ├── CubeMap 预烘焙（Hosek 模型离线渲染到 CubeMap）
│   ├── 动态时间-of-day 天空：每帧或每隔几帧更新 CubeMap
│   └── 太阳/月亮精灵叠加
├── 场景雾融合
│   ├── 问题：Unity 内置雾是固定颜色，和散射天空不匹配
│   ├── 方案：自定义 Fog Shader，用同一套大气参数计算雾色
│   ├── 高度雾 + 距离雾混合
│   └── 雾色 = 天空该方向的 in-scattering 颜色（保证远景过渡到天空无接缝）
└── 性能优化
    ├── LUT 精度选择（移动端用 RGBA16F，PC 用 RGBA32F）
    ├── 天空 Shader 全屏开销控制（只做 1 次视线积分，不做 Ray Marching）
    ├── 静态天空缓存（时间不变时用 RenderTexture 缓存）
    └── 与 Volumetric Fog 的融合（共用 Froxel 结构）
```

#### 代码实现

**移动端简化大气散射 Shader（分析模型，无 LUT 依赖）：**

```hlsl
// AtmosphericScattering.shader
// 基于简化 Rayleigh + Mie 模型，适合移动端
// 全屏后处理：对场景深度做大气散射计算

Shader "Hidden/Custom/AtmosphericScattering"
{
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline"}
        Cull Off ZWrite Off ZTest Always

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 viewDir : TEXCOORD1; // 世界空间视线方向
            };

            TEXTURE2D(_MainTex); SAMPLER(sampler_MainTex);
            TEXTURE2D(_CameraDepthTexture); SAMPLER(sampler_CameraDepthTexture);

            // 大气参数（可由 Volume Component 控制）
            float3 _RayleighCoeffs;    // Rayleigh 散射系数 (波长依赖)
            float3 _MieCoeffs;         // Mie 散射系数
            float  _RayleighScaleH;    // Rayleigh scale height（通常 8400m）
            float  _MieScaleH;         // Mie scale height（通常 1200m）
            float3 _SunDirection;      // 太阳方向（归一化）
            float3 _SunIntensity;      // 太阳光颜色 × 强度
            float  _PlanetRadius;      // 行星半径（地球 ~6371km）
            float  _AtmosphereHeight;  // 大气层厚度（~100km）

            v2f vert(float3 pos : POSITION, float2 uv : TEXCOORD0)
            {
                v2f o;
                o.pos = TransformObjectToHClip(pos);
                o.uv = uv;

                // 重建世界空间视线方向
                float3 worldPos = ComputeWorldSpacePosition(
                    o.uv, 1.0, UNITY_MATRIX_I_VP
                );
                o.viewDir = normalize(worldPos - _WorldSpaceCameraPos);
                return o;
            }

            // Rayleigh 相位函数（各向同性，与方向几乎无关）
            float RayleighPhase(float cosTheta)
            {
                return 0.0596831 * (1.0 + cosTheta * cosTheta);
                // = 3/(16π) × (1 + cos²θ)
            }

            // Henyel-Greenstein Mie 相位函数
            float MiePhase(float cosTheta, float g)
            {
                float g2 = g * g;
                return 0.1193662 * (1.0 - g2) /
                       pow(1.0 + g2 - 2.0 * g * cosTheta, 1.5);
                // = (3/(8π)) × (1-g²) / (1+g²-2g·cosθ)^1.5
            }

            // 指数高度衰减
            float DensityAtHeight(float height, float scaleHeight)
            {
                return exp(-height / scaleHeight);
            }

            // 简化大气散射计算（分析模型，不做数值积分）
            float3 ComputeAtmosphericScattering(
                float3 viewDir,
                float3 worldPos,
                out float3 transmittance
            ) {
                float3 camPos = _WorldSpaceCameraPos;
                float viewDistance = length(worldPos - camPos);

                // 高度近似（假设平面地球，大世界足够用）
                float avgHeight = camPos.y * 0.5 + worldPos.y * 0.5;
                avgHeight = max(avgHeight, 0.0); // 不允许负高度

                // 大气密度（Rayleigh + Mie）
                float rayleighDensity = DensityAtHeight(avgHeight, _RayleighScaleH);
                float mieDensity = DensityAtHeight(avgHeight, _MieScaleH);

                // 光学深度（简化：密度 × 距离）
                float3 rayleighOpticalDepth = _RayleighCoeffs * rayleighDensity * viewDistance;
                float3 mieOpticalDepth = _MieCoeffs * mieDensity * viewDistance;

                // 透射率：Beer-Lambert 定律
                transmittance = exp(-(rayleighOpticalDepth + mieOpticalDepth));

                // 太阳方向与视线的夹角
                float cosTheta = dot(viewDir, _SunDirection);

                // 相位函数
                float rayleighPhase = RayleighPhase(cosTheta);
                float miePhase = MiePhase(cosTheta, 0.758); // g ≈ 0.76 为典型晴空值

                // In-scattering：太阳光被散射到视线方向
                // 简化模型：忽略太阳光方向的透射损失
                float3 rayleighScatter = _RayleighCoeffs * rayleighDensity
                                       * rayleighPhase * _SunIntensity;
                float3 mieScatter = _MieCoeffs * mieDensity
                                  * miePhase * _SunIntensity;

                // 总散射光 = (Rayleigh + Mie) / (Rayleigh + Mie 光学深度)
                // 使用近似公式避免除零
                float3 totalScatter = rayleighScatter + mieScatter;
                float3 totalExtinction = rayleighOpticalDepth + mieOpticalDepth + 0.0001;

                float3 inScattering = totalScatter * (1.0 - exp(-totalExtinction)) / totalExtinction;

                return inScattering;
            }

            float4 frag(v2f i) : SV_Target
            {
                float3 sceneColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv).rgb;

                // 采样深度，重建世界坐标
                float depth = SAMPLE_TEXTURE2D(_CameraDepthTexture, sampler_CameraDepthTexture, i.uv).r;
                depth = LinearEyeDepth(depth, _ZBufferParams);

                // 天空像素（深度 = 1.0 / 远裁面）使用固定大气距离
                float3 viewDir = normalize(i.viewDir);
                float3 worldPos;

                if (depth >= 0.999)
                {
                    // 天空：投射到大气层顶部
                    worldPos = _WorldSpaceCameraPos + viewDir * _AtmosphereHeight * 1000.0;
                }
                else
                {
                    // 场景物体：正常深度重建
                    worldPos = ComputeWorldSpacePosition(i.uv, depth, UNITY_MATRIX_I_VP);
                }

                float3 transmittance;
                float3 inScatter = ComputeAtmosphericScattering(viewDir, worldPos, transmittance);

                // 最终合成：原色 × 透射率 + 散射光
                float3 finalColor = sceneColor * transmittance + inScatter;

                return float4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**天空盒渲染（单独 Pass，全屏三角面 + 大气散射）：**

```hlsl
// SkyAtmosphere.shader — 渲染到天空（深度 = far plane 的像素）

Shader "Custom/SkyAtmosphere"
{
    SubShader
    {
        Tags { "RenderType"="Background" "RenderPipeline"="UniversalPipeline"
               "PreviewType"="Skybox" }
        Cull Off ZWrite Off

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            float3 _SunDirection;
            float3 _SunIntensity;
            float3 _RayleighCoeffs;
            float3 _MieCoeffs;
            float  _AtmosphereHeight;

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 viewDir : TEXCOORD0;
            };

            v2f vert(float3 pos : POSITION)
            {
                v2f o;
                // 天空盒直接用顶点位置作为视线方向
                o.pos = TransformObjectToHClip(pos);
                o.viewDir = normalize(mul((float3x3)unity_ObjectToWorld, pos));
                return o;
            }

            // 简化天空颜色（Hosek-like 近似）
            float3 ComputeSkyColor(float3 viewDir)
            {
                float cosTheta = dot(viewDir, _SunDirection);
                float cosSunZenith = _SunDirection.y; // 太阳高度角

                // 天顶 → 地平线的渐变因子
                float zenithFactor = saturate(viewDir.y);
                float horizonFactor = 1.0 - zenithFactor;

                // 天空基色（天顶蓝 → 地平线浅蓝白）
                float3 zenithColor = _RayleighCoeffs * 8.0;
                float3 horizonColor = _RayleighCoeffs * 3.0 + _MieCoeffs * 2.0;
                float3 skyTint = lerp(horizonColor, zenithColor, zenithFactor);

                // 太阳光晕（视线靠近太阳方向时增亮）
                float sunGlow = pow(max(cosTheta, 0.0), 32.0);
                float3 sunColor = _SunIntensity * sunGlow * 0.5;

                // 日出/日落：太阳低时红色增强
                float sunsetFactor = saturate(1.0 - cosSunZenith * 5.0);
                float3 sunsetTint = float3(1.5, 0.5, 0.2) * sunsetFactor * horizonFactor;

                return skyTint + sunColor + sunsetTint;
            }

            float4 frag(v2f i) : SV_Target
            {
                float3 skyColor = ComputeSkyColor(normalize(i.viewDir));

                // 太阳本体（小亮点）
                float sunDisk = smoothstep(0.9995, 0.9998,
                    dot(normalize(i.viewDir), _SunDirection));
                skyColor += _SunIntensity * sunDisk * 10.0;

                return float4(skyColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

### ⚡ 实战经验

**坑 1：场景雾色与天空不匹配**
> 项目用 Unity 内置雾（固定颜色），天空用自定义大气散射 Shader。结果地平线处场景雾是灰色，天空是蓝紫色，接缝极其明显。解法：自定义场景雾 Shader 用同一套大气参数计算雾色，保证 `fog_color = sky_inscatter_color_at_horizon`。

**坑 2：Rayleigh 散射系数反直觉**
> 一开始直觉认为散射系数应该是 RGB 相等的，结果天空不蓝。实际上 Rayleigh 散射系数与波长 4 次方成反比：β_r = (5.8, 13.5, 33.1)×10⁻⁶（红/绿/蓝），蓝色是红色的 ~5.7 倍。这个比例不对，天空颜色就不对。

**坑 3：移动端太阳光晕过亮**
> `pow(max(cosTheta, 0.0), 32.0)` 中的指数太小（如 8），太阳周围会出现大面积过亮黄白区域。手机屏幕亮度本来就高，效果更夸张。移动端建议指数 ≥ 128，让太阳光晕更集中。

**坑 4：时间-of-day 系统与大气散射的同步**
> 昼夜交替系统更新太阳方向时，如果只更新 Directional Light 的 rotation，但大气散射 Shader 的 `_SunDirection` 参数没同步更新，会出现"天空太阳在左边，场景影子在右边"的诡异画面。确保 TimeOfDay 系统在同一帧内同时更新 Light 和 Material 参数。

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明 |
|--------|-----------|-----------|
| 为什么远山是蓝色的？用物理原理解释。 | 理解 Rayleigh 散射原理 | 缺少大气物理基础 |
| Rayleigh 散射和 Mie 散射的区别是什么？ | 理解两种散射模型 | 只知道"有雾" |
| 天空盒用贴图和用物理散射有什么区别？ | 理解物理化天空优势 | 缺少天空渲染方案视野 |
| Beer-Lambert 定律在大气散射中的作用？ | 理解透射率计算 | 缺少光学基础 |
| 移动端大气散射的预算是多少 ms？ | 有移动端性能基线 | 缺少移动端工程经验 |
| 大气散射和场景雾如何统一？ | 有全链路融合思维 | 只会单一系统配置 |
| Hosek-Wilkie 和 Bruneton 方案的区别？ | 理解不同级别的实现方案 | 缺少方案对比知识 |

### 🔗 相关问题

- [URP 体积雾实现](../rendering/urp-volumetric-fog) — 大气散射与体积雾的融合方案
- [实时体积云渲染](../rendering/volumetric-cloud-rendering) — 云层与大气散射的交互
- [GPU 带宽优化](../optimization/gpu-bandwidth-optimization) — LUT 纹理的带宽控制
