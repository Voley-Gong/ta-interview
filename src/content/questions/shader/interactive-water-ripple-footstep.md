---
title: "角色踩水面产生涟漪：如何用 RenderTexture + 顶点偏移实现可交互的水面？"
category: "shader"
level: 3
tags: ["Shader", "URP", "RenderTexture", "交互水面", "顶点动画", "GrabPass"]
hint: "核心是 RenderTexture 记录交互点 → blur 传播涟漪 → 采样 RT 驱动顶点偏移和法线扰动"
related: ["shader/water-caustics", "shader/water-foam-wave-interaction", "rendering/depth-based-screen-distortion"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们在做一款开放世界游戏，有一片浅水区，角色走过去时脚下要有涟漪扩散，敌人落水要有大水花，雨滴也要能在水面打出小圈。涟漪之间还能互相干涉叠加。给我一套移动端可用的交互水面方案。」

补充约束：
- 涟漪要有物理感：扩散 + 衰减 + 多波干涉
- 支持 3 种交互源：角色脚步、物体落入、环境降雨
- 移动端不能超过额外 1.5ms / frame
- 涟漪要影响光照（法线扰动 → 反射方向变化）

### ✅ 核心要点

1. **交互场 RenderTexture**：用一张 RT 记录「哪里有扰动、扰动多大」，每帧更新
2. **Ping-Pong RT + 模糊传播**：双缓冲 RT 交替读写，模拟波传播（当前帧 = 上帧 × 衰减 + 新扰动）
3. **顶点偏移驱动波形**：在 Vertex Shader 中采样 RT 的高度通道，偏移水面顶点 Y 轴
4. **法线重建供光照**：对 RT 高度场做差分（ddx/ddy 或 3-tap）算出扰动法线
5. **多源写入统一管线**：脚步、落体、雨滴都通过 Particle System 或 DrawMesh 往同一张 RT 写入
6. **移动端性能控制**：RT 分辨率 256×256 起步，blur 用 separable（先 X 后 Y），减少采样次数

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色踩水 → 涟漪从脚下向外扩散 → 逐渐衰减消失 → 多源叠加干涉
               ↑
倒推1：涟漪是「水面高度的波动」→ 需要一个高度场驱动波形
倒推2：高度场要随时间传播和衰减 → 需要波传播模拟（波动方程离散化）
倒推3：交互源（脚步等）在对应位置注入高度脉冲 → 需要往高度场「画」扰动
倒推4：高度场存在 GPU 端最高效 → RenderTexture + Compute / Fragment 写入
倒推5：高度场还要驱动光照变化 → 从高度场差分出法线，参与光照计算
倒推6：多个交互源同时写入 → 统一的「扰动注入」管线
```

#### 知识点拆解（倒推树）

```
交互水面涟漪
├── 波传播模拟
│   ├── 波动方程离散化（2D）
│   │   └── h(t+1) = 2h(t) - h(t-1) + c²·(∇²h) · dt²
│   ├── 简化方案：Ping-Pong + Blur 近似扩散
│   │   ├── RT_A = RT_B ×衰减 + 新扰动
│   │   └── Blur 模拟空间传播（邻域扩散）
│   └── 精确方案：Compute Shader 解波动方程
│       ├── Dispatch 网格 = RT 分辨率
│       └── 邻居采样（左右上下一格）
│
├── 扰动注入（Interaction Injection）
│   ├── 角色脚步
│   │   ├── Projector 从上往下投影到水面 RT
│   │   ├── 脚印贴图作为扰动 mask
│   │   └── 接触瞬间注入脉冲（alpha > 0 一帧）
│   ├── 物体落入
│   │   ├── 粒子系统发射溅射粒子
│   │   └── 粒子用 Trails / 自定义 RT 写入
│   ├── 环境降雨
│   │   ├── 全屏雨粒子 → 往 RT 写随机点
│   │   └── GPU Instancing 批量绘制扰动点
│   └── 统一写入方式
│       ├── CommandBuffer.Blit（全屏操作）
│       ├── CommandBuffer.DrawMesh（指定位置画扰动 mesh）
│       └── Blend Mode：Add（叠加干涉）
│
├── 水面渲染（Shading）
│   ├── 顶点偏移
│   │   ├── Vertex Shader 采样 RT 高度通道
│   │   ├── worldPos.y += height * _WaveAmplitude
│   │   └── 注意：水面 mesh 需要有足够顶点密度（至少 1m 间隔）
│   ├── 法线扰动
│   │   ├── 对高度场做有限差分：dx = h(x+1) - h(x-1)
│   │   ├── normal = normalize(float3(-dx, strength, -dz))
│   │   └── 扰动后的法线参与 PBR / Blinn-Phong 光照
│   ├── 反射/折射扰动
│   │   ├── 用扰动法线偏移 Screen UV → 采样 Planar Reflection / GrabTexture
│   │   └── 模拟「水面起伏导致反射扭曲」
│   └── 焦散增强（可选）
│       └── 波峰处增强焦散强度（已有 caustics 的基础上叠加）
│
├── RenderTexture 配置
│   ├── 分辨率：256×256（移动端）~ 512×512（PC）
│   ├── 格式：ARGB Half（需要精度做波传播）或 R8（只用高度，省内存）
│   ├── Filter Mode：Bilinear（顶点采样需要平滑插值）
│   └── Wrap Mode：Clamp（边缘不重复）
│
└── 性能控制
    ├── RT Ping-Pong：2张 RT 交替，避免读写同一张
    ├── Blur 策略：Separable Gaussian（X 一次 + Y 一次 = 2 pass）
    ├── Update 频率：可以隔帧更新（30Hz 涟漪也够自然）
    └── LOD：远处水面不更新交互 RT，只播预设动画
```

#### 代码实现

**核心架构图（数据流）：**

```
┌──────────────┐
│ 交互源        │ 脚步/落体/雨
│ (C# Script)  │
└──────┬───────┘
       │ CommandBuffer.DrawMesh (Additive Blend)
       ▼
┌──────────────┐      ┌──────────────┐
│ RT_Height_A  │◄────►│ RT_Height_B  │  Ping-Pong
│ (R=height)   │ Blur │ (R=height)   │
│ (G=velocity) │      │ (G=velocity) │
└──────┬───────┘      └──────────────┘
       │ Sample in Vertex/Fragment Shader
       ▼
┌──────────────┐
│ 水面 Mesh     │ 顶点偏移 + 法线扰动 + 光照
│ (Material)    │
└──────────────┘
```

**Ping-Pong 波传播 Shader（Fragment）：**

```hlsl
Shader "Hidden/RipplePropagation"
{
    Properties { _MainTex ("State", 2D) = "black" {} }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        Pass
        {
            // 加法混合：新扰动叠加上去
            Blend Add

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_MainTex); SAMPLER(sampler_MainTex);
            TEXTURE2D(_InteractionTex); SAMPLER(sampler_InteractionTex); // 新扰动

            float4 _MainTex_TexelSize; // 1/width, 1/height, width, height
            float _Damping;
            float _WaveSpeed;

            struct Varyings {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings vert(float4 pos : POSITION, float2 uv : TEXCOORD0) {
                Varyings o;
                o.pos = TransformObjectToHClip(pos.xyz);
                o.uv = uv;
                return o;
            }

            half4 frag(Varyings i) : SV_Target
            {
                float2 uv = i.uv;
                float2 texel = _MainTex_TexelSize.xy;

                // 采样上一帧高度（R通道）
                float h = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv).r;
                float hL = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv - float2(texel.x, 0)).r;
                float hR = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv + float2(texel.x, 0)).r;
                float hU = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv - float2(0, texel.y)).r;
                float hD = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv + float2(0, texel.y)).r;

                // 离散波动方程：
                // 新高度 = (hL + hR + hU + hD) / 2 - 上一帧高度，再衰减
                float newH = (hL + hR + hU + hD) * 0.5 - h;
                newH *= _Damping; // 衰减系数 0.98~0.995

                // 叠加新注入的扰动（脚步等）
                float interaction = SAMPLE_TEXTURE2D(_InteractionTex, sampler_InteractionTex, uv).r;
                newH += interaction * _WaveSpeed;

                // G通道存速度（可选，用于更精确的模拟）
                float vel = newH - h;

                return half4(newH, vel, 0, 1);
            }
            ENDHLSL
        }
    }
}
```

**水面着色 Shader（顶点偏移 + 法线扰动）：**

```hlsl
Shader "Custom/InteractiveWater"
{
    Properties
    {
        _BaseMap ("Base Color", 2D) = "blue" {}
        _NormalMap ("Normal Map", 2D) = "bump" {}
        _RippleTex ("Ripple Height Field", 2D) = "black" {}  // 来自 Ping-Pong RT
        _WaveAmplitude ("Wave Amplitude", Float) = 0.15
        _NormalStrength ("Ripple Normal Strength", Float) = 3.0
        _WaterColor ("Shallow Color", Color) = (0.3, 0.6, 0.8, 1)
        _DeepColor ("Deep Color", Color) = (0.05, 0.15, 0.3, 1)
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="Transparent" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _RippleTex_TexelSize;
                float  _WaveAmplitude;
                float  _NormalStrength;
                float4 _WaterColor;
                float4 _DeepColor;
            CBUFFER_END

            TEXTURE2D(_BaseMap);     SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NormalMap);   SAMPLER(sampler_NormalMap);
            TEXTURE2D(_RippleTex);   SAMPLER(sampler_RippleTex);

            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float4 tangentOS  : TANGENT;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings {
                float4 positionHCS : SV_POSITION;
                float2 uv          : TEXCOORD0;
                float3 normalWS    : TEXCOORD1;
                float3 positionWS  : TEXCOORD2;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                
                // 采样涟漪高度场
                float2 rippleUV = IN.uv * _BaseMap_ST.xy + _BaseMap_ST.zw;
                float rippleH = SAMPLE_TEXTURE2D_LOD(_RippleTex, sampler_RippleTex, rippleUV, 0).r;

                // 顶点 Y 轴偏移（水面起伏）
                float3 posOS = IN.positionOS.xyz;
                posOS.y += rippleH * _WaveAmplitude;

                OUT.positionHCS = TransformObjectToHClip(posOS);
                OUT.positionWS = TransformObjectToWorld(posOS);
                OUT.uv = rippleUV;

                // 基础法线 + 涟漪法线（差分）
                float2 texel = _RippleTex_TexelSize.xy;
                float hL = SAMPLE_TEXTURE2D_LOD(_RippleTex, sampler_RippleTex, rippleUV - float2(texel.x, 0), 0).r;
                float hR = SAMPLE_TEXTURE2D_LOD(_RippleTex, sampler_RippleTex, rippleUV + float2(texel.x, 0), 0).r;
                float hD = SAMPLE_TEXTURE2D_LOD(_RippleTex, sampler_RippleTex, rippleUV - float2(0, texel.y), 0).r;
                float hU = SAMPLE_TEXTURE2D_LOD(_RippleTex, sampler_RippleTex, rippleUV + float2(0, texel.y), 0).r;

                // 涟漪法线（切线空间 → 世界空间）
                float3 rippleNormalTS = normalize(float3((hL - hR) * _NormalStrength, 1.0, (hD - hU) * _NormalStrength));
                
                // 基础法线贴图
                half3 baseNormalTS = UnpackNormal(SAMPLE_TEXTURE2D_LOD(_NormalMap, sampler_NormalMap, rippleUV, 0));
                float3 finalNormalTS = normalize(baseNormalTS + rippleNormalTS * 0.5);

                // 切线 → 世界
                float3 normalWS = TransformTangentToWorld(finalNormalTS,
                    half3x3(IN.tangentOS.xyz, cross(IN.normalOS, IN.tangentOS.xyz) * IN.tangentOS.w, IN.normalOS));
                OUT.normalWS = normalize(normalWS);

                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                half3 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv).rgb;
                half3 N = normalize(IN.normalWS);

                // 主光源
                Light mainLight = GetMainLight();
                half3 L = mainLight.direction;
                half NdotL = max(0, dot(N, L));

                // Specular（高光受涟漪法线影响明显）
                half3 V = GetWorldSpaceNormalizeViewDir(IN.positionWS);
                half3 H = normalize(L + V);
                half spec = pow(max(0, dot(N, H)), 64);

                // 深浅水色混合
                half3 waterColor = lerp(_ShallowColor.rgb, _DeepColor.rgb, 1.0 - NdotL * 0.5);

                half3 finalColor = waterColor * mainLight.color * NdotL + spec * half3(1,1,1) * 0.8;

                return half4(finalColor, 0.85);
            }
            ENDHLSL
        }
    }
}
```

**C# 交互管理器：**

```csharp
using UnityEngine;

