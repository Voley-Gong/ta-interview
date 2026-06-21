---
title: "项目从零到上线，你怎么制定 LOD 规范并建立质量验收流程？"
category: "technical-art"
level: 3
tags: ["LOD", "美术规范", "质量验收", "自动化检查", "性能预算", "工具管线", "团队协作"]
hint: "核心考点：不是会设 LOD 就行——要制定团队规范、建立验收标准、做自动化检查、并持续维护整个流程"
related: ["technical-art/mobile-texture-compression", "pipeline/maya-lod-automation"]
---

## 参考答案

### 🎬 场景描述

> 面试官：你刚加入一个新项目，团队有 20 个美术，之前做 LOD 全凭感觉，没有统一规范。结果上线前性能 Profile 发现场景忽卡忽流畅——有的建筑 LOD0 五万面，有的只有八千；有的角色做了 4 级 LOD，有的只做了 2 级。
>
> 现在交给你：
> 1. 制定一份完整的 LOD 规范文档
> 2. 建立美术提交时的自动化检查流程
> 3. 设计质量验收标准
>
> 你会怎么做？

### ✅ 核心要点

1. **按资产类型分级制定规范**：角色 / 环境 / 植被 / 道具各有不同的 LOD 策略和距离区间
2. **面数预算倒推法**：从目标帧率 → 单帧三角形预算 → 按资产权重分配
3. **LOD 切换距离的科学设定**：基于屏幕占比（Screen-Relative Size）而非绝对距离
4. **自动化验收工具链**：Maya/Blender 脚本 + CI 检查 + 引擎内 Profile Report
5. **视觉回归测试**：LOD 切换处的 Pop 效果检测与可控衰减

### 📖 深度展开

#### 解决思路（从目标倒推规范）

```
目标：60fps 稳定 → 单帧三角形预算 ≤ 500K（中端手机）
  ↓
分配策略
  ├── 角色（同屏 10 个）：每个 LOD0 8K → 80K
  ├── 环境（建筑+地形）：300K
  ├── 植被：80K
  └── 特效/UI：40K
  ↓
每类资产的 LOD 级数和切换点
  ├── 角色：3 级 LOD（8K → 3K → 800 → BillBoard）
  ├── 建筑：3 级 LOD（15K → 5K → 1.5K）
  ├── 植被：2 级 LOD + Imposter（2K → 600 → Billboard）
  └── 道具：2 级 LOD（1K → 300）
  ↓
自动化验收
  ├── Maya 脚本：提交前检查顶点数 / UV / 法线
  ├── 引擎 CI：导入后自动跑 LOD 面数报告
  └── 截图对比：每个 LOD 切换点自动截图做 A/B 对比
```

#### 知识点拆解（倒推树）

```
LOD 规范制定与验收
├── 面数预算体系
│   ├── 从帧率目标倒推
│   │   ├── 60fps → 16.6ms/帧 → GPU 预算 ~10ms
│   │   ├── 目标平台三角填充率参考
│   │   └── 减去光照/后处理/特效开销 → 可用三角形预算
│   ├── 按资产权重分配
│   │   ├── 关键资产（主角/BOSS）：高预算
│   │   ├── 次要资产（NPC/普通建筑）：中预算
│   │   └── 背景资产（远景/小道具）：低预算
│   └── 动态调整机制
│       ├── 按场景复杂度分档（主城/野外/副本）
│       └── 性能等级适配（高/中/低端机）
│
├── LOD 切换距离策略
│   ├── 绝对距离的问题
│   │   ├── 不同分辨率屏幕下视觉差异大
│   │   └── 1080p 和 4K 下同一距离的像素覆盖不同
│   ├── Screen-Relative Size（推荐）
│   │   ├── 基于 mesh 在屏幕上的像素占比
│   │   ├── Unity: LOD Group 的 Screen-Relative Transition
│   │   ├── UE: Screen Size 阈值
│   │   └── 所有分辨率/FOV 下行为一致
│   └── 距离 × Screen Size 矩阵示例
│
├── LOD 制作质量标准
│   ├── 几何保真度
│   │   ├── 轮廓线一致性（Silhouette 误差 < 阈值）
│   │   ├── UV 保持（避免 LOD 切换时贴图跳动）
│   │   ├── 法线方向一致性（避免光照突变）
│   │   └── 蒙皮权重兼容（角色 LOD 要适配骨骼）
│   ├── 纹理配套
│   │   ├── LOD0: 2K → LOD1: 1K → LOD2: 512
│   │   └── Mipmap 链要与 LOD 级别对齐
│   └── 特殊属性保持
│       ├── 顶点色（如果有）
│       └── 自定义顶点数据（如 Wind 顶点色）
│
├── 自动化检查工具链
│   ├── DCC 端（Maya/Blender）
│   │   ├── 检查脚本：顶点数/面数/UV/法线
│   │   ├── 自动生成报告 + 不合格标红
│   │   └── 阻断式：不合格不允许导出
│   ├── 引擎端
│   │   ├── Import 后自动检查 LOD 数量/面数
│   │   ├── 生成 LOD Summary Report
│   │   └── 与预算对比，超出标红
│   └── CI/CD 集成
│       ├── 每次 Art 提交触发检查
│       ├── 性能回归：自动跑空场景 Profile
│       └── 发送报告到美术群/看板
│
└── 视觉验收流程
    ├── LOD Pop 效果检测
    │   ├── 自动录制 LOD 切换前后帧
    │   ├── 像素差异计算 → 量化 Pop 程度
    │   └── 阈值内 OK，超出人工复查
    ├── 同屏多 LOD 检查
    │   └── 确保同类型资产在同一距离档使用相同 LOD 级
    └── 极端视角检查（远近快速切换/俯视/仰视）
```

