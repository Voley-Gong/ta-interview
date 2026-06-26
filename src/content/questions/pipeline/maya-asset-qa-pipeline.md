---
title: "美术提交的资产全不合格：如何用Python搭建一套Maya自动检查+修复管线？"
category: "pipeline"
level: 4
tags: ["Maya", "Python", "资产检查", "自动化管线", "PyMEL", "QA", "规范校验"]
hint: "不是写几个检查函数——是从美术提交入口到反馈修复，建一条自动化的资产质量流水线"
related: ["pipeline/maya-lod-automation", "pipeline/unity-asset-checker-tool", "pipeline/batch-material-audit-tool", "pipeline/editor-asset-normalizer-tool"]
---

## 参考答案

### 🎯 场景描述

面试官（管线 TA Lead / 工具开发负责人）说：

> "我们项目 40 个美术，每天提交约 200 个 Maya 文件到 Perforce。现在的问题：美术提交的模型各种不合格——
> - 命名不规范（`Mesh_01` / `final_final_v2` / 中文命名）
> - 历史残留没冻结（Freeze Transform 没做）
> - 轴心点乱放（角色轴心在脚底但偏移了 0.3 单位）
> - UV 重叠 / UV 超出 0-1 范围
> - 材质用了 Maya 默认 `lambert1` / `initialShadingGroup`
> - 多边形面数超标（单角色 20 万三角面）
> - Duplicate Shape 节点残留
>
> 你现在要建一套自动化检查系统。要求：
> 1. 美术提交时自动触发检查
> 2. 检查出问题后，能自动修复的就自动修复
> 3. 不能自动修复的，生成报告反馈给美术
> 4. 检查规则要可配置，不同资产类型（角色/场景/道具）规则不同"

这是网易、字节、米哈游等中大型团队管线 TA 岗位的高频面试题。核心考察 **Python 编程能力 + Maya API 理解 + 管线设计思维**。

### ✅ 核心要点

1. **不是写散装检查函数，而是建一套 Rule Engine（规则引擎）**
2. **三层架构**：检查规则层 → 修复动作层 → 报告反馈层
3. **PyMEL vs cmds vs OpenMaya API**：性能依次递增，易用性依次递减
4. **自动修复优先级**：能自动修的（Freeze / Delete History / 重命名）→ 给提示的（面数超标 / UV 问题）
5. **集成到提交流程**：Perforce Pre-Submit Hook → Maya 内置检查面板 → CI/CD 批量检查

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：200个文件/天，自动检查+修复+反馈，规则可配置
     ↓ 倒推
"反馈给美术" = 生成结构化报告（JSON + HTML），附截图标注
     ↓ 倒推
"自动修复" = 定义修复动作（Fix Action），与检查规则一一对应
     ↓ 倒推
"自动触发" = 集成到提交流程（Maya Save Hook / Perforce Trigger）
     ↓ 倒推
"规则可配置" = 检查规则是数据（JSON/YAML），不是硬编码
     ↓ 倒推
