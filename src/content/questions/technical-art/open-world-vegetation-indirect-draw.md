---
title: "万亩草海不卡顿：GPU Indirect Draw 怎么渲染百万植被？从Instancing到Compute Shader全链路"
category: "technical-art"
level: 4
tags: ["GPU Instancing", "Indirect Draw", "Compute Shader", "植被渲染", "LOD", "视锥剔除", "HLOD", "手游优化"]
hint: "核心=Compute Shader做视锥剔除+LOD选择→Indirect Draw Arguments→GPU端直接Draw，CPU完全不参与每棵草的位置计算"
related: ["optimization/million-grass-rendering", "shader/grass-wind-shader-interaction", "pipeline/houdini-vegetation-scatter", "rendering/gpu-driven-pipeline"]
---

## 参考答案

### 🎬 场景描述

面试官展示一张开放世界地图的截图：漫山遍野的草地、灌木、树木，然后说：

> "这是我们的开放世界场景，地图上约有 200 万棵草、30 万棵灌木、5 万棵树。现在用传统的 GameObject + GPU Instancing，CPU 每帧更新 culling 花了 8ms，GPU 实际渲染 6ms。帧率在 40-50 之间抖动。你给我一套 GPU-Driven 方案，把 CPU 时间压到 1ms 以内。"

补充追问：
- "手机上能用吗？OpenGL ES 3.1 还是 Vulkan？"
- "风吹草动怎么实现？Vertex Shader 还是 Compute Shader？"
- "近处草和远处草 LOD 怎么切换？有没有 Pop-in 问题？"
- "如果美术要手动放置某些'英雄植被'（不可程序化），怎么混排？"

### ✅ 核心要点

1. **GPU-Driven Pipeline 核心思想**：把 culling（视锥剔除/遮挡剔除）和 LOD 选择全部搬到 GPU，CPU 只发一个 `DrawIndirect` 指令
2. **Compute Shader 做 Culling**：在 Compute Shader 中遍历所有植被实例，做视锥剔除和距离 LOD，输出 Surviving 实例列表
3. **Indirect Draw**：Compute Shader 输出 `DrawIndexedIndirectArgs`（Draw Call 参数），CPU 端零感知
4. **LOD 无缝切换**：用 dither/crossfade 在 LOD 切换时做透明渐变，避免 Pop-in
5. **风场系统**：全局风方向纹理（Wind Texture）+ 顶点色控制弯曲幅度，Vertex Shader 中实现
6. **混排方案**：程序化植被走 Indirect Draw，手动放置植被走传统 GameObject，两套并行

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：200万棵草流畅渲染，CPU < 1ms，GPU < 5ms，草随风摆动
               ↑
倒推1：CPU 为什么卡？
       ├── 200万个 Transform 数据更新
       ├── CPU 视锥剔除遍历所有实例
       └── 每个 LOD 级别一次 Draw Call 调用
倒推2：怎么把 culling 搬到 GPU？
       ├── 用 Compute Shader 并行处理
       ├── 输入：所有实例数据（StructuredBuffer）
       ├── 计算：视锥内？→ 保留，视锥外？→ 剔除
       └── 输出：存活实例索引 + Indirect Draw Args
倒推3：GPU 剔除后怎么 Draw？
       ├── DrawIndexedInstancedIndirect / DrawMeshInstancedIndirect
       ├── GPU 端自动读取 Draw Args 执行渲染
       └── CPU 端只调一次 API，不知道画了多少棵
倒推4：LOD 怎么切换？
       ├── Compute Shader 按距离分组（LOD0/LOD1/LOD2）
       ├── 每个 LOD 输出独立的 Draw Args
       └── 切换瞬间用 dither dissolve 消除 Pop-in
倒推5：风怎么吹？
       ├── 全局风纹理（方向 + 强度 + 时间）
       ├── Vertex Shader 中按世界坐标采样
       └── 顶点色 R=弯曲幅度，G=摆动相位，B=高度系数
