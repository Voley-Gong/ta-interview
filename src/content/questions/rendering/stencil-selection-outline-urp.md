---
title: "URP下用Stencil实现角色选中高亮描边：面试官说不能用法线外扩法"
category: rendering
level: 3
tags: ["URP", "Stencil", "描边", "选中高亮", "Renderer Feature"]
hint: "Stencil标记角色区域 → 全屏描边Pass只对标记像素做边缘检测"
related: ["npr-outline-cartoon", "custom-post-processing-urp", "planar-projection-shadow"]
---

## 参考答案

### 🎬 场景描述

面试官画了一个RTS游戏原型草图，说：

> "玩家点击角色后，角色周围出现一圈发光描边表示'被选中'。限制条件：
> 1. 用URP
> 2. 不允许用法线外扩法（因为角色模型法线质量参差不齐）
> 3. 描边颜色要能动态切换（友方蓝色、敌方红色）
> 4. 描边只在角色外围一圈，不是全模覆盖
> 
> 你怎么实现？"

### ✅ 核心要点

1. **两遍渲染**：第一遍渲染角色时写入 Stencil，第二遍全屏Pass对 Stencil 区域做边缘检测
2. **描边生成**：采样上下左右4个像素的Stencil值，如果当前像素在Stencil内但邻居在Stencil外，就是描边像素
3. **URP集成**：自定义 RendererFeature，在角色渲染后插入描边Pass
4. **颜色动态切换**：描边Pass用 `_OutlineColor` 全局参数，C#脚本实时修改

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：选中角色外围出现彩色描边光环
        ↓ 倒推
描边 = 角色像素的边缘
        ↓ 倒推
怎么找边缘？需要知道哪些像素是角色，哪些不是 → Stencil标记
        ↓ 倒推
两遍渲染：
  Pass 1: 角色正常渲染，同时写 Stencil=1
  Pass 2: 全屏Quad，对每个像素检查Stencil，用边缘检测找出描边
        ↓ 倒推
URP里怎么插入这两遍？→ ScriptableRendererFeature + 两个RenderPass
        ↓ 倒推
必须掌握：Stencil缓冲区、边缘检测Shader、URP多Pass编排
```

#### 知识点拆解（倒推树）

```
Stencil描边高亮
├── Stencil 机制
│   ├── Stencil Buffer 是什么（独立于Color/Depth的逐像素标记）
│   ├── Ref / Comp / Pass 写法（WriteMask / ReadMask）
│   └── 怎么让角色"先画"再"读"
├── 边缘检测原理
│   ├── 对 Stencil 纹理做 Sobel / 4方向检测
│   ├── 当前像素 Stencil=1 但邻居=0 → 边缘
│   └── 描边宽度 = 采样偏移量 × 像素尺寸
├── URP 实现
│   ├── RendererFeature 中管理 Pass 顺序
│   ├── 第一遍：OverrideMaterial 写 Stencil
│   ├── 第二遍：全屏Blit做边缘检测
│   └── RTHandle 临时纹理管理
└── 工程细节
    ├── 多角色多颜色（Stencil分层：友方Ref=1，敌方Ref=2）
    ├── 抗锯齿问题（MSAA下Stencil边缘锯齿）
    └── 性能：全屏Pass成本 vs 只渲染角色边界