架构设计：Rule Engine + Asset Context + Fix Action + Report Generator
```

#### 知识点拆解（倒推树）

```
Maya 资产自动检查管线
├── Python 基础
│   ├── PyMEL API（面向对象的 Maya 节点访问）
│   ├── maya.cmds vs PyMEL vs OpenMaya（性能选型）
│   └── Decorator 模式（规则注册、性能计时）
├── Maya 节点体系
│   ├── Transform 节点（Freeze Transform / 轴心点）
│   ├── Mesh 节点（History / UV / 法线 / 面数）
│   ├── Material 节点（ShadingEngine / 材质引用检查）
│   └── Outliner 结构（命名规范 / 层级规范）
├── 检查规则设计
│   ├── 命名规范检查（正则匹配 / 资产类型前缀）
│   ├── 几何体检查（面数 / 三角面 / 四边面比例 / 退化面）
│   ├── UV 检查（重叠 / 超界 / 重叠岛 / UV 利用率）
│   ├── 变换检查（Freeze Transform / 轴心点 / Scale 值）
│   ├── 材质检查（默认材质 / 缺失贴图 / 未使用材质）
│   └── 场景清理（未引用节点 / 空 Shader / 历史）
├── 修复动作
│   ├── 安全修复（冻结变换 / 删除历史 / 清理空节点）
│   ├── 交互修复（弹出窗口让美术确认）
│   └── 不可修复（面数超标 → 退回给美术）
├── 管线集成
│   ├── Perforce Trigger（提交前自动检查）
│   ├── Maya ScriptNode（保存时触发）
│   ├── PySide2 UI（工具面板嵌入 Maya）
│   └── CI/CD 批量扫描（Jenkins / GitHub Actions）
└── 报告系统
    ├── JSON 结构化数据（程序读取）
    ├── HTML 可视化报告（人工查看）
    └── 截图标注（Maya viewport 截图 + 问题区域高亮）
```

#### 代码实现

**核心：Rule Engine 框架**

```python
# asset_qa/core/rule_engine.py
"""
资产检查规则引擎
- 基于装饰器自动注册检查规则
- 支持按资产类型过滤规则
- 每条规则关联一个修复动作
"""

import abc
import re
import json
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import List, Dict, Optional, Callable
import pymel.core as pm


# ─── 数据结构 ───

class Severity(Enum):
    ERROR = "error"      # 必须修复，阻止提交
    WARNING = "warning"  # 建议修复，不阻止提交
    INFO = "info"        # 仅提示

class AssetType(Enum):
    CHARACTER = "character"
    ENVIRONMENT = "environment"
    PROP = "prop"
    VEHICLE = "vehicle"
    GENERIC = "generic"

@dataclass
class CheckResult:
    """单条检查结果"""
    rule_name: str
    severity: Severity
    passed: bool
    message: str
    node_names: List[str] = field(default_factory=list)
    auto_fixable: bool = False
    metadata: Dict = field(default_factory=dict)

@dataclass
class AssetContext:
    """当前资产的上下文信息"""
    asset_type: AssetType
    file_path: str
    scene_name: str
    all_meshes: List[str] = field(default_factory=list)
    all_materials: List[str] = field(default_factory=list)
    all_transforms: List[str] = field(default_factory=list)
    poly_count: int = 0


# ─── 规则注册器 ───

_REGISTRY: Dict[str, 'Rule'] = {}

def register_rule(
    name: str,
    asset_types: List[AssetType] = None,
    severity: Severity = Severity.ERROR,
    auto_fixable: bool = False
):
    """装饰器：注册一条检查规则"""
    def decorator(cls):
        cls.rule_name = name
        cls.severity = severity
        cls.auto_fixable = auto_fixable
        cls.asset_types = asset_types or list(AssetType)
        _REGISTRY[name] = cls()
        return cls
    return decorator


# ─── 基类 ───

class Rule(abc.ABC):
    """检查规则基类"""
    rule_name: str = ""
    severity: Severity = Severity.ERROR
    auto_fixable: bool = False
    asset_types: List[AssetType] = []
    
    @abc.abstractmethod
    def check(self, ctx: AssetContext) -> CheckResult:
        """执行检查"""
        pass
    
    def fix(self, ctx: AssetContext, result: CheckResult) -> CheckResult:
        """执行修复（如果 auto_fixable=True）"""
        if not self.auto_fixable:
            return result
        raise NotImplementedError(f"{self.rule_name} 需要实现 fix()")


# ─── 具体规则实现 ───

