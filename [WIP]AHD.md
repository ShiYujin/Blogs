# [WIP]AHD

AHD（Ambient hightlight direction）是一种光照烘焙技术，简单来说，烘焙完成后在保存lightmap时，AHD会将irradiance分解为ambient和highlight两部分分开存储。这样虽然存储压力增加了，但是在实时渲染时，可以利用highlight来生成高光色，规避了光照烘焙只能烘焙漫反射光的问题，使得渲染结果更真实。

## Ambient and Highlight Direction
传统的AHD会在烘焙完成时，保存三个变量：

- 环境光项$C_{amb}$
- 主光项$C_{dir}$
- 主光的方向$\vec{L}$

环境光项类似于漫反射，表示

