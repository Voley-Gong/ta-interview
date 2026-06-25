---
title: "角色面部阴影闪烁抖动：移动端法线精度不够怎么办？"
category: "technical-art"
level: 3
tags: ["法线贴图", "阴影抖动", "面部渲染", "BC5压缩", "移动端", "NPR"]
hint: "面部法线贴图压缩后精度丢失导致阴影抖动——核心是法线重构精度与导线方向的平衡"
related: ["technical-art/mobile-normal-map-compression", "rendering/cel-shading-toon-pipeline", "shader/npr-outline-cartoon"]
---

## 参考答案

### 🎬 场景描述

面试官展示一段移动端游戏的角色面部特写视频，角色微微转头时，鼻子侧面和额头处的阴影边缘剧烈抖动（像心电图一样跳），然后说：

> "这是我们的主角面部，美术说 TD 给的法线贴图质量没问题，在 PC 上完全正常。但到了手机端（Android ASTC 6x6 + 法线用 BC5 压缩）就开始抖。你是 TA，告诉我原因和解决方案。"

这是叠纸、鹰角、米哈游等做二次元/写实角色项目的 **TA 中高级面试必问题**。考察的是法线贴图压缩、面部渲染管线、以及精度问题的系统排查能力。

### ✅ 核心要点

1. **根因：法线精度丢失**：BC5 / ASTC 压缩将法线从 PC 端的 RGBA32（每通道 8bit）降精度，微小角度差异被量化抹平
2. **面部放大效应**：面部曲面变化平缓，相邻像素法线差异极小，压缩后这些微小差异被「四舍五入」到同一值，导致阶梯式跳变
3. **阴影敏感区**：Carol-Toon / Ramp Shading 中，法线 · 光照方向的点积落在 ramp 的硬切变边界附近时，极小的法线变化 → 阴影翻转
4. **解决方案矩阵**：从法线生成 → 压缩格式 → Shader 补偿 → Ramp 调整四个层面攻防

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终现象：面部阴影边缘抖动（尤其在鼻子侧面、眉弓、下唇下方）
                ↑
倒推1：阴影由 N·L 决定 → N（法线）在相邻像素间出现阶梯跳变
倒推2：法线跳变来自压缩精度丢失 → BC5 双通道 8bit 重建法线，z 分量误差大
倒推3：面部法线变化本就平缓 → 压缩前相邻像素法线差异 < 量化步长
倒推4：Ramp Shading 的硬边切变放大了问题 → 微小法线变化越过阈值 = 阴影翻转
倒推5：PC 不抖是因为未压缩（RGBA32）精度足够
```

#### 知识点拆解（倒推树）

```
面部阴影抖动
├── 法线贴图压缩
│   ├── BC5（DXN）原理：存 XY 两通道，Z = sqrt(1 - x² - y²) 重建
│   ├── ASTC 压缩对法线的精度损失（块效应）
│   ├── 移动端替代方案：BC5 → ASTC 6x6 → RGBA16F → 未压缩
│   └── 为什么面部尤其敏感（曲面平缓 → 微小差异被量化淹没）
├── 面部渲染管线（NPR / Ramp Shading）
│   ├── Ramp 贴图的硬边切变（hard breakpoint）
│   ├── N·L 的精度敏感性：在切变边界 ±0.001 的波动 → 阴影翻转
│   ├── 软化 Ramp（smoothstep / AA-break）对抖动的缓解程度
│   └── SDF（Signed Distance Field）面部阴影方案（原神方案）
├── 法线贴图生成
│   ├── 高模 → 低模烘焙时的 Ray Cast 角度
│   ├── 法线贴图的「导线方向」问题（Tangent Space vs Object Space）
│   ├── 面部专用法线：手绘法线 vs 烘焙法线
│   └── 为什么不能直接用高模法线（导线方向错误 → 压缩后更糟）
├── Shader 补偿方案
│   ├── 在 Shader 中做法线高频保持（derivative-based）
│   ├── 屏幕空间法线抖动消除（temporal AA 思路）
│   └── 双法线方案：压缩法线 + 低精度导数法线混合
└── 工程权衡
    ├── 内存预算：未压缩法线 vs BC5 vs ASTC 的内存占用
    ├── 品质基线：中低端机是否降级使用 ramp 软化
    └── 美术工作流：面部专属导出 Preset（不同于衣服/道具）
```

#### 代码实现

**方案1：Ramp 软化（最快见效，不改资源）**

```hlsl
// 原始 Ramp（硬切变，放大法线误差）
half rampValue = NdotL > 0.5 ? 1.0 : 0.3;

// 软化 Ramp（在切变边界做平滑过渡）
half rampValue = smoothstep(0.45, 0.55, NdotL);
half3 shaded = lerp(_ShadowColor, _BaseColor, rampValue);