```

#### 代码实现

**Pass 1：角色渲染时写 Stencil**

```hlsl
// StencilWrite.shader — 角色的Override Shader
Shader "Hidden/StencilWrite"
{
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            // 不写入颜色，只写Stencil
            ColorMask 0
            ZWrite Off
            ZTest LEqual

            Stencil
            {
                Ref 1
                Comp Always
                Pass Replace
            }

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes { float4 pos : POSITION; };
            struct Varyings { float4 pos : SV_POSITION; };

            Varyings Vert(Attributes i)
            {
                Varyings o;
                o.pos = TransformObjectToHClip(i.pos.xyz);
                return o;
            }

            half4 Frag(Varyings i) : SV_Target { return 0; }
            ENDHLSL
        }
    }
}
```

**Pass 2：全屏边缘检测描边**

```hlsl
// StencilOutline.shader — 全屏描边检测
Shader "Hidden/StencilOutline"
{
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment Frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_StencilTex);
            SAMPLER(sampler_StencilTex);
            float4 _StencilTex_TexelSize; // x=1/width, y=1/height

            float4 _OutlineColor;
            float _OutlineWidth; // 像素数

            // 采样Stencil纹理某点是否为角色
            float IsMarked(float2 uv)
            {
                return SAMPLE_TEXTURE2D(_StencilTex, sampler_StencilTex, uv).r;
            }

            half4 Frag(Varyings input) : SV_Target
            {
                float2 uv = input.uv;
                float2 texel = _StencilTex_TexelSize.xy * _OutlineWidth;

                // 4方向检测
                float center = IsMarked(uv);
                float up    = IsMarked(uv + float2(0,  texel.y));
                float down  = IsMarked(uv + float2(0, -texel.y));
                float left  = IsMarked(uv + float2(-texel.x, 0));
                float right = IsMarked(uv + float2( texel.x, 0));

                // 当前不在标记内，但旁边有标记 → 描边
                float edge = 0;
                edge = max(edge, step(0.5, center) * step(0.5, 1.0 - up));
                edge = max(edge, step(0.5, center) * step(0.5, 1.0 - down));
                edge = max(edge, step(0.5, center) * step(0.5, 1.0 - left));
                edge = max(edge, step(0.5, center) * step(0.5, 1.0 - right));

                // 更好的做法：当前像素不在内但周围在内也算边缘（外描边）
                float surround = up + down + left + right;
                edge = max(edge, step(0.5, 1.0 - center) * step(1.5, surround));

                return half4(_OutlineColor.rgb * edge, edge);
            }
            ENDHLSL
        }
    }
}
```

**RendererFeature 编排：**

```csharp
// StencilOutlineFeature.cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class StencilOutlineFeature : ScriptableRendererFeature
{
    public Material stencilWriteMat;   // StencilWrite
    public Material outlineMat;        // StencilOutline
    public RenderPassEvent passEvent = RenderPassEvent.AfterRenderingOpaques;
    public Color outlineColor = Color.yellow;
    [Range(1, 10)] public float outlineWidth = 2f;

    private StencilOutlinePass _pass;
    private bool _isSelected = false;

    public override void Create()
    {
        _pass = new StencilOutlinePass(stencilWriteMat, outlineMat)
        {
            renderPassEvent = passEvent
        };
    }

    public void SetSelected(bool selected, Color color)
    {
        _isSelected = selected;
        outlineColor = color;
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (!_isSelected || outlineMat == null || stencilWriteMat == null) return;

        outlineMat.SetColor("_OutlineColor", outlineColor);
        outlineMat.SetFloat("_OutlineWidth", outlineWidth);
        _pass.Setup(renderer.cameraColorTargetHandle);
        renderer.EnqueuePass(_pass);
    }
}
```

### ⚡ 实战经验

1. **为什么不用法线外扩**：法线外扩依赖模型顶点法线质量。如果模型没规范化法线（比如硬边模型），描边会断裂、不均匀。Stencil方案与模型几何无关，效果稳定
2. **描边宽度按屏幕像素而非UV**：用 `_TexelSize * width` 保证不同分辨率下描边视觉宽度一致
3. **Stencil纹理的获取**：URP中不直接暴露Stencil Buffer，实际做法是渲染一个只含Stencil信息的单通道RT（R8格式），移动端成本更低
4. **多角色多颜色**：用 Stencil Ref 分层——友方写 Ref=1，敌方写 Ref=2，描边Shader里根据采样值选颜色
5. **MSAA问题**：开启MSAA后Stencil边缘会多采样，导致描边锯齿。建议此Pass在MSAA Resolve之后执行

### 🎯 能力体检清单

| 如果答不上来... | 说明盲区在 |
|---|---|
| 不知道Stencil是什么，没写过 `Stencil { Ref Comp Pass }` | 渲染管线 Stencil 机制 |
| 知道Stencil但不知道怎么从全屏找出"边缘" | 图像边缘检测原理 |
| 在URP里不知道怎么"让角色写Stencil再全屏检测" | URP 多Pass编排 + RTHandle |
| 描边宽度在不同分辨率下忽粗忽细 | UV坐标 vs 像素坐标的区别 |
| 没考虑性能：全屏Pass有没有更省的做法 | 移动端后处理性能优化 |

### 🔗 相关问题

- [卡通渲染Outline：四种描边方案你怎么选？](npr-outline-cartoon)
- [URP自定义后处理：Renderer Feature从入门到上线](custom-post-processing-urp)
- [平面投影阴影：角色脚下的影子怎么画？](planar-projection-shadow)
