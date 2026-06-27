---
title: "角色风衣动态摆动：蒙皮角色布料模拟的全链路方案怎么做？"
category: "technical-art"
level: 3
tags: ["布料模拟", "骨骼动画", "Verlet积分", "Unity Cloth", "性能优化", "角色渲染"]
hint: "核心是物理模拟（Verlet/AABB）+ 蒙皮绑定 + 碰撞约束 + LOD 分级——不是给美术一个 Unity Cloth 就完事，要整合到角色渲染管线中"
related: ["technical-art/skeletal-animation-jitter-precision", "technical-art/lod-spec-and-qa", "optimization/skinned-mesh-vertex-animation-cost"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一个开放世界游戏，主角穿长风衣，需要在跑动、转身、跳跃、下蹲时风衣自然摆动。不能穿模到身体里，风衣下摆要拖在地上但又不能卡地面。给你 2000 个三角形的布料预算和移动端 2ms 的 CPU 预算，给我你的方案。」

追问1：「如果用 Verlet 物理，你怎么处理与角色骨骼的碰撞约束？」

追问2：「多角色同屏（比如 10 个 NPC 都穿布料）时怎么控制性能？」

### ✅ 核心要点

1. **方案选型三叉路**：Unity 内置 Cloth / 自研 Verlet 物理 / DCC 动画烘焙——各有适用场景
2. **Verlet 积分方案**：粒子-弹簧系统，约束求解 + 蒙皮回 Mesh，最灵活也最复杂
3. **碰撞体系**：角色骨骼胶囊体碰撞 + 地面高度场碰撞 + 自碰撞
4. **LOD 分级**：近距离物理模拟 → 中距离骨骼驱动 → 远距离顶点着色器伪摆动
5. **性能预算**：2ms CPU 预算下最大约 200-300 个粒子（移动端）

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色穿风衣，运动时自然摆动，不穿模，不卡地面
                ↑
倒推方案路径：
├── 自然摆动 → 需要物理模拟（不是纯动画，否则每个动画都要烘焙布料）
├── 不穿模 → 碰撞约束（布料粒子不能穿透身体碰撞体）
├── 不卡地面 → 地面高度查询 + 约束
├── 三种运动状态都要正确 → 需要继承角色速度 + 角速度 + 风力
├── 方案选择
│   ├── Unity 内置 Cloth → 简单但不可控（碰撞体只能 Sphere），无法做 LOD
│   ├── 自研 Verlet → 灵活可控，但开发量大
│   └── DCC 烘焙 → 每个动画独立烘焙布料，播放时混合——内存翻倍且不能实时反应
├── 最终选择：自研 Verlet + LOD 分级
│   ├── 近距离：Verlet 物理模拟（~200 粒子）
│   ├── 中距离：简化 Verlet（~50 粒子）或骨骼跟随
│   └── 远距离：Vertex Shader 伪摆动（正弦波 + 速度方向）
└── 蒙皮整合
    ├── 布料粒子位置 → 驱动额外骨骼 → 蒙皮到风衣 Mesh
    └── 或直接在 Compute Shader 中更新顶点位置
```

#### 知识点拆解（倒推树）

```
角色布料模拟全链路
├── 物理模拟层
│   ├── Verlet 积分
│   │   ├── 基本公式：x_new = x + (x - x_prev) * damping + a * dt²
│   │   ├── 优点：不需要存速度，位置驱动，天然稳定
│   │   ├── 粒子类型：固定点（绑定到骨骼）、动态点（自由运动）
│   │   └── 多次迭代约束求解（约束求解器迭代 3-5 次）
│   ├── 距离约束（弹簧）
│   │   ├── 两粒子间保持目标距离：push/pull 修正
│   │   ├── 风衣布料：结构约束（横纵）+ 剪切约束（对角）+ 弯曲约束（隔点）
│   │   └── 约束权重：靠近腰部的约束硬，下摆约束软
│   ├── 碰撞约束
│   │   ├── 角色身体：胶囊体或球体碰撞（大腿2个、躯干1个、手臂2个）
│   │   ├── 碰撞响应：穿透后将粒子推到碰撞体表面 + 摩擦衰减
│   │   └── 自碰撞：布料与布料的碰撞（移动端通常省略，用弯曲约束近似）
│   └── 外力
│       ├── 角色速度：从 Rigidbody 或动画 delta position 推算
│       ├── 风力：全局风向 + 噪声扰动
│       └── 角速度：转身时风衣应该甩开（离心力效果）
├── 蒙皮整合层
│   ├── 方案A：粒子 → 额外骨骼 → 标准 SkinnedMeshRenderer 蒙皮
│   │   ├── 每个 Verlet 粒子对应一个 Transform（DynamicBone 思路）
│   │   ├── 风衣 Mesh 蒙皮到这些 Transform + 原始骨骼
│   │   └── 优点：渲染管线零改动；缺点：Transform 更新有开销
│   ├── 方案B：粒子 → 直接写顶点位置（Custom 路径）
│   │   ├── CPU 更新顶点 Buffer → GraphicsBuffer
│   │   ├── 或 Compute Shader 直接更新 Vertex Buffer
│   │   └── 优点：高性能；缺点：与标准蒙皮管线不兼容
│   └── 方案C：混合——固定点走骨骼蒙皮，动态点走 Vertex Offset
│       ├── 最实用的方案
│       └── 风衣上部（腰带处）正常蒙皮，下部用物理 offset
├── 地面碰撞
│   ├── LayerMask 地面射线检测
│   ├── 下摆粒子向下运动时检测到地面 → 推回地面以上 + 摩擦力
│   └── 性能优化：不是每帧检测，每 2-3 帧检测一次
├── LOD 分级
│   ├── LOD0 (<15m)：完整 Verlet 模拟，200 粒子，5 次约束迭代
│   ├── LOD1 (15-30m)：简化 Verlet，50 粒子，2 次迭代
│   ├── LOD2 (30-60m)：骨骼链摆动（类似 DynamicBone），0 物理迭代
│   └── LOD3 (>60m)：Vertex Shader 正弦波摆动，CPU 零开销
└── 性能预算分配（移动端 2ms）
    ├── Verlet 积分更新：0.8ms（200 粒子 × 5 次迭代）
    ├── 碰撞检测：0.6ms（简化为球体碰撞）
    ├── 蒙皮更新：0.3ms
    ├── 地面检测：0.2ms（每 3 帧轮询）
    └── 缓冲：0.1ms
```

#### 代码实现

**Verlet 布料模拟核心（C#，Unity）：**

```csharp
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// 轻量级 Verlet 布料模拟器
/// 适用于角色风衣、披风、裙摆等
/// </summary>
public class VerletClothSimulator : MonoBehaviour
{
    [System.Serializable]
    public class ClothParticle
    {
        public Vector3 position;      // 当前位置（世界空间）
        public Vector3 oldPosition;   // 上一帧位置
        public Vector3 parentLocalPos; // 绑定到骨骼时的局部偏移
        public Transform parentBone;   // 绑定的骨骼（null = 自由粒子）
        public float radius = 0.05f;   // 碰撞半径
        public bool isPinned => parentBone != null;
    }
    
    [System.Serializable]
    public class ClothConstraint
    {
        public int particleA;
        public int particleB;
        public float restLength;
        public float stiffness = 1f; // 0-1
    }
    
    [Header("模拟参数")]
    [SerializeField] private int constraintIterations = 4;
    [SerializeField] private float globalDamping = 0.98f;
    [SerializeField] private Vector3 gravity = new Vector3(0, -9.81f, 0);
    [SerializeField] private float windStrength = 1.5f;
    [SerializeField] private Vector3 windDirection = new Vector3(1, 0, 0.3f);
    
    [Header("碰撞体")]
    [SerializeField] private SphereCollider[] bodyColliders; // 角色身体碰撞球
    
    [Header("LOD")]
    [SerializeField] private float lod0Distance = 15f;
    [SerializeField] private float lod1Distance = 30f;
    [SerializeField] private float lod2Distance = 60f;
    
    private List<ClothParticle> particles = new List<ClothParticle>();
    private List<ClothConstraint> constraints = new List<ClothConstraint>();
    private float fixedDeltaTime;
    private int currentLOD = 0;
    
    /// <summary>
    /// 初始化布料网格（在 Awake/Start 中调用）
    /// </summary>
    public void InitializeGrid(
        Vector3 originPos, 
        int width, int height, 
        float cellSize,
        Transform[] topRowBones)
    {
        particles.Clear();
        constraints.Clear();
        
        // 创建粒子网格
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                var p = new ClothParticle();
                Vector3 localOffset = new Vector3(
                    (x - width * 0.5f) * cellSize,
                    -y * cellSize, // 向下
                    0
                );
                p.position = originPos + localOffset;
                p.oldPosition = p.position;
                
                // 顶行绑定到骨骼
                if (y == 0 && x < topRowBones.Length)
                {
                    p.parentBone = topRowBones[x];
                    p.parentLocalPos = localOffset;
                }
                
                particles.Add(p);
            }
        }
        
        // 创建约束
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                int idx = y * width + x;
                
                // 结构约束（横向）
                if (x < width - 1)
                    AddConstraint(idx, idx + 1, cellSize, 0.9f);
                
                // 结构约束（纵向）
                if (y < height - 1)
                    AddConstraint(idx, idx + width, cellSize, 
                        y == 0 ? 0.95f : 0.7f); // 上部硬、下部软
                
                // 剪切约束（对角线）
                if (x < width - 1 && y < height - 1)
                    AddConstraint(idx, idx + width + 1, 
                        cellSize * 1.414f, 0.3f);
                
                // 弯曲约束（跳一格）
                if (x < width - 2)
                    AddConstraint(idx, idx + 2, cellSize * 2, 0.1f);
                if (y < height - 2)
                    AddConstraint(idx, idx + width * 2, cellSize * 2, 0.1f);
            }
        }
    }
    
    void AddConstraint(int a, int b, float restLen, float stiff)
    {
        constraints.Add(new ClothConstraint
        {
            particleA = a,
            particleB = b,
            restLength = restLen,
            stiffness = stiff
        });
    }
    
    void Update()
    {
        UpdateLOD();
    }
    
    void FixedUpdate()
    {
        float dt = Time.fixedDeltaTime;
        
        switch (currentLOD)
        {
            case 0: SimulateFull(dt); break;
            case 1: SimulateSimplified(dt); break;
            case 2: SimulateBoneChain(dt); break;
            // LOD3: 不需要 CPU 模拟，VS 处理
        }
    }
    
    void SimulateFull(float dt)
    {
        // Step 1: Verlet 积分
        for (int i = 0; i < particles.Count; i++)
        {
            var p = particles[i];
            if (p.isPinned)
            {
                // 固定点跟随骨骼
                p.position = p.parentBone.TransformPoint(p.parentLocalPos);
                p.oldPosition = p.position;
                continue;
            }
            
            Vector3 velocity = (p.position - p.oldPosition) * globalDamping;
            Vector3 acceleration = gravity + GetWindForce();
            
            p.oldPosition = p.position;
            p.position += velocity + acceleration * dt * dt;
        }
        
        // Step 2: 约束求解（多次迭代）
        for (int iter = 0; iter < constraintIterations; iter++)
        {
            for (int i = 0; i < constraints.Count; i++)
            {
                var c = constraints[i];
                var pa = particles[c.particleA];
                var pb = particles[c.particleB];
                
                Vector3 delta = pb.position - pa.position;
                float dist = delta.magnitude;
                if (dist < 0.0001f) continue;
                
                float diff = (dist - c.restLength) / dist;
                Vector3 correction = delta * 0.5f * diff * c.stiffness;
                
                if (!pa.isPinned) pa.position += correction;
                if (!pb.isPinned) pb.position -= correction;
            }
        }
        
        // Step 3: 碰撞约束
        ResolveCollisions();
        
        // Step 4: 地面碰撞
        ResolveGroundCollision();
    }
    
    void ResolveCollisions()
    {
        for (int i = 0; i < particles.Count; i++)
        {
            var p = particles[i];
            if (p.isPinned) continue;
            
            foreach (var col in bodyColliders)
            {
                Vector3 colCenter = col.transform.TransformPoint(col.center);
                float colRadius = col.radius * Mathf.Max(
                    col.transform.lossyScale.x,
                    col.transform.lossyScale.y,
                    col.transform.lossyScale.z);
                
                Vector3 toParticle = p.position - colCenter;
                float dist = toParticle.magnitude;
                float minDist = colRadius + p.radius;
                
                if (dist < minDist)
                {
                    // 推到球面上
                    p.position = colCenter + toParticle.normalized * minDist;
                }
            }
        }
    }
    
    void ResolveGroundCollision()
    {
        // 简化版：只检测下摆粒子
        for (int i = particles.Count - 1; i >= 0; i--)
        {
            var p = particles[i];
            if (p.isPinned) continue;
            
            // 射线检测地面（每 2 帧检测一次以节省性能）
            if (Time.frameCount % 2 == 0)
            {
                if (Physics.Raycast(p.oldPosition + Vector3.up * 0.1f, 
                    Vector3.down, out RaycastHit hit, 0.5f, 
                    LayerMask.GetMask("Ground")))
                {
                    if (p.position.y < hit.point.y + p.radius)
                    {
                        p.position = new Vector3(
                            p.position.x,
                            hit.point.y + p.radius,
                            p.position.z
                        );
                        // 摩擦衰减
                        p.oldPosition = new Vector3(
                            p.position.x + (p.oldPosition.x - p.position.x) * 0.3f,
                            p.oldPosition.y,
                            p.position.z + (p.oldPosition.z - p.position.z) * 0.3f
                        );
                    }
                }
            }
        }
    }
    
    Vector3 GetWindForce()
    {
        // 基础风向 + Perlin 噪声扰动
        float noiseT = Time.time * 0.5f;
        float noiseX = Mathf.PerlinNoise(noiseT, 0f) - 0.5f;
        float noiseZ = Mathf.PerlinNoise(0f, noiseT) - 0.5f;
        return windDirection.normalized * windStrength + 
               new Vector3(noiseX, 0, noiseZ) * windStrength * 0.5f;
    }
    
    void SimulateSimplified(float dt)
    {
        // LOD1: 减少迭代次数，跳过部分碰撞
        constraintIterations = 2;
        SimulateFull(dt);
    }
    
    void SimulateBoneChain(float dt)
    {
        // LOD2: 只用骨骼链摆动（类似 DynamicBone 简化版）
        // 每个"骨骼"是一个粒子，没有布料网格约束
        for (int i = 0; i < particles.Count; i++)
        {
            var p = particles[i];
            if (p.isPinned)
            {
                p.position = p.parentBone.TransformPoint(p.parentLocalPos);
                p.oldPosition = p.position;
                continue;
            }
            
            // 简化：只做重力 + 阻尼，不做约束迭代
            Vector3 velocity = (p.position - p.oldPosition) * 0.95f;
            p.oldPosition = p.position;
            p.position += velocity + gravity * dt * dt;
            
            // 距离约束（只保持与父粒子的距离）
            if (i > 0)
            {
                var parent = particles[i - 1];
                Vector3 delta = p.position - parent.position;
                float dist = delta.magnitude;
                float restLen = (parent.oldPosition - p.oldPosition).magnitude;
                if (dist > restLen * 1.2f)
                {
                    p.position = parent.position + delta.normalized * restLen * 1.2f;
                }
            }
        }
    }
    
    void UpdateLOD()
    {
        float distToCamera = Vector3.Distance(
            transform.position, 
            Camera.main.transform.position);
        
        int newLOD = distToCamera switch
        {
            < 15f => 0,
            < 30f => 1,
            < 60f => 2,
            _ => 3
        };
        
        if (newLOD != currentLOD)
        {
            currentLOD = newLOD;
            // 可以在此切换 Material（LOD3 用 VS 摆动 Shader）
        }
    }
    
    /// <summary>
    /// 获取粒子位置数组（供 GraphicsBuffer 或骨骼更新使用）
    /// </summary>
    public Vector3[] GetParticlePositions()
    {
        var positions = new Vector3[particles.Count];
        for (int i = 0; i < particles.Count; i++)
        {
            positions[i] = transform.InverseTransformPoint(particles[i].position);
        }
        return positions;
    }
    
    void OnDrawGizmosSelected()
    {
        // 调试可视化
        Gizmos.color = Color.cyan;
        foreach (var p in particles)
        {
            Gizmos.DrawWireSphere(p.position, p.radius);
        }
        
        Gizmos.color = Color.yellow;
        foreach (var c in constraints)
        {
            if (c.particleA < particles.Count && c.particleB < particles.Count)
            {
                Gizmos.DrawLine(
                    particles[c.particleA].position,
                    particles[c.particleB].position);
            }
        }
    }
}
```

**Vertex Shader 伪摆动（LOD3 远距离用）：**

```hlsl
// 远距离布料伪摆动 Shader（零 CPU 开销）
// 在顶点着色器中用正弦波模拟摆动

