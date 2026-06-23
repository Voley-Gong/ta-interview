---
title: "玩家反馈远处地形糊成马赛克——纹理流送和 Mip 偏差怎么调？"
category: "technical-art"
level: 3
tags: ["Texture Streaming", "Mip Chain", "Mipmap Bias", "内存预算", "带宽优化", "Unity", "UE"]
hint: "远处纹理模糊 = Mip 选错了层级 = Texture Streaming 没覆盖 / Mip Bias 设错 / 内存预算不够"
related: ["optimization/gpu-bandwidth-optimization", "optimization/gpu-memory-budget", "technical-art/mobile-texture-compression"]
---

## 参考答案

### 🎬 场景描述

美术跑来说：「场景中的草地贴图在近处看很清晰，但拉远看远处地形就变成马赛克了，整个地面像在糊。但是远处角色身上的盔甲贴图却很清晰，这是为什么？」

面试官追问：

第一，你能从渲染管线角度解释这个现象的根本原因吗？

第二，如果是 Texture Streaming 没有正确加载对应 Mip 层级，你会怎么排查和修复？

第三，手游上如果内存预算不够，不能加载所有 Mip，你作为 TA 会怎么制定贴图预算和 Streaming 策略？

追问一：Mipmap 的本质是什么？为什么有了 Mipmap 远处反而可能更模糊？

追问二：什么是 Mip Bias（LOD Bias）？正偏移和负偏移分别是什么效果？

### ✅ 核心要点

1. **远处模糊根因**：远处像素覆盖的 UV 范围大 → 需要 Mip 高层级（低分辨率）→ 如果 Streaming 没加载该 Mip → 降级到最低 Mip → 模糊
2. **Mipmap 双刃剑**：降低远处走样（Aliasing）+ 节省带宽，但过度偏移会让远处看起来比实际更糊
3. **Texture Streaming 机制**：按需加载 Mip 层级到 GPU 内存，不全量加载。决策依据是「这个纹理在屏幕上占多大」
4. **Mip Bias = 手动干预 Mip 选择**：正值 → 选更高层级 Mip（更模糊，省带宽）；负值 → 选更低层级 Mip（更清晰，耗带宽）
5. **贴图预算分配**：全场景贴图总内存远超手机可用内存 → Streaming 是刚需 → TA 需要制定优先级策略

### 📖 深度展开

#### 解决思路（从「远处马赛克」现象倒推）

```
现象：远处地形纹理糊成马赛克
                ↑
倒推1：远处像素在采样哪个 Mip？→ 应该是 Mip Level 3-5（降低走样）
倒推2：GPU 内存里有这个 Mip 吗？→ Texture Streaming 可能没加载
倒推3：Streaming 怎么决定加载哪些 Mip？→ 基于纹理在屏幕上的投影面积
倒推4：投影面积怎么算？→ 物体距离相机越远，纹理在屏幕上越小 → 需要更高 Mip 层
倒推5：为什么角色盔甲不糊？→ 盔甲贴图小（512x512），全部 Mip 都加载了，不存在 Streaming 降级
倒推6：地形贴图大（4096x4096 且平铺重复）→ Streaming 只加载了基础 Mip → 远处降级到 Mip 0 或最低可用层
倒推7：修复方向 → 检查 Streaming Pool 预算 / 手动设置地形贴图的 Streaming Priority / 调整 Mip Bias
```

#### 知识点拆解（倒推树）

