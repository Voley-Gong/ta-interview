---
title: "同屏20个角色阴影：URP级联阴影复用与角色阴影性能控制"
category: "rendering"
level: 3
tags: ["URP", "阴影", "级联阴影", "性能优化", "手游渲染", "ShadowMap"]
hint: "20个角色同屏阴影崩帧——问题不在GPU画阴影，而在ShadowMap分辨率不够+角色DrawCall在Shadow Pass翻倍"
related: ["rendering/deferred-multi-light", "rendering/shadow-acne-peter-panning-fix-urp", "optimization/drawcall-500-to-100", "rendering/custom-screen-space-shadow-soften"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款MOBA手游，团战时同屏20个角色，每个角色都要有动态阴影。当前URP默认设置下，开启所有角色阴影后帧率从60fps掉到35fps。Profiler显示Shadow Pass的耗时占了8ms。你作为TA怎么优化？」

附加约束：
- 角色阴影必须有自阴影（手指、头发不能没有阴影）
- 地面阴影边缘要柔和（不能锯齿明显）
- 适配中端机型（骁龙7 Gen 1级别）

这是腾讯、网易、字节做MOBA/MMO的TA面试经典题——多角色阴影是3D实时渲染的性能重灾区。

### ✅ 核心要点

1. **ShadowMap分辨率控制**：不需要每个角色都用2048的ShadowMap——256~512足矣
2. **阴影渲染频率优化**：不需要每帧更新所有角色的ShadowMap，可以分帧轮换
3. **Shadow Caster LOD**：Shadow Pass使用低面数LOD，大幅减少顶点处理
4. **级联阴影 vs 点光源阴影的取舍**：远距离角色用简单圆形阴影替代ShadowMap
5. **Atlas ShadowMap**：多角色ShadowMap打包到一张Atlas上，减少Pass切换
6. **距离剔除**：屏幕外的角色不渲染阴影

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：20角色同屏阴影，Shadow Pass从8ms降到2ms以内
              ↑
Step 1：定位瓶颈——到底是GPU还是CPU？
  ├── Shadow Pass = Shadow Caster Pass (画ShadowMap) + Shadow Receiver Pass (采样ShadowMap)
  ├── 如果Caster耗时高 → 顶点数太多（角色模型太复杂）
  ├── 如果Receiver耗时高 → 采样太贵（PCF采样次数过多）
  └── DrawCall：20角色 × (本体1 + ShadowCaster 1) = 40 DrawCall（翻倍！）
              ↑
Step 2：Shadow Caster优化（减少画ShadowMap的开销）
  ├── 使用LOD1或更低的模型画ShadowMap（看不到细节）
  ├── 合并小角色到一个ShadowMap Atlas
  └── 分帧更新（不是所有角色每帧都更新ShadowMap）
              ↑
Step 3：Shadow Receiver优化（减少采样ShadowMap的开销）
  ├── PCF 3x3 → 2x2 Tent Filter（采样次数从9→4）
  ├── 远距离角色使用Hard Shadow（不做柔化）
  └── 阴影距离按角色重要性调整
              ↑
Step 4：替代方案（最激进的优化）
  ├── 远距离角色：Blob Shadow（贴花圆形阴影，零ShadowMap开销）
  ├── 中距离角色：简化的ShadowMap（128分辨率，Hard Shadow）
  └── 近距离主角：完整ShadowMap（512分辨率，Soft Shadow）
```

#### 知识点拆解（倒推树）

```
多角色阴影优化
├── 1. URP 阴影管线理解
│   ├── Main Light Shadow
│   │   ├── Cascade Shadow Map（CSM）
│   │   ├── 默认最多4级级联
│   │   └── 每级独立ShadowMap分辨率
│   ├── Additional Light Shadow
│   │   ├── Point Light → CubeMap Shadow（6面，极贵）
│   │   ├── Spot Light → 单张ShadowMap
│   │   └── URP默认关闭Additional Light Shadow
│   └── Shadow Pass 流程
│       ├── 1. 渲染ShadowMap（Depth Only，从光源视角）
│       ├── 2. 屏幕空间Shadow Pass（采样ShadowMap，生成阴影遮罩）
│       └── 3. 光照Pass中应用阴影遮罩
├── 2. ShadowMap 分辨率优化
│   ├── URP 阴影分辨率档位
│   │   ├── Low: 256
│   │   ├── Medium: 512
│   │   ├── High: 1024
│   │   └── Ultra: 2048
│   ├── 移动端推荐
│   │   ├── 主角/重要NPC: 512
│   │   ├── 普通角色: 256
│   │   └── 远距离: 不渲染阴影（用Blob Shadow替代）
│   └── 级联分配策略
│       ├── Cascade 0 (最近 0-15m): 512
│       ├── Cascade 1 (15-30m): 256
│       ├── Cascade 2 (30-50m): 不分配（用Blob Shadow）
│       └── Cascade 3 (50m+): 关闭
├── 3. Shadow Caster 优化
│   ├── LOD 策略
│   │   ├── Shadow Pass 强制使用 LOD1 或 LOD2
│   │   ├── Shadow Caster 专用低模（比LOD1更低）
│   │   └── 实现：Shader Pass 中用 `lod` 或 C# 控制
│   ├── 剔除不重要的 Shadow Caster
│   │   ├── 屏幕外角色不画ShadowMap
│   │   ├── 太远的角色跳过Shadow Caster
│   │   └── 死亡/隐身角色不画Shadow
│   └── 分帧更新（ShadowMap 翻转率）
│       ├── 20个角色分3帧更新
│       ├── 每帧只更新约7个角色的ShadowMap
│       ├── 视觉上几乎无感知（60fps下每帧16ms，3帧=50ms延迟）
│       └── 实现：ScriptableRendererFeature控制
├── 4. Shadow Receiver 优化
│   ├── 采样次数控制
│   │   ├── PCF 2x2 (4次采样) → 移动端默认
│   │   ├── PCF 3x3 (9次采样) → 中高端可选
│   │   ├── PCF 5x5 (25次采样) → 移动端禁用
│   │   └── Tent Filter (VSM/ESM) → 更少采样更好效果
│   ├── 距离自适应柔化
│   │   ├── 近处：3x3 PCF
│   │   ├── 中距离：2x2 PCF
│   │   └── 远处：Hard Shadow (1次采样)
│   └── Shadow Bias 优化
│       ├── Slope-Scale Bias 减少Shadow Acne
│       ├── Normal Bias 法线偏移避免Peter Panning
│       └── 自适应Bias（根据Surface Normal和Light Direction计算）
├── 5. Blob Shadow（贴花阴影）替代方案
│   ├── 原理
│   │   ├── 在角色脚下贴一个圆形/椭圆贴花
│   │   ├── 贴花颜色为半透明黑色
│   │   └── 不需要ShadowMap，只是个Decal
│   ├── 实现方式
│   │   ├── URP Decal Projector（贴花投影器）
│   │   ├── Shader：角色脚下的圆盘，根据高度衰减
│   │   └── 透明度随角色跳跃高度递减
│   ├── 优势
│   │   ├── 零ShadowMap开销
│   │   ├── 1个DrawCall搞定
│   │   └── 性能极好
│   ├── 局限
│   │   ├── 没有自阴影
│   │   ├── 阴影形状固定（圆形/椭圆）
│   │   └── 不适合需要精确阴影的场景
│   └── 适用场景
│       ├── MOBA 远距离小兵
│       ├── MMO 同屏大量NPC
│       └── 策略游戏中的俯视场景
├── 6. Atlas ShadowMap（高级方案）
│   ├── 原理
│   │   ├── 多个角色的ShadowMap渲染到一张大RT的不同区域
│   │   ├── 类似Texture Atlas的思路
│   │   └── 一次SetRenderTarget，多次Viewport设置
│   ├── 实现
│   │   ├── 1024x1024 Atlas，每个角色分配256x256区域
│   │   ├── 每个角色有自己的光源VP矩阵
│   │   └── 采样时根据角色ID选择对应UV区域
│   └── 优势
│       ├── 减少 RenderTarget 切换
│       ├── 减少 Pass 切换
│       └── 适合大量角色场景
└── 7. 替代阴影算法
    ├── Variance Shadow Map (VSM)
    │   ├── 存深度和深度平方
    │   ├── 预滤波后用切比雪夫不等式估算遮挡
    │   └── 优点：可以硬件双线性滤波，柔化效果好
    ├── Exponential Shadow Map (ESM)
    │   ├── 深度差取指数
    │   ├── 同样支持预滤波
    │   └── 内存比VSM少（不需要存平方）
    └── Screen Space Shadow（URP默认）
        ├── 屏幕空间逐像素采样ShadowMap
        ├── 生成屏幕空间阴影遮罩
        └── 可以在此做后处理柔化
```

#### 代码实现

**URP 阴影LOD控制（C# RendererFeature）:**

```csharp
// ShadowLODFeature.cs — Shadow Pass 中强制使用低LOD
public class ShadowLODFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public int shadowCasterLOD = 1;       // Shadow Pass使用的LOD层级
        public float distanceCutoff = 50f;     // 超过此距离不画Shadow
        public int framesPerShadowUpdate = 1;  // 每N帧更新一次Shadow
    }

    public Settings settings = new Settings();
    private ShadowLODPass pass;

    public override void Create()
    {
        pass = new ShadowLODPass(settings)
        {
            renderPassEvent = RenderPassEvent.BeforeRenderingShadows
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer,
        ref RenderingData renderingData)
    {
        renderer.EnqueuePass(pass);
    }
}

