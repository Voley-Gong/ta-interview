---
title: "卡通渲染管线：如何从零搭建角色的 Cel-Shading 全套效果？"
category: "rendering"
level: 3
tags: ["Cel-Shading", "NPR", "卡通渲染", "Ramp", "Outline", "URP", "米哈游"]
hint: "核心是光照 Ramp + 背面描边 + 面部特殊 shading——卡通渲染不是'简化光照'，是'重新定义光照'"
related: ["shader/npr-outline-cartoon", "rendering/urp-renderer-feature", "shader/hair-anisotropic-lighting", "rendering/sss-skin-rendering"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们要做一款二次元风格的游戏，角色需要完整的卡通渲染效果。包括：卡通光影（硬边分界）、描边、面部阴影（SDF 精确控制）、头发高光、衣服褶皱。在 URP 下搭建整套 NPR（Non-Photorealistic Rendering）管线，你来做方案。」

这是米哈游、鹰角、叠纸等二次元项目 TA 岗位的**核心必考题**。不考察单个 Shader，而是考察你对整个角色渲染管线的理解和架构能力。

### ✅ 核心要点

1. **光照 Ramp 纹理**：用 1D/2D Ramp 纹理将连续光照量化为色阶
2. **描边系统**：背面法线外扩 + 颜色控制（不同部位不同描边色/宽度）
3. **面部 SDF 阴影**：用 SDF 贴图精确控制面部阴影边界（避免鼻子/眼眶诡异阴影）
4. **头发高光**：基于法线偏移 + 金属感高光带（Kajiya-Kay 变体）
5. **Shader 分区管理**：面部、头发、衣服、皮肤各自不同的 Shader 逻辑
6. **管线架构**：多 Pass + Renderer Feature 在 URP 中组织渲染流程

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：二次元角色完整渲染
├── 卡通光影（明暗硬分界）
├── 描边（不同部位不同颜色和粗细）
├── 面部阴影（精确控制，不靠物理光照）
├── 头发高光（天使环效果）
├── 衣服渲染（色阶 + 细节描边）
└── 整体协调（风格统一）

倒推实现路径：

Step 1：基础 Cel-Shading
  → NdotL → Ramp 纹理采样 → 硬色阶
  → 暗部色、亮部色、环境光补偿

Step 2：描边系统
  → Pass 1: Cull Front + 顶点法线外扩
  → 顶点色控制描边宽度
  → UV2 存储描边颜色

Step 3：面部特殊处理
  → 不用物理光照（鼻子/眼眶会出问题）
  → SDF 贴图：记录面部阴影变化场
  → 光照方向 → 采样 SDF → 阴影 mask

Step 4：头发高光
  → 偏移法线后做 Kajiya-Kay
  → 噪声纹理打散高光带
  → 高光偏移和形状可调

Step 5：Shader 分区
  → 用 SubShader 或 Shader 关键字切换
  → 面部/头发/身体用不同材质
  → 统一的参数命名规范

Step 6：URP 管线组织
  → Renderer Feature 管理描边 Pass
  → 透明物体排序
  → 后处理配合（Bloom 控制高光溢出）
```

#### 知识点拆解（倒推树）

```
卡通渲染完整管线
├── 基础 Cel-Shading
│   ├── NdotL 计算（兰伯特光照量化）
│   ├── Ramp 纹理（1D 梯度 / 2D 区域控制）
│   ├── 色阶混合（smoothstep 硬边 vs soft edge）
│   ├── 环境光补偿（球谐光照 + 角色 AO）
│   └── 方向光颜色 + 半兰伯特封装
├── 描边系统
│   ├── 背面法线外扩（Cull Front + vertex normal * width）
│   ├── 顶点色控制（R=宽度，G=颜色ID）
│   ├── 平滑法线传递（Maya/Houdini 烘焙平滑法线到 UV2）
│   ├── 描边颜色策略（固定色 vs 顶点色 vs 采样贴图）
│   ├── Z-Fight 阄值控制（depth bias）
│   └── 距离自适应描边宽度（screen-space width）
├── 面部 SDF 阴影
│   ├── SDF 贴图原理（Signed Distance Field 存储shadow边界）
│   ├── 光照方向 → UV 偏移采样
│   ├── 阴影 mask 生成（binary 0/1 边界）
│   ├── 多角度 SDF（正面/侧面/背面 LUT）
│   └── 边缘柔化（smoothstep 控制 mask 过渡）
├── 头发高光
│   ├── Kajiya-Kay 模型（头发各向异性）
│   ├── 切线方向偏移（控制高光位置）
│   ├── 噪声纹理（打散连续高光带）
│   ├── 双层高光（diffuse + specular shift）
│   └── 天使环（rim light 效果）
├── 衣服渲染
│   ├── 色阶 ramp（不同材质不同 ramp）
│   ├── 细节描边（内描边 / cross hatch）
│   ├── 法线贴图驱动细节阴影
│   └── 材质感区分（丝绸 vs 棉布 vs 金属）
├── URP 管线架构
│   ├── Shader 分区（多材质多 Shader）
│   ├── Renderer Feature（描边 Pass）
│   ├── Render Object 特性（控制渲染顺序）
│   ├── 全局光照参数（方向光/SH 传入 Shader）
│   └── 后处理配合（Bloom / Color Grading）
└── 性能与规范
    ├── 贴图预算（Ramp/SDF/噪声 共享）
    ├── Shader 变体控制（keyword 管控）
    ├── 移动端简化（降 Ramp 精度，去 SDF 改 vertex 计算）
    └── 合批策略（同 Shader 同材质才能合批）
```

#### 代码实现

**1. 基础 Cel-Shading Shader（身体部分）：**

```hlsl
Shader "NPR/CharacterBody"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _ShadowRamp ("Shadow Ramp (1D)", 2D) = "white" {}
        _ShadowColor ("Shadow Color", Color) = (0.6, 0.5, 0.7, 1)
        _BrightColor ("Bright Color", Color) = (1, 1, 1, 1)
        _RampThreshold ("Ramp Threshold", Range(-1, 1)) = 0.0
        _RampSmooth ("Ramp Smoothness", Range(0, 0.5)) = 0.02
        _SpecularColor ("Specular Color", Color) = (1, 1, 1, 1)
        _SpecularPower ("Specular Power", Float) = 32
        _RimColor ("Rim Color", Color) = (1, 1, 1, 1)
        _RimPower ("Rim Power", Float) = 4.0
        _OutlineColor ("Outline Color", Color) = (0.1, 0.1, 0.1, 1)
        _OutlineWidth ("Outline Width", Range(0, 0.01)) = 0.002
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }

        // === Pass 0: 描边（背面法线外扩） ===
        Pass
        {
            Name "OUTLINE"
            Tags { "LightMode"="SRPDefaultUnlit" }
            Cull Front
            ZWrite On

            HLSLPROGRAM
            #pragma vertex vertOutline
            #pragma fragment fragOutline
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _OutlineColor;
                float _OutlineWidth;
            CBUFFER_END

            struct Attr {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float4 vertexColor : COLOR; // r = width scale
            };

            struct Vary {
                float4 positionHCS : SV_POSITION;
            };

            Vary vertOutline(Attr IN)
            {
                Vary OUT;
                // 法线外扩 + 顶点色宽度控制
                float width = _OutlineWidth * IN.vertexColor.r;
                float3 pos = IN.positionOS.xyz + IN.normalOS * width;
                OUT.positionHCS = TransformObjectToHClip(pos);
                return OUT;
            }

            half4 fragOutline(Vary IN) : SV_Target
            {
                return half4(_OutlineColor.rgb, 1);
            }
            ENDHLSL
        }

        // === Pass 1: 主光照 ===
        Pass
        {
            Name "MAIN"
            Tags { "LightMode"="UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _ShadowColor;
                float4 _BrightColor;
                float4 _SpecularColor;
                float4 _RimColor;
                float _RampThreshold;
                float _RampSmooth;
                float _SpecularPower;
                float _RimPower;
            CBUFFER_END

            TEXTURE2D(_BaseMap); SAMPLER(sampler_BaseMap);
            TEXTURE2D(_ShadowRamp); SAMPLER(sampler_ShadowRamp);

            struct Attr {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Vary {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 normalWS : TEXCOORD1;
                float3 viewDirWS : TEXCOORD2;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Vary vert(Attr IN)
            {
                Vary OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                OUT.viewDirWS = GetWorldSpaceNormalizeViewDir(TransformObjectToWorld(IN.positionOS.xyz));
                return OUT;
            }

            half4 frag(Vary IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);
                Light mainLight = GetMainLight();
                float3 N = normalize(IN.normalWS);
                float3 L = normalize(mainLight.direction);
                float3 V = normalize(IN.viewDirWS);
                float3 H = normalize(L + V);

                // === Cel-Shading 核心：半兰伯特 + Ramp ===
                float NdotL = dot(N, L);
                float halfLambert = NdotL * 0.5 + 0.5; // [0,1] 映射
                float rampCoord = saturate(halfLambert + _RampThreshold);
                // 硬色阶
                float ramp = smoothstep(0.5 - _RampSmooth, 0.5 + _RampSmooth, rampCoord);

                // Ramp 纹理采样（可选，更灵活的色彩控制）
                half3 rampColor = SAMPLE_TEXTURE2D(_ShadowRamp, sampler_ShadowRamp, float2(ramp, 0.5)).rgb;

                // 明暗色混合
                half3 baseCol = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv).rgb;
                half3 shadowCol = baseCol * _ShadowColor.rgb * rampColor;
                half3 brightCol = baseCol * _BrightColor.rgb * rampColor;
                half3 diffuse = lerp(shadowCol, brightCol, ramp);

                // === 高光 ===
                float NdotH = saturate(dot(N, H));
                float spec = pow(NdotH, _SpecularPower);
                spec = step(0.5, spec); // 卡通硬边高光
                diffuse += _SpecularColor.rgb * spec;

                // === 边缘光 ===
                float fresnel = 1.0 - saturate(dot(N, V));
                fresnel = pow(fresnel, _RimPower);
                diffuse += _RimColor.rgb * fresnel * 0.5;

                // === 环境光 ===
                half3 SH = SampleSH(N) * baseCol * 0.3;
                diffuse += SH;

                return half4(diffuse, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**2. 面部 SDF 阴影 Shader（核心片段）：**

```hlsl
// 面部不使用物理光照，用 SDF 贴图控制阴影边界
// SDF 贴图：R 通道存储有符号距离，0 为阴影边界
half3 FaceShading(float2 uv, float3 lightDirWS, float3 faceForwardWS)
{
    // 将光照方向投影到面部平面
    float2 lightProj;
    lightProj.x = dot(normalize(lightDirWS), faceForwardWS);
    lightProj.y = dot(normalize(lightDirWS), cross(faceForwardWS, float3(0,1,0)));

    // 用光照方向采样 SDF LUT（2D 查找表）
    // SDF LUT 布局：横向=光照水平角度，纵向=光照垂直角度
    float2 sdfUV = uv;
    sdfUV.x += lightProj.x * 0.1; // 根据光照偏移 UV

    half sdfValue = SAMPLE_TEXTURE2D(_FaceSDF, sampler_FaceSDF, sdfUV).r;
    half faceShadow = step(0.5, sdfValue); // 0 = 阴影, 1 = 亮部

    // 柔化边缘
    faceShadow = smoothstep(0.45, 0.55, sdfValue);

    // 面部基础色
    half3 baseCol = SAMPLE_TEXTURE2D(_FaceMap, sampler_FaceMap, uv).rgb;
    half3 shadowCol = baseCol * _FaceShadowColor.rgb;
    half3 brightCol = baseCol * _FaceBrightColor.rgb;

    return lerp(shadowCol, brightCol, faceShadow);
}
```

**3. 描边系统架构对比：**

| 描边方案 | 原理 | 优点 | 缺点 | 适用场景 |
|---------|------|------|------|----------|
| 背面法线外扩 | Cull Front + 法线偏移 | 简单高效 | 硬边处断裂 | 主流方案 |
| 法线平滑烘焙 | 预处理平滑法线到 UV2 | 解决硬边断裂 | 需要美术管线 | 高质量描边 |
| 后处理描边 | 边缘检测（Sobel/Roberts） | 全屏统一 | 无法控制线条粗细 | 补充描边 |
| Geometry Shader | 几何着色器生成描边 | 拓扑感知 | 移动端不支持 | PC/主机 |
| 倒角描边 | 额外 Mesh 包边 | 完美控制 | 建模成本高 | 高品质角色 |

**URP Renderer Feature 配置（描边作为独立 Pass）：**

```csharp
// 描述：使用 Render Objects Renderer Feature 添加描边 Pass
// 配置步骤（Unity 编辑器）：
// 1. URP Renderer Data → Add Renderer Feature → Render Objects
// 2. Name: Character Outline
// 3. Event: After Rendering Opaques
// 4. Filters → Layer Mask: Character
// 5. Overrides → Material: OutlineMaterial（Cull Front 的材质）
// 6. 将角色描边 Shader 的 Pass 0 设为 SRPDefaultUnlit
```

### ⚡ 实战经验

- **描边宽度在屏幕空间**：用 `TransformWorldToHClip` 后的 `positionCS.w` 归一化，保证描边宽度不随距离变化
- **Ramp 纹理不只是梯度**：用 2D Ramp（横轴=NdotL，纵轴=材质ID）可以让不同部位共用一张 Ramp
- **面部阴影是二次元最核心的技术难点**：物理光照在面部会产生鼻子下方、眼眶周围的诡异阴影——SDF 方案是米哈游/叠纸的标配
- **头发高光偏移**：用切线方向控制高光位置，美术需要能直观调整
- **描边色不是纯黑**：暗部偏冷色（蓝紫）、亮部偏暖色——描边也参与色彩表达
- **合批断裂问题**：不同部位的 Shader/材质不同会打断合批，SRP Batcher 可缓解但不是万能

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 卡通光影分界太硬或太软 | smoothstep / ramp 控制 | 学 smoothstep 原理与调试 |
| 描边在硬边处断裂 | 法线平滑 / 烘焙 | 学平滑法线传递管线 |
| 面部阴影乱跑 | SDF 面部阴影原理 | 学 SDF 贴图制作与采样 |
| 头发高光是一条死直线 | Kajiya-Kay + 噪声扰动 | 学头发各向异性光照 |
| 性能掉帧（角色多） | Shader 变体 / 合批 | 学 SRP Batcher + 变体管理 |
| 整体风格不统一 | Ramp 色彩管理 | 学色彩管理 + LUT 工作流 |

### 🔗 相关问题

- 如何实现动态天气（雨/雪）下的卡通角色受影响效果？
- 多角色同屏渲染时如何管理 Shader 变体和合批？
- 如何将物理渲染（PBR）和卡通渲染（NPR）在同一场景中共存？
- 移动端做卡通渲染需要砍掉哪些效果？（Ramp 精度？SDF？描边？）
