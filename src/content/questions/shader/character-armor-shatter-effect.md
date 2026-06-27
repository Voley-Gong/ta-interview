---
title: "角色受击爆甲：如何用 Shader + 粒子实现盔甲破碎飞溅效果？"
category: "shader"
level: 3
tags: ["Shader", "Vertex Displacement", "噪声", "粒子", "URP", "受击反馈"]
hint: "核心是 Voronoi 噪声驱动碎片划分 + Vertex Offset 控制飞散方向 + 发光裂缝 + 粒子补位——不是简单 discard，碎片要有体积感和物理运动"
related: ["shader/dissolve-effect", "shader/hit-flash-damage-blink", "shader/energy-shield-effect"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一个动作游戏，Boss 被玩家打出破防态后，身上的盔甲要炸裂成碎片飞散开来——裂缝处有发光效果，碎片有翻滚和重力下落，最后消失。整个效果要在 URP 下用 Shader + 粒子配合实现，给我完整方案。」

### ✅ 核心要点

1. **Voronoi 碎片划分**：用 Cellular Noise（Voronoi）在模型表面生成不规则碎片区块
2. **渐进式破碎传播**：沿冲击点向外扩散，破碎半径随时间增长
3. **Vertex Offset 飞散**：碎片中心法线方向偏移 + 重力下落 + 随机翻滚
4. **裂缝发光**：碎片边缘叠加 Emission，模拟能量断裂
5. **粒子物理补位**：Shader 负责大面积破碎，粒子负责小碎屑和烟尘

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：Boss 盔甲碎裂 → 裂缝从受击点扩散 → 碎片飞散翻滚下落 → 边缘发光 → 消失

倒推实现路径：
├── 碎片飞散的体积感 → 不能用平面 alpha clip，必须 Vertex Offset
├── 不规则碎片形状 → 不能用均匀网格切割，需要 Voronoi/Cellular Noise
├── 从受击点扩散 → 需要距离场控制：距离受击点越近越先破碎
├── 裂缝发光 → 碎片边界（Cell 边缘）叠加 Emission
├── 碎片翻滚 → 每个碎片有自己的旋转轴和角速度（从 Cell ID 哈希）
├── 重力下落 → Vertex Offset 的 Y 分量叠加 -g*t² 项
└── 最终消失 → 碎片飞远后 scale → 0 或 alpha fade
```

#### 知识点拆解（倒推树）

```
角色爆甲破碎效果
├── Voronoi/Cellular Noise（3D）
│   ├── 原理：空间中随机种子点 → 每个像素属于最近的种子区域
│   ├── F1/F2 值：F1 = 到最近种子距离, F2 = 到第二近种子距离
│   └── 边界检测：|F2 - F1| < ε → 碎片边缘（用于发光）
├── 距离场扩散
│   ├── 受击点坐标传入 Shader（world space）
│   ├── 碎片破碎进度 = saturate(progress - distance(hitPoint, fragPos) * falloff)
│   └── 这与 dissolve 的 noise-threshold 思路一致，但从点向外辐射
├── Vertex Offset 计算
│   ├── 碎片质心方向偏移：normalize(fragPos - cellCenter) * pushForce
│   ├── 翻滚旋转：绕碎片随机轴旋转（轴 = hash(cellID) * 任意向量）
│   ├── 重力叠加：offset.y -= 0.5 * g * t²
│   └── 初始冲量：从受击点向外的爆发力
├── 裂缝发光
│   ├── Voronoi 边缘 = |F2 - F1| 小于阈值的区域
│   ├── Emission 颜色随破碎进度从白热 → 橙红 → 熄灭
│   └── 需要 Bloom 后处理配合
├── 粒子系统配合
│   ├── GPU 粒子：大量小碎屑（1000+），继承碎片飞散速度
│   ├── 烟尘粒子：在破碎扩散前沿生成
│   └── 冲击波：一个球形扩散 mesh + 透明 shader
└── 性能控制
    ├── 爆甲只在单个角色上触发，不需要全屏
    ├── 碎片数受 Voronoi 频率控制（不要太高）
    └── 粒子总数上限 + LOD 距离裁剪
```

#### 代码实现

**核心 Shader（HLSL，URP）：**

```hlsl
// 盔甲破碎 Shader - 关键 Pass
// Properties:
//   _ShatterProgress : 破碎进度 0→1（C# 脚本驱动）
//   _HitPoint        : 受击世界坐标
//   _ShatterRadius   : 破碎扩散速度/半径
//   _PushForce       : 碎片飞散力度
//   _CrackColor      : 裂缝发光颜色
//   _CellDensity     : Voronoi 密度

struct Attributes {
    float3 positionOS : POSITION;
    float3 normalOS   : NORMAL;
    float2 uv         : TEXCOORD0;
};

struct Varyings {
    float4 positionCS : SV_POSITION;
    float3 positionWS : TEXCOORD0;
    float3 normalWS   : TEXCOORD1;
    float2 voronoiID  : TEXCOORD2;
    float  edgeFactor : TEXCOORD3;
    float  alpha      : TEXCOORD4;
};

// 3D Cellular Noise (Voronoi)
// 返回: x=F1, y=F2, z=cellID.x, w=cellID.y
float4 voronoi3D(float3 p, float density) {
    float3 g = floor(p * density);
    float3 f = frac(p * density);
    float4 result = float4(1e8, 1e8, 0, 0);
    
    [unroll]
    for (int x = -1; x <= 1; x++) {
        [unroll]
        for (int y = -1; y <= 1; y++) {
            [unroll]
            for (int z = -1; z <= 1; z++) {
                float3 offset = float3(x, y, z);
                float3 cell = offset + frac(sin(dot(g + offset, 
                    float3(127.1, 311.7, 74.7))) * 43758.5453) - f;
                float d = length(cell);
                if (d < result.x) {
                    result.yz = result.xz; // F2 和 cellID 降级
                    result.x = d;
                    result.zw = float2(dot(g + offset, float3(1, 37, 17)), 
                                       dot(g + offset, float3(1, 17, 31)));
                } else if (d < result.y) {
                    result.y = d;
                }
            }
        }
    }
    result.xy = sqrt(result.xy); // 距离修正
    return result;
}

// 从 cellID 生成随机旋转轴和角速度
float3 hashRotAxis(float2 cellID) {
    return normalize(frac(sin(float3(
        dot(cellID, float2(127.1, 311.7)),
        dot(cellID, float2(269.5, 183.3)),
        dot(cellID, float2(419.2, 371.9))
    )) * 43758.5453) * 2 - 1);
}

Varyings vert(Attributes input) {
    Varyings output = (Varyings)0;
    
    float3 posWS = TransformObjectToWorld(input.positionOS);
    
    // 1. Voronoi 碎片划分
    float4 voro = voronoi3D(input.positionOS, _CellDensity);
    float2 cellID = voro.zw;
    
    // 2. 距离场控制破碎扩散
    float distToHit = distance(posWS, _HitPoint);
    float localProgress = saturate(_ShatterProgress - distToHit * _ShatterRadius);
    
    // 3. 碎片质心（近似：cellID → 空间位置）
    float3 cellCenterDir = normalize(input.positionOS - 
        (frac(cellID.xyx / 100.0) - 0.5));
    
    // 4. 翻滚旋转
    float3 rotAxis = hashRotAxis(cellID);
    float angle = localProgress * 3.14159 * 4.0; // 旋转 2 圈
    float3 fragPos = input.positionOS;
    
    // Rodrigues 旋转公式（绕任意轴）
    float cosA = cos(angle * localProgress);
    float sinA = sin(angle * localProgress);
    fragPos = fragPos * cosA + cross(rotAxis, fragPos) * sinA + 
              rotAxis * dot(rotAxis, fragPos) * (1 - cosA);
    
    // 5. 飞散偏移：外推 + 重力
    float3 pushDir = cellCenterDir * _PushForce * localProgress;
    float3 gravity = float3(0, -1, 0) * 0.5 * localProgress * localProgress * 20.0;
    fragPos += pushDir + gravity;
    
    // 6. 裂缝边缘检测
    float edgeWidth = 0.05;
    output.edgeFactor = saturate((voro.y - voro.x) / edgeWidth);
    
    // 7. Alpha 控制：碎片飞远后淡出
    output.alpha = saturate(1.0 - localProgress * 1.3);
    
    output.positionWS = TransformObjectToWorld(fragPos);
    output.positionCS = TransformWorldToHClip(output.positionWS);
    output.normalWS = TransformObjectToWorldNormal(input.normalOS);
    output.voronoiID = cellID;
    
    return output;
}

half4 frag(Varyings input) : SV_Target {
    // 基础材质色
    half3 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, 
        input.positionWS.xz * 0.1).rgb;
    
    // 裂缝发光：Voronoi 边缘处高强度发光
    half3 crackGlow = _CrackColor * (1.0 - input.edgeFactor) * 3.0;
    
    // 混合
    half3 finalColor = baseColor + crackGlow;
    
    // Alpha 裁剪 + 淡出
    clip(input.alpha - 0.01);
    
    return half4(finalColor, input.alpha);
}
```

**C# 触发脚本：**

```csharp
using UnityEngine;

