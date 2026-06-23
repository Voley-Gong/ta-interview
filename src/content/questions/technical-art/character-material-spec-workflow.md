---
title: "新项目角色材质规范怎么定？面试官给你一个角色原画和5个外包模型"
category: "technical-art"
level: 3
tags: ["材质规范", "PBR", "Substance", "美术流程", "外包管理", "QA"]
hint: "材质规范不是写个文档就完事——从PBR校准到Substance模板锁死，再到自动检测工具，三步闭环"
related: ["technical-art/pbr-material-authoring", "technical-art/shader-template-system", "pipeline/unity-asset-checker-tool", "soft-skills/art-program-translation"]
---

## 参考答案

### 🎬 场景描述

面试官把一张角色原画推到你面前，同时打开一个文件夹说：

> "这是我们新项目的主角。美术总监已经在 Substance Painter 里做好了高模版本，现在要交付给外包做剩下 4 个 NPC。你来制定这个项目的角色材质规范，确保外包交付的模型贴图质量统一、导入引擎后效果一致。你的方案是什么？"

这是米哈游、叠纸、鹰角等二次元/写实角色向项目 TA 岗的**必考场景**。考察的不是你懂多少 PBR 理论，而是你能不能把理论变成**可执行的规范 + 可检测的工具链**。

### ✅ 核心要点

1. **PBR 校准基准**：定义项目统一的灯光环境（ACES/Agtex 色卡），所有材质在此环境下审核
2. **Substance 模板锁定**：给外包的 `.spp` 模板预置输出通道、分辨率、命名规范，锁死流程
3. **贴图规范文档**：通道排列、色彩空间、分辨率分档（主角 4K / NPC 2K / 群众 1K）、压缩格式
4. **自动检测工具**：引擎导入时自动检查贴图分辨率、色彩空间、通道映射是否合规
5. **审核反馈闭环**：TA → 外包的反馈要可量化（色差值、粗糙度范围），不是"感觉不对"

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终目标：5个角色（主角+4个NPC）放一起，材质风格统一、质量达标
          ↓ 倒推
统一标准 = 所有人在同一灯光环境下看起来协调
          ↓ 倒推
协调从哪来 = PBR参数范围一致 + 贴图风格一致 + Shader 一致
          ↓ 倒推
外包怎么遵守 = 给模板、给规范文档、给参考图
          ↓ 倒推
交付后怎么验收 = 自动检测工具 + 标准灯光环境截图比对
          ↓ 倒推
返工怎么减少 = 规范要具体到数值（如：BaseColor sRGB、Roughness [0.15~0.85]）
```

#### 知识点拆解（倒推树）

```
角色材质规范制定
├── PBR 校准体系
│   ├── 色卡/灰球校准（标准灯光环境搭建）
│   ├── BaseColor 数值范围参考表（皮肤/金属/布料各不同）
│   ├── Roughness/Specular 取值规范（避免纯 0 / 纯 1）
│   ├── Metallic 二值化原则（金属 = 1，非金属 = 0，不做中间值）
│   └── ACES Tonemapping 下的色彩管理
├── Substance Painter 模板工程
│   ├── .spp 模板预设（输出集、分辨率、导出命名）
│   ├── Smart Material 库（项目统一的材质预设）
│   ├── Anchor / Mesh Map 锁定（烘焙结果不可改）
│   └── 导出预设（通道映射与引擎对齐）
├── 贴图规范文档
│   ├── 通道排列（URP: R=Metallic, A=Smoothness / SRP 自定义）
│   ├── 色彩空间（BaseColor/Normal=sRGB? Linear? 校准！）
│   ├── 分辨率分档策略（主角/NPC/群众三级）
│   ├── 压缩格式（ASTC 级别选择与质量验证）
│   └── Mipmap 生成与过滤规范
├── 自动检测工具链
│   ├── Unity AssetPostprocessor 导入拦截
│   ├── 贴图属性自动校验脚本（分辨率/格式/色彩空间/通道）
│   ├── 材质参数范围检测（Roughness 直方图分析）
│   └── 标准环境截图自动化（RenderTexture 比对）
└── 外包管理流程
    ├── 交付物清单（.fbx + 贴图集 + .spp 工程 + 截图）
    ├── 验收标准 Checklist（量化指标，不是"感觉"）
    ├── 返工反馈模板（标注问题区域 + 数值偏差）
    └── 版本管理规范（命名 v01/v02/...final）
