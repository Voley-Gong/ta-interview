---
title: "美术资源提交门禁：如何搭建 DCC → 引擎的自动化 CI/CD 审查流水线？"
category: "pipeline"
level: 3
tags: ["CI/CD", "自动化", "资产审查", "Git", "Python", "管线"]
hint: "不是等资源进引擎再检查——在 DCC 导出时就拦截：命名规范、贴图尺寸、LOD 完整性、材质引用，pre-commit 阶段全卡住"
related: ["pipeline/maya-asset-qa-pipeline", "pipeline/unity-asset-checker-tool", "pipeline/art-asset-diff-review-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们团队 30 个美术，每天往 SVN/Git 提交上百个资源文件。目前全靠 TA 人工抽查，漏检率 15%，不合规资源经常跑到构建包里才被发现。你能不能设计一套自动化审查流水线，让资源在提交时就被自动检查？不需要等 CI 构建发现。」

### ✅ 核心要点

1. **左移检查（Shift-Left）**：越早拦截成本越低——DCC 导出时 > Pre-commit > CI 构建 > 打包后
2. **分阶段门禁**：Pre-commit Hook（本地秒级）→ CI Pipeline（服务端分钟级）→ Nightly Full Audit（全量深度）
3. **规则可配置**：不同项目阶段（原型/Alpha/Beta/Release）用不同严格等级
4. **开发者反馈闭环**：检查失败时给出清晰的修复指引，而不是一行 "Reject"
5. **旁路机制**：紧急情况可 bypass，但自动记录 + 事后补审

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：美术提交资源 → 0.5秒内本地反馈「通过/不通过+原因」→ CI 再做深度检查 → 全部绿灯才能合并
                ↑
倒推1：需要「提交时拦截」→ Git pre-commit hook / SVN pre-commit hook
倒推2：需要「快速检查」→ 规则分两级：
      L1 快速（< 2秒）：文件命名、尺寸、格式、空引用
      L2 深度（< 30秒）：UV 重叠、材质参数范围、骨骼层级
倒推3：需要「规则引擎」→ JSON/YAML 配置驱动，不用改代码就能加规则
倒推4：需要「团队可维护」→ 规则模板、项目继承、白名单机制
倒推5：需要「可视化报告」→ HTML 报告 + Jenkins/GitLab CI Dashboard
倒推6：需要「修复指引」→ 每条规则附带修复教程链接
```

#### 知识点拆解（倒推树）

```
美术资源 CI/CD 门禁
├── 本地拦截层（Pre-commit）
│   ├── Git Hook：pre-commit 框架（Python）
│   ├── SVN Hook：pre-commit 脚本（Bash/Python）
│   ├── 检查速度控制：L1 快速规则 < 2s
│   └── 依赖缓存：规则包、参考表本地缓存
├── 规则引擎设计
│   ├── 规则定义格式：YAML schema（name, severity, check_fn, fix_guide）
│   ├── 规则分类：
│   │   ├── 命名规范（naming convention）
│   │   ├── 贴图规格（尺寸、格式、色彩空间）
│   │   ├── 网格检查（三角面数、UV、LOD 完整性）
│   │   ├── 材质检查（引用完整性、参数范围）
│   │   └── 目录结构（分类归属）
│   ├── 严格等级：prototype / alpha / beta / release
│   └── 自定义规则接口：Python 插件式注册
├── CI 集成层
│   ├── GitLab CI / Jenkins / GitHub Actions
│   ├── Pipeline Stage：lint → deep_audit → report → gate
│   ├── 并行加速：大规模提交分片检查
│   └── 失败策略：fail-fast vs collect-all-errors
├── 报告与可视化
│   ├── HTML 报告（Jinja2 模板）
│   ├── IDE/DCC 插件面板（内嵌检查结果）
│   ├── Dashboard 统计（通过率、高频错误 Top 10）
│   └── 飞书/钉钉/企业微信通知机器人
├── 旁路与例外
│   ├── Bypass 机制：[SKIP-LINT] 标记 + 审批流程
│   ├── 白名单：特定文件/目录免检
│   └── 紧急热修：快速通道 + 事后补审工单
└── 基础设施
    ├── 规则版本管理：与项目仓库同步
    ├── 统一 CLI 工具：`art-lint check <path> --level=beta`
    └── Docker 镜像：CI 环境一致性
```

#### 代码实现

**规则定义（YAML 配置驱动）：**

```yaml
# rules/textures.yaml
- name: "texture_max_size"
  severity: error
  level: [alpha, beta, release]
  check:
    type: texture_dimension
    max: 2048
    exclude_power_of_two: false
  fix_guide: |
    贴图尺寸超过 {actual}x{actual}，最大允许 {max}x{max}。
    请在 Substance Painter 中修改 Output Size，或使用脚本批量缩放。
    教程链接：https://wiki.internal/texture-resize

- name: "texture_format_mobile"
  severity: error
  level: [beta, release]
  check:
    type: texture_format
    allowed: ["ASTC_6x6", "ASTC_4x4", "ETC2"]
  fix_guide: |
    贴图格式 {actual} 不在移动端允许列表中。
    请在 Unity Texture Importer 中设置 Format 为 ASTC_6x6。

- name: "naming_convention"
  severity: warn
  level: [alpha, beta, release]
  check:
    type: regex_name
    pattern: "^[A-Z][a-z]+_[A-Z]+_\\d{2}$"
    examples: ["Hero_DIFF_01", "Weapon_NRM_02"]
  fix_guide: |
    文件名 '{actual}' 不符合规范。
    格式：类别_类型_编号，如 Hero_DIFF_01
```

**Pre-commit Hook（Python 核心）：**

```python
#!/usr/bin/env python3
# hooks/pre_commit_art_lint.py
"""Git pre-commit hook: 美术资源快速门禁检查"""

import sys
import time
import yaml
from pathlib import Path
from collections import defaultdict

class ArtLint:
    def __init__(self, project_root: str):
        self.root = Path(project_root)
        self.rules = self._load_rules()
        self.errors = []
        self.warnings = []

    def _load_rules(self):
        """加载规则配置"""
        rules_path = self.root / "tools" / "art-lint" / "rules"
        all_rules = []
        for yml in rules_path.glob("*.yaml"):
            with open(yml) as f:
                all_rules.extend(yaml.safe_load(f))
        return all_rules

    def get_staged_files(self):
        """获取 Git 暂存区中需要检查的文件"""
        import subprocess
        result = subprocess.run(
            ["git", "diff", "--cached", "--name-only", "--diff-filter=ACM"],
            capture_output=True, text=True, cwd=self.root
        )
        # 只检查美术资源
        art_exts = {".fbx", ".obj", ".png", ".tga", ".jpg", ".exr",
                    ".tif", ".material", ".mat", ".prefab"}
        files = [self.root / f for f in result.stdout.strip().split("\n")
                 if Path(f).suffix.lower() in art_exts]
        return files

    def check_fast(self, files):
        """L1 快速检查（< 2秒）"""
        start = time.time()
        for f in files:
            for rule in self.rules:
                if rule["severity"] != "error":
                    continue  # L1 只跑 error 级
                if not self._is_fast_check(rule):
                    continue  # 跳过深度检查

                result = self._apply_rule(f, rule)
                if result is False:
                    self.errors.append({
                        "file": str(f.relative_to(self.root)),
                        "rule": rule["name"],
                        "guide": rule.get("fix_guide", "").format(
                            actual=result.actual if hasattr(result, 'actual') else 'N/A'
                        )
                    })

        elapsed = time.time() - start
        print(f"[art-lint] L1 快速检查完成，耗时 {elapsed:.2f}s，"
              f"检查 {len(files)} 个文件")
        return len(self.errors) == 0

    def _is_fast_check(self, rule):
        """判断是否为快速检查规则（不涉及文件内容解析）"""
        fast_types = {"regex_name", "file_extension", "file_size"}
        return rule["check"].get("type") in fast_types

    def report(self):
        """输出检查报告"""
        if self.errors:
            print("\n❌ 以下资源未通过门禁检查：")
            for e in self.errors:
                print(f"  {e['file']}")
                print(f"    规则: {e['rule']}")
                print(f"  修复: {e['guide'][:80]}...")
                print()
            print(f"共 {len(self.errors)} 个错误。请修复后重新提交。")
            print("（紧急情况可使用 git commit --no-verify 跳过，"
                  "但需在 JIRA 创建补审工单）")
            return False

        print("✅ 全部资源通过门禁检查。")
        return True


if __name__ == "__main__":
    lint = ArtLint(project_root=".")
    files = lint.get_staged_files()

    if not files:
        sys.exit(0)  # 没有美术文件变更，直接通过

    lint.check_fast(files)
    sys.exit(0 if lint.report() else 1)
```

**GitLab CI Pipeline 配置：**

```yaml
# .gitlab-ci.yml
stages:
  - art-lint
  - art-deep-audit
  - report

art_lint_fast:
  stage: art-lint
  image: registry.internal/art-lint:latest
  rules:
    - changes:
        - "**/*.fbx"
        - "**/*.png"
        - "**/*.tga"
        - "**/*.mat"
  script:
    - art-lint check --level=beta --report=lint_report.json
  artifacts:
    reports:
      junit: lint_report.json
    paths:
      - lint_report.html
    expire_in: 7 days
  allow_failure: false  # 门禁：检查失败则阻止合并

art_deep_audit:
  stage: art-deep-audit
  image: registry.internal/art-lint:latest
  script:
    - art-lint deep-check --level=beta --jobs=4
      --checks=uv_overlap,triangle_budget,material_refs,skeleton_hierarchy
      --report=deep_report.html
  artifacts:
    paths:
      - deep_report.html
    expire_in: 14 days
  allow_failure: true  # 深度检查仅警告，不阻塞

notify_dashboard:
  stage: report
  script:
    - python3 tools/send_report.py --webhook=$FEISHU_WEBHOOK
      --report=lint_report.json --project=$CI_PROJECT_NAME
  only:
    - main
    - develop
```

### ⚡ 实战经验

1. **规则要"零代码"维护**：TA 写 YAML 配置就行，不用改 Python 代码。这决定了规则能否快速推广到新项目
2. **分等级是灵魂**：原型阶段只查命名和格式；Beta 加面数和 UV；Release 加色彩空间和压缩格式。一刀切会被美术抵制
3. **修复指引比报错更重要**：美术看到 `texture_format_error` 不知道怎么办——必须附带操作步骤和截图教程链接
4. **Bypass 要有审计链**：`--no-verify` 跳过可以，但必须自动创建一张 JIRA 补审工单，Assignee 是提交者，Reviewer 是 TA Lead
5. **真实数据驱动规则**：先跑 2 周"仅报告不拦截"模式，收集 Top 10 高频错误，再针对性地设为 error 级
6. **DCC 插件 > Git Hook**：如果有条件，在 Maya/Blender 导出插件里就做检查——美术在 DCC 里看到红线比在终端看到报错友好得多

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能解释 Git pre-commit hook 的执行时机和限制 | → Git 基础盲区：理解 Git hooks 生命周期 |
| 能设计一个可扩展的规则引擎架构（插件式） | → 软件工程盲区：策略模式 / 插件架构 |
| 知道如何平衡检查严格度与团队接受度 | → 工程管理盲区：渐进式规范推行 |
| 能在 GitLab CI / Jenkins 中编写多阶段 Pipeline | → DevOps 盲区：CI/CD Pipeline 设计 |
| 理解为什么 L1 快速检查不能用 DCC API | → 性能意识盲区：进程启动开销 / IO 瓶颈 |

### 🔗 相关问题

- [pipeline/maya-asset-qa-pipeline](../pipeline/maya-asset-qa-pipeline.md) — Maya 资产 QA 自动化检查
- [pipeline/unity-asset-checker-tool](../pipeline/unity-asset-checker-tool.md) — Unity 引擎侧资源检查工具
- [pipeline/art-asset-diff-review-pipeline](../pipeline/art-asset-diff-review-pipeline.md) — 美术资源 Diff 审查流程
