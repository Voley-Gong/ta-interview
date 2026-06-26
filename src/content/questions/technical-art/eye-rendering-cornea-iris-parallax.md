---
title: "角色眼球渲染：如何实现会追随目光、有深度感的真实眼睛？"
category: "technical-art"
level: 3
tags: ["眼球渲染", "角膜折射", "视差", "PBR", "角色材质", "二次元", "写实"]
hint: "核心难点：角膜透明层 + 虹膜视差深度 + 瞳孔形变 + 环境反射，四层叠加才有'活'的感觉"
related: ["shader/eye-iris-refraction", "technical-art/pbr-material-authoring", "technical-art/face-shadow-shimmering"]
---

## 参考答案

### 🎬 场景描述

面试官说：「我们做一个角色向游戏（叠纸/鹰角风格），角色特写镜头时眼睛很出戏——眼球看起来像贴纸一样平。你来设计一套眼球渲染方案，要满足：1）虹膜有深度感不是平的；2）角膜有真实的透明折射；3）瞳孔会随光照收缩放大；4）要有环境反射让眼睛'活'起来。给我完整的材质方案和 Shader 思路。」

### ✅ 核心要点

1. **眼球分层结构**：巩膜（眼白）+ 虹膜（彩色）+ 瞳孔（黑洞）+ 角膜（透明外层），每层独立贴图
2. **虹膜深度视差**：用 Parallax Mapping 或 Parallax Occlusion Mapping 让虹膜"凹进去"
3. **角膜透明折射**：透明叠加层 + 法线扰动模拟折射，或用预计算的环境贴图反射
4. **瞳孔光照响应**：根据场景亮度动态缩放瞳孔 UV 范围
5. **环境反射**：角膜层叠加 Planar Reflection 或 Cubemap 反射，增加"湿润感"
6. **视线方向控制**：通过材质参数或骨骼控制虹膜/瞳孔偏移，实现目光追随

### 📖 深度展开

#### 解决思路（从效果倒推实现）

```
最终效果：角色眼睛有深度感、有反射、瞳孔会动态变化、目光会追随
                ↑
倒推1：眼睛"平" → 缺少深度 → 虹膜用视差贴图模拟凹面
倒推2：眼睛"死" → 缺少反射 → 角膜层加环境反射（Cubemap/SSR）
倒推3：瞳孔不动 → 缺少光照响应 → 用 Luminance 驱动瞳孔缩放
倒推4：角膜透明层 → 需要独立的透明 Pass + 折射扰动
倒推5：目光追随 → 需要虹膜+瞳孔沿眼球表面滑动（UV偏移或骨骼旋转）
倒推6：特写无锯齿 → 虹膜贴图分辨率要够 + Mipmap Bias 调整
```

#### 知识点拆解（倒推树）

```
角色眼球渲染
├── 解剖结构理解
│   ├── 巩膜 / 虹膜 / 瞳孔 / 角膜 四层
│   └── 各层的材质属性差异（粗糙度/透射/反射）
├── 虹膜深度
│   ├── Parallax Mapping（廉价方案）
│   ├── Parallax Occlusion Mapping（中等）
│   └── 实际几何凹面建模（高端方案，Tessellation）
├── 角膜层
│   ├── 透明混合（Alpha Blending vs Alpha Test）
│   ├── 折射近似（法线扰动 + 背景采样偏移）
│   └── 环境反射（Cubemap / SSR / Planar Reflection）
├── 瞳孔动态
│   ├── 场景亮度计算（Luminance from Light Color/Intensity）
│   ├── 瞳孔缩放范围（2mm~8mm，映射到 UV Scale）
│   └── 平滑过渡（lerp + damping）
├── 视线追随
│   ├── 方案A：UV 偏移（简单但有限）
│   ├── 方案B：骨骼驱动眼球旋转（标准方案）
│   └── 方案C：Look-At Constraint + IK（高级方案）
├── 贴图规范
│   ├── 虹膜贴图分辨率（至少 512×512，特写需要 1K）
│   ├── 法线贴图（虹膜纤维细节）
│   └── 粗糙度贴图（巩膜湿润感 vs 虹膜干燥感）
└── 移动端适配
    ├── 简化：去掉角膜折射，只用视差 + 反射
    ├── 虹膜贴图降分辨率 + Mipmap Bias
    └── 瞳孔缩放用顶点色控制代替像素计算
```