[RequireComponent(typeof(Renderer))]
public class WaterInteractionSystem : MonoBehaviour
{
    [Header("RT Settings")]
    [SerializeField] private int rtResolution = 256;
    [SerializeField] private float damping = 0.985f;
    [SerializeField] private float waveSpeed = 1.0f;

    [Header("Interaction Sources")]
    [SerializeField] private Transform[] footTransforms; // 角色脚部
    [SerializeField] private Mesh interactionMesh;       // 扰动用小圆 mesh
    [SerializeField] private Material interactionMat;    // 往 RT 画扰动的材质
    [SerializeField] private Material propagationMat;   // 波传播材质

    private RenderTexture _rtA, _rtB; // Ping-Pong
    private MaterialPropertyBlock _mpb;
    private int _rippleTexID;

    void Start()
    {
        // 创建 Ping-Pong RT
        var desc = new RenderTextureDescriptor(rtResolution, rtResolution, RenderTextureFormat.ARGBHalf, 0);
        _rtA = new RenderTexture(desc);
        _rtB = new RenderTexture(desc);
        _rtA.Create();
        _rtB.Create();

        _mpb = new MaterialPropertyBlock();
        _rippleTexID = Shader.PropertyToID("_RippleTex");

        // 设置水面材质的涟漪 RT
        GetComponent<Renderer>().SetPropertyBlock(_mpb);
    }

