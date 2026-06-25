---
title: "角色传送门漩涡扭曲消失效果怎么做？"
category: "shader"
level: 3
tags: ["传送门", "漩涡扭曲", "UV动画", "噪声", "GrabPass", " dissolve", "URP"]
hint: "核心是极坐标变换 + 噪声扰动 + 透明度裁剪三层叠加，配合屏幕扭曲实现空间撕裂感"
related: ["shader/dissolve-effect.md", "shader/hologram-projection-effect.md", "rendering/depth-based-screen-distortion.md"]
---

## 参考答案

### 🎬 场景描述

面试官给你看了一段演示视频——角色站在原地，脚下展开一个蓝紫色漩涡传送门，角色的身体从脚开始被"吸入"漩涡，身体逐渐扭曲、拉伸、碎裂成粒子消散。然后说：

> "我们要做角色传送入场/离场效果。玩家点传送后，角色脚下出现一个旋转漩涡，角色身体像被吸入一样扭曲变形然后消失。你来写这个 Shader。URP 移动端，效果要炫但不能卡。"

### ✅ 核心要点

1. **三层效果合成**：漩涡传送门地面盘（地面 Decal/Quad）+ 角色身体扭曲消散（角色材质）+ 屏幕空间扭曲后处理（可选增强）
2. **极坐标变换是漩涡的核心**：笛卡尔 → 极坐标后，对角度施加时间旋转，对半径施加径向收缩，形成螺旋吸入感
3. **噪声扰动制造撕裂感**：用 Perlin/Simplex 噪声对 UV 做偏移，强度随时间递增，让角色边缘开始抖动扭曲
4. **消散遮罩从下到上**：用世界坐标 Y 值或顶点色控制消散进度，配合噪声做不规则边缘，避免直线裁剪
5. **性能控制**：移动端角色材质增加 2 次噪声采样 + 极坐标计算（约 15 ALU），不做 GrabPass（太贵），用顶点动画 + Alpha Clip 替代屏幕扭曲

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色被吸入脚下漩涡，扭曲→碎裂→消失
     ↓ 倒推
角色消散 = 不规则遮罩从脚到身裁剪 + 边缘粒子化
     ↓ 倒推
不规则遮罩 = 噪声纹理 × 高度梯度 × 全局进度参数
     ↓ 倒推
扭曲变形 = UV偏移（噪声驱动）+ 顶点位置向漩涡中心收缩
     ↓ 倒推
漩涡地面盘 = 极坐标旋转 UV + 同心圆渐变 + 噪声边缘溶解
     ↓ 倒推
极坐标变换 = atan2(UV.y, UV.x) 得到角度, length(UV) 得到半径
     ↓ 倒推
统一控制参数：_Progress (0→1) 全局进度，驱动所有阶段
```

#### 知识点拆解（倒推树）

```
传送门漩涡消失效果
├── 漩涡地面盘（独立 Mesh/Decal）
│   ├── 极坐标 UV 变换（atan2 + length）
│   ├── 角度旋转动画（角度 += time × speed）
│   ├── 螺旋纹理合成（同心圆 × 噪声 × 螺旋角度）
│   ├── 边缘溶解（噪声 × progress → Alpha Clip）
│   └── 发光颜色叠加（ fresnel 或中心高亮）
│
├── 角色扭曲消散（角色材质变体）
│   ├── 消散遮罩生成
│   │   ├── 世界空间高度梯度（_WorldSpacePos.y 归一化）
│   │   ├── 噪声纹理采样（Simplex/Perlin）
│   │   └── 遮罩 = smoothstep(梯度 - 噪声 × 范围, progress)
│   ├── 顶点位置收缩
│   │   ├── 向漩涡中心方向偏移（_PortalCenter - _WorldPos）
│   │   ├── 收缩强度 = (1 - 遮罩) × _SuckStrength
│   │   └── Y 方向压缩（被吸入地面感）
│   ├── UV 扭曲（Fragment 阶段）
│   │   ├── 噪声 UV 偏移（采样两次噪声做方向偏移）
│   │   └── 偏移强度随消散进度增大
│   └── Alpha Clip + 边缘发光
│       ├── clip(mask - threshold)
│       └── 边缘 fresnel 发光（消散边缘的能量光圈）
│
├── 屏幕空间扭曲（可选，PC/主机端增强）
│   ├── GrabPass / Opaque Color Texture 采样
│   ├── 以漩涡中心为极点的径向扭曲
│   └── 移动端跳过（用角色顶点动画替代）
│
└── 粒子辅助（漏斗状光粒子向中心汇聚）
    └── GPU Instancing 粒子，沿螺旋路径运动
