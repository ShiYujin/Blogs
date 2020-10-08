# PBR Workflow
本文介绍两种常用的PBR材质工作流：金属/粗糙度工作流（Metal/Roughness）和镜面反射/光泽度工作流（Specular/Glossiness）。这两种工作流都可以用来制作一个支持PBR的材质，并用PBR渲染出逼真的效果，它们没有优劣之分，只是对同一种现象的不同实现。理解了这两种工作流，才可以更好的理解PBR着色器的一些参数及其使用背后的原理。

本文参考了Substance出版的*The PBR Guide*，原文连接在[1]，如果想看中文版可以参考[2]。

## PBR需要什么
无论是金属/粗糙度工作流还是镜面反射/光泽度工作流，他们的目的都是进行PBR渲染，获得尽可能真实的渲染结果。[前文](https://blog.csdn.net/u010281174/article/details/108939572)介绍了基于微表面理论的BRDF模型需要这样几个参数：

- 粗糙度或者光滑度；
- $F_{0}$，入射角为$0\degree$时的反射光线radiance占入射光线radiance的比例；

除此之外，有一个默认参数：$F_{90}=1$，入射角为$90\degree$时的反射光线radiance占入射光线radiance的比例。

$F_{0}$的主要作用是，决定DiffuseColor和SpecularColor。
$F_{0}$的值与物体是否为金属有很大关系，要说明白这一点，先看一下金属、非金属的区别。

<!-- 以上三个参数都是BRDF项所需的。为了计算出射光线的颜色，还需要`DiffuseColor`和`SpecularColor`，分别表示材质的Diffuse颜色和Specular颜色。
 -->
### 金属 vs. 非金属
金属是优良的导体，导电性和传热性都非常好。当光线打到金属表面，部分被反射，折射部分全部被吸收。由于金属对不同波长的光线的吸收程度不同，所以金属的反射光线通常是有颜色的，即$F_{0}$的RGB三通道值不同。而这也是金属本身的颜色的来源。比如黄金会吸收高频的蓝色波段，因此黄金看起来是黄色的。

金属还有一个特性，那就是生锈（被腐蚀）。生锈的金属会失去金属光泽，变得类似非金属的材质。

对于非金属（或者称为介电质、非导体、绝缘体），导电性很差，折射进入材质的光线会被散射或者吸收，然后折射离开材质。而非金属反射的光线也比金属少，即非金属的$F_{0}$更小，通常在0.02到0.05之间，绝大部分情况都不会超过0.04。

总结一下，金属的$F_{0}$跟材质本身有关，非金属的$F_{0}$约等于0.04。

## 金属/粗糙度工作流

![mrworkflow](images/PBR%20workflow/mrworkflow.png)

金属/粗糙度工作流（简写为M/R工作流）采用了三张贴图来表示PBR所需的参数：

- baseColor，RGB贴图，表示非导体（电介质）的反照率颜色（Albedo）或金属的$F_{0}$，它不应该含有任何光照信息；
- metallic，灰度贴图，可以视为一个遮罩，表示金属成分的占比：1表示100%的纯金属，0表示100%的非金属，metallic贴图可以用于表示金属-非金属的混合状态；
- roughness，灰度贴图，数值越大越粗糙；

M/R工作流的优劣：

- 非金属的$F_{0}$固定为0.04，无法调整；
- 主流的工作流，用途广泛；

## 镜面反射/光泽度工作流

![sgworkflow](images/PBR%20workflow/sgworkflow.png)

镜面反射/光泽度工作流（简写为S/G工作流）

- Diffuse，即Albedo，RGB贴图，表示非金属的Albedo，如果是金属，Diffuse贴图给出的值是纯黑（0）；
- Specular，RGB贴图，定义金属和非金属的$F_{0}$；
- Glossiness，灰度贴图，数值越大月光滑，跟M/R工作流的roughness刚好相反；

S/G工作流的优劣：

- 可以对金属、非金属的$F_{0}$自由调整，但是这也非常容易做出违反能量守恒定律的材质；
- 两张RGB贴图，对性能的要求会更高；

## 总结
下图展示了两个工作流的相同与不同。

![workflow](images/PBR%20workflow/workflows.png)

### 通用的贴图
两个工作流有几个通用的纹理贴图，包括：

- 环境光遮蔽/环境光吸收贴图Ambient Occlusion，灰度贴图，表示环境光对材质的影响，这个贴图会作为一个系数与环境光结合；
- 高度贴图Height，用于displacement mapping效果，模拟表面细节；
- 法向贴图Normal，用于normal mapping，模拟表面细节；

### 分辨率与纹素密度
这个话题是The PBR Guide[1]提出的，我在实践中还没有遇到过。这里根据The PBR Guide的介绍理解一下。

![Resolution](images/PBR%20workflow/Resolution.png)

这种异常的边界通常出现在金属/非金属交界的位置，出现的原因是两张纹理贴图（M/R工作流中的baseColor和Metallic，S/G工作流中的Diffuse和Specular）之间的微小的误差。比如在M/R工作流中，白边处的Metallic比较小，但是baseColor认为它还是金属，因此baseColor里包含的非常亮的金属反射值，导致了白边。而S/G工作流中，Diffuse贴图中因为原始金属并没有漫射色，所以呈现为黑色。

从上面的分析可以看出，出现这个问题主要原因是纹理贴图的精度不够，提高纹理的纹素密度，即纹理涂片的分辨率就可以解决这个问题。

### 辨析baseColor，Diffuse，Albedo，F0，DiffuseColor
这几个词在本文频繁出现，他们的意义非常接近，但是又有所不同，所以单独拎出来解释一下。

首先，baseColor和Diffuse被用在了两个工作流里面，他们的意义前面已经详细介绍过了，这两个词都只是一个对纹理的称呼，可以理解为在特定工作流里面的专有名词。

Albedo称为反射率本征图，表示物体本身的颜色，这个颜色是与光照无关的。

$F_{0}$不是一个颜色，而是一个介于$[0,1]$的比例，表示反射光线radiance占入射光线radiance的比例。这个值主要是用在BRDF公式里面的菲涅尔方程里，通常这个值与albedo有关。

DiffuseColor更像是渲染里面的用词，表示入射光线或者出射光线的Diffuse分量，和SpecularColor相对。

## 实例
我在Github找到一个repo[3]，支持了上面介绍的两个工作流，可以拿来参考一下。

这个repo的做法是，将S/G工作流的参数转为M/R工作流的参数，它提取出来的参数包括：

```C++
float perceptualRoughness;
float metallic;
vec4 baseColor;
```

具体的shader实现如下，细节不在赘述：

```C++
struct PBRInfo
{
	float NdotL;                  // cos angle between normal and light direction
	float NdotV;                  // cos angle between normal and view direction
	float NdotH;                  // cos angle between normal and half vector
	float LdotH;                  // cos angle between light direction and half vector
	float VdotH;                  // cos angle between view direction and half vector
	float perceptualRoughness;    // roughness value, as authored by the model creator (input to shader)
	float metalness;              // metallic value at the surface
	vec3 reflectance0;            // full reflectance color (normal incidence angle)
	vec3 reflectance90;           // reflectance color at grazing angle
	float alphaRoughness;         // roughness mapped to a more linear change in the roughness (proposed by [2])
	vec3 diffuseColor;            // color contribution from diffuse lighting
	vec3 specularColor;           // color contribution from specular lighting
};

// Gets metallic factor from specular glossiness workflow inputs 
float convertMetallic(vec3 diffuse, vec3 specular, float maxSpecular) {
	float perceivedDiffuse = sqrt(0.299 * diffuse.r * diffuse.r + 0.587 * diffuse.g * diffuse.g + 0.114 * diffuse.b * diffuse.b);
	float perceivedSpecular = sqrt(0.299 * specular.r * specular.r + 0.587 * specular.g * specular.g + 0.114 * specular.b * specular.b);
	if (perceivedSpecular < c_MinRoughness) {
		return 0.0;
	}
	float a = c_MinRoughness;
	float b = perceivedDiffuse * (1.0 - maxSpecular) / (1.0 - c_MinRoughness) + perceivedSpecular - 2.0 * c_MinRoughness;
	float c = c_MinRoughness - perceivedSpecular;
	float D = max(b * b - 4.0 * a * c, 0.0);
	return clamp((-b + sqrt(D)) / (2.0 * a), 0.0, 1.0);
}

void main()
{
	float perceptualRoughness;
	float metallic;
	vec4 baseColor;

	vec3 f0 = vec3(0.04);

	if (material.alphaMask == 1.0f) {
		if (material.baseColorTextureSet > -1) {
			baseColor = SRGBtoLINEAR(texture(colorMap, inUV)) * material.baseColorFactor;
		} else {
			baseColor = material.baseColorFactor;
		}
		if (baseColor.a < material.alphaMaskCutoff) {
			discard;
		}
	}

	if (material.workflow == PBR_WORKFLOW_METALLIC_ROUGHNESS) {
		// Metallic and Roughness material properties are packed together
		// In glTF, these factors can be specified by fixed scalar values
		// or from a metallic-roughness map
		perceptualRoughness = material.roughnessFactor;
		metallic = material.metallicFactor;

        // Roughness is stored in the 'g' channel, metallic is stored in the 'b' channel.
        // This layout intentionally reserves the 'r' channel for (optional) occlusion map data
        vec4 mrSample = texture(physicalDescriptorMap, inUV);
        perceptualRoughness = mrSample.g * perceptualRoughness;
        metallic = mrSample.b * metallic;

		// The albedo may be defined from a base texture or a flat color
        baseColor = SRGBtoLINEAR(texture(colorMap, inUV)) * material.baseColorFactor;
	}

	if (material.workflow == PBR_WORKFLOW_SPECULAR_GLOSINESS) {
		// Values from specular glossiness workflow are converted to metallic roughness
        perceptualRoughness = 1.0 - texture(physicalDescriptorMap, inUV).a;

		const float epsilon = 1e-6;

		vec4 diffuse = SRGBtoLINEAR(texture(colorMap, inUV0));
		vec3 specular = SRGBtoLINEAR(texture(physicalDescriptorMap, inUV0)).rgb;

		float maxSpecular = max(max(specular.r, specular.g), specular.b);

		// Convert metallic value from specular glossiness inputs
		metallic = convertMetallic(diffuse.rgb, specular, maxSpecular);

		vec3 baseColorDiffusePart = diffuse.rgb * ((1.0 - maxSpecular) / (1 - c_MinRoughness) / max(1 - metallic, epsilon)) * material.diffuseFactor.rgb;
		vec3 baseColorSpecularPart = specular - (vec3(c_MinRoughness) * (1 - metallic) * (1 / max(metallic, epsilon))) * material.specularFactor.rgb;
		baseColor = vec4(mix(baseColorDiffusePart, baseColorSpecularPart, metallic * metallic), diffuse.a);

	}

	vec3 diffuseColor;
	diffuseColor = baseColor.rgb * (vec3(1.0) - f0);
	diffuseColor *= 1.0 - metallic;
		
	float alphaRoughness = perceptualRoughness * perceptualRoughness;

	vec3 specularColor = mix(f0, baseColor.rgb, metallic);

	// Compute reflectance.
	float reflectance = max(max(specularColor.r, specularColor.g), specularColor.b);

	// For typical incident reflectance range (between 4% to 100%) set the grazing reflectance to 100% for typical fresnel effect.
	// For very low reflectance range on highly diffuse objects (below 4%), incrementally reduce grazing reflecance to 0%.
	float reflectance90 = clamp(reflectance * 25.0, 0.0, 1.0);
	vec3 specularEnvironmentR0 = specularColor.rgb;
	vec3 specularEnvironmentR90 = vec3(1.0, 1.0, 1.0) * reflectance90;

	vec3 n = (material.normalTextureSet > -1) ? getNormal() : normalize(inNormal);
	vec3 v = normalize(ubo.camPos - inWorldPos);    // Vector from surface point to camera
	vec3 l = normalize(uboParams.lightDir.xyz);     // Vector from surface point to light
	vec3 h = normalize(l+v);                        // Half vector between both l and v
	vec3 reflection = -normalize(reflect(v, n));
	reflection.y *= -1.0f;

	float NdotL = clamp(dot(n, l), 0.001, 1.0);
	float NdotV = clamp(abs(dot(n, v)), 0.001, 1.0);
	float NdotH = clamp(dot(n, h), 0.0, 1.0);
	float LdotH = clamp(dot(l, h), 0.0, 1.0);
	float VdotH = clamp(dot(v, h), 0.0, 1.0);

	PBRInfo pbrInputs = PBRInfo(
		NdotL,
		NdotV,
		NdotH,
		LdotH,
		VdotH,
		perceptualRoughness,
		metallic,
		specularEnvironmentR0,
		specularEnvironmentR90,
		alphaRoughness,
		diffuseColor,
		specularColor
	);

    // ...
```

最后利用纹理提取出来的材质信息以及光源信息构建结构体`PBRInfo`，用这个结构体的数据计算输出的颜色。

## Reference
1. The PBR Guide, [part 1](https://academy.substance3d.com/courses/the-pbr-guide-part-1) and [part 2](https://academy.substance3d.com/courses/the-pbr-guide-part-2)
2. ISUX译文 | The PBR Guide 基于物理的渲染指引，[上](https://www.zcool.com.cn/article/ZMTA5NDY5Ng==.html)和[下](https://www.zcool.com.cn/article/ZMTExNTA5Ng==.html)
3. github repo: [Vulkan-glTF-PBR](https://github.com/SaschaWillems/Vulkan-glTF-PBR)
