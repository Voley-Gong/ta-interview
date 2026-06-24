---
title: "角色纹身贴花：如何用投影Decal Shader实现可旋转的皮肤彩绘？"
category: "shader"
level: 3
tags: ["Shader", "贴花", "投影", "角色定制", "URP", "Decal"]
hint: "核心是 Decal 投影到曲面 + 基于 Tangent 空间的 UV 映射 + 边缘柔化处理，还要处理贴花随骨骼运动的形变"
related: ["shader/dissolve-effect", "rendering/planar-projection-shadow", "technical-art/character-material-spec-workflow"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们在做一个角色定制系统，玩家可以选择不同的纹身/彩绘贴纸贴到角色身体上。要求：
> 1. 纹身可以在身体任意位置放置
> 2. 支持旋转、缩放、移动调节
> 3. 贴在关节弯曲处时不能严重拉伸
> 4. 纹身要能随角色动画一起运动
> 5. 移动端 30fps，同屏 5 个有贴花的角色
>
> 你怎么实现？

这是叠纸（无限暖暖）、腾讯（和平精英/王者定制系统）、网易（永劫无间捏脸）等项目的经典题。考察的是**贴花投影系统 + 曲面 UV 映射 + 角色定制管线**的综合能力。

### ✅ 核心要点

1. **Decal 投影到曲面**：从放置点发射正交/透视投影，将贴花投射到角色网格表面
2. **Tangent 空间 UV**：基于角色的切线/副切线空间计算贴花 UV，避免严重拉伸
3. **边缘柔化**：用距离衰减或 Alpha 蒙版做贴花边缘羽化，避免硬边
4. **多贴花叠加**：Blended 贴花支持多个贴花叠放（纹身+疤痕+油彩）
5. **骨骼跟随**：贴花附着在蒙皮网格上，随骨骼动画自然变形

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
期望效果：任意位置放置纹身 → 随曲面变形 → 可旋转缩放 → 多层叠加 → 随动画运动
         ↓
方案选择：
  方案A：修改角色纹理图（运行时 Texture2D.SetPixels）
    → ❌ 不能旋转/缩放、内存翻倍、不支持多层
  方案B：平面投影贴花（Planar Decal）
    → ✅ 可调位置/旋转/缩放，但需要处理曲面拉伸
  方案C：UV 空间贴花（修改 UV2 通道）
    → 精确但需要预处理，不支持运行时自由放置
  方案D：Decal Projector 组件（URP 内置）
    → ✅ 推荐方案：引擎原生支持，性能可控

最终选择：方案D（URP Decal Projector）+ 方案B 的自定义扩展
```

#### 知识点拆解（倒推树）

```
角色贴花系统
├── Decal 投影原理
│   ├── 需要理解：Decal 是什么？
│   │   └── 从一个 Volume（Box/Sphere）内向网格投影纹理
│   ├── 需要理解：投影矩阵的构建
│   │   └── 正交投影矩阵 × 视图矩阵 = 贴花 MVP
│   └── URP Decal Projector 组件
│       ├── 已处理深度测试（不会穿过墙壁）
│       └── 支持 Normal/AO 贴花影响
│
├── 曲面 UV 映射（防拉伸）
│   ├── 需要理解：为什么直接投影会拉伸？
│   │   └── 投影方向与曲面法线夹角大时，纹理被拉长
│   ├── 方案A：Triplanar 映射（三方向投影取最优）
│   │   └── 慢但质量高，适合静态贴花
│   ├── 方案B：法线权重混合（按法线方向选择投影轴）
│   │   └── 性能较好，移动端可接受
│   └── 方案C：预计算 Tangent 空间（最佳质量）
│       └── 离线为每个网格生成贴花模板 UV
│
├── 边缘柔化
│   ├── 需要：距离 Volume 边界的 Alpha 衰减
│   │   └── 1 - smoothstep(0.8, 1.0, distFromCenter)
│   ├── 需要：贴花纹理本身的 Alpha 羽化
│   └── 需要：法线夹角剔除（背墙面不投影）
│
├── 旋转/缩放/移动
│   ├── 需要：Decal Projector 的 Transform 控制位置/旋转
│   ├── 需要：Scale 控制贴花大小
│   └── 需要：UV 旋转矩阵在 Shader 中处理纹理旋转
│
├── 多贴花叠加
│   ├── 需要：多个 Decal Projector（简单但 Draw Call 增加）
│   ├── 需要：Blended Decal Material（一张 Atlas 图集多个贴花）
│   │   └── 用 UV offset 在图集中切换贴花
│   └── 性能优化：贴花 Draw Call 合批
│
└── 骨骼动画跟随
    ├── Decal Projector 是世界空间的，角色运动会"脱节"
    ├── 解决：将 Projector 挂在骨骼节点上作为子物体
    └── 注意：蒙皮变形区域贴花仍会有轻微滞后
```

#### 代码实现

**URP Decal Shader（支持旋转+边缘柔化）：**

```hlsl
// BodyDecal.shader — URP Decal 专用
Shader "TA/BodyPaintDecal" {
    Properties {
        _DecalTex ("Decal Texture", 2D) = "white" {}
        _DecalColor ("Decal Color Tint", Color) = (1,1,1,1)
        
        // 旋转控制
        _UVRotation ("UV Rotation (degrees)", Float) = 0
        
        // 边缘柔化
        _EdgeSoftness ("Edge Softness", Range(0,1)) = 0.2
        _NormalAngleFalloff ("Normal Angle Falloff", Range(0,1)) = 0.5
        
        // 混合模式
        [Enum(Multiply, 0, Add, 1, Lerp, 2, Screen, 3)] 
        _BlendMode ("Blend Mode", Float) = 2
        
        // 多贴花层 ID
        _DecalLayer ("Decal Layer Mask", Float) = 1
    }
    
    SubShader {
        Tags { 
            "RenderPipeline"="UniversalPipeline" 
            "RenderType"="MergedDecal"
        }
        
        Pass {
            Name "BodyDecal"
            Tags { "LightMode"="Decal" }
            
            Blend SrcAlpha OneMinusSrcAlpha
            ZWrite Off
            ZTest Off  // Decal Pass 不做深度测试（投影器已处理）
            Cull Front // 渲染 Decal Volume 内表面
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Decal.hlsl"
            
            TEXTURE2D(_DecalTex); SAMPLER(sampler_DecalTex);
            
            CBUFFER_START(UnityPerMaterial)
                float4 _DecalTex_ST;
                float4 _DecalColor;
                float  _UVRotation;
                float  _EdgeSoftness;
                float  _NormalAngleFalloff;
                float  _BlendMode;
                float  _DecalLayer;
            CBUFFER_END
            
            struct Varyings {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;  // Decal 投影 UV
                float3 normalWS   : TEXCOORD1;
                float4 screenPos  : TEXCOORD2;
            };
            
            Varyings vert(Attributes input) {
                Varyings output;
                output.positionCS = TransformWorldToHClip(input.positionWS);
                output.uv = input.uv;
                output.normalWS = input.normalWS;
                output.screenPos = ComputeScreenPos(output.positionCS);
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target {
                // === UV 旋转 ===
                float rad = _UVRotation * 0.0174533; // deg2rad
                float cosR = cos(rad);
                float sinR = sin(rad);
                float2x2 rotMatrix = float2x2(cosR, -sinR, sinR, cosR);
                
                // 以 0.5 为中心旋转
                float2 uv = (input.uv - 0.5) ;
                uv = mul(rotMatrix, uv);
                uv += 0.5;
                
                // 超出 [0,1] 范围直接丢弃
                clip(0.999 - uv.x);
                clip(0.999 - uv.y);
                clip(uv.x - 0.001);
                clip(uv.y - 0.001);
                
                // === 采样贴花纹理 ===
                half4 decalCol = SAMPLE_TEXTURE2D(_DecalTex, sampler_DecalTex, uv);
                decalCol.rgb *= _DecalColor.rgb;
                
                // === 边缘柔化 ===
                // 距离中心的距离衰减
                float2 distFromCenter = abs(input.uv - 0.5) * 2.0; // 0~1
                float maxDist = max(distFromCenter.x, distFromCenter.y);
                float edgeAlpha = 1.0 - smoothstep(1.0 - _EdgeSoftness, 1.0, maxDist);
                
                // 法线角度衰减（避免侧面投影太强）
                float3 viewDir = normalize(GetWorldSpaceViewDir(input.positionCS));
                float NdotV = saturate(dot(input.normalWS, viewDir));
                float normalFactor = smoothstep(
                    1.0 - _NormalAngleFalloff, 
                    1.0, 
                    NdotV
                );
                
                // 最终 Alpha
                float finalAlpha = decalCol.a * edgeAlpha * normalFactor;
                
                // === 混合模式 ===
                half3 finalColor = decalCol.rgb;
                if (_BlendMode == 0) {
                    // Multiply
                    finalColor = finalColor * (1.0 - finalAlpha * 0.5);
                } else if (_BlendMode == 1) {
                    // Add
                    finalColor = finalColor * finalAlpha;
                } else if (_BlendMode == 2) {
                    // Lerp (default)
                    // Alpha blend handled by Blend State
                } else {
                    // Screen
                    finalColor = 1.0 - (1.0 - finalColor) * (1.0 - decalCol.rgb * finalAlpha);
                }
                
                return half4(finalColor, finalAlpha);
            }
            ENDHLSL
        }
    }
}
```

**C# 贴花放置与管理工具：**

```csharp
using UnityEngine;
using UnityEngine.Rendering.Universal;
using System.Collections.Generic;

[ExecuteAlways]
public class BodyPaintDecalSystem : MonoBehaviour {
    [Header("Target Character")]
    public SkinnedMeshRenderer targetMesh;
    public Transform boneRoot;
    
    [Header("Decal Prefab")]
    public GameObject decalProjectorPrefab;
    
    [System.Serializable]
    public class DecalInstance {
        public string id;
        public DecalProjector projector;
        public Transform attachBone;
        public Vector3 localOffset;
        public float rotation;
        public float scale = 1f;
        public int layerMask;
    }
    
    public List<DecalInstance> activeDecals = new();
    
    /// <summary>
    /// 在角色表面放置贴花（射线检测位置 + 法线方向）
    /// </summary>
    public DecalInstance PlaceDecal(Texture2D decalTex, Vector3 worldPos, Vector3 normal) {
        // 1. 射线检测确定最近的骨骼
        Transform attachBone = FindClosestBone(worldPos);
        
        // 2. 实例化 Decal Projector
        GameObject decalObj = Instantiate(decalProjectorPrefab, attachBone);
        DecalProjector projector = decalObj.GetComponent<DecalProjector>();
        
        // 3. 设置贴花纹理
        Material decalMat = new Material(projector.material);
        decalMat.SetTexture("_DecalTex", decalTex);
        projector.material = decalMat;
        
        // 4. 定位：沿法线方向偏移，面向法线方向
        Vector3 localPos = attachBone.InverseTransformPoint(worldPos + normal * 0.01f);
        decalObj.transform.localPosition = localPos;
        decalObj.transform.rotation = Quaternion.LookRotation(-normal, Vector3.up);
        
        // 5. 注册
        var instance = new DecalInstance {
            id = System.Guid.NewGuid().ToString("N"),
            projector = projector,
            attachBone = attachBone,
            localOffset = localPos,
            rotation = 0f,
            scale = 1f
        };
        activeDecals.Add(instance);
        
        return instance;
    }
    
    /// <summary>
    /// 调整贴花的旋转和缩放
    /// </summary>
    public void AdjustDecal(string id, float newRotation, float newScale) {
        var decal = activeDecals.Find(d => d.id == id);
        if (decal == null) return;
        
        decal.rotation = newRotation;
        decal.scale = newScale;
        
        decal.projector.transform.localRotation = 
            Quaternion.Euler(90, 0, newRotation);
        decal.projector.size = new Vector3(
            decal.projector.size.x * newScale,
            decal.projector.size.y * newScale,
            decal.projector.size.z
        );
        
        decal.projector.material.SetFloat("_UVRotation", newRotation);
    }
    
    /// <summary>
    /// 在角色骨骼树中找到距离目标点最近的骨骼
    /// </summary>
    Transform FindClosestBone(Vector3 worldPos) {
        Transform closest = boneRoot;
        float minDist = float.MaxValue;
        
        foreach (Transform bone in boneRoot.GetComponentsInChildren<Transform>()) {
            float dist = Vector3.Distance(bone.position, worldPos);
            if (dist < minDist) {
                minDist = dist;
                closest = bone;
            }
        }
        return closest;
    }
}
```

**性能预算与分级（移动端）：**

| 贴花数量 | Draw Call 影响 | 内存影响 | 策略 |
|----------|---------------|----------|------|
| 1-3 个 | +1~3 DC | 2-4MB（Atlas） | ✅ 全画质支持 |
| 4-8 个 | +4~8 DC | 4-8MB | ⚠️ 仅高画质 |
| 8+ 个 | 需合批 | 需要 Atlas | ❌ 用 RenderTexture 烘焙 |
| 烘焙到纹理 | +0 DC（合并到角色贴图） | +4-8MB RT | ✅ 终极方案 |

### ⚡ 实战经验

1. **RenderTexture 烘焙是终极方案**：如果贴花位置在游戏运行中不需要实时改变（比如选好纹身后固定），把贴花烘焙到角色的扩展贴图通道（如 ORM 的 R 通道或一张单独的 Mask 图），运行时完全零 Draw Call 开销
2. **关节弯曲处拉伸问题**：Decal 投影到弯曲的肘部时拉伸不可避免。解决方案：在弯曲区域缩小贴花范围 + 用法线角度衰减让弯曲处贴花自然淡出
3. **贴花图集化**：把 10 个纹身放到一张 1024² 的 Atlas 上，用 UV Offset 切换，只需要 1 个材质 + 1 个 Draw Call
4. **URP Decal 的性能陷阱**：Decal Projector 会触发额外的深度 Prepass。5+ 个贴花时，Prepass 开销显著增加。解决方案：减少 Decal Draw Distance，或降级为简单的 Mesh 贴片

### 🎯 能力体检清单

- [ ] **如果不懂 Decal 投影原理** → 你需要补：投影矩阵数学、Volume 裁剪、深度重建
- [ ] **如果不会处理曲面拉伸** → 你需要补：Tangent 空间、Triplanar 映射、法线权重混合
- [ ] **如果不懂骨骼跟随** → 你需要补：Unity 蒙皮系统、Bone Hierarchy、世界/局部空间转换
- [ ] **如果不会优化贴花性能** → 你需要补：Draw Call 合批策略、RenderTexture 烘焙、Decal Atlas
- [ ] **如果不了解 URP Decal 系统** → 你需要补：Decal Projector 组件、Decal Shader 结构、Decal Layer Mask

### 🔗 相关问题

- URP Decal Projector 和传统的 Mesh 贴片贴花各有什么优缺点？
- 如何实现"可擦除"的贴花系统（如涂鸦游戏）？需要用到 Compute Shader 吗？
- 贴花在 VR/第一人称近视角下的特殊问题（双目投影冲突）