```

#### 代码实现

**1. 漩涡地面盘 Shader（URP Shader Graph / HLSL）**

```hlsl
// 漩涡地面盘 - Fragment 核心逻辑
// 输入：UV 已以中心为原点归一化到 [-1, 1]

float2 centerUV = i.uv - 0.5;
float angle = atan2(centerUV.y, centerUV.x);  // 极坐标角度
float radius = length(centerUV);               // 极坐标半径

// 螺旋旋转：角度随时间旋转，旋转速度随半径变化（外快内慢 or 内快外慢）
float spiralAngle = angle + _Time.y * _RotateSpeed * (1.0 - radius * 0.5);

// 螺旋纹理：多个同心圆 × 角度条纹 × 噪声
float stripes = sin(spiralAngle * _Arms + radius * _RingFreq * 6.2831);
stripes = stripes * 0.5 + 0.5;

// 噪声扰动
float noise = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, 
    float2(spiralAngle * 0.1, radius * 3.0 + _Time.y * 0.3)).r;

// 漩涡亮度 = 条纹 × 噪声 × 径向衰减（中心亮边缘暗）
float swirl = stripes * noise * smoothstep(0.5, 0.0, radius);

// 颜色混合：中心高温色 → 外围冷色
float3 color = lerp(_EdgeColor, _CoreColor, smoothstep(0.5, 0.0, radius));
color *= swirl * _Intensity;

// Alpha：径向遮罩 × 进度
float alpha = smoothstep(0.5, 0.45, radius) * _Progress;

return float4(color, alpha);
```

**2. 角色消散扭曲 Shader 核心**

```hlsl
// 角色消散 - Vertex 阶段
float3 worldPos = TransformObjectToWorld(i.positionOS);

// 高度遮罩：从脚(0)到头(1)，_Progress 从0开始向上消散
float heightGradient = saturate((worldPos.y - _CharacterMinY) / _CharacterHeight);
float noise = SAMPLE_TEXTURE2D_LOD(_NoiseTex, sampler_NoiseTex, 
    worldPos.xz * _NoiseScale, 0).r;

// 消散遮罩：噪声让边缘不规则
float mask = smoothstep(heightGradient + noise * _NoiseAmount, 
                         heightGradient + noise * _NoiseAmount - _EdgeWidth, 
                         _Progress);

// 顶点向漩涡中心收缩
float3 toCenter = _PortalCenter - worldPos;
float suckStrength = (1.0 - mask) * _SuckForce;
worldPos.xz += toCenter.xz * suckStrength;
worldPos.y -= suckStrength * _VerticalCompress;  // 向下压

i.positionWS = worldPos;
i.positionCS = TransformWorldToHClip(worldPos);

// ------- Fragment 阶段 -------
// UV 扭曲：噪声偏移采样
float2 distortUV = i.uv + float2(
    noise * _DistortStrength * (1.0 - mask),
    noise * _DistortStrength * 0.5 * (1.0 - mask)
);

// 采样主贴图（用扭曲后的 UV）
float4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, distortUV);

// Alpha Clip 消散
clip(mask - 0.01);

// 边缘发光（消散边界能量光圈）
float edgeGlow = smoothstep(_EdgeWidth, 0.0, abs(mask - 0.5)) * _EdgeGlowIntensity;
float3 finalColor = baseColor.rgb + _EdgeGlowColor * edgeGlow;