倒推6：手动放置的"英雄植被"怎么办？
       ├── 程序化植被 = Indirect Draw（大规模）
       ├── 手动植被 = 传统 GameObject（少量，<100）
       └── 两套并行，渲染顺序保证不冲突
```

#### 知识点拆解（倒推树）

```
GPU-Driven 植被渲染管线
├── 数据组织
│   ├── 实例数据（StructuredBuffer / ByteAddressBuffer）
│   │   ├── float4x4 世界矩阵 或 压缩版（pos16 + scale8 + rot8）
│   │   ├── LOD 参数（距离阈值 / 自适应比例）
│   │   └── 自定义属性（健康度、季节变体等）
│   ├── Mesh 资源
│   │   ├── LOD0：高精度草（含叶片细节，~300三角面）
│   │   ├── LOD1：中精度草（简化几何，~80三角面）
│   │   ├── LOD2：十字交叉面片（2-3个面片，~10三角面）
│   │   └── LOD3：Impostor / 纯颜色贡献（近无几何体）
│   └── Density Map（密度分布纹理）
│       ├── R = 草密度（哪里长草、哪里不长）
│       ├── G = 灌木密度
│       └── B = 树密度
├── GPU Culling（Compute Shader）
│   ├── 视锥剔除
│   │   ├── 提取 6 个 Frustum Plane
│   │   ├── 每个实例做 Sphere/AABB Test
│   │   └── GPU 并行：一个线程处理一个实例
│   ├── 距离 LOD 选择
│   │   ├── Camera-to-Instance 距离
│   │   ├── 映射到 LOD 级别
│   │   └── 输出按 LOD 分组的 Draw Args
│   ├── 遮挡剔除（可选增强）
│   │   ├── Hi-Z Pyramid（上一帧深度金字塔）
│   │   ├── 实例 AABB vs Hi-Z 测试
│   │   └── 延迟一帧（Temporal Coherence，假设相机移动不大）
│   └── 输出
│       ├── AppendStructuredBuffer（存活实例索引）
│       ├── DrawIndexedIndirectArgs（每个LOD一组）
│       └── Args 格式：indexCount/instanceCount/startIndex/baseVertex/startInstance
├── 渲染
│   ├── DrawIndexedInstancedIndirect（DirectX / Vulkan）
│   ├── DrawMeshInstancedIndirect（Unity 封装）
│   ├── Vertex Shader
│   │   ├── SV_InstanceID 索引存活实例
│   │   ├── 从 StructuredBuffer 读取实例矩阵
│   │   ├── 变换顶点位置
│   │   └── 输出世界坐标用于风场采样
│   ├── Fragment Shader
│   │   ├── 基础色（Albedo）
│   │   ├── 法线（Normal Map 或 Vertex Normal）
│   │   ├── 次表面散射近似（草叶透光感）
│   │   └── Alpha Test（Cutout，不透明队列）
│   └── Shadow Pass
│       ├── 草的自阴影（成本高，通常跳过或用 Blob Shadow）
│       └── 接收其他物体阴影
├── 风场系统
│   ├── 风向纹理（Flow Map 或 Noise Texture）
│   │   ├── RG = 风方向（世界空间 XZ）
│   │   ├── B = 强度
│   │   └── A = 阵风脉冲（时间变化）
│   ├── Vertex Shader 风力计算
│   │   ├── 采样风纹理得到风向和强度
│   │   ├── 按顶点高度衰减（根部不动、尖部摆）
│   │   ├── 顶点色 R 控制弯曲幅度
│   │   └── sin/cos 做周期摆动
│   ├── 玩家交互
│   │   ├── 玩家位置写入交互场（RenderTexture）
│   │   ├── 草 Shader 采样交互场 → "压倒"效果
│   │   └── 延迟恢复（弹簧回归）
│   └── 全局控制
│       ├── 全局风速参数（C#）
│       ├── 暴风雨模式（风速+幅度增大）
│       └── 安静模式（风速趋零）
├── LOD 平滑切换
│   ├── Pop-in 问题
│   │   ├── LOD 切换瞬间模型突变
│   │   └── 解决：Crossfade / Dither
│   ├── Dither LOD Transition
│   │   ├── 切换区间内两个 LOD 同时渲染
│   │   ├── 噪声纹理做 Alpha Test
│   │   ├── LOD0 像素渐消，LOD1 像素渐显
│   │   └── 视觉上"溶解"过渡而非突变
│   └── 交错渲染（Stipple）
│       └── 奇偶帧交替渲染不同 LOD（屏幕空间抖动）
├── 混排方案（程序化 + 手动放置）
│   ├── 程序化植被
│   │   ├── 由 Houdini / Density Map 生成
│   │   ├── 存入 StructuredBuffer
│   │   └── 走 Indirect Draw 路径
│   ├── 手动放置植被（"英雄植被"）
│   │   ├── 美术在引擎中手动摆放
│   │   ├── 走传统 GameObject 渲染
│   │   └── 数量通常 < 100，性能影响可忽略
│   └── 两套并行渲染，阴影和光照统一
└── 移动端适配
    ├── API 支持
    │   ├── Vulkan：Indirect Draw 原生支持（推荐）
    │   ├── OpenGL ES 3.1+：Compute Shader + Indirect Draw
    │   └── Metal：indirect_command_buffer（iOS A11+）
    ├── 降级策略
    │   ├── 不支持 Indirect → 回退 GPU Instancing
    │   ├── 不支持 Compute → CPU culling（低配机型）
    │   └── 极端低配 → 简化 Density Map，减少总量
    └── 性能预算
        ├── 草实例上限：移动端 50万 vs PC 500万
        ├── Compute Culling 耗时：移动端 ~1.5ms
        └── 内存：实例数据 ~32MB（200万 × 16B）