```

#### 代码实现

**1. Substance Painter 导出预设（JSON 配置）**

```json
{
  "exportName": "Character_URP_Export",
  "presetType": "textures",
  "textureSet": "Character_Main",
  "resolution": "4096",
  "channels": [
    { "name": "BaseColor", "source": "baseColor", "colorSpace": "sRGB" },
    { "name": "Normal", "source": "normal", "colorSpace": "Linear", "format": "OpenGL" },
    { "name": "ORM", "source": "combined", "channels": {
        "R": "ambientOcclusion",
        "G": "roughness",
        "B": "metallic"
    }, "colorSpace": "Linear" }
  ],
  "namingConvention": "{meshName}_{channel}_{resolution}",
  "padding": 16
}
```

**2. Unity 导入自动检测脚本**

```csharp
using UnityEditor;
using UnityEngine;

public class TextureImportChecker : AssetPostprocessor
{
    private static readonly int[] AllowedSizes = { 1024, 2048, 4096 };
    private const string CharacterTexturePath = "Assets/Art/Characters/";

    void OnPreprocessTexture()
    {
        // 只检查角色目录
        if (!assetPath.StartsWith(CharacterTexturePath)) return;

        TextureImporter importer = (TextureImporter)assetImporter;

        // === 1. 检查分辨率 ===
        // 预检查无法获取实际尺寸，放到 OnPostprocessTexture
    }

    static void OnPostprocessAllAssets(
        string[] imported, string[] deleted, string[] moved, string[] movedFrom)
    {
        foreach (string path in imported)
        {
            if (!path.StartsWith(CharacterTexturePath)) continue;
            if (!path.EndsWith(".png") && !path.EndsWith(".tga")) continue;

            Texture2D tex = AssetDatabase.LoadAssetAtPath<Texture2D>(path);
            if (tex == null) continue;

            var issues = new System.Collections.Generic.List<string>();

            // 1. 分辨率检查
            int maxSize = Mathf.Max(tex.width, tex.height);
            if (System.Array.IndexOf(AllowedSizes, maxSize) == -1)
            {
                issues.Add($"[分辨率] {tex.width}x{tex.height} 不在规范内（要求: " +
                           string.Join("/", System.Array.ConvertAll(AllowedSizes, s => s.ToString())) + "）");
            }

            // 2. 文件名规范检查（_BC/_N/_ORM 后缀）
            string filename = System.IO.Path.GetFileNameWithoutExtension(path);
            bool validSuffix = filename.EndsWith("_BC") || filename.EndsWith("_N") ||
                               filename.EndsWith("_ORM");
            if (!validSuffix)
            {
                issues.Add($"[命名] '{filename}' 缺少标准后缀（_BC/_N/_ORM）");
            }

            // 3. 法线贴图标记检查
            if (filename.EndsWith("_N"))
            {
                TextureImporter ti = AssetImporter.GetAtPath(path) as TextureImporter;
                if (ti != null && ti.textureType != TextureImporterType.NormalMap)
                {
                    issues.Add($"[类型] 法线贴图 '{filename}' 未标记为 NormalMap 类型");
                    ti.textureType = TextureImporterType.NormalMap;
                    ti.SaveAndReimport();
                }
            }

            // 4. 色彩空间检查（通过 AssetImporter）
            TextureImporter texImporter = AssetImporter.GetAtPath(path) as TextureImporter;
            if (texImporter != null)
            {
                bool shouldBeSRGB = filename.EndsWith("_BC"); // BaseColor = sRGB
                if (texImporter.sRGBTexture != shouldBeSRGB)
                {
                    issues.Add($"[色彩空间] '{filename}' sRGB={texImporter.sRGBTexture}，" +
                               $"应为 {shouldBeSRGB}");
                    texImporter.sRGBTexture = shouldBeSRGB;
                    texImporter.SaveAndReimport();
                }
            }

            // 5. Roughness 直方图分析（检测反差过大或全灰）
            if (filename.EndsWith("_ORM"))
            {
                AnalyzeRoughnessRange(tex, filename, issues);
            }

            // 输出报告
            if (issues.Count > 0)
            {
                string report = $"🔍 [材质规范检查] {path}\n" +
                                string.Join("\n", issues);
                Debug.LogWarning(report, tex);

                // 可选：写入审核日志文件
                LogToAuditFile(path, issues);
            }
        }
    }

