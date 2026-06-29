---
title: "换装系统材质混乱：如何用 Material ID Map 控制角色不同部位的材质替换？"
category: "technical-art"
level: 3
tags: ["Material ID Map", "换装系统", "材质规范", "Mask贴图", "PBR", "工作流"]
hint: "一个好用的换装系统不是一个部位一个材质球——而是用 ID Mask 在一张贴图上分区控制，用一个 Shader 搞定全身"
related: ["technical-art/character-material-spec-workflow", "technical-art/character-outfit-swap-skeleton-sharing", "technical-art/gpu-instancing-outfit-swap"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款二次元开放世界游戏，主角有换装系统——头盔、上衣、裤子、鞋子、手套各 20+ 套。美术现在每个部位一个材质球，全身 5 个材质球 × Draw Call，加上其他角色同屏 10 人就是 50+ Draw Call。而且换装时需要实时切换材质，导致卡顿。你作为 TA 怎么重新设计材质方案？」

这是叠纸、米哈游、鹰角等做角色定制系统的公司的高频面试题。考察的不是写 Shader——而是材质系统架构设计能力。

### ✅ 核心要点

1. **Material ID Map**：一张 Mask 贴图，用不同颜色通道标记角色的不同部位区域
2. **单 Draw Call 目标**：全身合并为一个材质球 + 一个 Shader，通过 ID Mask 在 Shader 内区分部位
3. **部位颜色/参数替换**：用 Uniform 数组或 LUT 贴图传入不同部位的 PBR 参数
4. **贴图合并策略**：Albedo / Normal / ORM 各一张全身合并贴图 + 一张 ID Mask
5. **换装性能优化**：换装时只更新参数（MaterialPropertyBlock），不替换材质实例

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
目标：换装系统，全身 1 个 Draw Call，支持实时切换 5 个部位的外观
     ↓
Step 1：分析现有问题
  ├── 5 个材质球 = 5 个 Draw Call（无 SRP Batcher 兼容问题但增加了批次）
  ├── 换装时 SetMaterial 运行时分配新 Material Instance → GC + 卡顿
  └── 不同部位材质参数格式不统一 → 维护噩梦
     ↓
Step 2：Material ID Map 方案
  ├── 烘焙一张 ID Mask 贴图（RGBA 各通道标记不同部位）
  │   ├── R = 头部/头盔
  │   ├── G = 上衣
  │   ├── B = 裤子
  │   ├── A = 鞋子+手套（或用更高位区分）
  │   └── 多通道扩展：ID 1-255 用完整 RGBA 编码
  ├── 所有部位合并到一张 Albedo / Normal / ORM 贴图集上
  └── 一个 Shader 读取 ID Mask + 部位参数表 → 输出正确材质
     ↓
Step 3：参数存储方案选型
  ├── 方案A：Uniform 数组（float _PartColors[20][5]）
  │   └── 优点：直接，缺点：数量受 Shader 常量限制（通常 ≤ 256 floats）
  ├── 方案B：LUT 贴图（1D 或 2D Lookup Texture）
  │   └── 每个部位一行像素存 PBR 参数（Albedo.rgb + Metallic + Roughness + Emission）
  │   └── 优点：无数量限制，缺点：需要采样
  └── 方案C：Texture2DArray（每件装备一层）
      └── 优点：支持完整贴图集，缺点：创建/更新成本高
     ↓
Step 4：换装时数据更新
  ├── CPU 侧：更新 MaterialPropertyBlock 或更新 LUT 贴图的某个像素
  ├── GPU 侧：Shader 自动用新参数渲染
  └── 无需实例化新材质，无 GC，无 Draw Call 变化
```

#### 知识点拆解（倒推树）

```
Material ID Map 换装系统
├── ID Mask 贴图设计
│   ├── 通道分配策略
│   │   ├── 简单方案：RGBA = 4 个部位（够用于 头/衣/裤/鞋）
│   │   ├── 编码方案：RGBA8 编码 0-255 个 ID（支持复杂部位细分）
│   │   └── 多级方案：R 通道 = 大类，G 通道 = 子类
│   ├── 烘焙方式
│   │   ├── Substance Painter 中用 Color ID 填充
│   │   ├── Maya/Blender 中按材质 ID 烘焙
│   │   └── Houdini 程序化生成（批量角色场景）
│   └── 精度要求
│       ├── 边缘抗锯齿（ID 边界需要 1-2 像素过渡）
│       └── 分辨率：通常和 Albedo 同分辨率或半分辨率
├── 贴图集合并
│   ├── UV 重映射
│   │   ├── 各部位 UV 打包到一张 UV 空间
│   │   ├── 保留 UV Padding 防止 Mipmap 溢出
│   │   └── 重复纹理（金属、皮革）用 Tiling + ID Mask 区域控制
│   ├── Mipmap 处理
│   │   ├── ID Mask 必须禁用 Mipmap（或限制到 2 级）
│   │   ├── 边界像素渗色问题（相邻区域 Mipmap 混合）
│   │   └── 方案：手动 Mipmap + 边缘像素外扩（Bleed）
│   └── 贴图预算
│       ├── 全身 Albedo：1024×1024 或 2048×2048
│       ├── 全身 Normal：1024×1024
│       ├── 全身 ORM：512×512 或 1024×1024
│       └── ID Mask：512×512（不需要高精度）
├── 参数存储方案
│   ├── LUT 贴图设计
│   │   ├── 布局：每行 1 个装备 ID，列 = PBR 参数
│   │   ├── 格式：RGBA32F 或 RGBA16F（精度需求）
│   │   │   R=Albedo.r, G=Albedo.g, B=Albedo.b, A=Metallic
│   │   ├── 第二行 LUT 存 Roughness + Emission + 其他
│   │   └── 尺寸：32×2 像素即可支持 32 套装备
│   ├── Texture2DArray 方案
│   │   ├── 每件装备 = 一层 Albedo + 一层 Normal
│   │   ├── 换装 = 切换采样的 array slice index
│   │   ├── 优点：支持完整贴图替换（不只是颜色参数）
│   │   └── 缺点：Texture2DArray 内存占用大
│   └── Hybrid 方案
│       ├── 基础外观（Albedo/Normal/ORM）用 Texture2DArray
│       ├── 颜色染色 / 参数微调用 LUT
│       └── 特效层（发光、流光）用 ID Mask 控制
├── Shader 实现
│   ├── ID Mask 采样 + 解码
│   │   ├── tex2D(_IDMask, uv).r → 头部区域权重
│   │   ├── 多通道混合：lerp(baseColor, helmetColor, mask.r)
│   │   └── 边界软化：smoothstep 处理 ID 边界
│   ├── LUT 参数采样
│   │   ├── 根据装备 Index 计算 LUT UV：float2(0.5/32, (index+0.5)/32)
│   │   ├── 采样 PBR 参数并应用到光照
│   │   └── 支持多 LUT 混合（如：手套 + 戒指叠加效果）
│   └── Texture2DArray 采样
│       ├── float3(uv.xy, layerIndex) → UnityTexture2DArray.Sample()
│       ├── Mipmap 支持：需要手动计算 lod 或使用 CalculateLevelOfDetail
│       └── 层间混合（如：半透明披风叠加底层衣服）
├── 换装逻辑
│   ├── C# 侧
│   │   ├── 维护 EquipSlot 数据结构（headID, bodyID, legID, footID, handID）
│   │   ├── 换装触发：EquipChanged → 更新 MaterialPropertyBlock
│   │   ├── LUT 更新：Texture2D.SetPixel + Apply(false)（不重建 Mipmap）
│   │   └── Texture2DArray 更新：需要整体重建（性能坑！）
│   └── 网络同步
│       ├── 只同步装备 ID 列表（几个 int）
│       ├── 接收方根据 ID 查本地 LUT/Texture2DArray
│       └── 无需传输贴图数据
└── 性能对比
    ├── 旧方案：5 材质球 × 10 角色 = 50 Draw Call
    ├── 新方案：1 材质球 × 10 角色 = 10 Draw Call（SRP Batcher 可合批）
    ├── 代价：Shader 复杂度增加（+10 ALU）
    └── 内存：+1 张 ID Mask (256KB) + LUT (几KB)
```

#### 代码实现

**核心 Shader（URP HLSL）：**

```hlsl
// CharacterOutfit.shader
Shader "Custom/CharacterOutfit"
{
    Properties
    {
        _BaseMap ("Albedo Atlas", 2D) = "white" {}
        _NormalMap ("Normal Atlas", 2D) = "bump" {}
        _ORMMap ("ORM Atlas (AO/Roughness/Metallic)", 2D) = "white" {}
        _IDMask ("Material ID Mask", 2D) = "black" {}

        // 装备 LUT（32 套装备 × 2 行参数）
        _EquipLUT ("Equipment Parameter LUT", 2D) = "white" {}
        _EquipCount ("Equipment Count", Int) = 32

        // 当前角色装备的 ID（CPU 设置）
        _HeadEquipID ("Head Equipment ID", Float) = 0
        _BodyEquipID ("Body Equipment ID", Float) = 1
        _LegEquipID  ("Leg Equipment ID", Float)  = 2
        _FootEquipID ("Foot Equipment ID", Float) = 3
        _HandEquipID ("Hand Equipment ID", Float) = 4
    }

    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

    TEXTURE2D(_BaseMap);     SAMPLER(sampler_BaseMap);
    TEXTURE2D(_NormalMap);   SAMPLER(sampler_NormalMap);
    TEXTURE2D(_ORMMap);      SAMPLER(sampler_ORMMap);
    TEXTURE2D(_IDMask);      SAMPLER(sampler_IDMask);
    TEXTURE2D(_EquipLUT);    SAMPLER(sampler_EquipLUT);

    float _HeadEquipID, _BodyEquipID, _LegEquipID, _FootEquipID, _HandEquipID;
    int _EquipCount;

    // 从 LUT 采样装备颜色参数
    half3 SampleEquipColor(float equipID)
    {
        float2 lutUV = float2(
            (equipID + 0.5) / (float)_EquipCount,
            0.5 / 2.0  // 第一行：Albedo + Metallic
        );
        return SAMPLE_TEXTURE2D(_EquipLUT, sampler_EquipLUT, lutUV).rgb;
    }

    // 从 LUT 采样装备 PBR 参数
    half2 SampleEquipParams(float equipID)
    {
        float2 lutUV = float2(
            (equipID + 0.5) / (float)_EquipCount,
            1.5 / 2.0  // 第二行：Roughness + Metallic
        );
        return SAMPLE_TEXTURE2D(_EquipLUT, sampler_EquipLUT, lutUV).rg;
    }

    struct Attributes
    {
        float4 positionOS : POSITION;
        float3 normalOS   : NORMAL;
        float4 tangentOS  : TANGENT;
        float2 uv         : TEXCOORD0;
    };

    struct Varyings
    {
        float4 positionCS : SV_POSITION;
        float3 positionWS : TEXCOORD0;
        float3 normalWS   : TEXCOORD1;
        float4 tangentWS  : TEXCOORD2;
        float2 uv         : TEXCOORD3;
    };

    Varyings Vert(Attributes input)
    {
        Varyings output;
        VertexPositionInputs posInputs = GetVertexPositionInputs(input.positionOS.xyz);
        VertexNormalInputs normInputs  = GetVertexNormalInputs(input.normalOS, input.tangentOS);

        output.positionCS = posInputs.positionCS;
        output.positionWS = posInputs.positionWS;
        output.normalWS   = normInputs.normalWS;
        output.tangentWS  = float4(normInputs.tangentWS, input.tangentOS.w);
        output.uv         = input.uv;
        return output;
    }

    half4 Frag(Varyings input) : SV_Target
    {
        // === 采样基础贴图 ===
        half3 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv).rgb;
        half3 normalTS  = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, input.uv));
        half3 orm       = SAMPLE_TEXTURE2D(_ORMMap, sampler_ORMMap, input.uv).rgb;
        half  ao        = orm.r;
        half  roughness = orm.g;
        half  metallic  = orm.b;

        // === 采样 ID Mask ===
        half4 idMask = SAMPLE_TEXTURE2D(_IDMask, sampler_IDMask, input.uv);

        // === 根据 ID Mask 混合不同部位的装备颜色 ===
        // 每个部位的装备颜色从 LUT 查询
        half3 headColor = SampleEquipColor(_HeadEquipID);
        half3 bodyColor = SampleEquipColor(_BodyEquipID);
        half3 legColor  = SampleEquipColor(_LegEquipID);
        half3 footColor = SampleEquipColor(_FootEquipID);
        half3 handColor = SampleEquipColor(_HandEquipID);

        // ID Mask 各通道加权混合
        half3 finalColor = baseColor;
        finalColor = lerp(finalColor, headColor, idMask.r);
        finalColor = lerp(finalColor, bodyColor, idMask.g);
        finalColor = lerp(finalColor, legColor,  idMask.b);
        finalColor = lerp(finalColor, footColor, idMask.a);

        // 手套（如果 idMask.a 和 footColor 重叠，用额外的 UV2 通道或编码方案）
        // 这里简化：手套和鞋子共用 A 通道，按 UV 区域区分

        // === 装备参数（Roughness/Metallic）也根据部位混合 ===
        half2 headParam = SampleEquipParams(_HeadEquipID);
        half2 bodyParam = SampleEquipParams(_BodyEquipID);
        half  finalRough = roughness;
        half  finalMetal = metallic;

        finalRough = lerp(finalRough, headParam.r, idMask.r);
        finalRough = lerp(finalRough, bodyParam.r, idMask.g);
        finalMetal = lerp(finalMetal, headParam.g, idMask.r);
        finalMetal = lerp(finalMetal, bodyParam.g, idMask.g);

        // === 标准 URP 光照 ===
        InputData inputData = (InputData)0;
        inputData.positionWS = input.positionWS;
        inputData.normalWS   = NormalizeNormalPerPixel(input.normalWS);
        inputData.viewDirWS  = GetWorldSpaceNormalizeViewDir(input.positionWS);

        SurfaceData surfaceData = (SurfaceData)0;
        surfaceData.albedo     = finalColor;
        surfaceData.metallic   = finalMetal;
        surfaceData.smoothness = 1.0 - finalRough;
        surfaceData.occlusion  = ao;
        surfaceData.normalTS   = normalTS;

        half3 color = UniversalFragmentPBR(inputData, surfaceData);
        return half4(color, 1.0);
    }
    ENDHLSL

    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }
        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS
            #pragma multi_compile _ _ADDITIONAL_LIGHTS
            ENDHLSL
        }
    }
}
```

**换装管理（C# 侧）：**

```csharp
using UnityEngine;