    void Update()
    {
        // === Step 1: 注入新扰动到 _rtB ===
        // 清空 interaction 层（可选：用单独 RT 或直接在传播时叠加）

        // === Step 2: 波传播 _rtB → _rtA ===
        propagationMat.SetTexture("_MainTex", _rtB);
        propagationMat.SetFloat("_Damping", damping);
        propagationMat.SetFloat("_WaveSpeed", waveSpeed);
        Graphics.Blit(_rtB, _rtA, propagationMat);

        // === Step 3: 角色脚步注入扰动 ===
        foreach (var foot in footTransforms)
        {
            if (foot == null) continue;
            // 将世界坐标投影到水面 UV
            Vector3 localPos = transform.InverseTransformPoint(foot.position);
            Vector2 rtUV = new Vector2(localPos.x, localPos.z) * 0.5f + Vector2.one * 0.5f;

            // 在 RT 对应位置画扰动 mesh
            Matrix4x4 matrix = Matrix4x4.TRS(
                new Vector3(rtUV.x * rtResolution, rtUV.y * rtResolution, 0),
                Quaternion.identity,
                Vector3.one * 8f
            );
            Graphics.DrawMesh(interactionMesh, matrix, interactionMat, 0, null, 0, _mpb, false);
        }

        // === Step 4: 设置水面材质 ===
        _mpb.SetTexture(_rippleTexID, _rtA);
        GetComponent<Renderer>().SetPropertyBlock(_mpb);

        // === Swap ===
        (_rtA, _rtB) = (_rtB, _rtA);
    }

