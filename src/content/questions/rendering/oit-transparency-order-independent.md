---
title: "半透明排序崩了：OIT 方案如何在角色头发/特效中拯救渲染顺序？"
category: "rendering"
level: 4
tags: ["OIT", "半透明排序", "Weighted Blended", "Per-Pixel Linked List", "渲染管线", "头发渲染"]
hint: "核心痛点：传统 Alpha Blending 依赖绘制顺序，复杂重叠区域排序崩溃 → OIT 让顺序无关"
related: ["rendering/forward-plus-cluster", "shader/hair-anisotropic-lighting", "rendering/hiz-screen-space-reflection"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们游戏的角色有大量头发卡片（Hair Cards），加上翅膀特效、半透明裙摆，在特定角度下半透明排序完全崩溃——头发穿过身体的部分一会儿在前一会儿在后。性能还不能砍太多，移动端也要能跑。给我你的解决方案。」

### ✅ 核心要点

1. **问题本质**：硬件 Alpha Blending 是 order-dependent 的，`blend(src, dst)` 依赖绘制顺序，GPU 不做排序
2. **OIT 核心思想**：先收集所有半透明片元的贡献，再按正确深度顺序合成
3. **三大方案选型**：
   - **WBOIT（Weighted Blended OIT）**：加权混合，近似但不精确，性能最好
   - **Per-Pixel Linked List OIT**：GPU 链表精确排序，质量最高，显存开销大
   - **Depth-Peeling**：逐层剥离，精确但 Pass 数多
4. **头发特殊处理**：头发卡片可以用排序网格 + Alpha Test 混合策略绕过
5. **移动端现实**：纯 WBOIT 或 Alpha to Coverage 是移动端唯一可行方案

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：半透明物体无论从哪个角度看，前后遮挡关系都正确
                ↑
倒推1：传统 Alpha Blending 需要严格从后往前排序 → CPU 排序不可靠
倒推2：需要让渲染结果与绘制顺序无关 → OIT (Order-Independent Transparency)
倒推3：OIT 需要收集所有半透明片元 → 需要 UAV/StructuredBuffer 存储
倒推4：收集后需要排序 → 按 depth 排序后 blend，或用加权近似 blend
倒推5：移动端 UAV 受限 → 只能用加权近似方案（WBOIT）
倒推6：性能敏感 → 头发可以用 Alpha to Coverage（MSAA 的 1bit 透明）
```

#### 知识点拆解（倒推树）

```
OIT 半透明排序
├── 问题根源
│   ├── Alpha Blending 公式：dst = src * alpha + dst * (1-alpha)
│   ├── 数学上不可交换 → (A over B) ≠ (B over A)
│   └── CPU 排序只对物体级别，像素级别重叠无法处理
├── WBOIT（Weighted Blended OIT）
│   ├── 原理：用深度加权代替精确排序
│   ├── 两个 RT：累积色 (accumColor) + 透明度权重 (accumAlpha)
│   ├── 合成 Pass：dst = accumColor / (1 - accumAlpha)
│   ├── 权重函数选择：linear / exp / exp2（越远权重越小）
│   ├── 优点：单 Pass 收集 + 单 Pass 合成，移动端可跑
│   └── 缺点：近似，有色彩偏移，远处半透明可能消失
├── Per-Pixel Linked List (PPLL) OIT
│   ├── 原理：GPU 维护每像素的片元链表
│   ├── 数据结构：
│   │   ├── Head Pointer Texture（每像素指向第一个片元）
│   │   ├── Node Buffer（存储片元：color + depth + next指针）
│   │   └── Atomic Counter（分配 node 索引）
│   ├── 排序：像素着色器内对链表做插入排序
│   ├── 优点：精确排序，质量最高
│   └── 缺点：显存开销大，移动端不支持（无 Atomic Counter）
├── Depth Peeling
│   ├── 原理：每 Pass 剥离最前面一层半透明
│   ├── Dual Depth Peeling：同时从前后剥离，Pass 数减半
│   ├── 优点：精确，不依赖 UAV
│   └── 缺点：N 层需要 N 个 Pass，性能差
├── Alpha to Coverage
│   ├── 原理：将 Alpha 映射到 MSAA 采样掩码
│   ├── 适用：头发卡片、树叶等高频细节半透明
│   ├── 优点：硬件自动处理深度，无需排序
│   └── 缺点：需要开 MSAA，边缘有锯齿
├── 混合策略（实际项目常用）
│   ├── 头发 → Alpha to Coverage（4x MSAA）
│   ├── 大面积半透明（裙摆/翅膀）→ WBOIT
│   ├── 特效粒子 → 保持传统排序（数量少可接受）
│   └── 关键特写镜头 → PPLL（桌面端高画质模式）
└── 性能对比
    ├── WBOIT：+1 RT，+1 Blit Pass，约 5% 开销
    ├── PPLL：+32 bytes/pixel node buffer，约 15% 开销
    └── Depth Peeling (4层)：+4 Pass，约 30% 开销
```

#### 代码实现

**WBOIT 实现核心（URP HLSL）：**

```hlsl
// === Pass 1: 收集半透明片元（修改半透明物体的 Shader）===
// 输出到两张 RT：
//   RT0: accumColor (RGBA16F) — 加权颜色 × alpha
//   RT1: accumWeight (R16F) — 累积权重

// 深度加权函数（越远权重越低）
float ComputeWeight(float3 color, float depth, float alpha)
{
    // 方案 A：指数衰减（McGuire & Bavoil 2013）
    float weight = clamp(0.03 / (1e-5 + pow(depth / 100, 4.0)), 1e-2, 3e3);

    // 方案 B：线性 + 亮度加权（对暗色半透明更友好）
    // float luminance = dot(color, float3(0.299, 0.587, 0.114));
    // float weight = alpha * max(1e-2, 3e3 * pow(1 - depth, 3));

    return weight;
}

// 半透明收集 Pass 像素着色器
half4 FragCollect(Varyings input) : SV_Target
{
    float depth = input.positionCS.z / input.positionCS.w; // NDC 深度
    half4 color = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, input.uv);
    color.a *= _Color.a; // 材质透明度

    float weight = ComputeWeight(color.rgb, depth, color.a);

    // 输出到 MRT
    // RT0: 加权颜色（premultiplied）
    half4 accumColor = half4(color.rgb * color.a, color.a) * weight;
    // RT1: 权重累积
    half accumWeight = weight;

    return accumColor; // 实际用 MRT 输出到两个 RT
}