#### 代码实现

**角膜层 Shader（URP HLSL）核心片段：**

```hlsl
// === 角膜透明层 ===
// 角膜是一层透明的弧面，覆盖在虹膜上方
// 需要：透明叠加 + 折射扰动 + 环境反射

// 1. 角膜法线（使用法线贴图 + 细微扰动）
half3 corneaNormal = UnpackNormal(tex2D(_CorneaNormalMap, uv));
corneaNormal = normalize(corneaNormal * _NormalScale);

// 2. 折射近似：用法线扰动偏移虹膜 UV
float2 refractedUV = uv + corneaNormal.xy * _RefractionStrength;
half3 irisColor = tex2D(_IrisMap, refractedUV).rgb;

// 3. 虹膜视差深度（让虹膜"凹进去"）
// 使用 Parallax Occlusion Mapping 简化版
float parallaxDepth = tex2D(_IrisDepthMap, refractedUV).r;
float2 parallaxOffset = normalize(viewDirTan.xy) * parallaxDepth * _ParallaxScale;
half3 irisColorWithDepth = tex2D(_IrisMap, refractedUV - parallaxOffset).rgb;

// 4. 瞳孔缩放（根据场景亮度动态变化）
float sceneLuminance = saturate(dot(_WorldSpaceLightPos0.xyz, float3(0.3, 0.59, 0.11)));
float pupilScale = lerp(_PupilMaxScale, _PupilMinScale, sceneLuminance);
// 以虹膜中心为原点缩放瞳孔区域
float2 pupilUV = (uv - _IrisCenterUV) * pupilScale + _IrisCenterUV;
float isInPupil = step(length(uv - _IrisCenterUV), _PupilRadius * pupilScale);
half3 finalIris = lerp(irisColorWithDepth, _PupilColor, isInPupil);

// 5. 环境反射（角膜湿润感）
half3 reflectDir = reflect(-viewDir, corneaNormal);
half3 envReflection = SAMPLE_TEXTURECUBE(_EnvCubeMap, sampler_EnvCubeMap, reflectDir).rgb;
envReflection *= _ReflectionIntensity;

// 6. 组合：角膜 = 虹膜底色 + 环境反射 + 镜面高光
half NdotV = saturate(dot(corneaNormal, viewDir));
half fresnel = pow(1.0 - NdotV, 5.0);
half3 corneaColor = finalIris * (1.0 - fresnel * 0.5) + envReflection * fresnel;

// 镜面高光（眼神光）
half3 specHighlight = pow(saturate(dot(corneaNormal, halfDir)), _GlossPower) * _SpecIntensity;
corneaColor += specHighlight;

return half4(corneaColor, _CorneaAlpha);
```

**C# 瞳孔动态响应控制脚本：**

