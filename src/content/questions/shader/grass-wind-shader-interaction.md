---
title: "开放世界草地 Shader：风吹摇摆 + 角色踩踏交互怎么做？"
category: "shader"
level: 3
tags: ["草地Shader", "顶点动画", "噪声", "交互", "RenderTexture", "URP", "开放世界"]
hint: "风吹用正弦+噪声驱动顶点弯曲，交互用 RenderTexture 记录角色位置→采样扰动——两层叠加才有真实感"
related: ["optimization/million-grass-rendering", "shader/interactive-water-ripple-footstep", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

面试官说：

> "我们开放世界有一大片草地，美术要求草要随风摆动，角色走过去时草要被踩弯，走过后慢慢恢复。你怎么做这个 Shader？性能预算 1ms。"

这是腾讯天美、网易盘古 TA 面试的经典实战题——考察你是否能把自然现象建模为数学函数，同时兼顾工程性能。

### ✅ 核心要点

1. **风吹摇摆**：用正弦波 + 噪声纹理模拟自然风场，按草的高度做权重（根部不动、顶部摆）
2. **交互踩踏**：用 RenderTexture 记录角色位置，Shader 采样后做距离衰减的弯曲
3. **恢复机制**：交互 RT 每帧 Fade（乘以 0.95），让踩踏痕迹自然消退
4. **性能控制**：顶点级动画而非像素级，草叶用 Billboard/Cross-mesh 而非真实几何

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：草地随风摆 + 角色踩弯 + 走过后恢复
    ↑
两层叠加
├── 风层（持续，全局）
│   ├── 顶点 XZ 偏移：sin(time + worldPos * freq) * amplitude
│   ├── 噪声扰动叠加：打破规律性
│   └── 高度权重：root 不动，tip 摆动最大
│       └── 用顶点色的 G 通道（或 UV.y）编码草叶高度
│
└── 交互层（瞬时，局部）
    ├── RenderTexture（R=踩踏强度, G=方向）
    │   ├── 角色/物体每帧在 RT 上画一个圆（CommandBuffer DrawMesh）
    │   └── RT 每帧 Fade：Blit multiply 0.95
    ├── Shader 采样 RT → 距离场弯曲
    │   ├── uv = worldPos.xz / terrainSize
    │   ├── interactColor = tex2D(_InteractTex, uv)
    │   └── bendDir = interactColor.r * directionFromG
    └── 恢复 = RT Fade（不需要 Shader 端做）
```

#### 知识点拆解（倒推树）

```
草地 Shader
├── 1. 草叶建模
│   ├── Cross-quad / Billboard / 真实 mesh（性能三角）
│   ├── 顶点色编码：
│   │   ├── R = AO / 遮蔽
│   │   ├── G = 高度参数（0=根，1=尖）
│   │   ├── B = 随机种子（让每株草不同步）
│   │   └── A = 风力权重
│   └── UV.y = 从根(0)到尖(1)
│
├── 2. 风场模拟
│   ├── 基础风：sin(time * windSpeed + worldPos.x * freq) 
│   ├── 噪声风：tex2D(_NoiseTex, worldPos.xz * scale + time * dir)
│   ├── 两层叠加：低频大摆 + 高频细颤
│   ├── 方向控制：_WindDir float2（可以是全局风向）
│   └── 暴风模式：amplitude 随天气系统变化
│
├── 3. 交互系统
│   ├── RenderTexture 方案（经典做法）
│   │   ├── 分辨率：256×256 或 512×512（足够）
│   │   ├── 格式：ARGBHalf（HDR 精度）或 RGHalf
│   │   ├── 坐标映射：worldPos.xz → UV
│   │   └── 写入：CommandBuffer + DrawMesh（画一个角色脚下的圆盘）
│   ├── 替代方案：StructuredBuffer（Compute Shader 写入）
│   │   └── 适合多角色/怪物大规模交互
│   └── RT Fade：每帧 Blit(_InteractRT, _TempRT, _FadeMaterial)
│       └── Fade Material = Multiply 0.93~0.97
│
├── 4. 弯曲数学
│   ├── 风弯曲：
│   │   bendX = sin(time * speed + pos.x * f) * amp * heightWeight
│   │   bendZ = cos(time * speed + pos.z * f) * amp * heightWeight
│   ├── 交互弯曲：
│   │   float2 interactUV = pos.xz / _TerrainSize;
│   │   float4 interact = tex2Dlod(_InteractTex, float4(interactUV,0,0));
│   │   float interactStrength = interact.r;
│   │   float2 interactDir = (interact.ga - 0.5) * 2; // decode direction
│   │   bendX += interactDir.x * interactStrength * bendScale;
│   │   bendZ += interactDir.y * interactStrength * bendScale;
│   └── 保底：总弯曲量不超过草叶长度（避免 90° 翻转）
│
└── 5. 性能优化
    ├── 草叶 mesh 控制：4顶点 cross-quad 是极致省版
    ├── 顶点 Shader 做所有动画，Fragment 只做颜色
    ├── 交互 RT 降分辨率到 128×128（远处草地不需要精确交互）
    ├── 视锥外的草地不更新交互 RT
    └── LOD 远景：直接用 impostor + 纯 Shader 风吹（无交互）
```

#### 代码实现

**完整草叶 Shader（URP）**

```hlsl
Shader "Custom/GrassWind" {
    Properties {
        _BaseMap ("Base Texture", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (0.3, 0.6, 0.2, 1)
        _NoiseTex ("Wind Noise", 2D) = "white" {}
        _WindSpeed ("Wind Speed", Float) = 1.5
        _WindStrength ("Wind Strength", Float) = 0.3
        _WindDir ("Wind Direction", Vector) = (1, 0, 0.5, 0)
        _InteractTex ("Interaction RT", 2D) = "black" {}
        _TerrainSize ("Terrain Size", Float) = 100
        _InteractStrength ("Interaction Strength", Float) = 1.5
        _FadeRate ("Fade Rate", Float) = 0.95
    }

    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Cull Off // 草叶双面

        Pass {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
                float4 color      : COLOR; // vertex color encoding
            };

            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 color      : TEXCOORD1;
            };

            TEXTURE2D(_BaseMap);       SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NoiseTex);      SAMPLER(sampler_NoiseTex);
            TEXTURE2D(_InteractTex);   SAMPLER(sampler_InteractTex);

            float _WindSpeed, _WindStrength, _TerrainSize, _InteractStrength;
            float4 _WindDir;

            Varyings vert(Attributes input) {
                Varyings output;
                
                // 草叶高度权重：UV.y 0=根 1=尖，顶点色G备用
                float heightWeight = input.uv.y * input.uv.y; // 二次曲线，顶部摆动更大
                // 也可以用 input.color.g
                
                // === 风场 ===
                float3 worldPos = TransformObjectToWorld(input.positionOS.xyz);
                
                // 低频风（大摆）
                float windPhase = _Time.y * _WindSpeed + worldPos.x * 0.1 + worldPos.z * 0.08;
                float2 bigWind = float2(sin(windPhase), cos(windPhase * 0.8));
                
                // 高频噪声风（细颤）
                float2 noiseUV = worldPos.xz * 0.02 + _Time.y * _WindDir.xy * 0.05;
                float4 noise = SAMPLE_TEXTURE2D_LOD(_NoiseTex, sampler_NoiseTex, float4(noiseUV, 0, 0));
                float2 detailWind = (noise.rg - 0.5) * 2;
                
                // 合成风力
                float2 windForce = (bigWind * 0.7 + detailWind * 0.3) * _WindStrength * heightWeight;
                windForce += _WindDir.xy * _WindStrength * 0.3 * heightWeight; // 全局风向偏置
                
                // === 交互踩踏 ===
                float2 interactUV = worldPos.xz / _TerrainSize + 0.5;
                float4 interact = SAMPLE_TEXTURE2D_LOD(_InteractTex, sampler_InteractTex, float4(interactUV, 0, 0));
                float interactStrength = interact.r;
                float2 interactDir = (interact.ga - 0.5) * 2; // R=强度, GA=方向
                
                float2 interactForce = interactDir * interactStrength * _InteractStrength * heightWeight;
                
                // === 合成位移 ===
                float2 totalOffset = windForce + interactForce;
                
                // 限制最大弯曲（避免翻转）
                float maxBend = 0.5;
                totalOffset = min(totalOffset, maxBend);
                
                // 应用到顶点
                input.positionOS.xz += totalOffset.xxy * float3(1, 0, 1); // X和Z偏移
                
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                
                // AO 顶点色传递给片元做明暗
                output.color = input.color.rgb;
                return output;
            }

            half4 frag(Varyings input) : SV_Target {
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;
                // 根部暗，尖部亮
                float ao = lerp(0.5, 1.0, input.uv.y);
                albedo.rgb *= ao;
                return albedo;
            }
            ENDHLSL
        }
    }
}
```

**交互 RT 更新脚本（C#）**

```csharp
public class GrassInteraction : MonoBehaviour {
    private RTHandle m_InteractRT;
    private RTHandle m_TempRT;
    private Material m_FadeMaterial;
    public Material m_DrawMaterial; // 角色脚下圆盘材质
    public float m_TerrainSize = 100f;
    
