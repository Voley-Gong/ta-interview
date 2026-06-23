---
title: "开放世界百万棵草怎么渲染？面试官说GPU Instancing不够用"
category: "optimization"
level: 4
tags: ["GPUInstancing", "ComputeShader", "草地", "视锥剔除", "LOD", "开放世界", "DrawCall"]
hint: "从GPU Instancing到ComputeShader裁剪——百万草地=ComputeShader视锥剔除+IndirectInstancing+LOD分级+ impostor远景"
related: ["optimization/drawcall-500-to-100", "rendering/gpu-driven-pipeline", "optimization/gpu-bandwidth-optimization", "pipeline/houdini-vegetation-scatter"]
---

## 参考答案

### 🎬 场景描述

面试官展示一段开放世界大草原的游戏截图（类似《原神》蒙德城外或《荒野大镖客2》的平原），然后说：

> "这个场景里有大概 100 万棵草，每棵草只有几十个三角面。现在用 GPU Instancing 渲染，Draw Call 降到 1 个了，但帧率只有 22fps——GPU 被顶点处理和 OverDraw 拖垮了。你是 TA，给我一个方案把这 100 万棵草优化到 60fps。"

这是米哈游、网易、腾讯等做开放世界项目的 **TA 高级岗必考题**。考察的是从「渲染管线 → Compute Shader → LOD 策略 → 带宽优化」的完整能力链。

### ✅ 核心要点

1. **GPU Instancing 不够用**：100 万个实例全部提交给 GPU，顶点着色器处理负担过大，且无法做逐实例剔除
2. **Compute Shader 视锥剔除**：在 GPU 上计算每棵草是否在视锥内，只渲染可见的草（通常只有 5%~15%）
3. **Indirect Draw（DrawIndexedInstancedIndirect）**：剔除结果直接在 GPU 端提交 Draw Call，不需要回读 CPU
4. **距离 LOD**：近处用 3D 模型（三角面草），中距离用 BillBoard（纸片草），远处用 Impostor 或纹理化
5. **密度控制和 Wind 动画**：用 Compute Shader 同时计算风力偏移，减少 Pixel Shader 负担

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
问题：100 万棵草 = 3000 万三角面 → GPU 顶点处理瓶颈 + OverDraw 爆炸
                    ↓
关键洞察：100 万棵草中，视锥内可见的可能只有 5~15 万棵
                    ↓
方案 1：Compute Shader 视锥剔除
  ├── CPU 端：把 100 万棵草的位置/缩放/旋转上传 GPU StructuredBuffer
  ├── GPU 端（Compute Shader）：对每个实例做视锥 AABB 测试
  ├── 输出：AppendStructuredBuffer 只存可见实例
  └── Indirect Draw：用可见数量做 DrawIndexedInstancedIndirect
                    ↓
方案 2：距离 LOD 分层
  ├── 0~15m：3D 三角面草模型（3~5 个三角形）
  ├── 15~50m：Cross Billboard（十字纸片，2 个四边形）
  ├── 50~150m：单面 Billboard + 降采样纹理
  └── 150m+：地面纹理（草地图 + 细节法线）或 Impostor
                    ↓
方案 3：OverDraw 控制
  ├── 草叶半透明 → 改用 Alpha Test（clip），不做 Alpha Blend
  ├── Pre-Z Pass 先写深度（减少被遮挡草的 Pixel Shader 执行）
  └── 地形混合：远处不用几何体草，直接在地面 Shader 里画"草纹理"
                    ↓
方案 4：Wind 动画优化
  ├── Wind 数据存顶点色（顶点色 R 通道 = 风力权重）
  ├── 风力偏移在 Vertex Shader 里算（不是 Pixel Shader）
  └── 用 Simplex Noise 纹理代替实时噪声计算
