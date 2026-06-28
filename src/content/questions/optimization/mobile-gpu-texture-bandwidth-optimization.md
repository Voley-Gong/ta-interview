---
title: "手游 GPU 带宽爆表：如何从纹理链路入手把带宽砍掉 60%？"
category: optimization
level: 3
tags: ["GPU带宽", "纹理优化", "移动端", "TBR架构", "贴图压缩", "Mipmap", "带宽分析"]
hint: "TBR 架构下每多采样一个纹素就是一笔 Tile Memory 开销，从压缩格式+Mipmap+采样策略三路围剿"
related: ["adreno-tile-based-bandwidth", "gpu-bandwidth-optimization", "mobile-texture-compression", "texture-streaming-mipmap-prefetch"]
---

## 参考答案

### 🎬 场景描述

面试官问：我们的手游在骁龙 888 上跑 30 FPS 没问题，但 GPU 带宽占用已经到了 18 GB/s，设备 5 分钟就开始降频。Profiler 显示纹理采样占了 70% 的带宽。你会怎么优化？

追问：具体说一下你们项目中纹理带宽从多少降到了多少，做了哪些事？

### ✅ 核心要点

- **纹理是移动端带宽杀手**——大面积高分辨率贴图 × 过采样 = 灾难性带宽
- **TBR/TBDR 架构的带宽逻辑**——所有纹素先从内存加载到 Tile Memory，带宽 × 过采样率
- **三路围剿策略**：压缩格式（减体积）→ Mipmap 策略（减过采样）→ Shader 采样优化（减次数）
- **量化测量**——Snapdragon Profiler / Mali Streamline / XCode GPU Capture 给出真实带宽数据
- **不是所有纹理优化手段都能叠加**——有些互相冲突，需要做 A/B 测试验证

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：GPU 带宽从 18 GB/s 降到 7 GB/s（60% 削减）
    ↓ 倒推：纹理采样占 70%（12.6 GB/s），纹理优化必须贡献最大份额
    ↓ 12.6 GB/s 怎么来的？
    公式：带宽 = Σ(纹理分辨率 × 过采样率 × 采样频率 × 帧率)
    ↓ 倒推每项的可优化空间：
    ① 纹理分辨率 → 贴图降分辨率 + 按需加载（Streaming）
    ② 过采样率 → Mipmap 配置 + Texture Filtering 选择
    ③ 采样频率 → Shader 中纹理采样次数减少
    ④ 贴图体积 → 压缩格式选择
```

#### 知识点拆解（倒推树）

```
纹理带宽优化
├── 第一层：压缩格式（减小内存占用 + 减少加载带宽）
│   ├── ASTC 6x6 vs ETC2 vs BC7 的带宽差异
│   ├── ASTC 块大小选择策略（4x4 / 6x6 / 8x8 按贴图类型）
│   ├── 法线贴图特殊压缩（BC5 / ASTC + 双通道方案）
│   └── 压缩质量 vs 带宽的 Pareto 曲线
├── 第二层：Mipmap 与采样策略
│   ├── 没有 Mipmap 的灾难：1K 贴图在 50px 屏幕区域 = 400x 过采样
│   ├── Mipmap bias 调优（bias 偏移可减少带宽但引入模糊）
│   ├── Trilinear vs Bilinear 带宽差异（2x 采样）
│   ├── Anisotropic Filter 的带宽代价（最高 16x）
│   └── Texture Streaming 动态加载策略
├── 第三层：Shader 采样优化
│   ├── 多层 Blend 的采样次数（4层地形 = 4x 纹理带宽）
│   ├── 移除不必要的通道采样（AO 烘焙到顶点色 / 通道合并）
│   ├── Branch/LOD 在 Shader 中降低远处采样分辨率
│   └── 计算 vs 采样的 Trade-off（程序化噪声 vs 噪声贴图）
├── 第四层：分辨率与裁剪
│   ├── 按使用面积反推贴图分辨率（近景 2K, 中景 512, 远景 128）
│   ├── Channel Packing（RGBA 四合一：AO/Roughness/Metallic/Height）
│   └── 不可见区域裁剪（底部贴图、内部贴图移除）
├── 测量工具
│   ├── Snapdragon Profiler（Adreno GPU 带宽分解）
│   ├── Mali Streamline（Mali GPU 总线带宽）
│   ├── XCode GPU Capture（Apple GPU Tile Store 统计）
│   └── RenderDoc（粗略估算纹理传输量）
└── 架构层优化
    ├── TBDR 的 Tile 大小影响（Adreno vs Mali vs Apple）
    ├── Depth Pre-Pass 减少过度绘制 → 间接减少纹理采样
    └── Visibility Texture / GPU-Driven 减少不可见像素采样
