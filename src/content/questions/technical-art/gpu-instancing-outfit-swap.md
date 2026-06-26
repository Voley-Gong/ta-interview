---
title: "面试官给你100个NPC同屏：GPU Instancing换装怎么做到不爆骨骼和材质？"
category: technical-art
level: 4
tags: ["GPU Instancing", "骨骼动画", "换装", "Draw Call", "SRP Batcher"]
hint: "动画纹理烘焙 + 顶点ID索引 + 材质合批，三件套缺一不可"
related: ["character-outfit-swap-skeleton-sharing", "skinned-mesh-vertex-animation-cost", "drawcall-500-to-100"]
---

## 参考答案

### 🎬 场景描述

面试官给你看一个开放世界游戏的截图——一个集市广场上有上百个NPC，各有不同的服装、动作。然后说：

> "这个场景有120个NPC同屏，每个NPC有3个换装部位（头、身体、腿），每个部位有5-10种替换件。
> 现在的性能数据：
> - Draw Call: 360（120 × 3部位）
> - 每帧CPU骨骼计算: 18ms（瓶颈）
> - GPU上每NPC各自一套材质实例
> 
> 面试问题：你要怎么把Draw Call降到5以内、CPU骨骼计算降到2ms以内？
> 
> 限制：不能用Impostor（视角太近看得出），不能用LOD隐藏（所有NPC都在视野内）。"

### ✅ 核心要点

1. **动画纹理烘焙**：将骨骼动画烘焙到纹理中，GPU采样获取变换矩阵，消除CPU骨骼计算
2. **网格合并**：同部位的换装件合并成一个大Mesh，用顶点属性区分
3. **GPU Instancing + Material Property Block**：一个Draw Call渲染所有实例，通过实例属性传差异化数据
4. **材质合批策略**：所有换装件共享一个Shader，通过纹理数组实现"一个材质，N种外观"

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终目标：120 NPC × 3部位 = 360 Draw Call → ≤5 Draw Call
                    ↓ 倒推
Draw Call要合并 → 同材质同Mesh才能GPU Instance
                    ↓ 倒推
问题：每个NPC换装不同 → 材质不同 → 无法Instance
                    ↓ 倒推
解法：把"换装差异"从材质层面移到顶点/实例数据层面
  - 纹理数组：一套材质，切换 _SliceIndex 选择贴图
  - 顶点色/UV2：标记部位ID
                    ↓ 倒推
CPU骨骼18ms问题 → 把骨骼计算移到GPU
                    ↓ 倒推
动画烘焙到纹理：Animation Texture = 每帧每骨骼的4×4矩阵
                    ↓ 倒推
必须掌握：GPU Instancing约束、动画纹理方案、纹理数组、Mesh合并
```

#### 知识点拆解（倒推树）

```
GPU Instancing 换装系统
├── 动画纹理烘焙 (Animation Texture Baking)
│   ├── 原理：每帧每根骨骼存为纹素（texel），RGBA = 矩阵一行
│   ├── 纹理格式：Float RGBA 或 Half RGBA（移动端）
│   ├── 采样：根据 _Time 和 _AnimFrameOffset 计算UV
│   └── 支持动画混合/过渡？（双采样 + 插值）
├── GPU Instancing 约束
│   ├── 必须同一Mesh + 同一Material
│   ├── 不能有 per-instance 纹理（只能传 scalar/vector）
│   ├── MaterialPropertyBlock 的限制
│   └── SRP Batcher vs GPU Instancing 互斥问题
├── 换装实现
│   ├── 纹理数组 (Texture2DArray)：一个材质 × N种贴图切片
│   ├── 每实例传 _SliceIndex（不同NPC选不同贴图）
│   ├── Mesh合并：同部位的10种头 → 1个合并Mesh × 顶点偏移
│   └── 遮罩：用顶点色决定显示哪个部位
├── 骨骼与变换
│   ├── 骨骼索引：每个顶点存 boneIndex → 查动画纹理
│   ├── 蒙皮在GPU完成（Vertex Shader内）
│   └── 无需CPU SkinnedMeshRenderer
└── 性能验证
    ├── Draw Call 实际数量验证
    ├── GPU 带宽成本（纹理采样数量）
    └── 内存成本（动画纹理大小）
```

#### 代码实现

**Step 1：动画烘焙到纹理**

```csharp
// AnimationTextureBaker.cs — 将AnimationClip烘焙成纹理
using UnityEngine;

