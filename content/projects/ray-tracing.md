---
title: "RayTracingInOneWeekend"
date: 2024-09-14
tags: ["C++", "光线追踪", "渲染"]
categories: ["项目"]
ShowToc: true
summary: "跟着经典教程从零实现 CPU 光线追踪器，支持材质、景深和运动模糊。"
---

## Ray Tracing in One Weekend

[GitHub](https://github.com/hsiang0117/RayTracingInOneWeekend)

基于 Peter Shirley 经典教程系列，从零实现的 CPU 光线追踪器。

### 实现特性

- 🌐 **球体 & 三角形** — 基础几何体求交
- 🪞 **金属 & 玻璃材质** — 反射、折射、Fresnel
- 📷 **景深** — 模拟真实镜头散焦
- 🏃 **运动模糊** — 时间采样
- 📐 **BVH 加速** — 层次包围盒加速求交
- 🖼️ **HDR 输出** — PPM / PNG 图像输出

### 关键实现

```cpp
// Lambertian 漫反射材质
class Lambertian : public Material {
    Color albedo;
public:
    bool scatter(const Ray& r_in, const HitRecord& rec,
                 Color& attenuation, Ray& scattered) const override {
        auto scatter_dir = rec.normal + random_unit_vector();
        scattered = Ray(rec.p, scatter_dir);
        attenuation = albedo;
        return true;
    }
};
```

### 学习收获

- 理解渲染方程及其 Monte Carlo 近似
- 掌握 BVH 空间加速结构
- 实践 C++ 面向对象设计在图形学中的应用
