---
title: "手游GPU带宽爆了，如何系统性优化？"
category: "optimization"
level: 3
tags: ["GPU带宽", "移动端优化", "渲染性能", "Tile-Based GPU", "Mali/Adreno"]
hint: "移动端GPU是Tile-Based架构，带宽瓶颈往往不在计算而在访存——你的Shader在反复读写显存吗？"
related: ["optimization/drawcall-500-to-100", "rendering/urp-renderer-feature"]
---

## 参考答案

### 🎬 场景描述

> 面试官：我们的一款手游在高端安卓机上帧率稳定，但在中低端机型上发热严重、帧率波动大。Profiler显示GPU耗时占比很高，但Draw Call只有80多个，三角形数也不高。你排查后发现是带宽瓶颈。请说说你的分析思路和优化方案。

### ✅ 核心要点

1. **确认瓶颈类型**：是计算瓶颈（ALU）还是带宽瓶颈（Memory Bandwidth），二者优化方向完全不同
2. **理解Tile-Based渲染**：移动端GPU（Mali/Adreno/PowerVR）采用分块渲染，理解Tile、Tiler、HLSL对带宽的影响
3. **识别高带宽操作**：Blur、SSAO、多次采样纹理、Alpha Test、颜色格式不当等都是带宽杀手
4. **Framebuffer优化**：合理选择颜色格式（RGBA32 vs RGBA64）、减少RT切换、利用Force Store/Load控制
5. **Shader级别优化**：减少纹理采样次数、使用Mipmap Chain代替多层手动LOD、避免分支导致的梯度计算

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
中低端机发热 + 帧率波动
  ↓ Step 1: 抓帧分析（GAPID / RenderDoc / XCode GPU Capture）
确认是带宽瓶颈而非计算瓶颈
  ↓ Step 2: 逐Pass分析带宽消耗
找出带宽Top 3的渲染Pass
  ↓ Step 3: 针对性优化
每个高带宽Pass → 具体优化策略
  ↓ Step 4: 验证 & 回归
优化后重新抓帧对比，确认带宽下降
```

#### 知识点拆解（倒推树）

```
GPU带宽优化
├── 移动端GPU架构
│   ├── Tile-Based Deferred Rendering (TBDR)
│   ├── Tile Memory（片上内存）→ 避免往返主存
│   ├── Tiler（几何处理阶段）vs Shader Core（像素处理阶段）
│   └── Load/Store Action：DontCare / Load / Store / StoreResolve
│
├── 带宽热点定位
│   ├── Mali Streamline / Mali Offline Compiler
│   ├── Adreno Profiler / Snapdragon Profiler
│   ├── RenderDoc：查看纹理格式 & 采样次数
│   └── 关键指标：Bytes/Frame、Reads/Frag、Writes/Frag
│
├── 高带宽操作识别与优化
│   ├── 后处理链路（最常见元凶）
│   │   ├── Blur：分离卷积（两遍Pass）代替全卷积
│   │   ├── Bloom：降采样后再模糊
│   │   └── Color Grading：LUT代替逐像素计算
│   ├── G-Buffer / MRT（延迟渲染在移动端的带宽代价）
│   ├── Alpha Test / Kill：破坏Early-Z，导致过度渲染
│   └── 多次Blit / CopyTexture
│
├── Framebuffer策略
│   ├── 颜色格式选择：RGB565 / RGBA4444 / RGBA8 / RGBA16F
│   ├── 深度格式：D24S8 够用就别上 D32FS8
│   ├── MSAA vs FXAA：硬件MSAA在TBDR上带宽代价远小于PC
│   └── RenderTarget复用与生命周期管理
│
└── Shader级优化
    ├── 纹理采样：3张采样改2张（通道合并）
    ├── Mipmap：开启 trilinear/anisotropic 的代价
    ├── 分支与梯度：ddx/ddy 在分支内的问题
    └── 精度选择：mediump vs highp（移动端影响带宽和功耗）
```

#### 代码实现

**带宽分析表（实战模板）**

| Render Pass | RT格式 | 分辨率 | 采样纹理数 | Load/Store | 估算带宽/帧 | 优化方案 |
|---|---|---|---|---|---|---|
| Opaque | RGBA8 | 1080p | 4 | DontCare→Store | 8.3 MB | ✅ 已优化 |
| UI | RGBA8 | 1080p | 2 | Load→Store | 8.3 MB | 合并到最终Pass |
| Bloom Downsample | RGBA16F | 540p→135p | 1 | DontCare→Store | 3.4 MB | ✅ 已降采样 |
| Gaussian Blur H | RGBA16F | 540p | 1 | Load→Store | 4.2 MB | 改RGB565 |
| Gaussian Blur V | RGBA16F | 540p | 1 | Load→Store | 4.2 MB | 改RGB565 |
| Color Grading | RGBA8 | 1080p | 2 | Load→Store | 8.3 MB | 用LUT纹理 |

**Unity URP 中控制 Load/Store Action（核心代码）：**

```csharp
// 自定义 ScriptableRendererFeature 中控制 RT 的 Store Action
public class OptimizedBloomFeature : ScriptableRendererFeature
{
    class BloomPass : ScriptableRenderPass
    {
        RTHandle m_Source;
        RTHandle m_TempTarget;

        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            // 关键：使用 DontCare 作为 LoadAction，避免读取上一帧内容
            var loadOp = RenderBufferLoadAction.DontCare;
            var storeOp = RenderBufferStoreAction.Store;
            