@register_rule(
    "naming_convention",
    severity=Severity.ERROR,
    auto_fixable=True
)
class NamingConventionRule(Rule):
    """命名规范检查：前缀_资产名_序号（如 chr_hero_001）"""
    
    NAMING_PATTERNS = {
        AssetType.CHARACTER: r"^(chr|sk|rig)_([a-z][a-z0-9_]+)$",
        AssetType.ENVIRONMENT: r"^(env|bg|prop)_([a-z][a-z0-9_]+)$",
        AssetType.PROP: r"^(prop|itm)_([a-z][a-z0-9_]+)$",
    }
    
    def check(self, ctx: AssetContext) -> CheckResult:
        pattern = self.NAMING_PATTERNS.get(ctx.asset_type, r"^[a-z][a-z0-9_]+$")
        bad_names = []
        
        for mesh_name in ctx.all_meshes:
            # 检查 transform 节点名（不是 shape 节点）
            if not re.match(pattern, mesh_name):
                bad_names.append(mesh_name)
        
        if bad_names:
            return CheckResult(
                rule_name=self.rule_name,
                severity=self.severity,
                passed=False,
                message=f"{len(bad_names)} 个物体命名不规范",
                node_names=bad_names[:20],  # 最多报告20个
                auto_fixable=self.auto_fixable,
                metadata={"pattern": pattern}
            )
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=True,
            message="命名规范检查通过"
        )
    
    def fix(self, ctx: AssetContext, result: CheckResult) -> CheckResult:
        """自动重命名为合法格式"""
        fixed = []
        for name in result.node_names:
            # 转小写 + 替换非法字符
            new_name = re.sub(r"[^a-z0-9_]", "_", name.lower())
            new_name = f"{ctx.asset_type.value[:3]}_{new_name}"
            try:
                pm.rename(name, new_name)
                fixed.append(f"{name} → {new_name}")
            except Exception as e:
                pass
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=True,
            message=f"已自动修复 {len(fixed)} 个命名",
            metadata={"fixed": fixed}
        )


@register_rule(
    "unfrozen_transform",
    severity=Severity.ERROR,
    auto_fixable=True
)
class FreezeTransformRule(Rule):
    """检查 Transform 是否冻结"""
    
    def check(self, ctx: AssetContext) -> CheckResult:
        unfrozen = []
        for mesh_name in ctx.all_meshes:
            try:
                node = pm.PyNode(mesh_name)
                t = node.translate.get()
                r = node.rotate.get()
                s = node.scale.get()
                # 检查是否有非零变换（scale 非单位也算未冻结）
                if (abs(t[0]) > 0.001 or abs(t[1]) > 0.001 or abs(t[2]) > 0.001 or
                    abs(r[0]) > 0.01 or abs(r[1]) > 0.01 or abs(r[2]) > 0.01 or
                    abs(s[0] - 1) > 0.001 or abs(s[1] - 1) > 0.001 or abs(s[2] - 1) > 0.001):
                    unfrozen.append(mesh_name)
            except:
                pass
        
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=len(unfrozen) == 0,
            message=f"{len(unfrozen)} 个物体未冻结变换" if unfrozen else "变换冻结检查通过",
            node_names=unfrozen,
            auto_fixable=True
        )
    
    def fix(self, ctx: AssetContext, result: CheckResult) -> CheckResult:
        for name in result.node_names:
            pm.select(name, r=True)
            pm.makeIdentity(apply=True, t=1, r=1, s=1, n=0)
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=True,
            message=f"已冻结 {len(result.node_names)} 个物体的变换"
        )


@register_rule(
    "history_cleanup",
    severity=Severity.WARNING,
    auto_fixable=True
)
class DeleteHistoryRule(Rule):
    """检查是否删除构造历史"""
    
    def check(self, ctx: AssetContext) -> CheckResult:
        has_history = []
        for mesh_name in ctx.all_meshes:
            try:
                shape = pm.listRelatives(mesh_name, s=True)[0]
                history = pm.listHistory(shape, groupLevels=True)
                # 过滤掉 mesh 节点本身和 tweak 节点
                construction_history = [h for h in history 
                                        if h.typeName() not in ("mesh", "tweak")]
                if construction_history:
                    has_history.append(mesh_name)
            except:
                pass
        
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=len(has_history) == 0,
            message=f"{len(has_history)} 个物体有构造历史" if has_history else "历史清理检查通过",
            node_names=has_history,
            auto_fixable=True
        )
    
    def fix(self, ctx: AssetContext, result: CheckResult) -> CheckResult:
        for name in result.node_names:
            pm.select(name, r=True)
            pm.delete(ch=True)  # Delete by Type: History
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=True,
            message=f"已删除 {len(result.node_names)} 个物体的构造历史"
        )


