---
title: "屏幕空间反射（SSR）：如何用 Ray Marching 实现角色盔甲的实时镜面反射？"
category: "shader"
level: 3
tags: ["SSR", "Ray Marching", "后处理", "反射", "URP", "屏幕空间"]
hint: "核心是屏幕空间光线步进 + Hi-Z 加速 + Roughness 控制反射模糊——别忘了边缘截断修复"
related: ["rendering/forward-plus-cluster", "rendering/urp-renderer-feature", "shader/water-caustics"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们做一个动作游戏，主角穿着金属盔甲，在潮湿的地牢场景中战斗。美术要求盔甲能反射周围环境的灯光和湿润地面的倒影。平面反射（Planar Reflection）不够灵活——地面不是平的，盔甲是曲面。你会怎么实现一套通用的屏幕空间反射方案？URP 下怎么落地？

这是米哈游、叠纸、腾讯光子等高品质项目的 TA 渲染方向面试题。考察点涵盖 Ray Marching、Hi-Z、Roughness 模糊、边缘修复等核心技术。

### ✅ 核心要点

1. **屏幕空间光线步进**：在屏幕空间对每个像素发射反射射线，步进采样深度缓冲找到碰撞点
2. **Hi-Z 加速**：逐级降采样深度图构建 Mip Pyramid，大步跨查询 → 小步精定位，从 O(N) 降到 O(logN)
3. **Roughness 驱动模糊**：不是所有反射都清晰——根据材质粗糙度对采样射线 jitter 并累积，模拟模糊反射
4. **边缘截断修复**：屏幕边缘信息缺失（射线打出屏幕外），需要用时空复用（TAA 思路）或 Cubemap 兜底
5. **性能预算控制**：SSR 是全屏后处理，移动端需激进降采样，主机/PC 可全分辨率 + Hi-Z

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：盔甲表面反射出环境（灯光、地面、墙壁），湿润地面有倒影
                ↑
倒推1：反射射线从摄像机视角的每个像素出发
     → 需要屏幕空间的位置重建（世界坐标 → 屏幕坐标）
倒推2：射线步进找到碰撞点
     → 需要深度缓冲 + 步进采样
倒推3：暴力步进太慢（1920x1080 × 64步 = 1.3亿次采样）
     → 需要 Hi-Z Mip Pyramid 加速
倒推4：粗糙表面的反射应该是模糊的
     → 需要基于 Roughness 的射线 Jitter + 多采样累积
倒推5：屏幕边缘没有信息（射线打到屏幕外）
     → 需要边缘遮罩 + Cubemap/Planar 兜底
倒推6：单帧 SSR 噪声大
     → 需要时域累积（TAA 思路：历史帧 + 当前帧 Blending）
```

#### 知识点拆解（倒推树）

```
屏幕空间反射（SSR）
├── 屏幕空间重建
│   ├── 深度图 → 世界坐标重建
│   │   ├── 方法1：Inverse ViewProjection 矩阵
│   │   └── 方法2：View Ray 插值 + depth * linearEye
│   ├── 法线图（G-Buffer 或 法线 Pass）
│   └── 反射向量计算：R = reflect(-V, N)
│
├── Ray Marching 核心
│   ├── 步进策略
│   │   ├── 固定步长（简单但慢）
│   │   ├── 基于深度的自适应步长（远距离大步）
│   │   └── Hi-Z 加速（最优）
│   ├── 碰撞检测
│   │   ├── 采样深度图 vs 射线深度
│   │   └── 厚度容忍（Thickness tolerance）避免穿透
│   └── 二分精定位（Binary Search 细化碰撞点）
│
├── Hi-Z 加速结构
│   ├── 构建：逐级取 max/min depth → Mip Chain
│   ├── 查询：从最粗 Mip 开始，大步跳 → 命中后降 Mim 级别精定位
│   └── 复杂度：O(屏幕分辨率) → O(log(最大步数))
│
├── 模糊反射（Glossy SSR）
│   ├── 基于 Roughness 的射线 Jitter
│   │   └── 在理想反射向量周围按圆锥角度抖动
│   ├── 多采样累积（Importance Sampling）
│   │   └── 每像素 4-8 条射线 → 平均
│   └── 时域累积（TAA 思路）
│       └── 历史帧反射结果 + 当前帧 → 指数移动平均
│
├── 边缘修复
│   ├── 屏幕边缘遮罩（Fade by screen UV distance to border）
│   ├── 射线出屏处理（Ray misses → 用 Cubemap 采样兜底）
│   └── 时域遮罩（Reject 历史帧中遮挡关系变化的像素）
│
└── URP 集成
    ├── ScriptableRendererFeature 挂载 SSR Pass
    ├── Blit 链：CameraColor → SSR Pass → Output
    ├── 深度图获取：CameraDepthTexture（需开启）
    ├── 法线图获取：自定义 Normals Pass 或 G-Buffer
    └── Compute Shader vs Fragment Shader 选择
```

#### 代码实现

**URP 下 SSR Renderer Feature 核心结构：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class SSRFeature : ScriptableRendererFeature {
    public SSRSettings settings = new SSRSettings();
    private SSRPass ssrPass;

    public override void Create() {
        ssrPass = new SSRPass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        // 在后处理阶段注入
        renderer.EnqueuePass(ssrPass);
    }
}

public class SSRPass : ScriptableRenderPass {
    private SSRSettings settings;
    private Material ssrMaterial;
    private RTHandle ssrRT;
    private RTHandle tempRT;
    private int hiZMipLevels;

    public SSRPass(SSRSettings settings) {
        this.settings = settings;
        renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        ssrMaterial = CoreUtils.CreateEngineMaterial(settings.ssrShader);
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData) {
        // 申请临时 RT
        var desc = renderingData.cameraData.cameraTargetDescriptor;
        desc.depthBufferBits = 0;
        
        // 降采样（性能控制）
        desc.width = Mathf.Max(1, desc.width / settings.resolutionScale);
        desc.height = Mathf.Max(1, desc.height / settings.resolutionScale);
        
        RenderingUtils.ReAllocateIfNeeded(ref ssrRT, desc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_SSRResult");
        RenderingUtils.ReAllocateIfNeeded(ref tempRT, desc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_SSRTemp");
        
        // 配置输入纹理
        ConfigureInput(ScriptableRenderPassInput.Depth | ScriptableRenderPassInput.Normal);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
        CommandBuffer cmd = CommandBufferPool.Get("SSR");
        
        // === Pass 1: Hi-Z 构建（Compute Shader） ===
        if (settings.enableHiZ) {
            cmd.SetComputeIntParam(settings.hiZCompute, "_MipLevels", hiZMipLevels);
            // 逐级降采样 depth texture → Hi-Z Mip Chain
            for (int i = 1; i <= hiZMipLevels; i++) {
                int srcMip = i - 1;
                cmd.SetComputeTextureParam(settings.hiZCompute, 0, "_SrcMip", ...);
                cmd.DispatchCompute(...);
            }
        }
        
        // === Pass 2: Ray Marching ===
        cmd.SetRenderTarget(ssrRT);
        ssrMaterial.SetTexture("_CameraDepthTexture", ...);
        ssrMaterial.SetTexture("_CameraNormalsTexture", ...);
        ssrMaterial.SetTexture("_CameraColorTexture", ...);
        ssrMaterial.SetFloat("_MaxSteps", settings.maxSteps);
        ssrMaterial.SetFloat("_StepSize", settings.stepSize);
        ssrMaterial.SetFloat("_Thickness", settings.thickness);
        ssrMaterial.SetFloat("_MaxDistance", settings.maxDistance);
        
        // 执行 Ray Marching（Draw Fullscreen Triangle）
        cmd.DrawMesh(RenderingUtils.fullscreenMesh, Matrix4x4.identity, ssrMaterial, 0, 0);
        
        // === Pass 3: 时域累积 + 边缘修复 ===
        cmd.SetRenderTarget(tempRT);
        ssrMaterial.SetTexture("_SSRResult", ssrRT);
        ssrMaterial.SetTexture("_SSRHistory", ssrHistory);
        ssrMaterial.SetFloat("_TemporalWeight", settings.temporalWeight);
        cmd.DrawMesh(RenderingUtils.fullscreenMesh, Matrix4x4.identity, ssrMaterial, 0, 1);
        
        // Blit 回 camera target
        Blitter.BlitCameraTexture(cmd, tempRT, renderingData.cameraColorTargetHandle);
        
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```

**SSR Ray Marching Shader（核心 Fragment）：**

```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

TEXTURE2D(_CameraNormalsTexture);  SAMPLER(sampler_CameraNormalsTexture);
TEXTURE2D(_CameraColorTexture);    SAMPLER(sampler_CameraColorTexture);
TEXTURE2D(_HiZDepth);              SAMPLER(sampler_HiZDepth);

float _MaxSteps;
float _StepSize;
float _Thickness;
float _MaxDistance;

struct Varyings {
    float4 positionHCS : SV_POSITION;
    float2 uv : TEXCOORD0;
    float3 viewDir : TEXCOORD1;  // 视空间方向
};

// 屏幕空间 → 视空间深度
float LinearEyeDepthSSR(float rawDepth) {
    return LinearEyeDepth(rawDepth, _ZBufferParams);
}

// 反射向量 → 屏幕空间步进
half4 SSRFragment(Varyings IN) : SV_Target {
    float2 uv = IN.uv;
    
    // 重建世界/视空间位置和法线
    float rawDepth = SampleSceneDepth(uv);
    if (rawDepth == 1.0) return half4(0,0,0,0); // 天空盒
    
    float3 viewPos = ComputeViewSpacePosition(uv, rawDepth, UNITY_MATRIX_I_P);
    float3 worldNormal = SAMPLE_TEXTURE2D(_CameraNormalsTexture, sampler_CameraNormalsTexture, uv).xyz;
    float3 viewNormal = TransformWorldToViewDir(worldNormal);
    viewNormal = normalize(viewNormal);
    
    // 计算反射方向（视空间）
    float3 viewDir = normalize(viewPos);
    float3 reflectDir = reflect(viewDir, viewNormal);
    
    // 如果反射朝向摄像机 → 无反射
    if (reflectDir.z > 0) return half4(0,0,0,0);
    
    // === Ray Marching（Hi-Z 加速） ===
    float3 rayPos = viewPos;
    float2 rayUV = uv;
    float2 prevUV = uv;
    bool hit = false;
    
    [unroll(4)]
    for (int i = 0; i < (int)_MaxSteps; i++) {
        // 步进（自适应步长：根据当前深度调整）
        float stepLen = _StepSize * (1.0 + abs(rayPos.z) * 0.05);
        rayPos += reflectDir * stepLen;
        
        // 视空间 → 裁剪空间 → UV
        float4 clipPos = mul(UNITY_MATRIX_P, float4(rayPos, 1.0));
        float3 ndc = clipPos.xyz / clipPos.w;
        rayUV = ndc.xy * 0.5 + 0.5;
        
        // 射线出屏 → 终止
        if (rayUV.x < 0 || rayUV.x > 1 || rayUV.y < 0 || rayUV.y > 1) break;
        
        // 采样深度图
        float sampledDepth = SampleSceneDepth(rayUV);
        float sampledViewZ = LinearEyeDepthSSR(sampledDepth);
        float rayViewZ = -rayPos.z; // 视空间 Z 取反（Unity 中正值）
        
        // 碰撞检测：射线深度 > 采样深度（射线在表面后面）
        float depthDiff = rayViewZ - sampledViewZ;
        if (depthDiff > 0 && depthDiff < _Thickness) {
            // 命中！采样颜色
            hit = true;
            break;
        }
        
        // 超过最大距离
        if (length(rayPos - viewPos) > _MaxDistance) break;
        
        prevUV = rayUV;
    }
    
    if (!hit) return half4(0, 0, 0, 0);
    
    // 采样被反射的颜色
    half3 reflectedColor = SAMPLE_TEXTURE2D(_CameraColorTexture, sampler_CameraColorTexture, rayUV).rgb;
    
    // 边缘遮罩（距离屏幕边缘越近 → 反射越弱）
    float2 edgeDist = min(rayUV, 1 - rayUV);
    float edgeMask = saturate(min(edgeDist.x, edgeDist.y) * 10);
    
    // 菲涅尔效应：掠射角反射更强
    float NdotV = saturate(dot(viewNormal, -viewDir));
    float fresnel = pow(1.0 - NdotV, 5.0);
    fresnel = lerp(0.04, 1.0, fresnel); // Schlick 近似
    
    return half4(reflectedColor * edgeMask, edgeMask * fresnel);
}
```

**Hi-Z 构建对比表：**

| 方案 | 复杂度 | 质量 | 适用平台 | 实现难度 |
|------|--------|------|----------|----------|
| 暴力步进（固定步长） | O(N) | 低（噪声/漏检） | 全平台 | ⭐ |
| 自适应步长 | O(N) | 中 | 全平台 | ⭐⭐ |
| Hi-Z Ray Marching | O(logN) | 高 | PC/主机 | ⭐⭐⭐⭐ |
| Hi-Z + 时域累积 | O(logN) | 最高 | PC/主机 | ⭐⭐⭐⭐⭐ |

### ⚡ 实战经验

1. **移动端 SSR 的取舍**：手机上做全分辨率 SSR 几乎不可能。常见方案是 1/4 分辨率 + 时域上采样 + 只对特定材质（金属/湿润）开启。红米级别设备直接砍掉 SSR，用 Cubemap 替代
2. **Hi-Z 是性能分水岭**：不加 Hi-Z 的暴力 Ray Marching 在 1080p 下需要 64 步以上才能覆盖中远距离，帧时间直接爆炸。Hi-Z 可以在 8-12 步内完成等效覆盖
3. **Roughness 截断**：Roughness > 0.6 的材质直接不发射射线——反射太模糊，用 Irradiance SH 或 Cubemap 模糊版本替代，性价比高 10 倍
4. **遮挡变换时的鬼影问题**：角色移动时，历史帧的反射信息可能指向已被遮挡的物体。需要在 Reprojection 后做一个深度一致性检查（类似 TAA 的 History Reject）

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么从深度图重建世界坐标 | 深度图原理、逆矩阵投影 | 复习渲染数学：ViewSpace / ClipSpace / NDC 转换 |
| Ray Marching 总是穿透或漏检 | 碰撞检测逻辑、厚度容忍 | 研究 SSE/SSDO 算法的厚度处理策略 |
| 反射有大量噪点 | 时域累积、重要性采样 | 学习 TAA 原理 → 应用到 SSR 时域累积 |
| 不知道怎么在 URP 挂载 SSR | RendererFeature 机制 | 官方文档 + URP 自定义 Pass 教程 |
| Hi-Z 构建报错或效果不对 | Compute Shader + Mip Chain | 学习 Compute Shader 基础 + Hi-Z tracing 论文（如 DICE GDC 2015） |

### 🔗 相关问题

- Planar Reflection 和 SSR 各自的适用场景？（提示：水面 vs 曲面，性能 vs 精度）
- 如何用 Cubemap + SSR 混合实现完整的 IBL 反射体系？
- UE5 的 Lumen 反射和传统 SSR 有什么本质区别？（Surface Cache + Screen Space + Ray Tracing 三级 fallback）