public class AnimationTextureBaker
{
    public static Texture2D Bake(AnimationClip clip, SkinnedMeshRenderer smr, int fps = 30)
    {
        int boneCount = smr.bones.Length;
        int frameCount = Mathf.CeilToInt(clip.length * fps);

        // 每根骨骼需要3个纹素（存3x4矩阵的3行，第4行用不到）
        int texWidth = boneCount * 3;
        int texHeight = frameCount;

        Texture2D tex = new Texture2D(texWidth, texHeight, TextureFormat.RGBAHalf, false);
        tex.filterMode = FilterMode.Point; // 精确采样

        GameObject tempObj = Object.Instantiate(smr.transform.root).gameObject;
        Animator animator = tempObj.GetComponent<Animator>();
        SkinnedMeshRenderer tempSmr = tempObj.GetComponentInChildren<SkinnedMeshRenderer>();

        for (int f = 0; f < frameCount; f++)
        {
            float time = (float)f / fps;
            clip.SampleAnimation(tempObj, time);

            for (int b = 0; b < boneCount; b++)
            {
                Matrix4x4 m = tempSmr.bones[b].localToWorldMatrix;
                // 写入3行 × 4通道
                tex.SetPixel(b * 3 + 0, f, new Color(m.m00, m.m01, m.m02, m.m03));
                tex.SetPixel(b * 3 + 1, f, new Color(m.m10, m.m11, m.m12, m.m13));
                tex.SetPixel(b * 3 + 2, f, new Color(m.m20, m.m21, m.m22, m.m23));
            }
        }

        tex.Apply();
        Object.DestroyImmediate(tempObj);
        return tex;
    }
}
```

**Step 2：GPU Instancing 渲染（Vertex Shader采样动画纹理）**

```hlsl
// InstancedCharacter.shader
Shader "Custom/InstancedCharacter"
{
    Properties
    {
        _DiffuseArray ("Diffuse Texture Array", 2DArray) = "" {}
        _AnimTex ("Animation Texture", 2D) = "white" {}
        _BoneCount ("Bone Count", Float) = 30
        _AnimFPS ("Animation FPS", Float) = 30
        _AnimFrameCount ("Total Frames", Float) = 60
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            #pragma multi_compile_instancing
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D_ARRAY(_DiffuseArray);
            SAMPLER(sampler_DiffuseArray);

            TEXTURE2D(_AnimTex);
            SAMPLER(sampler_AnimTex);
            float _BoneCount, _AnimFPS, _AnimFrameCount;

            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float, _SliceIndex)    // 每NPC换装贴图索引
                UNITY_DEFINE_INSTANCED_PROP(float, _AnimOffset)    // 每NPC动画时间偏移（错峰）
                UNITY_DEFINE_INSTANCED_PROP(float, _AnimSpeed)     // 每NPC动画速度
            UNITY_INSTANCING_BUFFER_END(Props)

            struct Attributes
            {
                float4 pos : POSITION;
                float2 uv : TEXCOORD0;
                float4 boneWeight : BLENDWEIGHT;
                int4 boneIndex : BLENDINDICES;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            // 从动画纹理采样某根骨骼在某帧的矩阵
            float4x4 SampleBoneMatrix(int boneIdx, float frame)
            {
                float u0 = (boneIdx * 3 + 0 + 0.5) / (_BoneCount * 3);
                float u1 = (boneIdx * 3 + 1 + 0.5) / (_BoneCount * 3);
                float u2 = (boneIdx * 3 + 2 + 0.5) / (_BoneCount * 3);
                float v = (frame + 0.5) / _AnimFrameCount;

                float4 row0 = SAMPLE_TEXTURE2D_LOD(_AnimTex, sampler_AnimTex, float2(u0, v), 0);
                float4 row1 = SAMPLE_TEXTURE2D_LOD(_AnimTex, sampler_AnimTex, float2(u1, v), 0);
                float4 row2 = SAMPLE_TEXTURE2D_LOD(_AnimTex, sampler_AnimTex, float2(u2, v), 0);

                return float4x4(row0.x, row0.y, row0.z, row0.w,
                                row1.x, row1.y, row1.z, row1.w,
                                row2.x, row2.y, row2.z, row2.w,
                                0, 0, 0, 1);
            }

            Varyings Vert(Attributes input)
            {
                UNITY_SETUP_INSTANCE_ID(input);
                Varyings output;
                UNITY_TRANSFER_INSTANCE_ID(input, output);

                // 计算当前帧
                float speed = UNITY_ACCESS_INSTANCED_PROP(Props, _AnimSpeed);
                float offset = UNITY_ACCESS_INSTANCED_PROP(Props, _AnimOffset);
                float time = _Time.y * speed + offset;
                float frame = fmod(time * _AnimFPS, _AnimFrameCount);

                // 4骨骼蒙皮
                float4x4 skinMat = 0;
                skinMat += input.boneWeight.x * SampleBoneMatrix(input.boneIndex.x, frame);
                skinMat += input.boneWeight.y * SampleBoneMatrix(input.boneIndex.y, frame);
                skinMat += input.boneWeight.z * SampleBoneMatrix(input.boneIndex.z, frame);
                skinMat += input.boneWeight.w * SampleBoneMatrix(input.boneIndex.w, frame);

                float3 worldPos = mul(skinMat, float4(input.pos.xyz, 1.0)).xyz;
                output.pos = TransformWorldToHClip(worldPos);
                output.uv = input.uv;
                return output;
            }

            half4 Frag(Varyings input) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                float slice = UNITY_ACCESS_INSTANCED_PROP(Props, _SliceIndex);
                half4 color = SAMPLE_TEXTURE2D_ARRAY(_DiffuseArray, sampler_DiffuseArray, input.uv, slice);
                return color;
            }
            ENDHLSL
        }
    }
}
```

**Step 3：C# 批量渲染**

```csharp
// InstancedNPCRenderer.cs
using UnityEngine;