// === Pass 2: 合成（全屏后处理）===
half4 FragComposite(Varyings input) : SV_Target
{
    half4 accumColor = SAMPLE_TEXTURE2D(_AccumColorTex, sampler_AccumColorTex, input.uv);
    half accumWeight = SAMPLE_TEXTURE2D(_AccumWeightTex, sampler_AccumWeightTex, input.uv).r;

    half3 finalColor;
    if (accumWeight > 0.0)
    {
        // 归一化加权颜色
        finalColor = accumColor.rgb / accumWeight;
    }

    // 与不透明背景混合
    // accumColor.a 是透明度覆盖率
    half revealage = saturate(accumColor.a);
    half3 backgroundColor = SAMPLE_TEXTURE2D(_CameraColorTexture, sampler_CameraColorTexture, input.uv).rgb;

    half3 result = lerp(finalColor, backgroundColor, revealance);
    return half4(result, 1.0);
}
```

**Per-Pixel Linked List OIT（桌面端，HLSL）：**

```hlsl
// 数据结构定义
struct TransparentNode
{
    float4 color;  // RGBA
    float  depth;
    uint   next;   // 下一个节点索引
};

RWStructuredBuffer<TransparentNode> NodeBuffer : register(u1);
RWTexture2D<uint> HeadPointer : register(u2);
RWByteAddressBuffer NodeCounter : register(u3);

// 收集 Pass：构建每像素链表
void FragCollectPPLL(float4 color, float depth, float2 pixelCoord)
{
    // 分配一个新节点
    uint nodeIndex;
    NodeCounter.InterlockedAdd(0, 1, nodeIndex);

    if (nodeIndex < MaxNodes)
    {
        // 填充节点数据
        NodeBuffer[nodeIndex].color = color;
        NodeBuffer[nodeIndex].depth = depth;

        // 原子操作更新链表头
        uint2 coord = uint2(pixelCoord);
        uint oldHead;
        InterlockedExchange(HeadPointer[coord], nodeIndex, oldHead);
        NodeBuffer[nodeIndex].next = oldHead;
    }
}