```

#### 知识点拆解（倒推树）

```
百万级草地渲染
├── GPU Instancing（基础）
│   ├── Graphics.DrawMeshInstanced / DrawMeshInstancedIndirect
│   ├── 实例数据打包（位置/缩放/旋转/颜色 → float4x4 或拆分打包）
│   ├── 实例数量上限（Unity: 1023/次调用，Indirect 无上限）
│   └── 实例常量缓冲 vs StructuredBuffer
├── Compute Shader 视锥剔除
│   ├── 视锥体平面方程（6 个平面，点-平面测试）
│   ├── AABB / Bounding Sphere 包围体测试
│   ├── AppendStructuredBuffer / ConsumeStructuredBuffer
│   ├── Compute Buffer 的 Counter 机制
│   ├── DrawIndexedInstancedIndirect 的参数填充
│   └── GPU→GPU 数据流（零 CPU 回读）
├── LOD 分层策略
│   ├── 距离分层阈值（根据屏幕占比 / 像素覆盖率）
│   ├── 三角面草 → Billboard 草过渡（dither / cross-fade）
│   ├── Cross Billboard（双面纸片）vs 单面 Billboard
│   ├── Impostor（八方向预渲染到纹理）
│   └── 远景：地面材质直接画草（细节纹理 + 法线扰动）
├── OverDraw 与填充率
│   ├── Alpha Test（clip(alpha - threshold)）vs Alpha Blend
│   ├── Early-Z / Hi-Z 优化（Alpha Test 能利用 Early-Z）
│   ├── Pre-Z Pass（多一遍顶点处理，减少像素处理）
│   └── 分辨率缩放（草地区域用 0.7x 分辨率渲染再 upsample）
├── 风力动画
│   ├── 顶点色存储风力权重 + 弯曲方向
│   ├── 全局风向参数（_WindDirection + _WindStrength）
│   ├── Simplex/Stochastic Noise 纹理采样
│   ├── 时间偏移（_Time.y * windSpeed + positionHash 去同步）
│   └── 波浪传播（基于位置的相位偏移，让草形成"麦浪"效果）
├── 地形交互
│   ├── 角色踩踏（Splat Mask 贴图，角色位置画一个压扁的圆）
│   ├── 物理弯曲（角色周围的草额外弯曲）
│   └── 弹性恢复（踩踏区域逐渐恢复，需要动态纹理或 Compute Buffer）
└── 数据管理与生成
    ├── Houdini / World Creator 生成草地散布数据
    ├── Density Map（密度图：哪里有草、哪里没有）
    ├── Height Map 控制草的高度变化（海拔越高草越短）
    └── 运行时分块加载（Chunk-based streaming）
```

#### 代码实现

**1. Compute Shader 视锥剔除（HLSL）**

```hlsl
#pragma kernel CSCullGrass

#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

// === 输入 ===
StructuredBuffer<float4> _GrassData;    // xyz=位置, w=缩放
uint _GrassCount;
float4 _FrustumPlanes[6];               // 6 个视锥平面方程 (a,b,c,d)
float3 _CameraPos;
float _CullDistance;                     // 最大渲染距离

// === 输出 ===
AppendStructuredBuffer<float4> _VisibleGrass;  // 可见草地数据
RWStructuredBuffer<uint> _IndirectArgs;         // Indirect Draw 参数

// 点到平面的距离
inline float DistanceToPlane(float4 plane, float3 point)
{
    return dot(plane.xyz, point) + plane.w;
}

// 视锥剔除：检查点是否在所有 6 个平面内侧
inline bool IsInFrustum(float3 pos, float radius)
{
    UNITY_UNROLL
    for (int i = 0; i < 6; i++)
    {
        if (DistanceToPlane(_FrustumPlanes[i], pos) < -radius)
            return false;
    }
    return true;
}

[numthreads(64, 1, 1)]
void CSCullGrass(uint3 id : SV_DispatchThreadID)
{
    if (id.x >= _GrassCount) return;

    float4 data = _GrassData[id.x];
    float3 worldPos = data.xyz;
    float scale = data.w;

    // 1. 距离剔除
    float distToCamera = distance(worldPos, _CameraPos);
    if (distToCamera > _CullDistance) return;

    // 2. 视锥剔除（草的包围球半径约 0.5 * scale）
    if (!IsInFrustum(worldPos, 0.5 * scale)) return;

    // 3. 通过剔除，追加到可见列表
    _VisibleGrass.Append(data);
}

#pragma kernel CSFillIndirectArgs

