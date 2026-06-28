---
title: "手游弹孔/污渍/印记系统：URP 下如何实现高性能贴花投影？"
category: "rendering"
level: 3
tags: ["Decal", "贴花", "URP", "Deferred", "弹孔", "污渍", "移动端"]
hint: "URP 14+ 有 Decal Projector 组件，但移动端延迟贴花不可用——前向渲染下要用屏幕空间贴花或 Mesh 贴花方案"
related: ["rendering/urp-renderer-feature", "rendering/deferred-multi-light", "shader/body-paint-decal-projection"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们在做一款 FPS 手游，开枪后墙上、地上、角色身上都要留下弹孔贴花。美术还要求：血迹、泥浆、涂鸦印记都走同一套系统。给我一套 URP 下的贴花方案，要考虑移动端性能，最多同屏 50 个贴花，不能掉帧。」

### ✅ 核心要点

1. **贴花投影本质**：把一张 2D 贴图"贴"到不规则 3D 表面上，跟随表面凹凸变形
2. **三种方案各有适用场景**：屏幕空间贴花（Screen Space Decal）、Decal Projector（URP 内置）、Mesh 贴花（预生成几何体）
3. **移动端前向渲染限制**：URP Decal Projector 在 Forward 模式下通过 `Decal Subpass` 或 DBuffer 实现，但移动端 GPU 支持差异大
4. **性能关键**：贴花数量控制、距离剔除、LOD 降级、贴图图集

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：开枪 → 子弹击中表面 → 弹孔贴花出现在命中点 → 贴花贴合表面曲率 → 50个贴花不掉帧
                ↑
倒推1：贴花如何"贴"在3D表面？
      → 方案A：Decal Projector（盒型投影体，从指定方向投射贴图到命中表面）
      → 方案B：屏幕空间贴花（全屏后处理，从相机视角投影）
      → 方案C：运行时生成贴花 Mesh（吸附到命中面）
倒推2：贴花如何跟随表面凹凸？
      → 需要Deferred Path的G-Buffer（法线/深度），或DBuffer（Decal Buffer）
      → Forward Path下：用深度重构世界坐标，采样表面法线，偏移贴花UV
倒推3：50个贴花如何管理？
      → 不能50个Draw Call → 用 ComputeBuffer/StructuredBuffer 批量传参
      → 或用 Draw Procedural 一次绘制所有贴花
      → 距离剔除：近处用高质量贴花，远处降级或消失
倒推4：弹孔/血迹/涂鸦共用系统？
      → 贴花图集（Atlas）：所有贴花类型放在一张大图
      → 贴花数据结构：{位置, 方向, 类型(SampleUV), 缩放, 生命周期}
倒推5：贴花生命周期管理
      → 血迹永久、弹孔永久、泥浆会干涸（渐隐）、涂鸦可被覆盖
      → 对象池：预分配贴花槽位，循环复用
```

#### 知识点拆解（倒推树）

```
贴花投影系统
├── URP Decal Projector（官方方案）
│   ├── 原理：盒型投影体 → 渲染到 DBuffer 或屏幕空间
│   ├── Forward 模式：需要 DBuffer（额外 GBuffer-like 通道）
│   ├── Deferred 模式：直接写入 G-Buffer（法线/反照率）
│   ├── 限制：移动端 DBuffer 支持有限（需要检查 GPU 能力）
│   └── 优点：Unity 原生支持，无需自己写投影数学
├── 屏幕空间贴花（Screen Space Decal）
│   ├── 原理：渲染一个面向相机的盒体 → 用屏幕空间深度重建世界坐标 → 投影UV
│   ├── 核心步骤：
│   │   1. 渲染盒体（Back Faces），在 Fragment 中计算投影
│   │   2. 采样场景深度 → 重建世界坐标
│   │   3. 将世界坐标转换到贴花投影空间 → 得到 UV
│   │   4. 如果 UV 在 [0,1] 内 → 采样贴花贴图
│   ├── 优点：不依赖 G-Buffer，Forward/Deferred 都可用
│   ├── 缺点：不能贴到相机视野外的表面、不能贴到遮挡面
│   └── 移动端友好：核心成本是一次额外全屏深度采样
├── Mesh 贴花（预生成几何体）
│   ├── 原理：在命中点生成一个贴合表面的简化 Mesh
│   ├── 生成方式：射线检测命中面 → 沿法线方向偏移 → 生成平面 Mesh
│   ├── 优点：性能最好（就是普通 Mesh 渲染），兼容性最佳
│   ├── 缺点：不规则表面贴合差（需要运行时变形）、贴花数量多时顶点暴涨
│   └── 适用：墙壁弹孔等平面贴花
├── 批量渲染（50个贴花不掉帧）
│   ├── GPU Instancing：所有同类贴花用同一个 Material + Instanced 绘制
│   ├── StructuredBuffer 传递贴花数据：
│   │     struct DecalData { float3 position; float3 forward; float scale; float2 atlasUV; float life; }
│   ├── Draw Procedural / Draw Mesh Instanced（Indirect）
│   └── 一次 Draw Call 绘制所有贴花
├── 贴花图集（Atlas）
│   ├── 所有弹孔/血迹/涂鸦烘焙到一张 2048×2048 Atlas
│   ├── 每个贴花存 Atlas UV 矩形偏移
│   ├── 优点：一次采样，一个材质
│   └── 美术工作流：Substance Painter 输出 Atlas → 导入 Unity
├── 贴花与表面法线对齐
│   ├── 方案A（简单）：贴花投影方向 = 命中面法线
│   ├── 方案B（精确）：采样场景法线 → 贴花切面跟随法线倾斜
│   └── 方案C（DBuffer）：写入 DBuffer 法线通道，后续光照自动正确
├── 生命周期与对象池
│   ├── 预分配 N 个贴花槽位（如 64 个）
│   ├── 新贴花覆盖最老的
│   ├── 血迹/弹孔 = 永久（life = ∞）
│   └── 泥浆/水渍 = 衰减消失（life = 10s → alpha fade）
└── 移动端性能优化
    ├── 距离剔除：> 30m 的贴花不渲染
    ├── 分辨率降级：远处贴花用 1/2 分辨率
    ├── 限制同屏数量：通过预算控制（如最多 32 个活跃贴花）
    └── 贴花图集压缩：ASTC 6×6 对于 RGBA 贴花质量/尺寸平衡好
```

#### 代码实现

**屏幕空间贴花 Shader（URP Forward 兼容）：**

```hlsl
// ScreenSpaceDecal.shader
Shader "Custom/ScreenSpaceDecal"
{
    Properties
    {
        _DecalTex ("Decal Atlas", 2D) = "white" {}
        _NormalBlend ("Normal Blend", Range(0,1)) = 0.5
        _FadeDistance ("Fade Distance", Float) = 30.0
    }

    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="AlphaTest+1" }
        // 关键：只渲染背面，避免盒体正面遮挡
        Cull Front
        ZWrite Off
        ZTest Off  // 贴花不写深度

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float3 rayDir : TEXCOORD0;  // 从相机到顶点的射线
                float2 uv : TEXCOORD1;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            // 贴花投影体参数（通过 Instancing 传入）
            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float4x4, _DecalMatrix)  // 投影世界矩阵
                UNITY_DEFINE_INSTANCED_PROP(float4, _AtlasUVRect)    // Atlas UV 矩形
                UNITY_DEFINE_INSTANCED_PROP(float, _DecalScale)
            UNITY_INSTANCING_BUFFER_END(Props)

            TEXTURE2D(_DecalTex); SAMPLER(sampler_DecalTex);

            Varyings vert(Attributes input)
            {
                Varyings output;
                UNITY_SETUP_INSTANCE_ID(input);
                UNITY_TRANSFER_INSTANCE_ID(input, output);

                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;

                // 计算从相机到顶点的世界空间射线
                float3 worldPos = TransformObjectToWorld(input.positionOS.xyz);
                output.rayDir = worldPos - _WorldSpaceCameraPos;

                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);

                // 1. 采样场景深度，重建世界坐标
                float2 screenUV = input.positionCS.xy / _ScaledScreenParams.xy;
                float rawDepth = SampleSceneDepth(screenUV);
                float3 rayDir = normalize(input.rayDir);

                // 透视投影重建世界坐标
                float3 worldPos = ComputeWorldSpacePosition(
                    screenUV, rawDepth, UNITY_MATRIX_I_VP);

                // 2. 将世界坐标转换到贴花投影空间
                float4x4 decalMat = UNITY_ACCESS_INSTANCED_PROP(Props, _DecalMatrix);
                float3 localPos = mul(decalMat, float4(worldPos, 1.0)).xyz;

                // 3. 只在投影盒体范围内渲染
                clip(0.5 - abs(localPos.x));
                clip(0.5 - abs(localPos.y));
                clip(0.5 - abs(localPos.z));  // 深度裁剪

                // 4. 投影盒体内的坐标 → UV
                float2 decalUV = localPos.xy + 0.5;

                // 5. 从 Atlas 中取正确的贴花区域
                float4 atlasRect = UNITY_ACCESS_INSTANCED_PROP(Props, _AtlasUVRect);
                float2 finalUV = atlasRect.xy + decalUV * atlasRect.zw;

                // 6. 采样贴花
                half4 decalColor = SAMPLE_TEXTURE2D(_DecalTex, sampler_DecalTex, finalUV);

                // 7. 距离淡出
                float dist = length(worldPos - _WorldSpaceCameraPos);
                float fade = saturate(1.0 - (dist - _FadeDistance + 5.0) / 5.0);
                decalColor.a *= fade;

                // 8. 深度偏移修正（避免 Z-Fighting）
                clip(decalColor.a - 0.01);

                return decalColor;
            }
            ENDHLSL
        }
    }
}
```

**C# 贴花管理器（对象池 + GPU Instancing）：**

```csharp
// DecalManager.cs —— 管理运行时贴花生命周期
using UnityEngine;
using System.Collections.Generic;

