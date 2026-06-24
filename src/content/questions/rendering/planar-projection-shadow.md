---
title: "卡通角色专用阴影：URP 下平面投影阴影 + Blob Shadow 全方案"
category: "rendering"
level: 2
tags: ["平面投影阴影", "Blob Shadow", "卡通渲染", "URP", "Render Feature", "手游阴影"]
hint: "ShadowMap 对卡通角色太重——平面投影 / Blob Shadow 才是手游标配，但怎么处理边缘柔化和多角色重叠？"
related: ["rendering/urp-renderer-feature", "rendering/cel-shading-toon-pipeline", "optimization/drawcall-500-to-100"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一款卡通风格手游，角色用 ShadowMap 阴影开销太大，渲染品质也不匹配——NPR 角色需要的是干净的、有形状的圆形阴影。你在 URP 下给我一套角色专用阴影方案。」

### ✅ 核心要点

1. **方案选型**：平面投影阴影（Planar Projection）适合角色脚下圆斑；Blob Shadow 适合精确控制形状
2. **实现路径**：Render Feature（URP）捕获角色轮廓 → 投影到地面 → blur 边缘柔化
3. **性能优势**：无需 ShadowMap Pass，省一个 Draw Call 的深度渲染；移动端 GPU 友好
4. **多角色重叠**：用一张共享的 `_ShadowRT` 做 alpha 叠加，正确处理重叠区域
5. **边缘柔化**：Gaussian Blur 或 disk-shaped kernel 让投影边缘自然过渡

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色脚下有干净圆形阴影 → 边缘柔和 → 多角色不重叠加色 → 跟随角色移动
                ↑
倒推1：需要「角色形状的阴影」→ 方案A: 从角色轮廓投影到地面
倒推2：需要「圆形简化」→ 方案B: Blob Shadow（预渲染 blob texture 贴在脚下）
倒推3：需要「边缘柔化」→ Render Texture 上做 blur / 或 blob texture 本身带柔边
倒推4：需要「正确叠加」→ 共享 RT 用 max blend 避免重叠区域过暗
倒推5：需要「跟随移动」→ C# 脚本 raycast 地面位置 → 更新投影位置
```

#### 知识点拆解（倒推树）

```
卡通角色专用阴影
├── 方案选型
│   ├── Planar Projection Shader（真实轮廓投影，效果准确）
│   ├── Blob Shadow Projector（预渲染纹理投影，简单高效）
│   ├── Decal System（URP 4.x+ 原生支持，推荐方案）
│   └── 对比：ShadowMap（写实标配，卡通不推荐）
├── URP Render Feature 实现
│   ├── ScriptableRendererFeature（自定义渲染 pass）
│   ├── RenderTarget 管理（分配 / 释放 _ShadowRT）
│   ├── Blur Pass（边缘柔化）
│   └── Blit Pass（合成到屏幕）
├── Blob Shadow 方案
│   ├── Projector 组件（Built-in 管线）
│   ├── Decal Projector（URP 原生，2021.2+）
│   ├── 预渲染 Blob Texture（不同形状：圆形 / 椭圆 / 角色轮廓）
│   └── 脚本控制位置 + 朝向（Raycast 到地面）
├── 多角色叠加
│   ├── 共享 RT + max blend（避免重叠过暗）
│   ├── 按角色 ID 分通道（R = 角色1, G = 角色2）
│   └── 透明度叠加 vs max blend（视觉效果对比）
└── 性能与品质
    ├── RT 分辨率选择（512 vs 1024，与画面分辨率解耦）
    ├── Blur kernel 大小（移动端用 5x5，PC 用 9x9）
    └── 阴影颜色与环境光协调（不是纯黑，偏蓝/偏暖）
```

#### 代码实现

**方案A：URP Decal Projector（最简方案，2021.2+）：**

```csharp
using UnityEngine;
using UnityEngine.Rendering.Universal;

[RequireComponent(typeof(DecalProjector))]
public class CharacterBlobShadow : MonoBehaviour
{
    [SerializeField] private Transform character;
    [SerializeField] private LayerMask groundMask = ~0;
    [SerializeField] private float raycastHeight = 2f;
    [SerializeField] private float shadowSize = 1.5f;

    private DecalProjector _decal;
    private Vector3 _offset;

    void Awake()
    {
        _decal = GetComponent<DecalProjector>();
        _offset = transform.localPosition;
    }

    void LateUpdate()
    {
        // Raycast 找地面高度
        Vector3 origin = character.position + Vector3.up * raycastHeight;
        if (Physics.Raycast(origin, Vector3.down, out RaycastHit hit, raycastHeight + 5f, groundMask))
        {
            // 更新投影位置到地面
            Vector3 pos = hit.point + Vector3.up * 0.01f;
            _decal.transform.position = new Vector3(character.position.x, pos.y, character.position.z);

            // 根据高度调整阴影大小（越高越大越淡）
            float heightAboveGround = character.position.y - hit.point.y;
            float scaleFactor = 1f + Mathf.Clamp01(heightAboveGround / 5f) * 0.5f;
            _decal.size = new Vector3(shadowSize * scaleFactor, shadowSize * scaleFactor, 5f);

            // 高度越高越淡
            float opacity = Mathf.Lerp(0.8f, 0.2f, Mathf.Clamp01(heightAboveGround / 5f));
            // URP Decal 不直接暴露 opacity，需要材质参数控制
            _decal.material.SetFloat("_Opacity", opacity);
        }
    }
}
```

**方案B：Render Feature + 平面投影 Shader（更精确的方案）：**

```csharp
// CharacterShadowFeature.cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class CharacterShadowFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent passEvent = RenderPassEvent.AfterRenderingOpaques;
        public LayerMask characterLayer;
        public LayerMask groundLayer;
        public int shadowRTSize = 512;
        public float blurRadius = 3f;
        public Color shadowColor = new Color(0, 0, 0, 0.5f);
    }

    public Settings settings = new Settings();
    private CharacterShadowPass _pass;

    public override void Create()
    {
        _pass = new CharacterShadowPass(settings);
        _pass.renderPassEvent = settings.passEvent;
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        renderer.EnqueuePass(_pass);
    }

    protected override void Dispose(bool disposing)
    {
        _pass?.Dispose();
    }
}

