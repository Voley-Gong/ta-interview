---
title: "头发渲染方案选型：Strand Hair 还是 Hair Card？面试官问你怎么选"
category: "technical-art"
level: 3
tags: ["头发渲染", "StrandHair", "HairCard", "Marschner", "各向异性", "性能优化", "跨平台"]
hint: "核心不是哪个更好——而是根据平台预算、品质需求、工具链成熟度做选型矩阵"
related: ["shader/hair-anisotropic-lighting", "technical-art/character-material-spec-workflow", "optimization/gpu-bandwidth-optimization"]
---

## 参考答案

### 🎬 场景描述

面试官展示两张角色截图——一张是《黑神话：悟空》中细腻飘逸的头发（Strand-based），另一张是《原神》中风格化的头发（Hair Card）。然后问：

> "我们新项目是一个跨平台角色 ARPG，PC 端想要 3A 级头发品质，移动端要 60fps。你来定头发渲染方案——Strand 还是 Card？还是混合？给我完整的选型分析和技术方案。"

这是叠纸、鹰角、米哈游等做高品质角色项目的 **TA 架构师必问题**，考察的不只是技术知识，更是决策方法论。

### ✅ 核心要点

1. **Strand Hair（毛发/线缆渲染）**：逐发丝几何体，物理正确的高光和阴影，但计算量巨大
2. **Hair Card（卡片头发）**：用透明面片 + 发丝纹理模拟头发束，性能友好但表现力有上限
3. **Marschner 反射模型**：头发光照的行业标准——R（反射）、TT（透射）、TRT（内反射透射）三条散射路径
4. **混合策略**：PC 用 Strand + 移动端用 Card，通过统一的资产管线做自动转换
5. **关键决策维度**：平台性能预算、品质标准、制作工期、工具链成熟度

### 📖 深度展开

#### 解决思路（从需求倒推方案）

```
需求：跨平台 ARPG，PC 3A 品质，移动端 60fps
                    ↓
问题分解：
  ├── 品质需求：PC 端头发需要逐丝高光、自阴影、物理飘动
  ├── 性能需求：移动端头发预算 ≤ 1.5ms（GPU）
  ├── 工作流：美术用同一套 DCC 工具制作 → 自动导出两种方案
  └── 一致性：跨平台视觉差异不能太大（玩家不会觉得是两个游戏）
                    ↓
方案矩阵：
  ┌─────────────┬──────────────┬──────────────┐
  │             │  Strand Hair  │  Hair Card   │
  ├─────────────┼──────────────┼──────────────┤
  │ 视觉品质     │ ★★★★★       │ ★★★☆☆       │
  │ GPU 开销     │ 3-6ms        │ 0.3-0.8ms   │
  │ 内存占用     │ 高（每丝数据）│ 低（纹理+面片）│
  │ 制作复杂度   │ 高（XGen/Houdini）│ 中（PS笔刷）│
  │ 移动端可行   │ ❌            │ ✅           │
  │ 物理模拟     │ ✅（原生支持） │ ⚠️（顶点着色器模拟）│
  └─────────────┴──────────────┴──────────────┘
                    ↓
推荐方案：混合管线
  ├── PC：Strand Hair（Wwise Hair 或 TressFX）
  ├── 移动端：Hair Card（从 Strand 烘焙转换）
  ├── 统一美术管线：XGen → Strand → Bake to Card
  └── 视觉一致性：共享 Marschner 光照参数
```

#### 知识点拆解（倒推树）

