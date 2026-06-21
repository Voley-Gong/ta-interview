---
title: "角色卡通渲染：如何实现稳定的描边 Outline？"
category: "shader"
level: 2
tags: ["NPR", "卡渲", "Outline", "Shader", "后处理"]
hint: "描边的核心是「边缘检测」还是「法线外扩」？各有什么优缺点？"
related: ["shader/sss-skin-rendering", "rendering/post-process-outline"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们项目用的是卡通渲染风格，角色需要描边效果。现在角色在近距离镜头特写时描边会闪烁、断裂，远距离时描边又消失。你会怎么解决？

这是米哈游、鹰角、叠纸等二次元项目的经典面试题。描边是卡渲的"门面"，面试官通过这道题能考察你对 Shader、渲染管线、几何处理的多层理解。

### ✅ 核心要点

1. **描边方案选型**：法线外扩 / Back-face 描边 / 后处理描边 / 几何描边——不同方案的适用场景
2. **法线平滑**：顶点法线不平滑是描边断裂的根本原因，需要法线平滑或法线贴图
3. **描边宽度稳定性**：根据距离调整宽度，避免远距离消失、近距离过粗
4. **描边颜色控制**：顶点色控制粗细、UV2 存储平滑法线——资产规范与 Shader 配合
5. **性能取舍**：移动端用 Back-face，PC/主机可考虑后处理几何描边

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
期望效果：稳定、均匀、可控粗细的角色描边
         ↓
问题根因分析：
  ├── 描边断裂 → 顶点法线不连续（硬边处外扩方向突变）
  ├── 描边闪烁 → 精度问题或 Z-Fighting
  ├── 近距离过粗 → 描边宽度没有做距离自适应
  └── 远距离消失 → 宽度太小被采样丢掉
         ↓
解决方案：
  Step 1：平滑法线（关键！）
  Step 2：选择合适的描边技术方案
  Step 3：距离自适应宽度
  Step 4：资产规范配合（顶点色/UV通道）
```

#### 知识点拆解（倒推树）

```
稳定描边
├── 法线外扩方案（最常用）
│   ├── 需要理解：顶点着色器中沿法线膨胀顶点
│   │   └── HLSL: pos += normal * outlineWidth
│   ├── 需要解决：硬边顶点法线不连续
│   │   └── 法线平滑算法：邻接顶点法线加权平均
│   ├── 需要存储：平滑法线怎么传给 Shader？
│   │   ├── 方案A：DCC工具烘焙到 UV2.xy（不影响原始 UV）
│   │   ├── 方案B：烘焙到顶点色 RGB（占用了颜色通道）
│   │   └── 方案C：烘焙到切线空间 tangent
│   └── 需要理解：两 Pass 渲染（先画描边再画本体）
│       └── Cull Front → 放大背面 → 仅露出边缘
│
├── 后处理描边（Roberts/Sobel 边缘检测）
│   ├── 需要理解：屏幕空间边缘检测
│   ├── 需要解决：深度/法线作为检测源
│   └── 缺点：无法控制线条粗细变化、性能开销大
│
├── 几何描边（Geometry Shader 生成）
│   ├── 需要理解：GS 阶段生成额外的边缘四边形
│   └── 缺点：移动端不支持 GS，顶点开销大
│
└── 距离自适应
    ├── 需要理解：屏幕空间宽度一致性
    │   └── outlineWidth *= clamp(distance, minDist, maxDist)
    └── 需要理解：CSS-like 的 width 自适应公式
```

#### 代码实现

**法线外扩描边 Shader（URP 兼容）：**

```hlsl
// OutlinePass.vert — 顶点着色器
struct Attributes {
    float4 positionOS  : POSITION;
    float3 normalOS    : NORMAL;
    float2 uv          : TEXCOORD0;
    float2 smoothNormal : TEXCOORD1; // UV2 存储平滑法线
};

struct Varyings {
    float4 positionCS  : SV_POSITION;
    float2 uv          : TEXCOORD0;
};

float _OutlineWidth;
float4 _OutlineColor;
float  _DistanceScale;
float  _FadeDistance;
float  _FadeRange;

Varyings OutlinePassVert(Attributes input) {
    Varyings output;

    // 使用平滑法线（存在 UV2 中）替代原始法线
    float3 normalOS = input.smoothNormal.xyy;
    // 如果没有平滑法线资产，可以 fallback 到原始法线
    // float3 normalOS = input.normalOS;

    // 顶点沿平滑法线外扩
    float3 posOS = input.positionOS.xyz + normalOS * _OutlineWidth;

    output.positionCS = TransformObjectToHClip(posOS);
    output.uv = input.uv;

    // 距离自适应：远距离不缩小太多，近距离不放大太多
    float viewDist = length(GetCameraPositionWS() - TransformObjectToWorld(input.positionOS.xyz));
    float fadeFactor = 1.0 - saturate((viewDist - _FadeDistance) / _FadeRange);
    output.positionCS.xy *= fadeFactor; // 远距离收缩描边

    return output;
}

half4 OutlinePassFrag(Varyings input) : SV_Target {
    return half4(_OutlineColor.rgb, 1.0);
}
```

**URP Renderer Feature 注册描边 Pass：**

```csharp
// 需要 URP 14+ (Unity 2022 LTS+)
public class OutlineRendererFeature : ScriptableRendererFeature {
    public Material outlineMaterial;
    
    class OutlinePass : ScriptableRenderPass {
        private Material _mat;
        
        public OutlinePass(Material mat) { _mat = mat; }
        
        public override void Execute(ScriptableRenderContext ctx, ref RenderingData data) {
            // 在不透明物体之后绘制描边
            var cmd = CommandBufferPool.Get("Outline");
            // 设置 Cull Front，渲染背面作为描边
            cmd.DrawRenderers(data.cullResults, ref data.drawSettings,
                ref data.filteringSettings);
            ctx.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

**法线平滑工具脚本（Maya Python）：**

```python
# 在 Maya 中计算邻接顶点的平滑法线，存入 UV2
import maya.cmds as cmds
import maya.api.OpenMaya as om

def bake_smooth_normal_to_uv2(mesh_name):
    sel = om.MSelectionList()
    sel.add(mesh_name)
    dag = sel.getDagPath(0)
    fn_mesh = om.MFnMesh(dag)
    
    # 获取顶点法线
    normals = fn_mesh.getVertexNormals(False, om.MSpace.kObject)
    
    # 计算邻接顶点的平滑法线（共享边的顶点法线平均）
    # 实际项目中通常用 angle-weighted 或 area-weighted 平均
    smooth_normals = compute_adjacent_average(fn_mesh, normals)
    
    # 写入 UV set "smoothNormalUV"
    uv_util = om.MFnMesh(dag)
    uvs = [(n.x, n.y) for n in smooth_normals]  # 只存 xy，z 可推导
    uv_util.setUVs(uvs, [], "smoothNormalUV")
```

**三种描边方案对比：**

| 维度 | 法线外扩 | 后处理描边 | 几何描边 |
|------|----------|------------|----------|
| 描边质量 | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| 性能开销 | 低（多一个Pass） | 中高（全屏后处理） | 中（额外几何体） |
| 线条风格控制 | 强（可逐顶点控制） | 弱（全局参数） | 中 |
| 移动端兼容 | ✅ | ⚠️（带宽敏感） | ❌（无 GS 支持） |
| 实现复杂度 | 中 | 低 | 高 |
| 适用项目 | 二次元手游（原神/崩坏） | 欧美卡通风格 | 主机高品质 |

### ⚡ 实战经验

1. **法线平滑是描边质量的灵魂**：90% 的描边断裂问题都是法线不平滑导致的。在 DCC 工具中做一次 angle-weighted normal smoothing，存到独立 UV channel，效果立竿见影
2. **描边 Pass 的 Z-Test 设为 Less Equal**：描边背面如果被深度测试剔除，会导致描边"缺角"。设为 `ZTest Always` 或 `ZTest Less Equal` 都可以，取决于你想要描边是否穿过前景
3. **顶点色作为描边控制通道**：R 通道控制粗细（0=无描边, 1=最粗），G 通道控制颜色变化。这套规范在叠纸的游戏中被广泛使用
4. **移动端用 Half Precision**：描边宽度的计算用 `half` 而非 `float`，在移动端可以省一半带宽

### 🎯 能力体检清单

- [ ] **如果不知道法线平滑** → 你需要补：几何处理基础、顶点法线与面法线的区别、DCC 工具中的法线编辑
- [ ] **如果不知道两 Pass 描边原理** → 你需要补：渲染管线 Pass 概念、Cull Mode 的工作原理
- [ ] **如果不会做距离自适应** → 你需要补：观察空间/裁剪空间转换、屏幕空间一致性概念
- [ ] **如果不会写 URP Renderer Feature** → 你需要补：URP 渲染管线架构、ScriptableRendererFeature 机制
- [ ] **如果不知道资产规范配合** → 你需要补：TA 与美术协作的资产标准制定

### 🔗 相关问题

- 卡通渲染中如何实现 Ramp 光照（Toon Shading 的核心）？
- 描边的线条粗细如何做到「概念线条」级别（非均匀宽度）？
- 原神的描边方案和传统的有什么区别？