#### 代码实现

**LOD 规范文档核心结构（示例）：**

```markdown
# LOD 规范 v2.0

## 面数预算（中端机型基准）

| 资产类型 | LOD0 | LOD1 | LOD2 | Billboard | 切换阈值(Screen%) |
|----------|------|------|------|-----------|------------------|
| 主角     | 12K  | 5K   | 1.5K | -         | 100% / 30% / 10% |
| NPC      | 8K   | 3K   | 800  | ✅        | 80% / 25% / 8%  |
| 主要建筑 | 20K  | 8K   | 2K   | -         | 60% / 20% / 6%  |
| 次要建筑 | 10K  | 3K   | 800  | ✅        | 50% / 15% / 5%  |
| 植被(树) | 3K   | 1K   | 300  | ✅        | 40% / 12% / 4%  |
| 道具     | 1.5K | 400  | -    | -         | 30% / 8%        |

## 质量要求
- LOD 间轮廓误差 < 2%（Symmetry/Hausdorff 距离）
- UV 必须保持，不允许重新展开
- 法线方向偏差 < 15°
- 顶点色/自定义数据必须保留
```

**Maya 自动检查脚本（Python）：**

```python
import maya.cmds as cmds
import json

LOD_SPEC = {
    "character_main": {
        "LOD0": {"max_verts": 12000, "max_tris": 20000},
        "LOD1": {"max_verts": 5000,  "max_tris": 9000},
        "LOD2": {"max_verts": 1500,  "max_tris": 2500},
    },
    "character_npc": {
        "LOD0": {"max_verts": 8000,  "max_tris": 14000},
        "LOD1": {"max_verts": 3000,  "max_tris": 5500},
        "LOD2": {"max_verts": 800,   "max_tris": 1400},
    }
}

def check_lod_compliance(asset_name, asset_type):
    """检查当前 Maya 场景中的 LOD 组是否合规"""
    results = []
    spec = LOD_SPEC.get(asset_type)
    if not spec:
        return [{"status": "ERROR", "msg": f"未知资产类型: {asset_type}"}]
    
    for lod_name, limits in spec.items():
        # 查找场景中的 LOD 组
        lod_objs = cmds.ls(f"*{lod_name}*", transforms=True)
        if not lod_objs:
            results.append({
                "lod": lod_name,
                "status": "MISSING",
                "msg": f"{lod_name} 未找到"
            })
            continue
        
        mesh = cmds.listRelatives(lod_objs[0], shapes=True)
        if not mesh:
            continue
        
        verts = cmds.polyEvaluate(mesh[0], vertex=True)
        tris = cmds.polyEvaluate(mesh[0], triangle=True)
        
        issues = []
        if verts > limits["max_verts"]:
            issues.append(f"顶点超限: {verts} > {limits['max_verts']}")
        if tris > limits["max_tris"]:
            issues.append(f"三角形超限: {tris} > {limits['max_tris']}")
        
        # UV 检查
        uvs = cmds.polyEvaluate(mesh[0], uvcoord=True)
        if uvs == 0:
            issues.append("缺少 UV")
        
        # 法线检查
        normals = cmds.polyNormalPerVertex(mesh[0], query=True)
        if not normals:
            issues.append("缺少法线数据")
        
        results.append({
            "lod": lod_name,
            "verts": verts,
            "tris": tris,
            "status": "PASS" if not issues else "FAIL",
            "issues": issues
        })
    
    return results

# 执行检查并输出报告
report = check_lod_compliance("hero_knight", "character_main")
print(json.dumps(report, indent=2, ensure_ascii=False))

# 导出为 JSON 供 CI 使用
with open("lod_check_report.json", "w") as f:
    json.dump({"asset": "hero_knight", "results": report}, f, ensure_ascii=False)
```