[System.Serializable]
public struct EquipSlot
{
    public int headID;
    public int bodyID;
    public int legID;
    public int footID;
    public int handID;
}

[RequireComponent(typeof(SkinnedMeshRenderer))]
public class CharacterOutfitController : MonoBehaviour
{
    [Header("当前装备")]
    public EquipSlot currentEquip;

    [Header("装备数据库")]
    public Texture2D equipLUT;        // LUT 贴图
    public int equipCount = 32;

    private SkinnedMeshRenderer _smr;
    private MaterialPropertyBlock _mpb;
    private static readonly int HeadID = Shader.PropertyToID("_HeadEquipID");
    private static readonly int BodyID = Shader.PropertyToID("_BodyEquipID");
    private static readonly int LegID  = Shader.PropertyToID("_LegEquipID");
    private static readonly int FootID = Shader.PropertyToID("_FootEquipID");
    private static readonly int HandID = Shader.PropertyToID("_HandEquipID");

    void Awake()
    {
        _smr = GetComponent<SkinnedMeshRenderer>();
        _mpb = new MaterialPropertyBlock();
        ApplyEquip();
    }

    /// <summary>
    /// 换装入口：更新装备 ID 并刷新材质参数
    /// </summary>
    public void ChangeEquipment(EquipPart part, int equipID)
    {
        switch (part)
        {
            case EquipPart.Head: currentEquip.headID = equipID; break;
            case EquipPart.Body: currentEquip.bodyID = equipID; break;
            case EquipPart.Leg:  currentEquip.legID  = equipID; break;
            case EquipPart.Foot: currentEquip.footID = equipID; break;
            case EquipPart.Hand: currentEquip.handID = equipID; break;
        }
        ApplyEquip();
    }