```
纹理流送与 Mipmap 管理
├── Mipmap 基础
│   ├── 什么是 Mip Chain
│   │   ├── 原始分辨率 → 1/2 → 1/4 → ... → 1x1
│   │   ├── Level 0 = 原始分辨率（最近清晰）
│   │   └── Level N = 边长 / 2^N（最远模糊）
│   ├── 为什么需要 Mipmap
│   │   ├── 抗走样（远处多个纹素压缩到一个像素 → 如果只采样 Level 0 会 Moiré/闪烁）
│   │   ├── 提升缓存命中率（远处用小 Mip，纹素在 GPU Texture Cache 内）
│   │   └── 节省带宽（不需要全分辨率贴图来回传输）
│   └── Mip 选择原理
│       ├── GPU 硬件自动选择（基于 UV 导数 ddx/ddy）
│       ├── 屏幕像素覆盖面积大 → 高 Mip Level
│       └── 可通过 texture.LOD / bias 手动干预
├── Texture Streaming 机制
│   ├── Unity Texture Streaming
│   │   ├── 按 Mip 层级按需加载到 GPU
│   │   ├── Texture Streaming Priority（0-128，越高越优先保留高分辨率 Mip）
│   │   ├── Streaming Mipmap Active（勾选才参与 Streaming）
│   │   └── Memory Budget（总预算，超了先丢弃低优先级纹理的高 Mip）
│   ├── UE Virtual Texturing
│   │   ├── 虚拟纹理 = 纹理切 Tile + 按 Page（~128x128）按需加载
│   │   ├── 不以 Mip 为单位，而是以 Page 为单位（更细粒度）
│   │   ├── 适合超大纹理（8K/16K terrain、超大场景）
│   │   └── VT Page Cache → GPU 缓存命中率优于传统 Streaming
│   └── 决策逻辑
│       ├── 屏幕空间投影面积 → 需要的 Mip Level
│       ├── 当前 GPU 内存中有没有 → 没有就异步加载
│       └── 内存不够 → 驱逐低优先级纹理的高分辨率 Mip
├── Mip Bias / LOD Bias
│   ├── 正偏移（+0.5 → +2.0）→ 强制选更高 Mip（更模糊，省带宽/内存）
│   ├── 负偏移（-0.5 → -1.0）→ 强制选更低 Mip（更清晰，增带宽）
│   ├── 使用场景
│   │   ├── 手游带宽优化：全局 +0.5 Mip Bias（远处略糊但帧率提升）
│   │   ├── UI / 近景特写：负 Bias 确保清晰
│   │   └── VR 项目：不能容忍远处糊 → 负 Bias 或关闭 Streaming
│   └── 副作用
│       ├── 正 Bias → 远处细节丢失、抗锯齿过度
│       └── 负 Bias → 带宽飙升、可能引入 Moiré 闪烁
├── 贴图预算管理
│   ├── 手机端总预算参考
│   │   ├── 高端机（8GB+）：纹理预算 ~1.5-2GB
│   │   ├── 中端机（4-6GB）：纹理预算 ~512MB-1GB
│   │   └── 低端机（≤3GB）：纹理预算 ~256-512MB
│   ├── 单纹理预算规范
│   │   ├── 角色主体：2048x2048 × 5-8 张（Albedo/Normal/ORM/Emissive/Metalness）
│   │   ├── 场景 tileable：512-1024（重复利用 UV，不需要超大）
│   │   ├── 地形 splat：每层 512（不需要 4096）
│   │   ├── UI：1024-2048（全分辨率，不压缩或轻度压缩）
│   │   └── 天空盒：2048x1024（Half-res cubemap）
│   └── Streaming 优先级策略
│       ├── P0（始终加载全 Mip）：主角、UI、过场动画
│       ├── P1（高优先级）：当前视野内场景、交互物体
│       ├── P2（中等）：远景、装饰物
│       └── P3（低优先级）：远处在屏外的、小物体
└── 常见问题排查
    ├── 远处马赛克 → Streaming 没加载对应 Mip → 提升 Priority / 增加预算
    ├── 远处闪烁/Moiré → 没有 Mipmap 或 Mip Bias 太负 → 检查 Generate Mip / 加正 Bias
    ├── 加载卡顿（Pop-in）→ Streaming 加载速度不够 → 增大 I/O 预算 / 预加载
    └── 内存溢出 → 预算不够 → 降贴图分辨率 / 改用 ASTC 更高压缩比
```

#### 代码实现

**1. Unity Texture Streaming 配置与调试（C# 编辑器脚本）：**

