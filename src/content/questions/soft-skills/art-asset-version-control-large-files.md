---
title: "美术资源版本管理：大文件 Git 协作方案怎么选？"
category: "soft-skills"
level: 2
tags: ["版本管理", "Git LFS", "Perforce", "美术协作", "SVN", "团队流程"]
hint: "Git 不适合管理二进制大文件——美术团队需要 Perforce（游戏行业标准）或 Git LFS，关键是选对工具并建立提交规范"
related: ["pipeline/unity-asset-checker-tool", "pipeline/art-asset-diff-review-pipeline", "soft-skills/cross-department-conflict"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们团队 30 人——10 个美术、15 个程序、5 个策划。现在美术资源和代码放同一个 Git 仓库，美术抱怨 .psd / .mb / .hip 文件太大 push 不上去，程序抱怨 clone 仓库要 2 小时。你作为 TA 怎么解决这个版本管理问题？给我一个完整方案。」

### ✅ 核心要点

1. **Git 不是为二进制大文件设计的**：Git 的 delta 差异算法对二进制无效，每次修改都是全量存储
2. **游戏行业标准是 Perforce（Helix Core）**：支持二进制文件 Lock + 差异存储 + 大仓库流畅操作
3. **Git LFS 是折中方案**：适合小团队或以代码为主的仓库，但有配额和性能上限
4. **代码和美术应该分离**：代码用 Git，美术资源用 Perforce / SVN / LFS 独立管理
5. **提交规范比工具选择更重要**：无论用什么工具，命名、分类、Review 流程才是核心

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：美术能顺畅提交大文件 → 程序能快速 clone → 版本可追溯 → 无冲突覆盖
                ↑
倒推1：Git 不适合大文件 → 要么换工具，要么用 LFS 扩展
倒推2：30 人团队有并发冲突风险 → 需要 Lock 机制（Check-out）
倒推3：美术不擅长命令行 → 需要 GUI 工具（P4V / Fork / Sourcetree）
倒推4：程序和美术需求不同 → 仓库分离（代码 vs 资源）
倒推5：上线前需要 Review → 美术资源也需要 CI 检查 + Review 流程
```

#### 知识点拆解（倒推树）

```
美术资源版本管理方案
├── 工具选型对比
│   ├── Perforce（Helix Core）—— 行业标准
│   │   ├── 优点：二进制 Lock、大仓库性能好、Chunk 下载、权限细粒度
│   │   ├── 缺点：Server 部署重、许可证费用（免费版 5 用户/20 workspace）
│   │   ├── 适用：3A / 中型团队 / 大量二进制资源
│   │   └── 工作流：Check-Out → 修改 → Submit → 自动集成到引擎
│   ├── Git LFS（Large File Storage）
│   │   ├── 优点：与 Git 无缝集成、分布式、免费
│   │   ├── 缺点：LFS Server 配额限制、clone 慢、Lock 功能弱
│   │   ├── 适用：小团队 / 程序主导 / 二进制文件量适中
│   │   └── 注意：GitHub LFS 配额 1GB free，超出需付费
│   ├── SVN（Subversion）
│   │   ├── 优点：集中式、Lock 支持、对二进制友好
│   │   ├── 缺点：无分布式、分支慢、社区萎缩
│   │   └── 适用：传统团队 / 已有基础设施
│   └── Plastic SCM（Unity）
│       ├── 优点：分布式 + 集中式混合、Lock 支持、与 Unity 集成
│       ├── 缺点：用户少、文档少、生态弱
│       └── 适用：Unity 团队 / 喜欢 Git 风格但需要 Lock
├── 仓库分离策略
│   ├── 方案A：Git（代码） + Perforce（美术） —— 推荐
│   │   ├── Git 管理 Scripts / Shaders / Configs
│   │   ├── P4 管理 Models / Textures / Animations
│   │   └── 交叉引用：P4 中维护一个 manifest 文件映射到 Git
│   ├── 方案B：单 Git 仓库 + LFS
│   │   ├── .gitattributes 配置 LFS 跟踪规则
│   │   ├── 适合 10 人以下小团队
│   │   └── 注意定期清理 LFS 历史（git lfs prune）
│   └── 方案C：单 Perforce 仓库（全部）
│       ├── 大团队 / 3A 项目常见
│       └── 程序也用 P4（虽然有抵触，但 Perforce 的分支也能管代码）
├── 美术提交规范
│   ├── 目录结构
│   │   ├── /Art/Characters/{角色名}/Source/ （DCC 源文件）
│   │   ├── /Art/Characters/{角色名}/Export/ （导出给引擎的文件）
│   │   ├── /Art/Environment/{场景名}/...
│   │   ├── /Art/UI/...
│   │   └── /Art/Shaders/...
│   ├── 命名规范
│   │   ├── 文件名：{类型}_{名称}_{版本}.{后缀}
│   │   ├── 例：CHAR_Heroine_Body_v03.fbx
│   │   └── 禁止中文、空格、特殊字符
│   ├── 提交信息模板
│   │   ├── [WIP] 标记未完成
│   │   ├── [REVIEW] 标记待审核
│   │   ├── [FINAL] 标记最终版
│   │   └── 必须填写变更说明（改了什么、为什么改）
│   └── Lock 规则
│       ├── 同一时刻同一文件只能一人 Check-Out
│       ├── 导出目录（Export/）通常不 Lock（允许覆盖）
│       └── 源文件（Source/）必须 Lock
├── CI/CD 集成
│   ├── 美术资源提交触发
│   │   ├── 自动检查：命名、尺寸、格式、通道
│   │   ├── 自动导入：引擎 FBX 导入 + 材质生成
│   │   └── 自动预览：截图发布到群/看板
│   └── 引擎构建触发
│       └── 美术资源更新 → 增量烘焙 → 打包测试
└── TA 职责
    ├── 制定规范（目录/命名/Lock）
    ├── 搭建工具链（P4/Git 配置 + 钩子脚本）
    ├── 培训美术（GUI 使用 + 流程培训）
    └── 维护 CI 检查（自动验收工具）
```

#### 代码实现

**Git LFS 配置示例（.gitattributes）：**

```gitattributes
# Git LFS 跟踪规则
# 大型二进制文件交给 LFS 管理

# 3D 资源
*.fbx filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
*.mb filter=lfs diff=lfs merge=lfs -text
*.ma filter=lfs diff=lfs merge=lfs -text
*.hip filter=lfs diff=lfs merge=lfs -text

# 贴图
*.psd filter=lfs diff=lfs merge=lfs -text
*.png filter=lfs diff=lfs merge=lfs -text
*.tga filter=lfs diff=lfs merge=lfs -text
*.tif filter=lfs diff=lfs merge=lfs -text
*.exr filter=lfs diff=lfs merge=lfs -text

# 音频/视频
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text

# Unity 特殊文件
*.unity filter=lfs diff=lfs merge=lfs -text
*.asset filter=lfs diff=lfs merge=lfs -text
*.prefab filter=lfs diff=lfs merge=lfs -text
```

**Perforce 提交触发脚本（P4 Trigger）：**

```python
#!/usr/bin/env python3
"""
P4 提交前检查脚本
功能：拦截不合规的美术资源提交
"""
import sys
import re
import subprocess
from pathlib import Path

# 命名规范正则
NAMING_PATTERN = re.compile(
    r'^(?:CHAR|ENV|UI|FX|SHD|SND)_[A-Z][a-zA-Z0-9]+_v\d+\.(?:fbx|psd|tga|png|wav)$',
    re.IGNORECASE
)

# 允许的目录
ALLOWED_DIRS = [
    '//depot/Art/Characters/',
    '//depot/Art/Environment/',
    '//depot/Art/UI/',
    '//depot/Art/FX/',
    '//depot/Art/Shaders/',
]

# 文件大小限制（MB）
MAX_FILE_SIZE_MB = {
    '.psd': 200,
    '.fbx': 50,
    '.tga': 30,
    '.png': 20,
}

def get_changed_files(change_list):
    """获取 changelist 中的文件列表"""
    result = subprocess.run(
        ['p4', 'describe', '-s', change_list],
        capture_output=True, text=True
    )
    files = []
    for line in result.stdout.split('\n'):
        if line.startswith('//'):
            parts = line.split('#')
            if parts:
                files.append(parts[0].strip())
    return files

def check_file(changelist, filepath):
    """检查单个文件是否合规"""
    errors = []

    # 1. 目录检查
    if not any(filepath.startswith(d) for d in ALLOWED_DIRS):
        errors.append(f"❌ 不在允许目录内: {filepath}")

    # 2. 命名检查
    filename = Path(filepath).name
    if not NAMING_PATTERN.match(filename):
        errors.append(
            f"❌ 命名不合规: {filename}\n"
            f"   规范: {{类型}}_{{名称}}_v{{版本}}.{{后缀}}\n"
            f"   例: CHAR_Heroine_Body_v03.fbx"
        )

    # 3. 大小检查
    ext = Path(filepath).suffix.lower()
    if ext in MAX_FILE_SIZE_MB:
        result = subprocess.run(
            ['p4', 'fstat', '-Ol', filepath],
            capture_output=True, text=True
        )
        for line in result.stdout.split('\n'):
            if 'lFileSize' in line:
                size_mb = int(line.split()[-1]) / (1024 * 1024)
                if size_mb > MAX_FILE_SIZE_MB[ext]:
                    errors.append(
                        f"❌ 文件过大: {filename} ({size_mb:.1f}MB > "
                        f"限制 {MAX_FILE_SIZE_MB[ext]}MB)"
                    )

    return errors

def main():
    changelist = sys.argv[1]
    files = get_changed_files(changelist)

    all_errors = []
    for f in files:
        all_errors.extend(check_file(changelist, f))

    if all_errors:
        print("\n".join(all_errors))
        print(f"\n⛔ 提交被拒绝，共 {len(all_errors)} 个问题。请修正后重新提交。")
        sys.exit(1)
    else:
        print(f"✅ 检查通过，{len(files)} 个文件合规。")
        sys.exit(0)

if __name__ == '__main__':
    main()
```

**方案对比表：**

| 维度 | Perforce (Helix Core) | Git LFS | SVN | Plastic SCM |
|------|:---:|:---:|:---:|:---:|
| 二进制大文件 | ★★★★★ | ★★★ | ★★★★ | ★★★★ |
| Lock 机制 | ★★★★★（原生） | ★★★（LFS Lock） | ★★★★ | ★★★★ |
| 分布式 | ❌（集中式） | ✅ | ❌ | ✅（混合） |
| 代码管理 | ★★★（可用但不爽） | ★★★★★ | ★★★ | ★★★ |
| GUI 友好度 | ★★★★★（P4V） | ★★★（Fork/Sourcetree） | ★★★（TortoiseSVN） | ★★★（Gluon） |
| 部署难度 | ★★★（Server 重） | ★★（配 LFS Server） | ★★ | ★★★ |
| 成本 | 免费5人 / 商用付费 | 免费（GitHub LFS 配额内） | 免费 | 免费(CE)/付费 |
| 行业采用率 | 3A/Midcore 最高 | Indie 最高 | 遗留系统 | Unity 生态 |

### ⚡ 实战经验

- **Perforce 的 Lock 是美术的救星**：两个人同时改同一个 .psd 在 Git 里是不可解决的冲突，P4 的 Check-Out Lock 直接从根源杜绝——美术一旦习惯了就回不去
- **Git LFS 的隐藏成本**：LFS 的带宽是按下载量计费的，30 人团队每天 clone / pull 的流量可能很快超出免费配额（GitHub 1GB → 超出 $5/月/5GB）
- **导出文件 vs 源文件分目录**：Source/（.psd / .mb / .hip）需要 Lock，Export/（.fbx / .png）不需要 Lock——因为导出文件是可再生的，Lock 反而降低效率
- **美术培训是成功的关键**：工具再好，美术不会用就是白搭。P4V 的基础操作（Check-Out / Submit / Resolve）需要 1-2 次集中培训 + 写图文手册
- **不要把引擎运行时生成的文件纳入版本管理**：Unity 的 Library/、Unreal 的 Intermediate/、DerivedDataCache/ 等目录必须 gitignore / p4 ignore
- **Changelist Description 模板化**：强制美术填写「改了什么 / 为什么改 / 影响范围」，用 P4 Trigger 拦截空 description 的提交

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道选 Perforce 还是 Git LFS | 版本管理工具对比 | 理解集中式 vs 分布式、Lock 机制差异 |
| 美术不会用命令行工具 | GUI 工具选型 / 培训流程 | 学 P4V / Fork / Gluon 使用和培训方法 |
| 仓库越来越大 clone 慢 | 大文件存储策略 / 浅克隆 | 学 Git LFS / Perforce Proxy / Sparse Checkout |
| 美术经常覆盖别人的修改 | Lock 机制 / Check-Out 流程 | 实施 P4 Check-Out 或 Git LFS Lock |
| CI 中美术资源检查缺失 | P4 Trigger / Git Hook 编写 | 学提交钩子 + 自动化验收工具 |

### 🔗 相关问题

- Perforce 怎么做分支管理？和 Git 分支有什么本质区别？（提示：Stream / Branch Mapping / Inter-Branch Merge）
- 美术外包团队怎么接入版本管理？（提示：Perforce Edge Server / Git 子仓库 / 交付包模式）
- 如何在不停止开发的情况下做仓库迁移？（提示：增量同步 + 冻结期 + 双跑验证）
