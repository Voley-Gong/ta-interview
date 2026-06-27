---
title: "角色边缘光（Rim Light）：如何用菲涅尔效应做出高级感的轮廓发光？"
category: "shader"
level: 2
tags: ["Fresnel", "RimLight", "NPR", "URP", "卡通渲染"]
hint: "核心是 N·V 点积——视线方向与法线越垂直的边缘越亮，但纯数学菲涅尔不够，需要美术可控的遮罩与色调映射"
related: ["shader/npr-outline-cartoon", "shader/hit-flash-damage-blink", "rendering/cel-shading-toon-pipeline"]
---

## 参考答案

### 🎬 圇景描述

面试官说：「我们的角色在暗场景里辨识度太低，美术想要一种边缘光（Rim Light）效果——角色轮廓有一圈柔和的发光，颜色可以自定义，要能跟场景雾效和后处理兼容。URP Shader Graph 或 HLSL 都行，给我完整方案。」

### ✅ 核心要点

1. **菲涅尔核心公式**：`rim = pow(1 - dot(N, V), power)` — 法线越垂直于视线越亮
2. **美术可控参数**：强度、幂次（边缘软硬）、颜色、遮罩贴图
3. **不能只靠数学**：纯菲涅尔在背光面会全亮，需要光源方向修正或遮罩控制
4. **NPR 风格化**：卡通渲染中常做阶梯化（step / smoothstep）而非连续渐变
5. **后处理兼容**：Rim Light 贡献到 Emission 通道才能被 Bloom 拾取

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色轮廓有一圈可控颜色的柔和发光，暗面也有辨识度
                ↑
倒推1：需要「边缘亮、中间暗」→ Fresnel = 1 - dot(N, V)
倒推2：需要「边缘柔和」→ pow(fresnel, _RimPower) 控制衰减曲线
倒推3：纯菲涅尔在背光面全亮 → 引入主光源方向修正 rim *= saturate(dot(N, L))
倒推4：需要「美术可控区域」→ Mask 贴图（R通道控制金属不发rim，皮肤多发rim）
倒推5：需要「发光感」→ rim * _RimColor 贡献到 Emission，让 Bloom 拾取
倒推6：卡通风格 → smoothstep 做硬边阶梯，不要平滑渐变
```

#### 知识点拆解（倒推树）

```
角色边缘光（Rim Light）
├── 数学基础
│   ├── 点积几何含义（dot(N,V) = cosθ，θ为法线与视线的夹角）
│   ├── 菲涅尔效应物理原理（Schlick 近似 vs 简化版 1-N·V）
│   └── pow() 对衰减曲线的塑形作用（幂次越高边缘越窄越锐）
├── 光照修正
│   ├── 方向光修正：rim *= saturate(dot(N, L)) — 只在受光面发rim
│   ├── 半球光修正：rim *= 0.5 + 0.5 * dot(N, _UpVector) — 天空环境光方向
│   └── 无修正全边缘光（适合科幻/特效场景，不适合写实PBR）
├── 美术控制
│   ├── Rim Mask 贴图（按材质区域控制rim强度：皮肤>衣服>金属）
│   ├── 颜色渐变（顶部冷色/底部暖色 vs 单色）
│   └── 阶梯化（NPR）：smoothstep(_RimThreshold, _RimThreshold + _RimSoftness, rim)
├── URP 集成
│   ├── Shader Graph：Fresnel Node → Power → Multiply(Color) → Add to Emission
│   ├── HLSL：Custom Function 节点或直接写在 fragment 中
│   ├── SRP Batcher 兼容：所有参数放进 CBUFFER
│   └── 多 Pass 方案（Outline Pass 共享法线数据的注意事项）
├── 后处理配合
│   ├── Bloom 拾取：rim 写入 emission/austomaticHDR
│   ├── Tonemapping 后颜色偏移（ACES 下蓝色rim会偏青）
│   └── 雾效兼容（距离衰减：rim *= exp2(-distance * fogDensity)）
```

#### 代码实现

**HLSL Fragment 核心片段（URP 兼容）：**

```hlsl
// === Rim Light 核心 ===
// 法线与视线方向的菲涅尔项
float3 viewDir = normalize(_WorldSpaceCameraPos - positionWS);
float NdotV = saturate(dot(normalWS, viewDir));