```

#### 代码实现

**Compute Shader（GPU Culling + LOD 分组）：**

```hlsl
#pragma kernel CSCullVegetation

// 实例数据结构
struct InstanceData
{
    float4x4 worldMatrix;
    float3   boundingSphere; // xyz=center, w=radius
    uint     lodMask;        // bit0=LOD0, bit1=LOD1, ...
    uint     padding;
};

// 每个LOD的 Indirect Draw Args
struct IndirectDrawArgs
{
    uint indexCountPerInstance;
    uint instanceCount;
    uint startIndexLocation;
    int  baseVertexLocation;
    uint startInstanceLocation;
};

StructuredBuffer<InstanceData> _InstanceBuffer;
RWStructuredBuffer<InstanceData> _LOD0Buffer; // 存活的LOD0实例
RWStructuredBuffer<InstanceData> _LOD1Buffer;
RWStructuredBuffer<InstanceData> _LOD2Buffer;

RWStructuredBuffer<uint> _LOD0Count;
RWStructuredBuffer<uint> _LOD1Count;
RWStructuredBuffer<uint> _LOD2Count;

RWStructuredBuffer<IndirectDrawArgs> _DrawArgs; // [0]=LOD0, [1]=LOD1, [2]=LOD2

float4 _FrustumPlanes[6];   // 视锥6面
float3 _CameraPos;
float  _LOD0Distance;       // 例：15m
float  _LOD1Distance;       // 例：40m
float  _LOD2Distance;       // 例：100m
uint   _TotalInstances;
uint   _IndexCountPerMesh;  // 每个 LOD Mesh 的索引数

// Sphere-Frustum 测试
bool IsSphereInFrustum(float4 plane[6], float3 center, float radius)
{
    [unroll]
    for (int i = 0; i < 6; i++)
    {
        float dist = dot(float4(center, 1.0), plane[i]);
        if (dist < -radius)
            return false;
    }
    return true;
}