    /// <summary>
    /// 分析 Roughness 通道（G 通道）的数值分布
    /// 正常角色材质 Roughness 应在 [0.08, 0.95] 范围内
    /// </summary>
    static void AnalyzeRoughnessRange(Texture2D tex, string name,
        System.Collections.Generic.List<string> issues)
    {
        RenderTexture rt = RenderTexture.GetTemporary(tex.width, tex.height, 0,
            RenderTextureFormat.ARGB32);
        Graphics.Blit(tex, rt);
        RenderTexture.active = rt;

        // 采样 G 通道（Roughness）
        // 注意：这里是简化版，实际应使用 ComputeShader 做 GPU 直方图
        Color[] pixels = null;
        try
        {
            // 降采样到 256x256 做快速分析
            Texture2D small = new Texture2D(256, 256, TextureFormat.RGBA32, false);
            small.ReadPixels(new Rect(0, 0, 256, 256), 0, 0);
            small.Apply();
            pixels = small.GetPixels();

            float minR = 1f, maxR = 0f, avgR = 0f;
            int belowThreshold = 0, aboveThreshold = 0;

            foreach (var p in pixels)
            {
                // G 通道 = Roughness（ORM 打包格式）
                float r = p.g;
                minR = Mathf.Min(minR, r);
                maxR = Mathf.Max(maxR, r);
                avgR += r;

                if (r < 0.08f) belowThreshold++;
                if (r > 0.95f) aboveThreshold++;
            }
            avgR /= pixels.Length;

            float belowPct = (float)belowThreshold / pixels.Length * 100f;
            float abovePct = (float)aboveThreshold / pixels.Length * 100f;

            // 检查是否有过多的极端值
            if (belowPct > 5f)
                issues.Add($"[Roughness] {name} 有 {belowPct:F1}% 像素 < 0.08，" +
                           $"可能导致高光过亮（金属反射尖刺）");
            if (abovePct > 10f)
                issues.Add($"[Roughness] {name} 有 {abovePct:F1}% 像素 > 0.95，" +
                           $"可能导致大面积无高光（像橡胶）");
            if (maxR - minR < 0.1f)
                issues.Add($"[Roughness] {name} Roughness 动态范围过小 " +
                           $"({minR:F2}~{maxR:F2})，材质缺乏质感变化");

            Object.DestroyImmediate(small);
        }
        finally
        {
            RenderTexture.ReleaseTemporary(rt);
        }
    }

    static void LogToAuditFile(string assetPath, System.Collections.Generic.List<string> issues)
    {
        string logPath = "Assets/Art/Characters/_audit_log.txt";
        string entry = $"[{System.DateTime.Now:yyyy-MM-dd HH:mm}] {assetPath}\n" +
                       string.Join("\n", issues) + "\n---\n";
        System.IO.File.AppendAllText(logPath, entry);
        AssetDatabase.ImportAsset(logPath);
    }
}
```

**3. 标准审核环境搭建（伪代码）**

```csharp
/// <summary>
/// 在专用审核场景中渲染角色截图
/// 场景配置：3点光照（Key/Fill/Rim）+ 中性灰背景 + ACES Tonemapping
/// </summary>
public class MaterialReviewCapture : MonoBehaviour
{
    public Camera reviewCamera;
    public Light keyLight, fillLight, rimLight;
    public string outputFolder = "Screenshots/Review/";

    [Header("标准光照参数（仿 Agtex / Quixel 校准环境）")]
    public float keyIntensity = 3.5f;   // 主光
    public float fillIntensity = 1.2f;  // 补光
    public float rimIntensity = 2.0f;   // 轮廓光
    public Color keyColor = new Color(1f, 0.96f, 0.92f);  // 暖白
    public Color fillColor = new Color(0.85f, 0.9f, 1f);  // 冷白

    public void CaptureCharacter(GameObject character, string characterName)
    {
        // 设置标准灯光
        SetupStandardLighting();

        // 实例化角色到审核位置
        var go = Instantiate(character, transform.position, transform.rotation);

        // 渲染正面 / 侧面 45° / 侧面 90°
        string[] angles = { "front", "side45", "side90" };
        float[] rotations = { 0f, 45f, 90f };

        for (int i = 0; i < 3; i++)
        {
            reviewCamera.transform.rotation =
                Quaternion.Euler(0, rotations[i], 0);
            RenderTexture rt = new RenderTexture(1920, 1080, 24);
            reviewCamera.targetTexture = rt;
            reviewCamera.Render();

            SaveRenderTexture(rt, $"{outputFolder}{characterName}_{angles[i]}.png");
            reviewCamera.targetTexture = null;
            rt.Release();
        }

        DestroyImmediate(go);
    }

