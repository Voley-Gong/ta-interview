---
title: "手游 Draw Call 从 500 降到 100：你的优化策略是什么？"
category: "optimization"
level: 2
tags: ["DrawCall", "合批", "SRP Batcher", "GPU Instancing", "性能优化"]
hint: "Draw Call 优化的本质是减少 CPU 提交次数——SRP Batcher、GPU Instancing、Static/Dynamic Batching 各自的适用条件是什么？"
related: ["optimization/gpu-bandwidth-mobile", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们的一款 MMO 手游，同屏角色多、特效密集，Draw Call 飙到了 500+，中低端机上帧率掉到 30fps 以下。Profiler 显示瓶颈在 CPU 的渲染提交。你接手后，会怎么把 Draw Call 降到 100 以内？

这是腾讯、网易、字节等大厂 TA 性能优化方向的高频面试题。考察点不仅仅是"知道合批技术"，而是能否系统化地定位瓶颈、制定优化方案、并量化效果。

### ✅ 核心要点

1. **先 Profile 再优化**：用 Frame Debugger / Render Doc / Profiler 精确定位 Draw Call 来源（角色？特效？UI？地形？）
2. **合批技术选型矩阵**：SRP Batcher > GPU Instancing > Static/Dynamic Batching，各有适用条件
3. **材质合并**：相同 Shader 不同材质实例无法合批——做材质模板化
4. **特效合批**：粒子系统是 Draw Call 大户——特效图集 + GPU Instancing 粒子
5. **LOD + 合批组合拳**：远距离用低面数 + 合批，近距离才用独立材质

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：Draw Call 从 500 → 100（降 80%）
     ↓
Step 1：Draw Call 构成分析
  ├── 角色（150 个 DC）→ 同 Shader 但不同材质参数
  ├── 特效（200 个 DC）→ 粒子系统各自独立
  ├── 场景（100 个 DC）→ 静态物体没合批
  ├── UI（30 个 DC）→ 图集不全
  └── 阴影/反射（20 个 DC）→ 多 Pass 开销
     ↓
Step 2：按收益排序优化
  特效（-120 DC）> 角色（-100 DC）> 场景（-80 DC）> UI（-20 DC）
     ↓
Step 3：技术方案匹配
  特效：粒子合批 + Effect Atlas
  角色：SRP Batcher + 材质模板
  场景：Static Batching + GPU Instancing
  UI：完善图集
```

#### 知识点拆解（倒推树）

```
Draw Call 优化
├── 合批技术体系（减少 SetPassCall）
│   ├── SRP Batcher
│   │   ├── 原理：相同 Shader + 不同材质参数 → 按 Shader 分组，只更新 CBUFFER
│   │   ├── 条件：URP/HDRP + Shader 兼容（CBUFFER_START/END）
│   │   ├── 效果：角色类场景可降 50-70% DC
│   │   └── 限制：不能有 Shader 变体爆炸、材质需要用 CBUFFER 声明
│   │
│   ├── GPU Instancing
│   │   ├── 原理：相同 Mesh + 相同 Material → 一个 Draw Call 画多个
│   │   ├── 条件：网格和材质完全相同（参数可不同用 instancing properties）
│   │   ├── 适用：植被、石头、重复道具、简单角色（换装少的 NPC）
│   │   └── 限制：不同 Mesh 不能合批、有蒙皮的角色需要特殊处理
│   │
│   ├── Static Batching
│   │   ├── 原理：静态物体在编辑时合并到大 Vertex Buffer
│   │   ├── 条件：标记 Static、不移动
│   │   ├── 适用：建筑、地形装饰物
│   │   └── 限制：增加内存（合并后的 VB），但不增加 DC
│   │
│   └── Dynamic Batching
│       ├── 原理：运行时 CPU 合并小网格
│       ├── 条件：顶点数 < 300、面数 < 900
│       └── 已基本被 SRP Batcher / GPU Instancing 取代
│
├── 材质优化
│   ├── 材质模板化：一套 Shader + 参数差异 → CBUFFER 驱动
│   ├── 贴图图集：多张小图合一张大图，减少材质切换
│   └── 变体控制：去掉不必要的 multi_compile / shader_feature
│
├── 粒子/特效优化
│   ├── Effect Atlas：所有特效贴图合图集
│   ├── 粒子合批：同图集的粒子可以合批
│   ├── GPU 粒子：Compute Shader 模拟，一个 Draw Call 渲染上万粒子
│   └── LOD 粒子：远距离简化粒子发射数
│
└── 引擎层面
    ├── Camera.Layer：合理分层，减少不必要的渲染
    ├── Occlusion Culling：遮挡剔除减少不可见物体
    └── Shadow Pass：减少实时阴影的物体数量
```

#### 代码实现

**SRP Batcher 兼容的 Shader 检查清单：**

```hlsl
// ✅ SRP Batcher 兼容写法
// 所有材质参数必须在 CBUFFER 中声明
CBUFFER_START(UnityPerMaterial)
    float4 _BaseMap_ST;
    float4 _BaseColor;
    float  _Cutoff;
    float  _Smoothness;
CBUFFER_END

// ❌ 不兼容：在 CBUFFER 外声明材质参数
// float4 _BaseColor; // 这会导致 SRP Batcher 失效！

// ❌ 不兼容：使用了 Uniform 数组
// float4 _Colors[10]; // SRP Batcher 不支持
```

**GPU Instancing 的条件与启用：**

```csharp
// 在材质面板勾选 "Enable GPU Instancing"
// 或在代码中设置：
Material mat = new Material(shader);
mat.enableInstancing = true;

// GPU Instancing 需要 Mesh + Material 完全一致
// 角色换装导致 Mesh 不同 → 无法 Instancing
// 解决方案：使用 MaterialPropertyBlock 传递差异参数

// ✅ 正确做法：NPC 共享基础 Mesh，用 PropertyBlock 区分颜色
MaterialPropertyBlock mpb = new MaterialPropertyBlock();
mpb.SetColor("_BaseColor", npcColor);
renderer.SetPropertyBlock(mpb); // 不创建新材质实例
```

**Frame Debugger 分析脚本（编辑器工具）：**

```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;

public class DrawCallAnalyzer : EditorWindow {
    [MenuItem("TA/Draw Call Analyzer")]
    static void ShowWindow() => GetWindow<DrawCallAnalyzer>();
    
    void OnGUI() {
        if (GUILayout.Button("Analyze Current Scene")) {
            var renderers = FindObjectsByType<Renderer>(FindObjectsSortMode.None);
            var groups = new System.Collections.Generic.Dictionary<string, int>();
            
            foreach (var r in renderers) {
                if (!r.enabled || !r.gameObject.activeInHierarchy) continue;
                
                foreach (var mat in r.sharedMaterials) {
                    if (mat == null) continue;
                    string key = mat.shader.name;
                    if (!groups.ContainsKey(key)) groups[key] = 0;
                    groups[key]++;
                }
            }
            
            // 按 Draw Call 贡献排序
            var sorted = System.Linq.Enumerable.OrderByDescending(groups, x => x.Value);
            foreach (var kv in sorted) {
                EditorGUILayout.LabelField($"{kv.Key}", $"Renderers: {kv.Value}");
            }
            
            // SRP Batcher 兼容性检查
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("SRP Batcher Compatibility Check:", EditorStyles.boldLabel);
            foreach (var r in renderers) {
                foreach (var mat in r.sharedMaterials) {
                    if (mat == null) continue;
                    bool ok = mat.shader.IsSRPBatcherCompatible();
                    if (!ok) {
                        EditorGUILayout.LabelField($"  ❌ {mat.name} ({mat.shader.name})");
                    }
                }
            }
        }
    }
}
#endif
```

**优化前后的 Draw Call 分布预估：**

| 分类 | 优化前 DC | 优化策略 | 优化后 DC | 降幅 |
|------|-----------|----------|-----------|------|
| 角色 (30个) | 150 | SRP Batcher + 材质模板 | 50 | 67% |
| 特效 (50组) | 200 | 特效图集 + 粒子合批 | 30 | 85% |
| 场景建筑 | 100 | Static Batching | 12 | 88% |
| UI | 30 | 补全图集 | 8 | 73% |
| **总计** | **500** | — | **100** | **80%** |

### ⚡ 实战经验

1. **SRP Batcher 不是银弹**：如果你的 Shader 变体太多（multi_compile 爆炸），SRP Batcher 的分组会非常多，反而增加 CPU 开销。用 `ShaderUtil.GetShaderVariantCount()` 检查变体数
2. **GPU Instancing 和 SRP Batcher 互斥**：同一物体只能用其中一种。策略是：相同 Mesh 用 Instancing，不同 Mesh 同 Shader 用 SRP Batcher
3. **粒子的最大杀手是"各自独立的材质"**：美术做特效时习惯拖不同贴图。建立特效图集规范后，同屏 100 个粒子可以合到 1-3 个 Draw Call
4. **Profiler 永远是第一步**：不要凭猜测优化。Unity Profiler 的 CPU Usage > Hierarchy 面板可以看到 Render.RenderVisible → Render.Mesh 的具体耗时

### 🎯 能力体检清单

- [ ] **如果不知道 SRP Batcher 原理** → 你需要补：URP/HDRP 的渲染批处理机制、CBUFFER 声明规范、Shader 变体管理
- [ ] **如果混淆几种合批技术** → 你需要补：做一张合批技术对比表，标注每种技术的触发条件和适用场景
- [ ] **如果不知道怎么定位 Draw Call 来源** → 你需要补：Frame Debugger 使用、Render Doc 分析流程
- [ ] **如果不会评估优化收益** → 你需要补：性能预算的概念（如移动端 DC 预算 100-150）、不同硬件的性能基准
- [ ] **如果不懂材质模板化** → 你需要补：TA 的资产规范设计、材质系统架构

### 🔗 相关问题

- GPU 带宽优化的策略有哪些？（手机 Memory Bandwidth 预算多少？）
- 如何用 Render Profiler 定位 CPU vs GPU 瓶颈？
- 角色换装系统如何在保证表现力的同时控制 Draw Call？
