---
title: "如何实现移动端可用的SSS皮肤次表面散射效果？"
category: "rendering"
level: 3
tags: ["SSS", "次表面散射", "皮肤渲染", "URP", "屏幕空间", "移动端优化"]
hint: "核心思路是用预积分纹理LUT或屏幕空间模糊近似次表面散射的颜色扩散"
related: ["shader/npr-outline-cartoon", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官给你看两张角色面部截图——一张是塑料感明显的标准 PBR，另一张是柔和有血色感的真实皮肤。然后说：

> "这是我们的角色脸部渲染，现在像塑料。我要你加入 SSS 次表面散射效果，让它看起来有真实皮肤的质感。平台是移动端，URP 管线。你有什么方案？"

### ✅ 核心要点

1. **SSS 本质**：光线进入皮肤后，在真皮层散射再射出，形成柔和的红色渗透感——尤其鼻翼、耳廓、指尖
2. **移动端方案选择**：不能用真正的离线光线追踪（Path Tracing），必须用近似方法——预积分 LUT 或屏幕空间模糊
3. **预积分贴图是性价比之王**：一张 2D LUT（曲率 × NdotL）烘焙好散射结果，Fragment 阶段一次采样即可
4. **法线/曲率是关键输入**：曲率（Curvature）决定散射范围，法线决定光照方向，两者构成 LUT 的 UV
5. **性能预算严格**：移动端 Fragment 额外 1 次采样 + 5 ALU 以内，总 SSS pass 不超过 0.5ms

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
真实皮肤效果：光在皮下散射，形成柔和红润的明暗过渡
     ↓ 倒推
散射 = 光照在非表面层多次弹射后扩散开
     ↓ 倒推
近似方法（移动端三选一）：
  ├── 方案A：预积分 LUT（1张纹理，1次采样）→ ✅ 首选
  ├── 方案B：屏幕空间高斯模糊（需要额外 RT）→ 适合高质量
  └── 方案C：Wrap Lighting 数学近似（最简，效果一般）→ 低端机
     ↓ 倒推
LUT 的输入 = （NdotL 光照方向, 曲率 1/r）→ 查表得到散射后亮度
     ↓ 倒推
曲率计算 = 从法线贴图或预烘焙的 Curvature Map 获取
     ↓ 倒推
集成到 URP = 在 Fragment 中替换标准漫反射为 LUT 采样结果
```

#### 知识点拆解（倒推树）

```
SSS 皮肤渲染
├── 理论基础
│   ├── BSSRDF（双向散射表面反射率分布函数）
│   ├── 皮肤的物理特性：表皮层 + 真皮层 + 皮下组织
│   ├── 光线渗透距离（红色光渗透最远 ≈ 3-5mm）
│   └── 曲率与散射半径的关系
├── 实现方案
│   ├── 预积分 LUT（Penner & Chen 2011）
│   │   ├── LUT 横轴 = NdotL
│   │   ├── LUT 纵轴 = 1/r（曲率）
│   │   └── 烘焙：对每对 (NdotL, curvature) 做高斯卷积积分
│   ├── 屏幕空间模糊（Separable SSSS by Jimenez 2015）
│   │   ├── 需要颜色 RT
│   │   ├── 多次高斯模糊（宽核+窄核）
│   │   └── 沿深度方向调整模糊宽度
│   └── Wrap Lighting（最简近似）
│       └── diffuse = saturate((NdotL + wrap) / (1 + wrap))
├── 曲率获取
│   ├── 方法1：预烘焙 Curvature Map（DCC 工具生成）
│   ├── 方法2：运行时从法线贴图推导（ddx/ddy 计算）
│   └── 方法3：顶点色存储曲率值（低成本）
├── URP 集成
│   ├── 方案A：在 Shader 中直接替换 diffuse 项（最简单）
│   ├── 方案B：Renderer Feature 做 SSS 后处理 pass
│   └── Stencil 标记皮肤区域（只对皮肤做 SSS）
└── 性能与质量平衡
    ├── LUT 采样 + 1 次法线采样（移动端推荐）
    ├── 屏幕空间方案需额外 1-2 张 RT（带宽成本）
    └── 逐角色 LOD：主角用 LUT，NPC 用 Wrap Lighting
```

#### 代码实现

**方案A：预积分 LUT（移动端首选）**

```hlsl
// ─── SSS Skin Shader（URP 兼容） ───

// 预积分的 SSS LUT（256x256 纹理）
TEXTURE2D(_SSSLUT);  SAMPLER(sampler_SSSLUT);

// 曲率贴图（预烘焙：白色=高曲率区域如鼻翼耳廓）
TEXTURE2D(_CurvatureMap);  SAMPLER(sampler_CurvatureMap);

// SSS 颜色偏移（控制红色渗透强度）
float3 _SSSColorTint = float3(1.0, 0.3, 0.2);  // 偏红色

half3 SSSDiffuse(float2 uv, float3 normalWS, float3 lightDirWS)
{
    // 1. 计算光照方向点积
    float NdotL = dot(normalize(normalWS), normalize(lightDirWS));
    // 映射到 [0,1]
    float u = NdotL * 0.5 + 0.5;

    // 2. 采样曲率贴图
    float curvature = SAMPLE_TEXTURE2D(_CurvatureMap, sampler_CurvatureMap, uv).r;
    // 映射曲率到 LUT 纵轴（0=平面，1=高曲率如鼻尖）
    float v = saturate(curvature);

    // 3. 查 LUT 获得散射后的漫反射值
    float sssValue = SAMPLE_TEXTURE2D(_SSSLUT, sampler_SSSLUT, float2(u, v)).r;

    // 4. 颜色偏移：暗部加红（模拟皮下血液散射）
    float3 sssColor = sssValue * _SSSColorTint;

    return sssColor;
}

// ─── 完整 Fragment（集成到标准 PBR） ───
half4 frag(Varyings IN) : SV_Target
{
    float3 normalWS = GetNormalWS(IN);       // 法线
    float3 lightDir = GetMainLight().direction;

    // 标准 PBR 漫反射
    half NdotL = saturate(dot(normalWS, lightDir));
    half3 standardDiffuse = NdotL * _BaseColor.rgb;

    // SSS 漫反射（替换标准漫反射）
    half3 sssDiffuse = SSSDiffuse(IN.uv, normalWS, lightDir) * _BaseColor.rgb;

    // 混合：高曲率区域用更多 SSS，低曲率区域用更多标准
    float curvature = SAMPLE_TEXTURE2D(_CurvatureMap, sampler_CurvatureMap, IN.uv).r;
    half3 diffuse = lerp(standardDiffuse, sssDiffuse, saturate(curvature * 2.0));

    // 加上高光（皮肤高光用 Kelemen/Szirmay-Kalos 模型更真实）
    half3 specular = CalcSkinSpecular(IN, normalWS, lightDir);

    half3 finalColor = diffuse + specular;
    return half4(finalColor, 1.0);
}
```

**方案B：Wrap Lighting（超低端机 / 快速原型）**

```hlsl
// 最简 SSS 近似——半兰伯特 + 红色渗透
half3 WrapSSSDiffuse(float3 normalWS, float3 lightDirWS)
{
    float wrap = 0.5;  // wrap 值越大，散射越柔
    float NdotL = dot(normalWS, lightDirWS);
    float wrappedNdotL = saturate((NdotL + wrap) / (1.0 + wrap));

    // 暗部添加红色（模拟皮下散射光色）
    float3 shadowTint = float3(1.0, 0.4, 0.3);
    float shadowMask = saturate(1.0 - NdotL * 0.5);
    half3 sssTint = lerp(float3(1,1,1), shadowTint, shadowMask * 0.3);

    return wrappedNdotL * sssTint;
}
```

**三种方案对比：**

| 方案 | 额外纹理 | ALU | 视觉质量 | 适用平台 | 实现难度 |
|------|---------|-----|---------|---------|---------|
| Wrap Lighting | 0 | ~5 | ★★☆ | 低端 Android | ⭐ 简单 |
| **预积分 LUT** | **LUT + 曲率图 = 2张** | **~8** | **★★★★** | **主流手游** | **⭐⭐ 中等** |
| 屏幕空间模糊 (SSSS) | 2张 RT | ~30 | ★★★★★ | 高端机 / PC | ⭐⭐⭐ 复杂 |
| 真正的 BSSRDF 积分 | N/A | 100+ | ★★★★★+ | 离线渲染 / 电影 | ⭐⭐⭐⭐⭐ |

**LUT 预烘焙核心代码（Unity Editor 脚本）：**

```csharp
// 生成 SSS LUT 的烘焙脚本
[MenuItem("TA/Bake SSS LUT")]
static void BakeSSSLUT()
{
    int size = 256;
    Texture2D lut = new Texture2D(size, size, TextureFormat.RGBA32, false);

    for (int y = 0; y < size; y++)
    {
        float curvature = (float)y / (size - 1);  // 0~1
        float scatterRadius = Mathf.Lerp(0.001f, 0.02f, curvature);

        for (int x = 0; x < size; x++)
        {
            float NdotL = (float)x / (size - 1) * 2.0f - 1.0f;  // -1~1

            // 高斯卷积近似：对 NdotL 附近做加权积分
            float sum = 0, weightSum = 0;
            for (int s = -8; s <= 8; s++)
            {
                float sampleNdotL = NdotL + s * 0.02f;
                float diffuse = Mathf.Clamp01(sampleNdotL * 0.5f + 0.5f);
                float gaussWeight = Mathf.Exp(-(s * s) / 8.0f);
                sum += diffuse * gaussWeight;
                weightSum += gaussWeight;
            }
            float value = sum / weightSum;

            // 暗部偏红（皮肤散射颜色特性）
            float r = value;
            float g = value * Mathf.Lerp(0.8f, 1.0f, NdotL * 0.5f + 0.5f);
            float b = value * Mathf.Lerp(0.6f, 1.0f, NdotL * 0.5f + 0.5f);

            lut.SetPixel(x, y, new Color(r, g, b, 1));
        }
    }
    lut.Apply();

    byte[] png = lut.EncodeToPNG();
    File.WriteAllBytes("Assets/TA/SSS_LUT.png", png);
    AssetDatabase.Refresh();
    Debug.Log("SSS LUT baked successfully!");
}
```

### ⚡ 实战经验

- **曲率贴图是质量分水岭**：没有曲率图，SSS 只能全局模糊；有了曲率图，鼻翼耳廓等细节区域的散射才真实。可以让 TA 在 Substance Painter 中用 Curvature 节点直接烘焙
- **暗部的红色渗透是灵魂**：标准漫反射在暗部是纯黑的，但真实皮肤暗部会有微弱的红色透出（皮下血液）。这个细节决定了角色"像不像真人"
- **LUT 精度够用就行**：256×256 的 LUT 已经完全够用，不要用 1024——采样精度不会更好但包体变大
- **别忘了高光**：皮肤的 specular 要单独处理。用 Kelemen/Szirmay-Kalos 模型替代标准 GGX，皮肤的油性感更强。T 字区和额头的高光形状会和标准 PBR 明显不同
- **NPC 用廉价方案**：主角用 LUT，NPC 用 Wrap Lighting，远处 NPC 直接标准 PBR。三档 LOD 保证帧率

### 🎯 能力体检清单

| 卡住的环节 | 说明你缺失的知识点 | 补习建议 |
|-----------|-------------------|---------|
| 不理解 SSS 和普通漫反射的区别 | 缺少 BSSRDF 物理基础 | 阅读 "Real-Time Rendering" 第 9.8 节（Subsurface Scattering） |
| 不知道怎么获取曲率 | 不了解曲率贴图的生成方式 | 学习 Substance Painter 的 Curvature 烘焙 / 法线 ddx/ddy 推导 |
| LUT 不知道怎么烘焙 | 缺少离线积分 / Editor 脚本能力 | 学习 Unity Editor Scripting + 高斯卷积数学 |
| 效果像塑料不是皮肤 | 高光模型不对 / 缺少暗部红色 | 研究 Kelemen 皮肤高光 + 暗部色彩偏移 |
| 移动端帧率掉太多 | 采样次数 / RT 带宽超预算 | 降级到 Wrap Lighting 方案，减少纹理采样 |

### 🔗 相关问题

- 如何在 URP 中用 Renderer Feature 实现屏幕空间 SSS（Separable SSSS）？需要哪些 Pass？
- 头发渲染的 Marschner 模型和皮肤 SSS 有什么共通的光学原理？
- 如果角色同时需要 SSS 和描边（卡通渲染），渲染顺序和 Pass 怎么组织？