    void SetupStandardLighting()
    {
        keyLight.color = keyColor;
        keyLight.intensity = keyIntensity;
        keyLight.transform.rotation = Quaternion.Euler(45, -30, 0);

        fillLight.color = fillColor;
        fillLight.intensity = fillIntensity;
        fillLight.transform.rotation = Quaternion.Euler(30, 60, 0);

        rimLight.color = Color.white;
        rimLight.intensity = rimIntensity;
        rimLight.transform.rotation = Quaternion.Euler(-20, 180, 0);
    }

    void SaveRenderTexture(RenderTexture rt, string path)
    {
        RenderTexture.active = rt;
        Texture2D tex = new Texture2D(rt.width, rt.height, TextureFormat.RGBA32, false);
        tex.ReadPixels(new Rect(0, 0, rt.width, rt.height), 0, 0);
        tex.Apply();
        System.IO.File.WriteAllBytes(path, tex.EncodeToPNG());
        DestroyImmediate(tex);
    }
}
```

### ⚡ 实战经验

**踩坑 1：BaseColor 的 sRGB 陷阱**
外包交付的 BaseColor 贴图经常是 Linear 空间的——在 Substance 里看着没问题（因为 Substance 内部做了转换），导入 Unity 后颜色变灰、饱和度下降。**规范里必须明确：BaseColor = sRGB 纹理，Normal/ORM = Linear 纹理**。在 AssetPostprocessor 里强制设置，不依赖人工。

**踩坑 2：法线贴图的 OpenGL/DirectX 翻转**
Substance Painter 导出默认是 OpenGL 格式（Y 轴朝上），Unity 标准也是 OpenGL。但如果外包用了 Maya（默认 DirectX），法线 G 通道会翻转，在引擎里看到的是凹凸反向的"月球表面"。**规范里要锁定导出格式 = OpenGL，并在导入工具里加 G 通道翻转检测**。

**踩坑 3：Roughness vs Smoothness 的坑**
Substance Painter 里画的是 Roughness（粗糙度），Unity URP 默认的 Lit Shader 用的是 Smoothness（光滑度 = 1 - Roughness）。如果导出的是 ORM（G = Roughness），在 Shader 里直接采样 G 通道当 Smoothness 用，所有材质看起来都像橡胶。**规范要明确 ORM 通道定义 + Shader 端做翻转**。

**踩坑 4：外包材质的"过度细化"**
外包 3D 美术经常在 Substance 里画非常精细的细节（毛孔、纤维），4K 贴图塞满。但在游戏中角色只占屏幕 200 像素高时，这些细节全是浪费——还额外增加带宽和内存。**规范里要设定有效分辨率（角色在屏幕上的像素占比 → 实际需要多少贴图分辨率）**。

### 🎯 能力体检清单

| 检查项 | 如果你答不上来… |
|--------|----------------|
| 能说出 PBR 四张贴图分别是什么、各自什么色彩空间？ | → PBR 基础有盲区 |
| 知道 Substance Painter 导出的 ORM 在 Unity URP 里怎么映射？ | → 工作流有断层 |
| 能解释为什么 Roughness 不应该出现 0 或 1 的极端值？ | → 材质原理不扎实 |
| 会写 AssetPostprocessor 自动检查贴图属性？ | → 工具链能力不足 |
| 能搭建标准灯光审核环境（三点光 + 色卡）？ | → 校准体系缺失 |
| 知道 ASTC 压缩对不同通道的影响（法线 vs 颜色）？ | → 移动端压缩知识盲区 |
| 能制定量化的验收标准（不是"看着差不多"）？ | → 规范制定经验不足 |
| 知道如何管理多外包团队的材质一致性？ | → 项目流程经验不足 |

### 🔗 相关问题

- [PBR 材质从零到一：角色皮肤怎么做出真实质感？](technical-art/pbr-material-authoring)
- [Shader 模板系统设计：让美术不会写错 Shader](technical-art/shader-template-system)
- [Unity 资源检查工具：美术资源批量检测](pipeline/unity-asset-checker-tool)
- [美术与技术沟通：如何让程序理解美术需求](soft-skills/art-program-translation)