```csharp
using UnityEngine;

[RequireComponent(typeof(Renderer))]
public class EyePupilController : MonoBehaviour
{
    [Header("瞳孔参数")]
    [SerializeField] private float pupilMinScale = 0.6f;  // 亮环境收缩
    [SerializeField] private float pupilMaxScale = 1.4f;  // 暗环境放大
    [SerializeField] private float responseSpeed = 3.0f;  // 收缩/放大速度
    [SerializeField] private Light keyLight;              // 主光源参考

    private Material _eyeMaterial;
    private float _currentPupilScale = 1.0f;
    private int _PupilScaleID;

    void Start()
    {
        _eyeMaterial = GetComponent<Renderer>().material;
        _PupilScaleID = Shader.PropertyToID("_PupilScale");
    }

    void Update()
    {
        // 计算场景亮度（主光源强度 × 颜色亮度）
        float luminance = 0.3f;
        if (keyLight != null)
        {
            Color c = keyLight.color;
            float colorLum = c.r * 0.299f + c.g * 0.587f + c.b * 0.114f;
            luminance = keyLight.intensity * colorLum;
        }

        // 目标瞳孔缩放
        float targetScale = Mathf.Lerp(pupilMaxScale, pupilMinScale, Mathf.Clamp01(luminance));

        // 平滑过渡
        _currentPupilScale = Mathf.Lerp(_currentPupilScale, targetScale, Time.deltaTime * responseSpeed);
        _eyeMaterial.SetFloat(_PupilScaleID, _currentPupilScale);
    }
}
```

**贴图制作规范（Substance Painter 层级）：**

```
眼球贴图集（1张模型，多张贴图）：
├── _IrisMap（虹膜颜色）     : RGB，512×512，sRGB
├── _IrisDepthMap（虹膜深度）: R通道，512×512，Linear
│   └── 虹膜纤维凹凸 → 白色凸起，黑色凹陷
├── _CorneaNormalMap（角膜法线）: RGB，256×256，Linear
│   └── 非常细微的扰动，模拟泪膜不规则
├── _ScleraMap（巩膜/眼白）   : RGB，512×512，sRGB
│   └── 微血管细节，边缘偏红黄
├── _AOOcclusion（环境光遮蔽） : R通道，256×256，Linear
│   └── 眼球与眼眶交界处的暗角
└── _RoughnessMap（粗糙度）
    ├── 巩膜：0.3~0.5（微湿润）
    ├── 虹膜：0.6~0.9（较干燥）
    └── 角膜：0.05~0.1（非常光滑）
```

### ⚡ 实战经验

1. **不要用半球做眼球**：很多项目偷懒用一个 UV 球加贴图，虹膜看起来就是平的圆。至少要给虹膜区域单独做视差或凹面建模。
2. **角膜反射强度要克制**：二次元风格反射强度要比写实低 50%+，否则眼睛看起来像玻璃珠。
3. **瞳孔缩放不要瞬时**：真实瞳孔响应有 0.2~0.5 秒延迟，加 damping 让动画自然。
4. **特写镜头单独做 LOD**：眼球特写时切换高精度材质（2K 虹膜 + POM），正常镜头用 512 + 简单视差。
5. **目光追随用骨骼最稳**：UV 偏移方案在极端角度会拉伸虹膜，骨骼旋转眼球是标准做法。
6. **NSFW 检查**：眼球法线贴图如果反了，角色会看起来像恶魔——贴图导入时检查法线方向。

### 🎯 能力体检清单

- [ ] 能说清眼球四层结构（巩膜/虹膜/瞳孔/角膜）的材质差异吗？
- [ ] 虹膜深度你会用 Parallax Mapping 还是几何建模？各自的代价？
- [ ] 角膜折射你用的是什么近似方案？和真实物理折射差在哪？
- [ ] 瞳孔缩放是纯美术驱动还是程序驱动？如果要做自动曝光响应怎么接入？
- [ ] 移动端这套方案怎么砍？哪些层可以合并？哪些效果可以去掉？
- [ ] 眼球贴图在什么情况下会出现摩尔纹？怎么解决？
- [ ] 如果美术说"眼睛看起来没有光"，你的排查路径是什么？

### 🔗 相关问题

- [角色虹膜折射：Shader 中如何模拟角膜的折射效果？](shader/eye-iris-refraction)
- [PBR 材质制作：从 Substance 到引擎的完整工作流](technical-art/pbr-material-authoring)
- [面部阴影抖动：角色脸部法线贴图导致的阴影闪烁问题](technical-art/face-shadow-shimmering)
- [角色卡通描边：NPR 渲染中的多种 Outline 方案对比](shader/npr-outline-cartoon)