struct Attributes {
    float3 positionOS : POSITION;
    float3 normalOS   : NORMAL;
    float2 uv         : TEXCOORD0;
};

// 材质属性
float _WindStrength;
float _WindFrequency;
float3 _WindDirection;
float _ClothHeightFactor; // 基于UV.y的摆动权重（下摆摆得多）

Varyings vert(Attributes input) {
    Varyings output = (Varyings)0;
    
    // 摆动权重：UV.y 越大（下摆）摆动越强
    float weight = smoothstep(0.3, 1.0, input.uv.y);
    
    // 正弦波摆动（两个不同频率叠加，避免过于规律）
    float windPhase1 = _Time.y * _WindFrequency;
    float windPhase2 = _Time.y * _WindFrequency * 1.7 + 1.3;
    
    float sway = sin(windPhase1 + input.positionOS.x * 3.0) * 0.5 + 
                 sin(windPhase2 + input.positionOS.z * 2.0) * 0.3;
    
    // 摆动方向：风向 XZ 平面
    float3 offset = float3(
        _WindDirection.x * sway * _WindStrength * weight,
        abs(sway) * -0.3 * weight, // 轻微下垂
        _WindDirection.z * sway * _WindStrength * weight
    );
    
    float3 posOffset = input.positionOS + offset;
    
    output.positionCS = TransformObjectToHClip(posOffset);
    // ... 其他属性传递
    return output;
}
```

### ⚡ 实战经验

1. **Unity 内置 Cloth 组件的陷阱**：它只能在编辑器中配置碰撞球，不能运行时动态添加/移除碰撞体。角色换装系统（不同装备 → 不同碰撞体）基本不能用它。自研方案最大的价值就是可控性

2. **Verlet 的稳定性依赖于固定时间步长**：不要在 `Update` 里跑物理模拟！必须用 `FixedUpdate` 或固定步长循环。帧率波动时如果 dt 忽大忽小，Verlet 积分会爆炸（布料飞到天际）

3. **约束求解器的性能优化**：约束求解是最贵的部分（O(constraints × iterations)）。实战技巧：① 第一次迭代只处理结构约束，第二次再加剪切约束；② 用 Job System + Burst Compiler 并行化约束求解；③ 把约束数据转成 NativeArray 避免 GC

4. **多角色同屏的实例化方案**：10 个 NPC 各自 200 粒子 = 2000 粒子 × 4 次迭代 = 8000 次约束求解。用 ECS（DOTS）可以将所有角色的布料数据放在连续内存中，Burst 编译后性能提升 5-10 倍

5. **穿模问题的终极处理**：即使有碰撞约束，快速运动时仍可能穿模（Tunneling）。解决方案：① 增大碰撞球半径（比视觉体积大 20%）；② 连续碰撞检测（CCD）——检测粒子从 oldPos 到 newPos 的线段是否穿透碰撞体；③ 降低物理模拟频率不要太高，避免大步长

6. **动画与布料的混合**：角色做特定动画（如蹲下、翻滚）时，布料应该贴身而不是飞起来。方案：在动画事件中标记"布料黏附区"，该区域粒子的约束 stiffness 暂时提高到 0.99，动画结束后恢复

### 🎯 能力体检清单

| 知识点 | 自检问题 | 盲区信号 |
|--------|----------|----------|
| Verlet 积分原理 | 能手写 Verlet 积分公式吗？为什么它不需要显式存速度？ | ❌ 只知道"物理模拟"→ 深入追问就答不上来 |
| 约束求解 | 距离约束的 push/pull 修正怎么算？为什么需要多次迭代？ | ❌ 不了解迭代收敛 → 布料会抖动或被拉穿 |
| 碰撞处理 | 球体碰撞的修正公式是什么？CCD（连续碰撞检测）的原理？ | ❌ 只会简单距离检测 → 高速运动穿模 |
| LOD 策略 | 布料 LOD 分几级？每级砍什么？远距离用什么方案？ | ❌ 没有分级思路 → 10 个 NPC 同屏直接卡死 |
| 性能预算 | 移动端 2ms 内能模拟多少粒子？瓶颈在哪？ | ❌ 没有量化概念 → 无法回答面试官的预算追问 |
| 蒙皮整合 | 布料粒子位置如何驱动 Mesh 变形？方案A/B/C 的区别？ | ❌ 只知道 Unity Cloth 组件 → 深入就断了 |

### 🔗 相关问题

- [骨骼动画精度问题与抖动](../technical-art/skeletal-animation-jitter-precision.md) — 布料固定点绑定骨骼时的精度问题
- [LOD 规范制定与 QA](../technical-art/lod-spec-and-qa.md) — 布料 LOD 分级的标准制定
- [蒙皮网格顶点动画开销](../optimization/skinned-mesh-vertex-animation-cost.md) — 布料蒙皮的性能预算