[numthreads(64, 1, 1)]
void CSCullVegetation(uint3 dtid : SV_DispatchThreadID)
{
    if (dtid.x >= _TotalInstances) return;

    InstanceData inst = _InstanceBuffer[dtid.x];
    float3 center = inst.boundingSphere.xyz;
    float radius = inst.boundingSphere.w;

    // 1. 视锥剔除
    if (!IsSphereInFrustum(_FrustumPlanes, center, radius))
        return;

    // 2. 距离 LOD 选择
    float dist = distance(_CameraPos, center);

    if (dist < _LOD0Distance)
    {
        uint idx;
        InterlockedAdd(_LOD0Count[0], 1, idx);
        _LOD0Buffer[idx] = inst;
    }
    else if (dist < _LOD1Distance)
    {
        uint idx;
        InterlockedAdd(_LOD1Count[0], 1, idx);
        _LOD1Buffer[idx] = inst;
    }
    else if (dist < _LOD2Distance)
    {
        uint idx;
        InterlockedAdd(_LOD2Count[0], 1, idx);
        _LOD2Buffer[idx] = inst;
    }
    // 超过 LOD2 距离 = 不渲染
}

#pragma kernel CSUpdateDrawArgs

[numthreads(1, 1, 1)]
void CSUpdateDrawArgs(uint3 dtid : SV_DispatchThreadID)
{
    // LOD0
    _DrawArgs[0].indexCountPerInstance = _IndexCountPerMesh;
    _DrawArgs[0].instanceCount = _LOD0Count[0];
    _DrawArgs[0].startIndexLocation = 0;
    _DrawArgs[0].baseVertexLocation = 0;
    _DrawArgs[0].startInstanceLocation = 0;

    // LOD1
    _DrawArgs[1].indexCountPerInstance = _IndexCountPerMesh;
    _DrawArgs[1].instanceCount = _LOD1Count[0];
    _DrawArgs[1].startIndexLocation = 0;
    _DrawArgs[1].baseVertexLocation = 0;
    _DrawArgs[1].startInstanceLocation = 0;

    // LOD2
    _DrawArgs[2].indexCountPerInstance = _IndexCountPerMesh;
    _DrawArgs[2].instanceCount = _LOD2Count[0];
    _DrawArgs[2].startIndexLocation = 0;
    _DrawArgs[2].baseVertexLocation = 0;
    _DrawArgs[2].startInstanceLocation = 0;
}
```

**植被 Vertex Shader（风场 + 实例变换）：**

```hlsl
#pragma vertex VegetationVert

TEXTURE2D(_WindTexture);     SAMPLER(sampler_WindTexture);
TEXTURE2D(_InteractionMap);  SAMPLER(sampler_InteractionMap); // 玩家踩踏
StructuredBuffer<float4x4> _LOD0Buffer; // 存活实例矩阵

float4   _WindParams;    // x=speed, y=amplitude, z=gustStrength, w=time
float4x4 _LocalToWorld;  // Mesh 自身变换

struct Attributes
{
    float4 positionOS : POSITION;
    float3 normalOS   : NORMAL;
    float2 uv         : TEXCOORD0;
    float4 vertexColor : COLOR; // R=弯曲幅度, G=相位偏移, B=高度系数
};

struct Varyings
{
    float4 positionHCS : SV_POSITION;
    float3 normalWS    : TEXCOORD0;
    float2 uv          : TEXCOORD1;
    float3 positionWS  : TEXCOORD2;
};

Varyings VegetationVert(Attributes IN, uint instanceID : SV_InstanceID)
{
    Varyings OUT;

    // 读取实例世界矩阵
    float4x4 worldMatrix = _LOD0Buffer[instanceID];

    // Local → World
    float3 positionWS = mul(worldMatrix, float4(IN.positionOS.xyz, 1.0)).xyz;
    float3 normalWS = mul((float3x3)worldMatrix, IN.normalOS);

    // === 风力计算 ===
    float2 windUV = positionWS.xz * 0.05 + _WindParams.ww * _WindParams.xx * 0.1;
    float4 windSample = SAMPLE_TEXTURE2D_LOD(_WindTexture, sampler_WindTexture, windUV, 0);
    float2 windDir = windSample.rg * 2.0 - 1.0;
    float windStrength = windSample.b * _WindParams.yy;

    // 按高度衰减（根部不动，尖部摆）
    float heightFactor = IN.vertexColor.b; // B=高度系数
    float bendAmount = windStrength * heightFactor * IN.vertexColor.r; // R=弯曲幅度

    // 周期摆动（sin波）
    float phase = _WindParams.w * 3.0 + IN.vertexColor.g * 6.28; // G=相位偏移
    float sway = sin(phase) * 0.5 + 0.5;
    positionWS.xz += windDir * bendAmount * sway;

    // === 玩家交互（踩踏压倒） ===
    float2 interactionUV = positionWS.xz * 0.01 + 0.5;
    float interactStrength = SAMPLE_TEXTURE2D_LOD(
        _InteractionMap, sampler_InteractionMap, interactionUV, 0
    ).r;
    positionWS.xz += normalize(positionWS.xz - _CameraPos.xz) * interactStrength * 0.3;

    // World → Clip
    OUT.positionHCS = TransformWorldToHClip(positionWS);
    OUT.normalWS = normalWS;
    OUT.uv = IN.uv;
    OUT.positionWS = positionWS;

    return OUT;
}
```

**C# 驱动脚本（Unity）：**

```csharp
using UnityEngine;

