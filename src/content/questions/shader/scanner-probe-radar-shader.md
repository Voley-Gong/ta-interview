---
title: "战术扫描探测 Shader：如何实现科幻 RTS 风格的雷达扫描圈？"
category: "shader"
level: 3
tags: ["扫描线", "雷达", "PostProcess", "Shader", "URP", "科幻"]
hint: "核心是极坐标变换 + 时间驱动扫描角度 + 扫描过的区域残留衰减发光——不是单纯的旋转纹理"
related: ["shader/hologram-projection-effect", "rendering/custom-post-processing-urp", "shader/fresnel-rim-light"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款科幻 RTS 手游，玩家在地图上点击某个位置后，需要从该点向外发射一圈可见的扫描波，扫描波经过的区域暂时高亮显示敌方单位（类似雷达扫描）。扫描波要有一圈明亮的前沿，扫过的区域有渐变残留发光。给我 URP 下的实现方案。」

### ✅ 核心要点

1. **极坐标扫描**：将屏幕 UV 或世界坐标转换为以扫描中心为原点的极坐标（angle, radius）
2. **时间驱动旋转**：扫描角度 = `fmod(_Time.y * _Speed, 2π)`，前沿带 = 角度差小于阈值的高亮区域
3. **径向衰减传播**：扫描波半径随时间从 0 扩展到 `_MaxRadius`，形成扩散圈
4. **残留发光**：用一张 RenderTexture 记录已扫描区域，随时间衰减（类似热衰减残影）
5. **敌方高亮**：扫描波经过敌方单位时，在后处理中叠加高亮轮廓（需要深度/ID 区分敌我）

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：点击点 → 扩散的圆形扫描波 → 前沿明亮 → 扫过区域残留发光 → 敌方单位被点亮
                ↑
倒推1：需要「从中心向外扩散的圆」→ 极坐标 radius 与时间的关系
倒推2：需要「旋转扫描」→ 极坐标 angle 与 _Time 的关系，两种模式：
      A. 扩散圆模式：radius 从 0 → max，到边后消失（声纳脉冲）
      B. 旋转扇形模式：固定角度的扇形绕中心旋转（雷达扫描）
      → 本题需要 A+B 混合：扩散圆 + 旋转扫描线
倒推3：需要「前沿明亮带」→ 在波前 ± _BandWidth 范围叠加高亮
倒推4：需要「残留衰减」→ RenderTexture blend：每帧 _FadeRate × 旧 + 新扫描
倒推5：需要「敌方高亮」→ 采样深度/自定义 ID texture，区分敌我
倒推6：整合为 URP 后处理或 Blit Renderer Feature
```

#### 知识点拆解（倒推树）

```
战术扫描探测
├── 极坐标数学
│   ├── UV → 极坐标转换：atan2(dy, dx), length(d)
│   ├── 扫描中心可配（_ScanCenter 世界坐标投影到屏幕）
│   └── 角度归一化：将 [-π, π] 映射到 [0, 1] 处理循环
├── 扫描波前沿
│   ├── 扩散圆：currentRadius = _Time.y × _Speed % _MaxRadius
│   ├── 前沿检测：abs(distance - currentRadius) < _BandWidth
│   ├── smoothstep 软边缘：避免硬边
│   └── 旋转扇形（可选）：angleDiff = abs(angle - scanAngle) < _FovHalf
├── 残留发光系统
│   ├── RenderTexture 双缓冲 ping-pong
│   ├── 每帧 Blend：dest = dest × _FadeRate + newScan
│   ├── CommandBuffer / RendererFeature 执行 Blit
│   └── 衰减率控制残留时长（_FadeRate = 0.95 ≈ 20帧消失）
├── 敌我识别（高阶）
│   ├── 方案A：Replacement Shader 渲染 ID → ID Texture
│   ├── 方案B：Stencil Buffer 标记敌方
│   └── 扫描波 + ID 匹配 → 该位置高亮
├── URP 集成
│   ├── Renderer Feature → Blit（全屏后处理）
│   ├── ScriptableRenderPass 编写
│   ├── 材质参数暴露到 Shader Graph 或 HLSL
│   └── 移动端适配：降低 RT 分辨率（半分辨率 Blit）
└── 性能考量
    ├── RT 分辨率：半分辨率做扫描计算，最后 upscale 合成
    ├── Blend 操作成本：移动端 blend 带宽敏感
    └── 多次扫描波叠加：用数组参数或 RT 累加
```

#### 代码实现

**核心 Shader（HLSL，全屏 Blit 用）：**

```hlsl
// scanner_probe.shader
Shader "Hidden/ScannerProbe"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
        _ScanTex ("Scan Accumulation RT", 2D) = "black" {}
        _ScanCenter ("Scan Center (Screen UV)", Vector) = (0.5, 0.5, 0, 0)
        _ScanColor ("Scan Color", Color) = (0.2, 0.8, 1.0, 1.0)
        _MaxRadius ("Max Radius", Float) = 0.7
        _BandWidth ("Band Width", Float) = 0.03
        _Speed ("Expand Speed", Float) = 0.35
        _FadeRate ("Fade Rate", Range(0.8, 1.0)) = 0.94
    }

    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

    struct Attributes
    {
        float4 positionOS : POSITION;
        float2 uv : TEXCOORD0;
    };

    struct Varyings
    {
        float4 positionCS : SV_POSITION;
        float2 uv : TEXCOORD0;
    };

    Varyings Vert(Attributes input)
    {
        Varyings output;
        output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
        output.uv = input.uv;
        return output;
    }

    TEXTURE2D(_MainTex);    SAMPLER(sampler_MainTex);
    TEXTURE2D(_ScanTex);    SAMPLER(sampler_ScanTex);

    float4 _ScanCenter;
    float4 _ScanColor;
    float _MaxRadius;
    float _BandWidth;
    float _Speed;
    float _FadeRate;

    // 扫描波强度计算
    float ScanWave(float2 uv)
    {
        // 1. 计算到扫描中心的距离
        float2 d = uv - _ScanCenter.xy;
        float dist = length(d);

        // 2. 当前扫描波半径（循环扩展）
        float currentR = fmod(_Time.y * _Speed, _MaxRadius);

        // 3. 前沿带：在 currentR ± _BandWidth 范围高亮
        float band = smoothstep(_BandWidth, 0.0, abs(dist - currentR));

        // 4. 波尾衰减：已扫过区域亮度渐降
        float trail = smoothstep(currentR, currentR - 0.15, dist) * 0.3;

        return saturate(band + trail);
    }

    // 更新累积 RT（残留发光）
    half4 FragAccumulate(Varyings i) : SV_Target
    {
        float wave = ScanWave(i.uv);

        // 上一帧残留 × 衰减率 + 新扫描
        half4 prev = SAMPLE_TEXTURE2D(_ScanTex, sampler_ScanTex, i.uv);
        half3 accumulated = prev.rgb * _FadeRate + _ScanColor.rgb * wave;

        return half4(accumulated, 1.0);
    }

    // 最终合成（叠加到场景）
    half4 FragComposite(Varyings i) : SV_Target
    {
        half3 scene = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv).rgb;
        half3 scan = SAMPLE_TEXTURE2D(_ScanTex, sampler_ScanTex, i.uv).rgb;

        // Additive blend
        return half4(scene + scan * _ScanColor.a, 1.0);
    }
    ENDHLSL

    SubShader
    {
        // Pass 0: 更新累积 RT
        Pass
        {
            Name "Accumulate"
            ZWrite Off ZTest Always Cull Off
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragAccumulate
            ENDHLSL
        }
        // Pass 1: 最终合成
        Pass
        {
            Name "Composite"
            ZWrite Off ZTest Always Cull Off
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragComposite
            ENDHLSL
        }
    }
}
```

**C# 驱动（Renderer Feature 简化版）：**

```csharp
// ScannerProbeFeature.cs
public class ScannerProbeFeature : ScriptableRendererFeature
{
    public Material scanMaterial;
    public RenderPassEvent passEvent = RenderPassEvent.BeforeRenderingPostProcessing;

