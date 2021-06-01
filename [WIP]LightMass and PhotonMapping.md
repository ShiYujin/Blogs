# [WIP]UE4 LightMass源码剖析
[TODO]背景简介

本文将关注在LightMass的烘焙过程，不会过多关注多线程、LightMass与UE4 Editor的数据交互等细节。

本文采用的UE4源码版本是`4.26`，具体的commit id为：`01c5cc4fe97f709b05e9039ecd806be00380aeee`。

UE4 LightMass的完整实现都在源码目录`Engine/Source/Programs/UnrealLightmass`下。目录结构如下：

```
Engine/Source/Programs/UnrealLightmass
├── Private
│   ├── CPUSolver
│   │   ├── CPUSolver.cpp
│   │   └── CPUSolver.h
│   ├── ImportExport                            // <-- 数据的导入导出
│   │   ├── 3DVisualizer.cpp
│   │   ├── 3DVisualizer.h
│   │   ├── Exporter.cpp
│   │   ├── Exporter.h
│   │   ├── Importer.cpp
│   │   ├── Importer.h
│   │   ├── LightmassScene.cpp
│   │   ├── LightmassScene.h
│   │   ├── LightmassSwarm.cpp
│   │   ├── LightmassSwarm.h
│   │   ├── Material.cpp
│   │   ├── Material.h
│   │   ├── Mesh.cpp
│   │   └── Mesh.h
│   ├── Launch
│   │   ├── UnitTest.cpp
│   │   ├── UnitTest.h
│   │   ├── UnrealLightmass.cpp
│   │   └── UnrealLightmass.h
│   ├── Lighting
│   │   ├── AdaptiveVolumetricLightmap.cpp
│   │   ├── BSP.cpp
│   │   ├── BSP.h
│   │   ├── BuildOptions.h
│   │   ├── Collision.cpp
│   │   ├── Collision.h
│   │   ├── Embree.cpp
│   │   ├── Embree.h
│   │   ├── FinalGather.cpp
│   │   ├── FluidSurface.cpp
│   │   ├── FluidSurface.h
│   │   ├── GatheredLightingSample.h
│   │   ├── Landscape.cpp
│   │   ├── Landscape.h
│   │   ├── Lighting.h
│   │   ├── LightingCache.cpp
│   │   ├── LightingCache.h
│   │   ├── LightingMesh.cpp
│   │   ├── LightingMesh.h
│   │   ├── LightingSystem.cpp
│   │   ├── LightingSystem.h
│   │   ├── LightingSystem.inl
│   │   ├── LightmapData.cpp
│   │   ├── LightmapData.h
│   │   ├── Mappings.cpp
│   │   ├── Mappings.h
│   │   ├── MonteCarlo.cpp
│   │   ├── MonteCarlo.h
│   │   ├── PhotonMapping.cpp
│   │   ├── PrecomputedVisibility.cpp
│   │   ├── Radiosity.cpp
│   │   ├── Raster.h
│   │   ├── SampleVolume.cpp
│   │   ├── StaticMesh.cpp
│   │   ├── StaticMesh.h
│   │   ├── Texture.h
│   │   ├── TextureMapping.cpp
│   │   ├── TextureMappingSetup.h
│   │   └── VolumeDistanceField.cpp
│   └── LightmassCore
│       ├── LMCore.cpp
│       ├── LMCore.h
│       ├── Math
│       │   ├── LMCollision.h
│       │   ├── LMMath.cpp
│       │   ├── LMMath.h
│       │   ├── LMMathSSE.h
│       │   ├── LMOctree.h
│       │   ├── LMkDOP.h
│       │   ├── MersennePrimeTwister.tps
│       │   └── SFMT.cpp
│       ├── Misc
│       │   ├── LMDebug.cpp
│       │   ├── LMDebug.h
│       │   ├── LMHelpers.h
│       │   ├── LMStats.h
│       │   ├── LMThreading.cpp
│       │   └── LMThreading.h
│       └── Templates
│           └── LMQueue.h
├── Public
│   ├── FluidExport.h
│   ├── ImportExport.h
│   ├── LightmassPublic.h
│   ├── MaterialExport.h
│   ├── MeshExport.h
│   └── SceneExport.h
├── UnrealLightmass.Build.cs
└── UnrealLightmass.Target.cs
```