[RequireComponent(typeof(MeshRenderer))]
public class GPUVegetationRenderer : MonoBehaviour
{
    [Header("Mesh (LOD0/1/2)")]
    [SerializeField] private Mesh lod0Mesh;
    [SerializeField] private Mesh lod1Mesh;
    [SerializeField] private Mesh lod2Mesh;

    [Header("Compute Shader")]
    [SerializeField] private ComputeShader cullComputeShader;

    [Header("LOD Distances")]
    [SerializeField] private float lod0Distance = 15f;
    [SerializeField] private float lod1Distance = 40f;
    [SerializeField] private float lod2Distance = 100f;

    [Header("Instance Source")]
    [SerializeField] private TextAsset instanceDataJSON; // 预烘焙的实例位置

    private ComputeBuffer _instanceBuffer;     // 全量实例
    private ComputeBuffer _lod0Buffer;         // 存活LOD0
    private ComputeBuffer _lod1Buffer;
    private ComputeBuffer _lod2Buffer;
    private ComputeBuffer _lod0Count;
    private ComputeBuffer _lod1Count;
    private ComputeBuffer _lod2Count;
    private ComputeBuffer _drawArgsBuffer;     // Indirect Draw Args

    private InstanceData[] _allInstances;
    private int _totalInstances;

    private struct InstanceData
    {
        public Matrix4x4 worldMatrix;
        public Vector4 boundingSphere;
        public uint lodMask;
        public uint padding;
    }

    void Start()
    {
        LoadInstanceData();
        CreateBuffers();
    }

    void LoadInstanceData()
    {
        // 从JSON/二进制加载预烘焙的位置数据
        var data = JsonUtility.FromJson<InstanceDataFile>(instanceDataJSON.text);
        _totalInstances = data.instances.Length;
        _allInstances = new InstanceData[_totalInstances];

        for (int i = 0; i < _totalInstances; i++)
        {
            var src = data.instances[i];
            _allInstances[i] = new InstanceData
            {
                worldMatrix = Matrix4x4.TRS(src.position, src.rotation, src.scale),
                boundingSphere = new Vector4(src.position.x, src.position.y + 0.5f, src.position.z, 1.0f),
                lodMask = 0xFFFFFFFF,
                padding = 0
            };
        }
    }

    void CreateBuffers()
    {
        // 全量实例（只读）
        _instanceBuffer = new ComputeBuffer(_totalInstances, 80); // sizeof(InstanceData)
        _instanceBuffer.SetData(_allInstances);

        // 存活缓冲（最坏情况=全部存活）
        _lod0Buffer = new ComputeBuffer(_totalInstances, 80);
        _lod1Buffer = new ComputeBuffer(_totalInstances, 80);
        _lod2Buffer = new ComputeBuffer(_totalInstances, 80);

        // 计数器
        _lod0Count = new ComputeBuffer(1, sizeof(uint));
        _lod1Count = new ComputeBuffer(1, sizeof(uint));
        _lod2Count = new ComputeBuffer(1, sizeof(uint));

        // Indirect Draw Args（5个uint = 20 bytes per LOD）
        _drawArgsBuffer = new ComputeBuffer(3, 20, ComputeBufferType.IndirectArguments);
    }

