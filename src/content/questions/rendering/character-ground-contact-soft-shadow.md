---
title: "角色脚部接地阴影：不 用平面投影，怎么用 Depth-Normal 做软接触阴影？"
category: "rendering"
level: 3
tags: ["接地阴影", "Contact Shadow", "软阴影", "Depth Normal", "URP", "角色落地", "场景渲染"]
hint: "平面投影只适用于平整地面——真实场景有起伏地形和楼梯，用屏幕空间射线步进沿视线方向检测遮挡"
related: ["rendering/planar-projection-shadow", "rendering/custom-screen-space-shadow-soften", "rendering/custom-ssao-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做开放世界手游，角色在地形、楼梯、斜坡上走动时，脚部和地面之间总是浮空或者投影shadow不贴合。平面投影阴影只能投到水平面上，遇到斜坡就穿模。给我一个屏幕空间 Contact Shadow 方案——从角色脚部沿光线方向做射线步进，碰到地面就生成软阴影。移动端要能跑。」

### ✅ 核心要点

1. **屏幕空间射线步进（Screen-Space Ray Marching）**：在屏幕空间从像素沿光照方向步进，检测深度缓冲是否被遮挡
2. **厚度上限控制**：Contact Shadow 只检测近距离遮挡（0.3~0.5m），避免穿模到远处
3. **软边缘**：步进结果做 Poisson Disk 模糊或线性渐变，避免硬边
4. **URP Custom Pass / Renderer Feature**：在 SSAO 之后、不透明物体之前注入
5. **移动端优化**：降低步进次数（8~16步）、半分辨率渲染、只对角色周围区域执行

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色站在斜坡/楼梯/不平地形上 → 脚下出现贴合地面的软阴影 → 没有浮空感
                ↑
倒推1：不能用平面投影（只适配水平面）→ 需要屏幕空间方案
倒推2：屏幕空间射线步进 → 从当前像素出发，沿光照方向步进 → 检测是否被角色遮挡
倒推3：步进需要深度缓冲 → 重建世界空间位置 → 投影到光照方向 → 比较深度差
倒推4：厚度上限 → 只步进 0.3~0.5m 范围，超出认为不是 Contact Shadow
倒推5：软边缘 → 步进次数内做线性插值，或后处理模糊
倒推6：性能 → 半分辨率 RT，只对角色周围 N 米的屏幕区域执行
```

#### 知识点拆解（倒推树）

```
屏幕空间 Contact Shadow
├── 核心算法：屏幕空间射线步进
│   ├── 深度缓冲重建世界坐标（_CameraDepthTexture → ViewPos → WorldPos）
│   ├── 光照方向步进（沿 _LightDir 步进 N 步，每步检测深度差）
│   ├── 遮挡判定（步进点深度 > 场景深度 → 被遮挡 → 有阴影）
│   └── 阴影强度计算（第一次遮挡的距离 → 近遮蔽深，远遮蔽浅）
├── 参数调优
│   ├── 步进次数（移动端 8~16，PC 32~64）
│   ├── 步进距离上限（Contact Shadow 厚度 0.3~0.5m）
│   ├── 步长策略（等距 vs 对数 vs 早期退出）
│   └── 偏移修正（bias 防 self-shadowing）
├── 软边缘处理
│   ├── 步进过程中的渐变（最后一个遮挡点的距离比例）
│   ├── Poisson Disk 旋转采样（多方向采样取平均）
│   └── 双边滤波（保边模糊，不跨边缘）
├── URP 集成
│   ├── Renderer Feature 注入点（After Opaque，在 SSAO 之后）
│   ├── 深度纹理获取（_CameraDepthTexture，需开启 Depth Texture）
│   ├── 法线纹理（_CameraNormalsTexture，可选用于过滤）
│   └── Blit 回主 RT（ fullscreen pass）
├── 移动端优化策略
│   ├── 半分辨率 RT 渲染（RT 宽高 / 2）
│   ├── 限制执行区域（scissor rect 只渲染角色周围屏幕区域）
│   ├── 降步进次数（8 步粗检 + 4 步精检 = hybrid）
│   ├── Shader 变体（高/中/低画质三档）
│   └── 距离衰减（远距离角色直接跳过 Contact Shadow）
└── 与其他阴影系统配合
    ├── 主级联阴影（CSM）远距离阴影，Contact Shadow 近距离补足
    ├── 角色自定义阴影（Per-Object Shadow）+ Contact Shadow 叠加
    └── 烘焙光照场景中的动态角色阴影（没有实时阴影时用 Contact Shadow 替代）
```

#### 代码实现

**1. Contact Shadow Fullscreen Shader（HLSL）**

```hlsl
// ContactShadow.shader — 屏幕空间 Contact Shadow
Shader "Hidden/ContactShadow"
{
    Properties { _BlitTexture ("Source", 2D) = "white" {} }
    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }
        Pass
        {
            Name "ContactShadow"
            ZWrite Off ZTest Always Cull Off Blend Off

            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment ContactShadowFrag
            #pragma multi_compile _ _USE_NORMAL_REJECTION

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            float3 _ContactLightDir;   // 光照方向（世界空间，已归一化）
            float  _ContactMaxDistance; // 步进总距离（世界空间，如 0.5m）
            int    _ContactSteps;       // 步进次数
            float  _ContactBias;        // 偏移量（防自阴影）
            float  _ContactIntensity;   // 阴影强度

            // 屏幕空间 UV → 世界空间位置
            float3 ReconstructWorldPos(float2 uv, float depth01)
            {
                float3 worldPos = ComputeWorldSpacePosition(uv, depth01, UNITY_MATRIX_I_VP);
                return worldPos;
            }

            half4 ContactShadowFrag(VaryingsFullscreen input) : SV_Target
            {
                float2 uv = input.uv;

                // 采样当前像素深度
                float sceneDepth = SampleSceneDepth(uv);
                // 天空盒直接跳过
                if (sceneDepth >= 1.0) return half4(0, 0, 0, 0);

                // 重建世界空间位置
                float3 worldPos = ReconstructWorldPos(uv, sceneDepth);

                // 步进参数
                float3 lightDir = normalize(_ContactLightDir);
                float stepSize = _ContactMaxDistance / (float)_ContactSteps;
                float3 stepVec = lightDir * stepSize;

                float visibility = 1.0;
                float lastDiff = 0.0;

                [unroll(16)]
                for (int i = 1; i <= 16; i++) // 编译期展开，移动端用 8
                {
                    if (i > _ContactSteps) break;

                    // 沿光照方向步进
                    float3 samplePos = worldPos + stepVec * i + lightDir * _ContactBias;

                    // 投影到屏幕空间
                    float4 clipPos = mul(UNITY_MATRIX_VP, float4(samplePos, 1.0));
                    float3 ndcPos = clipPos.xyz / clipPos.w;
                    float2 sampleUV = ndcPos.xy * 0.5 + 0.5;
                    // 翻转 Y（平台差异）
                    sampleUV.y = _ProjectionParams.x > 0 ? 1.0 - sampleUV.y : sampleUV.y;

                    // 超出屏幕范围跳过
                    if (any(sampleUV < 0) || any(sampleUV > 1)) continue;

                    // 采样步进点的场景深度
                    float sampleDepth = SampleSceneDepth(sampleUV);
                    float3 sampledWorldPos = ReconstructWorldPos(sampleUV, sampleDepth);

                    // 深度差：步进点 Z 与场景 Z 的差
                    float depthDiff = sampledWorldPos.y - samplePos.y; // 用 Y 轴差近似高度差

                    // 如果场景点比步进点高（在光线方向上被遮挡）
                    if (depthDiff > 0 && depthDiff < stepSize * 2.0)
                    {
                        // 被遮挡：距离越近阴影越深
                        float occlusion = saturate(1.0 - (float)i / (float)_ContactSteps);
                        occlusion *= saturate(depthDiff / (stepSize * 2.0));
                        visibility -= occlusion * _ContactIntensity;
                        lastDiff = depthDiff;

                        // 早期退出
                        if (visibility < 0.01) { visibility = 0; break; }
                    }
                }

                visibility = saturate(visibility);

                // 输出：shadow mask（R 通道），其他通道留空
                return half4(visibility, 0, 0, 0);
            }
            ENDHLSL
        }
    }
}
```

**2. 合成到主画面（第二个 Blit Pass）**

```hlsl
// 将 Contact Shadow mask 叠加到最终画面
half4 CompositeFrag(VaryingsFullscreen input) : SV_Target
{
    float2 uv = input.uv;
    half3 srcColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv).rgb;
    half shadowMask = SAMPLE_TEXTURE2D_X(_ContactShadowTex, sampler_ContactShadowTex, uv).r;

    // shadowMask: 1 = 无阴影, 0 = 全阴影
    // 对最终颜色乘以 shadowMask（受阴影区域变暗）
    half3 shadowColor = half3(0.5, 0.55, 0.6); // 偏冷的阴影色
    half3 result = lerp(srcColor, srcColor * shadowColor, (1.0 - shadowMask) * 0.7);

    return half4(result, 1.0);
}
```

**3. C# Renderer Feature**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class ContactShadowFeature : ScriptableRendererFeature
{
    public Settings settings = new Settings();
    private ContactShadowPass _pass;

    [System.Serializable]
    public class Settings
    {
        public Material contactShadowMat;
        public Material compositeMat;
        public RenderPassEvent passEvent = RenderPassEvent.AfterRenderingOpaques;
        public int steps = 8;
        public float maxDistance = 0.5f;
        public float bias = 0.01f;
        public float intensity = 1.0f;
        public bool halfResolution = true;
    }

    public override void Create()
    {
        _pass = new ContactShadowPass(settings);
        _pass.renderPassEvent = settings.passEvent;
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.contactShadowMat == null) return;
        renderer.EnqueuePass(_pass);
    }
}

public class ContactShadowPass : ScriptableRenderPass
{
    private ContactShadowFeature.Settings _settings;
    private RTHandle _contactShadowRT;
    private int _rtID = Shader.PropertyToID("_ContactShadowTex");

    public ContactShadowPass(ContactShadowFeature.Settings settings)
    {
        _settings = settings;
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        if (_settings.halfResolution)
        {
            desc.width /= 2;
            desc.height /= 2;
        }
        desc.colorFormat = RenderTextureFormat.R8; // 只需要单通道
        desc.depthBufferBits = 0;

        RenderingUtils.ReAllocateIfNeeded(ref _contactShadowRT, desc, name: "_ContactShadowTex");
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("ContactShadow");

        // 设置 shader 参数
        Light mainLight = RenderSettings.sun;
        cmd.SetGlobalVector("_ContactLightDir", mainLight ? -mainLight.transform.forward : Vector3.down);
        cmd.SetGlobalFloat("_ContactMaxDistance", _settings.maxDistance);
        cmd.SetInt("_ContactSteps", _settings.steps);
        cmd.SetGlobalFloat("_ContactBias", _settings.bias);
        cmd.SetGlobalFloat("_ContactIntensity", _settings.intensity);

        // Blit 到 Contact Shadow RT
        Blitter.BlitCameraTexture(cmd, renderingData.cameraData.renderer.cameraColorTargetHandle,
            _contactShadowRT, _settings.contactShadowMat, 0);

        // 合成回主画面
        Blitter.BlitCameraTexture(cmd, _contactShadowRT,
            renderingData.cameraData.renderer.cameraColorTargetHandle, _settings.compositeMat, 0);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public override void OnCameraCleanup(CommandBuffer cmd)
    {
        // RT 由 ReAllocateIfNeeded 管理
    }
}
```