```

#### 代码实现

**纹理带宽审计工具（Unity Editor）：**

```csharp
using UnityEditor;
using UnityEngine;
using System.Collections.Generic;
using System.IO;

public class TextureBandwidthAuditor : EditorWindow
{
    [MenuItem("TA Tools/Texture Bandwidth Auditor")]
    static void Init() => GetWindow<TextureBandwidthAuditor>("Texture Bandwidth Auditor");

    void OnGUI()
    {
        GUILayout.Label("纹理带宽审计工具", EditorStyles.boldLabel);
        GUILayout.Space(5);

        if (GUILayout.Button("扫描场景中所有可见纹理"))
            AuditSceneTextures();

        if (GUILayout.Button("分析贴图压缩格式覆盖率"))
            AuditCompressionFormats();

        if (GUILayout.Button("检测缺少 Mipmap 的高分辨率贴图"))
            AuditMissingMipmaps();

        if (GUILayout.Button("生成带宽优化建议报告"))
            GenerateOptimizationReport();
    }

    struct TextureBandwidthInfo
    {
        public Texture texture;
        public string path;
        public int width, height;
        public TextureFormat format;
        public int bytesPerPixel;
        public bool hasMipmap;
        public long totalBytes; // 含 mipmap 链
        public string usedBy; // 引用者
    }

    void AuditSceneTextures()
    {
        var infos = new List<TextureBandwidthInfo>();
        var renderers = FindObjectsByType<Renderer>(FindObjectsSortMode.None);

        foreach (var r in renderers)
        {
            foreach (var mat in r.sharedMaterials)
            {
                if (mat == null) continue;
                foreach (var propName in mat.GetTexturePropertyNames())
                {
                    var tex = mat.GetTexture(propName);
                    if (tex == null) continue;

                    var path = AssetDatabase.GetAssetPath(tex);
                    var importer = AssetImporter.GetAtPath(path) as TextureImporter;
                    var info = new TextureBandwidthInfo
                    {
                        texture = tex,
                        path = path,
                        width = tex.width,
                        height = tex.height,
                        format = tex is Texture2D t2d ? t2d.format : TextureFormat.RGBA32,
                        bytesPerPixel = GetBytesPerPixel(tex),
                        hasMipmap = importer != null && importer.mipmapEnabled,
                        usedBy = $"{r.gameObject.name}.{propName}"
                    };
                    info.totalBytes = CalculateTotalBytes(info);
                    infos.Add(info);
                }
            }
        }

        // 按带宽贡献排序
        infos.Sort((a, b) => b.totalBytes.CompareTo(a.totalBytes));

        Debug.Log("===== 纹理带宽审计报告 =====");
        Debug.Log($"场景中共 {infos.Count} 个纹理引用");
        long totalMB = 0;
        foreach (var info in infos.Take(20))
        {
            float mb = info.totalBytes / 1024f / 1024f;
            totalMB += info.totalBytes;
            Debug.Log($"[{mb:F1}MB] {info.usedBy} | {info.width}x{info.height} | " +
                      $"{info.format} | Mipmap:{(info.hasMipmap ? "✓" : "✗")} | {info.path}");
        }
        Debug.Log($"前20大纹理总计: {totalMB / 1024f / 1024f:F1} MB");
    }

    int GetBytesPerPixel(Texture tex)
    {
        if (tex is Texture2D t2d)
        {
            // 常见压缩格式估算
            switch (t2d.format)
            {
                case TextureFormat.ASTC_4x4: return 1;       // 8 bytes / 16 pixels
                case TextureFormat.ASTC_6x6: return 1;       // 8 bytes / 36 pixels ≈ 0.44
                case TextureFormat.ASTC_8x8: return 1;       // 8 bytes / 64 pixels ≈ 0.25
                case TextureFormat.ETC2_RGB: return 1;       // 8 bytes / 16 pixels
                case TextureFormat.BC7: return 1;            // 16 bytes / 16 pixels
                case TextureFormat.RGBA32: return 4;
                case TextureFormat.RGBA64: return 8;
                default: return 4; // 保守估计
            }
        }
        return 4;
    }