## PhotonMapping介绍

## UE4 LightMass介绍

### 相关类及其功能
控制类：
`FLightmassSwarm`
烘焙主类：
`FStaticLightingSystem`
工具类：
`FGlobalStatistics`

### 配置
[TODO]参数解析在`LightmassMain`

### 数据统计
[TODO]`FGlobalStatistics`

### 场景组织
[TODO]Scene
Q: importance volume

### 子线程
<!-- `FStaticLightingThreadRunnable` -->

`FDirectPhotonEmittingThreadRunnable`
`FIndirectPhotonEmittingThreadRunnable`

`FIrradiancePhotonMarkingThreadRunnable`
`FMappingProcessingThreadRunnable` - `StaticLightingTask_CacheIrradiancePhotons`
`FIrradiancePhotonCalculatingThreadRunnable`

`FMappingProcessingThreadRunnable` - `StaticLightingTask_FinalizeSurfaceCache`

`FMappingProcessingThreadRunnable` - `StaticLightingTask_ProcessMappings`



## Baking Process
整个烘焙过程主要分为三个部分：

- 导入数据
- 烘焙准备
- 渲染数据（烘焙）
- 导出数据

### 导入数据
数据导入部分的逻辑主要在类`FStaticLightingSystem`的构造函数里，前前后后一共有四百多行。


### 烘焙准备
烘焙部分的逻辑主要在`FStaticLightingSystem::MultithreadProcess`函数下。如名字所示，这里会调用多线程完成整个烘焙任务。

准备工作：
1. 函数`CacheSamples`：保存采样缓存，包括
   - `CachedHemisphereSamples`
   - `CachedHemisphereSampleUniforms`
   - `CachedHemisphereSamplesForRadiosity`
   - `CachedHemisphereSamplesForRadiosityUniforms`
   - `CachedVolumetricLightmapUniformHemisphereSamples`
   - `CachedVolumetricLightmapUniformHemisphereSampleUniforms`
   - `CachedVolumetricLightmapVertexOffsets`
这些变量都保存在类`FStaticLightingSystem`下面。具体的采样算法实现在`Private/Lighting/MonteCarlo.h`下。可以看出，这里缓存了球面采样和光源采样，能够减少后面的重复计算。
2. 函数`EmitPhotons`：发射光子
   1. `EmitDirectPhotons`
      1. 根据每个光源发射出的direct photon采样
      2. 子线程`FDirectPhotonEmittingThreadRunnable`
      3. 输出`FDirectPhotonEmittingOutput`，结果会转存到`DirectPhotonMap`和`IrradiancePhotonMap`
   2. `EmitIndirectPhotons`
      1. 根据每个光源的power采样
      2. 子线程`FIndirectPhotonEmittingThreadRunnable`
      3. 输出`FIndirectPhotonEmittingOutput`，转存到`FirstBouncePhotonMap`，`FirstBounceEscapedPhotonMap`和`SecondBouncePhotonMap`
   3. `CacheIrradiancePhotons`
      1. 缓存IrradiancePhotons
3. [TODO]SetupRadiosity / RunRadiosityIterations
4. `FinalizeSurfaceCache`
   1. 光栅化场景
   2. 遍历所有texture的每个像素，根据缓存的IrradiancePhotons计算Irradiance。这一步已经将光子信息转化为了Irradiance信息，后面会根据Irradiance估计每个像素点的颜色。

#### EmitDirectPhotons
函数`FStaticLightingSystem::EmitDirectPhotonsWorkRange`

1. 抽样光源，根据每个光源发射的direct photon数量进行抽样
2. 抽样方向
3. 光线与场景求交
   1. miss或者交点为背面 - 退出
4. 计算光线衰减
   1. 光线能量过低 - 退出
5. 生成光子
   1. 判断光子是否在importance bounds内部以及光子数量是否超过限制 - 退出
