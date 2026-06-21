---
title: "PBR 材质制作规范：如何让美术出图和引擎实时渲染效果一致？"
category: "technical-art"
level: 2
tags: ["PBR", "材质规范", "Substance Painter", "色彩空间", " metallic-roughness", "资产管线"]
hint: "核心是色彩空间一致性（sRGB vs Linear）+ 材质属性正确性（金属度/粗糙度范围）+ SP 到引擎的校准流程"
related: ["technical-art/shader-template-system", "technical-art/mobile-texture-compression", "rendering/sss-skin-rendering"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们项目组发现一个问题——美术在 Substance Painter 里调出来的材质效果很好，但导入 Unity/UE 后看起来完全不一样：金属不像金属、皮革的高光范围不对、颜色整体发灰。美术说"我调对了呀"，程序说"引擎渲染没问题"。你作为 TA 来定位这个问题并制定材质规范，让 SP 到引擎的效果一致。

这是几乎所有项目初期都会遇到的 TA 经典问题。腾讯、网易、米哈游的 TA 面试中高频出现，考察的是 PBR 理论理解 + 色彩空间管理 + 工具链校准 + 跨部门规范制定能力。

### ✅ 核心要点

1. **色彩空间是第一杀手**：Albedo/Base Color 必须在 sRGB 空间，Normal/Roughness/Metallic/AO 必须在 Linear 空间——搞反了就是"效果不对"的根源
2. **PBR 属性值范围校验**：Metallic 只有 0 或 1（非金属 ~0.04），粗糙度 0.2-0.9 之间，Albedo 非金属 30-240 sRGB——超出范围的值会导致物理不正确
3. **SP → 引擎校准链**：相同的 HDR Environment + 同样的 Tone Mapping → SP 和引擎用同一套 IBL，效果才能一致
4. **材质验证工具**：写一个编辑器脚本自动检查贴图的色彩空间标记、值域范围、分辨率匹配
5. **跨部门规范文档**：TA 的核心价值——把"为什么"变成"怎么做"，制定可执行的规范文档

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
现象：SP 中效果好 → 引擎中效果差
                ↑
可能原因排查：
  ├── 色彩空间标记错误（最常见！）
  │   ├── Albedo 被标记为 Linear（偏暗）
  │   └── Normal/Roughness 被标记为 sRGB（偏色/高光错误）
  ├── 引擎 IBL 和 SP 不同
  │   ├── SP 用了 HDRI 预览 → 引擎没用同一套
  │   └── Tone Mapping 不一致
  ├── 材质属性值越界
  │   ├── Metallic 在 0.3-0.7 之间（应该是 0 或 1）
  │   └── Albedo 值过低（非金属不应低于 30 sRGB）
  ├── 贴图压缩损失
  │   ├── BC7 vs ASTC vs DXT5 质量/精度差异
  │   └── Normal Map 的 BC5 vs BC7 选择
  └── Shader 模型不同
      ├── SP 用标准 PBR → 引擎用了简化版
      └── BRDF 模型不一致（GGX vs Blinn-Phong）

解决方案链路：
  Step 1：统一色彩空间规范（sRGB vs Linear 标记表）
  Step 2：统一 IBL 环境（SP 导入引擎的 HDRI）
  Step 3：统一 Tone Mapping（ACES / Filmic / Neutral）
  Step 4：建立材质属性校验工具
  Step 5：输出规范文档 + 培训美术
```

#### 知识点拆解（倒推树）

```
PBR 材质一致性
├── 色彩空间管理
│   ├── sRGB 空间（伽马 2.2）
│   │   ├── 含义：为人眼感知优化的非线性编码
│   │   ├── 适用贴图：Albedo / Base Color（颜色数据）
│   │   └── 引擎处理：采样时自动 sRGB → Linear 转换
│   ├── Linear 空间
│   │   ├── 含义：物理线性的光强编码
│   │   ├── 适用贴图：Normal / Roughness / Metallic / AO / Height（数据贴图）
│   │   └── 引擎处理：直接使用，不做转换
│   └── 常见错误
│       ├── Normal 标为 sRGB → 法线偏色、高光位置偏移
│       ├── Roughness 标为 sRGB → 粗糙度值非线性偏移、高光范围失真
│       └── Albedo 标为 Linear → 颜色偏暗、饱和度异常
│
├── PBR 属性正确性
│   ├── Metallic（金属度）
│   │   ├── 物理含义：0 = 电介质（非金属），1 = 金属
│   │   ├── 值域规范：只有 0 或 1，过渡区仅用于边缘抗锯齿
│   │   └── 常见错误：用 Metallic 0.5 表示"半金属"（物理不存在）
│   ├── Roughness（粗糙度）
│   │   ├── 物理含义：微观表面凹凸程度 → 控制高光散射范围
│   │   ├── 值域规范：0.02-0.1（镜面）到 0.8-1.0（漫散射）
│   │   └── 常见错误：全局粗糙度过低 → 所有物体都油光发亮
│   ├── Albedo（反照率）
│   │   ├── 非金属值域：30-240 sRGB（物理合理范围）
│   │   ├── 金属值域：金属的 Albedo 就是反射率（F0），远高于非金属
│   │   └── 常见错误：Albedo 过暗（<30）→ 像涂了黑漆
│   └── F0（菲涅尔反射率）
│       ├── 非金属 F0：2-5%（大多数材质 ~4%）
│       ├── 金属 F0：70-100%（金 100%、铁 70%、钛 55%）
│       └── Unity/UE 自动从 Metallic + Albedo 推导 F0
│
├── IBL 环境一致性
│   ├── SP 的预览环境
│   │   └── 使用 HDRI Environment Map
│   ├── 引擎的环境光
│   │   ├── Image-Based Lighting（IBL）
│   │   ├── CubeMap / SH（球谐光照）
│   │   └── 反射探针（Reflection Probe）
│   └── 校准流程
│       ├── SP 和引擎导入同一套 HDRI
│       ├── SP 中导出截图 → 引擎中做 A/B 对比
│       └── 调整引擎 IBL 强度匹配 SP 预览
│
├── Tone Mapping（色调映射）
│   ├── 为什么需要：HDR 颜色 → LDR 显示器
│   ├── 常见方案
│   │   ├── ACES Filmic（影视级，对比度高）
│   │   ├── Neutral（中性，适合 UI 截图比对）
│   │   └── Filmic (UE 默认)
│   └── 一致性要求：SP 和引擎使用相同的 Tone Mapping 曲线
│
└── 贴图导入设置（Unity/UE）
    ├── Unity 贴图导入
    │   ├── sRGB 标记（Per-Texture）
    │   ├── Compression：BC7（桌面）/ ASTC（移动）
    │   ├── Normal Map 类型（Unity 自动处理 swizzle）
    │   └── Max Size / Mip Map
    └── UE 贴图导入
        ├── sRGB 复选框
        ├── Compression Settings：TC Vector Displacementmap（Normal）
        └── Texture Group（LOD/平台分配）
```

#### 代码实现

**Unity 编辑器 — PBR 材质校验工具：**

```csharp
using UnityEditor;
using UnityEngine;

public class PBRMaterialValidator : EditorWindow {
    [MenuItem("TA/PBR Material Validator")]
    static void ShowWindow() => GetWindow<PBRMaterialValidator>("PBR Validator");
    
    void OnGUI() {
        if (GUILayout.Button("扫描选中文件夹的所有材质")) {
            string[] guids = AssetDatabase.FindAssets("t:Material", new[] { "Assets/Game/Models" });
            int issues = 0;
            
            foreach (string guid in guids) {
                string path = AssetDatabase.GUIDToAssetPath(guid);
                Material mat = AssetDatabase.LoadAssetAtPath<Material>(path);
                if (mat == null) continue;
                
                issues += ValidateMaterial(mat, path);
            }
            
            Debug.Log($"扫描完成，发现 {issues} 个问题");
        }
        
        if (GUILayout.Button("检查选中材质的贴图色彩空间")) {
            foreach (var obj in Selection.GetFiltered<Material>(SelectionMode.Assets)) {
                ValidateMaterial((Material)obj, AssetDatabase.GetAssetPath(obj));
            }
        }
    }
    
    int ValidateMaterial(Material mat, string path) {
        int issues = 0;
        
        // 检查贴图的 sRGB 设置
        Texture albedo = mat.HasProperty("_BaseMap") ? mat.GetTexture("_BaseMap") : null;
        Texture normal = mat.HasProperty("_BumpMap") ? mat.GetTexture("_BumpMap") : null;
        Texture metallic = mat.HasProperty("_MetallicGlossMap") ? mat.GetTexture("_MetallicGlossMap") : null;
        
        if (albedo != null) {
            string texPath = AssetDatabase.GetAssetPath(albedo);
            var importer = AssetImporter.GetAtPath(texPath) as TextureImporter;
            if (importer != null && !importer.sRGBTexture) {
                Debug.LogWarning($"⚠️ {path}: Albedo 贴图 {albedo.name} 应标记为 sRGB！", mat);
                issues++;
            }
        }
        
        if (normal != null) {
            string texPath = AssetDatabase.GetAssetPath(normal);
            var importer = AssetImporter.GetAtPath(texPath) as TextureImporter;
            if (importer != null && importer.sRGBTexture) {
                Debug.LogError($"❌ {path}: Normal 贴图 {normal.name} 不应标记为 sRGB！", mat);
                issues++;
            }
        }
        
        if (metallic != null) {
            string texPath = AssetDatabase.GetAssetPath(metallic);
            var importer = AssetImporter.GetAtPath(texPath) as TextureImporter;
            if (importer != null && importer.sRGBTexture) {
                Debug.LogError($"❌ {path}: Metallic/Roughness 贴图不应标记为 sRGB！", mat);
                issues++;
            }
        }
        
        // 检查 Metallic 值域
        if (mat.HasProperty("_Metallic")) {
            float metallicVal = mat.GetFloat("_Metallic");
            if (metallicVal > 0.05f && metallicVal < 0.95f) {
                Debug.LogWarning($"⚠️ {path}: Metallic 值为 {metallicVal:F2}，理想值应为 0 或 1", mat);
                issues++;
            }
        }
        
        // 检查贴图分辨率一致性
        int refSize = albedo != null ? albedo.width : 0;
        Texture[] allTextures = { albedo, normal, metallic };
        foreach (var tex in allTextures) {
            if (tex != null && tex.width != refSize && refSize > 0) {
                Debug.LogWarning($"⚠️ {path}: 贴图 {tex.name} 分辨率 ({tex.width}) 与 Albedo ({refSize}) 不一致", mat);
                issues++;
            }
        }
        
        return issues;
    }
}
```

**Substance Painter 导出预设配置（Export Preset）：**

```json
{
  "name": "Unity URP PBR (Linear)",
  "maps": [
    {
      "name": "BaseColor",
      "fileName": "{meshName}_BaseMap",
      "colorSpace": "sRGB",
      "channels": "RGB",
      "source": "baseColor"
    },
    {
      "name": "Normal",
      "fileName": "{meshName}_Normal",
      "colorSpace": "Linear",
      "channels": "RGB",
      "source": "normalGL",
      "note": "OpenGL 绿线朝上（Unity 标准）"
    },
    {
      "name": "Metallic",
      "fileName": "{meshName}_Metallic",
      "colorSpace": "Linear",
      "channels": "R",
      "source": "metallic"
    },
    {
      "name": "Roughness",
      "fileName": "{meshName}_Roughness",
      "colorSpace": "Linear",
      "channels": "R",
      "source": "roughness"
    },
    {
      "name": "AO",
      "fileName": "{meshName}_AO",
      "colorSpace": "Linear",
      "channels": "R",
      "source": "ambientOcclusion"
    }
  ],
  "packing": {
    "mode": "separate",
    "note": "移动端可考虑 ORM 打包（O=AO, R=Roughness, M=Metallic）到 RGB 通道"
  }
}
```

**PBR 贴图色彩空间标记规范表：**

| 贴图类型 | 色彩空间 | Unity sRGB | UE sRGB | 说明 |
|----------|----------|------------|---------|------|
| Base Color / Albedo | sRGB | ✅ | ✅ | 颜色数据，为人眼感知优化 |
| Normal | Linear | ❌ | ❌ | 方向向量数据，必须线性 |
| Roughness | Linear | ❌ | ❌ | 数值数据，不能做伽马校正 |
| Metallic | Linear | ❌ | ❌ | 二值数据，不能做伽马校正 |
| Ambient Occlusion | Linear | ❌ | ❌ | 光照遮蔽数据 |
| Height | Linear | ❌ | ❌ | 位移量数据 |
| Emission | sRGB | ✅ | ✅ | 颜色数据（发光颜色） |
| Alpha | Linear | ❌ | ❌ | 不透明度数值 |

### ⚡ 实战经验

1. **90% 的"SP 到引擎效果不一致"是色彩空间标记错误**。新项目第一周就要做一次全量检查——美术拖入贴图时 Unity/UE 的自动猜测经常猜错（尤其是 Metallic 和 Roughness 被标为 sRGB）
2. **Normal Map 的 OpenGL vs DirectX 问题**：SP 默认导出 OpenGL（绿线朝上），Unity 用 OpenGL，UE 用 DirectX（绿线朝下）。UE 项目要在 SP 导出预设中翻转 G 通道，或引擎端勾选 "Flip Green Channel"
3. **移动端 ORM 打包**：把 AO（R）、Roughness（G）、Metallic（B）打包到一张贴图里，从 5 张贴图减到 2 张（BaseColor + ORM + Normal = 3 张），大幅降低内存和带宽。但美术在 SP 里看不到打包效果，需要提供预览 Shader
4. **建立"标准测试间"**：在引擎中搭建一个固定的测试场景——中性灰背景、校准过的 IBL、固定的 Tone Mapping。所有新材质先放到测试间截图对比 SP 输出，一致后才算通过

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不清楚 sRGB 和 Linear 的区别 | 色彩空间理论、伽马校正 | 学色彩管理基础：为什么需要 sRGB？Linear 空间做光照计算的原因 |
| 不知道 F0 是什么 | PBR 理论基础（菲涅尔反射） | 学 Disney Principled BRDF / Cook-Torrance BRDF |
| 不会写材质校验工具 | Unity Editor Scripting | 学 EditorWindow / AssetPostprocessor / 自定义 Inspector |
| 不知道怎么和美术沟通规范 | 跨部门协作、文档编写 | 学写 TA 规范文档（含"为什么"和"怎么做"两部分） |
| 分不清 Unity 和 UE 的 Normal 方向 | OpenGL vs DirectX Normal 约定 | 理解切线空间、法线的坐标系约定差异 |

### 🔗 相关问题

- 移动端 PBR 贴图如何做压缩方案？（提示：ASTC + Channel Packing + 分辨率分级策略）
- 如何在 Shader 中验证 PBR 参数的物理正确性？（提示：在 Shader 中做值域断言 + Debug View）
- Substance Designer 的程序化材质和 SP 手绘材质在项目中的分工是什么？