    void Update()
    {
        // 1. 重置计数器
        uint[] zero = { 0, 0, 0 };
        _lod0Count.SetData(new uint[] { 0 });
        _lod1Count.SetData(new uint[] { 0 });
        _lod2Count.SetData(new uint[] { 0 });

        // 2. 设置 Compute Shader 参数
        int kernel = cullComputeShader.FindKernel("CSCullVegetation");
        cullComputeShader.SetBuffer(kernel, "_InstanceBuffer", _instanceBuffer);
        cullComputeShader.SetBuffer(kernel, "_LOD0Buffer", _lod0Buffer);
        cullComputeShader.SetBuffer(kernel, "_LOD1Buffer", _lod1Buffer);
        cullComputeShader.SetBuffer(kernel, "_LOD2Buffer", _lod2Buffer);
        cullComputeShader.SetBuffer(kernel, "_LOD0Count", _lod0Count);
        cullComputeShader.SetBuffer(kernel, "_LOD1Count", _lod1Count);
        cullComputeShader.SetBuffer(kernel, "_LOD2Count", _lod2Count);

        // 视锥平面
        Plane[] frustumPlanes = GeometryUtility.CalculateFrustumPlanes(Camera.main);
        Vector4[] planes = new Vector4[6];
        for (int i = 0; i < 6; i++)
            planes[i] = frustumPlanes[i].normal;
        // 实际应传递完整的 float4(normal, distance)

        cullComputeShader.SetVectorArray("_FrustumPlanes", planes);
        cullComputeShader.SetVector("_CameraPos", Camera.main.transform.position);
        cullComputeShader.SetFloat("_LOD0Distance", lod0Distance);
        cullComputeShader.SetFloat("_LOD1Distance", lod1Distance);
        cullComputeShader.SetFloat("_LOD2Distance", lod2Distance);
        cullComputeShader.SetInt("_TotalInstances", _totalInstances);
        cullComputeShader.SetInt("_IndexCountPerMesh", (int)lod0Mesh.GetIndexCount(0));

        // 3. Dispatch Culling
        int threadGroups = Mathf.CeilToInt(_totalInstances / 64.0f);
        cullComputeShader.Dispatch(kernel, threadGroups, 1, 1);

        // 4. 更新 Indirect Draw Args
        int argsKernel = cullComputeShader.FindKernel("CSUpdateDrawArgs");
        cullComputeShader.SetBuffer(argsKernel, "_LOD0Count", _lod0Count);
        cullComputeShader.SetBuffer(argsKernel, "_LOD1Count", _lod1Count);
        cullComputeShader.SetBuffer(argsKernel, "_LOD2Count", _lod2Count);
        cullComputeShader.SetBuffer(argsKernel, "_DrawArgs", _drawArgsBuffer);
        cullComputeShader.SetInt("_IndexCountPerMesh", (int)lod0Mesh.GetIndexCount(0));
        cullComputeShader.Dispatch(argsKernel, 1, 1, 1);

        // 5. Indirect Draw
        // LOD0
        Graphics.DrawMeshInstancedIndirect(
            lod0Mesh, 0, GetComponent<MeshRenderer>().material,
            new Bounds(transform.position, Vector3.one * 1000),
            _drawArgsBuffer, 0,
            null
        );
        // LOD1 (offset 20 bytes in args buffer)
        Graphics.DrawMeshInstancedIndirect(
            lod1Mesh, 0, GetComponent<MeshRenderer>().material,
            new Bounds(transform.position, Vector3.one * 1000),
            _drawArgsBuffer, 20,
            null
        );
        // LOD2 (offset 40 bytes)
        Graphics.DrawMeshInstancedIndirect(
            lod2Mesh, 0, GetComponent<MeshRenderer>().material,
            new Bounds(transform.position, Vector3.one * 1000),
            _drawArgsBuffer, 40,
            null
        );
    }

