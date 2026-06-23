---
title: "镜头雨滴：如何在屏幕空间做出真实的雨水挂在镜头上的效果？"
category: "shader"
level: 3
tags: ["ScreenSpace", "后处理", "法线扰动", "URP", "雨景"]
hint: "核心是屏幕空间法线扰动——用噪声驱动水滴形变，模拟折射效果，而不是真的渲染水滴几何体"
related: ["shader/water-caustics", "rendering/custom-post-processing-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们在做一款开放世界游戏，下雨天需要镜头上挂水滴的效果——水滴会缓慢滑落、受重力影响、合并变大、滑过的地方留下拖尾。不能用粒子系统做几何体水滴，性能扛不住。给我一个屏幕空间的方案。」

（这是米哈游、腾讯天美、网易雷火的常见雨景 TA 面试题，考察后处理 Shader 能力和物理模拟能力）

### ✅ 核心要点

1. **屏幕空间后处理**：在最终画面上叠加水滴折射效果，不增加几何体
2. **水滴蒙版生成**：用噪声纹理 + 时间演化生成「水滴分布图」
3. **法线扰动折射**：水滴区域用法线扰动 UV 偏移采样场景颜色，模拟透镜折射
4. **物理滑落模拟**：在 Shader 中用简化物理模型驱动水滴向下移动
5. **拖尾区域**：水滴滑过路径降低表面张力，后续水滴更容易沿同路径流动

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：镜头上有水滴 → 水滴折射背后画面 → 水滴缓慢滑落 → 有拖尾
                ↑
倒推1：水滴「折射」效果 → 在水滴区域对场景颜色做 UV 偏移采样
倒推2：知道「哪里是水滴」→ 生成水滴蒙版图（RGBA：存在量 + 法线 XY）
倒推3：水滴会「移动」→ 在 RenderTexture 上做物理演化（每帧更新）
倒推4：水滴会「滑落」→ 重力方向 + 表面张力阈值 + 随机扰动
倒推5：水滴会「合并变大」→ 小水滴被经过的大水滴吸收
倒推6：水滴有「拖尾」→ 滑落路径降低该区域的表面张力
```

#### 知识点拆解（倒推树）

```
镜头雨滴效果
├── 后处理框架
│   ├── URP ScriptableRendererFeature 注入（BeforeRenderingPostProcessing）
│   ├── Blit 到临时 RT → 处理 → Blit 回
│   └── RT 格式选择（ARGBHalf for HDR 法线精度）
├── 水滴蒙版生成（核心难点）
│   ├── 初始化：随机散布水滴种子（hash-based 随机）
│   ├── 演化循环：
│   │   ├── 计算每个像素的水滴量（amount）
│   │   ├── 如果 amount > 表面张力阈值 → 开始滑落
│   │   ├── 滑落速度 = f(质量, 重力, 摩擦)
│   │   └── 移动后的位置写入新 RT（Ping-Pong RT）
│   ├── 合并逻辑：移动方向上的小水滴被吸收
│   └── 拖尾：滑落路径降低阈值，记录到 trail 通道
├── 折射渲染
│   ├── 从蒙版 RT 提取水滴法线（用 ddx/ddy 梯度近似）
│   ├── 法线 → UV 偏移量（折射强度参数控制）
│   └── 采样场景颜色：sceneColor.Sample(uv + offset)
├── 物理 Shader 模拟
│   ├── 表面张力模型：adhesion force vs gravity component
│   ├── 水滴大小 vs 临界质量：超过临界点才滑落
│   └── 时间步长：用 _Time.y 或 deltaTime 驱动
└── 性能控制
    ├── RT 分辨率：半分辨率模拟 + 全分辨率折射采样
    ├── 模拟频率：不一定每帧更新，可 30fps 模拟
    └── 移动端适配：简化物理模型，只用方向性流动
```

#### 代码实现

**URP RendererFeature 框架：**

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class LensRainRendererFeature : ScriptableRendererFeature
{
    public RenderPassEvent injectionPoint = RenderPassEvent.BeforeRenderingPostProcessing;
    public Material rainMaterial;
    public Shader simulationShader;
    public int simulationResolution = 960; // 半分辨率模拟

    private LensRainPass _pass;
    private RTHandle _simRT_A;
    private RTHandle _simRT_B;
    private bool _pingPong;

    public override void Create()
    {
        _pass = new LensRainPass(rainMaterial, simulationShader, simulationResolution);
        _pass.renderPassEvent = injectionPoint;
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (rainMaterial == null || simulationShader == null) return;
        _pass.Setup(renderer.cameraColorTargetHandle);
        renderer.EnqueuePass(_pass);
    }

    protected override void Dispose(bool disposing)
    {
        _simRT_A?.Release();
        _simRT_B?.Release();
    }
}
```

**水滴模拟 Shader（Ping-Pong Compute/Fragment）：**

```hlsl
// LensRainSimulation.shader
// 输入：上一帧状态 RT (_PrevState)
// 输出：当前帧状态 RT
// 通道：R = 水滴量, G = 流动速度, B = 拖尾强度, A = 水滴法线X

Shader "Hidden/LensRainSimulation"
{
    Properties { _MainTex ("State", 2D) = "black" {} }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_PrevState); SAMPLER(sampler_PrevState);
            float4 _PrevState_TexelSize; // x=1/width, y=1/height
            float _DeltaTime;
            float _Gravity;
            float _SurfaceTension;
            float _Evaporation;

            // 简化 hash 噪声
            float hash(float2 p)
            {
                return frac(sin(dot(p, float2(12.9898, 78.233))) * 43758.5453);
            }

            half4 frag(Varyings IN) : SV_Target
            {
                float2 uv = IN.uv;
                float2 texel = _PrevState_TexelSize.xy;

                // 采样上一帧状态
                half4 prev = SAMPLE_TEXTURE2D(_PrevState, sampler_PrevState, uv);
                float amount = prev.r;   // 水滴量
                float velocity = prev.g;  // 流动速度
                float trail = prev.b;     // 拖尾强度
                float normalX = prev.a;   // 法线 X

                // 采样上方像素（水滴从上面流下来）
                half4 above = SAMPLE_TEXTURE2D(_PrevState, sampler_PrevState, uv + float2(0, texel.y));

                // 表面张力判断：只有 amount + trail 超过阈值才开始流动
                float effectiveTension = _SurfaceTension * (1.0 - trail * 0.8);
                bool canFlow = (amount + above.r * 0.3) > effectiveTension;

                if (canFlow)
                {
                    // 加速下落
                    velocity += _Gravity * _DeltaTime * (amount * 2.0);
                    // 向下移动
                    float2 moveUV = uv + float2(normalX * 0.3, -velocity * texel.y * 60.0);
                    half4 source = SAMPLE_TEXTURE2D(_PrevState, sampler_PrevState, moveUV);
                    amount = max(amount, source.r * 0.9); // 吸收路径上的水
                    // 留下拖尾
                    trail = saturate(trail + 0.1);
                }

                // 随机生成新水滴（模拟雨打在镜头上）
                float rainNoise = hash(uv * 500.0 + _Time.y * 10.0);
                if (rainNoise > 0.998)
                {
                    amount = saturate(amount + 0.3);
                }

                // 蒸发
                amount = max(0, amount - _Evaporation * _DeltaTime);
                trail = max(0, trail - _Evaporation * 0.5 * _DeltaTime);

                // 计算法线（用梯度近似）
                float amountL = SAMPLE_TEXTURE2D(_PrevState, sampler_PrevState, uv - float2(texel.x, 0)).r;
                float amountR = SAMPLE_TEXTURE2D(_PrevState, sampler_PrevState, uv + float2(texel.x, 0)).r;
                normalX = (amountR - amountL) * 5.0;

                return half4(amount, velocity, trail, normalX);
            }
            ENDHLSL
        }
    }
}
```

**折射渲染 Shader（全分辨率）：**

```hlsl
// LensRainRender.shader — 读取模拟结果，对场景做折射偏移
Shader "Hidden/LensRainRender"
{
    SubShader
    {
        Pass
        {
            HLSLPROGRAM
            #pragma vertex FullscreenVert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_SceneColor); SAMPLER(sampler_SceneColor);
            TEXTURE2D(_SimState);   SAMPLER(sampler_SimState);
            float4 _SimState_TexelSize;
            float _RefractionStrength;
            float _DropletOpacity;

            half4 frag(Varyings IN) : SV_Target
            {
                float2 uv = IN.uv;

                // 采样模拟状态（注意模拟 RT 是半分辨率，需要 UV 缩放）
                half4 state = SAMPLE_TEXTURE2D(_SimState, sampler_SimState, uv);
                float amount = state.r;
                float normalX = state.a;

                // 用梯度计算法线 Y
                float amountU = SAMPLE_TEXTURE2D(_SimState, sampler_SimState, uv + float2(0, _SimState_TexelSize.y)).r;
                float amountD = SAMPLE_TEXTURE2D(_SimState, sampler_SimState, uv - float2(0, _SimState_TexelSize.y)).r;
                float normalY = (amountU - amountD) * 5.0;

                float2 dropletNormal = normalize(float2(normalX, normalY));

                // UV 偏移产生折射
                float2 offset = dropletNormal * amount * _RefractionStrength;

                half4 sceneColor = SAMPLE_TEXTURE2D(_SceneColor, sampler_SceneColor, uv + offset);

                // 水滴区域轻微增亮 + 饱和度变化（模拟水对光的吸收差异）
                float dropletMask = smoothstep(0.05, 0.3, amount);
                sceneColor.rgb = lerp(sceneColor.rgb, sceneColor.rgb * 1.1 + 0.02, dropletMask * _DropletOpacity);

                // 水滴边缘高光
                float edge = smoothstep(0.2, 0.35, amount) - smoothstep(0.35, 0.5, amount);
                sceneColor.rgb += edge * 0.15;

                return sceneColor;
            }
            ENDHLSL
        }
    }
}
```

**对比表：雨滴效果方案**

| 方案 | 真实度 | 性能 | 实现难度 | 适用场景 |
|------|--------|------|----------|----------|
| 粒子几何体水滴 | ★★★ | ★ | ★★ | 近景特写 |
| 屏幕空间法线扰动 | ★★★★ | ★★★ | ★★★★ | 主流方案（推荐） |
| 后处理扭曲贴图动画 | ★★ | ★★★★ | ★★ | 风格化/移动端 |
| 物理流体模拟 | ★★★★★ | ★ | ★★★★★ | 3A 级（PC） |

### ⚡ 实战经验

- **半分辨率模拟 + 全分辨率渲染**：模拟阶段在 960×540 的 RT 上做物理演化，渲染阶段全分辨率折射采样，性能和效果兼顾
- **Ping-Pong RT 优化**：模拟需要读写同一张 RT，用两张 RT 交替（A 读 B 写 → B 读 A 写），避免 UAV 在移动端兼容性问题
- **表面张力是灵魂**：没有表面张力阈值，所有水滴会一起滑落（像泼水），加了阈值才有「挂住 → 聚集 → 突然滑落」的真实感
- **拖尾通道复用**：拖尾数据存在 state RT 的 B 通道，不额外开 RT
- **移动端简化**：砍掉物理模拟，用预渲染的雨滴流动序列贴图（法线贴图动画），折射偏移用同一张法线
- **风向影响**：把重力方向从纯 (0,-1) 改为 (windX, -1)，水滴会斜着流，配合天气系统的风向参数

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么在屏幕空间做折射 | 后处理 UV 偏移采样原理 | 学习 Refraction 后处理、GrabPass 在 Built-in 中的原理 |
| 水滴不会「滑落」只有静态分布 | Shader 中物理模拟思路 | 学习 GPGPU / Fragment Shader 中做粒子模拟 |
| 水滴滑落看起来像「泼水」 | 缺少表面张力模型 | 理解表面张力、临界质量、接触角概念 |
| 合并逻辑不对，水滴重叠越来越亮 | Alpha blending vs max blending | RT blend mode 设置（Add vs Max） |
| 移动端跑不动物理模拟 | 移动端 GPU 带宽限制 | 学习移动端简化策略：预烘焙动画、降分辨率 |
| URP 下不知道怎么注入后处理 | ScriptableRendererFeature 机制 | 复习 URP 自定义 Render Pass 注入点 |

### 🔗 相关问题

- 如何让水滴在角色身上也有效果？（提示：需要世界空间方案，不是屏幕空间）
- 镜头上有泥点时，水滴会绕开泥点流——怎么实现？（提示：泥点蒙版影响表面张力）
- 如何在 Shader Graph 中实现简化版镜头雨滴？（提示：用 Custom Function 节点嵌入 HLSL）