// 填充 Indirect Draw 参数
[numthreads(1, 1, 1)]
void CSFillIndirectArgs(uint3 id : SV_DispatchThreadID)
{
    // DrawIndexedInstancedIndirect 参数:
    // [0] indexCountPerInstance
    // [1] instanceCount
    // [2] startIndexLocation
    // [3] baseVertexLocation
    // [4] startInstanceLocation

    uint visibleCount = _IndirectArgs[6]; // 由 AppendBuffer 的 counter 填充
    _IndirectArgs[0] = 8;                  // 每棵草 8 个索引（4 三角面）
    _IndirectArgs[1] = visibleCount;       // 可见实例数
    _IndirectArgs[2] = 0;
    _IndirectArgs[3] = 0;
    _IndirectArgs[4] = 0;
}
```

**2. 草地渲染 Shader（带风力动画 + LOD）**

```hlsl
Shader "Custom/GrassLOD"
{
    Properties
    {
        _BaseMap ("草纹理", 2D) = "white" {}
        _BaseColor ("基础颜色", Color) = (0.3, 0.6, 0.1, 1)
        _TipColor ("尖端颜色", Color) = (0.6, 0.8, 0.2, 1)
        _WindMap ("风噪声纹理", 2D) = "white" {}
        _WindStrength ("风力强度", Range(0, 1)) = 0.3
        _WindSpeed ("风速", Range(0, 5)) = 1.0
        _AlphaCutoff ("透明阈值", Range(0, 1)) = 0.5
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" "Queue" = "Geometry" }
        Cull Off    // 双面渲染
        LOD 100

        Pass
        {
            Name "GrassForward"
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #pragma multi_compile_instancing
            #pragma multi_compile _ _LOD_NEAR _LOD_MID _LOD_FAR

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            TEXTURE2D(_BaseMap);       SAMPLER(sampler_BaseMap);
            TEXTURE2D(_WindMap);       SAMPLER(sampler_WindMap);

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float4 _TipColor;
                float  _WindStrength;
                float  _WindSpeed;
                float  _AlphaCutoff;
                float4 _BaseMap_ST;
            CBUFFER_END

            // 全局风向（C# 设置）
            float3 _WindDirection;

            // 实例数据（来自 Compute Shader 的输出）
            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float4, _InstanceData) // xyz=pos, w=scale
            UNITY_INSTANCING_BUFFER_END(Props)

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 uv         : TEXCOORD0;
                float4 color      : COLOR;       // 顶点色：R=风力权重
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
                float3 normalWS   : TEXCOORD1;
                float  ambientOcclusion : TEXCOORD2;
                float4 color      : COLOR;
            };

            // === 风力动画 ===
            float3 ApplyWind(float3 posOS, float windWeight, float3 worldPos)
            {
                // 风噪声采样（用世界坐标做 UV，让风有空间变化）
                float2 windUV = worldPos.xz * 0.05 + _Time.y * _WindSpeed * 0.1;
                float windNoise = SAMPLE_TEXTURE2D_LOD(_WindMap, sampler_WindMap, windUV, 0).r;

                // 风力偏移（越高的顶点偏移越大 → 草尖弯曲）
                float bendAmount = windWeight * posOS.y; // y 是高度
                float3 windOffset = _WindDirection * windNoise * _WindStrength * bendAmount;

                return posOS + windOffset;
            }

            Varyings Vert(Attributes input)
            {
                Varyings output = (Varyings)0;
                UNITY_SETUP_INSTANCE_ID(input);

                float4 instanceData = UNITY_ACCESS_INSTANCED_PROP(Props, _InstanceData);
                float3 instancePos = instanceData.xyz;
                float  instanceScale = instanceData.w;

                // === 构建实例矩阵（平移+缩放） ===
                float3 posOS = input.positionOS * instanceScale;

                // === 风力动画 ===
                float3 worldPos = instancePos;
                posOS = ApplyWind(posOS, input.color.r, worldPos);

                // === 转到世界空间 ===
                float3 finalWorldPos = posOS + instancePos;

                output.positionCS = TransformWorldToHClip(finalWorldPos);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                output.normalWS = TransformObjectToWorldNormal(input.normalOS);
                output.ambientOcclusion = input.color.g; // 顶点色 G = AO
                output.color = input.color;

                return output;
            }

            half4 Frag(Varyings input) : SV_Target
            {
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv);

                // Alpha Test（不用 Blend，利用 Early-Z）
                clip(albedo.a - _AlphaCutoff);

                // 根部暗、尖端亮的渐变（用 UV.y）
                half3 color = lerp(_BaseColor.rgb * albedo.rgb,
                                   _TipColor.rgb * albedo.rgb,
                                   input.uv.y);

                // AO（顶点色 G）
                color *= lerp(0.5, 1.0, input.ambientOcclusion);

                // 简单光照（主光 + 环境光）
                Light mainLight = GetMainLight();
                half3 diffuse = color * saturate(dot(input.normalWS, mainLight.direction));
                half3 ambient = color * half3(0.3, 0.35, 0.4);

                // 次表面散射近似（草叶透光效果）
                half backLight = saturate(dot(-input.normalWS, mainLight.direction)) * 0.3;
                diffuse += color * backLight * half3(0.8, 0.6, 0.3);

                return half4(diffuse + ambient, 1.0);
            }
            ENDHLSL
        }
    }
}
```

**3. C# 调度控制器**

```csharp
using UnityEngine;