// 基础菲涅尔：边缘为1，正面为0
float fresnel = 1.0 - NdotV;
fresnel = pow(fresnel, _RimPower); // _RimPower 控制边缘软硬

// 方向光修正（可选）：只在受光面发rim
float NdotL = saturate(dot(normalWS, mainLightDir));
fresnel *= lerp(1.0, NdotL, _RimLightContribution); // 0=全方向rim, 1=只在受光面

// 美术遮罩
float rimMask = SAMPLE_TEXTURE2D(_RimMask, sampler_RimMask, uv).r;
fresnel *= rimMask;

// 阶梯化（NPR卡通风格用）
float rimStep = smoothstep(_RimThreshold, _RimThreshold + _RimSoftness, fresnel);

// 最终颜色贡献
float3 rimColor = _RimColor * rimStep * _RimIntensity;

// 输出：加到最终颜色上，同时写入 emission 供 Bloom 拾取
color.rgb += rimColor;
emission += rimColor * _RimEmissionWeight;
```

**Shader Graph 连接方式：**
```
[Fresnel Node] → (Power = _RimPower) → [Multiply] → [Smoothstep] → [Multiply(_RimColor)] → [Multiply(_RimIntensity)]
                     ↑                    ↑
              [MainLight Direction]   [Rim Mask Texture]
              [Dot N·L → Saturate]
```

### ⚡ 实战经验

- **别只用 `1 - dot(N,V)`**：在角色下巴、腋下等法线剧变区域会产生不自然的亮带。加一张 AO 贴图乘上去，或用 normal map 的细节法线做修正
- **移动端省算力**：把 `pow()` 换成查找表（LUT），或直接用 `fresnel * fresnel` 近似 `pow(fresnel, 2)`
- **多角色场景 rim 颜色区分**：友方暖色（橙黄）、敌方冷色（紫红）、NPC 中性色（白），玩家一眼区分阵营
- **跟 Outline 的冲突**：描边渲染的 Back-face Outline 会盖住 rim light。方案：Outline 用同一颜色，或者 Outline 做成 transparent 让 rim 透出来
- **过场动画特写**：rim 强度要单独控，远距离弱一些近景强一些，可以用 LOD 或 camera distance 做插值

### 🎯 能力体检清单

- [ ] 能否解释为什么 `1 - dot(N,V)` 在球体边缘等于1而正面等于0？（点积几何含义）
- [ ] `_RimPower` 从 1 变到 8，边缘光视觉上会怎么变化？（幂次对曲线的塑形）
- [ ] 如果角色在背光面不想要 rim 光，怎么修改？（方向光修正项）
- [ ] rim light 写到 emission 和直接加到 finalColor 有什么区别？（Bloom 后处理链路）
- [ ] 在 Shader Graph 里 Fresnel Effect 节点的原理是什么？能否手写 HLSL 替代？（节点底层理解）
- [ ] 移动端 rim light 性能开销在哪里？如何优化？（pow、normalize、纹理采样）
- [ ] NPR 卡通渲染的 rim 为什么要阶梯化？连续渐变有什么问题？（风格一致性）

### 🔗 相关问题

- [卡通渲染 Outline 方案](shader/npr-outline-cartoon) — 描边与 rim light 的配合
- [受击闪白特效](shader/hit-flash-damage-blink) — 同样基于菲涅尔的伤害反馈
- [Cel Shading 卡通管线](rendering/cel-shading-toon-pipeline) — 整体卡通渲染框架中 rim 的定位
