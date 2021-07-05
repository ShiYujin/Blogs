# UE源码学习笔记

## NewObject
创建新的类为什么用模板`NewObject`？
例子：Engine\Source\Runtime\Engine\Private\PreviewScene.cpp
```C++
	PreviewWorld = NewObject<UWorld>(GetTransientPackage(), NAME_None, NewObjectFlags);

```
`NewObject`的定义在：Engine\Source\Runtime\CoreUObject\Public\UObject\UObjectGlobals.h
```C++
template< class T >
FUNCTION_NON_NULL_RETURN_START
	T* NewObject(UObject* Outer = (UObject*)GetTransientPackage())
FUNCTION_NON_NULL_RETURN_END
{
	// Name is always None for this case
	FObjectInitializer::AssertIfInConstructor(Outer, TEXT("NewObject with empty name can't be used to create default subobjects (inside of UObject derived class constructor) as it produces inconsistent object names. Use ObjectInitializer.CreateDefaultSubobject<> instead."));

	FStaticConstructObjectParameters Params(T::StaticClass());
	Params.Outer = Outer;
	return static_cast<T*>(StaticConstructObject_Internal(Params));
}

template< class T >
FUNCTION_NON_NULL_RETURN_START
	T* NewObject(UObject* Outer, FName Name, EObjectFlags Flags = RF_NoFlags, UObject* Template = nullptr, bool bCopyTransientsFromClassDefaults = false, FObjectInstancingGraph* InInstanceGraph = nullptr)
FUNCTION_NON_NULL_RETURN_END
{
	if (Name == NAME_None)
	{
		FObjectInitializer::AssertIfInConstructor(Outer, TEXT("NewObject with empty name can't be used to create default subobjects (inside of UObject derived class constructor) as it produces inconsistent object names. Use ObjectInitializer.CreateDefaultSubobject<> instead."));
	}

	FStaticConstructObjectParameters Params(T::StaticClass());
	Params.Outer = Outer;
	Params.Name = Name;
	Params.SetFlags = Flags;
	Params.Template = Template;
	Params.bCopyTransientsFromClassDefaults = bCopyTransientsFromClassDefaults;
	Params.InstanceGraph = InInstanceGraph;
	return static_cast<T*>(StaticConstructObject_Internal(Params));
}

```

## 智能指针

## 配置保存.ini

## UE的全局变量

## Shading model
### 如何添加一个新的Shading model
确定新的Shading Model需要哪些参数

在Material Editor中添加新的Shading Model，主要添加这样四个enum/宏：
- MSM_AdvancedCloth
- PIXEL_INSPECTOR_SHADINGMODELID_ADVANCEDCLOTH
- MATERIAL_SHADINGMODEL_ADVANCEDCLOTH
- SHADINGMODELID_ADVANCEDCLOTH

可以找一个已有的模型作为样本，例如CLOTH模型，通过全局搜索找出它们的位置，然后照猫画虎加上自己的Shading Model。

MSM_AdvancedCloth：
1. 定义：`Engine\Source\Runtime\Engine\Classes\Engine\EngineTypes.h`:`enum EMaterialShadingModel`
```C++
UENUM()
enum EMaterialShadingModel
{
	//// ...
	MSM_AdvancedCloth			UMETA(DisplayName="MSM_AdvancedCloth")			/*=12*/,
	//// ...
	MSM_MAX
};
```

2. 设置Shading Model的描述，其实就是将enum定义转换为字符串：`Engine\Source\Runtime\Engine\Private\Materials\MaterialShader.cpp`:`GetShadingModelString`
```C++
/** Converts an EMaterialShadingModel to a string description. */
FString GetShadingModelString(EMaterialShadingModel ShadingModel)
{
	FString ShadingModelName;
	switch(ShadingModel)
	{
		//// ...
		case MSM_AdvancedCloth:		ShadingModelName = TEXT("MSM_AdvancedCloth"); break;
		//// ...
	}
	return ShadingModelName;
}
```

3. 设置Shading Model下拉菜单的合法选项：`Engine\Source\Runtime\Engine\Classes\Materials\MaterialExpressionShadingModel.h`:
```C++
UPROPERTY(EditAnywhere, Category = ShadingModel, meta = (ValidEnumValues = "MSM_DefaultLit, MSM_Subsurface, MSM_PreintegratedSkin, MSM_ClearCoat, MSM_SubsurfaceProfile, MSM_TwoSidedFoliage, MSM_Hair, MSM_Cloth, MSM_Eye, MSM_AdvancedCloth"))
TEnumAsByte<enum EMaterialShadingModel> ShadingModel = MSM_DefaultLit;
```