6. 将光子保存，加入`DirectPhotos`
   1. 抽样 - continue
   2. 创建IrradiancePhoton，并加入`IrradiancePhotons`
7. 判断间接光路径是否足够
   1. 是 - 退出
8. 求交点的BRDF颜色
9.  采样一个反射方向，射出光线（半球采样）
10. 新的光线与场景求交
    1.  miss或者交点为背面 - 退出
11. 保存路径`IndirectPathRays`用于计算间接光照

整理一下整个过程保存的结果：
- `DirectPhotons`: step 6
- `IrradiancePhotons`: step 6.2
- `IndirectPathRays`: step 11

#### EmitIndirectPhotons
函数`FStaticLightingSystem::EmitIndirectPhotonsWorkRange`

1. 抽样光源，根据每个光源的功率（power）进行抽样
2. 抽样方向
   1. 如果该光源有保存的`IndirectPathRays`，就从中抽样
   2. 如果没有，均匀采样光源
3. 判断路径是否经过importance volume
   1. 不经过 - 退出
   2. 经过 - clip到importance volume
4. 光线与场景求交
   1. miss或者交点为背面 - 退出
   2. 计算光线衰减
      1. 光线能量过低 - 退出
   3. 判断与场景的交点是否在importance bounds内部以及光子数量是否在限制内
      1. 生成光子
      2. 判断反射次数
         1. 两次反射 - 加入 `FirstBouncePhotons`
         2. 超过两次反射 - `SecondBouncePhotons`
      3. 采样决定是否生成Irradiance光子
         1.  是 - 生成并加入`IrradiancePhotons`
   4. 判断反弹次数
      1. 超过阈值 - 退出
   5. 求交点的BRDF颜色
   6. 采样一个反射方向，射出光线（半球采样）
   7. 判断反射次数
      1. 一次反射 - 更新光子颜色
      2. 两次反射 - Russian Roulette采样（概率为出射颜色与入射颜色的绿色通道的商）
         1. 未通过 - 退出
   8. [TODO]Clip到importance volume
   9. 光线与场景求交，返回4.1
5. 对于一次反射的光子
   1. [TODO]根据一定概率进行VolumeLighting
   2. 加入`FirstBounceEscapedPhotons`

整理一下整个过程保存的结果：
- `FirstBouncePhotons`: step 4.3.2.1
- `SecondBouncePhotons`: step 4.3.2.2
- `IrradiancePhotons`: step 4.3.3.1
- `FirstBounceEscapedPhotons`: step 5.2

#### CacheIrradiancePhotons
缓存IrradiancePhotons过程分为三步：
`MarkIrradiancePhotons`：遍历所有的`IrradiancePhotons`，标记每个`IrradiancePhotons`的周围是否有direct photon
`CacheIrradiancePhotons`：[TODO]先光栅化每一张TextureMapping，然后将IrradiancePhotons的缓存写入到TextureMapping中，计算方式是遍历texture的每一个像素，计算该像素对应的空间周围的IrradiancePhotons，将Photons保存下来，并标记为已使用。
`CalculateIrradiancePhotons`：计算IrradiancePhotons的照度，光子照度 = 直接光子照度 + 第一次反弹照度 + 第二次及后续反弹照度，irradiance的计算都是调用函数`CalculatePhotonIrradiance`，会查询该点附近的所有指定光子（直接光子or一次反弹or二次反弹光子），并且只计算可见光子的贡献。输出：`mIrradiancePhotons`，将光子信息缓存到了里`mIrradiancePhotons`。

### 渲染数据
入口：`FStaticLightingSystem::ThreadLoop`函数，由进程`FMappingProcessingThreadRunnable` - `StaticLightingTask_ProcessMappings`发起。

1. 如果烘焙没有完成
2. 检查已有的task是否完成
3. 从SWarm请求下一个任务
   1. 收到退出信号（用户终止任务）或所有任务已经完成 - 退出