```csharp
using UnityEngine;
using UnityEditor;
using System.Collections.Generic;
using System.Linq;

public class TextureStreamingAuditor : EditorWindow
{
    [MenuItem("Tools/TA/Texture Streaming Auditor")]
    static void Open() => GetWindow<TextureStreamingAuditor>("Texture Streaming Auditor");

    void OnGUI()
    {
        if (GUILayout.Button("Audit All Scene Textures"))
            AuditSceneTextures();

        if (GUILayout.Button("Find Textures Without Mipmaps"))
            FindNoMipTextures();

        if (GUILayout.Button("Calculate Total Texture Memory"))
            CalculateTotalMemory();
    }

    /// <summary>
    /// 审计场景中所有纹理的 Streaming 配置
    /// </summary>
    static void AuditSceneTextures()
    {
        var textures = new HashSet<Texture>();
        var renderers = FindObjectsByType<Renderer>(FindObjectsSortMode.None);

        foreach (var r in renderers)
        {
            foreach (var mat in r.sharedMaterials)
            {
                if (mat == null) continue;
                // 收集材质中所有纹理属性
                int propCount = ShaderUtil.GetPropertyCount(mat.shader);
                for (int i = 0; i < propCount; i++)
                {
                    if (ShaderUtil.GetPropertyType(mat.shader, i) == ShaderPropertyType.Texture)
                    {
                        string propName = ShaderUtil.GetPropertyName(mat.shader, i);
                        var tex = mat.GetTexture(propName);
                        if (tex != null) textures.Add(tex);
                    }
                }
            }
        }

        var report = textures
            .OrderByDescending(t => t.format.ToString().Contains("ASTC") ? 0 : 1)
            .ThenByDescending(t => t.width * t.height)
            .Select(t =>
            {
                long memBytes = Profiler.GetRuntimeMemorySizeLong(t);
                float memMB = memBytes / 1024f / 1024f;
                bool hasMip = t.mipmapCount > 1;
                bool streaming = t.streamingMipmaps;
                int priority = t.streamingMipmapsPriority;
                return string.Format(
                    "[{0,6:F1}MB] {1} | Mip:{2} | Stream:{3} | Pri:{4} | {5}x{6} | {7}",
                    memMB, t.name, hasMip ? "✓" : "✗",
                    streaming ? "ON" : "OFF",
                    priority, t.width, t.height, t.format
                );
            })
            .ToList();

        report.Insert(0, $"=== Texture Streaming Audit: {textures.Count} textures ===\n");
        Debug.Log(string.Join("\n", report));
    }

    /// <summary>
    /// 查找未生成 Mipmap 的纹理（可能导致远处闪烁）
    /// </summary>
    static void FindNoMipTextures()
    {
        var guids = AssetDatabase.FindAssets("t:Texture2D");
        var noMip = new List<string>();

        foreach (string guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            var importer = AssetImporter.GetAtPath(path) as TextureImporter;
            if (importer != null && !importer.mipmapEnabled &&
                !importer.textureShape.ToString().Contains("Cube")) // 跳过 Cubemap
            {
                // 只报告可能需要 Mip 的 2D 贴图（排除 UI、Lightmap 等）
                if (!path.Contains("UI/") && !path.Contains("Lightmap") &&
                    importer.maxTextureSize >= 512)
                {
                    noMip.Add($"[NO MIP] {path} ({importer.maxTextureSize}px)");
                }
            }
        }

        Debug.Log(noMip.Count > 0
            ? "Textures without Mipmaps:\n" + string.Join("\n", noMip)
            : "All qualifying textures have mipmaps ✓");
    }

    /// <summary>
    /// 计算场景总纹理内存（含 Mip Chain）
    /// </summary>
    static void CalculateTotalMemory()
    {
        QualitySettings.textureStreamingBudget = 1024; // 设置 Streaming Pool 预算（MB）

        long totalBytes = 0;
        var allTextures = Resources.FindObjectsOfTypeAll<Texture2D>();
        foreach (var tex in allTextures)
        {
            totalBytes += Profiler.GetRuntimeMemorySizeLong(tex);
        }

        float totalMB = totalBytes / 1024f / 1024f;
        Debug.Log($"Total Texture Memory: {totalMB:F1} MB | " +
                  $"Streaming Budget: {QualitySettings.textureStreamingBudget} MB");
    }
}
```

**2. Shader 中手动设置 Mip Bias（URP / HLSL）：**

```hlsl
// === 在 Shader 中使用 tex2Dlod 或 SampleLevel 手动控制 Mip ===
// 适合特殊场景：如地形 Splat 混合时手动选 Mip

// 方法1: tex2Dbias（手动偏移 GPU 自动选择的 Mip Level）
// bias 值会被加到 GPU 计算的 LOD 上
// 正 bias → 选更高 Mip（更模糊），负 bias → 选更低 Mip（更清晰）
float4 color = tex2Dbias(_MainTex, float4(uv.xy, 0, mipBias));

// 方法2: tex2Dlod（直接指定 Mip Level）
// 适合精确控制：比如远处强制用 Mip 3
float4 color = tex2Dlod(_MainTex, float4(uv.xy, 0, mipLevel));

// === 地形 Splat 贴图的 Mip Bias 优化 ===
// 问题：远处地形 Splat 混合时如果用了多个 1024x1024 Tileable 贴图
// 解决：远处强制 +1 Mip Bias → 省带宽、减少远处闪烁
float4 terrainColor = tex2Dbias(_SplatTex0, float4(uv * _TileScale, 0, 1.0)); // +1 bias
```

**3. UE5 虚拟纹理配置蓝图（伪代码参考）：**