    private ScannerProbePass _pass;

    public override void Create()
    {
        _pass = new ScannerProbePass(scanMaterial)
        {
            renderPassEvent = passEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer,
        ref RenderingData renderingData)
    {
        if (scanMaterial == null) return;
        renderer.EnqueuePass(_pass);
    }
}

// ScannerProbePass.cs —— 核心 RT ping-pong 逻辑
public class ScannerProbePass : ScriptableRenderPass
{
    private Material _mat;
    private int _rtA = Shader.PropertyToID("_ScanAccumA");
    private int _rtB = Shader.PropertyToID("_ScanAccumB");
    private bool _useA = true;

    public ScannerProbePass(Material mat) => _mat = mat;

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData data)
    {
        var desc = data.cameraData.cameraTargetDescriptor;
        desc.depthBufferBits = 0;
        // 半分辨率降低带宽
        desc.width /= 2;
        desc.height /= 2;
        cmd.GetTemporaryRT(_rtA, desc);
        cmd.GetTemporaryRT(_rtB, desc);
    }

    public override void Execute(ScriptableRenderContext ctx, ref RenderingData data)
    {
        var cmd = CommandBufferPool.Get("ScannerProbe");

        int src = _useA ? _rtA : _rtB;
        int dst = _useA ? _rtB : _rtA;

        _mat.SetTexture("_ScanTex", src);

        // Pass 0: 累积更新到 dst
        cmd.Blit(null, dst, _mat, 0);

        // Pass 1: 合成到摄像机画面
        _mat.SetTexture("_ScanTex", dst);
        cmd.Blit(data.cameraData.renderer.cameraColorTargetHandle,
                 data.cameraData.renderer.cameraColorTargetHandle, _mat, 1);

        _useA = !_useA;

        ctx.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public override void OnCameraCleanup(CommandBuffer cmd)
    {
        cmd.ReleaseTemporaryRT(_rtA);
        cmd.ReleaseTemporaryRT(_rtB);
    }
}
```

### ⚡ 实战经验

1. **RT ping-pong 是关键**：扫描残留效果本质是「时间域累积」，每帧把上一帧结果衰减后加上新内容，双缓冲 RT 交替读写
2. **移动端半分辨率**：累积 RT 用半分辨率完全够看，最后合成时全屏叠加，带宽省 75%
3. **多波叠加**：如果策划要「同时存在 3 个扫描波」，不要开 3 个 Feature——用一个 `_ScanCenters[8]` 数组在 Shader 里循环
4. **与玩法联动**：扫描结果可以输出到 CPU（ReadPixel 或 ComputeBuffer），让逻辑层判断「哪些敌人被扫到了」，变成玩法机制而不仅是视觉效果
5. **替代方案**：如果不需要残留发光（只要前沿圈），可以用粒子系统 `TrailRenderer` 或 `ParticleSystem` 模拟，省掉 RT 操作

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能写出极坐标转换公式（atan2 + length） | → 数学基础盲区：复习向量数学 |
| 理解 ping-pong RT 为什么不能读写同一张纹理 | → GPU 渲染管线盲区：理解 GPU 并行模型 |
| 能在 URP 中编写自定义 Renderer Feature | → URP 扩展盲区：读 ScriptableRenderPass 文档 |
| 知道 smoothstep 在这里为什么优于 if-else | → Shader 编写盲区：理解分支成本 |
| 能解释 Blit 操作的输入输出 | → 渲染基础盲区：理解全屏后处理原理 |

### 🔗 相关问题

- [shader/hologram-projection-effect](../shader/hologram-projection-effect.md) — 全息投影中的扫描线效果
- [rendering/custom-post-processing-urp](../rendering/custom-post-processing-urp.md) — URP 后处理管线扩展
- [shader/fresnel-rim-light](../shader/fresnel-rim-light.md) — 边缘光菲涅尔效果
