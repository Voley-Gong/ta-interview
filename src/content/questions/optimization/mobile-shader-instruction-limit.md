---
title: "手机 GPU 着色器指令数超标：Pixel Shader 占用 70% 帧时间怎么压？"
category: "optimization"
level: 3
tags: ["Shader优化", "ALU", "指令数", "PixelShader", "移动端", "Mali/Adreno", "数理分析"]
hint: "不是减一个光照就完了——要定位是 ALU 还是纹理采样瓶颈，再决定从数学等价替换、预计算、LOD 哪个维度下手"
related: ["optimization/mobile-gpu-occupancy-bottleneck", "optimization/shader-variant-explosion", "technical-art/shader-lod-quality-tier-system"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们手游项目在中端机型（骁龙 7 Gen 1）上 GPU 帧时间 22ms（目标 16.6ms），Snapdragon Profiler 显示 Pixel Shader 阶段占了 14ms。角色材质用了 PBR + 半球光照 + Rim Light + Detail Normal Map + SSS 近似，效果很好但太贵了。

现在你作为 TA 要把这个 Pixel Shader 从 14ms 压到 7ms 以下，告诉我你的分析方法和优化手段。」

### ✅ 核心要点

1. **先定位瓶颈类型**：ALU-bound（算力）vs Texture-bound（带宽）vs Register-bound（寄存器压力）
2. **数学等价替换**：用更便宜的近似函数替代昂贵运算（pow→exp2、normalize→rsqrt、sin→多项式）
3. **光照模型精简**：PBR 完整公式 → 移动端简化版（去除高次幂菲涅尔、合并 Diffuse+Specular）
4. **预计算与查找表**：运行时复杂的数学运算烘焙成 LUT 纹理采样
5. **Shader LOD 策略**：高端机完整效果、低端机砍层（去 SSS、去 Detail Normal）
6. **寄存器优化**：减少临时变量数量，降低 register pressure → 提升 GPU 占用率

### 📖 深度展开

#### 解决思路（从 Profile 数据倒推优化路径）

```
14ms Pixel Shader 费用
  ↓
Step 1：分析热力图——哪些指令周期占比最高？
  ├── 发现1：normalize() 调用 12 次 → 每次 2 周期，共 24 周期
  ├── 发现2：pow(N, 5) 菲涅尔项 → 每次 ~10 周期
  ├── 发现3：3 张纹理采样 × 4 cycles = 12 cycles（不是主瓶颈）
  └── 发现4：寄存器 32 个 → 占用率 50%（理论上限 64）
  ↓
Step 2：按 ROI 排序优化
  ├── 优化1：normalize → rsqrt 替代（省 50% ALU）  → 预计省 2ms
  ├── 优化2：pow(N,5) → 用 Schlick 菲涅尔近似      → 预计省 1.5ms
  ├── 优化3：SSS 近似 → 预烘焙 LUT 采样            → 预计省 3ms
  ├── 优化4：临时变量复用，减少寄存器到 24 个        → 预计省 1ms（占用率提升）
  └── 优化5：Detail Normal → 只在 LOD0 使用          → 预计省 1ms（LOD1+）
  ↓
预期总节省：~8.5ms → Pixel Shader 降到 ~5.5ms ✅ 达标
```

#### 知识点拆解（倒推树）

```
移动端 Shader 指令优化
├── 瓶颈定位
│   ├── Snapdragon Profiler（Adreno）：Pixel ALU / Texture / Memory 周期拆解
│   ├── Mali Streamline / Offline Compiler：读取寄存器数、线程占用率
│   ├── ARM Mobile Studio：Mali Core 性能计数器
│   └── 引擎内 Profiler：RenderDoc GPU Trace 时间线
├── ALU 优化（算力型瓶颈）
│   ├── normalize(v) → v * rsqrt(dot(v,v))  // 省 1 周期
│   ├── pow(x, n) → exp2(log2(x) * n) 或预计算常数
│   ├── Schlick 菲涅尔近似：F0 + (1-F0) * (1-cosθ)^5  // 比 Cook-Torrance 便宜
│   ├── 消除冗余 normalize：复用已 normalize 的向量
│   ├── half 代替 float（移动端 half 有硬件加速）
│   ├── 分支消除：用 step/lerp 替代 if-else
│   └── 多项式近似 sin/cos（低精度场景）
├── 纹理采样优化（带宽型瓶颈）
│   ├── 合并纹理：AO + Metallic + Smoothness → RGBA 一张
│   ├── 降低纹理分辨率（Detail Normal 可以 128x128）
│   ├── 减少采样次数：双线性滤波代替多次采样取平均
│   └── 纹理预计算（BRDF LUT、环境光 irradiance map）
├── 寄存器优化（占用率瓶颈）
│   ├── 减少同时存活的临时变量
│   ├── 大循环展开可能反而降低占用率（寄存器溢出）
│   ├── half4 打包代替 4 个 float
│   └── 检查 Compiler 日志的 "spill" 警告
├── Shader LOD 分级
│   ├── Tier 0（旗舰）：PBR + SSS + Detail Normal（完整）
│   ├── Tier 1（中端）：PBR 简化 + Rim Light（去 SSS）
│   ├── Tier 2（低端）：Half Lambert + 纹理 + 卡通描边
│   └── 通过 #pragma multi_compile 或材质质量设置切换
└── 编译器提示
    ├── [branch] / [flatten] 属性控制分支编译行为
    ├── [loop] / [unroll] 控制循环展开
    └── 检查最终汇编指令数（Unity Shader Compiler 输出）
```

#### 代码实现

**优化前（完整 PBR 角色 Shader 核心片段）：**

```hlsl
// ❌ 优化前：完整 PBR，~180 ALU 指令
half3 CalculateLighting(half3 N, half3 V, half3 L, half3 albedo,
    half metallic, half roughness, half3 lightColor)
{
    half3 H = normalize(L + V);              // normalize #1
    half3 NdotL = max(dot(N, L), 0);
    half3 NdotH = max(dot(N, H), 0);
    half3 NdotV = max(dot(N, V), 0);
    half3 VdotH = max(dot(V, H), 0);

    // Cook-Torrance BRDF（昂贵）
    half D = GGX_D(NdotH, roughness);         // 包含 pow(NdotH, 32)
    half G = Smith_G(NdotV, NdotL, roughness); // 包含 2 次 normalize
    half3 F = Fresnel_Schlick(VdotH, F0);     // pow(VdotH, 5)

    half3 specular = (D * G * F) / (4.0 * NdotV * NdotL + 0.001);

    // Lambert Diffuse
    half3 diffuse = albedo * NdotL / PI;

    return (diffuse + specular) * lightColor;
}
```

**优化后（移动端精简版，~60 ALU 指令）：**

```hlsl
// ✅ 优化后：精简 PBR，~60 ALU 指令
half3 CalculateLighting_Fast(half3 N, half3 V, half3 L, half3 albedo,
    half metallic, half roughness, half3 lightColor)
{
    // 合并向量计算，减少 normalize 调用
    half3 H = normalize(L + V);              // 只保留一次 normalize
    half NdotL = saturate(dot(N, L));
    half NdotH = saturate(dot(N, H));
    half NdotV = saturate(dot(N, V));
    half HdotV = saturate(dot(H, V));

    // --- 优化1：Schlick 菲涅尔近似（避免 pow） ---
    // pow(HdotV, 5) → 5 次乘法等价
    half F0 = lerp(0.04, 1.0, metallic);
    half Fv = 1.0 - HdotV;
    half Fv2 = Fv * Fv;
    half Fv4 = Fv2 * Fv2;
    half3 F = F0 + (1.0 - F0) * Fv4 * Fv;    // Schlick: 5 次乘法 << pow 的 10+ 周期

    // --- 优化2：GGX 近似（避免倒数和 pow） ---
    half a2 = roughness * roughness;
    half denom = NdotH * NdotH * (a2 - 1.0) + 1.0;
    half D = a2 / (denom * denom * PI);       // 消除了 pow

    // --- 优化3：G 项极简近似 ---
    half k = roughness * 0.5;
    half G = NdotL * NdotV / (NdotL * (1 - k) + k); // Keelman-Schlick G

    // --- 合并 specular ---
    half3 specular = D * F * G * 0.25;        // 0.25 = 1/4，替代除法

    // --- Diffuse 优化：half Lambert ---
    half3 diffuse = albedo * NdotL * (1.0 - metallic);

    return (diffuse + specular) * lightColor;
}

// SSS 近似：用预计算 LUT 代替实时积分
half3 ApproximateSSS(half2 uv, half3 NdotL)
{
    // ❌ 优化前：实时计算 3 次高斯卷积（~40 ALU）
    // ✅ 优化后：采样预烘焙的曲率 LUT（~4 cycles）
    half curvature = SAMPLE_TEXTURE2D(_CurvatureLUT, sampler_CurvatureLUT, uv).r;
    half3 sssColor = SAMPLE_TEXTURE2D(_SSSLUT, sampler_SSSLUT,
        float2(curvature, NdotL * 0.5 + 0.5)).rgb;
    return sssColor;
}
```

**Shader LOD 分级方案：**

```hlsl
// 用 shader_feature 或 multi_compile 控制质量等级
#pragma multi_compile _QUALITY_HIGH _QUALITY_MEDIUM _QUALITY_LOW

#if defined(_QUALITY_HIGH)
    // 完整 PBR + SSS + Detail Normal
    half3 color = CalculateLighting(...) + ApproximateSSS(...);
    half3 detailN = UnpackNormal(SAMPLE_TEXTURE2D(_DetailNormalMap, ...));
    N = normalize(N + detailN * _DetailStrength);

#elif defined(_QUALITY_MEDIUM)
    // 精简 PBR + Rim Light，无 SSS
    half3 color = CalculateLighting_Fast(...);
    half rim = 1.0 - saturate(dot(N, V));
    color += _RimColor * pow(rim, 3);  // 轻量 Rim

#elif defined(_QUALITY_LOW)
    // Half Lambert + 固定环境光
    half NdotL = dot(N, L) * 0.5 + 0.5;
    half3 color = albedo * NdotL;
#endif
```

**指令数对比表格：**

| 优化项 | 优化前周期数 | 优化后周期数 | 节省 |
|--------|-------------|-------------|------|
| normalize 调用（12→3次） | ~24 | ~6 | 75% |
| 菲涅尔 pow(N,5) → Schlick | ~10 | ~5 | 50% |
| SSS 实时积分 → LUT 采样 | ~40 | ~4 | 90% |
| GGX D 项简化 | ~15 | ~6 | 60% |
| G 项简化 | ~12 | ~4 | 67% |
| 寄存器优化（32→24） | 占用率 50% | 占用率 75% | +50% |
| **合计** | **~180 ALU** | **~60 ALU** | **~67%** |

### ⚡ 实战经验

- **先 Profile 再优化**：不看 Profiler 数据凭直觉优化，50% 概率优化了错误的地方。Snapdragon Profiler 的 "GPU Stage" 视图直接告诉你 ALU / Texture / Memory 哪个是瓶颈
- **ALU vs Texture 权衡**：有时多采样一张 LUT 反而更快——移动端纹理带宽往往不是瓶颈，ALU 才是。这和 PC 端正好相反
- **half 精度红利**：Adreno 和 Mali 都对 half 有硬件加速（fp16），在 URP 中默认使用 `half` 而非 `float`。检查 Shader 中是否有意外的 `float` 声明
- **多光源最伤性能**：每增加一个 pixel light，几乎整个 Pixel Shader 都要重跑一次。手游优先用球面谐波（SH）环境光 + 1 个主光源
- **Shader Variant 数量爆炸**：用了 `multi_compile` 做 Shader LOD 后，要检查 Build 报告中的变体数量，及时 Strip
- **Unity Shader Compiler 日志**：在 Project Settings → Editor → Shader Compilation 中开启 "Log Shader Compile Times"，能看到每个 Shader 的实际指令数

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么看 GPU 瓶颈 | 移动端 Profiler 工具链 | 学 Snapdragon Profiler + Mali Offline Compiler |
| 优化后效果差太多 | 光照模型数学理解 | 对比完整 PBR vs 移动 PBR 的视觉差异 |
| 寄存器压力概念模糊 | GPU 架构与占用率 | 学 Adreno/Mali 的 Warp/Wavefront 调度原理 |
| 不知道怎么分级 | Shader LOD 系统 | 学 Unity Quality Settings + shader_feature |
| 优化后变体数量爆炸 | Shader Variant Strip | 学 IPC（Shader Variant Collection）+ Build Strip |

### 🔗 相关问题

- 如何使用 Mali Offline Compiler 反编译 Shader 并读取寄存器数？
- PBR 的 BRDF LUT 预计算在移动端是否值得？（带宽 vs ALU 权衡）
- 如果角色是卡通渲染（NPR），上述优化策略有哪些不适用？
- Compute Shader 的优化思路和 Pixel Shader 有什么本质区别？