public class ShadowLODPass : ScriptableRenderPass
{
    private Settings settings;
    private List<Renderer> managedRenderers = new List<Renderer>();
    private int frameCounter = 0;

    public ShadowLODPass(Settings settings) => this.settings = settings;

    public override void Execute(ScriptableRenderContext context,
        ref RenderingData renderingData)
    {
        frameCounter++;
        bool shouldUpdateShadow = (frameCounter % settings.framesPerShadowUpdate) == 0;

        foreach (var renderer in managedRenderers)
        {
            if (renderer == null) continue;

            float distance = Vector3.Distance(
                renderer.transform.position,
                renderingData.cameraData.camera.transform.position);

            // 距离剔除：远处角色不画Shadow
            if (distance > settings.distanceCutoff)
            {
                renderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.Off;
                continue;
            }

            // Shadow LOD 控制
            var lodGroup = renderer.GetComponent<LODGroup>();
            if (lodGroup != null)
            {
                // 强制Shadow使用低LOD
                lodGroup.ForceLOD(settings.shadowCasterLOD);
            }

            // 分帧更新
            renderer.shadowCastingMode = shouldUpdateShadow
                ? UnityEngine.Rendering.ShadowCastingMode.On
                : UnityEngine.Rendering.ShadowCastingMode.ShadowsOnly;
            // ShadowsOnly = 只画Shadow不画本体（下一帧跳过Shadow Caster）
        }
    }
}
```

**Blob Shadow（贴花阴影）Shader:**

```hlsl
// BlobShadow.shader — 角色脚下的程序化圆形阴影
Shader "Custom/BlobShadow"
{
    Properties
    {
        _Radius ("Shadow Radius", Float) = 1.0
        _Softness ("Edge Softness", Range(0.01, 1.0)) = 0.3
        _Opacity ("Shadow Opacity", Range(0, 1)) = 0.6
        _HeightFalloff ("Height Falloff", Float) = 5.0
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="AlphaTest+50" }
        Blend SrcAlpha OneMinusSrcAlpha
        ZWrite Off

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float _Radius;
                float _Softness;
                float _Opacity;
                float _HeightFalloff;
            CBUFFER_END

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 worldPos   : TEXCOORD1;
            };

            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                output.worldPos = TransformObjectToWorld(input.positionOS.xyz);
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                // 从中心到当前像素的距离
                float2 centeredUV = input.uv - 0.5;
                float dist = length(centeredUV) * 2.0; // 0=中心, 1=边缘

                // 圆形遮罩 + 边缘柔化
                float shadow = 1.0 - smoothstep(
                    1.0 - _Softness, 1.0, dist / _Radius);

                // 根据角色离地高度衰减（跳起来时阴影变淡）
                // _HeightFalloff 由 C# 设置，值为 1/角色离地高度
                float heightFade = saturate(1.0 / (1.0 + _HeightFalloff * 0.1));

                float alpha = shadow * _Opacity * heightFade;
                return half4(0, 0, 0, alpha);
            }
            ENDHLSL
        }
    }
}
```

### ⚡ 实战经验

1. **Shadow Pass 是隐形的Draw Call杀手**：20个角色开阴影，Shadow Caster Pass就额外多了20个DrawCall。很多人只盯着本体的DrawCall看，忽略了阴影翻倍效应
2. **分帧更新是最有效的优化**：人眼对阴影的更新频率不敏感——3帧更新一次（50ms延迟）几乎无感知，但直接省66%的Shadow Caster开销
3. **Blob Shadow被严重低估**：MOBA/MMS游戏中，屏幕上小如蚂蚁的角色根本不需要精确阴影。一个贴花圆形阴影 + 主角完整阴影的混合方案，效果几乎一致，性能差10倍
4. **ShadowMap分辨率不是越高越好**：2048 ShadowMap在移动端反而可能更卡——更多像素需要渲染，带宽压力更大。256~512是移动端甜点
5. **URP的Shadow Distance要调**：默认500m太远了，移动端设为30~50m，超出范围的阴影用Blob Shadow替代
6. **VSM/ESM 在移动端要谨慎**：虽然理论上采样次数少，但浮点RT在移动端TBDR架构上反而可能触发tile miss，得不偿失。先用PCF，确认瓶颈再换

### 🎯 能力体检清单

| 检查项 | 能答上说明 | 答不上说明 |
|--------|-----------|-----------|
| URP Shadow Pass 的完整渲染流程 | 理解URP阴影管线 | 需要补URP渲染管线基础 |
| Cascade Shadow Map 的原理 | 掌握CSM分级策略 | 阴影系统知识盲区 |
| Shadow Caster 和 Shadow Receiver 的区别 | 理解阴影的两阶段 | 概念不清，需系统学习 |
| PCF 采样的数学原理 | 掌握阴影柔化算法 | Shader数学基础薄弱 |
| 如何用 LOD 控制 Shadow Caster | 有实战阴影优化经验 | 缺乏工程优化经验 |
| Blob Shadow 的适用场景 | 有方案选型判断力 | 只知道一种方案 |
| DrawCall 在 Shadow Pass 中如何翻倍 | 理解阴影对CPU的影响 | DrawCall分析能力不足 |
| VSM/ESM 与 PCF 的优劣对比 | 深入理解阴影算法 | 高级阴影知识需要补 |

### 🔗 相关问题

- [rendering/shadow-acne-peter-panning-fix-urp](../rendering/shadow-acne-peter-panning-fix-urp.md) — Shadow Bias 相关的阴影问题
- [rendering/custom-screen-space-shadow-soften](../rendering/custom-screen-space-shadow-soften.md) — 屏幕空间阴影柔化方案
- [rendering/deferred-multi-light](../rendering/deferred-multi-light.md) — 延迟渲染下多光源阴影处理
- [optimization/drawcall-500-to-100](../optimization/drawcall-500-to-100.md) — DrawCall 优化的通用方法
