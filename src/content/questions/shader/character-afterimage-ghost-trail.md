---
title: "角色闪避残影：如何用Shader实现拖尾幻影——从逐帧抓色到屏幕空间重构"
category: "shader"
level: 3
tags: ["Shader", "URP", "残影", "GrabScreen", "顶点偏移", "技能特效"]
hint: "残影不是粒子拖尾——核心是「历史帧的角色轮廓」按时间衰减叠加，逐帧抓屏或顶点偏移+透明衰减是两条主流路线"
related: ["shader/dissolve-effect", "shader/hit-flash-damage-blink", "shader/hologram-projection-effect", "rendering/custom-motion-blur-urp"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款动作手游，角色有一个闪避技能——向左滑动后角色瞬移一段距离，在移动轨迹上留下3-5个半透明的残影，每个残影从清晰到模糊逐渐消散，持续0.8秒。整个效果不能影响战斗帧率（目标60fps）。给我Shader方案。」

补充追问：
- 残影需要保留角色当时的姿态（不是简单的Alpha淡出）
- 残影颜色偏向角色主色调，但要有一点偏色（比如偏蓝/偏紫，表示「幻影」）
- 多次连续闪避时残影不能堆积过多

这是米哈游、库洛、完美世界等做动作手游的TA面试高频题——残影效果是动作游戏的核心视觉反馈之一。

### ✅ 核心要点

1. **方案选型**：逐帧渲染到RenderTexture vs 顶点偏移+时间衰减——移动端优先后者
2. **顶点偏移残影**：在Vertex Shader中根据时间偏移顶点位置，制造「拖拽」效果
3. **多Pass叠加**：每个残影是一个Pass，在角色移动方向上偏移一定距离
4. **时间衰减控制**：每个残影有自己的生命周期，Alpha和模糊度随时间变化
5. **CommandBuffer方案**：在角色移动期间，每间隔一定帧抓取一次角色渲染结果到RT，后续叠加显示
6. **性能控制**：残影数量上限、LOD简化、半分辨率RT

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：闪避时角色身后留下3-5个半透明残影，逐渐消散
              ↑
倒推1：「残影」= 角色在过去某一帧的姿态快照
              ↑
方案A：渲染快照法 —— 每隔几帧把角色渲染到RT，叠加显示
方案B：顶点偏移法 —— 在Shader中偏移顶点，模拟运动模糊式的拖影
方案C：CommandBuffer法 —— 用URP RendererFeature在特定时机抓取角色
              ↑
选择依据：
├── 方案A：效果最好（真正保留姿态），但RT开销大
├── 方案B：性能最好，但残影姿态不准确（只是偏移轮廓）
└── 方案C：平衡方案，效果接近A，性能好于A
              ↑
移动端推荐方案C（CommandBuffer + 半分辨率RT）
PC/主机推荐方案A（全分辨率RT）
```

#### 知识点拆解（倒推树）

```
角色残影效果
├── 方案A：RenderTexture 快照法
│   ├── 抓取时机
│   │   ├── OnEnable 时开始抓取（闪避开始）
│   │   ├── 每隔 N 帧抓一次（如每3帧）
│   │   └── 闪避结束时停止抓取
│   ├── RT 管理
│   │   ├── 半分辨率 RT（节省带宽，残影本身就是模糊的）
│   │   ├── RT 池复用（3-5个RT循环使用）
│   │   └── RT 格式：ARGB32（半透明，不需要HDR）
│   ├── 渲染到 RT
│   │   ├── CommandBuffer.DrawRenderer 单独渲染角色到RT
│   │   ├── 替换材质为纯色材质（残影不需要细节）
│   │   └── 相机视角必须和主相机一致
│   ├── 叠加显示
│   │   ├── 全屏后处理 Pass 按顺序叠加所有RT
│   │   ├── 每张RT的Alpha按年龄递减
│   │   └── UV 微偏移让残影有抖动感
│   └── 清理
│       ├── 最老的RT被新RT覆盖（环形缓冲）
│       └── 所有残影消散后释放RT
├── 方案B：顶点偏移法
│   ├── 原理
│   │   ├── Vertex Shader中根据时间计算偏移量
│   │   ├── 偏移方向 = 闪避反方向
│   │   └── 偏移距离 = speed * timeSinceDodge
│   ├── 多Pass实现
│   │   ├── Pass1: 偏移0.0m，Alpha=1.0（本体）
│   │   ├── Pass2: 偏移0.3m，Alpha=0.6（残影1）
│   │   ├── Pass3: 偏移0.6m，Alpha=0.4（残影2）
│   │   └── Pass4: 偏移0.9m，Alpha=0.2（残影3）
│   ├── 局限
│   │   ├── 残影姿态和本体一样（只是位置偏移）
│   │   ├── 没有真正的姿态快照
│   │   └── 适合快速移动场景（看不清细节）
│   └── 优势
│       ├── 零额外内存（不需要RT）
│       ├── 性能极好（多一次Pass而已）
│       └── 实现简单
├── 方案C：CommandBuffer 法（推荐）
│   ├── URP RendererFeature 集成
│   │   ├── 在角色材质上标记特殊Layer/Tag
│   │   ├── AfterRenderingOpaques 时机抓取
│   │   └── 抓取时替换为残影材质
│   ├── 残影材质
│   │   ├── 纯色填充（取角色主色调）
│   │   ├── Fresnel 边缘增强（轮廓感）
│   │   └── 顶点往法线方向略膨胀（残影略大于本体）
│   ├── RT 叠加 Pass
│   │   ├── Screen Space 全屏绘制
│   │   ├── 按年龄排序：最老的在最底
│   │   └── Blend Mode: Alpha Blending
│   └── 性能控制
│       ├── 残影数上限4个
│       ├── RT 半分辨率
│       └── 闪避结束后快速消散（0.3s从Alpha 0.5→0）
├── 视觉调优
│   ├── 颜色偏移
│   │   ├── 冷色调（蓝/紫）→ 表示幻影/魔法
│   │   ├── 暖色调（橙/红）→ 表示速度/力量
│   │   └── 实现：原色 lerp 到偏色
│   ├── 边缘处理
│   │   ├── Fresnel 边缘光（让残影有轮廓发光感）
│   │   ├── 膨胀描边（Vertex Normal 外扩）
│   │   └── 扭曲背景（残影区域微扰UV，模拟空气扭曲）
│   └── 消散效果
│       ├── Alpha 衰减曲线（先慢后快，nonlinear）
│       ├── 噪声扰动消散边缘（结合dissolve思路）
│       └── 残影碎裂飞散（高级效果，粒子辅助）
└── 性能考量
    ├── 移动端 GPU 带宽
    │   ├── 半分辨率RT：带宽降到1/4
    │   ├── 不要用Float RT（ARGB32足够）
    │   └── 抓取频率控制（不是每帧都抓）
    ├── Draw Call 控制
    │   ├── 残影材质合并（一个Pass画所有残影）
    │   └── 残影用低LOD模型
    └── 内存控制
        ├── RT池：3个半分辨率RT循环使用
        └── 闪避结束0.5s后释放RT
```

#### 代码实现

**URP RendererFeature: 残影抓取与叠加**

```csharp
// AfterimageFeature.cs — URP 自定义 Renderer Feature
public class AfterimageFeature : ScriptableRendererFeature
{
    public RenderPassEvent injectionPoint = RenderPassEvent.AfterRenderingOpaques;
    public LayerMask afterimageLayer;          // 角色所在Layer
    public int maxAfterimageCount = 4;         // 最大残影数
    public float captureInterval = 0.05f;      // 抓取间隔(秒)
    public float afterimageLifetime = 0.8f;    // 残影存活时间

    private AfterimagePass pass;

    public override void Create()
    {
        pass = new AfterimagePass(maxAfterimageCount, captureInterval, afterimageLifetime)
        {
            renderPassEvent = injectionPoint,
            afterimageLayer = afterimageLayer
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        // 只在闪避激活时运行
        if (AfterimageManager.Instance != null && AfterimageManager.Instance.IsActive)
        {
            renderer.EnqueuePass(pass);
        }
    }
}
```

```csharp
// AfterimagePass.cs — 核心抓取与叠加逻辑
public class AfterimagePass : ScriptableRenderPass
{
    private int maxCount;
    private float captureInterval;
    private float lifetime;
    private LayerMask layerMask;

    // RT 池：环形缓冲
    private RTHandle[] afterimageRTs;
    private int currentIndex = 0;
    private float lastCaptureTime = 0f;

    // 残影材质（纯色+Fresnel）
    private Material afterimageMaterial;
    // 叠加材质
    private Material compositeMaterial;

    public AfterimagePass(int maxCount, float interval, float lifetime)
    {
        this.maxCount = maxCount;
        this.captureInterval = interval;
        this.lifetime = lifetime;

        // 创建半分辨率 RT 池
        afterimageRTs = new RTHandle[maxCount];
        for (int i = 0; i < maxCount; i++)
        {
            afterimageRTs[i] = RTHandles.Alloc(
                Vector2.one * 0.5f,  // 半分辨率
                name: $"_AfterimageRT_{i}"
            );
        }

        // 残影材质：纯色 + Fresnel
        var shader = Shader.Find("Hidden/AfterimageCapture");
        afterimageMaterial = new Material(shader);

        // 叠加材质
        var compositeShader = Shader.Find("Hidden/AfterimageComposite");
        compositeMaterial = new Material(compositeShader);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("Afterimage");

        float currentTime = (float)renderingData.cameraData.time;
        float dt = currentTime - lastCaptureTime;

        // Step 1: 按间隔抓取角色到 RT
        if (dt >= captureInterval)
        {
            lastCaptureTime = currentTime;

            // 绘制角色到当前RT（替换材质为残影材质）
            var renderers = AfterimageManager.Instance.RegisteredRenderers;
            foreach (var r in renderers)
            {
                cmd.DrawRenderer(r, afterimageMaterial);
            }

            // 抓取到RT
            var currentRT = afterimageRTs[currentIndex];
            Blit(cmd, renderingData.cameraData.renderer.cameraColorTargetHandle, currentRT);

            // 推进环形索引
            currentIndex = (currentIndex + 1) % maxCount;
        }

        // Step 2: 叠加所有残影到屏幕
        for (int i = 0; i < maxCount; i++)
        {
            int age = (currentIndex - 1 - i + maxCount) % maxCount;
            float normalizedAge = (i + 1) / (float)maxCount; // 0=最新, 1=最老
            float alpha = Mathf.Pow(1f - normalizedAge, 2f); // 非线性衰减

            compositeMaterial.SetFloat("_Alpha", alpha);
            compositeMaterial.SetTexture("_AfterimageTex", afterimageRTs[age]);
            compositeMaterial.SetColor("_TintColor",
                AfterimageManager.Instance.TintColor);

            Blit(cmd, afterimageRTs[age], renderingData.cameraData.renderer.cameraColorTargetHandle,
                compositeMaterial);
        }

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```

**残影抓取Shader:**

```hlsl
// AfterimageCapture.shader — 纯色 + Fresnel 边缘发光
Shader "Hidden/AfterimageCapture"
{
    Properties
    {
        _TintColor ("Tint Color", Color) = (0.3, 0.6, 1.0, 1.0) // 偏蓝
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }
        Blend SrcAlpha OneMinusSrcAlpha
        ZWrite Off
        Cull Back

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _TintColor;
            CBUFFER_END

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float3 normalWS   : TEXCOORD0;
                float3 viewDirWS  : TEXCOORD1;
            };

            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);

                // 顶点沿法线略膨胀（残影比本体略大一圈）
                float3 posOS = input.positionOS.xyz + input.normalOS * 0.005;
                output.positionCS = TransformObjectToHClip(posOS);

                output.normalWS = TransformObjectToWorldNormal(input.normalOS);
                output.viewDirWS = GetCameraPositionWS() -
                    TransformObjectToWorld(input.positionOS.xyz);
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                input.normalWS = normalize(input.normalWS);
                input.viewDirWS = normalize(input.viewDirWS);

                // Fresnel: 边缘亮，中心暗 → 突出轮廓
                float fresnel = 1.0 - saturate(dot(input.normalWS, input.viewDirWS));
                fresnel = pow(fresnel, 1.5);

                // 残影颜色：主色调 + Fresnel 增强
                float3 color = _TintColor.rgb * (0.3 + fresnel * 0.7);

                // Alpha: 边缘更不透明，中心更透明
                float alpha = saturate(0.4 + fresnel * 0.6);

                return half4(color, alpha);
            }
            ENDHLSL
        }
    }
}
```

**叠加Shader:**

```hlsl
// AfterimageComposite.shader — 将残影RT叠加到屏幕
Shader "Hidden/AfterimageComposite"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
        _AfterimageTex ("Afterimage", 2D) = "white" {}
        _Alpha ("Alpha", Float) = 0.5
        _TintColor ("Tint", Color) = (0.5, 0.7, 1.0, 1.0)
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" }
        Blend SrcAlpha OneMinusSrcAlpha
        ZWrite Off
        ZTest Always
        Cull Off

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
            };

            CBUFFER_START(UnityPerMaterial)
                float4 _MainTex_ST;
                float _Alpha;
                float4 _TintColor;
            CBUFFER_END

            TEXTURE2D(_AfterimageTex);
            SAMPLER(sampler_AfterimageTex);

            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                half4 afterimage = SAMPLE_TEXTURE2D(_AfterimageTex,
                    sampler_AfterimageTex, input.uv);

                // 偏色处理：残影颜色偏向TintColor
                afterimage.rgb = lerp(afterimage.rgb, _TintColor.rgb, 0.3);

                // 时间衰减Alpha
                afterimage.a *= _Alpha;

                return afterimage;
            }
            ENDHLSL
        }
    }
}
```

### ⚡ 实战经验

1. **不要用GrabScreen**：很多人第一反应是用GrabScreen做残影，但在URP下GrabScreen性能差且兼容性不好。用CommandBuffer + Blit才是正道
2. **半分辨率RT足够了**：残影本身就是半透明模糊的，没有人会盯着残影看细节。半分辨率直接省75%带宽
3. **残影材质要简化**：抓取残影时不要用角色原始材质（太复杂），替换成纯色+Fresnel的简化材质，4个残影的渲染成本约等于1次简单Pass
4. **消散曲线要非线性**：线性消散看起来很机械。用 `pow(1-age, 2)` 的曲线，前半段缓慢消失，后半段快速消散，更符合直觉
5. **多次闪避的冲突**：设置一个全局的AfterimageManager，闪避开始时清空旧残影并重新开始抓取。不要让多段闪避的残影互相叠加
6. **角色穿模问题**：残影使用Cull Back（只画背面），或者Cull Front只画轮廓——这样可以避免残影Z排序问题导致的穿模

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明 |
|--------|-----------|-----------|
| RenderTexture 的生命周期管理 | 理解GPU资源管理 | 需要补GPU内存管理基础 |
| CommandBuffer.DrawRenderer 的使用 | 熟悉URP/SRP扩展 | 需要学URP RendererFeature |
| Fresnel 效果原理 | 掌握基础Shader数学 | Shader入门需要加强 |
| Blit 操作的含义 | 理解屏幕空间后处理 | 需要补后处理管线知识 |
| 环形缓冲区设计 | 有工程数据结构基础 | 算法基础需要加强 |
| 半分辨率RT为什么够用 | 理解视觉感知与性能权衡 | 缺乏实战性能优化经验 |
| 残影和运动模糊的区别 | 理解不同效果的原理差异 | 概念混淆，需系统梳理 |
| SRP Batcher 对残影Pass的影响 | 深入理解SRP Batcher机制 | URP优化知识需要补 |

### 🔗 相关问题

- [shader/dissolve-effect](../shader/dissolve-effect.md) — 消散效果的噪声扰动可以复用到残影消散
- [shader/hologram-projection-effect](../shader/hologram-projection-effect.md) — 半透明+Fresnel+偏色的处理思路类似
- [rendering/custom-motion-blur-urp](../rendering/custom-motion-blur-urp.md) — 运动模糊与残影的技术路线对比
- [shader/hit-flash-damage-blink](../shader/hit-flash-damage-blink.md) — 同属角色反馈类Shader特效
