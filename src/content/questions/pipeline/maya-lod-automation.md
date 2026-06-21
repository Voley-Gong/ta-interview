---
title: "用Maya Python脚本自动化LOD生成与检查管线"
category: "pipeline"
level: 2
tags: ["Maya Python", "LOD自动化", "工具管线开发", "PyMEL", "资产规范"]
hint: "TA的核心能力之一是把重复性美术规范变成自动化工具——你的LOD生成还是手动减面吗？"
related: ["pipeline/houdini-vegetation-scatter", "technical-art/shader-template-system"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们项目有上千个静态网格体资产，美术团队手动做LOD效率太低而且质量参差不齐。现在需要一个Maya工具脚本，能自动为选中的模型生成3级LOD，并按照项目规范检查多边形数、UV完整性、命名规则，最后输出报告。你会怎么设计这个工具？

### ✅ 核心要点

1. **工具设计哲学**：TA写工具不是只写脚本，要考虑用户（美术）的使用体验——一键操作、可视化反馈、可配置参数
2. **LOD生成策略**：了解polyReduce算法的局限，知道何时用自动减面、何时需要拓扑保留（如角色脸部）
3. **规范检查自动化**：多边形数、UV、命名、材质数、骨骼绑定——所有人工检查项都应脚本化
4. **批量处理与日志**：上千个资产不能逐个手动跑，需要支持批量处理、进度反馈、错误日志
5. **可维护性**：TA写的工具会被其他TA维护，代码结构、配置分离、文档缺一不可

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
美术需求：选中一批模型 → 一键生成符合规范的LOD
  ↓ 倒推1：规范是什么？
项目LOD规范文档（多边形预算、命名规则、距离阈值）
  ↓ 倒推2：如何自动减面？
Maya polyReduce / 第三方算法（Simplygon / InstaLOD）
  ↓ 倒推3：减面后如何保证质量？
检查UV完整性、检查法线翻转、检查纹理拉伸
  ↓ 倒推4：如何组织工具结构？
UI（参数配置）+ 核心逻辑（生成+检查）+ 输出（报告+日志）
```

#### 知识点拆解（倒推树）

```
LOD自动化管线工具
├── Maya Python API
│   ├── PyMEL vs cmds vs OpenMaya API2.0
│   ├── 选择操作与遍历（ls、select、iterate）
│   ├── 网格操作（polyReduce、polyTriangulate、polyCleanup）
│   └── 节点与属性操作（DG节点图理解）
│
├── LOD生成策略
│   ├── 多边形减面算法：边折叠（Edge Collapse）、体素化
│   ├── 保留区域（Vertex Map / Painted Weight）
│   ├── 对称减面（避免破坏对称模型）
│   ├── UV边界保留（防止UV接缝处撕裂）
│   └── 多级LOD比例：LOD0→LOD1(50%)→LOD2(25%)→LOD3(10%)
│
├── 规范检查系统
│   ├── 多边形数检查（对比预算表）
│   ├── UV完整性（无重叠UV、无超出0-1空间）
│   ├── 命名规范（asset_LOD0_geo / asset_LOD1_geo）
│   ├── 材质数检查（移动端限制材质数量）
│   └── 法线检查（无翻转面、无零面积面）
│
├── 工具工程化
│   ├── UI设计（PyQt/PySide2）
│   │   ├── 参数面板（LOD级数、减面比例、保留区域）
│   │   ├── 进度条与日志窗口
│   │   └── 批量处理队列
│   ├── 配置管理（JSON/CSV存储规范参数）
│   ├── 日志系统（成功/警告/错误分级）
│   └── 版本控制与团队分发
│
└── 进阶：与引擎对接
    ├── 导出FBX时设置LOD Group
    ├── Unreal Engine LOD Import Settings
    └── Unity LOD Group 自动配置
```

#### 代码实现

**核心LOD生成脚本：**

```python
import pymel.core as pm
import json
import os
import datetime

class LODGenerator:
    """LOD自动生成工具 - 按项目规范生成多级LOD并检查质量"""
    
    def __init__(self, config_path=None):
        # 从配置文件加载规范
        if config_path and os.path.exists(config_path):
            with open(config_path, 'r') as f:
                self.config = json.load(f)
        else:
            # 默认配置
            self.config = {
                "lod_ratios": [1.0, 0.5, 0.25, 0.1],      # LOD0-3减面比例
                "screen_sizes": [0.0, 0.3, 0.1, 0.03],     # UE屏幕占比阈值
                "max_materials": 3,                          # 移动端材质数限制
                "naming_pattern": "{asset}_LOD{level}_geo", # 命名规范
                "preserve_uv_seams": True,                   # 保留UV接缝
                "symmetry_check": True,                      # 对称减面
            }
    
    def generate_lod_chain(self, mesh_node):
        """为单个网格体生成完整LOD链"""
        results = {"asset": mesh_node.name(), "levels": []}
        
        original_name = mesh_node.name().split("_LOD")[0].split(":")[-1]
        original_tris = mesh_node.numTriangles() if hasattr(mesh_node, 'numTriangles') else self._count_tris(mesh_node)
        
        for level, (ratio, screen_size) in enumerate(
            zip(self.config["lod_ratios"], self.config["screen_sizes"])
        ):
            if level == 0:
                # LOD0 = 原始模型，只检查不改
                lod_mesh = mesh_node
                target_name = self.config["naming_pattern"].format(
                    asset=original_name, level=0
                )
            else:
                # 复制原始模型作为LOD基础
                lod_mesh = pm.duplicate(mesh_node, name=f"{original_name}_LOD{level}_temp")[0]
                target_tris = int(original_tris * ratio)
                
                # 使用 polyReduce 减面
                pm.polyReduce(
                    lod_mesh,
                    percentage=(1 - ratio) * 100,
                    version=1,
                    keepBorder=self.config["preserve_uv_seams"],  # 保留UV边界
                    keepMapBorder=self.config["preserve_uv_seams"],
                    keepHardEdge=True,
                    keepColorBorder=True,
                    triangulate=True,
                    preserveQuad=True,
                )
                
                # 重命名
                target_name = self.config["naming_pattern"].format(
                    asset=original_name, level=level
                )
                lod_mesh.rename(target_name)
            
            # 质量检查
            check_result = self._run_checks(lod_mesh, level, target_tris if level > 0 else original_tris)
            results["levels"].append(check_result)
        
        return results
    
    def _run_checks(self, mesh, lod_level, expected_tris):
        """执行规范检查"""
        checks = {"name": mesh.name(), "passed": True, "warnings": [], "errors": []}
        
        actual_tris = self._count_tris(mesh)
        
        # Check 1: 三角面数在预算范围内
        checks["tri_count"] = actual_tris
        if lod_level > 0:
            deviation = abs(actual_tris - expected_tris) / max(expected_tris, 1)
            if deviation > 0.15:  # 允许15%偏差
                checks["warnings"].append(
                    f"三角面数偏差 {deviation:.0%}：目标{expected_tris}，实际{actual_tris}"
                )
        
        # Check 2: UV完整性
        uv_sets = mesh.getUVSetNames()
        if not uv_sets or mesh.numUVs() == 0:
            checks["errors"].append("缺少UV数据")
            checks["passed"] = False
        
        # Check 3: 材质数量
        shading_engines = pm.listConnections(mesh, type="shadingEngine")
        if len(shading_engines) > self.config["max_materials"]:
            checks["warnings"].append(
                f"材质数 {len(shading_engines)} 超过限制 {self.config['max_materials']}"
            )
        
        # Check 4: 法线检查（零面积面）
        try:
            faces = mesh.faces
            for face in faces:
                normal = face.getNormal()
                if normal.length() < 0.001:
                    checks["errors"].append(f"发现零面积面: {face}")
                    checks["passed"] = False
                    break
        except:
            checks["warnings"].append("法线检查跳过（OpenMaya异常）")
        
        # Check 5: 命名规范
        import re
        pattern = self.config["naming_pattern"].replace("{asset}", r"[\w]+").replace("{level}", r"\d+") + "$"
        if not re.match(pattern, mesh.name()):
            checks["warnings"].append(f"命名不符合规范: {mesh.name()}")
        
        if checks["errors"]:
            checks["passed"] = False
        return checks
    
    def _count_tris(self, mesh):
        """计算三角面数"""
        try:
            cmds = pm.PyNode(mesh)
            return len(cmds.faces)  # 简化：假设已三角化
        except:
            return pm.polyEvaluate(mesh, triangle=True)
    
    def batch_process(self, selection_only=True):
        """批量处理"""
        if selection_only:
            meshes = [n for n in pm.selected() if n.nodeType() == "mesh"]
            if not meshes:
                meshes = [
                    h.getShape() for h in pm.selected(type="transform")
                    if h.getShape() and h.getShape().nodeType() == "mesh"
                ]
        else:
            meshes = [n for n in pm.ls(type="mesh") if n.name() != "perspShape"]
        
        all_results = []
        for i, mesh in enumerate(meshes):
            pm.progressWindow(
                title="LOD生成中", progress=int((i/len(meshes))*100),
                status=f"处理 {mesh.name()} ({i+1}/{len(meshes)})"
            )
            try:
                result = self.generate_lod_chain(mesh)
                all_results.append(result)
            except Exception as e:
                all_results.append({"asset": mesh.name(), "error": str(e)})
        
        pm.progressWindow(endProgress=True)
        
        # 输出报告
        self._export_report(all_results)
        return all_results
    
    def _export_report(self, results, output_dir=None):
        """导出JSON + CSV报告"""
        if output_dir is None:
            output_dir = os.path.join(pm.workspace(q=True, rootDirectory=True), "lod_reports")
        os.makedirs(output_dir, exist_ok=True)
        
        timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        
        # JSON详细报告
        json_path = os.path.join(output_dir, f"lod_report_{timestamp}.json")
        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(results, f, indent=2, ensure_ascii=False)
        
        # CSV摘要
        csv_path = os.path.join(output_dir, f"lod_summary_{timestamp}.csv")
        with open(csv_path, 'w', encoding='utf-8') as f:
            f.write("Asset,LOD Level,Tri Count,Passed,Warnings,Errors\n")
            for r in results:
                if "error" in r:
                    f.write(f"{r['asset']},N/A,N/A,FAIL,N/A,{r['error']}\n")
                    continue
                for lv in r["levels"]:
                    w = len(lv.get("warnings", []))
                    e = len(lv.get("errors", []))
                    status = "PASS" if lv["passed"] else "FAIL"
                    f.write(f"{r['asset']},{lv['name']},{lv.get('tri_count','N/A')},{status},{w},{e}\n")
        
        print(f"✅ 报告已导出:\n  {json_path}\n  {csv_path}")


# 使用示例
if __name__ == "__main__":
    gen = LODGenerator()
    results = gen.batch_process(selection_only=True)
    
    # 打印摘要
    total = len(results)
    passed = sum(1 for r in results if "error" not in r)
    print(f"\n{'='*50}")
    print(f"LOD生成完成: {passed}/{total} 成功")
```

### ⚡ 实战经验

1. **polyReduce不是万能的**：对于角色脸部、机械硬表面等拓扑敏感的模型，自动减面会破坏形变和质量。实战中需要支持"减面+手动修正"的混合流程——先用脚本减到80%，关键区域留给美术手调
2. **美术工具的UX决定使用率**：工具再强大，如果操作流程超过3步、没有可视化反馈，美术就不会用。一定要做UI面板+进度条+一键操作。曾见过功能完美的脚本因为"要敲命令行"而无人使用
3. **配置与代码分离**：减面比例、命名规范、材质限制这些参数会随项目阶段变化。用JSON配置文件而不是硬编码，TA换项目时只改配置不改代码
4. **批量处理要做事务性**：处理1000个资产时，第500个崩溃了怎么办？每个资产处理后保存检查点，支持断点续做。这个教训是用一个通宵的血泪换来的

### 🎯 能力体检清单

| 卡点 | 说明 | 学习建议 |
|---|---|---|
| 不会Maya Python API | 你不知道如何在Maya中编程操作网格 | 学习PyMEL文档 + OpenMaya API2.0基础 |
| 不理解减面算法原理 | 你只会调用polyReduce但不理解参数背后的意义 | 研读Quadric Error Metrics（边折叠减面经典算法） |
| 工具没有UI | 你的工具是纯命令行，美术不会用 | 学习PySide2/PyQt在Maya中的集成 |
| 没有考虑批量容错 | 你的脚本处理到坏资产就崩溃 | 学习Python异常处理 + 事务性设计模式 |
| 不懂引擎LOD设置 | 你生成了LOD但不知道引擎端怎么配置 | 学习UE LOD Group / Unity LOD Group Import Settings |

### 🔗 相关问题

- [Houdini批量生成植被怎么做？](../pipeline/houdini-vegetation-scatter.md) —— 另一类程序化管线工具
- [Shader模板系统如何设计？](../technical-art/shader-template-system.md) —— TA工具化思维的另一个方向
- 如何与TA Leader协作制定美术资产规范？