    void ApplyEquip()
    {
        _smr.GetPropertyBlock(_mpb);
        _mpb.SetFloat(HeadID, currentEquip.headID);
        _mpb.SetFloat(BodyID, currentEquip.bodyID);
        _mpb.SetFloat(LegID,  currentEquip.legID);
        _mpb.SetFloat(FootID, currentEquip.footID);
        _mpb.SetFloat(HandID, currentEquip.handID);
        _smr.SetPropertyBlock(_mpb);
    }

    /// <summary>
    /// 批量更新 LUT（如：玩家染色系统）
    /// </summary>
    public void UpdateEquipLUT(int equipID, Color albedo, float metallic, float roughness)
    {
        // 更新 LUT 贴图的指定行
        int x = equipID;
        equipLUT.SetPixel(x, 0, new Color(albedo.r, albedo.g, albedo.b, metallic));
        equipLUT.SetPixel(x, 1, new Color(roughness, 0, 0, 0));
        equipLUT.Apply(false); // 不重建 Mipmap
    }
}

public enum EquipPart { Head, Body, Leg, Foot, Hand }
```

**ID Mask 烘焙工具（Substance Painter → 导出）：**

```
# Substance Painter ID Mask 工作流
1. 在 SP 中用 Fill Layer 为每个部位填充纯色
   - 头部：R=255, G=0,   B=0,   A=255
   - 上衣：R=0,   G=255, B=0,   A=255
   - 裤子：R=0,   G=0,   B=255, A=255
   - 鞋子：R=0,   G=0,   B=0,   A=255