```
头发渲染方案选型
├── Marschner 反射模型（理论基础）
│   ├── 头发的圆柱体假设
│   ├── 三条散射路径
│   │   ├── R：表面反射（高光主峰）
│   │   ├── TT：两次透射（背光透射）
│   │   └── TRT：内反射+透射（次高光，偏移色）
│   ├── M_r / M_tt / M_trt：三个高斯分布纵向散射
│   └── N_r / N_tt / N_trt：三个方位角散射函数
├── Strand Hair（线缆渲染）
│   ├── 几何体：Bezier 曲线 / Line Strip
│   ├── 数据量：每角色 1万~10万根发丝
│   ├── 渲染管线
│   │   ├── Hair Depth Pass：发丝深度写入
│   │   ├── Hair Shadow Pass：发丝自阴影
│   │   ├── Hair Shading Pass：Marschner 着色
│   │   └── Hair OIT：顺序无关半透明
│   ├── 物理模拟
│   │   ├── Mass-Spring System：质量弹簧系统
│   │   ├── PBD（Position Based Dynamics）
│   │   └── Wind / Collision：风力和碰撞响应
│   ├── 代表方案
│   │   ├── Wwise Hair（UE 插件）
│   │   ├── AMD TressFX
│   │   ├── NVIDIA HairWorks（已弃用）
│   │   └── Unity Hair Asymptote（第三方）
│   └── 性能瓶颈
│       ├── 几何体处理：10万根 × 10段 = 100万顶点
│       ├── OverDraw：发丝密集区域的像素覆盖
│       └── 半透明排序：OIT 或 per-pixel linked list
├── Hair Card（卡片头发）
│   ├── 几何体：透明面片（Card / Patch）
│   ├── 纹理
│   │   ├── 发束透明度图（Alpha Map）
│   │   ├── 发束深度图（Root/Tip Depth）
│   │   ├── ID Map（发束分组，用于偏移变化）
│   │   └── Flow Map（发流方向）
│   ├── 着色
│   │   ├── Marschner 近似（卡片上跑简化版）
│   │   ├── Shift Map：偏移高光位置
│   │   └── 二次高光（TRT）：噪声纹理扰动
│   ├── 飘动
│   │   ├── 顶点色 / UV2 控制飘动权重
│   │   ├── Wind Noise 纹理采样
│   │   └── Root Pin：根部固定、尖端自由摆动
│   └── 制作流程
│       ├── XGen / Houdini 生成 Strand → 烘焙 Card 纹理
│       ├── Photoshop 手绘发丝（传统流程）
│       └── 程序化生成（Substance Painter Hair Card）
├── Strand → Card 烘焙转换
│   ├── 将 Strand 渲染到正交相机 → 生成 Alpha/Depth/ID Map
│   ├── LOD 生成：近 Strand → 中 Card → 远 Billboard
│   └── 自动工具：Houdini PDG / Python 脚本
├── 自阴影
│   ├── Strand 自阴影：Deep Shadow Map（深度阴影图）
│   ├── Card 自阴影：常规 ShadowMap + 透明度修正
│   └── Approximate：球谐 / 球形 Gaussian 近似
├── 半透明排序
│   ├── OIT（Order-Independent Transparency）
│   ├── Per-Pixel Linked List（每像素链表）
│   ├── WBOIT（Weighted Blended OIT）—— 移动端可行
│   └── Alpha Test + 手动排序（Hair Card 常用）
└── 跨平台一致性
    ├── 光照参数共享：Marschner R/TT/TRT 系数
    ├── 颜色映射：Diffuse + Specular 系数对齐
    └── 动态切换：根据平台自动加载 LOD 模型
```

#### 代码实现

**1. Marschner 头发着色（简化版，适用于 Strand 和 Card）**