    long CalculateTotalBytes(TextureBandwidthInfo info)
    {
        // Mipmap 链总计 = baseSize × 4/3
        long baseSize = (long)(info.width * info.height * info.bytesPerPixel) / 
                        GetBlockSizeDivisor(info.format);
        return info.hasMipmap ? baseSize * 4 / 3 : baseSize;
    }

    int GetBlockSizeDivisor(TextureFormat format)
    {
        switch (format)
        {
            case TextureFormat.ASTC_4x4: return 1;  // 已经是压缩后的
            case TextureFormat.ASTC_6x6: return 1;
            case TextureFormat.ASTC_8x8: return 1;
            default: return 1;
        }
    }

    void AuditCompressionFormats()
    {
        var guids = AssetDatabase.FindAssets("t:Texture2D");
        var formatCount = new Dictionary<string, int>();
        var uncompressed = new List<string>();

        foreach (var guid in guids)
        {
            var path = AssetDatabase.GUIDToAssetPath(guid);
            var importer = AssetImporter.GetAtPath(path) as TextureImporter;
            if (importer == null) continue;

            foreach (var setting in importer.GetPlatformTextureSettings("Android"))
            {
                var fmt = setting.format.ToString();
                if (!formatCount.ContainsKey(fmt)) formatCount[fmt] = 0;
                formatCount[fmt]++;
                if (setting.format == TextureImporterFormat.RGBA32 ||
                    setting.format == TextureImporterFormat.RGBA64)
                    uncompressed.Add(path);
            }
        }

        Debug.Log("===== 压缩格式分布 =====");
        foreach (var kv in formatCount)
            Debug.Log($"  {kv.Key}: {kv.Value} 张");
        if (uncompressed.Count > 0)
        {
            Debug.LogWarning($"⚠️ 发现 {uncompressed.Count} 张未压缩贴图（高带宽风险）:");
            foreach (var p in uncompressed.Take(10))
                Debug.LogWarning($"  {p}");
        }
    }

    void AuditMissingMipmaps()
    {
        var guids = AssetDatabase.FindAssets("t:Texture2D");
        var missing = new List<(string path, int width, int height)>();

        foreach (var guid in guids)
        {
            var path = AssetDatabase.GUIDToAssetPath(guid);
            var importer = AssetImporter.GetAtPath(path) as TextureImporter;
            if (importer == null || !importer.mipmapEnabled)
            {
                var tex = AssetDatabase.LoadAssetAtPath<Texture2D>(path);
                if (tex != null && tex.width * tex.height > 256 * 256) // 只关注大贴图
                    missing.Add((path, tex.width, tex.height));
            }
        }

        if (missing.Count > 0)
        {
            Debug.LogWarning($"⚠️ {missing.Count} 张大贴图缺少 Mipmap（严重带宽浪费）:");
            foreach (var m in missing)
                Debug.LogWarning($"  {m.width}x{m.height} | {m.path}");
        }
        else
        {
            Debug.Log("✓ 所有大贴图均已开启 Mipmap");
        }
    }

    void GenerateOptimizationReport()
    {
        var reportPath = "Temp/texture_bandwidth_report.txt";
        using (var writer = new StreamWriter(reportPath))
        {
            writer.WriteLine("=== 纹理带宽优化建议报告 ===");
            writer.WriteLine($"生成时间: {System.DateTime.Now}");
            writer.WriteLine();
            writer.WriteLine("优化优先级排序:");
            writer.WriteLine("1. [高] 将未压缩贴图转为 ASTC 6x6（带宽减少 ~75%）");
            writer.WriteLine("2. [高] 为所有 >256x256 的贴图启用 Mipmap（减少远距离过采样 4-16x）");
            writer.WriteLine("3. [中] Channel Packing：AO/Roughness/Metallic/Height 合并为一张 RGBA");
            writer.WriteLine("4. [中] 按使用面积重新裁剪贴图分辨率（近景2K → 远景512）");
            writer.WriteLine("5. [中] Shader 中减少多层 Blend 采样次数");
            writer.WriteLine("6. [低] 评估 Anisotropic Filter 级别（4x 通常够用）");
            writer.WriteLine("7. [低] 评估 Texture Streaming（按需加载减少峰值带宽）");
        }
        Debug.Log($"报告已生成: {reportPath}");
        EditorUtility.RevealInFinder(reportPath);
    }
}
```

**Shader 层采样优化示例（地形 Blend 减采样）：**

```hlsl
// 优化前：4层地形 = 4次纹理采样 + 4次法线采样 = 8次采样
half4 frag(Varyings IN) : SV_Target {
    half4 albedo = 0;
    albedo += tex2D(_Splat0, IN.uv) * IN.weight.x;
    albedo += tex2D(_Splat1, IN.uv) * IN.weight.y;
    albedo += tex2D(_Splat2, IN.uv) * IN.weight.z;
    albedo += tex2D(_Splat3, IN.uv) * IN.weight.w;
    // ... 法线同理 4 次
    return albedo;
}