@register_rule(
    "default_material_usage",
    severity=Severity.ERROR,
    auto_fixable=False
)
class DefaultMaterialRule(Rule):
    """检查是否使用了 Maya 默认材质"""
    
    DEFAULT_MATERIALS = {"lambert1", "initialShadingGroup"}
    
    def check(self, ctx: AssetContext) -> CheckResult:
        offenders = []
        for mesh_name in ctx.all_meshes:
            try:
                shading_engines = pm.listConnections(mesh_name, type="shadingEngine")
                for se in shading_engines:
                    if se.name() in self.DEFAULT_MATERIALS:
                        offenders.append(mesh_name)
                        break
            except:
                pass
        
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=len(offenders) == 0,
            message=f"{len(offenders)} 个物体使用了默认材质（lambert1）" if offenders 
                   else "材质检查通过",
            node_names=offenders,
            auto_fixable=False  # 材质替换需要美术判断，不自动修
        )


@register_rule(
    "poly_count_budget",
    severity=Severity.ERROR,
    auto_fixable=False
)
class PolyCountRule(Rule):
    """面数预算检查"""
    
    BUDGETS = {
        AssetType.CHARACTER: 80000,      # 三角面
        AssetType.ENVIRONMENT: 50000,    # 单个场景物件
        AssetType.PROP: 15000,
        AssetType.VEHICLE: 60000,
    }
    
    def check(self, ctx: AssetContext) -> CheckResult:
        budget = self.BUDGETS.get(ctx.asset_type, 50000)
        offenders = []
        total_polys = 0
        
        for mesh_name in ctx.all_meshes:
            try:
                shape = pm.listRelatives(mesh_name, s=True)[0]
                # 三角面数（Maya 显示的是三角化后的面数）
                tris = pm.polyEvaluate(shape, triangle=True) or 0
                total_polys += tris
                if tris > budget:
                    offenders.append(f"{mesh_name} ({tris:,} tris)")
            except:
                pass
        
        return CheckResult(
            rule_name=self.rule_name,
            severity=self.severity,
            passed=len(offenders) == 0,
            message=f"{len(offenders)} 个物体超出面数预算（{budget:,} tris）" if offenders
                   else f"面数检查通过（总计 {total_polys:,} tris）",
            node_names=offenders[:10],
            auto_fixable=False,
            metadata={"budget": budget, "total": total_polys}
        )


# ─── 引擎主流程 ───

class AssetQAEngine:
    """资产 QA 引擎主类"""
    
    def __init__(self):
        self.rules = list(_REGISTRY.values())
    
    def run_all_checks(self, ctx: AssetContext, 
                       auto_fix: bool = True) -> List[CheckResult]:
        """执行所有适用的检查规则"""
        results = []
        
        for rule in self.rules:
            if ctx.asset_type not in rule.asset_types:
                continue
            
            start = time.time()
            result = rule.check(ctx)
            result.metadata["check_time_ms"] = (time.time() - start) * 1000
            
            # 自动修复
            if not result.passed and auto_fix and rule.auto_fixable:
                fix_result = rule.fix(ctx, result)
                results.append(fix_result)
            else:
                results.append(result)
        
        return results
    
    def generate_report(self, results: List[CheckResult], 
                        ctx: AssetContext) -> str:
        """生成 JSON 报告"""
        report = {
            "file": ctx.file_path,
            "asset_type": ctx.asset_type.value,
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
            "summary": {
                "total_rules": len(results),
                "passed": sum(1 for r in results if r.passed),
                "failed": sum(1 for r in results if not r.passed),
                "errors": sum(1 for r in results if not r.passed and r.severity == Severity.ERROR),
                "warnings": sum(1 for r in results if not r.passed and r.severity == Severity.WARNING),
            },
            "details": [
                {
                    "rule": r.rule_name,
                    "severity": r.severity.value,
                    "passed": r.passed,
                    "message": r.message,
                    "nodes": r.node_names[:10],
                    "auto_fixable": r.auto_fixable,
                    "time_ms": round(r.metadata.get("check_time_ms", 0), 2),
                }
                for r in results
            ]
        }
        return json.dumps(report, indent=2, ensure_ascii=False)