```hlsl
// Marschner 头发反射模型 - 简化版
// 参考: "Light Scattering from Human Hair Fibers" (Marschner et al., 2003)

// === 预计算的散射函数近似 ===
// 用解析函数代替完整的 Marschner 积分

float3 MarschnerHairBSDF(
    float3 L,           // 光照方向（指向光源）
    float3 V,           // 视线方向（指向相机）
    float3 T,           // 发丝切线方向（沿发丝生长方向）
    float3 baseColor,   // 头发基础色（黑色素吸收）
    out float  shadowTerm // 自阴影项输出
) {
    // === 计算散射角 ===
    float3 N = normalize(cross(T, float3(0, 1, 0) + 1e-4));
    float sinThetaL = dot(L, T);
    float cosThetaL = sqrt(1.0 - sinThetaL * sinThetaL);
    float sinThetaR = dot(V, T);
    float cosThetaR = sqrt(1.0 - sinThetaR * sinThetaR);

    // 方位角差
    float phiL = atan2(dot(L, N), dot(L, normalize(cross(N, T))));
    float phiR = atan2(dot(V, N), dot(V, normalize(cross(N, T))));
    float phiDiff = phiR - phiL;

    // === R 分量（表面反射 - 主高光） ===
    // 纵向散射：高斯分布
    float thetaH = sinThetaL + sinThetaR;
    float thetaH2 = thetaH * thetaH;
    float M_R = exp(-thetaH2 / (2.0 * 0.1 * 0.1)) /
               (sqrt(2.0 * PI) * 0.1);

    // 方位角散射：锐利峰
    float N_R = 0.5 * (1.0 + cos(PI - 2.0 * phiDiff));

    float3 R = M_R * N_R * float3(1.0, 1.0, 1.0); // R 通常无颜色吸收

    // === TT 分量（透射 - 背光边缘光） ===
    float M_TT = exp(-(thetaH2) / (2.0 * 0.3 * 0.3)) /
                 (sqrt(2.0 * PI) * 0.3);

    // TT 方位角：与折射率相关的偏移
    float eta = 1.55; // 头发折射率
    float phiTT = 2.0 * asin(sin(phiDiff / 2.0) / eta);
    float N_TT = 0.5 * (1.0 + cos(phiTT));

    // TT 颜色吸收：与发丝半径和颜色相关
    float3 absorptionTT = exp(-baseColor * 2.0);

    float3 TT = M_TT * N_TT * absorptionTT;

    // === TRT 分量（内反射 - 次高光，有色偏移） ===
    float M_TRT = exp(-thetaH2 / (2.0 * 0.2 * 0.2)) /
                  (sqrt(2.0 * PI) * 0.2);
    float N_TRT = 0.5 * (1.0 + cos(3.0 * phiDiff));

    // TRT 有额外的颜色吸收（光在发丝内部走了更长的路）
    float3 absorptionTRT = exp(-baseColor * 4.0);

    float3 TRT = M_TRT * N_TRT * absorptionTRT;

    // === 合成 ===
    float3 result = R + TT + TRT;

    // 添加漫反射项（头发不是纯镜面反射）
    float3 diffuse = baseColor * 0.25 * cosThetaL;

    shadowTerm = 1.0; // 外部计算
    return (result + diffuse) * cosThetaL;
}

// === 在 Fragment Shader 中调用 ===
half4 FragHair(Varyings input) : SV_Target
{
    float3 N = normalize(input.normalWS);
    float3 T = normalize(input.tangentWS);   // 发丝方向
    float3 V = normalize(_WorldSpaceCameraPos - input.positionWS);
    float3 L = normalize(_MainLightPosition.xyz);

    // 采样发束纹理（Card 方案用，Strand 方案可省略）
    half4 hairTex = SAMPLE_TEXTURE2D(_HairMap, sampler_HairMap, input.uv);
    clip(hairTex.a - 0.1); // Alpha Test

    // 自阴影（从 ShadowMap 采样）
    float shadow = ComputeShadow(input.positionWS, input.shadowCoord);

    float shadowTerm;
    float3 hairColor = MarschnerHairBSDF(L, V, T, _HairColor.xyz, shadowTerm);
    hairColor *= hairTex.rgb;   // 混合发束纹理颜色
    hairColor *= shadow;        // 应用自阴影

    // 高光偏移（Shift Map 控制高光位置，模拟头发丝滑感）
    float shift = SAMPLE_TEXTURE2D(_ShiftMap, sampler_ShiftMap, input.uv).r;
    // 这里简化处理：实际中 shift 会影响切线方向的扰动

    return half4(hairColor, 1.0);
}
```

**2. Hair Card 飘动 Vertex Shader**