// 合成 Pass：像素内链表排序 + 正确混合
half4 FragCompositePPLL(float2 pixelCoord)
{
    uint nodeIndex = HeadPointer[uint2(pixelCoord)];

    // 收集所有节点（最多 N 个，通常 8~16）
    #define MAX_NODES 16
    TransparentNode nodes[MAX_NODES];
    int count = 0;
    while (nodeIndex != 0xFFFFFFFF && count < MAX_NODES)
    {
        nodes[count] = NodeBuffer[nodeIndex];
        nodeIndex = nodes[count].next;
        count++;
    }

    // 按深度排序（从远到近，冒泡排序）
    for (int i = 0; i < count - 1; i++)
    {
        for (int j = 0; j < count - i - 1; j++)
        {
            if (nodes[j].depth < nodes[j + 1].depth)
            {
                TransparentNode tmp = nodes[j];
                nodes[j] = nodes[j + 1];
                nodes[j + 1] = tmp;
            }
        }
    }

    // 从远到近 over 混合
    half3 backgroundColor = tex2D(_BackgroundTex, pixelCoord).rgb;
    half3 result = backgroundColor;
    for (int k = 0; k < count; k++)
    {
        half a = nodes[k].color.a;
        result = nodes[k].color.rgb * a + result * (1.0 - a);
    }

    return half4(result, 1.0);
}
```

### ⚡ 实战经验

1. **WBOIT 权重函数是灵魂**：不同的权重函数效果差异巨大。 McGuire 的 exp2 方案适合大多数场景，但在大量暗色半透明叠合时会偏色。
2. **头发用 Alpha to Coverage 先解决**：头发卡片是半透明排序的重灾区，但 A2C 可以让硬件处理深度，完全绕过排序问题。代价是需要开 MSAA。
3. **PPLL 的 Node Buffer 要预分配**：移动端不支持，桌面端也要限制 MaxNodes（16 够用）。超过上限的片元直接丢弃，否则显存爆炸。
4. **WBOIT 不要和非 OIT 半透明混用**：WBOIT 的半透明和传统排序半透明在同一帧里会有视觉冲突。要么全用 OIT，要么明确分层。
5. **实测性能：WBOIT 是性价比之王**：在骁龙 8 Gen2 上，WBOIT 的额外开销约 0.3ms，完全可接受。PPLL 在 RTX 3060 上约 1.2ms，移动端不可行。
6. **裙摆/披风用 Pre-integrated 方案**：如果裙摆是简单的单层半透明，CPU 排序就够了，不需要 OIT。OIT 用于多层重叠的复杂场景。

### 🎯 能力体检清单

- [ ] 能解释为什么 Alpha Blending 是 order-dependent 的吗？数学公式写出来。
- [ ] WBOIT 的权重函数如何影响渲染质量？你会怎么选权重函数？
- [ ] Per-Pixel Linked List 在移动端为什么不可用？具体的硬件限制是什么？
- [ ] Alpha to Coverage 的原理是什么？它和 Alpha Test 有什么区别？
- [ ] 如果项目里同时有头发、翅膀、粒子、水面四种半透明，你的 OIT 策略是什么？
- [ ] Depth Peeling 为什么被淘汰了？它的替代方案比它好在哪里？
- [ ] WBOIT 的色彩偏移问题有没有修正方案？（提示：McGuire 2013 vs McGuire 2020）

### 🔗 相关问题

- [头发各向异性高光：Kajiya-Kay 模型在角色头发中的应用](shader/hair-anisotropic-lighting)
- [Forward+ 聚类渲染：URP 如何高效处理大量点光源？](rendering/forward-plus-cluster)
- [HiZ 屏幕空间反射：粗糙表面的模糊反射如何实现？](rendering/hiz-screen-space-reflection)
- [角色溶解消失：如何用 Shader 实现可控的 burn-out 效果？](shader/dissolve-effect)