[RequireComponent(typeof(Renderer))]
public class ArmorShatterController : MonoBehaviour
{
    [Header("破碎参数")]
    [SerializeField] private float shatterDuration = 2.5f;
    [SerializeField] private float maxPushForce = 3.0f;
    [SerializeField] private Transform hitPoint;
    
    private MaterialPropertyBlock mpb;
    private Renderer rend;
    private float shatterTimer = -1f;
    
    // 材质属性 ID 缓存
    private static readonly int ShatterProgressID = Shader.PropertyToID("_ShatterProgress");
    private static readonly int HitPointID = Shader.PropertyToID("_HitPoint");
    private static readonly int PushForceID = Shader.PropertyToID("_PushForce");
    
    void Awake() {
        rend = GetComponent<Renderer>();
        mpb = new MaterialPropertyBlock();
    }
    
    /// <summary>
    /// 外部调用：触发爆甲
    /// </summary>
    public void TriggerShatter(Vector3 worldHitPoint) {
        if (shatterTimer >= 0f) return; // 已在破碎中
        
        mpb.SetVector(HitPointID, worldHitPoint);
        mpb.SetFloat(PushForceID, maxPushForce);
        rend.SetPropertyBlock(mpb);
        
        shatterTimer = 0f;
        
        // 触发粒子系统
        var particles = GetComponent<ParticleSystem>();
        if (particles != null) particles.Play();
        
        // 延迟隐藏碰撞体
        Invoke(nameof(DisableCollider), 0.5f);
    }
    