```hlsl
// 头发卡片飘动 - Vertex Shader
// 原理：根部固定，尖端自由摆动，用顶点色 R 通道控制权重

Varyings VertHairCard(Attributes input)
{
    Varyings output = (Varyings)0;

    float3 posOS = input.positionOS.xyz;

    // === 飘动权重 ===
    // 顶点色 R：0=根部（固定），1=尖端（自由摆动）
    float swayWeight = input.color.r;

    // === 风力偏移 ===
    // 用世界坐标 + 时间 采样噪声纹理，产生空间和时间上的风
    float3 worldPos = TransformObjectToWorld(input.positionOS.xyz);
    float2 windUV = worldPos.xz * _WindTiling + _Time.y * _WindSpeed;
    float2 windNoise = SAMPLE_TEXTURE2D_LOD(_WindNoiseMap, sampler_WindNoiseMap, windUV, 0).rg;
    windNoise = (windNoise - 0.5) * 2.0; // [-1, 1]

    // 风力偏移量（尖端摆动幅度大）
    float3 windOffset = float3(windNoise.x, 0, windNoise.y) * _WindStrength * swayWeight;

    // === 重力下垂（角色移动时头发惯性飘起再落下） ===
    float3 velocity = _CharacterVelocity; // C# 每帧设置
    float3 gravityOffset = -velocity * swayWeight * _InertiaFactor;

    // === 应用偏移 ===
    posOS += TransformWorldToObject(windOffset + gravityOffset);

    output.positionHCS = TransformObjectToHClip(posOS);
    output.uv = input.uv;
    output.normalWS = TransformObjectToWorldNormal(input.normalOS);
    output.tangentWS = TransformObjectToWorldDir(input.tangentOS);
    output.color = input.color;

    return output;
}
```

**3. Strand → Card 烘焙工具（Houdini Python 伪代码）**

```python
# Houdini Strand to Hair Card 烘焙脚本
# 将 XGen Strand 头发烘焙成 Hair Card 纹理 + 几何体

import hou

def bake_strand_to_card(strand_node, output_dir, card_resolution=512):
    """将 Strand 头发转换为 Card 资产"""

    # 1. 提取 Strand 曲线
    curves = strand_node.geometry().curves()
    print(f"[HairBaker] Found {len(curves)} strand curves")

    # 2. 按发束分组聚类（ID Map）
    groups = cluster_strands_by_proximity(curves, threshold=0.05)
    print(f"[HairBaker] Clustered into {len(groups)} hair cards")

    # 3. 为每组发束生成正交相机
    card_assets = []
    for i, group in enumerate(groups):
        # 计算发束的主轴方向和包围盒
        main_axis, bbox = compute_bounding_box(group)

        # 创建正交相机
        cam = create_ortho_camera(main_axis, bbox, resolution=card_resolution)

        # 4. 渲染 Alpha Map（透明度）
        alpha_map = render_alpha_pass(group, cam, card_resolution)

        # 5. 渲染 Depth Map（深度，用于着色视差）
        depth_map = render_depth_pass(group, cam, card_resolution)

        # 6. 渲染 ID Map（发丝分组，用于偏移变化）
        id_map = render_id_pass(group, cam, card_resolution)

        # 7. 生成 Card 几何体（四边形面片）
        card_mesh = generate_card_geometry(bbox, main_axis, card_index=i)

        # 8. 保存纹理
        alpha_map.save(f"{output_dir}/hair_card_{i:03d}_alpha.png")
        depth_map.save(f"{output_dir}/hair_card_{i:03d}_depth.png")
        id_map.save(f"{output_dir}/hair_card_{i:03d}_id.png")

        card_assets.append({
            'mesh': card_mesh,
            'textures': {'alpha': alpha_map, 'depth': depth_map, 'id': id_map}
        })

    # 8. 合并所有 Card 为一个 Mesh
    final_mesh = merge_cards(card_assets, output_dir)
    print(f"[HairBaker] Done. Output: {final_mesh}")

    return final_mesh

def cluster_strands_by_proximity(curves, threshold):
    """按空间邻近度将 Strand 聚类为发束"""
    groups = []
    assigned = set()

    for i, curve in enumerate(curves):
        if i in assigned:
            continue

        group = [curve]
        assigned.add(i)
        root_pos = curve.points()[0].position()

        for j, other in enumerate(curves[i+1:], start=i+1):
            if j in assigned:
                continue
            other_root = other.points()[0].position()
            if (root_pos - other_root).length() < threshold:
                group.append(other)
                assigned.add(j)

        groups.append(group)

    return groups
```