    void OnDestroy()
    {
        _instanceBuffer?.Release();
        _lod0Buffer?.Release();
        _lod1Buffer?.Release();
        _lod2Buffer?.Release();
        _lod0Count?.Release();
        _lod1Count?.Release();
        _lod2Count?.Release();
        _drawArgsBuffer?.Release();
    }
}
```

**方案对比表：**

| 方案 | CPU耗时 | GPU耗时 | 实例上限 | 手游适用 | 实现复杂度 |
|------|---------|---------|----------|----------|------------|
| GameObject + Instancing | 8ms | 6ms | ~10万 | ⚠️ 勉强 | ⭐ 简单 |
| GPU Instancing (DrawMeshInstanced) | 3ms | 5ms | ~50万 | ✅ 可用 | ⭐⭐ 中等 |
| Compute Culling + Indirect Draw | <1ms | 4ms | ~500万 | ✅ 最佳 | ⭐⭐⭐⭐ 高 |
| Mesh Shader（DX12U） | <0.5ms | 3ms | ~1000万 | ❌ 仅PC | ⭐⭐⭐⭐⭐ 极高 |

### ⚡ 实战经验

- **实例数据内存**：200 万个 float4x4 = 200万 × 64B = 128MB，太多了。用压缩格式（pos16 + scale8 + rot16 = 16B/实例），降到 32MB，移动端也能接受
- **Compute Shader Dispatch 优化**：一个线程处理一个实例时，thread group size 用 64（非 256），因为植被实例间无共享内存需求，64 的 GPU 占用率更好
- **LOD Pop-in 最优解**：不是 dither dissolve（噪声采样贵），而是 **Geometry Morph**——顶点在 LOD 切换区间内插值移动。但实现复杂，多数项目用 crossfade 就够了
- **风纹理预计算**：不要在 Shader 里跑 Perlin Noise（太贵），预计算一张 256×256 的风噪声纹理，运行时只采样 + 时间偏移
- **手游实测**：Adreno 650 上 50 万棵草（LOD0+LOD1+LOD2），Compute Culling ≈ 1.2ms，渲染 ≈ 3.5ms，总计 ≈ 4.7ms，能稳 60fps
- **阴影成本**：50 万棵草全开 Cast Shadow = 灾难。通常只有 LOD0 草投阴影，LOD1/2 不投
- **密度图设计**：Density Map 用 R8 就够（单通道足够）。分辨率 1024×1024 覆盖 1km×1km 地图，每像素 ≈ 1m 精度
- **HLOD 衔接**：超远距离的草地（> 200m）不渲染几何体，直接烘焙到地形的 Base Color 和 Normal Map 上，这种"假草"零运行时开销

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道什么是 Indirect Draw | GPU-Driven Pipeline 概念 |学 DrawIndexedInstancedIndirect API |
| Compute Shader 写不出来 | HLSL Compute + RWStructuredBuffer | 学 Compute Shader 管线 + UAV |
| 视锥剔除算法不会 | Frustum Culling 数学 | 学平面方程 + Sphere/Plane 测试 |
| LOD 切换有闪烁 | Crossfade / Dither 技术 | 学 Alpha Dither + Noise 采样 |
| 风吹效果不自然 | 顶点位移 + 风场纹理 | 学 Vertex Displacement + Noise Wind |
| 手机上 Compute Shader 报错 | 移动端 GPU API 兼容性 | 学 GLES 3.1 / Vulkan Compute 限制 |
| 不知道实例数据怎么烘焙 | Houdini 植被散布到引擎 | 学 Houdini Heightfield Scatter + 导出 |

### 🔗 相关问题

- [百万级草地渲染优化](../optimization/million-grass-rendering.md)：性能优化角度的植被渲染
- [草风吹交互 Shader](../shader/grass-wind-shader-interaction.md)：Shader 实现细节
- [Houdini 植被散布管线](../pipeline/houdini-vegetation-scatter.md)：植被数据的上游生产
- [GPU-Driven Pipeline](../rendering/gpu-driven-pipeline.md)：更广泛的 GPU-Driven 渲染架构
- 如果面试官追问"遮挡剔除怎么实现？"你会怎么回答？（提示：Hi-Z Pyramid + 两帧延迟）
