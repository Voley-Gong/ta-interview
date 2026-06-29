---
title: "角色眼球注视渲染：如何实现注视点渲染（Foveated Rendering）降低GPU开销40%？"
category: "rendering"
level: 4
tags: ["Foveated Rendering", "VR/AR", "Eye Tracking", "LOD", "分辨率", "GPU优化"]
hint: "人眼只有中心凹区域高清——周边降分辨率渲染，配合眼动追踪动态跟随注视点"
related: ["rendering/temporal-anti-aliasing-taa", "optimization/mobile-gpu-occupancy-bottleneck", "rendering/forward-plus-cluster"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做的是 VR 游戏，GPU 预算非常紧张。目前 4K 单眼渲染到 90fps 需要 RTX 4070 以上。听说注视点渲染（Foveated Rendering）能降低 40% 的 GPU 像素着色开销。你来设计一个方案——包括眼动追踪数据获取、分辨率分区策略、以及边缘平滑过渡。」

这是字节 PICO、腾讯 XR 中心、Sony（PSVR2）等 VR/XR 团队 TA 面试的高阶题目。即使不做 VR，这个技术对理解分辨率控制和渲染优化也极有价值。

### ✅ 核心要点

1. **人眼视觉原理**：中心凹（Fovea）约 1-2° 视角内是高清区域，外围视觉分辨率急剧下降
2. **分区渲染策略**：中心区域全分辨率 → 中圈 1/2 分辨率 → 外围 1/4 分辨率
3. **眼动追踪驱动**：实时获取注视点坐标，动态调整高分辨率区域中心
4. **边缘过渡处理**：硬边切换有可见接缝，需要双线性上采样 + TAA 时序平滑
5. **硬件 vs 软件方案**：原生支持（VRS / Variable Rate Shading）vs 手动多 Pass 渲染

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：GPU 像素着色开销降低 40%，玩家几乎感知不到画质下降
     ↓
倒推 1：如何让画面不同区域有不同分辨率？
  ├── 方案 A：多 Pass 渲染不同 RT 再合成（软件方案，通用）
  ├── 方案 B：硬件 VRS（Variable Rate Shading，NVIDIA Turing+ / AMD RDNA2+）
  └── 方案 C：Tile-based 可变分辨率（移动端 Adreno/Mali 支持）
     ↓
倒推 2：如何确定高分辨率区域的中心？
  ├── 眼动追踪设备（Tobii / PICO 眼动模组）提供 gaze (x, y)
  ├── 无眼动设备 → 用头部朝向预测注视点（退化方案）
  └── 预测延迟补偿（眼动数据通常有 20-50ms 延迟）
     ↓
倒推 3：如何消除分区接缝？
  ├── 中心 RT：全分辨率，全屏
  ├── 中圈 RT：1/2 分辨率，全屏 → 上采样后只取中圈区域
  ├── 外圈 RT：1/4 分辨率，全屏 → 上采样后只取外圈区域
  └── 用径向权重 Mask 在三个 RT 之间混合
     ↓
倒推 4：如何处理时序抖动？
  ├── 眼动数据抖动 → 高清区域跳动
  ├── TAA 的历史帧不一致 → 闪烁
  └── 解决：注视点位置做 Kalman 滤波平滑
```

#### 知识点拆解（倒推树）

```
注视点渲染（Foveated Rendering）
├── 人眼视觉基础
│   ├── 中心凹（Fovea Centralis）
│   │   ├── 视角范围：约 1-2°（高清区）
│   │   ├── 视锥细胞密度：~150,000 / mm²
│   │   └── 对应像素：约 4K RT 的中心 200x200 像素
│   ├── 副中心区（Parafoveal）
│   │   ├── 视角：2°~5°
│   │   └── 可接受 1/2 分辨率
│   ├── 外围区（Peripheral）
│   │   ├── 视角：>5°
│   │   └── 可接受 1/4 甚至更低分辨率
│   └── 不可感知阈值
│       └── 对比度敏感度函数（CSF）决定降采样不可见范围
├── 眼动追踪（Eye Tracking）
│   ├── 硬件方案
│   │   ├── 红外瞳孔-角膜反射（Pupil-CR）
│   │   ├── Tobii 眼动模组（PC VR）
│   │   └── PICO / Quest Pro 内置眼动
│   ├── 数据接口
│   │   ├── gaze direction (normalized vector)
│   │   ├── gaze origin (eye position)
│   │   ├── convergence distance
│   │   └── confidence / tracking quality
│   └── 延迟与补偿
│       ├── 感知延迟：~50ms
│       ├── 硬件延迟：20-50ms
│       └── Kalman 滤波 / 指数平滑
├── 渲染策略
│   ├── 软件多 RT 方案（通用）
│   │   ├── Pass 1：全分辨率渲染中心区域（scissor rect 裁剪）
│   │   ├── Pass 2：1/2 分辨率渲染全屏 → 上采样取中圈
│   │   ├── Pass 3：1/4 分辨率渲染全屏 → 上采样取外圈
│   │   └── Composite Pass：径向权重混合
│   ├── 硬件 VRS 方案
│   │   ├── NVIDIA VRS（DirectX 12 VariableRateShading）
│   │   ├── Tier 1：按 Draw Call 设置 shading rate
│   │   ├── Tier 2：per-tile shading rate（screen-space tile）
│   │   └── SV_ShadingRate：per-primitive rate
│   └── 移动端方案
│       ├── Adreno Foveated Rendering（Qualcomm extension）
│       └── Mali Fragment Density Map（ARM extension）
├── 过渡与平滑
│   ├── 空间过渡
│   │   ├── 径向 Mask（内圈权重 1 → 外圈权重 0）
│   │   ├── smoothstep 过渡带（避免硬边）
│   │   └── 各向异性调整（水平方向比垂直方向更敏感）
│   ├── 时序平滑
│   │   ├── TAA 历史帧复用（注意注视点移动导致的历史帧不匹配）
│   │   ├── 注视点速度感知：快速扫视时暂停 Foveated（saccade detection）
│   │   └── Kalman 滤波注视点位置
│   └── 上采样质量
│       ├── 双线性（快但有模糊）
│       ├── 对比度自适应上采样（CAS）
│       └── 机器学习超分（NVIDIA DLSS / AMD FSR）
└── 性能分析
    ├── 像素着色器调用次数
    │   ├── 全分辨率：100%（基准）
    │   ├── Foveated：约 35-45%（取决于分区策略）
    │   └── 带宽影响：纹理采样减少 → 带宽也降低
    └── 成本对比
        ├── 多 RT 方案：额外 DrawCall + 合成 Pass
        └── VRS 方案：零额外 DrawCall（硬件级）
```

#### 代码实现

**软件方案：多 RT Foveated Rendering（URP Renderer Feature）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class FoveatedRenderFeature : ScriptableRendererFeature
{
    public RenderPassEvent passEvent = RenderPassEvent.BeforeRenderingPostProcessing;
    public FoveatedSettings settings = new FoveatedSettings();

    private FoveatedRenderPass _pass;

    public override void Create()
    {
        _pass = new FoveatedRenderPass(settings)
        {
            renderPassEvent = passEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.foveatedMaterial == null) return;
        _pass.Setup(renderer.cameraColorTargetHandle);
        renderer.EnqueuePass(_pass);
    }
}

[System.Serializable]
public class FoveatedSettings
{
    public Material foveatedMaterial;
    [Range(0.02f, 0.3f)] public float foveaRadius = 0.12f;
    [Range(0.1f, 0.5f)] public float midRadius = 0.28f;
    [Range(0.1f, 0.3f)] public float transitionWidth = 0.15f;
    [Range(1, 4)] public int outerDownsample = 4;
    public bool smoothGaze = true;
}

public class FoveatedRenderPass : ScriptableRenderPass
{
    private FoveatedSettings _settings;
    private RTHandle _colorTarget;
    private RTHandle _lowResRT;
    private RTHandle _midResRT;

    private static readonly int FoveaRadiusID = Shader.PropertyToID("_FoveaRadius");
    private static readonly int MidRadiusID = Shader.PropertyToID("_MidRadius");
    private static readonly int TransitionWidthID = Shader.PropertyToID("_TransitionWidth");
    private static readonly int GazePointID = Shader.PropertyToID("_GazePoint");
    private static readonly int LowResTexID = Shader.PropertyToID("_LowResTex");
    private static readonly int MidResTexID = Shader.PropertyToID("_MidResTex");

    private Vector2 _smoothedGaze = new Vector2(0.5f, 0.5f);

    public FoveatedRenderPass(FoveatedSettings settings) => _settings = settings;
    public void Setup(RTHandle colorTarget) => _colorTarget = colorTarget;

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        desc.depthBufferBits = 0;

        var midDesc = desc;
        midDesc.width /= 2;
        midDesc.height /= 2;
        RenderingUtils.ReAllocateIfNeeded(ref _midResRT, midDesc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_FoveatedMidRes");

        var lowDesc = desc;
        lowDesc.width /= _settings.outerDownsample;
        lowDesc.height /= _settings.outerDownsample;
        RenderingUtils.ReAllocateIfNeeded(ref _lowResRT, lowDesc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_FoveatedLowRes");
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        var cmd = CommandBufferPool.Get("FoveatedRendering");

        Vector2 gaze = GetGazePoint(ref renderingData);
        if (_settings.smoothGaze)
        {
            _smoothedGaze = Vector2.Lerp(_smoothedGaze, gaze, 0.3f);
            gaze = _smoothedGaze;
        }

        Blitter.BlitCameraTexture(cmd, _colorTarget, _midResRT);
        Blitter.BlitCameraTexture(cmd, _colorTarget, _lowResRT);

        cmd.SetGlobalTexture(LowResTexID, _lowResRT);
        cmd.SetGlobalTexture(MidResTexID, _midResRT);
        cmd.SetGlobalVector(GazePointID, gaze);
        cmd.SetFloat(FoveaRadiusID, _settings.foveaRadius);
        cmd.SetFloat(MidRadiusID, _settings.midRadius);
        cmd.SetFloat(TransitionWidthID, _settings.transitionWidth);

        Blitter.BlitCameraTexture(cmd, _colorTarget, _colorTarget, _settings.foveatedMaterial, 0);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    private Vector2 GetGazePoint(ref RenderingData renderingData)
    {
        // 实际项目中替换为眼动追踪 SDK 的数据
        // 例：TobiiXR.GetGazeRay() 或 PICO Eye Tracking API
        #if UNITY_EDITOR
        var mouse = Input.mousePosition;
        var cam = renderingData.cameraData.camera;
        if (cam != null)
        {
            mouse.z = cam.nearClipPlane;
            var vp = cam.ScreenToViewportPoint(mouse);
            return new Vector2(vp.x, vp.y);
        }
        #endif
        return new Vector2(0.5f, 0.5f);
    }

    public override void OnCameraCleanup(CommandBuffer cmd) { }

    public void Dispose()
    {
        _lowResRT?.Release();
        _midResRT?.Release();
    }
}
```

**Foveated Composite Shader：**

```hlsl
Shader "Hidden/FoveatedComposite"
{
    Properties { _MainTex ("Texture", 2D) = "white" {} }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_MainTex);      SAMPLER(sampler_MainTex);
            TEXTURE2D(_MidResTex);     SAMPLER(sampler_MidResTex);
            TEXTURE2D(_LowResTex);     SAMPLER(sampler_LowResTex);

            float _FoveaRadius;
            float _MidRadius;
            float _TransitionWidth;
            float2 _GazePoint;

            struct Attributes { float4 positionOS : POSITION; float2 uv : TEXCOORD0; };
            struct Varyings { float4 positionHCS : SV_POSITION; float2 uv : TEXCOORD0; };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                float2 uv = IN.uv;
                float aspect = _ScreenParams.x / _ScreenParams.y;

                // 宽高比修正后的距离
                float2 aspectUV = float2(uv.x * aspect, uv.y);
                float2 aspectGaze = float2(_GazePoint.x * aspect, _GazePoint.y);
                float dist = distance(aspectUV, aspectGaze);

                // 三区域径向权重
                float wHigh = 1.0 - smoothstep(_FoveaRadius, _FoveaRadius + _TransitionWidth, dist);
                float wMid  = smoothstep(_FoveaRadius, _FoveaRadius + _TransitionWidth, dist)
                            * (1.0 - smoothstep(_MidRadius, _MidRadius + _TransitionWidth, dist));
                float wLow  = smoothstep(_MidRadius, _MidRadius + _TransitionWidth, dist);

                float wSum = max(wHigh + wMid + wLow, 0.001);
                wHigh /= wSum; wMid /= wSum; wLow /= wSum;

                half4 cHigh = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv);
                half4 cMid  = SAMPLE_TEXTURE2D(_MidResTex, sampler_MidResTex, uv);
                half4 cLow  = SAMPLE_TEXTURE2D(_LowResTex, sampler_LowResTex, uv);

                return cHigh * wHigh + cMid * wMid + cLow * wLow;
            }
            ENDHLSL
        }
    }
}
```

**方案对比表：**

| 方案 | GPU 节省 | 额外开销 | 适用平台 | 实现难度 |
|------|----------|----------|----------|----------|
| 软件 3-RT | 30-40% | 2次 Blit + 合成 | 全平台 | 中 |
| 硬件 VRS Tier 2 | 40-50% | 近零 | DX12 / Vulkan | 低（API 调用） |
| Adreno Foveated | 35-45% | 零 | 骁龙专属 | 低 |
| 多 Pass Scissor | 20-30% | 额外 Pass | 全平台 | 中 |

### ⚡ 实战经验

- **扫视抑制（Saccade Suppression）**：人眼快速扫视时大脑会抑制视觉感知，这个窗口（~50ms）内可以安全降低全局分辨率
- **宽高比修正**：屏幕 UV 的 X 和 Y 比例不同，计算距离时必须修正，否则注视区域是椭圆而非圆
- **TAA 冲突**：注视点移动时，TAA 的历史帧来自不同分辨率区域，会产生闪烁——注视点快速移动时回退全分辨率
- **VR 双眼差异**：左右眼注视点可能不同（辐辏反射），通常取两个注视点的中点作为 fovea 中心
- **VRS Tier 1 vs Tier 2**：Tier 1 只能按 Draw Call 设置 rate（粒度太粗），Tier 2 的 per-tile 才是真正可用的方案

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不理解为什么外围可降分辨率 | 人眼视觉生理学 | 学习中心凹视觉和 CSF 函数 |
| 多 RT 合成后有可见接缝 | 权重过渡函数 | smoothstep + 过渡带宽度调参 |
| 眼动数据延迟导致注视点偏移 | 传感器延迟与预测 | Kalman 滤波 / saccade 预测 |
| 不知道硬件 VRS 怎么用 | DX12 VRS API | D3D12 VariableRateShading Tier |
| VR 下效果不稳定 | 双眼辐辏 / IPD | 双眼视觉和立体渲染管线 |

### 🔗 相关问题

- 没有