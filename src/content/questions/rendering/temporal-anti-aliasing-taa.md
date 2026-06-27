---
title: "面试官让你在URP中实现TAA：时序抗锯齿从原理到落地怎么做？"
category: "rendering"
level: 4
tags: ["TAA", "时序抗锯齿", "Velocity Buffer", "Jitter", "History Buffer", "URP", "后处理"]
hint: "TAA 的核心三件套——Halton 序列 Jitter 抖动 + Velocity Buffer 重建 + History Buffer 邻域裁剪，缺一个都会糊"
related: ["rendering/motion-vectors-velocity", "rendering/custom-post-processing-urp", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官说：

> "我们项目用的 Forward 渲染，MSAA 在移动端开销太大，FXAA 又糊。你能在 URP 下实现一套 TAA（时序抗锯齿）吗？说说你的思路和会踩哪些坑。"

这是米哈游、腾讯、字节 TA 面试中的高频题——TAA 是现代渲染管线的标配，理解它意味着你真正懂「时域信息复用」。

### ✅ 核心要点

1. **Jitter（抖动）**：每帧用 Halton 序列偏移投影矩阵，让相机采样位置亚像素级变化
2. **Velocity Buffer（速度缓冲）**：记录每个像素的运动矢量，用于把历史帧像素对齐到当前帧
3. **History Buffer（历史缓冲）**：保存上一帧的融合结果，按速度重投影到当前帧
4. **Neighborhood Clamping（邻域裁剪）**：用当前帧 3×3 邻域的 min/max AABB 裁剪历史颜色，消除拖影
5. **Blend（融合）**：历史帧与当前帧按 α 混合（通常历史占 0.8~0.9）

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：无锯齿、无拖影、无闪烁、无模糊
    ↑
融合阶段：currentPixel × (1-α) + historyPixel × α
    ↑                    ↑
邻域裁剪              重投影（Reprojection）
clamp history          用 Velocity Buffer
to 3x3 AABB            沿运动矢量回溯
    ↑                    ↑
当前帧颜色           History Buffer（上一帧结果）
    ↑                    ↑
Jitter 偏移投影       上一帧的 Blit 输出
    ↑
Halton(2,3) 序列
```

#### 知识点拆解（倒推树）

```
TAA 落地
├── 1. Camera Jitter
│   ├── Halton 序列生成（低差异序列）
│   ├── 投影矩阵偏移（ViewProjection matrix modification）
│   └── Jitter 消除：最终画面不能看到抖动
│       └── TAA Pass 之后必须 Unjitter（用非抖动矩阵重新投影）
│
├── 2. Velocity Buffer（Motion Vector）
│   ├── 双 Pass 方案：记录上一帧和当前帧的位置 → 差值
│   ├── 矩阵方案：PrevViewProj × CurrViewProjInverse × CurrentPosition
│   ├── 蒙皮动画物体：需要 GPU Skinning 的双位置输出
│   └── 移动端简化：只用相机运动矢量，忽略物体动画（接受一定拖影）
│
├── 3. History Buffer 管理
│   ├── 双缓冲 RT：Ping-Pong 交替读写
│   ├── 重投影：historyUV = currentUV + velocity
│   ├── 边界处理：超出屏幕的像素用当前帧颜色替代
│   └── RT 格式选择：RGBAHalf（HDR）或 RGBA8（LDR）
│
├── 4. 邻域裁剪（Resolve 核心）
│   ├── 3×3 邻域取 min/max → 构建 AABB 颜色包围盒
│   ├── clamp history color 到 AABB 内
│   ├── Variance Clipping：用均值+方差替代 min/max，更柔和
│   └── 移动端简化：只取 5 tap（十字采样）
│
├── 5. 融合权重 α
│   ├── 静态场景：α = 0.9（历史占比高，更平滑）
│   ├── 运动区域：根据 Velocity 大小降低 α（减少拖影）
│   ├── 新解锁区域：α = 0（完全用当前帧，避免重投影错误）
│   └── Tonemapping 空间：在 perceptual 空间做融合更稳定
│
└── 6. 移动端工程化
    ├── RT 分辨率：半分辨率 Velocity + 全分辨率 TAA Resolve
    ├── 与其他后处理冲突：TAA 必须在 Bloom 之前
    ├── GPU Driven 预算：约 1.5-2.0ms（骁龙 888 基准）
    └── 退化策略：低端机直接关 TAA，退回 FXAA
```

#### 代码实现

**1. Camera Jitter（C#）**

```csharp
// Halton 序列生成
public static Vector2 GenerateHaltonSeq(int index, int baseX = 2, int baseY = 3)
{
    float x = 0f, y = 0f;
    float invX = 1f / baseX, invY = 1f / baseY;
    int i = index;
    while (i > 0)
    {
        x += (i % baseX) * invX;
        i /= baseX;
        invX /= baseX;
    }
    i = index;
    while (i > 0)
    {
        y += (i % baseY) * invY;
        i /= baseY;
        invY /= baseY;
    }
    return new Vector2(x, y);
}

// 应用 Jitter 到投影矩阵
private void ApplyJitter(Vector2 jitter)
{
    Matrix4x4 projection = camera.projectionMatrix;
    float texelOffsetX = (jitter.x - 0.5f) / camera.pixelWidth;
    float texelOffsetY = (jitter.y - 0.5f) / camera.pixelHeight;
    projection.m02 += texelOffsetX * 2;  // 注意 NDC 空间 [-1,1]
    projection.m12 += texelOffsetY * 2;
    camera.projectionMatrix = projection;
}

// 每帧调用
int frameIndex = Time.frameCount;
Vector2 jitter = GenerateHaltonSeq((frameIndex % 16) + 1);
ApplyJitter(jitter);
// 保存 jitter 传给 Shader 用于 Velocity 计算
Shader.SetGlobalVector("_Jitter", jitter);
```

**2. Velocity Buffer（Shader: Vertex Stage）**

```hlsl
// 在渲染 Pass 的顶点着色器中输出两个位置
struct Varyings {
    float4 positionCS : SV_POSITION;
    float4 positionNDC : TEXCOORD0;  // 当前帧 NDC
    float4 prevPositionNDC : TEXCOORD1; // 上一帧 NDC
};

Varyings vert(Attributes input) {
    Varyings output;
    // 当前帧
    output.positionCS = TransformObjectToHClip(input.positionOS);
    output.positionNDC = output.positionCS;
    // 上一帧（用 PrevViewProj）
    float4 prevPositionWS = mul(UNITY_PREV_MATRIX_M, float4(input.positionOS, 1.0));
    output.prevPositionNDC = mul(UNITY_PREV_MATRIX_VP, prevPositionWS);
    return output;
}

// Fragment Stage 输出速度
float2 frag(Varyings input) : SV_Target {
    float2 currUV = input.positionNDC.xy / input.positionNDC.w;
    float2 prevUV = input.prevPositionNDC.xy / input.prevPositionNDC.w;
    float2 velocity = (currUV - prevUV) * 0.5; // 编码到 [0,1]
    return velocity;
}
```

**3. TAA Resolve（Shader: Fragment）**

```hlsl
TEXTURE2D(_CurrentFrame);     SAMPLER(sampler_CurrentFrame);
TEXTURE2D(_HistoryFrame);     SAMPLER(sampler_HistoryFrame);
TEXTURE2D(_VelocityTex);      SAMPLER(sampler_VelocityTex);

float4 ResolveTAA(float2 uv) {
    // 1. 采样当前帧
    float3 currentColor = SAMPLE_TEXTURE2D(_CurrentFrame, sampler_CurrentFrame, uv).rgb;

    // 2. 采样速度，重投影历史帧
    float2 velocity = SAMPLE_TEXTURE2D(_VelocityTex, sampler_VelocityTex, uv).rg;
    float2 historyUV = uv - velocity;
    
    // 边界检查
    bool inBounds = all(historyUV >= 0) && all(historyUV <= 1);
    float3 historyColor = SAMPLE_TEXTURE2D(_HistoryFrame, sampler_HistoryFrame, historyUV).rgb;
    historyColor = inBounds ? historyColor : currentColor;

    // 3. 邻域裁剪（Variance Clipping）
    float3 colorMin = currentColor;
    float3 colorMax = currentColor;
    float3 colorM1 = currentColor;
    float3 colorM2 = currentColor * currentColor;
    
    // 3x3 邻域
    float2 texelSize = _TexelSize.xy;
    [unroll] for (int x = -1; x <= 1; x++) {
        [unroll] for (int y = -1; y <= 1; y++) {
            if (x == 0 && y == 0) continue;
            float3 neighbor = SAMPLE_TEXTURE2D(_CurrentFrame, sampler_CurrentFrame,
                                               uv + float2(x, y) * texelSize).rgb;
            colorMin = min(colorMin, neighbor);
            colorMax = max(colorMax, neighbor);
            colorM1 += neighbor;
            colorM2 += neighbor * neighbor;
        }
    }
    
    // 方差裁剪
    float n = 9.0;
    float3 mean = colorM1 / n;
    float3 variance = (colorM2 / n) - (mean * mean);
    float3 sigma = sqrt(max(variance, 0.0));
    colorMin = max(colorMin, mean - 0.5 * sigma);
    colorMax = min(colorMax, mean + 0.5 * sigma);
    
    // clamp history
    historyColor = clamp(historyColor, colorMin, colorMax);

    // 4. 自适应融合权重
    float velMag = length(velocity);
    float alpha = lerp(0.9, 0.4, saturate(velMag * 20));  // 运动大→降权重
    if (!inBounds) alpha = 0.0;  // 越界→完全用当前帧

    // 5. 混合
    float3 resolvedColor = lerp(historyColor, currentColor, 1.0 - alpha);
    return float4(resolvedColor, 1.0);
}
```

**4. URP Renderer Feature 结构**

```csharp
class TAARenderFeature : ScriptableRendererFeature {
    private RTHandle m_HistoryRT_A;
    private RTHandle m_HistoryRT_B;
    private bool m_UseA = true;

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        // 1. 在 AfterRenderingOpaques 之后注入 Velocity Pass
        // 2. 在 BeforeRenderingPostProcessing 之前注入 TAA Resolve Pass
        // 3. Ping-Pong 交替使用 HistoryRT_A / HistoryRT_B
        var src = renderer.cameraColorTargetHandle;
        var historyRead = m_UseA ? m_HistoryRT_A : m_HistoryRT_B;
        var historyWrite = m_UseA ? m_HistoryRT_B : m_HistoryRT_A;
        
        taaPass.Setup(src, historyRead, historyWrite, velocityPass.VelocityRT);
        renderer.EnqueuePass(taaPass);
        m_UseA = !m_UseA;
    }
}
```

### ⚡ 实战经验

1. **拖影是最常见的问题**——先检查 Velocity Buffer 是否正确（蒙皮物体的速度特别容易出错），再调邻域裁剪的松紧度
2. **TAA 必须在 Tonemapping 之后做**——如果你在 Linear 空间做 TAA 再 Tonemap，高亮区域会闪烁
3. **Jitter 范围不要超过 1 pixel**——Halton 序列乘以 texelSize，不要直接乘以 2（NDC 空间陷阱）
4. **移动端降级方案**：Adreno 6xx 以下直接关 TAA 退回 FXAA，TAA 的 RT 读写开销在低端 GPU 上不划算
5. **TAA + 透明物体 = 灾难**：透明物体没有正确的 Velocity，要么排除透明物体不做 TAA，要么用 per-pixel linked list 做 OIT + TAA
6. **调试技巧**：先关闭 Jitter 看 Velocity Buffer 是否正确（静止相机时应该全黑），再加 Jitter 验证收敛

### 🎯 能力体检清单

| 检查项 | 如果答不上来... |
|--------|----------------|
| Halton 序列是什么？为什么要用它而不是随机？ | → 低差异序列（Low-Discrepancy Sequence）基础盲区 |
| Velocity Buffer 对蒙皮动画物体怎么生成？ | → GPU Skinning + 双 Pass 位置输出，骨骼动画矩阵理解不足 |
| 邻域裁剪为什么用 Variance Clipping 而不是 min/max？ | → 统计学在渲染中的应用，min/max 过于保守 |
| TAA 为什么会"鬼影"？如何减轻？ | → 重投影误差 + 融合权重过高，运动区域自适应降权 |
| TAA 和 Motion Blur 的顺序关系？ | → TAA 在前，Motion Blur 在后（或者共享 Velocity 一起做） |
| 移动端 TAA 的 RT 格式怎么选？ | → RGBAHalf（HDR 场景必须），带宽预算评估盲区 |
| TAA 之后画面还在抖怎么办？ | → Unjitter 步骤缺失，或 jitter 没有正确累积到 history |

### 🔗 相关问题

- [运动模糊与 TAA 的基石：Motion Vector 渲染](../rendering/motion-vectors-velocity.md)
- [URP 后处理管线自定义](../rendering/custom-post-processing-urp.md)
- [URP 自定义 Renderer Feature](../rendering/urp-renderer-feature.md)
- [半透明排序崩了：OIT 方案](../rendering/oit-transparency-order-independent.md)
