---
title: "GPU 显存预算：3A 手游场景下显存超预算该怎么排查和削减？"
category: "optimization"
level: 3
tags: ["GPU显存", "内存预算", "纹理内存", "Mesh内存", "RenderTarget", "性能优化", "移动端"]
hint: "先量后砍——用 RenderDoc/XCode 抓帧分类显存占用，然后按贴图→RT→Mesh→Buffer 的优先级逐项削减"
related: ["optimization/gpu-bandwidth-optimization", "optimization/drawcall-500-to-100", "optimization/vertex-bound-bottleneck"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的开放世界手游在 iPhone 13 上出现了显存压力——峰值占用超过 1.2GB，导致系统杀进程。Android 端也有 OOM 崩溃。美术资源已经做了 LOD，但纹理和 Render Target 依然占大头。你作为 TA，怎么系统性地排查和削减 GPU 显存？给我完整的排查流程和优化方案。」

这是腾讯、网易、米哈游等大世界项目的真实场景题。考察的不是"知道哪个 API 省内存"，而是**系统化的显存审计能力**——测量、归因、优先级、方案落地。

### ✅ 核心要点

1. **先量化再优化**：抓帧分析显存构成，不靠猜
2. **贴图是显存大户**：通常占 50-70%，压缩格式和尺寸是关键
3. **Render Target 隐形占用**：后处理 RT、阴影 RT、G-Buffer 在移动端开销惊人
4. **Mesh 和 Buffer 不可忽视**：高模、Morph Target、Compute Buffer
5. **分级加载策略**：近景全精度、中景降级、远景用 Imposter
6. **平台差异化预算**：iOS/Android/PC 显存上限不同，需要分级策略

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
问题：GPU 显存峰值 1.2GB → 需要降到 800MB 以下（iPhone 13 安全线）

排查路径：
Step 1：抓帧 → 分类显存占用
  ├── 纹理 X MB（占比 ?%）
  ├── Render Target Y MB（占比 ?%）
  ├── Mesh / Vertex Buffer Z MB
  ├── Index Buffer W MB
  └── 其他（Shader / Uniform / Compute）

Step 2：按占比排序 → 从最大项开始优化

Step 3：纹理优化（通常第一个砍）
  ├── 未压缩纹理 → ASTC/BC 压缩
  ├── 过大尺寸 → Mip 链审计，合理最大分辨率
  ├── 冗余通道 → Alpha 分离打包 / Channel Packing
  └── 纹理图集 → 合并小贴图

Step 4：Render Target 优化
  ├── 后处理 RT 格式降级（RGBAHalf → RGB9E5）
  ├── 阴影贴图分辨率 → 视距分级
  ├── MSAA 降级 or 替代方案
  └── 临时 RT 复用 / 生命周期管理

Step 5：Mesh 优化
  ├── 顶点数审计 → 离屏面剔除
  ├── Morph Target 内存 → 关键表情 only
  └── Vertex Format 压缩（Float32 → Half/UNorm）

Step 6：Buffer 优化
  ├── Compute Buffer 的使用审计
  ├── CBUFFER 对齐与复用
  └── Structured Buffer vs Texture 选择

Step 7：分级加载
  ├── 纹理 Mip Streaming（Unity Texture Streaming）
  ├── 场景分块异步加载
  └── Imposter 替换远景
```

#### 知识点拆解（倒推树）

```
GPU 显存优化
├── 显存审计（先量后砍）
│   ├── 工具链
│   │   ├── RenderDoc（帧抓取 + Resource Inspector）
│   │   ├── Xcode GPU Frame Capture（iOS 专用）
│   │   ├── Android GPU Inspector / Snapdragon Profiler
│   │   ├── Unity Frame Debugger + Profiler
│   │   └── Unreal stat unitres / ProfileGPU
│   ├── 分类统计
│   │   ├── 纹理内存（按材质/物体分组）
│   │   ├── RT 内存（按 Pass 分组）
│   │   ├── Mesh 内存（顶点数 × stride）
│   │   └── Buffer 内存（CB / SB / UAV）
│   └── 热力图分析（哪个场景/区域峰值最高）
├── 纹理优化
│   ├── 压缩格式选择
│   │   ├── 移动端：ASTC（6x6 平衡，4x4 高品质）
│   │   ├── PC：BC7（高品质）/ BC1（Diffuse）
│   │   ├── iOS 特殊：PVRTC 已过时，统一 ASTC
│   │   └── 法线贴图：BC5 / ASTC + 双通道重构
│   ├── 分辨率策略
│   │   ├── Mip 链生成控制（手动指定 max resolution）
│   │   ├── Texture Streaming（按距离动态加载 Mip）
│   │   ├── UI 纹理固定尺寸 vs 3D 纹理可降级
│   │   └── 法线/细节贴图可更小（视觉感知阈值）
│   ├── Channel Packing
│   │   ├── ORM 贴图（Occlusion/Roughness/Metallic → RGB）
│   │   ├── Smoothness + AO + Detail Mask 合并
│   │   └── 减少 3 张 → 1 张 = 省 2/3
│   ├── 纹理图集
│   │   ├── 角色共用图集（512 × 多个角色 → 2048 一张）
│   │   ├── 场景 Tile Atlas
│   │   └── 注意 Mip 边缘泄漏
│   └── 纹理池化与卸载
│       ├── 引用计数管理
│       ├── 场景切换时卸载策略
│       └── Resources.UnloadUnusedAssets() 时机
├── Render Target 优化
│   ├── 格式选择
│   │   ├── HDR 场景：RGBAHalf(8B/px) vs RGB9E5(4B/px) vs R11G11B10(4B/px)
│   │   ├── LDR 场景：RGBA8(4B/px) 足够
│   │   ├── 深度贴图：D32(Depth32) vs D24S8
│   │   └── 阴影贴图：R16F 足够（不需要 RGBA）
│   ├── 阴影贴图优化
│   │   ├── 分辨率分级（近 2048 / 远 1024）
│   │   ├── 多光源阴影池化
│   │   └── 移动端阴影算法替代（Shadow Blob / Projected）
│   ├── 后处理 RT 复用
│   │   ├── 同尺寸 RT 共享（Bloom / Color Grading 复用）
│   │   ├── 生命周期管理（Pass 间及时 Release）
│   │   └── 降采样 RT（半分辨率做 Bloom / DOF）
│   └── MSAA
│       ├── 4x MSAA = 4 倍显存（移动端慎用）
│       ├── 替代：FXAA / TAA（Post-process，无显存膨胀）
│       └── Alpha-to-coverage 替代 Alpha Test
├── Mesh 优化
│   ├── 顶点格式压缩
│   │   ├── Position: Float3 → Float16 / UNorm16
│   │   ├── Normal: Float3 → UNorm8（1010102 格式）
│   │   ├── UV: Float2 → Half2 / UNorm16
│   │   └── Tangent: Float3 → 1010102
│   ├── LOD 审计
│   │   ├── LOD 切换距离合理
│   │   ├── 最低 LOD 顶点数（< 500）
│   │   └── Imposter 替换超远景
│   ├── Morph Target（Blend Shape）
│   │   ├── 只保留关键表情（裁剪冗余 shape）
│   │   ├── 系数精度压缩
│   │   └── 移动端限制数量（< 20 个）
│   └── 网格压缩
│       ├── Mesh Compression（Unity 内置）
│       ├── Index Buffer 16-bit vs 32-bit
│       └── Sub-mesh 合并减少 Draw Call
├── Buffer 优化
│   ├── Structured Buffer 审计
│   │   ├── 超大粒子数据 → 分帧上传
│   │   ├── Compute Buffer 复用
│   │   └── GPU Skin 矩阵缓存
│   ├── Constant Buffer 对齐
│   │   ├── 256B 对齐（D3D12）/ 16B（Vulkan）
│   │   ├── 合并小 CBUFFER
│   │   └── 避免 padding 浪费
│   └── 间接绘制 buffer（IndirectDraw）
│       └── 减少多 buffer 为单 buffer + offset
├── 分级加载与流式
│   ├── Unity Texture Streaming
│   │   ├── 设定预算上限（Memory Budget）
│   │   ├── 优先级标记（重要纹理不降级）
│   │   └── Mip 加载偏移（先加载低 Mip）
│   ├── 场景分块
│   │   ├── Grid 分块 + 距离加载
│   │   ├── 优先级队列（玩家前方优先）
│   │   └── 卸载策略（身后 N 秒卸载）
│   └── Imposter
│       ├── 远景建筑/树木替换为 Billboard
│       ├── 多角度快照 + 插值
│       └── 显存占用从 MB 级降到 KB 级
└── 平台分级预算
    ├── iOS
    │   ├── A14+：~1.5GB GPU 预算
    │   ├── A12-A13：~1.0GB
    │   └── 内存警告 → 降级策略
    ├── Android
    │   ├── 高端（骁龙8 Gen2+）：~1.5GB
    │   ├── 中端：~800MB
    │   └── 低端回退包
    └── PC
        ├── 独显：按需分配
        └── 集显：按内存比例
```

#### 代码实现

**显存预算管理器（Unity C#）：**

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;

/// <summary>
/// GPU 显存预算管理器
/// 按分类追踪纹理、RT、Mesh 的预估显存占用
/// </summary>
public class GPUMemoryBudgetManager : MonoBehaviour
{
    [System.Serializable]
    public class PlatformBudget
    {
        public string platformName;
        public long totalBudgetBytes;       // 总预算
        public long warningThresholdBytes;   // 警告阈值（80%）
        public long criticalThresholdBytes;  // 危险阈值（90%）
    }

    [Header("平台预算")]
    public PlatformBudget iOSHigh = new() { platformName = "iOS High (A14+)", totalBudgetBytes = 1500_000_000, warningThresholdBytes = 1_200_000_000, criticalThresholdBytes = 1_350_000_000 };
    public PlatformBudget iOSMid = new() { platformName = "iOS Mid (A12-A13)", totalBudgetBytes = 1_000_000_000, warningThresholdBytes = 800_000_000, criticalThresholdBytes = 900_000_000 };
    public PlatformBudget androidHigh = new() { platformName = "Android High", totalBudgetBytes = 1_500_000_000, warningThresholdBytes = 1_200_000_000, criticalThresholdBytes = 1_350_000_000 };
    public PlatformBudget androidMid = new() { platformName = "Android Mid", totalBudgetBytes = 800_000_000, warningThresholdBytes = 640_000_000, criticalThresholdBytes = 720_000_000 };

    [Header("当前状态")]
    [SerializeField] private long currentEstimatedUsage;
    [SerializeField] private BudgetLevel currentLevel = BudgetLevel.Safe;

    public enum BudgetLevel { Safe, Warning, Critical }

    private PlatformBudget _activeBudget;
    private Dictionary<string, long> _categoryUsage = new();

    /// <summary>
    /// 估算纹理显存占用（字节）
    /// </summary>
    public static long EstimateTextureMemory(Texture tex)
    {
        if (tex == null) return 0;

        int w = tex.width;
        int h = tex.height;
        int mipCount = tex.mipmapCount > 1 ? tex.mipmapCount : 1;

        // 压缩格式每像素位数
        int bitsPerPixel = tex.graphicsFormat switch
        {
            // ASTC 6x6 = 3.56 bpp
            UnityEngine.Experimental.Rendering.GraphicsFormat.RGBA_ASTC_6x6_SRGB_Block => 4,
            // ASTC 4x4 = 8 bpp
            UnityEngine.Experimental.Rendering.GraphicsFormat.RGBA_ASTC_4x4_SRGB_Block => 8,
            // BC7 = 8 bpp
            UnityEngine.Experimental.Rendering.GraphicsFormat.RGBA_DXT5_SRGB_Block => 8,
            // RGBA32 = 32 bpp
            UnityEngine.Experimental.Rendering.GraphicsFormat.R8G8B8A8_SRGB => 32,
            // RGBAHalf = 64 bpp
            UnityEngine.Experimental.Rendering.GraphicsFormat.R16G16B16A16_SFloat => 64,
            // 默认按 32 bpp
            _ => 32
        };

        long bytes = 0;
        for (int i = 0; i < mipCount; i++)
        {
            int mipW = Mathf.Max(1, w >> i);
            int mipH = Mathf.Max(1, h >> i);
            // 压缩格式按 block 计算
            if (bitsPerPixel <= 8)
            {
                // ASTC/BC：block-based
                int blockSize = bitsPerPixel <= 4 ? 6 : 4;
                int blocksX = Mathf.CeilToInt(mipW / (float)blockSize);
                int blocksY = Mathf.CeilToInt(mipH / (float)blockSize);
                bytes += blocksX * blocksY * 16; // 每个 block 16 字节
            }
            else
            {
                bytes += mipW * mipH * bitsPerPixel / 8;
            }
        }

        return bytes;
    }

    /// <summary>
    /// 估算 Render Target 显存
    /// </summary>
    public static long EstimateRTMemory(int width, int height, RenderTextureFormat format, int depth = 0)
    {
        int bpp = format switch
        {
            RenderTextureFormat.ARGB32 => 32,
            RenderTextureFormat.RGBA32 => 32,
            RenderTextureFormat.ARGBHalf => 64,
            RenderTextureFormat.RGBAFloat => 128,
            RenderTextureFormat.RGFloat => 64,
            RenderTextureFormat.RHalf => 16,
            RenderTextureFormat.Depth => 24,
            RenderTextureFormat.Shadowmap => 32,
            _ => 32
        };

        long colorBytes = width * height * bpp / 8L;
        long depthBytes = depth > 0 ? width * height * depth / 8L : 0;
        return colorBytes + depthBytes;
    }

    void Start()
    {
        _activeBudget = SelectPlatformBudget();
        Debug.Log($"[GPU Budget] Active budget: {_activeBudget.platformName}, Total: {_activeBudget.totalBudgetBytes / 1_000_000}MB");
    }

    PlatformBudget SelectPlatformBudget()
    {
#if UNITY_IOS
        // 简化的设备分级
        return SystemInfo.processorType.Contains("A14") || SystemInfo.processorType.Contains("A15")
            ? iOSHigh : iOSMid;
#elif UNITY_ANDROID
        return SystemInfo.systemMemorySize >= 8192 ? androidHigh : androidMid;
#else
        return iOSHigh; // 编辑器默认
#endif
    }

    void Update()
    {
        currentEstimatedUsage = QueryTotalEstimatedMemory();
        currentLevel = EvaluateLevel();

        if (currentLevel == BudgetLevel.Warning && Time.frameCount % 300 == 0)
        {
            Debug.LogWarning($"[GPU Budget] WARNING: {currentEstimatedUsage / 1_000_000}MB / {_activeBudget.totalBudgetBytes / 1_000_000}MB");
            ApplyDegradation(degradeFactor: 0.7f);
        }
        else if (currentLevel == BudgetLevel.Critical)
        {
            Debug.LogError($"[GPU Budget] CRITICAL: {currentEstimatedUsage / 1_000_000}MB / {_activeBudget.totalBudgetBytes / 1_000_000}MB");
            ApplyDegradation(degradeFactor: 0.5f);
        }
    }

    BudgetLevel EvaluateLevel()
    {
        if (currentEstimatedUsage >= _activeBudget.criticalThresholdBytes)
            return BudgetLevel.Critical;
        if (currentEstimatedUsage >= _activeBudget.warningThresholdBytes)
            return BudgetLevel.Warning;
        return BudgetLevel.Safe;
    }

    /// <summary>
    /// 降级策略：降低纹理 Streaming 预算、降 RT 分辨率
    /// </summary>
    void ApplyDegradation(float degradeFactor)
    {
        // 降低 Texture Streaming 预算
        QualitySettings.globalTextureMipmapLimit = degradeFactor < 0.6f ? 1 : 0;

        // 降低后处理 RT 分辨率
        var urpAsset = UniversalRenderPipeline.asset;
        if (urpAsset != null && degradeFactor < 0.6f)
        {
            urpAsset.renderScale = 0.85f; // 降低渲染分辨率
        }
    }

    long QueryTotalEstimatedMemory()
    {
        // 简化版：遍历场景中所有 Renderer 的纹理
        // 实际项目应对接引擎 Profiler API 或自定义统计
        long total = 0;
        var renderers = FindObjectsByType<Renderer>(FindObjectsSortMode.None);
        foreach (var r in renderers)
        {
            foreach (var mat in r.sharedMaterials)
            {
                if (mat == null) continue;
                foreach (var nameID in mat.GetTexturePropertyNames())
                {
                    var tex = mat.GetTexture(nameID);
                    if (tex != null) total += EstimateTextureMemory(tex);
                }
            }
        }
        return total;
    }
}
```

**RenderDoc 显存审计清单：**

```
RenderDoc 抓帧后 → Texture Viewer → 按 Size 排序
┌──────────────────────────────────────────────────────────────┐
│ 检查项                        │ 典型问题                      │
├──────────────────────────────────────────────────────────────┤
│ 最大纹理尺寸                  │ 是否有 4096×4096 未压缩？      │
│ 重复纹理                      │ 同一张纹理是否多次加载？       │
│ 未使用纹理                    │ 是否有绑定但 Shader 不采样的？ │
│ RT 生命周期                   │ RT 在 Pass 间是否及时释放？    │
│ Mip 链完整性                  │ 是否缺少 Mip 导致大纹理常驻？  │
│ 压缩格式                      │ 是否有未压缩的 RGBA32？        │
│ 通道冗余                      │ 单通道用 RGBA 是否合理？       │
└──────────────────────────────────────────────────────────────┘
```

**显存优化优先级矩阵：**

| 优化项 | 收益（典型） | 实施难度 | 风险 | 优先级 |
|--------|------------|---------|------|--------|
| RGBA32→ASTC 6x6 | 省 80%+ | 低 | 画质微降 | ★★★★★ |
| Channel Packing（3→1） | 省 66% | 中 | 需改 Shader | ★★★★★ |
| RT 格式降级（Half→9E5） | 省 50% | 低 | HDR 精度微降 | ★★★★☆ |
| Texture Streaming | 省 30-50% | 中 | Mip pop-in | ★★★★☆ |
| 阴影贴图分级 | 省 50-75% | 中 | 远景阴影质量 | ★★★★☆ |
| Mesh 顶点格式压缩 | 省 40-60% | 高 | 变形精度 | ★★★☆☆ |
| Imposter 替换远景 | 省 90%+ | 高 | 近距离穿帮 | ★★★☆☆ |
| Morph Target 裁剪 | 省 30-50% | 中 | 表情限制 | ★★☆☆☆ |

### ⚡ 实战经验

- **先量后砍**：不要凭感觉优化——RenderDoc 抓一帧，按大小排序，Top 10 占了 70% 以上的显存
- **Texture Streaming 是双刃剑**：省了显存但可能造成 Mip pop-in，关键纹理（UI、角色面部）应标记为 No Streaming
- **ASTC 块大小选择**：6x6 是移动端甜点（4bpp），法线贴图用 5x5（5.12bpp），UI 用 4x4（8bpp）
- **Render Target 是隐形杀手**：一张 1080p RGBAHalf RT = 16MB，后处理链 5-6 张 RT 就接近 100MB
- **iPhone OOM 和 Android OOM 不同**：iOS 是系统直接杀进程（Jetsam），Android 是 GPU 驱动崩溃或系统回收——监控方式不同
- **Mip 链的隐形成本**：每增加一级 Mip，显存增加 1/4（等比数列总和 ≈ 1.33×），但 Mip 是必要的——关键是控制最大分辨率
- **Compute Buffer 的坑**：`new ComputeBuffer(count, stride)` 即使不写数据也会分配显存——用完必须 Release()

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道显存被什么占了 | 抓帧工具使用 | 学 RenderDoc / Xcode GPU Capture |
| 纹理压了还是大 | 压缩格式选择 | 学 ASTC/BC7 格式特性与 bpp 计算 |
| RT 占用失控 | RT 生命周期管理 | 学 URP RT 分配机制与复用策略 |
| 优化后画质崩 | 感知优先级 | 学人眼对分辨率/色彩/精度的敏感度差异 |
| 无法量化优化效果 | Profiler 工具 | 学 Unity Memory Profiler / 自建统计 |
| 跨平台预算冲突 | 平台分级策略 | 学设备性能分级与 Quality Settings 联动 |

### 🔗 相关问题

- 如何建立项目级的美术资源规范来预防显存超标？（提示：导入管线 + CI 检查）
- Unity 的 Texture Streaming 在什么场景下会失效？（提示：Render Texture、ComputeBuffer、手动 Graphics.Blit）
- 延迟渲染在移动端的显存开销为何比前向渲染大很多？（提示：G-Buffer 多 RT）
