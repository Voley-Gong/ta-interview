---
title: "美术仓库 200GB+ 大文件导致 Git 崩溃：如何设计大文件版本控制迁移方案？"
category: "pipeline"
level: 3
tags: ["Git LFS", "版本控制", "美术管线", "仓库迁移", "Perforce", "Unity"]
hint: "核心不是选什么工具，而是分清哪些资产需要版本控制、哪些用对象存储——混合架构才是正解"
related: ["pipeline/unity-asset-checker-tool", "soft-skills/art-asset-version-control-large-files", "pipeline/art-asset-diff-review-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的美术团队 30 人，项目运行两年了，Git 仓库已经膨胀到 200GB+，clone 一次要 4 小时，新人入职第一天都在等 clone。Git LFS 配了但没配对，有些 .png .fbx 没走 LFS 直接进了 Git 历史，现在仓库已经无法挽回地膨胀了。Switch 分支要 10 分钟，IDE 经常卡死。给你 3 个月时间，设计一套迁移方案解决这个问题。」

补充约束：
- 美术人员不熟悉命令行，需要 GUI 操作
- 程序的代码历史必须保留
- 迁移期间不能停工超过 1 天
- 部分外包人员需要只读访问部分资产

### ✅ 核心要点

1. **问题诊断**：Git 不适合大二进制文件——LFS 是补丁不是银弹，200GB 说明管理已经失控
2. **混合架构是正解**：代码用 Git（轻量快速），大二进制资产用专用资产管理系统（Perforce /对象存储 + 数据库）
3. **历史清理是必须的**：即使开了 LFS，之前进了 Git 主体的二进制文件仍然在历史中——需要 `git filter-repo` 彻底清除
4. **迁移分阶段**：先止血（LFS 规范化），再外科手术（历史清理），最后架构升级（混合方案）
5. **外包权限设计**：细粒度的资产访问控制，不暴露全仓库

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：200GB 仓库 → 代码 < 500MB + 大文件走 LFS/Perforce + clone < 5 分钟
                          ↑
倒推1：为什么仓库这么大？→ 历史中混入了二进制大文件（.fbx .png .psb .wav）
                          ↑
倒推2：为什么混入了？→ LFS tracking 规则不全/添加太晚，部分文件已经 push 到历史
                          ↑
倒推3：怎么清理历史？→ git filter-repo 重写历史，剥离大文件 → 强制推送（破坏性操作）
                          ↑
倒推4：怎么防止再次膨胀？→ Pre-commit Hook 拦截大文件 + LFS 规则全覆盖 + .gitattributes 严格配置
                          ↑
倒推5：200GB 真的都该用 Git LFS 管理吗？→ 不！最终成品资产可以走 Perforce / S3 对象存储
                          ↑
倒推6：外包怎么访问？→ 分仓策略：代码仓（GitLab 权限）+ 资产仓（Perforce Streams 或 S3 预签名链接）
```

#### 知识点拆解（倒推树）

```
大文件仓库迁移
├── Phase 1：止血（1-2 周）
│   ├── 诊断仓库膨胀来源
│   │   ├── git filter-repo --analyze（分析历史中最大的文件）
│   │   ├── 统计各类型文件体积占比
│   │   └── 找出未走 LFS 的二进制文件
│   ├── LFS 规则完善
│   │   ├── .gitattributes 补全所有二进制类型
│   │   ├── 常见需要 LFS 的类型清单
│   │   │   ├── 模型：.fbx .obj .blend .ma .mb
│   │   │   ├── 贴图：.psd .psb .png(>1MB) .tga .exr
│   │   │   ├── 音频：.wav .mp3 .ogg
│   │   │   ├── 视频：.mp4 .mov .webm
│   │   │   └── Unity：.unity .asset .prefab（看情况）
│   │   └── git lfs migrate import（把已有文件迁入 LFS）
│   └── Pre-commit Hook
│       ├── 拦截 > 10MB 非 LFS 文件
│       ├── 拦截不在白名单的文件类型
│       └── 自动提示加入 LFS tracking
│
├── Phase 2：历史清理（2-3 周，需停工半天）
│   ├── git filter-repo（推荐，替代旧的 BFG）
│   │   ├── --strip-blobs-bigger-than 10M（移除历史中 >10M 的文件）
│   │   ├── --path-glob '*.psd' --invert-paths（移除特定类型）
│   │   └── 注意：重写历史后所有人必须重新 clone
│   ├── LFS 历史迁移
│   │   ├── git lfs migrate import --include="*.fbx" --everything
│   │   └── 这会把历史中的 .fbx 移到 LFS 存储
│   ├── GC 压缩
│   │   ├── git reflog expire --expire=now --all
│   │   ├── git gc --prune=now --aggressive
│   │   └── 确认 .git/objects 体积显著下降
│   ├── 远程仓库重建
│   │   ├── 推荐创建全新 bare 仓库（避免旧历史的引用残留）
│   │   ├── 推送清理后的代码
│   │   └── 通知所有人重新 clone
│   └── LFS 存储迁移
│       ├── 如果用 GitLab/GitHub LFS，确认 LFS 对象已迁移
│       └── 如果自建 LFS Server（如 lfs-server），需同步迁移
│
├── Phase 3：架构升级（1-2 月）
│   ├── 方案 A：Git + Git LFS（小团队推荐）
│   │   ├── LFS Server 自建 vs 托管（GitLab/GitHub）
│   │   ├── LFS 配额和带宽管理
│   │   ├── Partial Clone（--filter=blob:none）减少初始下载
│   │   └── Sparse Checkout 只拉取需要的目录
│   ├── 方案 B：Git（代码）+ Perforce（资产）（中大型团队推荐）
│   │   ├── Perforce Helix Core 处理大文件
│   │   ├── Streams 分支管理（类似 Git 分支但更适合二进制）
│   │   ├── Unity 的 Perforce 集成（Version Control 设置）
│   │   └── 代码仓和资产仓通过 Unity Smart Merge 关联
│   ├── 方案 C：Git + 对象存储（S3/OSS）（远程外包团队）
│   │   ├── 大文件上传到 S3/OSS，Git 只存元数据（hash + URL）
│   │   ├── 自定义 Unity Asset Importer 从对象存储拉取
│   │   ├── 版本信息存 Git（asset-manifest.json）
│   │   └── 外包人员通过预签名链接下载
│   └── 方案选型决策表
│       ├── 团队规模 < 10：Git LFS 足够
│       ├── 10~50：Git LFS + Partial Clone 或 Perforce
│       └── > 50 或多项目共享资产：Perforce Streams
│
├── 外包协作设计
│   ├── Perforce 方案
│   │   ├── Protections 权限（只读 / 部分目录可写）
│   │   ├── Streams 隔离（外包专用 Stream）
│   │   └── Shelf 机制（外包提交审查）
│   ├── Git 方案
│   │   ├── 分仓：code-repo（可 clone）+ asset-repo（LFS 按需拉取）
│   │   ├── GitLab Subgroup 权限
│   │   └── Sparse Checkout 让外包只拉特定目录
│   └── 安全审计
│       ├── 水印 / 元数据剥离（防止资产泄露溯源）
│       └── 访问日志（谁下载了什么）
│
└── 美术工作流优化
    ├── Unity Asset Pipeline
    │   ├── AssetImporter 版本锁定（避免导入设置冲突）
    │   ├── .meta 文件冲突处理策略
    │   └── Preset 系统统一导入设置
    ├── 分支策略
    │   ├── 美术不直接 push main，走 Merge Request
    │   ├── MR 自动检查（文件大小、命名规范、分辨率限制）
    │   └── Art Director 审核后合并
    └── 日常操作培训
        ├── SourceTree / Fork GUI 使用
        ├── LFS quota 查看和清理
        └── 常见冲突处理（.meta 文件冲突最常见）
```

#### 代码实现 / 配置示例

**`.gitattributes` 完整配置（Unity 项目）：**

```gitattributes
# === Git LFS Tracking ===

# 3D 模型
*.fbx filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
*.ma filter=lfs diff=lfs merge=lfs -text
*.mb filter=lfs diff=lfs merge=lfs -text
*.max filter=lfs diff=lfs merge=lfs -text

# 贴图（源文件）
*.psd filter=lfs diff=lfs merge=lfs -text
*.psb filter=lfs diff=lfs merge=lfs -text
*.tga filter=lfs diff=lfs merge=lfs -text
*.exr filter=lfs diff=lfs merge=lfs -text
*.tif filter=lfs diff=lfs merge=lfs -text
*.tiff filter=lfs diff=lfs merge=lfs -text

# 大尺寸 PNG/JPG 也走 LFS
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text

# 音频
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text
*.flac filter=lfs diff=lfs merge=lfs -text

# 视频
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text
*.webm filter=lfs diff=lfs merge=lfs -text
*.avi filter=lfs diff=lfs merge=lfs -text

# Unity 场景和预设（按需，看文件大小）
*.unity filter=lfs diff=lfs merge=lfs -text
*.prefab filter=lfs diff=lfs merge=lfs -text
*.asset filter=lfs diff=lfs merge=lfs -text

# HDR/材质
*.hdr filter=lfs diff=lfs merge=lfs -text
*.cubemap filter=lfs diff=lfs merge=lfs -text

# 其他
*.zip filter=lfs diff=lfs merge=lfs -text
*.rar filter=lfs diff=lfs merge=lfs -text
*.7z filter=lfs diff=lfs merge=lfs -text
*.pdf filter=lfs diff=lfs merge=lfs -text
```

**Pre-commit Hook（Python 跨平台）：**

```python
#!/usr/bin/env python3
# .git/hooks/pre-commit
# 拦截未走 LFS 的大文件

import os
import sys
import subprocess

MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB
LFS_EXTENSIONS = {
    '.fbx', '.obj', '.blend', '.ma', '.mb', '.max',
    '.psd', '.psb', '.tga', '.exr', '.tif', '.tiff',
    '.png', '.jpg', '.jpeg',
    '.wav', '.mp3', '.ogg', '.flac',
    '.mp4', '.mov', '.webm', '.avi',
    '.hdr', '.cubemap',
    '.zip', '.rar', '.7z',
}

def get_staged_files():
    result = subprocess.run(
        ['git', 'diff', '--cached', '--name-only', '--diff-filter=ACM'],
        capture_output=True, text=True
    )
    return result.stdout.strip().split('\n') if result.stdout.strip() else []

def check_lfs_tracked(filepath):
    """检查文件是否被 LFS tracking 规则匹配"""
    result = subprocess.run(
        ['git', 'check-attr', 'filter', '--', filepath],
        capture_output=True, text=True
    )
    return 'lfs' in result.stdout

def main():
    staged = get_staged_files()
    errors = []

    for filepath in staged:
        if not os.path.exists(filepath):
            continue

        ext = os.path.splitext(filepath)[1].lower()

        # 检查大文件
        size = os.path.getsize(filepath)
        if size > MAX_FILE_SIZE:
            if not check_lfs_tracked(filepath):
                errors.append(
                    f"❌ {filepath} ({size/1024/1024:.1f}MB) 超过 10MB 但未被 LFS tracking！\n"
                    f"   请在 .gitattributes 中添加: *{ext} filter=lfs diff=lfs merge=lfs -text\n"
                    f"   然后运行: git lfs migrate import --include='*{ext}'"
                )

        # 检查 LFS 类型是否遗漏
        if ext in LFS_EXTENSIONS and not check_lfs_tracked(filepath):
            errors.append(
                f"⚠️ {filepath} 是二进制类型 .{ext} 但未被 LFS tracking！\n"
                f"   请检查 .gitattributes 配置"
            )

    if errors:
        print('\n'.join(errors))
        print('\n🚫 Commit 已被拦截。修复以上问题后重新提交。')
        sys.exit(1)
    else:
        print('✅ Pre-commit 检查通过')

if __name__ == '__main__':
    main()
```

**历史清理脚本（自动化迁移）：**

```bash
#!/bin/bash
# migrate-large-files.sh
# ⚠️ 破坏性操作！先备份仓库！

set -e

REPO_DIR="/path/to/repo"
cd "$REPO_DIR"

echo "=== Step 1: 分析仓库中最大的文件 ==="
git filter-repo --analyze
# 结果在 .git/filter-repo/analysis/ 目录下

echo "=== Step 2: 安装 git-filter-repo（如果没装）==="
pip install git-filter-repo 2>/dev/null || true

echo "=== Step 3: 将已有的大文件类型迁移到 LFS ==="
# 先 migrate 到 LFS（这会重写历史）
git lfs migrate import \
    --include="*.fbx,*.psd,*.psb,*.tga,*.exr,*.wav,*.mp3,*.mp4" \
    --everything

echo "=== Step 4: 移除历史中残留的超大文件（>50MB，非 LFS 的）==="
git filter-repo --strip-blobs-bigger-than 50M

echo "=== Step 5: 清理引用和 GC ==="
git reflog expire --expire=now --all
git gc --prune=now --aggressive

echo "=== Step 6: 检查结果 ==="
echo "仓库大小: $(du -sh .git | cut -f1)"
echo "LFS 对象大小: $(du -sh .git/lfs 2>/dev/null | cut -f1 || echo 'N/A')"

echo ""
echo "⚠️ 重要：历史已被重写！"
echo "1. 在远程创建新的 bare 仓库"
echo "2. git remote add origin <new-url>"
echo "3. git push origin --all"
echo "4. git push origin --tags"
echo "5. 通知所有人删除本地仓库重新 clone"
echo "6. 更新 CI/CD 配置中的仓库地址"
```

**方案对比表：**

| 维度 | Git + LFS | Perforce | Git + 对象存储 |
|------|-----------|----------|---------------|
| 学习成本 | 低（美术用 GUI） | 中（需要培训） | 高（需自建工具） |
| 大文件性能 | 中（< 100GB 可控） | 优秀（专为大文件设计） | 优秀（CDN 加速） |
| 分支支持 | 强（Git 原生） | 中（Streams） | 弱（需手动管理） |
| Unity 集成 | 内置 VCS 支持 | 内置 VCS 支持 | 需自定义 |
| 权限控制 | 粗（仓库级） | 细（目录级） | 中（对象级） |
| 外包友好度 | 中（LFS 带宽限制） | 高（Proxy/Replica） | 高（预签名链接） |
| 成本 | 低（自建）/中（托管） | 高（License） | 低（按量付费） |
| 推荐规模 | < 30 人 | 30-300 人 | 分布式团队 |

### ⚡ 实战经验

1. **git filter-repo 替代 BFG Repo-Cleaner**：BFG 已不再积极维护，git-filter-repo 是 Git 官方推荐的历史重写工具，速度更快、功能更全
2. **Partial Clone 是救星**：`git clone --filter=blob:none` 只拉取 commit 和 tree，blob 按需下载。配合 Sparse Checkout，新人 clone 可以从 4 小时降到 3 分钟
3. **Unity .meta 文件冲突是美术团队最大痛点**：制定规范——同一资产同一时间只能一个人编辑。Git Lock（配合 LFS）可以实现文件锁定，美术人员通过 GUI 操作
4. **LFS 带宽是隐形成本**：GitHub Git LFS 免费额度只有 1GB/月，30 人团队轻松超限。自建 LFS Server（如 `lfs-self-hosted`）或用 GitLab 自托管 LFS 更可控
5. **Perforce 迁移别一次性全迁**：先迁当前活跃项目的资产，历史归档资产放到冷存储。Perforce 的 `p4 fuse` 可以直接挂载 S3 做冷热分层
6. **培训是被低估的环节**：给美术做一次 2 小时的 Git GUI + LFS 培训，能减少 80% 的日常 `git` 问题。准备一份图文操作手册

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道仓库为什么膨胀 | Git 存储模型 / LFS 原理 | 学 Git Object Model（Blob/Tree/Commit）、LFS Architecture |
| 不了解 Perforce 适不适合 | Perforce vs Git 的本质区别 | 学 Perforce Streams / Client Spec / Protections |
| 历史清理不敢动手 | git filter-repo / 历史重写 | 在测试仓库上练习 filter-repo，理解 force push 影响 |
| Partial Clone 不理解 | Git Protocol v2 / blob filter | 学 `--filter=blob:none` / `--filter=tree:0` |
| 外包权限设计混乱 | VCS 权限模型 | 学 Perforce Protections / GitLab Subgroup 权限 |
| Unity .meta 冲突频发 | Unity Asset Importer / .meta 机制 | 学 Unity Version Control 最佳实践、Git LFS Lock |

### 🔗 相关问题

- [美术资产 Diff 审查管线](../pipeline/art-asset-diff-review-pipeline.md)：版本控制解决了存储问题，审查解决了质量问题——两者配合
- [Unity Asset Checker 工具](../pipeline/unity-asset-checker-tool.md)：自动化检查资产规范，在 commit 前拦截问题
- 面试追问：如果项目要同时支持 PC 和移动端，资产管线怎么分？（提示：Perforce Streams 或 Git 分支 + Asset Variant 系统）
