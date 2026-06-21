---
title: "移动端贴图压缩方案选型：项目上线前你如何定规范？"
category: "technical-art"
level: 2
tags: ["贴图压缩", "ASTC", "ETC2", "移动端", "美术规范", "内存优化"]
hint: "核心考点：ASTC vs ETC2 vs BC7 的压缩率/质量/兼容性三角——以及如何按贴图类型差异化选型"
related: ["optimization/drawcall-500-to-100", "pipeline/material-template-system"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们新项目是一款手游，Unity URP，目标平台 iOS + 安卓中高端机。目前美术给的贴图都是 PNG 未压缩，包体 8GB，内存占用过高。请你来制定一套贴图压缩规范——告诉我每种贴图类型该用什么格式，为什么。」

### ✅ 核心要点

1. **格式不是万能的**：ASTC 是当前移动端主流推荐，但不是所有贴图都该用同一格式
2. **按类型差异化**：Diffuse/Normal/UI/Lightmap 各有最佳压缩格式
3. **质量-大小-兼容三角**：ASTC 质量最好但旧设备不支持；ETC2 兼容性最广但质量一般
4. **Mipmap + 尺寸规范**：压缩只是第一步，尺寸上限和 Mipmap 策略同样关键
5. **工具链落地**：规范写完不算完，要能用脚本批量检查和转换

### 📖 深度展开

#### 解决思路（从问题倒推方案）

```
问题：包体 8GB，内存过高，PNG 未压缩
                ↑
倒推1：PNG 没有运行时压缩 → 必须用 GPU 原生支持的压缩格式
倒推2：iOS 和安卓支持的格式不同 → 需要分平台设置
倒推3：不同贴图类型对质量要求不同 → 不能一刀切
倒推4：需要一套可执行的规范 + 工具链
倒推5：需要验收标准（质量 + 包体 + 内存 三维约束）
```

#### 知识点拆解（倒推树）

```
移动端贴图压缩规范
├── 压缩格式知识
│   ├── ASTC（4x4 ~ 12x12 block，质量/大小可调，iOS A7+ & 安卓 Adreno 3xx+ / Mali T628+）
│   ├── ETC2（安卓标配，4bpp 固定，EAC 用于 Alpha 通道）
│   ├── BC7（桌面/主机首选，不适用于移动端 GPU）
│   ├── PVRTC（iOS 旧设备，仅支持正方形 2 的幂次纹理）
│   └── RGBA16f / RGB9E5（HDR 环境贴图专用）
├── 按贴图类型选型
│   ├── Diffuse/Albedo → ASTC 6x6（质量足够，1.35 bpp）
│   ├── Normal Map → ASTC 5x5（法线对压缩敏感，需要更高质量）
│   ├── ORM（Occlusion/Roughness/Metallic）→ ASTC 6x6（通道合并，省纹理槽位）
│   ├── UI Atlas → ASTC 4x4 或 RGBA32（UI 对色带极敏感）
│   ├── Lightmap → ASTC 6x6 或 ETC2_RGB（可接受轻微失真）
│   └── Cubemap/Skybox → ASTC 6x6（mipmap 链必须完整）
├── 平台策略
│   ├── iOS：ASTC 为主（A8 起全系支持）
│   ├── 安卓高端：ASTC（Adreno 4xx+ / Mali Midgard+）
│   ├── 安卓低端兜底：ETC2 fallback
│   └── Unity 设置：Platform-specific override per texture
├── 尺寸与 Mipmap 规范
│   ├── 主角贴图：2048x2048 上限
│   ├── NPC：1024x1024
│   ├── 环境/道具：512x512 ~ 256x256
│   ├── UI：2048 atlas，少于 5 张
│   └── UI/小图标关闭 Mipmap（避免 1px 偏移抖动）
└── 工具链
    ├── Unity Texture Importer 预设（.preset 文件）
    ├── Editor 脚本批量检查违规贴图
    └── CI 卡口：打包前扫描包体超限的贴图
```

#### 格式对比表（核心知识）

| 格式 | 压缩率（vs RGBA32） | 质量 | 兼容性 | bpp | 适用场景 |
|------|---------------------|------|--------|-----|----------|
| ASTC 4x4 | 8:1 | 优秀 | iOS A7+/安卓部分 | 8.0 | UI / 高精度法线 |
| ASTC 5x5 | 12.8:1 | 很好 | 同上 | 5.12 | Normal Map |
| ASTC 6x6 | 18.5:1 | 好 | 同上 | 3.56 | Albedo / ORM |
| ASTC 8x8 | 32:1 | 一般 | 同上 | 2.0 | Lightmap / 远景 |
| ETC2_RGB | 6:1 | 中等 | 安卓全系 | 4.0 | 安卓兜底 RGB |
| ETC2_RGBA | 6:1 | 中等 | 安卓全系 | 8.0 | 安卓兜底 RGBA |
| PVRTC 4bpp | 8:1 | 较差 | iOS 旧设备 | 4.0 | 仅兜底用 |

**包体估算公式（ASTC 6x6 为例）：**

```
单张贴图大小 ≈ 宽 × 高 × bpp / 8
2048×2048 ASTC 6x6 ≈ 2048 × 2048 × 3.56 / 8 ≈ 1.86 MB
2048×2048 RGBA32   ≈ 2048 × 2048 × 32 / 8  ≈ 16.78 MB
压缩比 ≈ 9:1
```

#### 工具链实现

**Unity Texture Preset 批量应用脚本：**

```csharp
using UnityEditor;
using UnityEngine;

public class TextureCompressionAuditor : EditorWindow
{
    [MenuItem("Tools/TA/贴图规范检查")]
    static void AuditTextures()
    {
        string[] guids = AssetDatabase.FindAssets("t:Texture2D");
        int violations = 0;

        foreach (string guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            TextureImporter importer = AssetImporter.GetAtPath(path) as TextureImporter;

            if (importer == null) continue;

            // 检查 iOS 平台设置
            var iosSettings = importer.GetPlatformTextureSettings("iPhone");
            if (!iosSettings.overridden || iosSettings.format != TextureImporterFormat.ASTC)
            {
                Debug.LogWarning($"[违规] {path} - iOS 未使用 ASTC", 
                    AssetDatabase.LoadAssetAtPath<Object>(path));
                violations++;
            }

            // 检查最大尺寸
            if (importer.maxTextureSize > 2048 && !path.Contains("UI/"))
            {
                Debug.LogWarning($"[违规] {path} - 尺寸超过 2048: {importer.maxTextureSize}",
                    AssetDatabase.LoadAssetAtPath<Object>(path));
                violations++;
            }

            // UI 贴图不应开启 Mipmap
            if (path.Contains("UI/") && importer.mipmapEnabled)
            {
                Debug.LogWarning($"[违规] {path} - UI 贴图不应开启 Mipmap",
                    AssetDatabase.LoadAssetAtPath<Object>(path));
            }
        }

        EditorUtility.DisplayDialog("检查完成", $"共检查 {guids.Length} 张贴图，违规 {violations} 张", "OK");
    }
}
```

**贴图类型 → 压缩格式决策矩阵：**

```
                    iOS (ASTC)      安卓 (ASTC/ETC2)    WebGL (ETC2/BC7)
Albedo              6x6             6x6 / ETC2_RGB      ETC2_RGB
Normal Map          5x5             5x5 / ETC2_RGB      BC7
ORM (合并通道)      6x6             6x6 / ETC2_RGB      ETC2_RGB
UI Atlas            4x4             4x4 / ETC2_RGBA     RGBA32
Lightmap            6x6             6x6 / ETC2_RGB      BC7
Cubemap             6x6             6x6 / ETC2_RGB      BC7
```

### ⚡ 实战经验

- **ASTC 4x4 不等于 ETC2 质量**：ASTC 4x4 的质量远超 ETC2，但 bpp 一样（8 bpp），所以能用 ASTC 就别用 ETC2
- **Normal Map 是重灾区**：法线贴图压缩最容易出「块状伪影」，ASTC 5x5 是甜点，ETC2 会明显失真（尤其是蓝轴 Z 分量重建）
- **ORM 通道合并**：Unity 默认 Standard Shader 的 Occlusion/Roughness/Metallic 可以合并到一张 RGB 贴图，直接省 2 张纹理采样和 2/3 内存
- **iOS 兜底：PVRTC 是最后手段**：A7 之前设备不支持 ASTC，但现在（2024+）这部分设备占比已经 <1%，可以直接放弃

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 bpp 怎么算 | 压缩格式的 block 编码原理 | 学 ASTC 的 4×4 block 固定大小 vs 变 bpp |
| 选了 ASTC 但安卓闪退 | 设备兼容性检查 | 看 GPU 能力矩阵（Adreno/Mali/PowerVR） |
| 压缩后法线扭曲 | Normal Map 压缩特殊性 | 学 BC5/ASTC 对法线的处理 vs tangent-space 重建 |
| UI 图标有色带 | UI 对色深敏感 | UI 用 ASTC 4x4 或不压缩，避免 16-bit 中间格式 |
| 规范写了但没人执行 | 工具链不闭环 | CI 集成 + Editor 脚本卡口 |

### 🔗 相关问题

- 如果项目必须兼容 2019 年的老安卓设备（不支持 ASTC），你怎么处理 fallback 策略？
- Unity 的 Sprite Atlas 和 Texture2DArray 在内存优化上有什么区别？
- 如何用 Python + ASTC 编码器在 CI 流水线中批量预压缩贴图，加速构建？