### ⚡ 实战经验

- **步进方向用世界空间，不是屏幕空间**：有些实现直接在屏幕空间步进 UV，这样在视角变化时阴影会不稳定。世界空间步进虽然每步要投影到屏幕，但结果视角一致
- **只检测竖直方向高度差是个 hack**：严格来说应该用光线-深度缓冲的精确交点检测，但 `sampledY - samplePosY` 的近似在大多数场景够用，而且快得多
- **半分辨率 + 双边滤波是移动端标配**：直接半分辨率会锯齿严重，在合成 pass 做一次轻量双边滤波可以解决 90% 的锯齿
- **角色周围限定区域执行**：开放世界场景不可能全屏跑 Contact Shadow。用 `Graphics.DrawMesh` 或 Scissor Rect 限制到角色脚部周围 200x200 像素区域
- **和 SSAO 叠加效果最佳**：Contact Shadow 解决「脚不沾地」的硬阴影，SSAO 解决角落环境遮蔽。两者叠加才是完整的接地方案
- **自阴影处理**：Bias 太大阴影会漏光，太小会自阴影。建议用法线拒绝（`#if _USE_NORMAL_REJECTION`，检查 dot(normal, lightDir)）而不是单纯加 Bias

### 🎯 能力体检清单

- [ ] 屏幕空间射线步进的基本原理是什么？（从像素沿光方向步进，检测深度差）
- [ ] 为什么 Contact Shadow 不能用平面投影阴影替代？（不平整地形、斜坡、楼梯）
- [ ] 深度缓冲如何重建世界空间坐标？（NDC → Inverse ViewProj）
- [ ] 移动端 8 步够用吗？步进次数和距离怎么权衡？（次数少=粗但快；距离短=精确但覆盖少）
- [ ] 如何避免自阴影？（Bias / 法线拒绝 / 早期深度差过滤）
- [ ] URP 中 Renderer Feature 的注入时机怎么选？（After Opaque，在 SSAO 后合成）
- [ ] 如果场景没有实时阴影（纯烘焙光照），Contact Shadow 能独立工作吗？（能，它只依赖深度缓冲）
- [ ] 半分辨率 RT 的锯齿怎么处理？（双边滤波 / Poisson Disk）

### 🔗 相关问题

- [平面投影阴影](rendering/planar-projection-shadow.md) — 传统水平面投影方案
- [自定义屏幕空间阴影柔化](rendering/custom-screen-space-shadow-soften.md) — 阴影边缘柔化方案
- [自定义 SSAO URP](rendering/custom-ssao-urp.md) — 环境遮蔽基础方案