public class DecalManager : MonoBehaviour
{
    public static DecalManager Instance { get; private set; }

    [Header("贴花配置")]
    public Material decalMaterial;
    public Mesh decalMesh;  // 单位盒体 Mesh
    public int maxDecals = 64;

    [Header("Atlas 配置")]
    public Texture2D decalAtlas;
    public Vector2[] atlasRects;  // 每种贴花类型的 UV 矩形

    // 贴花数据（存在 GPU 端）
    private Matrix4x4[] _matrices;
    private Vector4[] _atlasUVs;
    private MaterialPropertyBlock _mpb;

    // 对象池
    private struct DecalInstance
    {
        public Vector3 position;
        public Quaternion rotation;
        public float scale;
        public int atlasIndex;
        public float life;        // 剩余生命，-1 = 永久
        public bool active;
    }

    private DecalInstance[] _decals;
    private int _nextIndex = 0;

    void Awake()
    {
        Instance = this;
        _matrices = new Matrix4x4[maxDecals];
        _atlasUVs = new Vector4[maxDecals];
        _decals = new DecalInstance[maxDecals];
        _mpb = new MaterialPropertyBlock();
    }

    /// <summary>
    /// 在指定位置生成贴花
    /// </summary>
    public void SpawnDecal(Vector3 pos, Vector3 normal, int atlasIndex,
                           float scale = 1f, float life = -1f)
    {
        // 对象池循环复用
        int slot = _nextIndex;
        _nextIndex = (_nextIndex + 1) % maxDecals;

        // 贴花方向：沿命中面法线投影
        Quaternion rot = Quaternion.LookRotation(normal, Vector3.up);

        _decals[slot] = new DecalInstance
        {
            position = pos + normal * 0.02f,  // 沿法线偏移避免 Z-Fighting
            rotation = rot,
            scale = scale,
            atlasIndex = atlasIndex,
            life = life,
            active = true
        };
    }