**方案选型决策表**

| 维度 | Strand Hair | Hair Card | 混合方案 |
|------|------------|-----------|---------|
| 视觉品质 | 3A 级（物理正确） | 良好（风格化项目够用） | 按平台自适应 |
| GPU 开销 | 3-6ms（PC） | 0.3-0.8ms（移动端） | 按平台分配 |
| 内存 | 高（每丝数据） | 低（纹理+面片） | LOD 分级 |
| 制作工期 | 长（XGen+物理调试） | 中（烘焙/手绘） | 一次制作两份导出 |
| 物理飘动 | 原生支持 | 顶点着色器模拟 | 各自方案 |
| 自阴影 | Deep Shadow Map | ShadowMap + Alpha | 各自方案 |
| 移动端可行 | ❌ | ✅ | ✅ |
| 代表项目 | 黑神话/对马岛/地平线 | 原神/星铁/鸣潮 | 叠纸/腾讯项目 |

### ⚡ 实战经验

**移动端 Hair Card 的三个性能杀手**

1. **OverDraw**：卡片头发密集区域 OverDraw 可达 10-20x。必须用 Alpha Test 而非 Alpha Blend，配合 Pre-Z Pass 减少填充率浪费
2. **Draw Call**：一个角色可能有 100+ 张 Card。需要合并 Mesh + GPU Instancing（同材质的 Card 可以合批）
3. **半透明排序**：如果用 Alpha Blend，需要按距离排序，CPU 开销大。建议用 Alpha Test + WBOIT 的折中方案

**从 Strand 烘焙 Card 的最大陷阱**

烘焙时正交相机的角度选择至关重要。如果相机角度和实际运行时视角偏差太大，会出现：
- 深度信息错误（着色视差不对）
- 高光位置不对（Marschner 依赖切线方向）
- **解决方案**：烘焙时使用多方向正交渲染（front/side/top），运行时根据视角混合

**Marschner 的实用简化**

完整 Marschner 计算量很大（三个高斯卷积 + 方位角积分）。移动端的实用简化：
- 只算 R 分量（主高光），用 Phong-like 近似
- TT 分量用一个常量背光代替
- TRT 分量用噪声纹理扰动高光位置
- 这样 GPU 指令数从 ~200 降到 ~50

### 🎯 能力体检清单

| 检查项 | 如果你答不上来… |
|--------|----------------|
| 能解释 Marschner 模型中 R/TT/TRT 三条路径的物理意义？ | → 头发光照理论基础不足 |
| 知道 Strand Hair 的 GPU 性能瓶颈在哪？ | → GPU 渲染管线理解不足 |
| 能说明 Hair Card 的 Alpha Test 和 Alpha Blend 各自的优劣？ | → 半透明渲染机制不熟 |
| 知道如何从 Strand 烘焙 Card 纹理？ | → DCC 工具链经验不足 |
| 能设计跨平台头发品质分级方案？ | → TA 架构决策能力不足 |
| 理解头发自阴影为什么需要 Deep Shadow Map？ | → 阴影系统知识有盲区 |
| 知道 WBOIT 的原理？ | → OIT 技术不熟 |

### 🔗 相关问题

- [各向异性头发高光 Shader：Marschner 的工程实现](shader/hair-anisotropic-lighting)
- [角色材质规范：从 Substance 到引擎的完整工作流](technical-art/character-material-spec-workflow)
- [GPU 带宽优化：移动端渲染的带宽管理](optimization/gpu-bandwidth-optimization)