public class GrassRenderer : MonoBehaviour
{
    [Header("草地数据")]
    public Mesh grassMesh;
    public Material grassMaterial;
    public int grassCount = 1000000;
    public float renderDistance = 120f;

    [Header("Compute Shader")]
    public ComputeShader cullComputeShader;

    private ComputeBuffer _grassDataBuffer;     // 所有草的数据
    private ComputeBuffer _visibleGrassBuffer;  // 可见草的数据
    private ComputeBuffer _indirectArgsBuffer;   // Indirect Draw 参数
    private ComputeBuffer _argsFillBuffer;       // 填充参数用

    private Camera _mainCamera;

    void Start()
    {
        _mainCamera = Camera.main;
        InitializeGrassData();
        InitializeBuffers();
    }

    void InitializeGrassData()
    {
        // 生成草地数据（实际项目从 Houdini / 地形 Density Map 加载）
        float4[] grassData = new float4[grassCount];
        float range = 500f; // 500x500 的区域

        for (int i = 0; i < grassCount; i++)
        {
            float x = Random.Range(-range, range);
            float z = Random.Range(-range, range);
            float y = 0f; // 地形高度（实际从 HeightMap 采样）
            float scale = Random.Range(0.7f, 1.5f);

            grassData[i] = new float4(x, y, z, scale);
        }

        _grassDataBuffer = new ComputeBuffer(grassCount, sizeof(float) * 4);
        _grassDataBuffer.SetData(grassData);
    }

    void InitializeBuffers()
    {
        // 可见草列表（上限 = 总数，但实际远小于）
        _visibleGrassBuffer = new ComputeBuffer(grassCount, sizeof(float) * 4,
            ComputeBufferType.Append);

        // Indirect Draw 参数: indexCount, instanceCount, startIndex, baseVertex, startInstance
        // + 额外 uint 用于可见计数
        _indirectArgsBuffer = new ComputeBuffer(7, sizeof(uint),
            ComputeBufferType.IndirectArguments);
    }

    void Update()
    {
        CullAndDraw();
    }

    void CullAndDraw()
    {
        // === 1. 提取视锥平面 ===
        Plane[] frustumPlanes = GeometryUtility.CalculateFrustumPlanes(_mainCamera);
        Vector4[] planes = new Vector4[6];
        for (int i = 0; i < 6; i++)
        {
            planes[i] = new Vector4(
                frustumPlanes[i].normal.x,
                frustumPlanes[i].normal.y,
                frustumPlanes[i].normal.z,
                frustumPlanes[i].distance);
        }

        // === 2. 设置 Compute Shader 参数 ===
        int kernelCull = cullComputeShader.FindKernel("CSCullGrass");
        cullComputeShader.SetBuffer(kernelCull, "_GrassData", _grassDataBuffer);
        cullComputeShader.SetBuffer(kernelCull, "_VisibleGrass", _visibleGrassBuffer);
        cullComputeShader.SetInt("_GrassCount", grassCount);
        cullComputeShader.SetVectorArray("_FrustumPlanes", planes);
        cullComputeShader.SetVector("_CameraPos", _mainCamera.transform.position);
        cullComputeShader.SetFloat("_CullDistance", renderDistance);

        // 清空 AppendBuffer
        _visibleGrassBuffer.SetCounterValue(0);

        // === 3. 调度 Compute Shader 做剔除 ===
        int threadGroups = Mathf.CeilToInt(grassCount / 64f);
        cullComputeShader.Dispatch(kernelCull, threadGroups, 1, 1);

        // === 4. 填充 Indirect Draw 参数 ===
        // 用 CopyCount 把 AppendBuffer 的元素数写入 argsBuffer[6]
        ComputeBuffer.CopyCount(_visibleGrassBuffer, _indirectArgsBuffer, 6 * sizeof(uint));

        int kernelFill = cullComputeShader.FindKernel("CSFillIndirectArgs");
        cullComputeShader.SetBuffer(kernelFill, "_IndirectArgs", _indirectArgsBuffer);
        cullComputeShader.Dispatch(kernelFill, 1, 1, 1);

        // === 5. 设置材质参数（风力等） ===
        grassMaterial.SetBuffer("_InstanceData", _visibleGrassBuffer);
        grassMaterial.SetVector("_WindDirection",
            new Vector3(Mathf.Sin(Time.time * 0.3f), 0, Mathf.Cos(Time.time * 0.3f)));

        // === 6. Indirect Draw ===
        Graphics.DrawMeshInstancedIndirect(
            grassMesh, 0, grassMaterial,
            new Bounds(Vector3.zero, new Vector3(1000, 100, 1000)),
            _indirectArgsBuffer, 0);
    }