return float4(finalColor, baseColor.a);
```

**3. 全局进度驱动（C# 脚本）**

```csharp
using UnityEngine;
using System.Collections;

public class PortalEffectController : MonoBehaviour
{
    public Material characterMat;
    public Material swirlMat;
    public float duration = 2.5f;
    
    public IEnumerator PlayTeleportOut()
    {
        MaterialPropertyBlock mpb = new MaterialPropertyBlock();
        float elapsed = 0f;
        
        // 阶段1: 漩涡展开 (0 - 0.3)
        while (elapsed < duration * 0.3f)
        {
            float t = elapsed / (duration * 0.3f);
            swirlMat.SetFloat("_Progress", t);
            yield return null;
            elapsed += Time.deltaTime;
        }
        
        // 阶段2: 角色消散 (0.3 - 1.0)
        float charElapsed = 0f;
        float charDuration = duration * 0.7f;
        while (charElapsed < charDuration)
        {
            float t = charElapsed / charDuration;
            characterMat.SetFloat("_Progress", t);
            swirlMat.SetFloat("_Progress", 1f);
            yield return null;
            charElapsed += Time.deltaTime;
        }
        
        // 阶段3: 漩涡收拢
        float collapseElapsed = 0f;
        float collapseDuration = 0.5f;
        while (collapseElapsed < collapseDuration)
        {
            float t = 1f - collapseElapsed / collapseDuration;
            swirlMat.SetFloat("_Progress", t);
            yield return null;
            collapseElapsed += Time.deltaTime;
        }
    }
}
```

### ⚡ 实战经验

**做过这个效果后踩的坑：**

1. **顶点收缩导致法线错乱**——角色被拉向中心时法线没更新，光照突变。解决方案：在 Vertex 阶段对法线也做同样的变换，或者消散阶段关闭法线光照只保留自发光/Unlit
2. **Alpha Clip 边缘锯齿**——直接 clip 太硬。加 0.5px 的 Alpha 渐变，或用 `smoothstep` 做边缘柔化。移动端 MSAA 可能没开，锯齿会更明显
3. **多角色同时传送的性能尖峰**——10 个角色同时触发消散，顶点动画 + 噪声采样叠在一起。加随机延迟（0.1-0.3s 错峰），避免同一帧全部开始
4. **漩涡地面盘 Z-Fighting**——Decal 贴地面上和地形穿插。加 Polygon Offset 或稍微抬高 0.01m
5. **消散后角色 Collider 还在**——别忘了和逻辑层同步，消散开始就关掉 Collider 和阴影投射
6. **噪声纹理选择**——Simplex 噪声效果最好但移动端计算贵，建议预烘焙到 RGBA 通道（R=低频，G=中频，B=高频，A=细节噪声），一次采样取多频段

### 🎯 能力体检清单

| 检查项 | 如果答不上来... |
|--------|----------------|
| 极坐标变换的 atan2 和 length 怎么用？ | UV 变换基础有盲区 |
| 顶点向中心收缩时法线怎么处理？ | 顶点动画 × 光照交互不理解 |
| 消散遮罩如何做到不规则边缘？ | 噪声 × smoothstep 组合运用不熟 |
| 移动端为什么不用 GrabPass？ | 移动端 GPU 带宽特性不了解 |
| 多个角色同时触发的性能怎么控制？ | 批量特效性能规划意识不足 |
| 漩涡螺旋臂数量和旋转速度怎么调出好看的效果？ | Shader 调参审美 + 数学直觉不够 |

### 🔗 相关问题

- [角色溶解消失效果](shader/dissolve-effect.md) — 消散类效果的基础
- [全息投影效果](shader/hologram-projection-effect.md) — 另一种角色视觉变化
- [屏幕空间深度扭曲](rendering/depth-based-screen-distortion.md) — 屏幕级扭曲方案
- [URP 自定义 Renderer Feature](rendering/urp-renderer-feature.md) — 后处理扭曲需要自定义 Pass