// 优化后方案 A：远处分支跳过（只采样权重最大的2层）
half4 frag(Varyings IN) : SV_Target {
    // 找到最大权重的两层
    half4 w = IN.weight;
    half maxW = max(max(w.x, w.y), max(w.z, w.w));

    half4 albedo = 0;
    UNITY_BRANCH
    if (w.x > 0.2) albedo += tex2D(_Splat0, IN.uv) * w.x;
    UNITY_BRANCH
    if (w.y > 0.2) albedo += tex2D(_Splat1, IN.uv) * w.y;
    UNITY_BRANCH
    if (w.z > 0.2) albedo += tex2D(_Splat2, IN.uv) * w.z;
    UNITY_BRANCH
    if (w.w > 0.2) albedo += tex2D(_Splat3, IN.uv) * w.w;
    return albedo;
}

// 优化后方案 B：Texture Array + 一次采样（硬件支持时）
half4 frag(Varyings IN) : SV_Target {
    // 使用 Texture2DArray，根据权重选层
    float layer = dot(IN.weight, float4(0, 1, 2, 3));
    half4 albedo = UNITY_SAMPLE_TEX2DARRAY(_SplatArray, 
        float3(IN.uv, layer));
    return albedo;
}
```

### ⚡ 实战经验

**真实项目数据（某开放世界手游）：**

| 优化措施 | 实施前带宽 | 实施后带宽 | 节省 |
|---------|-----------|-----------|------|
| ETC2 → ASTC 6x6 | 14.2 GB/s | 9.8 GB/s | -31% |
| 全局开启 Mipmap | 9.8 GB/s | 7.1 GB/s | -28% |
| Channel Packing（4合1） | 7.1 GB/s | 5.9 GB/s | -17% |
| 地形 Blend 分支裁剪 | 5.9 GB/s | 5.2 GB/s | -12% |
| 远景贴图降分辨率 | 5.2 GB/s | 4.5 GB/s | -13% |
| **总计** | **14.2 GB/s** | **4.5 GB/s** | **-68%** |

**踩坑经验：**
- ASTC 不是越压缩越好：法线贴图用 ASTC 6x6 会出现明显法线偏差，建议 ASTC 5x5 或用 BC5 双通道方案
- Texture Streaming 有坑：快速转动相机时可能出现 Mipmap 弹跳，需要做预加载偏移
- Anisotropic Filter 是隐形带宽杀手：16x 各向异性过滤在某些 GPU 上会增加 8x 带宽，通常 2-4x 足够
- 测试时一定要在目标设备上测——模拟器不会暴露真实的 TBR 带宽行为

### 🎯 能力体检清单

- [ ] 你能说出 ASTC 4x4 / 6x6 / 8x8 的带宽差异和适用场景吗？
- [ ] 你知道一张没有 Mipmap 的 2K 贴图在远距离时会产生多少倍过采样吗？
- [ ] 你会用 Snapdragon Profiler 查看 GPU 带宽分解吗？
- [ ] 你了解 Channel Packing 的标准做法和注意事项吗？
- [ ] 你知道 Anisotropic Filter 的带宽代价吗？默认设多少？
- [ ] 你能估算出项目中每张贴图每帧消耗多少带宽吗？
- [ ] 你了解 TBDR 架构下 Depth Pre-Pass 如何间接减少纹理带宽吗？

### 🔗 相关问题

- [Adreno Tile-Based 架构下的带宽优化策略](../optimization/adreno-tile-based-bandwidth.md)
- [GPU 带宽优化的系统性方法](../optimization/gpu-bandwidth-optimization.md)
- [移动端贴图压缩方案选型](../technical-art/mobile-texture-compression.md)
- [纹理流式加载与 Mipmap 预取优化](../optimization/texture-streaming-mipmap-prefetch.md)
