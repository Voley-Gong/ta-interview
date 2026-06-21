---
title: "Forward+ 聚类渲染：同屏 100+ 动态光源怎么不卡死？"
category: "rendering"
level: 3
tags: ["Forward+", "Cluster", "Tile-Based", "多光源", "Compute Shader", "渲染管线"]
hint: "核心是屏幕空间分块（Cluster）+ 深度分层 → Compute Shader 剔除光源 → 前向渲染读取光源列表"
related: ["rendering/urp-renderer-feature", "rendering/deferred-multi-light", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们在做一款夜景为主的动作游戏，场景里有大量动态光源——路灯、火把、手电筒、车辆前照灯、枪口闪光……同屏可能有 100-200 个光源。传统 Forward Rendering 只能处理少量实时光源（4-8 个），Deferred Rendering 可以处理很多光源但 MSAA 固难、透明物体不兼容。你会怎么设计一套能高效处理上百光源的渲染方案？

这是字节游戏、腾讯天美、米哈游等在高品质手游/端游项目中的渲染架构面试题。考察的是 Forward+（Cluster Forward）渲染管线的完整理解。

### ✅ 核心要点

1. **Forward+ = Forward + 光源剔除**：保持前向渲染的优势（MSAA、透明），但先用 Compute Pass 剔除光源
2. **屏幕空间 Cluster 分块**：将屏幕分成 2D Tile 或 3D Cluster（加深度切分），每个块只记录影响它的光源
3. **Compute Shader 光源剔除**：GPU 并行做 AABB/球体 vs Cluster 的碰撞检测，生成 Per-Cluster Light List
4. **渲染阶段读取列表**：Pixel Shader 从 Light Index List 中读取光源索引，只计算相关的光源
5. **移动端适配**：Vulkan/Metal 的 Compute Shader 支持是前提，GLES 3.1+ 也可行但需注意限制

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：同屏 100-200 动态光源，60fps，支持 MSAA 和半透明
                ↑
为什么 Forward 不行？
  → 每个 Pixel 要遍历所有光源 → 100 光源 × 全屏像素 = 灾难
                ↑
为什么 Deferred 不行？
  → G-Buffer 带宽高（移动端 4-8 张 MRT）、MSAA 困难、半透明需要单独 Forward Pass
                ↑
Forward+ 方案：
  Step 1：分块（Cluster Build）— 屏幕切 Tile + 深度切 Slice
  Step 2：光源剔除（Culling）— Compute Shader 算哪些光源影响哪些 Cluster
  Step 3：前向渲染（Shading）— 每个像素只遍历所在 Cluster 的光源列表
```

#### 知识点拆解（倒推树）

```
Forward+ 渲染管线
├── Cluster 构建
│   ├── 2D Tiling（屏幕空间分块）
│   │   ├── 16×16 或 32×32 像素一个 Tile
│   │   └── 简单但深度精度不均匀
│   ├── 3D Clustering（推荐）
│   │   ├── Tile × 深度 Slice（对数分布）
│   │   └── 深度切片公式：zSlice = near * (far/near)^(slice/numSlices)
│   └── 数据结构
│       ├── Cluster AABB（minPoint, maxPoint）
│       └── SSBO/UAV 存储到 GPU Buffer
│
├── 光源剔除（Compute Shader）
│   ├── 输入
│   │   ├── 光源数组（position, range, color, intensity）
│   │   └── Cluster AABB 数组
│   ├── 剔除算法
│   │   ├── 球体 vs AABB 碰撞检测
│   │   ├── 圆锥（Spotlight）vs AABB（更复杂）
│   │   └── 视锥剔除（摄像机后面的光源先排除）
│   ├── 输出
│   │   ├── Per-Cluster Light Index List（变长列表）
│   │   ├── 用全局 Light Index Buffer + Offset/Count 结构
│   │   └── 典型布局：lightGrid[clusterID] = (offset, count)
│   └── 并行策略
│       ├── 每个 Thread Group 处理一个 Cluster
│       ├── Group Shared Memory 缓存光源列表
│       └── LDS（Local Data Share）加速碰撞检测
│
├── 前向渲染（Shading Pass）
│   ├── Pixel Shader 读取流程
│   │   ├── 计算当前像素所属 Cluster ID
│   │   ├── 查 lightGrid → 得到 (offset, count)
│   │   ├── 遍历 lightIndexBuffer[offset .. offset+count]
│   │   └── 每个光源计算光照贡献
│   ├── BRDF 计算
│   │   └── 和普通 Forward 一样（Blinn-Phong / PBR / Cook-Torrance）
│   └── 多光源阴影
│       ├── 每光源 Cube Shadow Map（6面）
│       └── 优化：级联 Shadow Atlas + Per-Cluster Shadow LOD
│
├── 透明物体渲染
│   ├── Forward+ 天然支持透明（和 Forward 一样）
│   ├── 透明物体也读取 Cluster Light List
│   └── 这是 Forward+ 相比 Deferred 的核心优势
│
└── MSAA 兼容性
    ├── Forward+ 保持 Forward 的 MSAA 能力
    ├── 每个样本（Sample）读取同一 Cluster 的光源列表
    └── 比 Deferred 的 MSAA 实现简单得多
```

#### 代码实现

**Compute Shader — Cluster 光源剔除（核心）：**

```hlsl
#pragma kernel LightCulling

struct PointLight {
    float3 position;    // 世界坐标
    float  range;       // 影响范围
    float3 color;       // 颜色
    float  intensity;   // 强度
};

StructuredBuffer<PointLight> _AllLights : register(t0);
RWStructuredBuffer<uint>      _LightIndexBuffer : register(u0);  // 全局光源索引缓冲
RWBuffer<uint2>               _LightGrid : register(u1);          // per-cluster (offset, count)

cbuffer ClusterInfo {
    uint   _NumClustersX;
    uint   _NumClustersY;
    uint   _NumClustersZ;
    uint   _NumLights;
    float  _ScreenSizeX;
    float  _ScreenSizeY;
    float  _NearPlane;
    float  _FarPlane;
    float4x4 _ViewMatrix;
    float4x4 _ViewInvMatrix;
    float4x4 _ProjMatrix;
    float4x4 _ProjInvMatrix;
};

// Cluster 尺寸（像素）
#define TILE_SIZE 16
#define NUM_DEPTH_SLICES 24

groupshared uint gs_LightList[256];  // LDS 缓存候选光源
groupshared uint gs_LightCount = 0;

[numthreads(TILE_SIZE, TILE_SIZE, 1)]
void LightCulling(uint3 groupID : SV_GroupID, uint groupIndex : SV_GroupIndex) {
    // === 1. 计算 Cluster AABB（视空间） ===
    uint clusterZ = groupID.z;
    
    // 对数深度分布
    float near = _NearPlane;
    float far = _FarPlane;
    float zNear = near * pow(far / near, (float)clusterZ / NUM_DEPTH_SLICES);
    float zFar  = near * pow(far / near, (float)(clusterZ + 1) / NUM_DEPTH_SLICES);
    
    // Tile 的屏幕坐标范围
    float2 tileMin = groupID.xy * TILE_SIZE;
    float2 tileMax = tileMin + TILE_SIZE;
    
    // 屏幕角 → 视空间射线
    float4 ndcMin = float4(tileMin / float2(_ScreenSizeX, _ScreenSizeY) * 2 - 1, -1, 1);
    float4 ndcMax = float4(tileMax / float2(_ScreenSizeX, _ScreenSizeY) * 2 - 1, -1, 1);
    
    float4 viewMin = mul(_ProjInvMatrix, float4(ndcMin.xy, zNear, 1));
    viewMin /= viewMin.w;
    float4 viewMax = mul(_ProjInvMatrix, float4(ndcMax.xy, zFar, 1));
    viewMax /= viewMax.w;
    
    // Cluster AABB（视空间）
    float3 aabbMin = min(viewMin.xyz, viewMax.xyz);
    float3 aabbMax = max(viewMin.xyz, viewMax.xyz);
    
    // === 2. 光源遍历（每个 Thread 检查一个光源分片） ===
    if (groupIndex == 0) gs_LightCount = 0;
    GroupMemoryBarrierWithGroupSync();
    
    uint lightsPerThread = max(1, _NumLights / 256);
    for (uint i = groupIndex; i < _NumLights; i += 256) {
        PointLight light = _AllLights[i];
        
        // 世界 → 视空间
        float3 lightViewPos = mul(_ViewMatrix, float4(light.position, 1)).xyz;
        
        // 球体 vs AABB 碰撞检测
        float3 closest = clamp(lightViewPos, aabbMin, aabbMax);
        float3 diff = lightViewPos - closest;
        float distSq = dot(diff, diff);
        
        if (distSq <= light.range * light.range) {
            // 命中！写入 LDS
            uint slot;
            InterlockedAdd(gs_LightCount, 1, slot);
            gs_LightList[slot] = i;
        }
    }
    GroupMemoryBarrierWithGroupSync();
    
    // === 3. 写回全局 Buffer ===
    if (groupIndex == 0) {
        uint clusterID = groupID.z * _NumClustersX * _NumClustersY + groupID.y * _NumClustersX + groupID.x;
        
        // 申请 offset（全局原子操作）
        uint offset;
        InterlockedAdd(_LightIndexBuffer[0], gs_LightCount, offset); // [0] 是全局计数器
        
        _LightGrid[clusterID] = uint2(offset + 1, gs_LightCount); // +1 跳过计数器
    }
    GroupMemoryBarrierWithGroupSync();
    
    // 每个 Thread 写入光源索引
    if (groupIndex < gs_LightCount) {
        uint clusterID = groupID.z * _NumClustersX * _NumClustersY + groupID.y * _NumClustersX + groupID.x;
        uint offset = _LightGrid[clusterID].x;
        _LightIndexBuffer[offset + groupIndex] = gs_LightList[groupIndex];
    }
}
```

**Pixel Shader — 多光源着色：**

```hlsl
#include "ForwardPlusCommon.hlsl"

// 光源数据（来自 Compute Pass）
StructuredBuffer<uint>  _LightIndexBuffer;
Buffer<uint2>           _LightGrid;
StructuredBuffer<PointLight> _Lights;

// Cluster 参数
cbuffer CB {
    uint _NumClustersX;
    uint _NumClustersY;
    uint _NumClustersZ;
    float _TileSize;
    uint _NumDepthSlices;
    float _Near;
    float _Far;
};

struct Varyings {
    float4 positionHCS : SV_POSITION;
    float3 worldPos : TEXCOORD0;
    float3 normalWS : TEXCOORD1;
    float2 uv : TEXCOORD2;
};

// 计算像素所属 Cluster ID
uint3 ComputeClusterID(float2 pixelCoord, float viewDepth) {
    uint x = floor(pixelCoord.x / _TileSize);
    uint y = floor(pixelCoord.y / _TileSize);
    // 对数深度分布
    float z = log2(max(viewDepth / _Near, 1.0)) / log2(_Far / _Near) * _NumDepthSlices;
    uint zSlice = min(uint(z), _NumDepthSlices - 1);
    return uint3(x, y, zSlice);
}

half4 ForwardPlusFrag(Varyings IN) : SV_Target {
    float2 pixelCoord = IN.positionHCS.xy;
    float viewDepth = -TransformWorldToView(IN.worldPos).z;
    
    // 查找 Cluster
    uint3 clusterID = ComputeClusterID(pixelCoord, viewDepth);
    uint linearID = clusterID.z * _NumClustersX * _NumClustersY 
                  + clusterID.y * _NumClustersX + clusterID.x;
    
    uint2 lightGridEntry = _LightGrid[linearID];
    uint lightOffset = lightGridEntry.x;
    uint lightCount  = lightGridEntry.y;
    
    // 基础材质参数
    half3 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv).rgb;
    half3 N = normalize(IN.normalWS);
    half  roughness = 0.5;
    half  metallic = 0.0;
    
    // 环境光
    half3 ambient = SampleSH(N) * albedo * 0.3;
    half3 finalColor = ambient;
    
    // 主光源（太阳）
    half3 mainLightDir = normalize(_MainLightPosition.xyz);
    half3 mainLightColor = _MainLightColor.rgb;
    half NdotL = max(0, dot(N, mainLightDir));
    finalColor += albedo * mainLightColor * NdotL;
    
    // === 遍历 Cluster 光源列表 ===
    [loop]
    for (uint i = 0; i < lightCount; i++) {
        uint lightIdx = _LightIndexBuffer[lightOffset + i];
        PointLight light = _Lights[lightIdx];
        
        float3 L = light.position - IN.worldPos;
        float dist = length(L);
        L /= max(dist, 0.001);
        
        // 距离衰减
        float attenuation = 1.0 - saturate(dist / light.range);
        attenuation *= attenuation; // 平方衰减更自然
        
        half3 lightContribution = light.color * light.intensity * attenuation;
        
        // Diffuse
        half NdotL_local = max(0, dot(N, L));
        finalColor += albedo * lightContribution * NdotL_local;
    }
    
    return half4(finalColor, 1.0);
}
```

**Forward vs Deferred vs Forward+ 对比表：**

| 维度 | Forward | Deferred | Forward+ |
|------|---------|----------|----------|
| 多光源性能 | ❌ O(像素 × 光源数) | ✅ O(像素 + 光源数) | ✅ O(像素 × Cluster光源数) |
| MSAA 支持 | ✅ 原生支持 | ❌ 困难 | ✅ 原生支持 |
| 透明物体 | ✅ 原生支持 | ❌ 需 Forward Pass | ✅ 原生支持 |
| 材质多样性 | ✅ 无限制 | ❌ 受 G-Buffer 限制 | ✅ 无限制 |
| 带宽开销 | 低 | 高（G-Buffer） | 中（Light Index Buffer） |
| 移动端友好度 | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| 实现复杂度 | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

### ⚡ 实战经验

1. **Cluster 数量调参是关键**：16×16 Tile + 24 深度切片在 1080p 下产生约 19000 个 Cluster。太大 → 剔除不精确，太小 → Compute Shader 开销高。移动端可考虑 32×32 + 16 切片
2. **光源数据压缩**：每个 PointLight 可以打包成 `float4(position+range) + float4(color+intensity)` = 32 字节。200 个光源仅 6.4KB，完全可以每帧更新
3. **Spotlight 剔除更复杂**：Point Light 用球体-AABB 测试即可，Spotlight 需要圆锥-AABB 测试或用球+平面近似。建议先用球体做粗剔除，再在着色阶段做圆锥角度判断
4. **移动端 Compute Shader 注意事项**：Adreno GPU 的 Compute Shader 性能不如同代 Mali/Apple。高通平台建议用 FP16 光源数据 + 尽量减少 LDS 原子操作

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不理解 Cluster 的深度切分 | 对数深度分布原理 | 复习透视投影的深度分布特性、Z 精度问题 |
| Compute Shader 写不出来 | GPU 编程模型、Group/Thread | 学 Compute Shader 基础（numthreads、GroupSync、LDS） |
| 不知道光源剔除怎么做碰撞 | 球体 vs AABB 碰撞检测 | 复习计算几何：最近点-距离判定 |
| 搞不清几种渲染管线的优劣 | Forward/Deferred/Forward+ 架构 | 做一张对比表（G-Buffer、MSAA、透明、材质多样性） |
| 不知道怎么在 URP 中实现 | URP 自定义管线扩展 | 研究 URP Renderer Feature + Compute Pass 注入 |

### 🔗 相关问题

- 延迟渲染下如何处理多光源？（提示：G-Buffer + Light Volume + 全屏 Light Pass）
- Forward+ 的光源阴影怎么处理？（提示：Per-Light Shadow Map Atlas + Cascade）
- UE5 的 Nanite + Lumen 和 Forward+ 有什么关系？（Lumen GI 也用 Cluster 加速光线追踪）
