---
title: "URP下如何实现高质量运动模糊？"
category: "rendering"
level: 3
tags: ["运动模糊", "Motion Blur", "速度缓冲", "Velocity Buffer", "URP", "后处理", "Renderer Feature"]
hint: "核心是 Velocity Buffer（速度缓冲）的生成与基于速度方向的多采样模糊"
related: ["rendering/motion-vectors-velocity.md", "rendering/custom-post-processing-urp.md", "rendering/urp-renderer-feature.md"]
---

## 参考答案

### 🎬 场景描述

面试官给你看一段游戏竞速视频——高速漂移时画面产生了非常自然的运动模糊，车身、轮胎、背景都有方向性的拖影，但静止物体清晰锐利。然后说：

> "我们的竞速游戏在高速驾驶时缺乏速度感。你们能不能加一个运动模糊？不是全屏径向模糊那种廉价效果，要基于物体实际运动的真实运动模糊。URP 管线，目标主机平台。"

### ✅ 核心要点

1. **真正的 Motion Blur 需要速度缓冲**：每个像素的速度 = 当前帧位置 - 上一帧位置，需要 Motion Vectors（运动矢量）
2. **速度缓冲来源**：渲染每个物体时，用上一帧 ViewProjection 矩阵和当前帧矩阵的差值，计算出像素在屏幕空间的移动量
3. **模糊算法选择**：基于速度方向的多次采样（Max Filter / Gaussian along velocity），采样数与速度成正比——速度快多采样，速度低少采样
4. **URP 实现**：自定义 Renderer Feature，新增 Motion Vector Pass + Motion Blur Pass 两个 Pass
5. **关键难点**：动态物体（蒙皮角色）的顶点级运动矢量需要特殊处理；Tile 最大速度限制防止极端模糊
6. **性能预算**：主机端 1-2ms（1080p），采样数自适应（根据 GPU 时间反馈动态调整）

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
高质量运动模糊 = 物体快速运动时，像素沿运动方向产生方向性拖影
     ↓ 倒推
拖影 = 沿速度方向多次采样颜色取平均（方向性模糊）
     ↓ 倒推
速度方向 = 每个像素的屏幕空间速度向量（存在 Velocity Buffer 中）
     ↓ 倒推
Velocity Buffer = (当前帧 Clip Space 位置 - 上一帧 Clip Space 位置) → 屏幕 UV 差
     ↓ 倒推
需要上一帧的 ViewProjection 矩阵（_PrevViewProjMatrix）
     ↓ 倒推
物体不动（静态场景）→ 相机运动产生速度；物体动（动态角色）→ 需要上一帧顶点位置
```

#### 知识点拆解（倒推树）

```
URP 高质量 Motion Blur
├── Pass 1: Motion Vector Pass（生成速度缓冲）
│   ├── 静态场景
│   │   ├── Vertex: 当前帧 MVP → Clip Pos
│   │   ├── Vertex: 上一帧 _PrevViewProj × 上一帧 ObjectToWorld → Prev Clip Pos
│   │   └── Velocity = (CurNDC - PrevNDC) → 存入 RT
│   ├── 动态物体（刚体平移/旋转）
│   │   ├── 需要上一帧的 ObjectToWorld 矩阵
│   │   └── 相机不变，物体矩阵变 → 速度 = 两者差
│   ├── 蒙皮角色（顶点动画）
│   │   ├── 需要"上一帧的骨骼蒙皮位置" → 额外 Pass 预计算
│   │   └── 或退化为物体级速度（不够精确但可接受）
│   └── 输出格式：RG 通道存速度 (Float2)，B 通道存物体 ID（用于排除背景）
│
├── Pass 2: Motion Blur Pass（后处理模糊）
│   ├── 采样速度缓冲
│   │   ├── 当前像素速度向量
│   │   └── 中心采样 + 邻域采样（Tile 内最大速度）
│   ├── 沿速度方向采样
│   │   ├── 在速度向量 ±方向上 N 个位置采样 Scene Color
│   │   ├── 采样数 = clamp(length(velocity) × scale, min, max)
│   │   └── 加权平均（中心权重高，边缘递减）
│   ├── Tile 最大速度优化
│   │   ├── 分 Tile 计算最大速度 → 决定采样数
│   │   └── 避免每个像素独立判断采样数（性能浪费）
│   └── 背景保护：物体 ID 不匹配时不做模糊
│
├── URP 集成
│   ├── 自定义 Renderer Feature
│   │   ├── Motion Vector RT（R16G16B16A16_Float 或 RG16）
│   │   ├── Scene Color 在 Motion Blur 之前（Opaque 之后）
│   │   └── Blur 结果写回 Camera Color RT
│   └── 矩阵管理
│       ├── 每帧末尾保存 _CurrentViewProj → 下帧的 _PrevViewProj
│       └── 物体级：每帧末尾保存 ObjectToWorld → 下帧的 PrevObjectToWorld
│
└── 质量调优
    ├── 模糊强度（全局缩放速度）
    ├── 最大/最小采样数
    ├── 软边衰减（采样端点权重衰减）
    └── 相机抖动/快门速度模拟
