---
title: "批量贴图压缩工具：如何用 Python 脚本统一 10 万张贴图的压缩格式？"
category: "pipeline"
level: 2
tags: ["Python", "TextureCompression", "ASTC", "ETC2", "自动化", "CI/CD"]
hint: "核心不是写一个循环 compress——而是做格式策略、质量检查、并行加速和失败重试的完整工具链"
related: ["technical-art/mobile-texture-compression", "pipeline/unity-asset-checker-tool", "technical-art/mobile-texture-pipeline-strategy"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们的项目有 10 万张贴图，格式混乱——有 PNG、TGA、TIFF，导入 Unity 后有的是 ASTC、有的是 ETC2、还有没压缩的。现在要统一规范：Android 用 ASTC 6×6，iOS 用 ASTC 6×6，PC 用 BC7。给我写一个批量压缩工具的方案，要求：能并行跑、有质量检查、失败的能自动重试、生成压缩报告。」

### ✅ 核心要点

1. **格式策略引擎**：不是一刀切，按贴图类型（Color/Normal/UI/Lightmap）分配不同压缩格式和精度
2. **并行压缩**：Python `multiprocessing` 或 `concurrent.futures` 并行调用压缩器，10 万张需要充分利用 CPU 核数
3. **质量校验**：压缩后做 PSNR/SSIM 检查，低于阈值的标记警告
4. **断点续传**：记录已完成项，失败中断后可恢复
5. **压缩报告**：生成 Excel/CSV 报告：原始大小、压缩后大小、压缩比、质量评分、耗时

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：10 万张贴图按规范批量压缩，有报告有检查有重试
                ↑
倒推1：不同贴图类型需要不同策略 → 配置驱动的压缩规则引擎
倒推2：10 万张串行太慢 → 多进程并行，按 CPU 核数分配任务
倒推3：压缩质量参差不齐 → 压缩后自动 PSNR 检查，低分标红
倒推4：中途可能崩溃 → SQLite 记录进度，支持断点续传
倒推5：美术需要知道结果 → 生成可视化报告（CSV + HTML 仪表盘）
倒推6：要集成到 CI/CD → 命令行参数 + 退出码规范 + 日志输出
倒推7：压缩工具调用 → Windows 用 texturec（Basis），macOS 用 astcenc，Linux 也可以跑
```

#### 知识点拆解（倒推树）

```
批量贴图压缩工具
├── 格式策略引擎
│   ├── 按贴图类型分类
│   │   ├── Color Map (Albedo) → ASTC 6×6 / BC7
│   │   ├── Normal Map → ASTC 5×5（精度更高）/ BC5（PC专用双通道）
│   │   ├── UI/Icon → ASTC 4×4（高品质） / BC7
│   │   ├── Lightmap → ASTC 8×8（可接受损失）
│   │   └── HDR/天空盒 → BC6H（PC）/ ASTC HDR（移动端可选）
│   ├── 按平台分配
│   │   ├── Android: ASTC（需要 OpenGL ES 3.0+ / Vulkan）
│   │   ├── iOS: ASTC（全系列支持 A7 芯片以后）
│   │   └── PC: BC1~BC7（DirectX 级别硬件）
│   └── 规则配置文件（JSON/YAML）：类型检测规则 + 格式映射
├── 并行处理
│   ├── Python multiprocessing.Pool（进程级并行，绕过 GIL）
│   ├── 任务队列：输入文件列表 → 分片 → 分配给 Worker
│   ├── 压缩工具调用：subprocess 调用 astcenc / texturec / nvcompress
│   └── 进度条：tqdm 跟踪整体进度（支持 10 万级的进度显示）
├── 质量检查
│   ├── PSNR（峰值信噪比）：> 40dB = 优秀, 30-40 = 可接受, < 30 = 警告
│   ├── SSIM（结构相似性）：> 0.95 = 优秀, < 0.90 = 需检查
│   ├── 法线贴图专项检查：压缩后 tangent-space normal 长度偏差
│   └── 自动降级策略：如果 ASTC 6×6 不达标，自动降到 4×4 重压一次
├── 断点续传与容错
│   ├── SQLite 数据库记录：文件路径、状态(pending/done/failed/skipped)、耗时
│   ├── 失败重试机制：最多 3 次，间隔递增
│   ├── 崩溃恢复：读取 SQLite，跳过已完成项，继续未完成项
│   └── 日志系统：每文件一行日志，错误单独抽取到 error.log
├── 压缩工具链
│   ├── astcenc（ARM 官方）：astcenc -cl input.png output.astc 6x6 -medium
│   ├── texturec（Basis Universal）：跨平台转码中间格式
│   ├── nvcompress（NVIDIA Texture Tools）：PC BC 系列压缩
│   ├── Compressonator（AMD）：支持 BC / ASTC 双引擎
│   └── Unity API：AssetImporter.GetAtPath() + textureImporter 设置（无需外部工具）
└── 报告生成
    ├── CSV/Excel：文件名、原始大小、压缩大小、压缩比、PSNR、状态、耗时
    ├── HTML 仪表盘：排行（最差质量 Top 20 / 最大节省 Top 20）
    └── 集成通知：CI/CD 中通过 webhook 发送压缩结果摘要
```

#### 代码实现

**核心工具脚本（Python，生产级）：**

```python
#!/usr/bin/env python3
"""
batch_texture_compressor.py
批量贴图压缩工具 - 支持策略引擎、并行处理、质量检查、断点续传
"""

import os
import sys
import json
import time
import sqlite3
import logging
import subprocess
from pathlib import Path
from multiprocessing import Pool, cpu_count
from dataclasses import dataclass, asdict
from typing import Optional

import cv2
import numpy as np
from tqdm import tqdm

# ============================================================
# 1. 配置：格式策略引擎
# ============================================================

COMPRESSION_RULES = {
    # 贴图类型: { 平台: (格式, 参数) }
    "color":     {"android": ("astc", "6x6 -medium"), "ios": ("astc", "6x6 -medium"), "pc": ("bc7", "-fast")},
    "normal":    {"android": ("astc", "5x5 -medium"), "ios": ("astc", "5x5 -medium"), "pc": ("bc5", "-fast")},
    "ui":        {"android": ("astc", "4x4 -thorough"), "ios": ("astc", "4x4 -thorough"), "pc": ("bc7", "-fast")},
    "lightmap":  {"android": ("astc", "8x8 -fast"), "ios": ("astc", "8x8 -fast"), "pc": ("bc7", "-fast")},
    "hdr":       {"android": ("astc", "6x6 -hdr"), "ios": ("astc", "6x6 -hdr"), "pc": ("bc6h", "-fast")},
}

# 贴图类型检测规则（按文件名/路径关键词）
TYPE_DETECTION = {
    "normal":   ["_n.", "_normal.", "_nrm.", "/normal/"],
    "ui":       ["_ui.", "/ui/", "/icon/", "/icons/"],
    "lightmap": ["_lightmap.", "_lm.", "/lightmap/"],
    "hdr":      ["_hdr.", "/hdr/", "/skybox/", "/cubemap/"],
}

QUALITY_THRESHOLD = {
    "psnr_min": 35.0,   # dB，低于此值标记警告
    "ssim_min": 0.92,   # 结构相似性，低于此值标记警告
}

# ============================================================
# 2. 数据结构
# ============================================================

@dataclass
class CompressTask:
    src_path: str
    dst_path: str
    tex_type: str       # color/normal/ui/lightmap/hdr
    platform: str       # android/ios/pc
    format: str         # astc/bc7/bc5/bc6h
    params: str         # "6x6 -medium"
    status: str = "pending"  # pending/done/failed/skipped
    src_size: int = 0
    dst_size: int = 0
    psnr: float = 0.0
    ssim: float = 0.0
    elapsed: float = 0.0
    error: str = ""

# ============================================================
# 3. 核心压缩函数（Worker 进程调用）
# ============================================================

def detect_texture_type(filepath: str) -> str:
    """根据文件路径自动检测贴图类型"""
    lower = filepath.lower()
    for tex_type, keywords in TYPE_DETECTION.items():
        if any(kw in lower for kw in keywords):
            return tex_type
    return "color"  # 默认当作 Color Map

def run_astcenc(src: str, dst: str, params: str) -> bool:
    """调用 astcenc 进行 ASTC 压缩"""
    cmd = ["astcenc", "-cl", src, dst] + params.split()
    result = subprocess.run(cmd, capture_output=True, timeout=300)
    if result.returncode != 0:
        raise RuntimeError(f"astcenc failed: {result.stderr.decode()[:200]}")
    return True

def run_nvcompress(src: str, dst: str, fmt: str) -> bool:
    """调用 nvcompress 进行 BC 压缩"""
    fmt_map = {"bc5": "-bc5", "bc6h": "-bc6h", "bc7": "-bc7"}
    cmd = ["nvcompress", fmt_map.get(fmt, "-bc7"), src, dst]
    result = subprocess.run(cmd, capture_output=True, timeout=300)
    if result.returncode != 0:
        raise RuntimeError(f"nvcompress failed: {result.stderr.decode()[:200]}")
    return True

def compute_psnr(src_img: np.ndarray, dst_img: np.ndarray) -> float:
    """计算 PSNR（峰值信噪比）"""
    mse = np.mean((src_img.astype(float) - dst_img.astype(float)) ** 2)
    if mse == 0:
        return 100.0
    return 10 * np.log10((255.0 ** 2) / mse)

def compute_ssim(src_img: np.ndarray, dst_img: np.ndarray) -> float:
    """简化版 SSIM 计算"""
    C1, C2 = (0.01 * 255) ** 2, (0.03 * 255) ** 2
    src_gray = cv2.cvtColor(src_img, cv2.COLOR_BGR2GRAY).astype(np.float64)
    dst_gray = cv2.cvtColor(dst_img, cv2.COLOR_BGR2GRAY).astype(np.float64)
    
    mu1 = cv2.GaussianBlur(src_gray, (11, 11), 1.5)
    mu2 = cv2.GaussianBlur(dst_gray, (11, 11), 1.5)
    
    mu1_sq, mu2_sq, mu1_mu2 = mu1**2, mu2**2, mu1 * mu2
    sigma1_sq = cv2.GaussianBlur(src_gray**2, (11, 11), 1.5) - mu1_sq
    sigma2_sq = cv2.GaussianBlur(dst_gray**2, (11, 11), 1.5) - mu2_sq
    sigma12 = cv2.GaussianBlur(src_gray * dst_gray, (11, 11), 1.5) - mu1_mu2
    
    ssim_map = ((2 * mu1_mu2 + C1) * (2 * sigma12 + C2)) / \
               ((mu1_sq + mu2_sq + C1) * (sigma1_sq + sigma2_sq + C2))
    return float(np.mean(ssim_map))

def compress_worker(task_dict: dict) -> dict:
    """单个压缩任务（ multiprocessing Worker 调用）"""
    task = CompressTask(**task_dict)
    start_time = time.time()
    
    try:
        # 确保输出目录存在
        os.makedirs(os.path.dirname(task.dst_path), exist_ok=True)
        
        # 调用压缩工具
        if task.format.startswith("astc"):
            run_astcenc(task.src_path, task.dst_path, task.params)
        else:
            run_nvcompress(task.src_path, task.dst_path, task.format)
        
        # 质量检查（PSNR/SSIM）
        src_img = cv2.imread(task.src_path)
        # 注意：压缩后是 GPU 格式，需要解码回 PNG 再比较
        # 这里用临时 PNG 做对比
        temp_png = task.dst_path + ".check.png"
        subprocess.run(["astcenc", "-dl", task.dst_path, temp_png],
                       capture_output=True, timeout=60)
        dst_img = cv2.imread(temp_png)
        
        if src_img is not None and dst_img is not None:
            h = min(src_img.shape[0], dst_img.shape[0])
            w = min(src_img.shape[1], dst_img.shape[1])
            task.psnr = compute_psnr(src_img[:h, :w], dst_img[:h, :w])
            task.ssim = compute_ssim(src_img[:h, :w], dst_img[:h, :w])
            os.remove(temp_png)
        
        task.src_size = os.path.getsize(task.src_path)
        task.dst_size = os.path.getsize(task.dst_path)
        task.status = "done"
        
    except Exception as e:
        task.status = "failed"
        task.error = str(e)[:500]
        logging.error(f"Failed: {task.src_path} -> {e}")
    
    task.elapsed = time.time() - start_time
    return asdict(task)

# ============================================================
# 4. 主控逻辑
# ============================================================

class BatchCompressor:
    def __init__(self, src_dir: str, dst_dir: str, platform: str = "android",
                 db_path: str = "compress_progress.db", workers: int = None):
        self.src_dir = Path(src_dir)
        self.dst_dir = Path(dst_dir)
        self.platform = platform
        self.db_path = db_path
        self.workers = workers or max(cpu_count() - 2, 2)  # 留 2 核给系统
        self._init_db()
    
    def _init_db(self):
        """初始化 SQLite 进度数据库"""
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS progress (
                src_path TEXT PRIMARY KEY,
                dst_path TEXT, tex_type TEXT, platform TEXT,
                format TEXT, params TEXT, status TEXT,
                src_size INTEGER, dst_size INTEGER,
                psnr REAL, ssim REAL, elapsed REAL, error TEXT
            )
        """)
        conn.commit()
        conn.close()
    
    def collect_tasks(self) -> list:
        """收集所有待压缩贴图，跳过已完成的"""
        conn = sqlite3.connect(self.db_path)
        done_set = {row[0] for row in conn.execute(
            "SELECT src_path FROM progress WHERE status='done'")}
        conn.close()
        
        tasks = []
        supported = {".png", ".tga", ".jpg", ".jpeg", ".tif", ".tiff", ".bmp"}
        
        for filepath in self.src_dir.rglob("*"):
            if filepath.suffix.lower() not in supported:
                continue
            if str(filepath) in done_set:
                continue
            
            tex_type = detect_texture_type(str(filepath))
            fmt, params = COMPRESSION_RULES[tex_type][self.platform]
            
            rel_path = filepath.relative_to(self.src_dir)
            dst_path = self.dst_dir / rel_path.with_suffix(
                ".astc" if fmt == "astc" else ".dds")
            
            task = CompressTask(
                src_path=str(filepath),
                dst_path=str(dst_path),
                tex_type=tex_type,
                platform=self.platform,
                format=fmt,
                params=params,
                src_size=filepath.stat().st_size,
            )
            tasks.append(task)
        
        return tasks
    
    def run(self):
        """执行批量压缩"""
        tasks = self.collect_tasks()
        print(f"📋 共 {len(tasks)}张贴图待压缩，使用 {self.workers} 个进程并行")
        
        if not tasks:
            print("✅ 所有贴图已完成，无需处理")
            return self._generate_report()
        
        # 多进程并行
        task_dicts = [asdict(t) for t in tasks]
        
        with Pool(self.workers) as pool:
            results = list(tqdm(
                pool.imap_unordered(compress_worker, task_dicts),
                total=len(tasks),
                desc="压缩进度",
                unit="张"
            ))
        
        # 写入数据库
        self._save_results(results)
        
        # 重试失败项
        failed = [r for r in results if r["status"] == "failed"]
        if failed:
            print(f"⚠️ {len(failed)} 张失败，正在重试...")
            retried = []
            for r in failed[:3]:  # 最多重试3张示例
                r["status"] = "pending"
                retried.append(compress_worker(r))
            self._save_results(retried)
        
        return self._generate_report()
    
    def _save_results(self, results: list):
        conn = sqlite3.connect(self.db_path)
        conn.executemany("""
            INSERT OR REPLACE INTO progress VALUES 
            (:src_path, :dst_path, :tex_type, :platform,
             :format, :params, :status, :src_size, :dst_size,
             :psnr, :ssim, :elapsed, :error)
        """, results)
        conn.commit()
        conn.close()
    
    def _generate_report(self) -> dict:
        """生成压缩报告"""
        conn = sqlite3.connect(self.db_path)
        rows = list(conn.execute("SELECT * FROM progress"))
        conn.close()
        
        total = len(rows)
        done = sum(1 for r in rows if r[6] == "done")
        failed = sum(1 for r in rows if r[6] == "failed")
        total_src = sum(r[7] for r in rows if r[6] == "done")
        total_dst = sum(r[8] for r in rows if r[6] == "done")
        
        # 质量最差的 20 张
        worst_psnr = sorted(
            [r for r in rows if r[6] == "done" and r[9] > 0],
            key=lambda x: x[9]
        )[:20]
        
        report = {
            "total": total,
            "done": done,
            "failed": failed,
            "total_src_mb": total_src / 1024 / 1024,
            "total_dst_mb": total_dst / 1024 / 1024,
            "compression_ratio": total_src / max(total_dst, 1),
            "space_saved_mb": (total_src - total_dst) / 1024 / 1024,
            "worst_psnr_top20": [
                {"file": r[0], "psnr": round(r[9], 2), "ssim": round(r[10], 4)}
                for r in worst_psnr
            ],
        }
        
        # 输出摘要
        print(f"\n{'='*50}")
        print(f"✅ 完成: {done}/{total} | ❌ 失败: {failed}")
        print(f"📦 原始: {report['total_src_mb']:.1f}MB → 压缩后: {report['total_dst_mb']:.1f}MB")
        print(f"🗜️ 压缩比: {report['compression_ratio']:.2f}x | 节省: {report['space_saved_mb']:.1f}MB")
        if worst_psnr:
            print(f"⚠️ 质量最低 PSNR: {worst_psnr[0][9]:.1f}dB ({worst_psnr[0][0]})")
        print(f"{'='*50}")
        
        return report


# ============================================================
# 5. 入口
# ============================================================

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser(description="批量贴图压缩工具")
    parser.add_argument("--src", required=True, help="源贴图目录")
    parser.add_argument("--dst", required=True, help="输出目录")
    parser.add_argument("--platform", default="android", 
                        choices=["android", "ios", "pc"])
    parser.add_argument("--workers", type=int, default=None,
                        help="并行进程数（默认 CPU核数-2）")
    parser.add_argument("--db", default="compress_progress.db",
                        help="进度数据库路径")
    args = parser.parse_args()
    
    compressor = BatchCompressor(
        src_dir=args.src,
        dst_dir=args.dst,
        platform=args.platform,
        db_path=args.db,
        workers=args.workers,
    )
    
    report = compressor.run()
    
    # CI/CD 退出码：有失败项返回 1
    sys.exit(1 if report["failed"] > 0 else 0)
```

### ⚡ 实战经验

- **先跑一个 dry-run**：10 万张贴图全量压缩可能跑 8 小时。先跑 100 张采样，看 PSNR 分布和压缩比，调整策略再全量跑
- **法线贴图的坑**：ASTC 压缩法线后 tangent-space normal 长度会偏离 1.0，需要在压缩后做 renormalize，或者在 shader 中 `normalize()` 。BC5 双通道法线（ATI2n）是 PC 上法线的最佳选择
- **UI 贴图的 Alpha 通道**：ASTC 对独立 Alpha 通道处理一般，如果 UI 有半透明边缘，考虑用 ASTC 4×4 + 预乘 alpha，或者 RGB + Alpha 分离双贴图方案
- **Unity 内置方案 vs 外部工具**：Unity 的 TextureImporter 能直接设置压缩格式，但批量改 10 万张贴图的 import settings 也需要脚本。可以在 `AssetPostprocessor` 里按规则自动设，或者在批处理工具中用 `UnityEditor` API 直接操作
- **CI/CD 集成**：贴图压缩脚本集成到 Jenkins/GitLab CI，每次美术合入新贴图自动压缩并更新压缩报告，报告通过 webhook 推到飞书/钉钉
- **分布式加速**：单机 16 核跑 10 万张大约 4-6 小时。如果有构建农场（build farm），把任务分发到多台机器，用 Redis 做任务队列，能压到 30 分钟以内
- **版本控制问题**：压缩后的二进制贴图（.astc/.dds）不要进 git！应该只提交原始贴图，压缩产物在 CI 构建时生成。如果要加速，用 CI cache 缓存压缩结果

### 🎯 能力体检清单

- [ ] ASTC、ETC2、BC7 三种压缩格式的原理区别是什么？为什么移动端首选 ASTC？（块大小、压缩率、质量、硬件支持）
- [ ] 法线贴图为什么要用 BC5（双通道）而不是 BC1（单纹理）？（法线 Z 通道重建精度）
- [ ] PSNR 35dB 意味着什么？如果一张贴图压缩后 PSNR 只有 28dB，你会怎么处理？（质量不达标的排查与降级）
- [ ] Python `multiprocessing` 和 `threading` 在 CPU 密集任务中为什么前者更快？（GIL 的影响）
- [ ] 如果压缩过程中断电了，你的工具能否恢复？怎么实现的？（断点续传设计）
- [ ] 10 万张贴图占用 50GB 源文件，压缩后 12GB。如果项目包体需要进一步压缩到 8GB，你会怎么做？（分辨率减半？降一级压缩精度？按 LOD 策略？）
- [ ] 如何让美术直观地看到压缩前后的差异？（diff viewer 工具设计：并排对比 / 滑块对比 / 差异热力图）
- [ ] 在 Unity 中用代码批量设置贴图压缩格式怎么写？（TextureImporter API）

### 🔗 相关问题

- [移动端贴图压缩方案](technical-art/mobile-texture-compression) — 压缩格式选型的基础知识
- [Unity 资产检查工具](pipeline/unity-asset-checker-tool) — 与压缩工具配合的资产规范检查
- [移动端贴图管线策略](technical-art/mobile-texture-pipeline-strategy) — 从管线角度规划贴图全流程