// 进阶：带抗锯齿的 Ramp（fwidth 自适应宽度）
half aaWidth = fwidth(NdotL) * 1.5; // 根据屏幕导数自适应
half rampValue = smoothstep(0.5 - aaWidth, 0.5 + aaWidth, NdotL);
```

**方案2：法线贴图双采样 + 中心平滑（减少块效应）**

```hlsl
// 采样法线贴图，用三次采样平均减少压缩块效应
half3 n1 = UnpackNormal(SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, uv));
half3 n2 = UnpackNormal(SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, uv + _TexelSize.xy * 0.5));
half3 n3 = UnpackNormal(SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, uv - _TexelSize.xy * 0.5));
half3 normalTS = normalize((n1 + n2 + n3) * 0.333);
```

**方案3：SDF 面部阴影（原神方案简化版）**

```hlsl
// 不依赖法线贴图，用预烘焙的 SDF 贴图控制面部阴影
// SDF 贴图：R 通道 = 左侧阴影距离场，G 通道 = 右侧阴影距离场
Texture2D _FaceShadowSDF;
half2 sdf = SAMPLE_TEXTURE2D(_FaceShadowSDF, sampler_FaceShadowSDF, uv).rg;

// 根据 N·L 在 SDF 上查找阴影边界
half leftThreshold = 1.0 - sdf.r;  // 左脸阴影边界
half rightThreshold = 1.0 - sdf.g; // 右脸阴影边界

// 光照方向在切线空间的角度（0=正脸左→右）
half lightAngle = atan2(lightDirTS.x, lightDirTS.z) / PI * 0.5 + 0.5; // [0,1]

half faceShadow = 1.0;
if (lightAngle < 0.5) {
    faceShadow = step(leftThreshold + (0.5 - lightAngle), 0.5);
} else {
    faceShadow = step(rightThreshold + (lightAngle - 0.5), 0.5);
}

// 软化 SDF 边界
faceShadow = smoothstep(0.48, 0.52, faceShadow * (1.0 - sdf.r * (1.0 - lightAngle * 2)));
```

**方案对比表**

| 方案 | 改动成本 | 效果 | 内存增加 | 适用场景 |
|------|----------|------|----------|----------|
| Ramp smoothstep | Shader 改2行 | 中等 | 0 | 快速止血 |
| fwidth 自适应 Ramp | Shader 改5行 | 好 | 0 | 推荐首选 |
| 法线三采样平均 | Shader + 1次采样 | 中等 | 0 | 中端机 |
| 法线改 ASTC 8x8 | 导出设置 | 好 | +33% | 中高端机 |
| 法线改 RGBA16F | 导出设置 | 完美 | +100% | 旗舰机专属 |
| SDF 面部阴影 | 美术重做 | 完美 | +1张贴图 | 二次元项目 |

### ⚡ 实战经验

- **先查导线方向**：80% 的「PC 正常手机抖」案件，根因是切线空间导线方向（Tangent）在模型导入时没有正确设置。Unity 的 `Model Importer → Normals & Tangents → Tangent Space` 检查 Calculate vs Import
- **面部单独 Preset**：不要用全角色统一的法线压缩设置。面部贴图单独一个材质球，单独指定压缩格式
- **美术沟通**：告诉美术「面部法线贴图的渐变区域不要太平滑」——适当加一点高频细节反而能对抗压缩量化（噪声 dithering 效果）
- **SDF 方案的代价**：原神 SDF 面部阴影需要美术手调每个角色的 SDF 贴图（每个表情一张），成本高但对抖动问题是根治
- **ASTC vs BC5**：Android 上 ASTC 6x6 对法线的精度不如 iOS 的 ASTC 6x6（同格式但 iOS GPU 解压精度更高）。低端 Android 可以考虑 ASTC 8x8 + RGBA fallback
- **验证方法**：用 RenderDoc 抓帧对比 PC 和移动端的法线贴图采样结果，直接看量化误差

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道为什么 PC 正常手机抖 | 法线贴图压缩格式原理 | 学 BC5 / ASTC 压缩算法 |
| 只会说「提高精度」 | 缺乏 Shader 层面的补偿手段 | 学 fwidth / smoothstep 抗锯齿 |
| 不知道 SDF 面部阴影 | NPR 面部渲染前沿方案 | 读原神技术分享（SDF 面部阴影） |
| 分不清 Tangent/Object Space 法线 | 法线空间与导线方向 | 学切线空间基础与 Maya/Blender 导线 |
| 说不出 Ramp Shading 为什么放大抖动 | Ramp 切变与精度交互 | 手动实现 Cel-Shading，观察切变边界 |

### 🔗 相关问题

- 移动端法线贴图压缩格式怎么选？（BC5 vs ASTC vs ETC2 的精度对比）
- 原神的 SDF 面部阴影具体是怎么实现的？SDF 贴图怎么烘焙？
- 卡通渲染中 Ramp 贴图和法线贴图的精度如何平衡？
- 为什么面部要单独做一个材质球，不能和身体共用？