```cpp
// UE5: 启用 Virtual Texture for Terrain
// 在编辑器中：Texture Editor → 勾选 "Virtual Texture"

// C++ 侧：通过 Streaming Pool 监控
#include "Engine/Texture2D.h"
#include "TextureStreaming.h"

void UTextureStreamingManager::LogVirtualTextureStats()
{
    // 获取当前 VT Page Cache 统计
    FTextureStreamingStats Stats = ITextureStreamingManagerModule::Get().GetStats();

    UE_LOG(LogTemp, Log, TEXT("=== Virtual Texture Stats ==="));
    UE_LOG(LogTemp, Log, TEXT("VT Page Cache: %d / %d pages (%.1f%% used)"),
        Stats.VT_CachedPages, Stats.VT_MaxPages,
        100.0f * Stats.VT_CachedPages / Stats.VT_MaxPages);
    UE_LOG(LogTemp, Log, TEXT("Physical Pool: %.1f / %.1f MB"),
        Stats.VT_PhysicalPoolUsedMB, Stats.VT_PhysicalPoolMaxMB);
    UE_LOG(LogTemp, Log, TEXT("Page Faults this frame: %d"), Stats.VT_PageFaults);
}
```

**Mipmap 选择示意表（以 1024x1024 纹理为例）：**

| 物体距相机 | 屏幕投影大小 | GPU 自动选择 Mip | 分辨率 | 内存大小 |
|-----------|------------|-----------------|--------|---------|
| 1m（贴脸） | ~全屏 | Level 0 | 1024×1024 | ~1.3 MB |
| 5m | ~1/4 屏 | Level 1 | 512×512 | ~340 KB |
| 20m | ~1/16 屏 | Level 3 | 128×128 | ~21 KB |
| 50m | ~很小 | Level 5 | 32×32 | ~1.3 KB |
| 100m+ | ~像素级 | Level 7+ | 8×8 | ~85 B |
| **总计（全 Mip Chain）** | — | — | — | **~1.7 MB** |

> 注意：全 Mip Chain 总共只有原始大小的 ~1.33 倍（等比级数和 1 + 1/4 + 1/16 + ... ≈ 4/3）。但场景中有数千张纹理时，Streaming 的价值是只加载可见的 Mip。

### ⚡ 实战经验

- **先排除不是 Streaming 问题**：如果所有 Mip 都正确生成了，远处依然糊，那可能是 Mip Bias 太正了（有人全局加了 +1 来省带宽但牺牲了画质）
- **地形贴图是重灾区**：大面积地形用 Splat 混合 4-8 层贴图，每层重复平铺，UV 密度高。远处 8 层各选不同 Mip → 混合后像素精度极差。建议地形 Splat 层用 512 而非 1024，减少不必要的带宽
- **Streaming Pop-in 是另一个问题**：快速移动时远处纹理从低 Mip 突然跳到高 Mip（「弹入」），非常明显。解决方案：预加载前方视角的 Mip（Look-ahead Streaming）、增大 I/O 预算、或使用 VT（Virtual Texturing）替代传统 Streaming
- **VR 项目基本不能开 Streaming**：VR 双目视野中远处纹理突然弹入极度影响沉浸感。VR 建议关闭 Streaming + 负 Mip Bias
- **ASTC 压缩与 Mip 的关系**：ASTC 6×6 块压缩下，每加一层 Mip 节省的空间更显著（因为块大小固定，分辨率减半意味着块数减少 75%）。所以 ASTC + Mipmap 是手游标配
- **真机调试命令**：Unity 可以用 `Texture.streamingTextureCount`、`Texture.streamingTextureMemorySize` 实时监控；UE 有 `stat streaming` 和 VT Visualizer

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道远处为什么会糊 | Mipmap 选择原理 | 学习 GPU 纹理采样、ddx/ddy 导数 |
| 不理解 Texture Streaming 的决策逻辑 | Streaming 按需加载机制 | 学习 Unity/UE Streaming 文档 |
| Mip Bias 正负方向搞反 | LOD 选择公式 | 动手在引擎中调 Mip Bias 对比效果 |
| 不知道怎么计算贴图内存 | 纹理内存公式 | 学习 像素数 × 字节/像素 ÷ 压缩比 |
| VT（虚拟纹理）不理解 | VT Page 系统 | 学习 UE Virtual Texturing 文档 |

### 🔗 相关问题

- Texture Streaming 的 Pop-in（纹理弹入）问题怎么解决？（提示：预加载 + Look-ahead + 淡入过渡）
- 如果一张 4096 贴图占了 20MB，用 ASTC 6×6 能压到多少？质量损失大吗？
- UE5 的 Virtual Texturing 和传统 Texture Streaming 的本质区别是什么？什么场景该用 VT？