```

#### 代码实现

**1. Motion Vector Shader（生成速度缓冲）**

```hlsl
// MotionVector.shader - URP 兼容
Shader "Hidden/MotionVector"
{
    Properties { }
    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }
        
        Pass
        {
            Name "MotionVector"
            ZWrite Off ZTest LEqual Cull Back
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            CBUFFER_START(UnityPerFrame)
                float4x4 _PrevViewProjMatrix;  // 上一帧 VP 矩阵
            CBUFFER_END
            
            // 上一帧物体的 ObjectToWorld（每个动态物体需要记录）
            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float4x4, _PrevObjectToWorld)
            UNITY_INSTANCING_BUFFER_END(Props)
            
            struct Attributes {
                float4 positionOS : POSITION;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };
            
            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 curNDC : TEXCOORD0;
                float2 prevNDC : TEXCOORD1;
            };
            
            Varyings vert(Attributes i)
            {
                Varyings o;
                UNITY_SETUP_INSTANCE_ID(i);
                
                // 当前帧位置
                o.positionCS = TransformObjectToHClip(i.positionOS.xyz);
                o.curNDC = o.positionCS.xy / o.positionCS.w;
                
                // 上一帧位置
                float4x4 prevObjToWorld = UNITY_ACCESS_INSTANCED_PROP(Props, _PrevObjectToWorld);
                float3 prevWorldPos = mul(prevObjToWorld, float4(i.positionOS.xyz, 1.0)).xyz;
                float4 prevClipPos = mul(_PrevViewProjMatrix, float4(prevWorldPos, 1.0));
                o.prevNDC = prevClipPos.xy / prevClipPos.w;
                
                return o;
            }
            
            half4 frag(Varyings i) : SV_Target
            {
                // 屏幕空间速度（NDC → UV 空间）
                float2 velocity = (i.curNDC - i.prevNDC) * 0.5;  // NDC[-1,1] → UV[0,1] 差值
                return half4(velocity, 0, 1);
            }
            ENDHSLPROGRAM
        }
    }
}
```

**2. Motion Blur 后处理 Shader**

```hlsl
// MotionBlur.shader - 基于速度缓冲的模糊
Shader "Hidden/MotionBlur"
{
    Properties { }
    SubShader
    {
        Pass
        {
            Name "MotionBlur"
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            TEXTURE2D(_SceneColor);     SAMPLER(sampler_SceneColor);
            TEXTURE2D(_MotionVectorTex); SAMPLER(sampler_MotionVectorTex);
            
            float _BlurStrength;
            float _MaxSampleCount;
            float _MinSampleCount;
            
            half4 frag(Varyings i) : SV_Target
            {
                float2 uv = i.uv;
                
                // 采样速度缓冲
                float2 velocity = SAMPLE_TEXTURE2D(_MotionVectorTex, sampler_MotionVectorTex, uv).rg;
                velocity *= _BlurStrength;
                
                float speed = length(velocity);
                if (speed < 0.0001)
                    return SAMPLE_TEXTURE2D(_SceneColor, sampler_SceneColor, uv);
                
                // 采样数与速度成正比
                int sampleCount = (int)clamp(speed * 200, _MinSampleCount, _MaxSampleCount);
                
                // 沿速度方向采样
                half3 color = 0;
                float totalWeight = 0;
                
                for (int s = 0; s < sampleCount; s++)
                {
                    float t = (float)s / (sampleCount - 1) - 0.5;  // [-0.5, 0.5]
                    float2 sampleUV = uv + velocity * t;
                    
                    // 中心权重高，边缘递减（Gaussian-like）
                    float weight = exp(-t * t * 4.0);
                    
                    color += SAMPLE_TEXTURE2D(_SceneColor, sampler_SceneColor, sampleUV).rgb * weight;
                    totalWeight += weight;
                }
                
                return half4(color / totalWeight, 1.0);
            }
            ENDHSLPROGRAM
        }
    }
}
```

**3. URP Renderer Feature（C#）**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class MotionBlurFeature : ScriptableRendererFeature
{
    public RenderPassEvent motionVectorPassEvent = RenderPassEvent.AfterRenderingOpaques;
    public RenderPassEvent blurPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
    public Material motionVectorMat;
    public Material blurMat;
    
    private MotionVectorPass motionVectorPass;
    private MotionBlurPass blurPass;
    
    public override void Create()
    {
        motionVectorPass = new MotionVectorPass(motionVectorMat)
        {
            renderPassEvent = motionVectorPassEvent
        };
        blurPass = new MotionBlurPass(blurMat)
        {
            renderPassEvent = blurPassEvent
        };
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        // 跳过 SceneView（可选）
        if (renderingData.cameraData.cameraType == CameraType.Preview)
            return;
            
        renderer.EnqueuePass(motionVectorPass);
        renderer.EnqueuePass(blurPass);
    }
    
    // ===== Motion Vector Pass =====
    class MotionVectorPass : ScriptableRenderPass
    {
        private Material _mat;
        private RTHandle _motionVectorRT;
        private static readonly int _PrevViewProjID = Shader.PropertyToID("_PrevViewProjMatrix");
        private Matrix4x4 _prevViewProj;
        
        public MotionVectorPass(Material mat) { _mat = mat; }
        
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.depthBufferBits = 0;
            desc.colorFormat = RenderTextureFormat.RGHalf;
            
            RenderingUtils.ReAllocateIfNeeded(ref _motionVectorRT, desc, name: "_MotionVectorRT");
            cmd.SetGlobalTexture("_MotionVectorTex", _motionVectorRT);
        }
        
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("MotionVectorPass");
            
            // 设置上一帧 VP 矩阵
            cmd.SetGlobalMatrix(_PrevViewProjID, _prevViewProj);
            
            // 渲染所有标记了 Motion Vector 的物体（可以用特定 Layer）
            // 简化版：用 DrawRenderers 渲染 Opaque 物体
            var sortSettings = new SortingSettings(renderingData.cameraData.camera);
            var drawSettings = new DrawingSettings(new ShaderTagId("MotionVector"), sortSettings);
            var filterSettings = new FilteringSettings(RenderQueueRange.opaque);
            
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
            
            context.DrawRenderers(renderingData.cullResults, ref drawSettings, ref filterSettings);
        }
        
        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            // 保存当前帧 VP 供下一帧使用
            var cam = Camera.current;
            if (cam != null)
                _prevViewProj = cam.projectionMatrix * cam.worldToCameraMatrix;
        }
    }
    
    // ===== Motion Blur Pass =====
    class MotionBlurPass : ScriptableRenderPass
    {
        private Material _mat;
        private RTHandle _tempRT;
        
        public MotionBlurPass(Material mat) { _mat = mat; }
        
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.depthBufferBits = 0;
            RenderingUtils.ReAllocateIfNeeded(ref _tempRT, desc, name: "_MotionBlurTemp");
        }
        
        public override void Execute(ScriptableRenderPass context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("MotionBlurPass");
            var cameraColor = renderingData.cameraData.renderer.cameraColorTargetHandle;
            
            // Blit: SceneColor × MotionVectorTex → Temp → 复制回 CameraColor
            Blitter.BlitCameraTexture(cmd, cameraColor, _tempRT, _mat, 0);
            Blitter.BlitCameraTexture(cmd, _tempRT, cameraColor);
            
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
    
    protected override void Dispose(bool disposing)
    {
        _motionVectorRT?.Release();
        _tempRT?.Release();
    }
}
```