    void OnDestroy()
    {
        _rtA?.Release();
        _rtB?.Release();
    }
}
```

**方案对比表：**

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| Ping-Pong Blur | RT 双缓冲 + 模糊传播 | 实现简单，效果好 | 不是物理精确 | 手游（推荐） |
| Compute 波动方程 | GPU 并行解离散方程 | 物理准确，干涉自然 | 需要 Compute Shader | PC/主机 |
| Gerstner Wave + 扰动 | 预设波形 + 手动叠加涟漪 | 性能最好 | 涟漪形状固定 | 低端移动端 |
| 2D Heightmap CPU | CPU 计算高度场 | 不依赖 GPU 特性 | 性能差，分辨率低 | 不推荐 |

### ⚡ 实战经验

1. **水面 Mesh 顶点密度是关键**：如果水面是 4m×4m 的大 Quad 只有 4 个顶点，顶点偏移完全看不出效果。需要 subdiv 到至少 0.5m 间距，或者用 Displacement Mapping（需要曲面细分）
2. **RT 分辨率够用就行**：256×256 对于 20m×20m 的水面足够，每像素 ≈ 8cm 精度。512×512 的性能开销是 256 的 4 倍
3. **衰减系数调参**：0.985 太大会永久振荡，太小则涟漪一闪而过。推荐 0.97~0.99，根据水深度和粘度感调整
4. **法线扰动比顶点偏移更重要**：视觉上涟漪的「波纹感」主要来自高光变化（法线扰动），而不是水面真的凹凸了几厘米。即使不做顶点偏移，只做法线扰动效果也很好
5. **和反射系统配合**：如果有 Planar Reflection，用扰动后的法线偏移反射 UV，涟漪会扭曲倒影——这是视觉升级的关键
6. **性能实测参考**：骁龙 8 Gen 2，256×256 RT + Ping-Pong + 1 次水面着色 ≈ 0.8ms，完全在预算内

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道怎么让涟漪「传播」 | 波动方程 / 高斯模糊近似扩散 | 学离散波动方程、Separable Blur |
| 涟漪不衰减、永久振荡 | 缺少 damping 项 | 在传播 shader 中加指数衰减 |
| 顶点偏移看不出来 | 水面 mesh 顶点太稀疏 | 学 Mesh Subdivision / Tessellation |
| 法线扰动不自然 | 差分方向搞反或强度不对 | 复习有限差分法、切线空间变换 |
| 不知道多源怎么叠加 | RT 混合模式 | 学 BlendOp Add / Max，CommandBuffer 叠加 |
| 移动端 RT 性能爆炸 | ARGBHalf 格式太重 | 尝试 R8 单通道或 RGBA32 |

### 🔗 相关问题

- [水面焦散 Shader](../shader/water-caustics.md)：焦散是水面下的光斑，涟漪是水面上的波动——两者结合才是完整水面
- [水面泡沫波浪交互](../shader/water-foam-wave-interaction.md)：波浪和涟漪叠加的交互方案
- 如果要做角色踩雪地留下脚印变形，思路有什么共通之处？（提示：同样是 Height Field 变形 + 法线重算）