4. 设置新的Shading Model有哪些参数需要隐藏：`Engine\Source\Editor\PixelInspector\Private\PixelInspectorDetailsCustomization.cpp`:`FPixelInspectorDetailsCustomization::CustomizeDetails`:`switch (MaterialShadingModel)`
```C++
void FPixelInspectorDetailsCustomization::CustomizeDetails(IDetailLayoutBuilder& DetailBuilder)
{
	//// ...
	switch (MaterialShadingModel)
	{
		//// ...
		case MSM_AdvancedCloth:
		{
			DetailBuilder.HideProperty(SubSurfaceProfileProp);
			DetailBuilder.HideProperty(OpacityProp);
			DetailBuilder.HideProperty(ClearCoatProp);
			DetailBuilder.HideProperty(ClearCoatRoughnessProp);
			DetailBuilder.HideProperty(WorldNormalProp);
			DetailBuilder.HideProperty(BackLitProp);
			DetailBuilder.HideProperty(EyeTangentProp);
			DetailBuilder.HideProperty(IrisMaskProp);
			DetailBuilder.HideProperty(IrisDistanceProp);
		}
		break;
	}
	//// ...
}
```

5. 设置Material Editor中需要active的变量：`Engine\Source\Runtime\Engine\Private\Materials\Material.cpp`:`IsPropertyActive_Internal`
```C++
static bool IsPropertyActive_Internal(EMaterialProperty InProperty,
	EMaterialDomain Domain,
	EBlendMode BlendMode,
	FMaterialShadingModelField ShadingModels,
	ETranslucencyLightingMode TranslucencyLightingMode,
	EDecalBlendMode DecalBlendMode,
	bool bBlendableOutputAlpha,
	bool bHasTessellation,
	bool bHasRefraction,
	bool bIsMaskEdgeSoften,
	bool bUseSimplifyClearCoat,
	ESMobileMaterialResType::Type SMobileResType,
	bool bUsesShadingModelFromMaterialExpression)
{
	//// ...
	switch (InProperty)
	{
	//// ...
	case MP_SubsurfaceColor:
		//// ...
		if (ShadingModels.HasShadingModel(MSM_AdvancedCloth))
		{
			Active = true;
		}
		//// ...
		break;
	case MP_CustomData0:
		//// ...
		if (ShadingModels.HasShadingModel(MSM_AdvancedCloth))
		{
			Active = true;
		}
		//// ...
		break;
	//// ...
	}
	//// ...
}
```

6. 对于Shading Model的各个参数，如果需要修改它在Material Editor中展示的名称，可以在这里修改：`Engine\Source\Runtime\Engine\Private\Materials\MaterialShared.cpp`:`FMaterialAttributeDefinitionMap::GetAttributeOverrideForMaterial`，例如我们希望`SubsurfaceColo`r这一栏显示的名称变为`Fuzz Color`，以及`CustomData0`一栏变为`Cloth`：
```C++
FText FMaterialAttributeDefinitionMap::GetAttributeOverrideForMaterial(const FGuid& AttributeID, UMaterial* Material)
{
	//// ...
	switch (Property)
	{
	//// ...
	case MP_SubsurfaceColor:
		//// ...
		CustomPinNames.Add({MSM_AdvancedCloth, "Fuzz Color"});
		//// ...
	case MP_CustomData0:
		//// ...
		CustomPinNames.Add({MSM_AdvancedCloth, "Cloth"});
		//// ...
	}
	//// ...
}
```

7. 设置对CustomData的解析，CustomData包括？？？`Engine\Source\Editor\PixelInspector\Private\PixelInspectorResult.cpp`:`PixelInspectorResult::DecodeCustomData`:`switch (ShadingModel)`
```C++
void PixelInspectorResult::DecodeCustomData(FVector4 InCustomData)
{
	switch (ShadingModel)
	{
	//// ...
	case EMaterialShadingModel::MSM_AdvancedCloth:
	{
		SubSurfaceColor = FVector(InCustomData.X, InCustomData.Y, InCustomData.Z);
		Cloth = InCustomData.W;
	}
	break;
	//// ...
	};
}
```