    void Update() {
        if (shatterTimer < 0f) return;
        
        shatterTimer += Time.deltaTime;
        float progress = Mathf.Clamp01(shatterTimer / shatterDuration);
        
        rend.GetPropertyBlock(mpb);
        mpb.SetFloat(ShatterProgressID, progress);
        rend.SetPropertyBlock(mpb);
        
        if (progress >= 1f) {
            gameObject.SetActive(false);
        }
    }
    
    void DisableCollider() {
        var col = GetComponent<Collider>();
        if (col != null) col.enabled = false;
    }
}
```

### ⚡ 实战经验

1. **Voronoi 在顶点着色器中的性能问题**：3D Cellular Noise 需要遍历 27 个 cell，在顶点着色器中如果顶点数高（10万+）会很贵。解决方案：① 在 LOD 低模上算破碎，高模仅做 visual; ② 预计算 Voronoi 到顶点色/UV2; ③ 用 2D Voronoi（基于 UV 或屏幕空间）替代 3D，视觉效果接近但成本低得多

2. **碎片旋转的视觉陷阱**：纯数学旋转碎片会导致碎片之间穿插穿透。实战中可以容忍少量穿插（速度够快看不出来），但如果慢镜头需要严格分离，就要预计算每个碎片的分离方向（在 DCC 中预分割模型）

3. **SRP Batcher 兼容**：`MaterialPropertyBlock` 会打断 SRP Batcher。如果场景中有多个角色但只有一个触发爆甲，考虑用 `Graphics.DrawMesh` 单独渲染破碎中的角色，不影响其他角色的 batch

4. **移动端适配**：移动端建议简化——用预分割模型（在 Houdini/Maya 中切好碎片）+ 简单 vertex shader 做飞散，不做实时 Voronoi。Voronoi 方案适合 PC/主机

5. **断裂感的进阶**：加入「延迟断裂」——碎片先从母体脱离但保持原位（0.1s），然后才飞散，能极大增强力量感。在 progress 曲线上用 `step(progress, 0.05)` 做一个延迟门控

### 🎯 能力体检清单

| 知识点 | 自检问题 | 盲区信号 |
|--------|----------|----------|
| Voronoi Noise 原理 | 能手写 3D Cellular Noise 吗？F1/F2 的区别是什么？ | ❌ 不了解 F2 的用途 → 裂缝边缘检测做不了 |
| Vertex Offset | 知道如何在顶点着色器中做旋转和位移吗？Rodrigues 公式？ | ❌ 只会改 fragment color → 碎片是平的，没体积感 |
| MaterialPropertyBlock | 知道为什么用它而不是直接改材质吗？ | ❌ 直接 renderer.material.SetFloat → 实例化材质，内存泄漏 |
| 粒子与 Shader 配合 | 能说出 Shader 负责什么、粒子负责什么吗？ | ❌ 全靠粒子 → 数量爆炸；全靠 Shader → 缺少小碎屑细节 |
| URP 性能预算 | 移动端这个效果的帧时间预算是多少？ | ❌ 没概念 → 面试官追问移动端适配时无法回答 |

### 🔗 相关问题

- [角色溶解消失](../shader/dissolve-effect.md) — 同为"消失"类效果，但溶解是 alpha clip，爆甲是 vertex offset
- [受击闪烁反馈](../shader/hit-flash-damage-blink.md) — 受击反馈的第一层（闪烁），爆甲是第三层（模型破碎）
- [能量护盾受击效果](../shader/energy-shield-effect.md) — 受击反馈的第二层（护盾波动）