### ⚡ 实战经验

**实际项目中踩的坑和经验：**

1. **蒙皮角色的 Motion Vector 是最大的坑**——角色的顶点每帧都在动（骨骼蒙皮），如果用物体级速度代替，角色跑步时四肢不会产生正确的运动模糊。折中方案：额外渲染一个"上一帧骨骼姿势"的 Motion Vector Pass，或者干脆只对大型刚体（车辆）做精确 Motion Blur，角色用简化版

2. **透明物体无法做 Motion Blur**——Motion Vector Pass 是 Opaque 的。半透明粒子、UI、玻璃等不参与。需要在美术规范中明确

3. **矩阵精度问题**——`_PrevViewProjMatrix` 和当前帧的矩阵必须在完全相同的坐标系下。如果相机有抖动（TAA/抖动），需要用"未抖动"的矩阵。`float4x4` 精度在极端场景不够，考虑 `double` 预算再转 `float`

4. **Tile 最大速度优化是关键**——不做 Tile 优化的话，天空盒静止像素也要做全采样判断。先在 Compute Shader（或 Pixel Shader 的 Tile 级别）预计算每个 16×16 Tile 的最大速度，然后统一使用最大速度决定采样数

5. **竞速游戏的特殊需求**——车辆后方做"径向模糊增强"（虽然物理上不精确但视觉冲击力强），可以用径向模糊和 Motion Blur 混合，车速 > 150km/h 时加大径向模糊权重

