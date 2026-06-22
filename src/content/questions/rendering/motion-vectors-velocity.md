---
title: "运动模糊与 TAA 的基石：如何实现 Motion Vector（速度矢量）渲染？"
category: "rendering"
level: 3
tags: ["Motion Vector", "Velocity Buffer", "运动模糊", "TAA", "URP", "GPU Instancing", "渲染管线"]
hint: "核心是「双 Pass 记录当前/历史位置 → 差值写入速度纹理」，是 TAA 和 Motion Blur 的共用底座"
related: ["rendering/urp-renderer-feature", "rendering/custom-post-processing-urp", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官给你看一段游戏 demo，角色移动时画面有明显抖动和拖影：

> "我们项目要上 TAA（时域抗锯齿），但 TAA 需要每像素的速度矢量来做重投影（Reprojection）。目前项目用的是 URP，你需要：
> 1. 实现一个 Motion Vector Pass，输出每像素的速度矢量到 RT
> 2. 静态物体速度为 0，动态物体（骨骼动画 + 物体移动）要有正确速度
> 3. 透明物体暂时不需要处理
> 4. 注意不要把整个场景重新渲染一遍，要有性能意识"

### ✅ 核心要点

1. **Motion Vector = 当前帧 Clip Space 位置 − 上一帧 Clip Space 位置**
2. **静态物体用 Camera 矩阵差即可**：不需要逐物体渲染，可以全屏后处理算
3. **动态物体需要双矩阵记录**：上一帧 Model Matrix + 当前帧 Model Matrix
4. **URP 中用 Custom Pass / Renderer Feature 注入**：在 Opaque 之后、后处理之前
5. **精度控制是关键**：速度通常很小，用 16-bit float 就够，别浪费带宽

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
TAA 需要逐像素速度
  ↓ 速度怎么算？
当前像素位置 - 上一帧同一空间点的位置
  ↓ 静态物体怎么处理？
静态物体不动，但相机会动 → 用上一帧 VP 和当前帧 VP 差值
  ↓ 动态物体怎么处理？
物体本身的 Model Matrix 变了 → 需要上一帧和当前帧两个 Model Matrix
  ↓ 怎么获取上一帧矩阵？
每帧末尾把当前 Camera VP 存下来，每个动态物体存上一帧 Model Matrix
  ↓ 两个 Pass 还是一个 Pass？
方案 A：全场景再渲染一遍（慢但精确）
方案 B：只渲染动态物体 + 全屏 Camera Motion Pass（快，推荐）
  ↓ 最终输出到什么格式？
RG 浮点纹理，R = 速度X，G = 速度Y（NDC 空间，范围很小）
```

#### 知识点拆解（倒推树）

```
Motion Vector 渲染
├── 数学基础
│   ├── Reprojection：WorldPos → 当前 VP → 当前 NDC
│   ├── 同一 WorldPos → 上一帧 VP → 上一帧 NDC
│   └── Velocity = CurNDC.xy - PrevNDC.xy
├── Camera Motion（静态物体）
│   ├── 不需要逐物体渲染
│   ├── 全屏 Pass：从 Depth Buffer 反推 World Position
│   ├── 用上一帧 ViewProjection 矩阵变换
│   └── 速度 = 差值
├── Object Motion（动态物体）
│   ├── 上一帧 Model Matrix 存储
│   ├── Vertex Shader 中同时计算两帧位置
│   ├── 需要传入 prevModelMatrix uniform
│   └── 蒙皮动画：需要上一帧骨骼矩阵
├── URP 实现
│   ├── ScriptableRendererFeature 注册 Pass
│   ├── Pass 位置：After Rendering Opaques
│   ├── RT 格式：ARGBHalf（或 RGHalf）
│   └── Depth 拷贝：复用 CameraDepth
├── 性能优化
│   ├── 只渲染动态物体（静态走全屏 Pass）
│   ├── 16-bit float 精度足够
│   ├── 跳过小物体（Screen-space 面积阈值）
│   └── 异步计算管线（Async Compute）
└── TAA 消费端
    ├── 用 Velocity 做 Neighborhood Clamping
    ├── 历史帧采样偏移
    └── Ghosting 消除
```

#### 代码实现

**URP Custom Pass：Camera Motion Vector（全屏 Pass）：**

```hlsl
// CameraMotionVectors.shader
// 从 Depth Buffer 反推世界坐标，再用上一帧 VP 变换
#pragma fragment Frag

TEXTURE2D(_CameraDepthTexture);
TEXTURE2D_FLOAT(_CameraMotionVectorTexture); // 输出 RT
float4x4 _PrevViewProjMatrix;   // 上一帧 ViewProjection
float4x4 _CurViewProjMatrix;    // 当前帧 ViewProjection（Non-Jittered）
float4x4 _InverseViewProjMatrix; // 当前帧 VP 的逆

float2 Frag(Varyings IN) : SV_Target
{
    // 1. 采样深度，重建 NDC
    float depth = SAMPLE_TEXTURE2D(_CameraDepthTexture, sampler_CameraDepthTexture, IN.uv).r;
    #if UNITY_REVERSED_Z
        depth = 1.0 - depth;
    #endif

    // Early-out：天空盒速度为 0
    if (depth >= 1.0) return float2(0, 0);

    // 2. NDC → 世界坐标
    float4 ndc = float4(IN.uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    float4 worldPos = mul(_InverseViewProjMatrix, ndc);
    worldPos.xyz /= worldPos.w;

    // 3. 当前帧 NDC
    float4 curClip = mul(_CurViewProjMatrix, float4(worldPos.xyz, 1.0));
    float2 curNDC = curClip.xy / curClip.w;

    // 4. 上一帧 NDC（同一世界坐标，但用上一帧 VP）
    float4 prevClip = mul(_PrevViewProjMatrix, float4(worldPos.xyz, 1.0));
    float2 prevNDC = prevClip.xy / prevClip.w;

    // 5. 速度 = 差值（NDC 空间）
    return curNDC - prevNDC;
}
```

**Object Motion Vector（动态物体 Vertex/Fragment）：**

```hlsl
// ObjectMotionVectors.shader
// 动态物体专用 Pass
// 需要材质中传入上一帧 Model Matrix

CBUFFER_START(UnityPerMaterial)
    float4x4 _PrevModelMatrix; // 上一帧 Model Matrix
CBUFFER_END

// 从 C# 每帧更新
// material.SetMatrix("_PrevModelMatrix", prevModelMatrices[rendererId]);

struct VaryingsMotion
{
    float4 positionCS : SV_POSITION;
    float2 curNDC : TEXCOORD0;
    float2 prevNDC : TEXCOORD1;
};

VaryingsMotion VertMotion(Attributes IN)
{
    VaryingsMotion OUT;

    // 当前帧位置
    float4 curPos = mul(UNITY_MATRIX_M, float4(IN.positionOS.xyz, 1.0));
    OUT.positionCS = mul(UNITY_MATRIX_VP, curPos);
    OUT.curNDC = OUT.positionCS.xy / OUT.positionCS.w;

    // 上一帧位置（同一顶点，用上一帧 Model Matrix）
    float4 prevPos = mul(_PrevModelMatrix, float4(IN.positionOS.xyz, 1.0));
    float4 prevCS = mul(UNITY_PREV_MATRIX_VP, prevPos);
    OUT.prevNDC = prevCS.xy / prevCS.w;

    return OUT;
}

float2 FragMotion(VaryingsMotion IN) : SV_Target
{
    // 速度 = 当前 NDC - 上一帧 NDC
    return IN.curNDC - IN.prevNDC;
}
```

**C# 端 URP Renderer Feature（关键框架）：**

```csharp
// MotionVectorRendererFeature.cs
public class MotionVectorRendererFeature : ScriptableRendererFeature
{
    class MotionVectorPass : ScriptableRenderPass
    {
        private RTHandle _motionVectorRT;
        private Material _cameraMotionMat;
        private Material _objectMotionMat;

        // 每帧需要存储的矩阵
        private Matrix4x4 _prevViewProj;
        private Dictionary<int, Matrix4x4> _prevModelMatrices;

        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            // 申请 RT：RGHalf 精度足够
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.colorFormat = RenderTextureFormat.RGHalf;
            desc.depthBufferBits = 0;
            RenderingUtils.ReAllocateIfNeeded(ref _motionVectorRT, desc);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData data)
        {
            var cmd = CommandBufferPool.Get("MotionVectors");

            // === Pass 1: Camera Motion（全屏）===
            cmd.SetRenderTarget(_motionVectorRT);
            // 从 Depth 反推世界坐标算静态物体速度
            cmd.DrawFullscreen(_cameraMotionMat, _motionVectorRT);

            // === Pass 2: Object Motion（仅动态物体）===
            // 只渲染注册了 prevModelMatrix 的物体
            foreach (var kvp in _prevModelMatrices)
            {
                var renderer = GetRenderer(kvp.Key);
                if (renderer == null) continue;
                cmd.DrawRenderer(renderer, _objectMotionMat);
            }

            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }

        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            // 帧末：当前 VP → 上一帧 VP
            _prevViewProj = _curViewProj;
        }
    }
}
```

**速度纹理可视化（Debug）：**

| 通道 | 含义 | 视觉表现 |
|------|------|----------|
| R | 水平速度 | 右移为红 |
| G | 垂直速度 | 下移为绿 |
| 纯黑 | 静态 | 无运动 |
| 高亮区 | 快速运动 | 高速物体/边缘 |

### ⚡ 实战经验

1. **上一帧矩阵别用 Jittered VP**：TAA 本身会做 Jitter 抖动，但 Motion Vector Pass 必须用 Non-Jittered 矩阵，否则速度矢量和抖动叠加会产生重影
2. **骨骼动画是深坑**：蒙皮动画的上一帧位置需要上一帧的骨骼矩阵，要么存 Bone Matrix Buffer、要么用 Motion Buffer 技术（G-Buffer 存位置）
3. **移动端可以用更糙的方案**：直接在 Post-Processing 中用 Depth + Camera 矩阵差只算 Camera Motion，跳过 Object Motion，性能省 50%
4. **Velocity 精度别用 RGBA32**：速度差值极小（0.001 级），8-bit 会丢失精度，必须用 Half（16-bit float）

### 🎯 能力体检清单

| 卡住的环节 | 盲区在哪 | 补习建议 |
|------------|----------|----------|
| 不知道怎么从 Depth 反推世界坐标 | 逆矩阵变换 / NDC 空间理解 | 复习投影矩阵推导，手推逆投影 |
| 动态物体速度不正确 | 上一帧 Model Matrix 传递 | 理解矩阵级联和 Per-Object 数据传递 |
| URP 中不知道在哪插 Pass | ScriptableRendererFeature 机制 | 官方文档 + 看 URP 源码 RenderPasses |
| TAA 还是有拖影 | 可能 Jittered 矩阵混入 | 确认 Motion Pass 用的 Non-Jittered VP |
| 蒙皮动画 Motion 错乱 | 没处理上一帧骨骼矩阵 | 研究 Motion Buffer / Previous Pose 方案 |

### 🔗 相关问题

- [URP 自定义 Renderer Feature](rendering/urp-renderer-feature.md) — Motion Vector Pass 的基础设施
- [URP 自定义后处理](rendering/custom-post-processing-urp.md) — 消费 Motion Vector 的后效
- 蒙皮动画的 Motion Vector 如何优化？（延展：GPU Skinning + Previous Pose Buffer）
