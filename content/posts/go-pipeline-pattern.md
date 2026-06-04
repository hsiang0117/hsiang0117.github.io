---
title: "PBR 渲染基础：从理论到实现"
date: 2026-04-15
tags: ["图形学", "PBR", "OpenGL", "着色器"]
categories: ["图形学"]
series: ["渲染"]
ShowToc: true
TocOpen: true
summary: "深入理解 Physically Based Rendering 的核心概念，并在 OpenGL 中实现 Cook-Torrance BRDF。"
---

## 什么是 PBR？

PBR (Physically Based Rendering) 是基于物理的渲染方法，遵循能量守恒和微表面理论，生成更真实的材质效果。

核心原则：
- 🔋 **能量守恒** — 出射光能量 ≤ 入射光能量
- 🔬 **微表面理论** — 表面由微小镜面组成
- 🧪 **菲涅尔效应** — 掠射角反射率更高

## Cook-Torrance BRDF

BRDF（双向反射分布函数）描述光线如何在表面反射。Cook-Torrance 是一种广泛使用的微表面 BRDF：

\\[
f_r = \\frac{D \\cdot F \\cdot G}{4 \\cdot (n \\cdot l) \\cdot (n \\cdot v)}
\\]

三个核心项：

### 1. 法线分布函数 D（NDF）

描述微表面法线的分布。常用 **Trowbridge-Reitz GGX**：

```glsl
float DistributionGGX(vec3 N, vec3 H, float roughness) {
    float a = roughness * roughness;
    float a2 = a * a;
    float NdotH = max(dot(N, H), 0.0);
    float denom = NdotH * NdotH * (a2 - 1.0) + 1.0;
    return a2 / (PI * denom * denom);
}
```

### 2. 几何函数 G

模拟微表面自遮挡。使用 **Smith's Schlick-GGX**：

```glsl
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness) {
    float k = (roughness + 1.0) * (roughness + 1.0) / 8.0;
    float G1V = max(dot(N, V), 0.0) / (max(dot(N, V), 0.0) * (1.0 - k) + k);
    float G1L = max(dot(N, L), 0.0) / (max(dot(N, L), 0.0) * (1.0 - k) + k);
    return G1V * G1L;
}
```

### 3. 菲涅尔方程 F

**Schlick 近似**，快速且准确：

```glsl
vec3 FresnelSchlick(float cosTheta, vec3 F0) {
    return F0 + (1.0 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
}
```

## 完整着色器

```glsl
vec3 PBR(vec3 albedo, float metallic, float roughness,
         vec3 N, vec3 V, vec3 L, vec3 lightColor) {

    vec3 H = normalize(V + L);
    vec3 F0 = mix(vec3(0.04), albedo, metallic); // 金属用 albedo 作为 F0

    // Cook-Torrance BRDF
    float NDF = DistributionGGX(N, H, roughness);
    float G   = GeometrySmith(N, V, L, roughness);
    vec3  F   = FresnelSchlick(max(dot(H, V), 0.0), F0);

    vec3 numerator    = NDF * G * F;
    float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.001;
    vec3 specular     = numerator / denominator;

    // 漫反射部分 (Lambert)
    vec3 kD = (vec3(1.0) - F) * (1.0 - metallic);
    vec3 diffuse = kD * albedo / PI;

    // 最终出射辐射率
    float NdotL = max(dot(N, L), 0.0);
    return (diffuse + specular) * lightColor * NdotL;
}
```

## 金属度/粗糙度工作流

| 参数 | 范围 | 说明 |
|------|------|------|
| Albedo | RGB | 基础颜色（金属反射率，非金属散射色）|
| Metallic | [0, 1] | 0=非金属，1=纯金属 |
| Roughness | [0, 1] | 0=镜面，1=完全漫反射 |

## 总结

PBR 的核心就是将物理正确的 BRDF 模型应用到每个像素。掌握了 Cook-Torrance，你就打开了实时渲染的大门。下一步可以探索 IBL（基于图像的照明），用环境贴图替代解析光源。