    void OnDestroy()
    {
        _grassDataBuffer?.Release();
        _visibleGrassBuffer?.Release();
        _indirectArgsBuffer?.Release();
    }
}
```

### ⚡ 实战经验

**数字感知：百万草地的性能预算**

以 1080p / 60fps 为目标（每帧 16.67ms），草地渲染预算约 3-4ms：
| 策略 | 三角面数 | 耗时 | 适用场景 |
|------|---------|------|---------|
| 100万棵全部渲染（无剔除） | ~3000万 | 45ms+ | 不可行 |
| Compute Shader 剔除后（~10万可见） | ~80万 | 2-3ms | 近+中距离 |
| 近处3D草 + 中距离Billboard + 远景纹理 | ~10万 | 1.5-2ms | 推荐方案 |
| 全 Billboard（无3D草） | ~20万 | 0.8-1ms | 移动端 |

**最大陷阱：ComputeBuffer.CopyCount 的时机**

`ComputeBuffer.CopyCount` 是异步的——它在 GPU 执行序列中插入一条拷贝指令。如果紧接着在 CPU 端读 `_indirectArgsBuffer` 的数据，会读到上一帧的结果或者触发 GPU stall。**永远不要 CPU 回读 Indirect Draw 参数**，让数据完全在 GPU 端流转。

**移动端的 Compute Shader 限制**

- OpenGL ES 3.1+ 才支持 Compute Shader，Vulkan 更好
- `AppendStructuredBuffer` 在移动端某些驱动上有 bug（Adreno 尤其），用普通 `RWStructuredBuffer` + 原子计数更安全
- 替代方案：如果目标设备不支持 Compute Shader，CPU 端做粗粒度分块剔除（Chunk Culling），再 GPU Instancing

**OverDraw 是隐藏 Boss**

草地是典型的 OverDraw 杀手——100 万棵草互相重叠，每个像素可能被覆盖 5-20 次。优化手段：
1. **Alpha Test 而非 Alpha Blend**：Early-Z 可以跳过被遮挡的像素
2. **Pre-Z Pass**：先跑一遍简单的深度写入 Pass（只写 Z，不写颜色），然后正式渲染时做 ZTest Equal
3. **距离密度衰减**：远处草的密度自动降低（Cull Compute Shader 里加距离概率剔除）

### 🎯 能力体检清单

| 检查项 | 如果你答不上来… |
|--------|----------------|
| 知道 GPU Instancing 和 Indirect Instancing 的区别？ | → 渲染 API 层级不够 |
| 能写出 Compute Shader 做视锥剔除的伪代码？ | → GPU Programming 能力不足 |
| 理解 AppendStructuredBuffer 和 CopyCount 机制？ | → Compute Buffer 机制不熟 |
| 知道 Alpha Test 为什么能利用 Early-Z？ | → GPU 硬件渲染管线理解不足 |
| 能设计 3 级 LOD 策略（3D → Billboard → 纹理）？ | → LOD 系统设计经验不够 |
| 会写带风力动画的草地 Vertex Shader？ | → Shader 能力不足 |
| 知道移动端 Compute Shader 的限制和替代方案？ | → 跨平台优化经验不足 |
| 能解释 Pre-Z Pass 为什么能减少草地 OverDraw？ | → 深度优化原理不清 |

### 🔗 相关问题

- [Draw Call 从 500 降到 100：合批与渲染优化全景](optimization/drawcall-500-to-100)
- [GPU-Driven Pipeline：消灭 CPU 瓶颈的终极方案](rendering/gpu-driven-pipeline)
- [GPU 带宽优化：移动端渲染的带宽管理](optimization/gpu-bandwidth-optimization)
- [Houdini 批量植被散布：从 Density Map 到引擎数据](pipeline/houdini-vegetation-scatter)