4. 根据任务类型启动任务
   1. 生成TextureMapping：`FStaticLightingSystem::ProcessTextureMapping`
      1. 光栅化
      2. [TODO]遍历光源
         1. 计算直接光照，有几个不同的函数
            1. `CalculateDirectAreaLightingTextureMapping`
            2. `CalculateDirectLightingTextureMappingFiltered`
            3. `CalculateDirectSignedDistanceFieldLightingTextureMappingTextureSpace`
      3. `CalculateIndirectLightingTextureMapping`：计算间接光照
      4. `ViewMaterialAttributesTextureMapping`
      5. `ColorInvalidLightmapUVs`
      6. `PadTextureMapping`
      7. ...
   2. `BeginCalculateVolumeSamples`
   3. `CalculatePrecomputedVisibility`
   4. `CalculateAdaptiveVolumetricLightmap`
   5. `BeginCalculateVolumeDistanceField`
   6. `CalculateStaticShadowDepthMap`

#### CalculateDirectAreaLightingTextureMapping
计算光照，与前面计算的Photons关系不大。
用到了shadowmap，根据光源采样计算阴影系数，这里采样光源用的是前面cache的采样点。
粗粒度：采样`ShadowSettings.NumShadowRays`个点，bounce次数越多采样点越少（`ShadowSettings.NumBounceShadowRays / BounceNumber`）。
细粒度：采样`ShadowSettings.NumPenumbraShadowRays`个点，同样bounce次数越多采样点越少。只有位于半阴影(penumbra)区域的点才会进行细粒度阴影采样。
遍历整个Texture，根据阴影系数计算该点的颜色，保存在`LightMapData`中，此时会用RGB颜色计算二阶球谐（SH）系数。

#### CalculateIndirectLightingTextureMapping
计算多次反射、天空盒的光线。
先将所有task生成并保存，然后各子线程查询task并执行。
使用Final Gather + IrradianceCache 生成适量的Final Gather Sample Cache,然后再使用这些FG Sample Cache插值求解出最终的间接光照。
Final Gather代替了kNN方法，可以较好地消除噪声，但是时间开销比较大。
FG会均匀地向半球空间发射光线，用这些方向上的irradiance cache平均作为该点收到的irradiance。为了做到这一点，UE4用重要性采样+Irradiance gradient的方法来计算这些采样光想上的irradiance。
[TODO]重要性采样：`IncomingRadianceAdaptive`，用四叉树结构来refine
[TODO]Irradiance Gradient：`CalculateIrradianceGradients` - Irradiance Gradients
[TODO]错误估计 - An Approximate Global Illumination System for Computer Generated Films


### 导出数据
颜色，亮度，最大入射光线贡献方向

### 几处光栅化操作
第一处出现在`EmitPhotons - CacheIrradiancePhotons - CacheIrradiancePhotons - CacheIrradiancePhotonsTextureMapping`中，函数为`RasterizeToSurfaceCacheTextureMapping(TextureMapping, bDebugThisMapping, TexelToVertexMap);`，只计算importance volume之中的IrradiancePhotons。
第二处出现在`FinalizeSurfaceCache - FinalizeSurfaceCacheTextureMapping`中，函数为`RasterizeToSurfaceCacheTextureMapping(TextureMapping, bDebugThisMapping, TexelToVertexMap);`，计算完整的照度。
第三处出现在`ProcessTextureMapping - SetupTextureMapping`中，光栅化精度更高（5->7）。


## LightMap介绍


## 参考资料
[1] Jiff大神在知乎专栏撰写的UE4 LightMass源码分析系列
[1.1] [UE4 Lightmap格式解析](https://zhuanlan.zhihu.com/p/68754084)
[1.2] [UE4 Lightmap的解码](https://zhuanlan.zhihu.com/p/69284248)
[1.3] [Lightmass源码分析之 与Swarm交互](https://zhuanlan.zhihu.com/p/74401275)
[1.4] [LightMass源码分析之中间数据格式](https://zhuanlan.zhihu.com/p/74480132)
[1.5] [Lightmass源码分析之 经典Photon Mapping算法介绍](https://zhuanlan.zhihu.com/p/73749698)
[1.6] [LightMass源码分析之光子追踪实现](https://zhuanlan.zhihu.com/p/75578613)
[1.7] [LightMass源码分析之光照评估](https://zhuanlan.zhihu.com/p/77461850)
