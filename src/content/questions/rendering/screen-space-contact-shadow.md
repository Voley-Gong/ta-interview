---
title: "角色脚底硬阴影丢失？手写屏幕空间接触阴影（Contact Shadow）"
category: rendering
level: 3
tags: ["接触阴影", "Contact Shadow", "屏幕空间光线步进", "URP", "Shadow"]
hint: "用屏幕空间 Ray Marching 模拟近距离硬阴影，弥补传统 Shadow Map 在短距离的精度不足"
related: ["character-ground-contact-soft-shadow", "custom-screen-space-shadow-soften", "hiz-screen-space-reflection"]
---

## 参考答案

### 🎬 场景描述

面试官说："我们项目中角色脚底的阴影总是很飘——引擎自带 Shadow Map 因为精度问题，近处阴影边缘糊成一团，美术不满意要求'脚要踩实'。你能不能在不增加 Shadow Map 分辨率的前提下，解决这个问题？"

### ✅ 核心要点

- **问题本质**：Directional Light Shadow Map 的精度分布不均匀，近处像素覆盖面积大导致阴影模糊
- **解决方向**：屏幕空间 Contact Shadow——在 Shadow Map 之外叠加一层近距离高精度阴影
- **技术手段**：沿光照方向在屏幕空间做 Ray Marching，采样深度缓冲判断遮挡
- **性能关键**：步数控制（通常 8-16 步即可）、最大距离限制、半分辨率渲染

### 📖 深度展开

#### 解决思路（从效果倒推实现）

**最终效果**：角色脚下出现一圈锐利的、随距离衰减的接触阴影，与远处 Shadow Map 的柔和阴影自然过渡。

**倒推链**：
1. 要锐利阴影 → 需要「某点是否被光遮挡」的精确判断
2. 精确判断 → 从该像素沿光照方向步进，检查是否碰到几何体
3. 在屏幕空间操作 → 只需要深度图（Camera Depth Texture），不需要额外的 Shadow Map
4. 性能可控 → 限制步进距离（比如 0.5 米内）、减少步数、用半分辨率

#### 知识点拆解（倒推树）

```
屏幕空间接触阴影
├── 1. 光照方向投影到屏幕空间
│   ├── View Space 光照方向 → 屏幕 UV 偏移量
│   └── 透视投影下的非线性深度修正
├── 2. Ray Marching 核心循环
│   ├── 沿光照方向逐步步进
│   ├── 每步采样 Camera Depth Texture
│   ├── 深度差判断：当前步进深度 vs 采样深度
│   └── 记录最小遮挡比例（visibility）
├── 3. 遮挡计算与柔化
│   ├── 硬遮挡 → visibility = 0（完全遮挡）
│   ├── 半影区 → 基于深度差做平滑过渡
│   └── 距离衰减 → 近距离最强，远处渐变到 0
├── 4. 与引擎 Shadow Map 混合
│   ├── Contact Shadow 只在近距离生效（线性衰减）
│   └── 最终阴影 = max(ShadowMap, ContactShadow)
└── 5. URP Renderer Feature 集成
    ├── Render Pass 注入时机（After Rendering Opaques）
    ├── Blit 到半分辨率 RT
    └── Composite 回主画面
```

#### 代码实现

**HLSL 核心函数（屏幕空间 Ray Marching Contact Shadow）**：

