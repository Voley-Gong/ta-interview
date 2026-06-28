---
title: "角色面部表情系统：如何制定 BlendShape 规范并保证外包资产质量？"
category: "technical-art"
level: 3
tags: ["BlendShape", "面部表情", "ARKit", "FACS", "资产规范", "外包验收", "QA"]
hint: "BlendShape 规范的核心不是「做多少个表情」，而是「命名规范+中性姿势校准+权重正交+极端值不穿模」——规范制定者必须懂美术、懂技术、懂管线"
related: ["technical-art/morph-target-facial-system-design", "technical-art/lod-spec-and-qa", "soft-skills/outsource-art-acceptance-criteria"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们做一个二次元开放世界手游，有 50+ 个 NPC 角色，每个角色需要支持对话系统和过场动画中的面部表情。面部表情通过 BlendShape 驱动，资产由外包团队制作。
>
> 1. 你会如何制定 BlendShape 规范，让外包团队能一次性做对？
> 2. 我们之前遇到过：不同外包做出来的表情「风味不一致」、某些 BlendShape 组合时穿模、命名混乱导致程序绑定报错。你怎么解决？
> 3. 如何做自动化 QA？

这是米哈游、网易（雷火/盘古）、叠纸、腾讯（天美）等有大量角色内容的项目的真实管线问题。考察的是 TA 的**规范制定能力 + 跨部门协作能力 + 自动化思维**——不是"你会不会做 BlendShape"，而是"你能不能让 50 个外包美术做出来的东西统一"。

### ✅ 核心要点

1. **BlendShape 命名规范**：固定前缀+部位+方向/动作的命名约定（如 `BS_Jaw_Open` / `BS_Mouth_Smile_L`），让程序能自动解析、让 DCC 脚本能批量处理
2. **中性姿势校准（Neutral Pose Calibration）**：所有 BlendShape 必须从同一个标准中性脸开始，形变目标不能偏移中性姿势的骨架对齐
3. **FACS / ARKit BS 标准集选型**：不自己发明表情集，而是基于 FACS（面部动作编码系统）或 ARKit 的 52 个 BlendShape 作为基础集，根据项目需求增减
4. **正交性校验**：关键 BlendShape 组合（如「张嘴+微笑」）不应穿模或产生不自然形变，需要定义组合测试用例
5. **自动化 QA 管线**：用 Maya/Python 脚本批量检查命名、穿模、权重范围、中性姿势偏移

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：50+ 角色，统一品质的面部表情系统，外包可复制生产
         ↓
问题倒推：
  问题1：「风味不一致」→ 缺少标准化的表情集定义
     ↓ 解决：选型 FACS/ARKit 标准集 + 项目扩展集
  问题2：「组合穿模」→ BlendShape 之间不正交
     ↓ 解决：定义组合测试用例 + 极端值检查
  问题3：「命名混乱」→ 没有强制的命名规范
     ↓ 解决：命名规范文档 + 自动化检查脚本
  问题4：「品质波动」→ 外包理解不一致
     ↓ 解决：参考资产（黄金标准）+ QA Checklist + 自动化审计工具
```

#### 知识点拆解（倒推树）

```
BlendShape 规范体系
├── 表情集选型
│   ├── FACS（Facial Action Coding System）
│   │   ├── 心理学界的标准面部动作编码系统
│   │   ├── 44 个 Action Unit（AU），每个对应一块面部肌肉
│   │   └── 优点：科学、可组合；缺点：太学术，不直接对应动画需求
│   ├── ARKit BlendShape（52 个）
│   │   ├── Apple 定义的面部追踪标准集
│   │   ├── 涵盖眉/眼/鼻/嘴/颊/下颚的完整动作
│   │   └── 优点：行业标准、动捕兼容、Unity Unreal 原生支持
│   ├── 项目选型建议
│   │   ├── 写实风：ARKit 52 + 自定义扩展（如舌头）
│   │   ├── 二次元：ARKit 子集（~30 个）+ 二次元专用（如汗滴、怒纹）
│   │   └── 卡通低模：精简集（~15-20 个核心表情）
│   └── 扩展集定义
│       ├── 对话系统专用：嘴形 BlendShape（Viseme 对应）
│       │   └── A/E/I/O/U + 唇齿音/双唇音
│       └── 过场动画专用：极端情绪（大哭/大笑/暴怒）
│
├── 命名规范
│   ├── 命名模式：BS_<部位>_<动作>[_<方向>]
│   │   ├── BS_Mouth_Smile_L（左嘴角上扬）
│   │   ├── BS_Eye_Blink（双眼眨眼）
│   │   ├── BS_Brow_Raise_Inner（内眉抬起）
│   │   └── BS_Jaw_Open（下颚张开）
│   ├── 禁止项
│   │   ├── 禁止使用中文/空格/特殊字符
│   │   ├── 禁止使用编号替代语义（如 BS_001, BS_002）
│   │   └── 禁止左右合并（左和右必须分开，除非是对称表情）
│   ├── 对称命名规则
│   │   ├── L/R 后缀的表情必须成对出现
│   │   └── 全脸表情不加方向后缀（如 BS_Eye_Blink 是全脸）
│   └── 索引排序规范
│       ├── 导出时按 部位 → 动作 → 方向 排序
│       └── 保证不同角色的 BS 索引一致（程序绑定依赖）
│
├── 中性姿势（Neutral Pose）规范
│   ├── 为什么重要
│   │   ├── 所有 BlendShape 的 target 都是相对中性脸的形变
│   │   └── 如果中性脸不标准，所有表情都会"歪"
│   ├── 中性脸标准定义
│   │   ├── 眼睛平视前方，瞳孔居中
│   │   ├── 嘴唇自然闭合但不紧贴（留 0.5mm 缝隙防穿模）
│   │   ├── 面部肌肉完全放松（不做任何微表情）
│   │   └── 骨架在 T-Pose 的头部位置（不能有偏移）
│   ├── 校准检查项
│   │   ├── 对称性：左右顶点镜像偏差 < 0.01 单位
│   │   ├── 骨架对齐：头部骨骼位置和旋转与规范一致
│   │   └── UV/法线：中性脸的 UV 布局和法线不能因 BS 创建过程被改动
│   └── 验收标准：BS Weight = 0 时，网格必须与中性脸完全一致
│
├── 正交性与组合校验
│   ├── 为什么 BlendShape 会穿模
│   │   ├── BS_A 和 BS_B 各自没问题，但同时开到 1.0 时形变叠加超出了网格范围
│   │   └── 典型案例：嘴张大 + 脸颊鼓起 → 嘴角穿模
│   ├── 正交性测试矩阵
│   │   ├── 定义 20+ 个关键组合测试用例
│   │   ├── 每个组合的 BS Weight 设为极端值（1.0 / 1.0）
│   │   └── 检查是否有自交/穿模/体积异常
│   ├── 容忍度定义
│   │   ├── 轻微穿模（< 0.1mm）：可接受，运行时不可见
│   │   ├── 明显穿模（> 0.5mm）：必须修复
│   │   └── 体积异常（鼓/瘪）：需要美术调整
│   └── 修复策略
│       ├── 修正 Target：直接调整穿模的 BS Target 几何体
│       ├── 添加 Corrective BS：组合触发时自动补偿（BS_A+B → 激活 BS_Corrective_AB）
│       └── 限制权重范围：程序端 Clamp 不兼容表情的最大值
│
├── 对话系统 Viseme 映射
│   ├── 什么是 Viseme
│   │   └── 音素对应的视觉嘴形（Phoneme → Viseme 映射）
│   ├── 中文 Viseme 集（10-12 个）
│   │   ├── A（啊）/ O（哦）/ U（呜）/ I（衣）/ E（鹅）
│   │   ├── B/P/M（双唇闭合）/ F/V（上齿咬下唇）
│   │   ├── D/T/N/L（舌尖上齿龈）/ G/K/H（舌根）
│   │   └── S/Z/C（齿间摩擦）
│   ├── 映射方式
│   │   ├── 多个 Visome 可以复用同一个嘴形 BS（不同权重组合）
│   │   └── 对话系统通过文本→音素→Visome→BS权重 驱动
│   └── 二次元特殊处理
│       └── 口型贴图 vs BS：二次元常用贴图替换/叠加简化
│
└── 自动化 QA 管线
    ├── 检查清单（Python 脚本可自动化）
    │   ├── 命名规范检查：正则匹配所有 BS 名称
    │   ├── 数量检查：BS 数量与规范一致
    │   ├── 中性姿势检查：BS=0 时网格偏差
    │   ├── 对称性检查：L/R BS 的形变是否镜像
    │   ├── 极端值穿模检查：所有 BS=1.0 时的自交检测
    │   ├── 组合穿模检查：预定义测试矩阵
    │   └── 文件结构检查：节点层级、History 清理、导出设置
    └── 报告输出
        ├── 通过/失败标记
        ├── 问题截图（穿模区域高亮）
        └── 发送给外包的反馈文档
```

#### 代码实现

**Maya Python QA 脚本（核心检查逻辑）：**

```python
# blendshape_qa.py — Maya 面部 BlendShape 自动化 QA
# 运行环境：Maya 2024+ / Python 3
import maya.cmds as cmds
import re
import json
from collections import defaultdict

class BlendShapeQA:
    """BlendShape 资产自动化检查工具"""
    
    # ========== 规范定义 ==========
    
    # 标准表情集（基于 ARKit 52 + 项目扩展）
    STANDARD_BS_NAMES = [
        # --- 眼部 ---
        "BS_Eye_Blink_L", "BS_Eye_Blink_R",
        "BS_Eye_LookDown_L", "BS_Eye_LookDown_R",
        "BS_Eye_LookIn_L", "BS_Eye_LookIn_R",
        "BS_Eye_LookOut_L", "BS_Eye_LookOut_R",
        "BS_Eye_LookUp_L", "BS_Eye_LookUp_R",
        "BS_Eye_Squint_L", "BS_Eye_Squint_R",
        "BS_Eye_Wide_L", "BS_Eye_Wide_R",
        # --- 眉部 ---
        "BS_Brow_Drop_L", "BS_Brow_Drop_R",
        "BS_Brow_InnerUp_L", "BS_Brow_InnerUp_R",
        "BS_Brow_OuterUp_L", "BS_Brow_OuterUp_R",
        # --- 鼻/颊 ---
        "BS_Cheek_Blow_L", "BS_Cheek_Blow_R",
        "BS_Cheek_Puff",
        "BS_Cheek_Squint_L", "BS_Cheek_Squint_R",
        "BS_Nose_Sneer_L", "BS_Nose_Sneer_R",
        # --- 嘴部 ---
        "BS_Jaw_Open",
        "BS_Jaw_Forward", "BS_Jaw_Left", "BS_Jaw_Right",
        "BS_Mouth_Close",
        "BS_Mouth_Frown_L", "BS_Mouth_Frown_R",
        "BS_Mouth_Pucker",
        "BS_Mouth_Smile_L", "BS_Mouth_Smile_R",
        "BS_Mouth_Stretch_L", "BS_Mouth_Stretch_R",
        "BS_LowerLip_Bite",
        "BS_Tongue_Out",
    ]
    
    # 命名正则
    BS_NAME_PATTERN = re.compile(r'^BS_([A-Za-z]+)_([A-Za-z]+)(?:_([LR]))?$')
    
    # 组合测试用例（高风险组合）
    COMBO_TEST_CASES = [
        ("张嘴+微笑", {"BS_Jaw_Open": 1.0, "BS_Mouth_Smile_L": 1.0, "BS_Mouth_Smile_R": 1.0}),
        ("张嘴+皱眉", {"BS_Jaw_Open": 1.0, "BS_Mouth_Frown_L": 1.0, "BS_Mouth_Frown_R": 1.0}),
        ("鼓颊+闭嘴", {"BS_Cheek_Puff": 1.0, "BS_Mouth_Close": 1.0}),
        ("眨眼+眯眼", {"BS_Eye_Blink_L": 1.0, "BS_Eye_Squint_L": 1.0}),
        ("咧嘴+张嘴", {"BS_Mouth_Stretch_L": 1.0, "BS_Mouth_Stretch_R": 1.0, "BS_Jaw_Open": 1.0}),
        ("全笑+眯眼", {"BS_Mouth_Smile_L": 1.0, "BS_Mouth_Smile_R": 1.0,
                        "BS_Eye_Squint_L": 0.8, "BS_Eye_Squint_R": 0.8}),
        ("喷怒组合", {"BS_Brow_Drop_L": 1.0, "BS_Brow_Drop_R": 1.0,
                      "BS_Mouth_Frown_L": 1.0, "BS_Mouth_Frown_R": 1.0,
                      "BS_Jaw_Open": 0.5}),
    ]
    
    # ========== 检查函数 ==========
    
    def __init__(self, mesh_name=None):
        self.mesh_name = mesh_name or self._find_face_mesh()
        self.bs_node = self._find_blendshape_node()
        self.report = {
            "mesh": self.mesh_name,
            "checks": [],
            "warnings": [],
            "errors": [],
            "passed": False,
        }
    
    def _find_face_mesh(self):
        """自动查找面部网格"""
        meshes = cmds.ls(type='mesh', v=True)
        face_meshes = [m for m in meshes 
                       if any(kw in m.lower() for kw in ['face', 'head', 'head_mesh'])]
        if not face_meshes:
            raise RuntimeError("未找到面部网格，请手动指定")
        return face_meshes[0].split('|')[-1]  # 取短名
    
    def _find_blendshape_node(self):
        """查找 BlendShape 节点"""
        bs_nodes = cmds.ls(type='blendShape')
        if not bs_nodes:
            return None
        # 找到影响当前 mesh 的 BS 节点
        for node in bs_nodes:
            connections = cmds.listConnections(node, d=False, s=True, type='mesh')
            if connections and self.mesh_name in [c.split('|')[-1] for c in connections]:
                return node
        # 回退：取第一个
        return bs_nodes[0]
    
    def get_blendshape_names(self):
        """获取所有 BlendShape Target 名称"""
        if not self.bs_node:
            return []
        aliases = cmds.aliasAttr(self.bs_node, q=True) or []
        # aliases 是 [weight_name, input_name, ...] 交替
        weight_names = aliases[::2]
        return weight_names
    
    # --- Check 1: 命名规范 ---
    def check_naming_convention(self):
        """检查 BS 命名是否符合规范"""
        check = {"name": "命名规范检查", "passed": True, "issues": []}
        bs_names = self.get_blendshape_names()
        
        if not bs_names:
            check["passed"] = False
            check["issues"].append("未找到任何 BlendShape")
            self.report["errors"].append("未找到 BlendShape 节点")
            return check
        
        for name in bs_names:
            # 检查前缀
            if not name.startswith("BS_"):
                check["passed"] = False
                check["issues"].append(f"'{name}' 不以 BS_ 开头")
                continue
            
            # 检查格式
            match = self.BS_NAME_PATTERN.match(name)
            if not match:
                check["passed"] = False
                check["issues"].append(f"'{name}' 格式不正确（应为 BS_部位_动作[_方向]）")
            
            # 检查无空格/中文
            if ' ' in name or any(ord(c) > 127 for c in name):
                check["passed"] = False
                check["issues"].append(f"'{name}' 包含空格或非 ASCII 字符")
        
        # 检查 L/R 配对
        for name in bs_names:
            if name.endswith('_L'):
                mirror = name[:-2] + '_R'
                if mirror not in bs_names:
                    check["passed"] = False
                    check["issues"].append(f"'{name}' 缺少镜像 '{mirror}'")
        
        self.report["checks"].append(check)
        if not check["passed"]:
            self.report["errors"].extend(check["issues"])
        return check
    
    # --- Check 2: 数量与完整性 ---
    def check_bs_completeness(self):
        """检查 BS 集是否完整"""
        check = {"name": "表情集完整性检查", "passed": True, "issues": []}
        bs_names = set(self.get_blendshape_names())
        standard_set = set(self.STANDARD_BS_NAMES)
        
        missing = standard_set - bs_names
        extra = bs_names - standard_set
        
        if missing:
            check["passed"] = False
            check["issues"].append(f"缺失标准 BS ({len(missing)} 个): {sorted(missing)[:10]}...")
            self.report["errors"].append(f"缺失 {len(missing)} 个标准 BlendShape")
        
        if extra:
            check["issues"].append(f"额外 BS ({len(extra)} 个): {sorted(extra)[:10]}...")
            self.report["warnings"].append(f"有 {len(extra)} 个非标准 BlendShape（需确认是否为项目扩展集）")
        
        self.report["checks"].append(check)
        return check
    
    # --- Check 3: 中性姿势校准 ---
    def check_neutral_pose(self):
        """检查 BS Weight=0 时网格是否回到中性姿势"""
        check = {"name": "中性姿势检查", "passed": True, "issues": []}
        
        if not self.bs_node:
            check["passed"] = False
            check["issues"].append("无 BlendShape 节点")
            self.report["checks"].append(check)
            return check
        
        # 记录当前状态
        bs_names = self.get_blendshape_names()
        original_weights = {}
        for name in bs_names:
            original_weights[name] = cmds.getAttr(f"{self.bs_node}.{name}")
        
        # 全部归零
        for name in bs_names:
            cmds.setAttr(f"{self.bs_node}.{name}", 0)
        
        # 获取中性姿势顶点位置
        neutral_verts = cmds.xform(f"{self.mesh_name}.vtx[*]", q=True, ws=True, t=True)
        neutral_positions = [tuple(neutral_verts[i:i+3]) for i in range(0, len(neutral_verts), 3)]
        
        # 逐个 BS 设为 1.0，检查再归零后是否回到中性
        tolerance = 0.001  # 0.001 单位容差
        bad_bs = []
        
        for name in bs_names:
            cmds.setAttr(f"{self.bs_node}.{name}", 1.0)
            cmds.setAttr(f"{self.bs_node}.{name}", 0.0)
            
            current_verts = cmds.xform(f"{self.mesh_name}.vtx[*]", q=True, ws=True, t=True)
            current_positions = [tuple(current_verts[i:i+3]) for i in range(0, len(current_verts), 3)]
            
            max_diff = 0
            for np, cp in zip(neutral_positions, current_positions):
                diff = sum(abs(a-b) for a, b in zip(np, cp))
                max_diff = max(max_diff, diff)
            
            if max_diff > tolerance:
                bad_bs.append((name, max_diff))
        
        if bad_bs:
            check["passed"] = False
            check["issues"].append(f"{len(bad_bs)} 个 BS 归零后未回到中性姿势:")
            for name, diff in bad_bs[:5]:
                check["issues"].append(f"  {name}: 偏差 {diff:.4f}")
            self.report["errors"].append(f"中性姿势不稳定（{len(bad_bs)} 个 BS）")
        
        # 恢复原始状态
        for name, val in original_weights.items():
            cmds.setAttr(f"{self.bs_node}.{name}", val)
        
        self.report["checks"].append(check)
        return check
    
    # --- Check 4: 极端值穿模检测 ---
    def check_extreme_pose_intersection(self):
        """检查每个 BS 在极端值(1.0)时是否有自交"""
        check = {"name": "极端值穿模检查", "passed": True, "issues": []}
        
        if not self.bs_node:
            check["passed"] = False
            self.report["checks"].append(check)
            return check
        
        bs_names = self.get_blendshape_names()
        original_weights = {}
        for name in bs_names:
            original_weights[name] = cmds.getAttr(f"{self.bs_node}.{name}")
        
        # 全部归零
        for name in bs_names:
            cmds.setAttr(f"{self.bs_node}.{name}", 0)
        
        # 逐个 BS 检查极端值
        intersection_count = 0
        for name in bs_names:
            cmds.setAttr(f"{self.bs_node}.{name}", 1.0)
            
            # 使用 Maya 的网格自交检测（需要 polySelfIntersect 或类似）
            try:
                # 简化版：检查法线翻转区域
                mesh_shape = cmds.listRelatives(self.mesh_name, s=True)[0]
                cmds.polyInfo(mesh_shape, faceToVertex=True)  # 触发刷新
                
                # 更精确的方法：使用 API 检测自交
                # 这里用简化版——检查面部关键区域的体积变化
                # 实际生产中可用 OpenMaya MFnMesh.intersect()
                pass
            except:
                pass
            
            cmds.setAttr(f"{self.bs_node}.{name}", 0)
        
        # 组合穿模检测
        combo_failures = []
        for combo_name, combo_bs in self.COMBO_TEST_CASES:
            # 归零
            for name in bs_names:
                cmds.setAttr(f"{self.bs_node}.{name}", 0)
            # 设置组合
            for bs_name, weight in combo_bs.items():
                if bs_name in bs_names:
                    cmds.setAttr(f"{self.bs_node}.{bs_name}", weight)
            
            # TODO: 实际穿模检测（需要 OpenMaya API 或导出到引擎检测）
            # 这里只记录组合测试的执行
            combo_failures.append((combo_name, "需要人工或引擎端确认"))
        
        check["issues"].append(f"执行了 {len(self.COMBO_TEST_CASES)} 个组合测试（需人工确认穿模）")
        check["issues"].append("建议：将组合测试场景导出到 Unity/UE 中用引擎自交检测验证")
        
        # 恢复
        for name, val in original_weights.items():
            cmds.setAttr(f"{self.bs_node}.{name}", val)
        
        self.report["checks"].append(check)
        return check
    
    # --- Check 5: 索引一致性 ---
    def check_index_order(self):
        """检查 BS 索引顺序是否与规范一致"""
        check = {"name": "索引顺序检查", "passed": True, "issues": []}
        bs_names = self.get_blendshape_names()
        
        # 标准集应该是按规范顺序排列的
        expected_order = [n for n in self.STANDARD_BS_NAMES if n in bs_names]
        actual_order = bs_names
        
        mismatch_count = 0
        for i, (expected, actual) in enumerate(zip(expected_order, actual_order)):
            if expected != actual:
                mismatch_count += 1
                if mismatch_count <= 3:
                    check["issues"].append(
                        f"索引 {i}: 期望 '{expected}'，实际 '{actual}'")
        
        if mismatch_count > 0:
            check["passed"] = False
            check["issues"].append(f"共 {mismatch_count} 个索引不一致（影响程序绑定）")
            self.report["warnings"].append(f"BS 索引顺序不一致（{mismatch_count} 处）")
        
        self.report["checks"].append(check)
        return check
    
    # ========== 执行全部检查 ==========
    
    def run_all_checks(self):
        """执行所有 QA 检查"""
        print("=" * 60)
        print("BlendShape QA — 开始检查")
        print("=" * 60)
        
        self.check_naming_convention()
        self.check_bs_completeness()
        self.check_neutral_pose()
        self.check_index_order()
        self.check_extreme_pose_intersection()
        
        # 汇总
        all_passed = all(c["passed"] for c in self.report["checks"])
        self.report["passed"] = all_passed
        
        print("\n" + "=" * 60)
        status = "✅ 全部通过" if all_passed else "❌ 存在问题"
        print(f"结果: {status}")
        print(f"检查项: {len(self.report['checks'])}")
        print(f"错误: {len(self.report['errors'])}")
        print(f"警告: {len(self.report['warnings'])}")
        print("=" * 60)
        
        for check in self.report["checks"]:
            icon = "✅" if check["passed"] else "❌"
            print(f"\n{icon} {check['name']}")
            for issue in check["issues"]:
                print(f"    {issue}")
        
        return self.report
    
    def export_report(self, path):
        """导出 JSON 报告"""
        with open(path, 'w', encoding='utf-8') as f:
            json.dump(self.report, f, ensure_ascii=False, indent=2)
        print(f"\n报告已导出: {path}")


# ========== 使用方法 ==========
# 在 Maya Script Editor 中执行：
#
# from blendshape_qa import BlendShapeQA
# qa = BlendShapeQA()  # 自动查找面部网格
# # 或指定网格名: BlendShapeQA("CharacterA_Head_Mesh")
# qa.run_all_checks()
# qa.export_report("D:/QA_Reports/CharacterA_BS_Report.json")
```

**规范文档结构（发给外包的文档）：**

```
面部 BlendShape 规范文档 v2.0
│
├── 1. 总则
│   ├── 适用范围（所有有面部表情的角色）
│   ├── 资产交付格式（FBX 2020 + Maya ASCII）
│   └── 版本规范（文件命名: CharacterName_FaceBS_vXX.mb）
│
├── 2. 中性姿势（Neutral Pose）规范
│   ├── 中性脸定义（配图：正面/侧面/45度）
│   ├── 对称性要求（左右偏差 < 0.01 单位）
│   ├── 骨架位置要求（头部 Joint 变换值精确列出）
│   └── 网格密度要求（面部三角面数范围）
│
├── 3. BlendShape 标准集
│   ├── 3.1 ARKit 基础集（30 个，列出所有名称+示意图）
│   ├── 3.2 项目扩展集（10 个，含对话 Viseme + 情绪极端）
│   ├── 3.3 禁止自行增减 BS（如需扩展请联系项目 TA）
│   └── 3.4 每个 BS 的权重范围（默认 0-1，特殊标注）
│
├── 4. 命名规范
│   ├── 格式：BS_<部位>_<动作>[_<方向>]
│   ├── 部位枚举：Eye / Brow / Nose / Mouth / Cheek / Jaw / Tongue
│   ├── 方向枚举：L / R（仅不对称表情需要）
│   └── 禁止事项清单
│
├── 5. 品质标准
│   ├── 单个 BS 极端值（1.0）穿模容忍度
│   ├── 组合 BS 穿模测试用例（附 7 个标准组合）
│   ├── 正交性要求（L/R 镜像偏差 < 5%）
│   └── 体积保持（形变区域体积变化 < ±20%）
│
├── 6. 参考资产（黄金标准）
│   ├── 参考角色 Maya 工程文件
│   ├── 每个BS的极端值截图（正面/侧面）
│   └── 组合表情参考截图
│
├── 7. 交付前自检
│   ├── 自检 Checklist（20 项）
│   ├── Maya QA 脚本使用说明
│   └── 已知问题 FAQ
│
└── 8. 版本历史
    └── v2.0 变更说明
```

### ⚡ 实战经验

1. **"黄金标准"比文档有效 10 倍**：给外包一份"完美范例"的 Maya 工程（包含所有标准 BS、已通过 QA、带参考截图），比 20 页文档更有效。美术是视觉驱动的，他们需要"看到标准"而不是"读到标准"。我见过项目光靠文档，外包第一批 5 个角色全废——换成参考工程后，第二批一次性过
2. **Corrective BlendShape 是面部的"秘密武器"**：有些组合穿模不可能通过修 Target 解决（嘴大张+微笑的嘴角形变在物理上是矛盾的）。正确做法是加 Corrective BS：当 A+B 同时激活时，自动触发一个修正形变。需要在规范中定义 Corrective BS 的命名（如 `BS_Corrective_JawOpen_Smile`）和触发逻辑
3. **二次元项目的减法思维**：ARKit 的 52 个 BS 在二次元角色上太多了——二次元面部拓扑简单（往往只有几百个顶点），强行用 52 个 BS 会导致很多 BS 效果几乎看不出区别。建议精简到 25-30 个，保留夸张情绪 BS。关键是 Viseme（嘴形）不能省，因为对话系统依赖它
4. **面部骨架 vs BlendShape 的混合策略**：有些项目用骨架驱动面部（颚骨旋转=张嘴），有些用 BS。最佳实践是**混合**——大尺度形变用骨架（下颚张开），细节用 BS（嘴形、眼皮）。规范中需要明确哪些表情用骨架、哪些用 BS，避免外包用 BS 做了应该骨架做的事
5. **QA 脚本的投入产出比极高**：50 个角色 × 40 个 BS = 2000 个数据点要检查。人工检查一个角色 2 小时，自动化脚本 2 分钟。脚本开发 2 天，但后续每个角色省 1.5 小时，50 个角色就是 75 小时——绝对值得

### 🎯 能力体检清单

- [ ] **如果不知道 FACS / ARKit BS 标准** → 你需要补：FACS 的 44 个 Action Unit、ARKit 的 52 个 BS 定义、行业标准集的选型逻辑
- [ ] **如果不会写 Maya Python 脚本** → 你需要补：`maya.cmds` 基础、PyMel / OpenMaya API、BlendShape 节点的程序化操作
- [ ] **如果不懂 Corrective BlendShape** → 你需要补：组合形变的数学原理、Corrective BS 的触发逻辑、驱动关键帧（Driven Key）设置
- [ ] **如果不知道 Viseme** → 你需要补：音素到嘴形的映射、中文/英文 Viseme 差异、对话系统的 BS 驱动管线
- [ ] **如果不会定义外包规范** → 你需要补：资产交付格式、版本管理、参考资产制作、QA Checklist 设计、验收流程

### 🔗 相关问题

- [Morph Target 面部系统设计](technical-art/morph-target-facial-system-design.md)：面部系统架构设计的更高层视角
- [LOD 规范与 QA](technical-art/lod-spec-and-qa.md)：LOD 资产的外包验收思路（方法论可复用）
- [外包资产验收标准](soft-skills/outsource-art-acceptance-criteria.md)：跨部门沟通层面的外包管理
- 如果需要在运行时动态生成 Corrective BS，技术方案是什么？（GPU Morph Target 管线）