    // 所有需要交互的角色/物体
    private List<Matrix4x4> m_ObjectMatrices = new();

    void Update() {
        // 1. Fade 历史 RT（让踩踏痕迹消退）
        m_FadeMaterial.SetFloat("_Fade", 0.94f);
        Graphics.Blit(m_InteractRT, m_TempRT, m_FadeMaterial);
        
        // 2. 画角色位置
        foreach (var mat in m_ObjectMatrices) {
            // DrawMesh 在 RT 上画一个圆盘，编码位置和方向
            Graphics.DrawMesh(m_QuadMesh, mat, m_DrawMaterial, 0, m_Camera, 0);
        }
        
        // 3. 交换 RT
        (m_InteractRT, m_TempRT) = (m_TempRT, m_InteractRT);
        
        // 4. 传给 Shader
        Shader.SetGlobalTexture("_InteractTex", m_InteractRT);
        Shader.SetGlobalFloat("_TerrainSize", m_TerrainSize);
    }
    
    public void RegisterInteractor(Vector3 worldPos, Vector3 velocity) {
        // 构建 MVP 矩阵把圆盘画到 RT 的正确位置
        Vector2 uv = new Vector2(worldPos.x / m_TerrainSize + 0.5f,
                                  worldPos.z / m_TerrainSize + 0.5f);
        // ... 构建正交投影矩阵画到 UV 位置
    }
}
```

### ⚡ 实战经验

1. **顶点色编码是草地 Shader 的灵魂**——R=AO、G=高度、B=随机种子、A=风力权重，一次烘焙终身受益
2. **噪声纹理用 RGA 通道分别存不同尺度的风**——一层不够自然，三层叠加才有真实感
3. **交互 RT 不需要高分辨率**——256×256 足够，关键是 UV 映射要和地形大小对齐
4. **`Cull Off` 是必须的**——草叶两面都要渲染，但要注意渲染顺序对 Overdraw 的影响
5. **远景草地优化**：超过 30m 的草地直接用 Impostor + 纯风吹（无交互），切换 LOD 时加一个颜色过渡
6. **天气联动**：暴风天把 `_WindStrength` 从 0.3 调到 1.0，配合 `_WindSpeed` 加快——美术会非常满意

### 🎯 能力体检清单

| 检查项 | 如果答不上来... |
|--------|----------------|
| 为什么用 UV.y 而不是顶点色的 R 通道编码高度？ | → 两种方案都行，关键是理解编码灵活性和数据精度 |
| 风的正弦波为什么需要叠加噪声？ | → 自然现象建模理解不足，纯正弦太规律 |
| 交互 RT 的 Fade 值 0.95 怎么定？ | → 帧率相关，需要 `pow(0.95, deltaTime * 60)` 做帧率无关化 |
| 角色跑动时草踩弯的方向怎么算？ | → 用速度方向编码到 RT 的 GA 通道，不是简单的距离场 |
| 草叶弯曲翻转（90度+）怎么避免？ | → 总弯曲量上限 clamp，或用旋转矩阵而非位移 |
| 草地 Shader 在低端机上怎么降级？ | → 关闭交互层只保留基础风场，降低噪声采样层数 |

### 🔗 相关问题

- [开放世界百万棵草怎么渲染？](../optimization/million-grass-rendering.md)
- [角色踩水面产生涟漪](../shader/interactive-water-ripple-footstep.md)
- [URP 自定义 Renderer Feature](../rendering/urp-renderer-feature.md)