# ─── 入口函数 ───

def run_qa_check(asset_type: str = "character"):
    """Maya 内调用入口"""
    # 构建上下文
    ctx = AssetContext(
        asset_type=AssetType(asset_type),
        file_path=pm.sceneName(),
        scene_name=pm.sceneName().split("/")[-1],
        all_meshes=[str(n) for n in pm.ls(type="transform") 
                     if pm.listRelatives(n, shapes=True, type="mesh")],
        all_materials=[str(n) for n in pm.ls(type="lambert")],
    )
    
    # 运行检查
    engine = AssetQAEngine()
    results = engine.run_all_checks(ctx, auto_fix=True)
    
    # 输出报告
    report = engine.generate_report(results, ctx)
    report_path = f"/tmp/qa_report_{ctx.scene_name}.json"
    with open(report_path, "w") as f:
        f.write(report)
    
    # 打印摘要
    summary = json.loads(report)["summary"]
    print(f"\n{'='*50}")
    print(f"📊 QA Report: {ctx.scene_name}")
    print(f"✅ Passed: {summary['passed']} | ❌ Failed: {summary['failed']}")
    print(f"🔴 Errors: {summary['errors']} | 🟡 Warnings: {summary['warnings']}")
    print(f"📄 Full report: {report_path}")
    print(f"{'='*50}\n")
    
    return results


# Maya 菜单注册
def build_qa_menu():
    """在 Maya 菜单栏添加 QA 工具入口"""
    menu_name = "AssetQA"
    if pm.menu(menu_name, exists=True):
        pm.deleteUI(menu_name)
    
    pm.menu(menu_name, parent="MayaWindow", label="资产QA")
    pm.menuItem(label="🔍 检查角色资产", command=lambda x: run_qa_check("character"))
    pm.menuItem(label="🔍 检查场景资产", command=lambda x: run_qa_check("environment"))
    pm.menuItem(label="🔍 检查道具资产", command=lambda x: run_qa_check("prop"))
    pm.menuItem(divider=True)
    pm.menuItem(label="⚙️ 打开规则配置", command=lambda x: open_rule_config())
```

**Perforce Trigger 集成（服务端自动触发）：**

```python
#!/usr/bin/env python3
# p4_trigger_qa.py — Perforce pre-submit trigger
"""
部署在 Perforce Server 上，美术提交 .ma/.mb 文件时自动触发检查。
配置方法（在 p4 protect 表中）：
  pre-user-submit //depot/art/.../*.ma  "/path/to/p4_trigger_qa.py %changelist%"
"""

import sys
import subprocess
import json
import tempfile
import maya.standalone
maya.standalone.initialize(name="python")
import pymel.core as pm