            var desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.depthBufferBits = 0; // 不需要深度，省带宽
            
            // 降采样：带宽与分辨率平方成正比
            desc.width /= 2;
            desc.height /= 2;
            // 用 RGBA8 或 RGB565 代替 RGBA16F 如果不需要HDR
            desc.colorFormat = RenderTextureFormat.ARGB32;
            
            RenderingUtils.ReAllocateIfNeeded(ref m_TempTarget, desc, name: "_BloomTemp");
            
            // 配置目标，指定 Load/Store Action
            ConfigureTarget(m_TempTarget);
            ConfigureClear(loadOp, storeOp, Color.clear);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("OptimizedBloom");
            // 分离式高斯模糊：水平+垂直两Pass，O(n)代替O(n²)
            cmd.SetComputeTextureParam(m_BloomCS, 0, "_Source", m_Source);
            cmd.SetComputeTextureParam(m_BloomCS, 0, "_Dest", m_TempTarget);
            // Dispatch...
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

**Shader精度优化（移动端关键）：**

```hlsl
// ❌ 全部 highp —— 浪费带宽和寄存器
half4 frag(v2f i) : SV_Target {
    half4 col = tex2D(_MainTex, i.uv);
    half3 lightDir = normalize(_WorldSpaceLightPos0.xyz);
    // 所有计算都用 highp...
}

// ✅ 合理使用 mediump/lowp
// 顶点着色器：位置用 highp，颜色用 mediump
// 片段着色器：颜色用 mediump（移动端 half = mediump）
half4 frag(v2f i) : SV_Target {
    // Unity 中 half 在移动端会编译为 mediump
    half4 col = tex2D(_MainTex, i.uv);
    half3 lightDir = normalize(_WorldSpaceLightPos0.xyz);
    half ndotl = max(0, dot(col.rgb, lightDir));
    
    // 最终输出提升精度
    return half4(col.rgb * ndotl, col.a);
}

// Vulkan/GLSL 中显式指定精度
// precision mediump float;
// precision highp int;
```

### ⚡ 实战经验

1. **后处理是带宽杀手No.1**：手游项目中Bloom+Blur占GPU带宽40%+是常态。优先降采样再模糊，540p甚至270p做模糊完全够用
2. **Alpha Test 是隐形带宽陷阱**：`clip()` 会破坏 Early-Z，导致被遮挡像素也被着色。移动端尽量用 Alpha Blend 代替 Alpha Test，或用两遍Pass（先Z-Prepass再着色）
3. **精简后处理链**：一个"真正的移动端后处理链"应该只有 Tone Mapping + 一个合并的 Bloom+Color Grading Pass。每多一个FullScreen Pass就多一帧的FrameBuffer带宽
4. **Mali GPU 特别注意**：Mali的Tiler有几何体上限，超限后会回退到Direct模式（丧失Tile优化），导致带宽暴涨。控制场景顶点数在Tiler预算内

### 🎯 能力体检清单

| 卡点 | 说明 | 学习建议 |
|---|---|---|
| 不知道Tile-Based架构 | 你不理解移动端GPU为什么和PC优化思路不同 | 研读ARM Mali GPU Architecture白皮书 |
| 不会用移动端Profiler | 你只会看Unity Frame Debugger但不会看GPU级别数据 | 学习 Snapdragon Profiler / Mali Streamline |
| 不清楚Load/Store Action | 你不理解为什么Render Pass的配置影响带宽 | 学习Vulkan/Metal的Render Pass概念 |
| 混淆计算瓶颈和带宽瓶颈 | 你优化方向不对，越优化越差 | 理解 ALU/Texture vs Bandwidth 的区别，学会用Profiler区分 |
| 不知道精度限定符的影响 | 你的Shader在移动端没有节省任何功耗 | 实践GLSL precision限定符，对比编译后的寄存器使用 |

### 🔗 相关问题

- [Draw Call从500降到100怎么做？](../optimization/drawcall-500-to-100.md) —— 计算与Draw Call优化方向
- [URP自定义Renderer Feature怎么写？](../rendering/urp-renderer-feature.md) —— 后处理Pass的工程实现
- 移动端延迟渲染可行吗？G-Buffer带宽代价如何评估？