8. 如果是支持Subsurface的Shading Model，还需要在这里设置一下：`Engine\Source\Runtime\Engine\Public\MaterialShared.h`:`bool IsSubsurfaceShadingModel`:
```C++
inline bool IsSubsurfaceShadingModel(FMaterialShadingModelField ShadingModel)
{
	return ShadingModel.HasShadingModel(MSM_Subsurface) || 
		ShadingModel.HasShadingModel(MSM_PreintegratedSkin) ||
		ShadingModel.HasShadingModel(MSM_SubsurfaceProfile) || 
		ShadingModel.HasShadingModel(MSM_TwoSidedFoliage) ||
		ShadingModel.HasShadingModel(MSM_Cloth) || 
		ShadingModel.HasShadingModel(MSM_Eye) ||
		ShadingModel.HasShadingModel(MSM_AdvancedCloth);
}
```

9. 设置Material shader的编译状态：`Engine\Source\Runtime\Engine\Private\Materials\MaterialShader.cpp`:`UpdateMaterialShaderCompilingStats`
```C++
/** Called for every material shader to update the appropriate stats. */
void UpdateMaterialShaderCompilingStats(const FMaterial* Material)
{
	//// ...
	if (ShadingModels.HasOnlyShadingModel(MSM_Unlit))
	{
		INC_DWORD_STAT_BY(STAT_ShaderCompiling_NumUnlitMaterialShaders, 1);
	}
	else if (ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_Subsurface, MSM_PreintegratedSkin, MSM_ClearCoat, MSM_Cloth, MSM_SubsurfaceProfile, MSM_TwoSidedFoliage, MSM_SingleLayerWater, MSM_ThinTranslucent, MSM_AdvancedCloth}))
	{
		INC_DWORD_STAT_BY(STAT_ShaderCompiling_NumLitMaterialShaders, 1);
	}
	//// ...
}
```

10. 设置HLSL编译时输入shader的环境变量，这个可以理解为从CPP文件到shader文件的参数传递：`Engine\Source\Runtime\Engine\Private\Materials\HLSLMaterialTranslator.cpp`:`FHLSLMaterialTranslator::GetMaterialEnvironment`
```C++
void FHLSLMaterialTranslator::GetMaterialEnvironment(EShaderPlatform InPlatform, FShaderCompilerEnvironment& OutEnvironment)
{
	//// ...
	if (ShadingModels.IsLit())
	{
	//// ...
		////SpeedEngine: Add by yujinshi(new shading model)
		if (ShadingModels.HasShadingModel(MSM_AdvancedCloth))
		{
			OutEnvironment.SetDefine(TEXT("MATERIAL_SHADINGMODEL_ADVANCEDCLOTH"), TEXT("1"));
			NumSetMaterials++;
		}
	//// ...
	}
	//// ...
}
```


PIXEL_INSPECTOR_SHADINGMODELID_ADVANCEDCLOTH：
1. 定义`Engine\Source\Editor\PixelInspector\Private\PixelInspectorResult.h`开头
```C++
#define PIXEL_INSPECTOR_SHADINGMODELID_ADVANCEDCLOTH 12
```

2. 转换为`EMaterialShadingModel`，`Engine\Source\Editor\PixelInspector\Private\PixelInspectorResult.cpp`:`PixelInspectorResult::DecodeShadingModel`:`switch (ShadingModelId)`
```C++
EMaterialShadingModel PixelInspectorResult::DecodeShadingModel(float InPackedChannel)
{
	//// ...
	switch (ShadingModelId)
	{
	//// ...
	case PIXEL_INSPECTOR_SHADINGMODELID_ADVANCEDCLOTH:
		return EMaterialShadingModel::MSM_AdvancedCloth;
	//// ...
	};
	//// ...
}
```

至此，C++部分文件已经改完了，接下来改shader。此时可以先让VS编译一下UE4项目了。

MATERIAL_SHADINGMODEL_ADVANCEDCLOTH:

1. 数据初始化，根据需要将传入Pixel Shader的数据进行整合、处理：`Engine\Shaders\Private\AcMobileBasePassShading_Character.ush`:`void InitAcShadingData`
```C++
void InitAcShadingData(in out FAcShadingData Data,
	in const FMaterialPixelParameters MaterialParameters,
	in const FPixelMaterialInputs PixelMaterialInputs)
{
	//// ...
#if MATERIAL_SHADINGMODEL_ADVANCEDCLOTH
	half3 ClothSpecular = GetMaterialSubsurfaceData(PixelMaterialInputs).xyz;
	Data.DiffuseColor = Data.BaseColor - Data.BaseColor * Data.Metallic;	// 1 mad
	Data.SpecularColor = (ClothSpecular - ClothSpecular * Data.Metallic) + Data.BaseColor * Data.Metallic;	// 2 mad
	//// ...
#endif
	//// ...
}
```