def check_changelist(cl_number):
    """检查 Perforce Changelist 中的 Maya 文件"""
    # 获取 changelist 中的文件列表
    files = subprocess.check_output(
        ["p4", "-F", "%depotFile%", "describe", cl_number]
    ).decode().splitlines()
    
    maya_files = [f for f in files if f.endswith((".ma", ".mb"))]
    
    if not maya_files:
        return 0  # 无 Maya 文件，允许提交
    
    all_reports = []
    for depot_file in maya_files:
        # p4 print 到临时文件
        local_path = tempfile.NamedTemporaryFile(
            suffix="." + depot_file.split(".")[-1], delete=False).name
        subprocess.check_call(["p4", "print", "-o", local_path, depot_file])
        
        # 打开 Maya 文件
        pm.openFile(local_path, force=True)
        
        # 推断资产类型（根据路径）
        asset_type = "character"
        if "/env/" in depot_file.lower():
            asset_type = "environment"
        elif "/prop/" in depot_file.lower():
            asset_type = "prop"
        
        # 执行检查（不自动修复，只报告）
        ctx = AssetContext(
            asset_type=AssetType(asset_type),
            file_path=depot_file,
            scene_name=depot_file.split("/")[-1],
            all_meshes=[str(n) for n in pm.ls(type="transform") 
                        if pm.listRelatives(n, shapes=True, type="mesh")],
        )
        
        engine = AssetQAEngine()
        results = engine.run_all_checks(ctx, auto_fix=False)
        report = engine.generate_report(results, ctx)
        all_reports.append(json.loads(report))
    
    # 判断是否有 ERROR 级别的问题
    for r in all_reports:
        if r["summary"]["errors"] > 0:
            print(f"\n❌ 提交被拒绝：{r['file']}")
            print(f"   {r['summary']['errors']} 个错误需要修复")
            for d in r["details"]:
                if not d["passed"] and d["severity"] == "error":
                    print(f"   - {d['rule']}: {d['message']}")
            return 1  # 阻止提交
    
    print("✅ 所有 Maya 资产检查通过，允许提交")
    return 0

if __name__ == "__main__":
    sys.exit(check_changelist(sys.argv[1]))
```

### ⚡ 实战经验

1. **先用规则扫描全量资产，不要一上来就做 Perforce Trigger**：先收集问题清单，和美术对齐规则后再上触发器
2. **自动修复要保守**：冻结变换、删历史这类安全操作可以自动做；但重命名、改材质需要美术确认
3. **性能是关键**：大型场景（500+ 物体）检查时间要控制在 5 秒以内——用 OpenMaya MFn API 替代 PyMEL 可提升 3-5 倍速度
4. **规则配置外置化**：检查规则用 JSON/YAML 配置文件管理，不同项目复用引擎代码，只改规则
5. **报告要可视化**：美术不看 JSON 文件——生成 HTML 报告 + viewport 截图标注问题物体
6. **分阶段推行**：先只报告不阻止 → 美术适应后开启 Warning 阻止 → 最后开启 Error 阻止
7. **Duplicate Shape 检查是大坑**：Maya 中一个 Transform 下可能有多个 Shape（duplicate 后残留），要检查 `pm.listRelatives(shapes=True)` 的返回值

### 🎯 能力体检清单

- [ ] PyMEL 和 cmds 的区别是什么？性能差异有多大？什么场景该用 OpenMaya C++ API？
- [ ] 如何检测一个 Transform 节点是否真正冻结了（不只是看 Translate 是否为零）？
- [ ] UV 重叠检测算法的核心思路是什么？（提示：UV 岛的边界匹配 + 面积异常检测）
- [ ] Perforce Trigger 的 pre-user-submit 能否修改提交内容？如果不能，如何实现"自动修复后再提交"？
- [ ] 如果美术团队反对自动化检查（觉得被监控），你如何说服他们？

### 🔗 相关问题

- [Maya LOD 自动化](pipeline/maya-lod-automation.md)
- [Unity 资产检查工具](pipeline/unity-asset-checker-tool.md)
- [批量材质审计工具](pipeline/batch-material-audit-tool.md)
- [Editor 资产规范化工具](pipeline/editor-asset-normalizer-tool.md)