```hlsl
// screen_uv: 当前像素 UV
// view_dir_ws: 视线方向（世界空间）
// light_dir_ws: 平行光方向（世界空间）
// depth_tex: Camera Depth Texture
float ContactShadow(float2 screen_uv, float3 view_dir_ws, float3 light_dir_ws,
                    Texture2D depth_tex, SamplerState point_clamp_sampler)
{
    // 参数
    const int STEP_COUNT = 12;
    const float MAX_DISTANCE = 0.5; // 接触阴影最大作用距离（世界空间米）
    const float THICKNESS = 0.02;   // 厚度容差，避免自阴影

    // 重建世界坐标
    float raw_depth = depth_tex.Sample(point_clamp_sampler, screen_uv).r;
    #if UNITY_REVERSED_Z
        raw_depth = 1.0 - raw_depth;
    #endif
    float3 world_pos = ComputeWorldPos(screen_uv, raw_depth);

    // 将光照方向投影到屏幕空间
    // 采样的邻居位置 = 当前位置 + 光照方向偏移
    // 但屏幕空间需要转成 UV 偏移
    float contact_shadow = 1.0; // 1 = 无遮挡

    float3 ray_pos = world_pos;
    float step_size = MAX_DISTANCE / STEP_COUNT;

    for (int i = 0; i < STEP_COUNT; i++)
    {
        // 沿光照方向步进（注意方向：从像素朝光源方向）
        ray_pos -= light_dir_ws * step_size;

        // 将步进位置投影到屏幕 UV
        float4 clip_pos = mul(UNITY_MATRIX_VP, float4(ray_pos, 1.0));
        float2 sample_uv = clip_pos.xy / clip_pos.w;
        sample_uv = sample_uv * 0.5 + 0.5;

        // 越界检查
        if (any(sample_uv < 0) || any(sample_uv > 1))
            break;

        // 采样深度并重建该位置的世界 Z
        float sample_depth = depth_tex.Sample(point_clamp_sampler, sample_uv).r;
        #if UNITY_REVERSED_Z
            sample_depth = 1.0 - sample_depth;
        #endif
        float3 sample_world_pos = ComputeWorldPos(sample_uv, sample_depth);

        // 深度比较：如果采样到的几何体比步进位置更近（挡住了光）
        // 说明当前像素被遮挡
        float depth_diff = ray_pos.z - sample_world_pos.z; // 视线方向 Z

        // 正值意味着采样点在射线前方（更靠近相机不太对）
        // 实际判断：几何体挡住了从该点看向光源的视线
        if (depth_diff > 0 && depth_diff < THICKNESS * (i + 1))
        {
            // 有遮挡，根据步数进度做衰减
            float attenuation = 1.0 - (float)i / STEP_COUNT;
            contact_shadow = min(contact_shadow, 1.0 - attenuation);
            break; // 找到遮挡即可
        }
    }

    // 距离衰减：远处不生效
    float distance_fade = smoothstep(MAX_DISTANCE, MAX_DISTANCE * 0.5,
                                      length(world_pos - _CameraWorldPos));
    contact_shadow = lerp(contact_shadow, 1.0, 1.0 - distance_fade);

    return contact_shadow;
}
```

**URP Renderer Feature 结构**：

```csharp
public class ContactShadowFeature : ScriptableRendererFeature
{
    public Settings settings;

    class ContactShadowPass : ScriptableRenderPass
    {
        const string k_Tag = "Contact Shadow";

        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            // 配置半分辨率 RT
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.width /= 2;
            desc.height /= 2;
            desc.depthBufferBits = 0;
            cmd.GetTemporaryRT(m_TemporaryRTID, desc);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            // 1. 获取 Camera Depth（需要 depth texture 开启）
            // 2. Blit Contact Shadow Material 到半分辨率 RT
            // 3. 设置全局 _ContactShadowTex 供场景 Shader 采样
        }

        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            cmd.ReleaseTemporaryRT(m_TemporaryRTID);
        }
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        renderer.EnqueuePass(m_Pass);
    }
}
```

**在角色材质或全局光照 Shader 中混合**：

```hlsl
// 在光照计算中合并阴影
float main_shadow = SAMPLE_SHADOWMAP atten; // 引擎 Shadow Map
float contact_shadow = tex2D(_ContactShadowTex, uv).r; // 接触阴影 RT

// Contact Shadow 只增强近距离
float final_shadow = min(main_shadow, contact_shadow);
```

### ⚡ 实战经验

1. **步数不是越多越好**：8 步在大多数场景已经够用，16 步以上移动端扛不住。关键是调好 `THICKNESS` 容差
2. **半分辨率 + 双线性滤波**：接触阴影本身就是软效果，半分辨率完全够用，最后做一次 Blur 更柔和
3. **和 SSAO 共享深度**：如果项目已经有 SSAO Pass，Contact Shadow 可以复用同样的深度采样，几乎零额外带宽
4. **室内场景慎用**：室内光源复杂、平行光贡献小的情况下，Contact Shadow 效果不明显，反而增加功耗
5. **调试技巧**：先用白色输出 `contact_shadow` 看覆盖范围，确认 Ray Marching 方向正确再调参数

### 🎯 能力体检清单

- [ ] 能解释为什么 Shadow Map 近距离精度差（Shadow Map 精度分布原理）
- [ ] 理解屏幕空间 Ray Marching 的原理与局限（只能看到屏幕内的几何体）
- [ ] 能写出将世界坐标投影到屏幕 UV 的代码
- [ ] 知道如何将 Contact Shadow 作为 Renderer Feature 集成到 URP
- [ ] 理解 `THICKNESS` 参数对自阴影伪影的影响
- [ ] 能设计 Contact Shadow 与传统 Shadow Map 的混合策略
- [ ] 如果面试官问"能不能替代 Shadow Map"，你能回答为什么不行（屏幕空间信息缺失、远处/背后物体无法检测）

### 🔗 相关问题

- [角色脚底软阴影丢失](../rendering/character-ground-contact-soft-shadow.md)
- [屏幕空间阴影柔化](../rendering/custom-screen-space-shadow-soften.md)
- [Hi-Z 屏幕空间反射](../rendering/hiz-screen-space-reflection.md)