**Unity 引擎内 LOD 面数报告生成器：**

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using System.Text;
using System.IO;
using System.Collections.Generic;

public class LODReportGenerator : EditorWindow
{
    [MenuItem("Tools/TA/LOD 质量报告")]
    static void GenerateReport()
    {
        var report = new StringBuilder();
        report.AppendLine("# LOD 质量报告");
        report.AppendLine($"生成时间: {System.DateTime.Now}");
        report.AppendLine();
        
        var allPrefabs = AssetDatabase.FindAssets("t:Prefab");
        var summary = new Dictionary<string, int>();
        
        foreach (var guid in allPrefabs)
        {
            var path = AssetDatabase.GUIDToAssetPath(guid);
            var go = AssetDatabase.LoadAssetAtPath<GameObject>(path);
            var lodGroups = go.GetComponentsInChildren<LODGroup>();
            
            foreach (var lodGroup in lodGroups)
            {
                var lods = lodGroup.GetLODs();
                report.AppendLine($"## {go.name}");
                report.AppendLine($"| LOD级 | 面数 | 切换阈值 |");
                report.AppendLine($"|-------|------|----------|");
                
                for (int i = 0; i < lods.Length; i++)
                {
                    int totalTris = 0;
                    foreach (var renderer in lods[i].renderers)
                    {
                        if (renderer is MeshRenderer mr)
                        {
                            var mf = mr.GetComponent<MeshFilter>();
                            if (mf && mf.sharedMesh)
                                totalTris += mf.sharedMesh.triangles.Length / 3;
                        }
                    }
                    report.AppendLine($"| LOD{i} | {totalTris:N0} | {lods[i].screenRelativeTransitionHeight * 100:F0}% |");
                }
                report.AppendLine();
            }
        }
        
        var outputPath = "Reports/lod_report.md";
        Directory.CreateDirectory("Reports");
        File.WriteAllText(outputPath, report.ToString());
        Debug.Log($"LOD 报告已生成: {Path.GetFullPath(outputPath)}");
    }
}
#endif
```

### ⚡ 实战经验

- **规范不是一次定死的**：第一版规范一定有不合理的地方。建议先出 v0.5 跑两周，收集美术反馈和性能数据，再出 v1.0。之后每大版本迭代复检一次
- **Screen-Relative 比 Distance 好太多**：以前用距离切换 LOD，不同手机分辨率表现差异巨大。切到 Screen Size 后，高端机和低端机的 LOD 切换视觉表现一致了，省了大量调参时间
- **自动化检查是唯一可持续的方案**：20 个美术靠人肉检查不可能持续。我们在 Maya 出口处加了 Python 检查脚本，不合规直接阻断导出并报错。一周内所有美术就养成了习惯
- **LOD Pop 要量化不要靠肉眼**：用自动截图 + 像素差异计算来评估 LOD 切换的 Pop 效果。设阈值（如差异 < 5%），超出自动标记需要美术复查。这比 QA 团队肉眼找效率高十倍

### 🎯 能力体检清单

| 卡住的环节 | 盲区诊断 | 学习建议 |
|------------|----------|----------|
| 不知道怎么定面数预算 | 缺乏从性能目标倒推的思维 | 学习 GPU Profiling，理解三角形吞吐量与帧预算的关系 |
| LOD 切换距离靠拍脑袋 | 不了解 Screen-Relative Size | 研究 Unity LODGroup / UE Screen Size 的文档 |
| 不知道怎么做自动化检查 | DCC 脚本和 CI 经验不足 | 从简单 Maya Python 脚本开始，逐步加到 CI pipeline |
| 规范推不下去 | 缺少跨部门协作经验 | 参考规范推行的案例，理解美术工作流和痛点 |
| LOD Pop 问题无法量化 | 视觉测试方法论缺失 | 学习 Image Diff 技术，理解像素级质量评估 |

### 🔗 相关问题

- [移动端贴图压缩方案选型](technical-art/mobile-texture-compression) — 贴图也有对应的 Mipmap/压缩规范体系
- [Maya 脚本自动化 LOD](pipeline/maya-lod-automation) — 自动生成 LOD 的工具端配合
- 如何设计 Imposter/ Billboard 替代系统作为 LOD 的终极形态？