public class InstancedNPCRenderer : MonoBehaviour
{
    public Mesh combinedMesh;       // 合并后的NPC网格
    public Material instancedMat;
    public Texture2DArray diffuseArray;  // 所有换装贴图打包
    public int npcCount = 120;

    private MaterialPropertyBlock _mpb;
    private Matrix4x4[] _matrices;
    private Vector4[] _sliceIndices;
    private Vector4[] _animOffsets;
    private Vector4[] _animSpeeds;

    void Start()
    {
        _mpb = new MaterialPropertyBlock();
        _matrices = new Matrix4x4[npcCount];
        _sliceIndices = new Vector4[npcCount];
        _animOffsets = new Vector4[npcCount];
        _animSpeeds = new Vector4[npcCount];

        for (int i = 0; i < npcCount; i++)
        {
            _matrices[i] = Matrix4x4.TRS(
                new Vector3(Random.Range(-20, 20), 0, Random.Range(-20, 20)),
                Quaternion.Euler(0, Random.Range(0, 360), 0),
                Vector3.one
            );
            _sliceIndices[i] = new Vector4(i % diffuseArray.depth, 0, 0, 0);
            _animOffsets[i] = new Vector4(Random.Range(0, 2f), 0, 0, 0);
            _animSpeeds[i] = new Vector4(Random.Range(0.9f, 1.1f), 0, 0, 0);
        }
    }

    void Update()
    {
        // 矩阵和参数必须用 Graphics.RenderMeshIndirect 或逐批 Render
        // 这里用简单分批方式（每批511个实例上限）
        int batchSize = 511;
        for (int batchStart = 0; batchStart < npcCount; batchStart += batchSize)
        {
            int count = Mathf.Min(batchSize, npcCount - batchStart);

            // 注意：MaterialPropertyBlock 只能传单值不能传数组
            // 实际项目中需要用 GraphicsBuffer 或 ComputeBuffer 传 per-instance 数据
            Graphics.DrawMeshInstanced(combinedMesh, 0, instancedMat, _matrices, count, _mpb);
        }
    }
}
```

> ⚠️ **重要注意**：Unity 的 `MaterialPropertyBlock` 对于 GPU Instancing per-instance 数据有严格限制（只能传最多 5 个 Vector4）。如果需要传更多数据，需要使用 `GraphicsBuffer`（Unity 2022+）或 ComputeBuffer + StructuredBuffer。

### ⚡ 实战经验

1. **SRP Batcher 与 GPU Instancing 互斥**：同一个Shader不能同时享受两者。对于大量相同Mesh的场景，GPU Instancing 更优
2. **动画纹理大小**：30根骨骼 × 3行 × 60帧 = 90 × 60 纹素，Half格式约 21KB/动画。一个NPC 5个动画约100KB，120个NPC共享同一套动画纹理
3. **动画过渡怎么做**：在Vertex Shader里双采样（frame₀ 和 frame₁）然后lerp，模拟动画混合
4. **纹理数组是关键**：没有 Texture2DArray，就要用图集，UV边界处理会变复杂
5. **实际项目的 Draw Call**：3个部位 × 1个Draw Call = 3（合并Mesh后甚至可以做到1个），比原来360降低99%

### 🎯 能力体检清单

| 如果答不上来... | 说明盲区在 |
|---|---|
| 不知道GPU Instancing的前提条件 | GPU Instancing 基本原理 |
| 知道Instancing但不知道怎么让每个NPC外观不同 | per-instance 数据传递机制 |
| 没想到把骨骼动画烘焙到纹理 | GPU Skinning / 动画纹理方案 |
| 知道烘焙但不会写采样Shader | 纹理采样 + 矩阵重建 |
| 没考虑MaterialPropertyBlock的数据上限 | Unity GPU Instancing API 细节 |

### 🔗 相关问题

- [角色换装系统：骨骼共享与材质合批怎么权衡？](character-outfit-swap-skeleton-sharing)
- [蒙皮网格的顶点动画成本：为什么CPU骨骼是性能杀手？](skinned-mesh-vertex-animation-cost)
- [手游 Draw Call 从500降到100：TA的性能优化方法论](drawcall-500-to-100)