    void Update()
    {
        int activeCount = 0;

        for (int i = 0; i < maxDecals; i++)
        {
            ref var d = ref _decals[i];
            if (!d.active) continue;

            // 生命周期衰减
            if (d.life > 0)
            {
                d.life -= Time.deltaTime;
                if (d.life <= 0)
                {
                    d.active = false;
                    continue;
                }
            }

            // 构建矩阵
            _matrices[activeCount] = Matrix4x4.TRS(
                d.position, d.rotation, Vector3.one * d.scale);
            _atlasUVs[activeCount] = atlasRects[d.atlasIndex];
            activeCount++;
        }

        // GPU Instancing 批量绘制
        if (activeCount > 0)
        {
            _mpb.SetVectorArray("_AtlasUVRect", _atlasUVs);
            Graphics.DrawMeshInstanced(
                decalMesh, 0, decalMaterial, _matrices,
                activeCount, _mpb,
                UnityEngine.Rendering.ShadowCastMode.Off, false);
        }
    }

    void OnGUI()
    {
        // Debug 信息
        GUILayout.Label($"Active Decals: {CountActive()}");
    }

    int CountActive()
    {
        int count = 0;
        foreach (var d in _decals)
            if (d.active) count++;
        return count;
    }
}
```

### ⚡ 实战经验

1. **Z-Fighting 是贴花第一大敌**：贴花必须沿命中面法线偏移 1-3cm，否则与原表面闪烁交错
2. **屏幕空间贴花的视野限制**：相机看不到的面就贴不上——对于角色身上的弹孔，需要切到 Mesh 贴花方案
3. **贴花图集规划要早**：后期让美术把 30 种贴花合并到 Atlas 非常痛苦，项目初期就定义好 Atlas 规格
4. **Decal Projector vs 屏幕空间**：如果项目用 Deferred Rendering，Decal Projector 是最佳选择（直接写 G-Buffer）；Forward 渲染的手游优先屏幕空间或 Mesh 方案
5. **贴花池大小调优**：FPS 游戏弹孔生成极快，池太小会导致旧弹孔突然消失——根据武器射速和场景面积调整
6. **贴花与光照的交互**：简单方案是贴花只覆盖 Albedo（不受光），高级方案让贴花参与光照计算（需要法线信息）

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能解释三种贴花方案（Projector/屏幕空间/Mesh）的优劣 | → 渲染方案盲区：理解不同贴花技术的适用场景 |
| 能手写屏幕空间贴花的深度重建世界坐标流程 | → 渲染管线盲区：理解深度缓冲与世界坐标的关系 |
| 知道 GPU Instancing 如何用于批量贴花渲染 | → 性能优化盲区：理解 Instanced Rendering 原理 |
| 能设计贴花对象池并管理生命周期 | → 架构设计盲区：对象池模式在渲染中的应用 |
| 知道贴花 Z-Fighting 的原因和解决方案 | → 渲染基础盲区：深度缓冲精度与偏移 |

### 🔗 相关问题

- [rendering/urp-renderer-feature](../rendering/urp-renderer-feature.md) — URP Renderer Feature 自定义
- [rendering/deferred-multi-light](../rendering/deferred-multi-light.md) — 延迟渲染下 G-Buffer 的利用
- [shader/body-paint-decal-projection](../shader/body-paint-decal-projection.md) — 角色身体涂装贴花投影
