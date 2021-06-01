# SDFDDGI

## SDF

## DDGI

### GI
GI(Global illumination)是和Direct illumination相对的一个概念。在实时渲染的时候，直接光照称为Dirct illumination，只需要渲染点本身的信息（BRDF）以及光源信息即可；而间接光照不仅仅需要渲染点自身的信息，还需要追踪光线射出去后在其他物体表面的反射，即需要全局的材质信息，这就是Global illumination。

为什么LI比较容易而GI这么难？主要是光栅化管线的限制，光栅化只能获取渲染像素点自身的信息（fragment shader），而无法获取场景的全局信息（例如阴影、透明等效果）。所以除了光线追踪的思路以外，还有很多在光栅化上做GI的尝试，例如SSGI、RTXGI、DDGI等。

GI算法有很多，主流的有两种，Lightmap和Probes。他们都是基于预计算的，不同的是lightmap通过纹理保存GI信息，而probes通过probe保存GI信息。

- Light maps
- Irradiance probes/voxels
- Virtual point lights
- Reflective shadow maps
- Light propagation volumes
- Sparse voxel cone tracing
- Denoised ray tracing

<!-- - Baked lightmap
  - Precomputed radiance transfer（PRT）
- Baked irradiance probes
  - Light field probes（LFP） -->

预计算方法的缺陷很明显，那就是只能对静态场景做，而且预计算部分的开销也不小。

### Irradiance probes

- 生成probe
  - 如何放置probe，以及probe保存什么
- 用probe渲染
  - 计算p周围八个probe的权重，加权平均。
  - 权重的计算？

Irradiance probes方法的主要缺陷在于：

1. 预计算开销大
2. 不支持动态场景、光源
3. Light/Shadow leak

### DDGI
如果希望对动态物体、光源渲染GI，那么就必须每帧更新GI数据。

- 生成probe
- 更新probe（单独thread）
  - 如何更新probe
- 用probe渲染


最常用的方法是NVIDIA提出的DDGI，它是对基于probe的算法Light field probes（LFP）的一种改进，通过实时更新Probe的方式，做到对动态场景的支持。同时通过在Probe里保存了深度信息，解决light leak的问题。

DDGI将GI分为两部分：

- Glossy GI：ray tracing
- Diffuse GI：bake it or fake it


## Reference

