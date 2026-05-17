# UE5 日式动漫风格材质技术文档

> 基于 Unreal Engine 5 + Unlit 材质模式的 Anime/Toon Shading 实现方案
> 适用模型：SKM_Manny（ThirdPerson 模板）或任意 SkeletalMesh

---

## 目录

1. [整体架构](#整体架构)
2. [前置准备](#前置准备)
3. [Layer 1 — Cel Shading 卡通光照](#layer-1--cel-shading-卡通光照)
4. [Layer 2 — 硬边高光](#layer-2--硬边高光)
5. [Layer 3 — 边缘光 Rim Light + 描边](#layer-3--边缘光-rim-light--描边)
6. [固有色贴图接入](#固有色贴图接入)
7. [后处理设置](#后处理设置)
8. [灯光设置](#灯光设置)
9. [参数速查表](#参数速查表)
10. [已知问题与优化方向](#已知问题与优化方向)

---

## 整体架构

```
DirectionalLight 方向
        ↓
   NdotL 计算
        ↓
   Cel Shading     →  两色阶卡通光照（阴影色 / 亮部色）
        ↓
   硬边高光         →  Blinn-Phong + Step 硬圆高光
        ↓
   Rim Light        →  1-NdotV 边缘光（蓝色）
        ↓
   内描边           →  法线夹角阈值变黑
        ↓
   × 固有色贴图
        ↓
   Emissive Color   →  主节点输出（Unlit 模式）
```

所有光照计算完全自定义，不依赖 UE 物理 PBR，Shading Model 设为 **Unlit**。

---

## 前置准备

### 材质主节点设置

新建 Material，命名 `M_AnimeToon`，双击打开后在主节点 Details 里：

| 属性 | 值 |
|------|------|
| Blend Mode | Opaque |
| Shading Model | **Unlit** |

> 切换 Unlit 后只有 Emissive Color 引脚有效，其余引脚变灰是正常的。

---

## Layer 1 — Cel Shading 卡通光照

### 节点连线

```
[Custom: LightDir] ──→ B ┐
                          ├─ [DotProduct] → [Clamp 0~1] → NdotL
[VertexNormalWS]   ──→ A ┘

NdotL ──────────────────────────────→ [Custom: CelShade] → NdotL
[ScalarParam: ShadowThreshold 0.5] → [Custom: CelShade] → Threshold
[VectorParam: ShadowColor]         → [Custom: CelShade] → ShadowColor
[VectorParam: LitColor]            → [Custom: CelShade] → LitColor
                                                             ↓
                                                        Emissive Color
```

### Custom 节点 1：获取光方向

```
Description : LightDir
Output Type : CMOT Float3
Inputs      : 无
```

```hlsl
return normalize(-ResolvedView.DirectionalLightDirection);
```

> 注意负号：UE 内置方向是"物体→光源"，取反后才是"光源→物体"方向，用于 NdotL 计算。

### Custom 节点 2：卡通色阶

```
Description : CelShade
Output Type : CMOT Float3
Inputs      : NdotL / Threshold / ShadowColor / LitColor
```

```hlsl
float cel = smoothstep(Threshold - 0.02, Threshold + 0.02, NdotL);
return lerp(ShadowColor, LitColor, cel);
```

`smoothstep` 的 `0.02` 范围提供微量软过渡，避免锯齿。想要更硬的边改成 `0.001`。

### 参数默认值

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ShadowThreshold | Scalar | 0.45 | 明暗分界位置，越大阴影越少 |
| ShadowColor | Vector | (0.12, 0.09, 0.22) | 蓝紫阴影色 |
| LitColor | Vector | (1.0, 0.97, 0.93) | 暖白亮部色 |

---

## Layer 2 — 硬边高光

### Custom 节点：Specular

```
Description : Specular
Output Type : CMOT Float3
Inputs      : CelColor / LightDir / CameraDir / Normal / Threshold / SpecColor
```

```hlsl
float3 H = normalize(LightDir + CameraDir);
float NdotH = saturate(dot(Normal, H));
float spec = step(Threshold, NdotH);
return CelColor + spec * SpecColor;
```

### 节点连线

| 引脚 | 连接来源 |
|------|------|
| CelColor | CelShade Custom 的输出 |
| LightDir | LightDir Custom 的输出 |
| CameraDir | CameraVectorWS 节点 |
| Normal | VertexNormalWS 节点 |
| Threshold | ScalarParameter `SpecThreshold`，默认 `0.92` |
| SpecColor | VectorParameter `SpecColor`，默认纯白 `(1,1,1)` |

> `SpecThreshold` 越接近 1，高光圆点越小；`0.88` 左右是比较明显的动漫高光大小。

---

## Layer 3 — 边缘光 Rim Light + 描边

### Custom 节点：RimLight（最终输出节点）

```
Description : RimLight
Output Type : CMOT Float3
Inputs      : CelColor / Normal / CameraDir / RimPower / RimColor / BaseColor
```

```hlsl
// 边缘光
float rim = 1.0 - saturate(dot(Normal, CameraDir));
rim = pow(rim, RimPower);
float rimLine = step(0.5, rim);

// 内描边（法线夹角阈值）
float outline = step(0.85, 1.0 - saturate(dot(Normal, CameraDir)));

// 叠加固有色
float3 result = CelColor * BaseColor + rimLine * RimColor;

// 描边覆盖
return lerp(result, float3(0, 0, 0), outline);
```

### 节点连线

| 引脚 | 连接来源 |
|------|------|
| CelColor | Specular Custom 的输出 |
| Normal | VertexNormalWS |
| CameraDir | CameraVectorWS |
| RimPower | ScalarParameter `RimPower`，默认 `3.0` |
| RimColor | VectorParameter `RimColor`，默认 `(0.4, 0.6, 1.0)` |
| BaseColor | TextureSampleParameter2D `BaseColorTex` 的 RGB 输出 |

RimLight Custom 的输出连到主节点 **Emissive Color**。

### 参数默认值

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| RimPower | Scalar | 3.0 | 边缘光宽度，值越大越细 |
| RimColor | Vector | (0.4, 0.6, 1.0) | 蓝色边缘光 |
| outline step | 内联 | 0.85 | 描边范围，降低则描边更宽 |

---

## 固有色贴图接入

放一个 **TextureSampleParameter2D** 节点：

```
Parameter Name : BaseColorTex
Texture        : 选择角色的 Diffuse 贴图（名称通常含 _D 后缀）
```

将 **RGB 输出**连到 RimLight Custom 节点的 `BaseColor` 引脚。

贴图在代码里以 `CelColor * BaseColor` 的方式乘入，只影响颜色而不影响阴影形状。

---

## 后处理设置

在场景里放 **PostProcessVolume**，勾选 **Infinite Extent (Unbound)**，调整以下参数：

### Color Grading → Global

| 参数 | 值 | 说明 |
|------|------|------|
| Saturation W | 1.4 | 全局饱和度提高，色彩更鲜艳 |
| Contrast W | 1.1 | 对比度微增 |

### Lens → Bloom

| 参数 | 值 | 说明 |
|------|------|------|
| Intensity | 0.2 | 压低 Bloom，避免破坏卡通硬边感 |

### Rendering → Ambient Occlusion

| 参数 | 值 | 说明 |
|------|------|------|
| Intensity | 0 | 关闭 AO，改用手绘 AO 贴图 |

---

## 灯光设置

### DirectionalLight

| 参数 | 值 |
|------|------|
| Intensity | 10 |
| Light Color | (1.0, 0.97, 0.85) 暖黄 |
| Rotation | Pitch -45, Yaw 60（从侧上方打） |

### SkyLight

| 参数 | 值 | 说明 |
|------|------|------|
| Intensity Scale | 0.3 | 大幅降低天光，让阴影更深更戏剧化 |

---

## 参数速查表

所有参数通过材质实例（MaterialInstance）调整，无需重新编译：

| 参数名 | 类型 | 推荐范围 | 效果 |
|--------|------|--------|------|
| ShadowThreshold | Scalar | 0.3 ~ 0.6 | 越大阴影越少 |
| ShadowColor | Vector | 蓝紫系 | 暗部颜色 |
| LitColor | Vector | 暖白系 | 亮部颜色 |
| SpecThreshold | Scalar | 0.85 ~ 0.95 | 越大高光越小 |
| SpecColor | Vector | 白色 | 高光颜色 |
| RimPower | Scalar | 2.0 ~ 8.0 | 越大边缘光越细 |
| RimColor | Vector | 蓝色系 | 边缘光颜色 |
| BaseColorTex | Texture2D | — | 固有色贴图 |

---

## 已知问题与优化方向

### 当前局限

| 问题 | 原因 | 解决方向 |
|------|------|------|
| 描边在低面数模型上有锯齿 | 内描边依赖顶点法线夹角 | 换高面数模型；或改用 Custom Depth 后处理描边 |
| 多光源支持差 | 只读取 DirectionalLight 方向 | 接入 GetSkyLightIntensity 或多光源采样 |
| 面部阴影不自然 | 普通 NdotL 计算 | 实现 SDF Face Shadow Map（参考原神方案） |

### 进阶优化方向

**描边方案升级：Custom Depth 后处理描边**
- 在场景里放 PostProcessVolume
- 材质设置 Custom Depth Pass
- PP 材质里读 SceneDepth vs CustomDepth 做边缘检测
- 优点：描边粗细精确可控，不受模型面数影响

**面部 SDF 阴影（Face Shadow Map）**
- 在 DCC 里预烘焙面部 SDF 阴影贴图
- 运行时根据光源水平角度（atan2）旋转采样
- 左右两张贴图根据光方向混合
- 适合对面部特写有高要求的项目

**Ramp 贴图替代 smoothstep**
- 制作 256×1 的渐变贴图精细控制阴影形状
- 用 NdotL 作为 U 坐标采样
- 可以实现三色阶、软硬混合等复杂效果

---

## 完整节点结构图

```
[Custom: LightDir] ──────────────────────────────────────┐
                                                          ↓ B
[VertexNormalWS] ─────────────────────────────────────→ A [DotProduct]
                  │                                           ↓
                  │                                       [Clamp 0~1]
                  │                                           ↓ NdotL
                  │    [ShadowThreshold] ─────────────→ Threshold
                  │    [ShadowColor] ────────────────→ ShadowColor  [CelShade Custom]
                  │    [LitColor] ──────────────────→ LitColor           ↓ CelColor
                  │                                                       ↓
                  │    [LightDir] ──────────────────→ LightDir
                  ├──→ Normal        [CameraVectorWS] → CameraDir  [Specular Custom]
                  │    [SpecThreshold] ──────────────→ Threshold        ↓ CelColor
                  │    [SpecColor] ──────────────────→ SpecColor
                  │                                                       ↓
                  ├──→ Normal        [CameraVectorWS] → CameraDir
                  │    [RimPower] ──────────────────→ RimPower    [RimLight Custom]
                  │    [RimColor] ──────────────────→ RimColor         ↓
                  │    [BaseColorTex RGB] ──────────→ BaseColor
                  │                                                       ↓
                  └──────────────────────────────────────── (Normal)  Emissive Color
```

---
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/7e5fb387-9a70-463f-b42c-966b68a7b7f7" />


*Made with Unreal Engine 5 + Claude*# ue5-sakuga