6. **TAA 与 Motion Blur 的冲突**——TAA 本身也用 Motion Vector 做重投影。两者同时存在时，Motion Blur 应该在 TAA 之后做，否则模糊后的颜色做时域累积会产生模糊闪烁

### 🎯 能力体检清单

| 检查项 | 如果答不上来... |
|--------|----------------|
| Motion Vector 怎么计算？需要什么矩阵？ | 渲染矩阵体系不熟悉 |
| 上一帧矩阵怎么传递给下一帧？ | 渲染管线全局状态管理不了解 |
| 为什么蒙皮角色的 Motion Vector 特别难做？ | 蒙皮动画 × 渲染管线交互不理解 |
| URP 自定义 Renderer Feature 的完整流程？ | URP 扩展机制不熟 |
| Tile 最大速度优化解决什么问题？ | GPU 并行渲染 + 后处理优化不熟 |
| Motion Blur 和 TAA 谁先执行？为什么？ | 后处理管线顺序 × 时域累积不熟 |
| 速度缓冲用什么纹理格式？为什么？ | 精度范围 × 纹理格式选型不熟 |

### 🔗 相关问题

- [Motion Vectors / Velocity Buffer](rendering/motion-vectors-velocity.md) — 速度缓冲的基础概念
- [URP 自定义后处理](rendering/custom-post-processing-urp.md) — URP 后处理通用框架
- [URP 自定义 Renderer Feature](rendering/urp-renderer-feature.md) — Renderer Feature 基础
- [GPU-Driven Pipeline](rendering/gpu-driven-pipeline.md) — 大规模物体的 Motion Vector 管理思路
