---
title: "Houdini 程序化城市建筑生成：如何用一套规则自动生成风格统一但形态各异的建筑群？"
category: "pipeline"
level: 3
tags: ["Houdini", "程序化建模", "城市生成", "Python", "PDG", "游戏管线"]
hint: "核心是模块化组件 + 规则驱动变体——将建筑拆为地块→体块→立面→细节四级，每级用 HDA 控制参数变体"
related: ["pipeline/houdini-vegetation-scatter", "pipeline/houdini-terrain-river-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们在做一个开放世界城市项目。美术手动建模太慢——100 栋楼花了 3 个月。希望用 Houdini 搭一套程序化建筑生成管线：输入地块数据和风格参数，自动输出不同形态但风格统一的建筑模型。你会怎么设计这套管线？」

### ✅ 核心要点

1. **分而治之**：将建筑拆解为地块（Lot）→ 体块（Massing）→ 立面（Facade）→ 细节（Details）四级流水线
2. **规则驱动变体**：用参数化规则控制建筑形态变化（高度、开间、材质、屋顶类型），保证风格统一但形态各异
3. **HDA（Houdini Digital Asset）封装**：每级封装为 HDA，策划/美术只需调参数，不需要懂 Houdini
4. **PDG/TOPs 批量调度**：用 PDG（Procedural Dependency Graph）并行处理上百个地块，自动输出到引擎
5. **引擎侧集成**：通过 Houdini Engine for Unreal/Unity，在引擎内实时预览和参数微调

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：100+ 栋风格统一但形态各异的建筑，自动生成，可直接导入引擎
                ↑
倒推1：需要每栋建筑「看起来不同但风格一致」→ 规则系统：变体参数 + 共享风格模板
倒推2：需要自动生成 → 程序化建模管线（Houdini SOP/VOP 网络）
倒推3：需要批量处理大量地块 → PDG/TOPs 并行调度
倒推4：需要美术可控 → HDA 封装参数面板 + 风格 Preset
倒推5：需要导入引擎 → Houdini Engine 插件（Unreal/Unity）
倒推6：需要可迭代 → 参数变更是非破坏性的，随时重新生成
```

#### 知识点拆解（倒推树）

```
Houdini 程序化建筑生成
├── 第一级：地块分配（Lot Subdivision）
│   ├── OSM/FBX 地块数据导入（城市路网 → 地块多边形）
│   ├── 地块分类（商业/住宅/工业，按面积/位置自动判定）
│   ├── 地块形状规整（凹多边形分割、狭长地块处理）
│   └── Setback 退线（前后院、侧院间距规则）
├── 第二级：体块生成（Massing）
│   ├── 高度规则（按地块类型 + 容积率参数计算楼层数）
│   ├── 体块变形（阶梯式收分、旋转、L形/U形组合）
│   ├── 屋顶生成（平顶/坡顶/穹顶，按风格模板选择）
│   └── 楼层切分（每层高度可变，底层可做挑高）
├── 第三级：立面生成（Facade）
│   ├── 开间系统（Window/Door Grid 布局算法）
│   │   ├── 均匀分割 vs 随机变体
│   │   └── 转角特殊处理（转角窗 vs 实墙）
│   ├── 窗户模块库（不同风格：现代玻璃/古典拱窗/工业条窗）
│   ├── 材质分配（按楼层/朝向/风格分配材质变体）
│   └── 阳台/空调外机/广告牌等附属物（按概率 + 规则分布）
├── 第四级：细节层（Details）
│   ├── 檐口/腰线/勒脚（建筑轮廓装饰）
│   ├── 街道设施（路灯、垃圾桶、行道树，沿地块边缘分布）
│   └── 材质 ID 烘焙（输出 Vertex Color / UV2 作为材质选择通道）
├── PDG/TOPs 批量调度
│   ├── 地块分批（按区域/LOD 层级分组）
│   ├── 并行 Cook（多 Worker 同时处理不同地块）
│   ├── 输出队列（FBX/USD → 引擎 Asset）
│   └── 失败重试与日志
├── HDA 封装
│   ├── 参数面板设计（风格选择、密度、高度范围、随机种子）
│   ├── Preset 系统（现代都市/民国老街/赛博朋克 预设）
│   └── 输出模板控制（LOD 生成、碰撞体生成）
└── 引擎集成
    ├── Houdini Engine for Unreal（HDA 作为 Actor）
    ├── Houdini Engine for Unity（HDA 作为 Component）
    └── 自动生成 LOD + 碰撞体 + 材质分配
```

#### 代码实现

**Houdini Python SOP — 地块分类与参数分配：**

```python
# This code lives inside a Python SOP node in Houdini
import hou
import random

node = hou.pwd()
geo = node.geometry()

# 参数获取
style_template = node.evalParm("style_template")    # 风格模板
density = node.evalParm("density")                   # 建筑密度
height_min = node.evalParm("height_min")             # 最小高度
height_max = node.evalParm("height_max")             # 最大高度
seed = node.evalParm("seed")                          # 随机种子

random.seed(seed)

# 遍历每个地块图元
for prim in geo.prims():
    # 计算地块面积
    bbox = prim.boundingBox()
    area = (bbox.sizevec()[0]) * (bbox.sizevec()[2])

    # 按面积分类建筑类型
    if area < 200:
        bldg_type = "small_residential"
        height_range = (height_min, height_min + (height_max - height_min) * 0.4)
    elif area < 800:
        bldg_type = "medium_residential"
        height_range = (height_min + (height_max - height_min) * 0.3,
                        height_min + (height_max - height_min) * 0.7)
    else:
        bldg_type = "commercial"
        height_range = (height_min + (height_max - height_min) * 0.5, height_max)

    # 分配属性
    prim.setAttribValue("bldg_type", bldg_type)
    prim.setAttribValue("target_height", random.uniform(*height_range))
    prim.setAttribValue("style", style_template)
    prim.setAttribValue("floor_height", random.choice([3.0, 3.3, 3.6, 4.0]))
    prim.setAttribValue("setback_front", random.uniform(2.0, 6.0))
    prim.setAttribValue("setback_side", random.uniform(0.0, 3.0))
    prim.setAttribValue("variation_seed", random.randint(0, 99999))
```

**PDG/TOPs 批量处理（Python 脚本片段）：**

```python
# PDG Wedge TOP 节点的脚本
# 为每个地块生成独立的处理任务

import pdg

# 获取所有地块数据
lot_data = pdg.IOManager.readJsonFile("lot_data.json")
lots = lot_data["lots"]

for lot in lots:
    # 创建 Wedge 节点变体
    task = graph.addTask("lot_processor")
    task.parameters["lot_id"] = lot["id"]
    task.parameters["lot_polygon"] = lot["polygon"]
    task.parameters["style"] = lot["style"]
    task.parameters["output_path"] = f"buildings/building_{lot['id']}.fbx"
    task.parameters["lod_levels"] = 3
    task.parameters["generate_collision"] = True

# PDG 自动调度并行 Cook
# 可配置 Worker 数量（如 8 个 Houdini 实例并行）
graph.cook(max_workers=8)
```

**Facade 立面生成（VEX 代码片段）：**

```c
// 在 Houdini 的 Wrangle 节点中处理立面窗户分布
// 输入：建筑墙面多边形（已展平到 UV 空间）
// 输出：每个窗户的位置、尺寸、类型

int primnum = i@primnum;
float wall_width = detail(1, "wall_width");     // 墙面宽度
float wall_height = detail(1, "wall_height");   // 墙面高度
float floor_height = detail(1, "floor_height"); // 每层高度
float window_width = detail(1, "window_width"); // 标准窗户宽度
float window_height = detail(1, "window_height");

int num_floors = int(wall_height / floor_height);
int num_windows_per_floor = int(wall_width / window_width);

int seed = i@variation_seed;

// 为每层每窗位生成实例
for (int f = 0; f < num_floors; f++)
{
    for (int w = 0; w < num_windows_per_floor; w++)
    {
        float pos_x = (w + 0.5) * window_width;
        float pos_y = (f + 0.5) * floor_height;

        // 随机窗户变体（80% 普通窗，10% 阳台窗，10% 实墙）
        float rand_val = rand(seed + f * 100 + w * 37);
        int window_type = 0; // 0=普通, 1=阳台, 2=实墙
        if (rand_val > 0.9) window_type = 1;
        else if (rand_val > 0.8) window_type = 2;

        // 输出实例点
        int pt = addpoint(0, set(pos_x, pos_y, 0));
        setpointattrib(0, "window_type", pt, window_type);
        setpointattrib(0, "window_size", pt, set(window_width, window_height, 0));
        setpointattrib(0, "floor", pt, f);
    }
}
```

**管线架构表：**

| 管线阶段 | Houdini 节点类型 | 输入 | 输出 | 耗时（100 栋） |
|----------|-----------------|------|------|----------------|
| 地块分配 | Python SOP | OSM/FBX 地块 | 分类后地块 + 属性 | 5s |
| 体块生成 | VEX Wrangle + For-Each | 地块 + 属性 | 建筑体块网格 | 45s |
| 立面生成 | Copy to Points + VEX | 体块 + 窗户资产 | 带窗户的立面 | 120s |
| 细节层 | Scatter + Copy | 立面 + 零件资产 | 完整建筑 | 60s |
| LOD 生成 | PolyReduce | 完整建筑 | LOD1-3 | 30s |
| 导出 | ROP FBX/USD | 建筑模型 | 引擎可用资产 | 40s |
| **总计** | | | | **~5 min** |

### ⚡ 实战经验

- **先做 3 栋再放大到 100 栋**：先手工调好 3 种典型建筑（商业/住宅/工业）的参数组合，确认风格正确后再批量生成。直接上 100 栋调试参数是灾难
- **风格模板是一等公民**：不要在 HDA 里塞 50 个参数让美术选。做好 3-5 个 Preset（现代都市/民国老街/赛博朋克），Preset 内部自动联动 50 个子参数
- **窗户库是核心资产**：建筑质量 70% 取决于窗户模块库的丰富度。准备 15-20 个窗户变体，规则系统只是选哪个的问题
- **LOD 自动生成时机**：不要在 Houdini 里做 LOD，用引擎的 Mesh Auto-LOD。Houdini 出来的顶点法线引擎再算一遍更准
- **USD 是未来**：如果项目支持，优先输出 USD 格式。FBX 材质信息丢失严重，USD 保留完整 Material Binding
- **实战坑点**：Copy to Points 的朝向一定要用 `@orient`（四元数）控制，不要用 `@N` + `@up`——在斜面地块上会出现窗户朝向错误

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道从哪里开始拆解需求 | 程序化建模方法论 | 学 Modular Environment Design + Shape Grammar |
| For-Each 循环不会用 | Houdini 循环节点 | 学 For-Each Subnetwork + Metadata 用法 |
| PDG/TOPs 跑不起来 | PDG 调度配置 | 学 TOP Network + Scheduler 配置 |
| HDA 参数面板不好用 | HDA Type Properties | 学 Parameter Interface 设计 + Preset 系统 |
| Houdini Engine 崩溃频繁 | Houdini Engine 限制 | 学 HDA 性能优化（减少 Cook 时间） |
| 建筑看起来千篇一律 | 随机性与多样性 | 学噪声场 + 分层随机（种子分层，区域变体） |

### 🔗 相关问题

- 如何让程序化建筑和手工地标建筑无缝衔接？（提示：地标手工建模 + 周边地块程序化填充）
- 程序化生成如何配合光照烘焙？（提示：输出 UV2 + 自动生成 Lightmap UV）
- 如果要做「可破坏建筑」，程序化模型需要额外输出什么数据？（提示：分块破碎 + Chunk ID）
- Houdini 和 Unity 的 DOTS ECS 如何配合处理大规模建筑实例化？