2. 导出 ID Map：
   - 输出模板：Custom → RGBA
   - 命名：character_base_IDMask.png
   - 格式：PNG 8bit（不需要 16bit 精度）

3. 边缘处理：
   - SP 中设置 Padding：Dilation=4px
   - 防止 UV 接缝处 ID 渗色
```

### ⚡ 实战经验

1. **ID Mask 是万能工具**：不只是换装——伤口贴花（红色区域受伤）、雪覆盖（白色区域积雪）、泥渍（褐色区域脏污）都可以用同一张 ID Mask 控制区域
2. **LUT 贴图比 Uniform 数组灵活**：Shader 中 `float _Colors[100]` 在不同平台兼容性差（GLSL/Vulkan/HLSL 数组处理不同），LUT 贴图跨平台无差异
3. **Mipmap 是 ID Mask 的天敌**：如果 ID Mask 开了 Mipmap，边界处两个区域的颜色会混合，导致装备颜色渗透。方案：ID Mask 禁用 Mipmap，或者用 `tex2Dlod` 强制 Lod=0
4. **Texture2DArray 适用于换贴图（不只是换颜色）**：如果换装不只是变色而是换整套纹理（从布衣变金属铠甲），需要 Texture2DArray 切换 Albedo/Normal 层
5. **与 GPU Instancing 配合**：用 `MaterialPropertyBlock` 设置装备 ID 后，同屏 10 个角色用同一 Shader 可以 SRP Batcher 合批，10 个角色 = 1 个 Draw Call

### 🎯 能力体检清单

| 检查项 | 如果答不上来… |
|--------|-------------|
| 能解释为什么用 ID Mask 而不是多材质球 | → 渲染优化基础盲区 |
| 知道 LUT 贴图存 PBR 参数的原理 | → Shader 数据存储盲区 |
| 能处理 ID Mask 的 Mipmap 边界渗色 | → 贴图管线盲区 |
| 理解 MaterialPropertyBlock 为什么不破坏合批 | → Unity 渲染机制盲区 |
| 能设计换装系统的数据流（CPU→GPU 参数传递） | → 系统架构设计盲区 |

### 🔗 相关问题

- [technical-art/character-material-spec-workflow](../technical-art/character-material-spec-workflow.md) — 角色材质规范制定
- [technical-art/character-outfit-swap-skeleton-sharing](../technical-art/character-outfit-swap-skeleton-sharing.md) — 换装系统骨骼共享与材质合批
- [technical-art/gpu-instancing-outfit-swap](../technical-art/gpu-instancing-outfit-swap.md) — GPU Instancing 换装方案
