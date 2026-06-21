---
title: "Houdini 批量植被撒点：如何程序化生成 10km² 的生态分布？"
category: "pipeline"
level: 3
tags: ["Houdini", "程序化生成", "植被分布", "PCG", "工具管线"]
hint: "核心考点：高度图驱动分布 + 坡度/海拔/密度遮罩 + LOD 分组 + 引擎对接"
related: ["pipeline/material-template-system", "optimization/gpu-bandwidth-mobile"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们需要做一个开放世界地图，大约 10km²。植被需要根据地形特征分布——低海拔是草地，山坡是混交林，高海拔是针叶林，水边是芦苇。如果靠美术手摆，一个场景要几周。你会怎么用 Houdini 做一套程序化植被工具？

这是网易、腾讯、米哈游等开放世界项目的 TA 工具管线核心面试题。考察的不只是"Houdini 会不会用"，而是**程序化思维 + 生态知识 + 引擎工作流**的综合能力。

### ✅ 核心要点

1. **地形分析驱动**：海拔、坡度、坡向、曲率、湿度——每个地形特征对应一个遮罩
2. **生态规则建模**：真实植被分布不是随机的，需要生态学规则（阳坡针叶、阴坡阔叶等）
3. **Scatter 系统设计**：基于密度场 + 遮罩的 Point Generate，不是简单随机撒点
4. **LOD 与分组输出**：生成时自动分配 LOD 层级和 Imposter 距离，配合引擎的 Instancing
5. **引擎对接管线**：Houdini Engine for Unity/UE 或自定义 JSON/Binary 导出

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
期望效果：10km² 地图，植被分布自然，不同地形有不同植被类型
         ↓
程序化生成流程：
  Step 1：导入地形数据（Heightmap + Splatmap）
     ↓
  Step 2：地形分析
  ├── 海拔分层（低/中/高）
  ├── 坡度计算（平地 vs 陡坡）
  ├── 坡向计算（阳坡 vs 阴坡）
  ├── 水流模拟（河流、湖泊周围）
  └── 曲率分析（山脊 vs 山谷）
     ↓
  Step 3：生态规则映射
  ├── 规则表：海拔 + 坡度 + 坡向 → 植被类型
  ├── 密度规则：不同植被的种植密度
  └── 变异规则：同种植被的大小/旋转变化范围
     ↓
  Step 4：点云生成（Scatter）
  ├── 按 Vegetation Zone 分组 Scatter
  ├── 密度场控制每平方米数量
  └── 避让规则（道路、建筑、水面不撒点）
     ↓
  Step 5：实例属性赋值
  ├── 每个点：植被类型、Scale、Rotation、LOD Group
  └── 导出属性 → 引擎读取
     ↓
  Step 6：导出到引擎
  ├── 方案A：Houdini Engine（HDA）→ Unity/UE
  ├── 方案B：自定义 Point Cloud 格式（JSON/Binary）
  └── 方案C：直接生成场景文件（.unity/.umap）
```

#### 知识点拆解（倒推树）

```
程序化植被生成
├── 地形分析（VOP / VEX）
│   ├── Height Field 节点链
│   │   └── HF Remap / HF Noise / HF Pattern
│   ├── 坡度计算
│   │   └── VEX: slope = length(gradient(heightfield))
│   ├── 坡向计算
│   │   └── VEX: aspect = atan2(dz/dx, dz/dy)
│   └── 水体遮罩
│       └── 基于 HeightField 的流体模拟或手动绘制
│
├── 生态分布规则
│   ├── 规则引擎（Attribute Wrangle / Python SOP）
│   │   └── 多维条件判断：elevation × slope × aspect → vegType
│   ├── 噪声控制（使边界自然过渡）
│   │   └── Anti-Aliased Noise / Curl Noise 打破规则边界
│   └── 密度场（每平方米的植被数量）
│       └── 基于生态学：森林边缘密度高、纯林内部密度均匀
│
├── Scatter 系统
│   ├── Points from Volume / Scatter SOP
│   │   └── 密度属性驱动：density attribute × mask
│   ├── 避让系统
│   │   ├── 道路：Buffer Zone + 距离衰减
│   │   ├── 建筑：Bounding Box 碰撞剔除
│   │   └── 水面：Height < waterLevel → 删除
│   └── 变异属性（Scale / Rotation / Lean）
│       └── 伪随机：rand(@ptnum) * variation_range
│
├── LOD 与性能
│   ├── 距离分区
│   │   ├── Near (<50m)：LOD0，独立模型
│   │   ├── Mid (50-200m)：LOD1，简化模型 + GPU Instancing
│   │   └── Far (>200m)：Imposter / Billboard
│   ├── 簇化（Clustering）
│   │   └── 远距离将多个点合并为一个 Cluster Mesh
│   └── 遮挡剔除预热
│       └── 生成时标记可能被山体遮挡的区域
│
└── 引擎对接
    ├── Houdini Engine for Unity (HDA)
    │   ├── 优点：实时参数调节、与地形系统联动
    │   └── 缺点：大量点时性能慢、需要 Houdini Engine 许可
    ├── 自定义 Point Cloud 导出
    │   ├── VEX 将属性序列化为 JSON
    │   └── Unity/UE 脚本读取并 Instancing
    └── 离线烘焙
        └── 直接输出引擎可读的场景/预制体文件
```

#### 代码实现

**Houdini VEX — 地形分析与生态规则：**

```c
// Attribute Wrangle (Detail 模式) — 地形分析
// 输入：HeightField (height volume)

// 获取当前点的地形参数
float elev = @P.y;                          // 海拔
vector grad = getgradient(0, "height", @P.x, @P.z);
float slope = length(grad);                  // 坡度 (0=平地, 1=垂直)
float aspect = atan2(grad.x, grad.z);        // 坡向 (-π ~ π)

// 写入属性供后续 Scatter 使用
@elevation = fit(elev, ch("min_h"), ch("max_h"), 0, 1);
@slope     = clamp(slope * ch("slope_scale"), 0, 1);
@aspect    = aspect;

// === 生态规则映射 ===
// 定义植被分布规则
// 返回值：0=草地, 1=灌木, 2=阔叶林, 3=针叶林, 4=芦苇

float northness = (cos(aspect) + 1) * 0.5;   // 0=南坡, 1=北坡

int determineVegType(float elev, slope, northness; float waterLevel) {
    // 水边 (海拔接近水位)
    if (elev < waterLevel + 0.5 && slope < 0.1) {
        return 4;  // 芦苇
    }
    
    // 低海拔平地 → 草地
    if (elev < 0.2 && slope < 0.15) {
        return 0;  // 草地
    }
    
    // 中低海拔 → 阔叶林（阳坡密度更高）
    if (elev < 0.5 && slope < 0.5) {
        return 2;  // 阔叶林
    }
    
    // 中高海拔 → 针叶林（北坡为主）
    if (elev >= 0.3 && elev < 0.8 && northness > 0.4) {
        return 3;  // 针叶林
    }
    
    // 中海拔陡坡 → 灌木
    if (slope > 0.5) {
        return 1;  // 灌木
    }
    
    // 高海拔 → 无植被（裸岩）
    if (elev > 0.8) {
        return -1; // 无
    }
    
    return 0; // 默认草地
}

@vegType = determineVegType(@elevation, @slope, northness, ch("water_level"));

// 密度计算：不同植被类型的密度
float densityLookup[] = {0.5, 0.15, 0.08, 0.06, 0.3}; // 草/灌木/阔叶/针叶/芦苇
@vegDensity = densityLookup[@vegType + 1]; // +1 因为数组从0开始

// 加入噪声打破均匀感
float n = anoise(@P * ch("noise_freq"));
@vegDensity *= fit(n, 0, 1, 0.5, 1.5);
```

**Unity 侧读取脚本 — Point Cloud Instancing：**

```csharp
using UnityEngine;
using System.IO;
using System.Collections.Generic;

// Houdini 导出的植被点云 JSON
[System.Serializable]
struct VegPoint {
    public float x, y, z;
    public float rx, ry, rz;  // rotation
    public float sx, sy, sz;  // scale
    public int   type;         // 0=grass, 1=bush, 2=broadleaf, 3=conifer, 4=reed
    public int   lodGroup;     // 0=near, 1=mid, 2=far
}

public class VegetationImporter : MonoBehaviour {
    public GameObject[] grassPrefabs;
    public GameObject[] bushPrefabs;
    public GameObject[] broadleafPrefabs;
    public GameObject[] coniferPrefabs;
    public GameObject[] reedPrefabs;
    
    public void ImportFromHoudini(string jsonPath) {
        string json = File.ReadAllText(jsonPath);
        VegPoint[] points = JsonHelper.FromJson<VegPoint>(json);
        
        // 按类型分组
        var groups = new Dictionary<int, List<VegPoint>>();
        foreach (var p in points) {
            if (!groups.ContainsKey(p.type)) groups[p.type] = new List<VegPoint>();
            groups[p.type].Add(p);
        }
        
        // 每种植被创建一个 ParentGO
        foreach (var kv in groups) {
            GameObject parent = new GameObject($"Vegetation_{kv.Key}");
            
            // 使用 GPU Instancer 批量渲染
            var instances = new List<Matrix4x4>();
            foreach (var p in kv.Value) {
                var pos = new Vector3(p.x, p.y, p.z);
                var rot = Quaternion.Euler(p.rx, p.ry, p.rz);
                var scl = new Vector3(p.sx, p.sy, p.sz);
                instances.Add(Matrix4x4.TRS(pos, rot, scl));
            }
            
            // 注册到 GPU Instancing 系统
            // 或使用 Graphics.RenderMeshInstanced
            var prefab = GetPrefab(kv.Key);
            var renderer = parent.AddComponent<VegetationGPUInstancer>();
            renderer.Initialize(prefab, instances);
        }
    }
    
    GameObject GetPrefab(int type) => type switch {
        0 => grassPrefabs[0],
        1 => bushPrefabs[0],
        2 => broadleafPrefabs[0],
        3 => coniferPrefabs[0],
        4 => reedPrefabs[0],
        _ => grassPrefabs[0],
    };
}
```

**Houdini Network 核心节点链：**

```
[HeightField Import] 
    → [HF Analyze: Slope/Aspect/Curvature]
    → [Attribute Wrangle: 生态规则]
    → [Scatter SOP: 密度驱动撒点]
    → [Attribute Wrangle: 变异属性]
    → [Avoidance: 道路/建筑避让]
    → [LOD Group: 距离分区]
    → [ROP Output: JSON/Point Cloud]
```

### ⚡ 实战经验

1. **不要一开始就追求"全自动"**：先做 70% 的程序化覆盖 + 30% 的美术手动调整。纯程序化的植被分布往往"太均匀"反而不自然，留出手绘覆盖通道
2. **密度噪声是质感的来源**：用 Curl Noise 或 Worley Noise 打破规则的边界，让针叶林和阔叶林的交界处呈现"渗透"效果，而不是干净的切割线
3. **导出格式要提前和程序约定**：10km² 的地图可能有几十万个植被点，JSON 会很大（100MB+）。用二进制格式（如自定义 Byte Stream）体积可缩小到 1/5，读取速度也快 10 倍
4. **HDA vs 离线导出的抉择**：HDA 适合开发期快速迭代，但发布前一定要切换到离线烘焙。HDA 在运行时需要 Houdini Engine 许可，且性能远不如直接读取

### 🎯 能力体检清单

- [ ] **如果不知道地形分析怎么做** → 你需要补：Houdini HeightField 系统、地形参数的数学定义（坡度/坡向/曲率）
- [ ] **如果不知道生态规则怎么定** → 你需要补：基础生态学知识（海拔分带、阴阳坡、植被演替）、与世界观美术沟通规则
- [ ] **如果不会做引擎对接** → 你需要补：Unity GPU Instancing API、引擎的植被渲染系统（如 Unity Terrain Detail）、Point Cloud 序列化
- [ ] **如果不知道 LOD 怎么配合** → 你需要补：LOD 系统、Imposter/ billboard 技术、距离分区的性能预算
- [ ] **如果不会做避让系统** → 你需要补：空间数据结构（BVH/Quadtree）、碰撞检测、引擎的 NavMesh/障碍物数据

### 🔗 相关问题

- 如何让程序化植被在不同季节呈现不同外观？（季节系统 + 植被 Shader）
- 大世界植被的内存预算怎么控制？（实例数据压缩 + LOD 策略）
- 如何在 UE5 中用 PCG 替代 Houdini？（Niagara PCG vs Houdini 对比）
