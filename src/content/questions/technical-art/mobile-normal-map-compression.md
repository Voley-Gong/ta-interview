---
title: "移动端法线贴图压缩：如何在 0.5MB 预算内保持法线精度？"
category: "technical-art"
level: 2
tags: ["法线贴图", "纹理压缩", "BC5", "ETC2", "移动端", "DXT5nm"]
hint: "法线贴图压缩不是选个格式那么简单——BC5 vs ETC2 vs ASTC 的精度差异、XY重计算Z、BC7 的崛起，每一步都是面试加分项"
related: ["technical-art/mobile-texture-compression", "optimization/gpu-bandwidth-optimization", "technical-art/texture-streaming-mipmap-bias"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们的一款手游角色，每个角色用了 4 张法线贴图（身体/头部/服装/武器），每张 1024×1024。美术反馈说中低端机上内存爆了。现在要求你：
> 1. 把法线贴图的总内存占用从当前的 8MB 压缩到 2MB 以内
> 2. 压缩后角色面部在 1 米距离内不能有明显锯齿/色块
> 3. 给出一套可复用的法线贴图压缩规范

这是叠纸、鹰角、米哈游等角色品质驱动的公司常见的 TA 资产规范面试题。考的不只是"知道压缩格式"，而是**精度 vs 体积的权衡决策 + 跨平台规范制定**。

### ✅ 核心要点

1. **法线贴图的特殊性**：法线是单位向量，Z 分量可由 XY 推导（`Z = sqrt(1 - X² - Y²)`），所以只需存两个通道
2. **压缩格式选型**：BC5（PC/主机）、ETC2（Android）、ASTC（全平台未来）、PVRTC（iOS 旧设备）
3. **通道重排技巧**：把法线 XY 放入 RG 通道（或 GB 通道），利用不同压缩格式的通道精度差异
4. **质量验证流程**：压缩前后的法线方向差异（Angular Error）需要工具量化，不能只看肉眼
5. **规范制定**：不同用途（角色/场景/道具）的法线贴图应有不同的压缩策略

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：4 张 1024 法线贴图 < 2MB，面部近距离无锯齿
                ↑
Step 1：分析当前内存
  压缩前：1024×1024 RGBA32 = 4MB/张 × 4 张 = 16MB（未压缩）
  当前 DXT5/ETC2 RGBA = 约 1MB/张 × 4 张 = 4MB
  目标：< 2MB（即每张 < 512KB）
                ↑
Step 2：压缩策略
  策略A：降低分辨率（1024 → 512）→ 直接减 1/4，但面部细节丢失
  策略B：换高压缩比格式（ASTC 6×6 → 8×8）→ 精度略降但体积减半
  策略C：只存双通道（BC5 / ETC2 双通道）→ 法线只需 XY，Z 可推导
  策略D：分级策略 → 面部用高精度，身体/服装用低精度
                ↑
Step 3：组合方案
  面部：512×512 ASTC 6×6 → 高精度
  身体：512×512 ASTC 8×8 → 中精度
  服装：512×512 ASTC 8×8 → 中精度
  武器：256×256 ASTC 8×8 → 低精度
  总计 ≈ 0.15 + 0.08 + 0.08 + 0.02 = 0.33MB ✅
```

#### 知识点拆解（倒推树）

```
法线贴图压缩
├── 法线贴图的数学基础
│   ├── 切线空间法线（Tangent Space Normal）
│   │   └── 大部分值为 (0, 0, 1) → 偏蓝
│   ├── 单位向量约束：X² + Y² + Z² = 1
│   │   └── 所以 Z = sqrt(1 - X² - Y²) → 只需存 XY
│   ├── 值域：X,Y ∈ [-1, 1] → 编码为 [0, 1] → (value + 1) / 2
│   └── 通道重要性：XY 携带高频信息，Z 比较平滑
│
├── 压缩格式对比
│   ├── DXT5nm / BC3（DirectX）
│   │   ├── 原理：Alpha 通道存 X（高精度），Green 通道存 Y（高精度）
│   │   ├── RGB 其他通道不用（填 1 或 255）
│   │   ├── 精度：每通道 8bit，块大小 4×4
│   │   └── 体积：1MB/张（1024²）
│   ├── BC5（DirectX 10+）
│   │   ├── 原理：两个独立 8bit 通道分别存 X 和 Y
│   │   ├── 精度：比 DXT5nm 更高（双通道各 8bit end-point）
│   │   ├── 体积：1MB/张（1024²）
│   │   └── 适用：PC / 主机 / 部分 Android
│   ├── ETC2（Android 标准）
│   │   ├── 原理：基于 ETC1 扩展，支持 RGBA
│   │   ├── 精度：RGB 各通道等权压缩，法线精度不如 BC5
│   │   ├── 体积：1MB/张（1024² RGBA）或 0.5MB（RGB）
│   │   └── 问题：法线的 XY 通道精度不够 → 角度误差大
│   ├── ASTC（Adaptive Scalable Texture Compression）
│   │   ├── 原理：自适应块大小（4×4 ~ 12×12），精度与体积可调
│   │   ├── 优势：块大小灵活，质量优于 ETC2
│   │   ├── 体积：1024² ASTC 6×6 = 0.59MB, 8×8 = 0.33MB, 10×10 = 0.21MB
│   │   └── 支持：iOS A8+ / Android 高通620+ / Mali Midgard+
│   └── PVRTC（iOS 旧设备）
│       ├── 4bpp：0.5MB/张（1024²）
│       └── 质量较差，正在被 ASTC 淘汰
│
├── 通道重排策略
│   ├── DXT5nm 技巧
│   │   ├── 把法线 X 存入 Alpha（DXT5 Alpha 是 8bit 独立压缩）
│   │   ├── 把法线 Y 存入 Green（DXT5 Green 精度最高）
│   │   ├── Red 和 Blue 填 255（引擎在采样时重建）
│   │   └── Shader 中：normal.xy = tex2D.a, tex2D.g; normal.z = sqrt(1 - dot(xy, xy))
│   ├── ETC2 双通道优化
│   │   ├── 用 ETC2 RGB 只存 XY+占位
│   │   └── 比 RGBA 减少 25% 体积
│   └── ASTC 通道配置
│       ├── ASTC 默认 RGB 或 RGBA
│       └── 只需 XY → 不存 Alpha → 减少压缩负担
│
├── 质量评估
│   ├── 角度误差（Angular Error）
│   │   ├── 解压后法线与原始法线的夹角（单位：度）
│   │   ├── 可接受范围：< 5°（远距离）/ < 2°（近距离）
│   │   └── 工具：Texture压缩质量分析器 / 自定义 Python 脚本
│   ├── PSNR（峰值信噪比）
│   │   └── > 35dB 为可接受
│   └── 肉眼检查
│       ├── 面部：鼻翼/嘴唇等高频区域是否出现色块
│       ├── 金属表面：反射是否抖动
│       └── 布料：褶皱是否平滑
│
└── 规范制定
    ├── 角色法线
    │   ├── 面部：512² ASTC 6×6（高品质）
    │   ├── 身体：512² ASTC 8×8（标准品质）
    │   └── 换装部件：256²~512² ASTC 8×8
    ├── 场景法线
    │   ├── 建筑：256²~512² ASTC 8×8
    │   └── 地面：512² ASTC 6×6（近距离可见）
    └── 道具法线
        └── 128²~256² ASTC 8×8
```

#### 代码实现

**Unity 平台覆盖设置 + 构建脚本：**

```csharp
using UnityEditor;
using UnityEngine;

public class NormalTexturePostProcessor : AssetPostprocessor
{
    // 根据纹理名称中的路径/关键词自动分配压缩格式
    void OnPreprocessTexture()
    {
        TextureImporter importer = (TextureImporter)assetImporter;
        string path = assetPath.ToLower();
        
        // 判断是否为法线贴图
        bool isNormalMap = path.Contains("_nrm") || path.Contains("_normal") || 
                          path.Contains("/normals/") || importer.textureType == TextureImporterType.NormalMap;
        
        if (!isNormalMap) return;
        
        importer.textureType = TextureImporterType.NormalMap;
        
        // === 面部法线 → 高精度 ===
        if (path.Contains("face") || path.Contains("head") || path.Contains("face"))
        {
            SetupNormalMap(importer, 512, TextureCompressionQuality.High, 6); // ASTC 6x6
            Debug.Log($"[NormalProcessor] Face normal: {path} → ASTC 6x6 @ 512");
        }
        // === 身体/服装法线 → 中精度 ===
        else if (path.Contains("body") || path.Contains("clothes") || path.Contains("costume"))
        {
            SetupNormalMap(importer, 512, TextureCompressionQuality.Normal, 8); // ASTC 8x8
            Debug.Log($"[NormalProcessor] Body normal: {path} → ASTC 8x8 @ 512");
        }
        // === 武器/小道具法线 → 低精度 ===
        else if (path.Contains("weapon") || path.Contains("prop"))
        {
            SetupNormalMap(importer, 256, TextureCompressionQuality.Normal, 8);
            Debug.Log($"[NormalProcessor] Weapon normal: {path} → ASTC 8x8 @ 256");
        }
        // === 默认 ===
        else
        {
            SetupNormalMap(importer, 512, TextureCompressionQuality.Normal, 8);
        }
        
        importer.SaveAndReimport();
    }
    
    void SetupNormalMap(TextureImporter importer, int maxSize, 
                         TextureCompressionQuality quality, int astcBlockSize)
    {
        importer.maxTextureSize = maxSize;
        importer.isReadable = false;
        importer.mipmapEnabled = true;
        importer.mipMapBias = -0.5f; // 法线贴图稍微偏锐
        
        // === Android: ASTC 优先，ETC2 fallback ===
        var androidSettings = importer.GetPlatformTextureSettings("Android");
        androidSettings.overridden = true;
        androidSettings.maxTextureSize = maxSize;
        androidSettings.format = TextureImporterFormat.ASTC_RGBA_8x8; // 8x8 块
        if (astcBlockSize <= 6)
        {
            androidSettings.format = TextureImporterFormat.ASTC_RGBA_6x6; // 高精度
        }
        androidSettings.compressionQuality = quality;
        importer.SetPlatformTextureSettings(androidSettings);
        
        // === iOS: ASTC ===
        var iosSettings = importer.GetPlatformTextureSettings("iPhone");
        iosSettings.overridden = true;
        iosSettings.maxTextureSize = maxSize;
        iosSettings.format = astcBlockSize <= 6 ? 
                             TextureImporterFormat.ASTC_RGBA_6x6 : 
                             TextureImporterFormat.ASTC_RGBA_8x8;
        iosSettings.compressionQuality = quality;
        importer.SetPlatformTextureSettings(iosSettings);
        
        // === PC: BC5 (DXT5nm) ===
        var pcSettings = importer.GetPlatformTextureSettings("Standalone");
        pcSettings.overridden = true;
        pcSettings.maxTextureSize = Mathf.Max(maxSize, 1024); // PC 可以用更大尺寸
        pcSettings.format = TextureImporterFormat.BC5; // PC 端用 BC5
        pcSettings.compressionQuality = TextureCompressionQuality.High;
        importer.SetPlatformTextureSettings(pcSettings);
    }
}
```

**法线压缩质量检测工具（Python，离线分析）：**

```python
#!/usr/bin/env python3
"""
法线贴图压缩质量分析器
对比原始法线和解压法线的角度误差
用法: python normal_quality.py original.png decompressed.png
"""

import numpy as np
from PIL import Image
import sys

def load_normal(path):
    """加载法线贴图，返回 [-1, 1] 范围的法线向量"""
    img = np.array(Image.open(path).convert("RGB"), dtype=np.float32)
    # [0, 255] → [-1, 1]
    normals = (img / 127.5) - 1.0
    # 归一化
    lengths = np.sqrt(np.sum(normals ** 2, axis=2, keepdims=True)) + 1e-8
    return normals / lengths

def angular_error(orig, decomp):
    """计算每像素的角度误差（度）"""
    dot = np.clip(np.sum(orig * decomp, axis=2), -1, 1)
    angles = np.degrees(np.arccos(dot))
    return angles

def analyze(original_path, decompressed_path):
    orig = load_normal(original_path)
    decomp = load_normal(decompressed_path)
    
    angles = angular_error(orig, decomp)
    
    print(f"=== 法线贴图压缩质量报告 ===")
    print(f"原始: {original_path}")
    print(f"解压: {decompressed_path}")
    print(f"")
    print(f"角度误差统计（度）:")
    print(f"  平均误差: {np.mean(angles):.3f}°")
    print(f"  中位数:   {np.median(angles):.3f}°")
    print(f"  90分位:   {np.percentile(angles, 90):.3f}°")
    print(f"  99分位:   {np.percentile(angles, 99):.3f}°")
    print(f"  最大误差: {np.max(angles):.3f}°")
    print(f"")
    
    # 质量评级
    mean_err = np.mean(angles)
    p99 = np.percentile(angles, 99)
    
    if mean_err < 1.0 and p99 < 5.0:
        grade = "S 级（极好）"
    elif mean_err < 2.0 and p99 < 10.0:
        grade = "A 级（好）"
    elif mean_err < 4.0 and p99 < 15.0:
        grade = "B 级（可接受）"
    elif mean_err < 8.0:
        grade = "C 级（需要优化）"
    else:
        grade = "D 级（不合格）"
    
    print(f"质量评级: {grade}")
    
    # 输出误差热力图
    heatmap = (angles / np.max(angles) * 255).astype(np.uint8)
    Image.fromarray(heatmap, mode="L").save("normal_error_heatmap.png")
    print(f"误差热力图已保存: normal_error_heatmap.png")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("用法: python normal_quality.py original.png decompressed.png")
        sys.exit(1)
    analyze(sys.argv[1], sys.argv[2])
```

**压缩格式对比表（1024×1024 法线贴图）**

| 格式 | 体积 | 精度（平均角度误差） | 平台 | 推荐场景 |
|------|------|---------------------|------|----------|
| 未压缩 RGBA32 | 4.0 MB | 0° | 全平台 | 仅开发参考 |
| BC5 (PC) | 1.0 MB | 0.3° | PC / 主机 | PC 端首选 |
| DXT5nm | 1.0 MB | 0.5° | PC / 主机 | 旧设备兼容 |
| ASTC 4×4 | 1.0 MB | 0.8° | 全平台 | 高品质移动端 |
| ASTC 6×6 | 0.59 MB | 1.2° | 全平台 | 移动端推荐 |
| ASTC 8×8 | 0.33 MB | 2.5° | 全平台 | 移动端经济型 |
| ETC2 RGB | 0.5 MB | 3.0° | Android | Android 兼容 |
| ETC2 RGBA | 1.0 MB | 2.5° | Android | Android 标准 |
| PVRTC 4bpp | 0.5 MB | 4.0° | iOS 旧 | 仅兼容用 |

### ⚡ 实战经验

1. **BC5nm 通道排布是常见踩坑点**：Unity 的 `TextureImporterType.NormalMap` 会自动处理通道重排。但如果用自定义 Shader 手动采样，必须知道 `tex2D.a` 是 X、`tex2D.g` 是 Y（DXT5nm 格式下）
2. **Mipmap 对法线的影响**：法线贴图的 Mipmap 生成不能简单 Box Filter——下采样后法线不再单位化，会导致光照偏暗。正确做法：下采样后重新归一化。Unity 的 Normal Map 导入器会自动做这个处理
3. **ASTC 是未来的统一答案**：iOS A8+ 和 Android 绝大多数 2016 年后的设备都支持 ASTC。如果你的最低配置允许，直接全平台 ASTC 可以大幅简化管线
4. **面部的法线贴图可以烘焙到顶点法线**：如果面部足够近，考虑用高精度 Mesh + 顶点法线替代法线贴图，这样完全没有纹理采样开销
5. **法线贴图 + Roughness 贴图共用一张图**：把法线 XY 放 RG 通道，Roughness 放 B 通道，可以减少一张纹理的开销。这是 PBR 流水线的常见优化

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道 Z 可以从 XY 推导 | 法线向量数学 | 复习向量数学 + Tangent Space 原理 |
| 不知道选哪种压缩格式 | 移动端 GPU 压缩格式 | 对比 BC5/ETC2/ASTC 精度与体积 |
| 压缩后法线看起来"平"了 | Mipmap + 压缩精度问题 | 学法线 Mipmap 归一化 + 角度误差分析 |
| 不同平台效果不一致 | 跨平台压缩差异 | 学 Unity Platform Texture Settings |
| 面部法线锯齿明显 | 高频细节的压缩损失 | 学 LOD 分级策略 + 高精度格式选用 |

### 🔗 相关问题

- 如何用 Substance Painter 导出符合移动端规范的法线贴图？（导出模板 + 通道配置）
- 法线贴图和细节法线贴图（Detail Normal）如何叠加？Blend Mode 怎么选？
- 为什么法线贴图导入 Unity 后颜色变了？（sRGB vs Linear 空间对法线的影响）
- 如何在 Shader 中手动从 XY 通道重建 Z？（`z = sqrt(1 - x*x - y*y)` 的精度陷阱）