2. 设置Shading Model的一些宏，用于告诉Vertex和Pixel Shader需要对这个Shading Model做哪些处理：`Engine\Shaders\Private\BasePassCommon.ush`，例如Cloth需要将CustomData写入GBuffer，那么修改这个宏：
```C++
#define WRITES_CUSTOMDATA_TO_GBUFFER		(USES_GBUFFER && (MATERIAL_SHADINGMODEL_SUBSURFACE || MATERIAL_SHADINGMODEL_PREINTEGRATED_SKIN || MATERIAL_SHADINGMODEL_SUBSURFACE_PROFILE || MATERIAL_SHADINGMODEL_CLEAR_COAT || MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE || MATERIAL_SHADINGMODEL_HAIR || MATERIAL_SHADINGMODEL_CLOTH || MATERIAL_SHADINGMODEL_EYE || MATERIAL_SHADINGMODEL_ADVANCEDCLOTH))
```

3. 为Shading Model准备GBuffer：`Engine\Shaders\Private\ShadingModelsMaterial.ush`:`void SetGBufferForShadingModel`，这一步的作用是生成GBuffer结构，用于后续的渲染，根据输入的参数组合GBuffer就可以了，公用的部分不能改，主要是处理当前Shading Model特异的部分。
```C++
void SetGBufferForShadingModel(
	in out FGBufferData GBuffer, 
	in const FMaterialPixelParameters MaterialParameters,
	const float Opacity,
	const half3 BaseColor,
	const half  Metallic,
	const half  Specular,
	const float Roughness,
	const float Anisotropy,
	const float3 SubsurfaceColor,
	const float SubsurfaceProfile,
	const float Dither,
	const uint ShadingModel)
{
	//// ...
#if MATERIAL_SHADINGMODEL_ADVANCEDCLOTH
	else if (ShadingModel == SHADINGMODELID_ADVANCEDCLOTH)
	{
		GBuffer.CustomData.rgb = EncodeSubsurfaceColor(SubsurfaceColor);
		GBuffer.CustomData.a = saturate( GetMaterialCustomData0(MaterialParameters) );	// Cloth
		GBuffer.IndirectIrradiance *= 1 - GBuffer.CustomData.a;
	}
#endif
	//// ...
}
```


4. 修改DeferredShading BasePass的Pixel Shader，这里需要利用宏`MATERIAL_SHADINGMODEL_ADVANCEDCLOTH`和enum`SHADINGMODELID_ADVANCEDCLOTH`来规划路径，指定新加入的Shading Model需要有哪些操作：`Engine\Shaders\Private\BasePassPixelShader.usf`:`void FPixelShaderInOut_MainPS`:


SHADINGMODELID_ADVANCEDCLOTH:


### UE4的Preintegrated Skin Shading MOdel
相对于GPU Pro 2中介绍的完整的Preintegrated Skin Shading，UE4的model

1. 只在Shadow域进行了预积分，Diffuse和Specular都采用了DefaultLit，也就是跟普通渲染没啥区别
2. UE4的Preintegrated Texture是一张灰度图，具体的用法是用NoL和Opacity（实际是1-Opacity）对它进行采样，然后乘上Subsurface Color
3. UE4的这个模型其实是一个高效但是并不精确的方法，因此Epic官方推荐使用Subsurface Profile模型渲染高精度的皮肤

```C++
FDirectLighting PreintegratedSkinBxDF( FGBufferData GBuffer, half3 N, half3 V, half3 L, float Falloff, float NoL, FAreaLight AreaLight, FShadowTerms Shadow )
{
	FDirectLighting Lighting = DefaultLitBxDF( GBuffer, N, V, L, Falloff, NoL, AreaLight, Shadow );
	
	float3 SubsurfaceColor = ExtractSubsurfaceColor(GBuffer);
	float Opacity = GBuffer.CustomData.a;

	float3 PreintegratedBRDF = Texture2DSampleLevel(View.PreIntegratedBRDF, View.PreIntegratedBRDFSampler, float2(saturate(dot(N, L) * .5 + .5), 1 - Opacity), 0).rgb;
	Lighting.Transmission = AreaLight.FalloffColor * Falloff * PreintegratedBRDF * SubsurfaceColor;

	return Lighting;
}
```

### GBuffer介绍以及如何修改它

### Shading model无法正常工作的bug
用visualization工具显示一下shading model，看是否是正确的shading model

Engine\Source\Runtime\Engine\Classes\Engine\EngineTypes.h 下面定义的`EMaterialShadingModel`序号，必须和Engine\Shaders\Private\ShadingCommon.ush定义的`SHADINGMODELID_XXX`一致