public class CharacterShadowPass : ScriptableRenderPass
{
    private CharacterShadowFeature.Settings _settings;
    private RTHandle _shadowRT;
    private Material _blurMat;
    private static readonly int ShadowRTID = Shader.PropertyToID("_CharacterShadowRT");

    public CharacterShadowPass(CharacterShadowFeature.Settings settings)
    {
        _settings = settings;
        var desc = new RenderTextureDescriptor(_settings.shadowRTSize, _settings.shadowRTSize,
            RenderTextureFormat.ARGB32, 0);
        RenderingUtils.ReAllocateIfNeeded(ref _shadowRT, desc, name: "_CharacterShadowRT");

        _blurMat = CoreUtils.CreateEngineMaterial(Shader.Find("Hidden/CharacterShadowBlur"));
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("CharacterShadow");

        // 1. 清空 RT
        cmd.SetRenderTarget(_shadowRT);
        cmd.ClearRenderTarget(false, true, Color.clear);

        // 2. 渲染角色轮廓到 RT（从正上方俯视）
        // 这里简化：实际需要设置正交相机矩阵
        // cmd.DrawRendererList(...) 渲染 characterLayer

        // 3. Blur
        _blurMat.SetFloat("_BlurRadius", _settings.blurRadius);
        cmd.Blit(_shadowRT, _shadowRT, _blurMat);

        // 4. 合成到屏幕（在地面区域混合）
        // 实际中用第二个 shader 做屏幕空间合成

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    public void Dispose()
    {
        _shadowRT?.Release();
        if (_blurMat != null) CoreUtils.Destroy(_blurMat);
    }
}
```

**平面投影 Shader（角色渲染到 RT 时使用）：**

```hlsl
// 将角色从世界空间投影到地面平面的阴影 Shader
// 简化版：从角色正上方渲染深度，投影到地面

half4 FragShadow(Varyings IN) : SV_Target
{
    // 渲染角色的纯色轮廓（白色），后续用作阴影 mask
    return half4(1, 1, 1, 1);
}
```

**Blob Texture 生成（运行时或预烘焙）：**

```csharp
// 程序化生成圆形 blob 阴影纹理
Texture2D GenerateBlobTexture(int size = 128)
{
    Texture2D tex = new Texture2D(size, size, TextureFormat.RGBA32, false);
    float center = size * 0.5f;
    float radius = center * 0.8f;

    for (int y = 0; y < size; y++)
    {
        for (int x = 0; x < size; x++)
        {
            float dist = Vector2.Distance(new Vector2(x, y), new Vector2(center, center));
            // 柔边：smoothstep 衰减
            float alpha = 1f - Mathf.SmoothStep(radius * 0.7f, radius, dist);
            tex.SetPixel(x, y, new Color(0, 0, 0, alpha));
        }
    }
    tex.Apply();
    return tex;
}
```

**方案对比表：**

| 方案 | 精度 | 性能 | 实现复杂度 | 适用场景 |
|------|------|------|-----------|----------|
| URP Decal Projector | 中（贴图投影） | ★★★★★ | 低 | 卡通手游（推荐首选） |
| Blob Shadow + 脚本 | 低（固定形状） | ★★★★★ | 低 | 简单手游 / 2.5D |
| Render Feature + RT | 高（真实轮廓） | ★★★☆☆ | 高 | 高品质卡通 / 需要精确轮廓 |
| ShadowMap (URP 原生) | 高 | ★★☆☆☆ | 低 | 写实渲染 |
| Screen Space Shadow | 高 | ★★☆☆☆ | 中 | PC / 主机 |

### ⚡ 实战经验

- **Decal Projector 是 2024 年后的首选**：URP 原生支持，无需自己写 Render Feature，3 分钟集成完毕
- **阴影颜色不是纯黑**：偏蓝 `RGBA(0.1, 0.1, 0.2, 0.5)` 或偏暖 `RGBA(0.15, 0.1, 0.08, 0.5)` 更自然
- **阴影跟随逻辑**：LateUpdate 中 Raycast 找地面，不要在 Update 里做（角色移动还没算完）
- **跳跃时阴影变大变淡**：高度越高，阴影尺寸 ×1.5、透明度 ×0.3，模拟真实物理光照
- **不要用 Projector 组件**：Built-in 管线的 Projector 在 URP 下不兼容，用 Decal Projector 替代
- **性能要点**：Decal 贴图用 64×64 足够，不需要高分辨率；blur 在 Shader 里做一次 3×3 就够

### 🎯 能力体检清单

| 卡点 | 盲区 | 补习方向 |
|------|------|----------|
| 不知道除了 ShadowMap 还有什么方案 | 投影 / Decal 系统 | 学 URP Decal 子系统 |
| Decal 投影拉伸到墙壁上 | Decal 裁剪逻辑 | 学 Decal Projector 的 size / pivot 设置 |
| 多角色阴影重叠区域太暗 | Blend 模式理解 | 学 max blend vs normal blend 的区别 |
| 阴影不跟随角色 | 生命周期 / 执行顺序 | 学 Unity 脚本执行顺序（Update vs LateUpdate） |
| 卡通角色用 ShadowMap 看起来脏 | NPR 渲染哲学 | 理解卡通渲染中「少即是多」的视觉设计 |

### 🔗 相关问题

- [URP 自定义 Renderer Feature](urp-renderer-feature)：Render Feature 的完整生命周期是什么？
- [卡通渲染管线](cel-shading-toon-pipeline)：NPR 角色的全套渲染方案中，阴影如何配合？
- [Draw Call 从 500 降到 100](optimization/drawcall-500-to-100)：每个角色一个 Decal 投影如何批量优化？
