渲染之十八 实时全局光照、探针体和LOD组
======

 * 支持实时全局光照
 * 计算自发光对全局光照的贡献
 * 使用光照探针代理体
 * 将 LOD 组与全局光照结合
 * 交叉淡入淡出 LOD 组

这篇是关于渲染的系列教程的第18部分。在第17部分完成烘焙全局光照后，我们继续支持实时全局光照，之后，我们还将支持光照探针代理体和交叉淡入淡出LOD组。

从本篇开始，教程使用 Unity 2017.1.0f3 制作，不适用于旧版本，因为我们最终将使用新的着色器功能。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/tutorial-image.jpg)  
*静态 LOD 组和实时全局光照的结合*

# 1 实时全局光照

烘焙光照对于静态几何体非常有效，并且借助光照探针也非常适合动态几何体。但是，它无法处理动态光源。混合模式下的光源可以进行一些实时调整，但是调整过多会使烘焙的间接光严重不协调。因此，当你有户外场景时，太阳必须保持不变，它不能像现实生活中那样东升西落，因为这就需要逐渐改变全局光照。所以场景必须保持在某个时间点不变。

为了使间接光照与移动的太阳一起工作，Unity 使用 Enlighten 系统来实时计算全局光照。除了在运行时实时计算光照贴图和探针外，它的工作方式与烘焙间接光一样。

想弄清楚间接光需要知道光如何在静态表面之间反弹，关键问题是哪些表面可能受到哪些表面的影响，以及其程度。这些工作量太大，不能实时完成，因此，这些数据由编辑器处理并存储以便在运行时使用。 Enlighten 使用这些数据来计算实时光照贴图和探针。即使这样，它也只适用于低分辨率光照贴图。

## 1.1 开启实时全局光照

实时全局光照和烘焙是独立的两个选项，您可以同时关闭、开启一个或两个都开启。其复选框位于 Lighting 窗口的 Realtime Lighting 部分。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-enabled.png)  
*实时全局光照和烘焙同时开启*

要查看实时全局光照，请将我们测试场景中主灯的模式设置为 Realtime。既然我们没有其他光源，这样事实上等于关闭了烘焙光照，即使它已启用。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-light.png)  
*实时的主光源*  

确保场景中的所有对象都使用我们的白色材质。就像上次一样，球体都是动态的，其他一切物体都是静态。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-dynamic-only.png)  
*只有动态物体接受实时光照*

事实证明，只有动态对象才能从实时全局光照中受益。静态物体会变得更暗，那是因为探针数据并入了实时光照贴图。所以静态对象也必须对实时光照贴图采样，而不仅是烘焙光照贴图，而我们的着色器还没有处理过实时光照贴图。

## 1.2 烘焙实时全局光照

Unity 在编辑模式下已生成实时光照贴图，因此您始终可以看到实时全局光照的效果。在编辑模式和播放模式之间切换时，这些图会重新生成，但内容是一样的。您可以切换到 Lighting 窗口的 Object maps 标签卡，选择一个静态对象，来检查它的实时光照贴图。选择 Realtime Intensity 以查看实时光照贴图数据。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-intensity.png)  
*实时光照贴图，所选物体是屋顶*

虽然实时光照贴图已经烘焙好了，并且它们看起来可能没什么问题，但我们的 meta pass 实际上使用的是错误的坐标。实时全局光照有自己的光照贴图坐标，和静态光照贴图的坐标不同。 Unity 根据光照贴图和物体的设置自动生成这些坐标，并存储在第三个 UV 通道中。 因此，将此参数添加到 My Lightmapping 中的 VertexData。

```c
struct VertexData {
	float4 vertex : POSITION;
	float2 uv : TEXCOORD0;
	float2 uv1 : TEXCOORD1;
	float2 uv2 : TEXCOORD2;
};
```

现在，MyLightmappingVertexProgram 必须使用第二个或第三个UV，来处理静态或动态光照贴图的缩放和偏移。我们可以使用 UnityMetaVertexPosition 函数来获取正确的数据。

```c
Interpolators MyLightmappingVertexProgram (VertexData v) {
	Interpolators i;
//	v.vertex.xy = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
//	v.vertex.z = v.vertex.z > 0 ? 0.0001 : 0;
//
//	i.pos = UnityObjectToClipPos(v.vertex);
	i.pos = UnityMetaVertexPosition(
		v.vertex, v.uv1, v.uv2, unity_LightmapST, unity_DynamicLightmapST
	);

	i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
	i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);
	return i;
}
```

> **UnityMetaVertexPosition 函数长什么样子？**
> 它做的就是我们以前做的事情，除了它使用 unity_MetaVertexControl 所提供的标记来决定使用哪套坐标和光照贴图。
```c
float4 UnityMetaVertexPosition (
	float4 vertex, float2 uv1, float2 uv2,
	float4 lightmapST, float4 dynlightmapST
) {
	if (unity_MetaVertexControl.x) {
		vertex.xy = uv1 * lightmapST.xy + lightmapST.zw;
		// OpenGL right now needs to actually use incoming vertex position,
		// so use it in a very dummy way
		vertex.z = vertex.z > 0 ? 1.0e-4f : 0.0f;
	}
	if (unity_MetaVertexControl.y) {
		vertex.xy = uv2 * dynlightmapST.xy + dynlightmapST.zw;
		// OpenGL right now needs to actually use incoming vertex position,
		// so use it in a very dummy way
		vertex.z = vertex.z > 0 ? 1.0e-4f : 0.0f;
	}
	return UnityObjectToClipPos(vertex);
}
```

请注意，meta pass 既用于烘焙也用于实时光照贴图。因此，当使用实时全局光照时，它也将包含在最终构建中。

## 1.3 采样实时光照贴图

为了采样实时光照贴图，My Lighting 中的 VertexData 也要添加第三套 UV。

```c
struct VertexData {
	float4 vertex : POSITION;
	float3 normal : NORMAL;
	float4 tangent : TANGENT;
	float2 uv : TEXCOORD0;
	float2 uv1 : TEXCOORD1;
	float2 uv2 : TEXCOORD2;
};
```

当使用实时光照贴图时，我们必须将其光照贴图坐标添加到插值器中。Unity 内置的标准着色器将两种光照贴图坐标集合在同一个插值器中——与其他一些数据复用——但我们可以为两者分别使用单独的插值器。 我们知道当定义了 DYNAMICLIGHTMAP_ON 关键字时会有动态光照数据。它属于 multi_compile_fwdbase 指令所代表的关键字列表的一部分。

```c
struct Interpolators {
	…

	#if defined(DYNAMICLIGHTMAP_ON)
		float2 dynamicLightmapUV : TEXCOORD7;
	#endif
};
```

像填充静态光照贴图坐标一样填充坐标，注意获取动态光照贴图的缩放和偏移是通过 unity_DynamicLightmapST 变量。

```c
Interpolators MyVertexProgram (VertexData v) {
	…

	#if defined(LIGHTMAP_ON) || ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
		i.lightmapUV = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
	#endif

	#if defined(DYNAMICLIGHTMAP_ON)
		i.dynamicLightmapUV =
			v.uv2 * unity_DynamicLightmapST.xy + unity_DynamicLightmapST.zw;
	#endif

	…
}
```

实时光照贴图的采样在 CreateIndirectLight 函数中完成。复制一下 #if defined(LIGHTMAP_ON) 代码块并做一些更改：首先，新块基于 DYNAMICLIGHTMAP_ON 关键字；其次，它应该使用 DecodeRealtimeLightmap 而非 DecodeLightmap，因为实时贴图使用的颜色格式不同。由于实时贴图可能会和烘焙照明叠加，因此不要直接赋值给 indirectLight.diffuse，而是使用一个中间变量以在最后加上去；最后，我们应该在既没使用烘焙也没使用实时光照贴图时采样球谐波。

```c
		#if defined(LIGHTMAP_ON)
			indirectLight.diffuse =
				DecodeLightmap(UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightmapUV));
			
			#if defined(DIRLIGHTMAP_COMBINED)
				float4 lightmapDirection = UNITY_SAMPLE_TEX2D_SAMPLER(
					unity_LightmapInd, unity_Lightmap, i.lightmapUV
				);
				indirectLight.diffuse = DecodeDirectionalLightmap(
					indirectLight.diffuse, lightmapDirection, i.normal
				);
			#endif

			ApplySubtractiveLighting(i, indirectLight);
//		#else
//			indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
		#endif

		#if defined(DYNAMICLIGHTMAP_ON)
			float3 dynamicLightDiffuse = DecodeRealtimeLightmap(
				UNITY_SAMPLE_TEX2D(unity_DynamicLightmap, i.dynamicLightmapUV)
			);

			#if defined(DIRLIGHTMAP_COMBINED)
				float4 dynamicLightmapDirection = UNITY_SAMPLE_TEX2D_SAMPLER(
					unity_DynamicDirectionality, unity_DynamicLightmap,
					i.dynamicLightmapUV
				);
            	indirectLight.diffuse += DecodeDirectionalLightmap(
            		dynamicLightDiffuse, dynamicLightmapDirection, i.normal
            	);
			#else
				indirectLight.diffuse += dynamicLightDiffuse;
			#endif
		#endif

		#if !defined(LIGHTMAP_ON) && !defined(DYNAMICLIGHTMAP_ON)
			indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
		#endif
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-everything.png)  
*所有物体都接受实时光照*  

现在我们的着色器使用了实时光照贴图。如果使用了距离阴影遮罩，一开始它可能看起来与使用混合烘焙照明没什么区别，但是在播放模式下关闭光源时，差异就会特别明显。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/mixed-light-runtime-disabled.png)  
*关闭了混合光源之后间接光还在*  

混合光源关闭之后，间接光会保持不变。 相比之下，实时全局光照的间接光会消失—— 并重新出现——理论上应该如此。 但是，在烘焙完全结束之前可能需要一小段时间，Enlighten 系统会逐渐调整光照贴图和探针，调整完成的速度取决于场景的复杂程度和 Graphics Setting 里的 Realtime Global Illumination CPU 质量等级设置。

[*视频：开关实时全局光照下的实时灯光。*](https://thumbs.gfycat.com/DenseLinearCornsnake-mobile.mp4)  
 

所有实时光源都对实时全局光照有贡献。然而，它的典型用途仅是用于主方向光，代表天空中移动的太阳。它完全适用于所有方向光源。点光源和聚光灯也可以工作，但是不会产生阴影。因此，当使用带阴影的点光源或聚光灯时，会出现不正确的间接光照。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/spotlight-inspector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-spotlight.png)  
*实时聚光灯，间接光没有阴影。*  

## 1.4 自发光光源

实时全局光照还可用于自发光的静态对象。这使得可以通过实时间接光调整自发光效果。让我们来试试，向场景中添加一个静态球体，并为其提供一种材质，该材质使用具有黑色反照率和白色自发光的着色器。一开始，我们只能通过静态光照贴图看到自发光的间接光的影响。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/baked-emissive.png)  
*自发光球体烘焙全局光*  

要将自发光烘焙到静态光照贴图中，我们必须在着色器 GUI 中设置材质的全局照明标记。由于我们总是将标志设置为 BakedEmissive ，因此自发光会烘焙到光照贴图中。当自发光恒定时，这很好，但这样就不允许有任何修改。

为了同时支持自发光的烘焙和实时照明，我们必须使其可配置。 我们可以通过 MaterialEditor.LightmapEmissionProperty 方法为 MyLightingShaderGUI 添加一个选项来实现。方法的唯一参数是属性面板上的的缩进级别。

```c#  
	void DoEmission () {
		MaterialProperty map = FindProperty("_EmissionMap");
		Texture tex = map.textureValue;
		EditorGUI.BeginChangeCheck();
		editor.TexturePropertyWithHDRColor(
			MakeLabel(map, "Emission (RGB)"), map, FindProperty("_Emission"),
			emissionConfig, false
		);
		editor.LightmapEmissionProperty(2);
		if (EditorGUI.EndChangeCheck()) {
			if (tex != map.textureValue) {
				SetKeyword("_EMISSION_MAP", map.textureValue);
			}

			foreach (Material m in editor.targets) {
				m.globalIlluminationFlags =
					MaterialGlobalIlluminationFlags.BakedEmissive;
			}
		}
	}
```

每次自发光属性发生变化时，我们还必须注意取消覆盖标记。实际上的情况更为复杂。其中一个标记选项是 EmissiveIsBlack，它表示可以跳过自发光的计算。新材质默认设置此标记。为了使间接自发光工作，无论我们选择实时还是烘焙，我们都必须保证不要设置此标记。可以通过始终屏蔽标记值的 EmissiveIsBlack 位来实现此目的。

```c#
			foreach (Material m in editor.targets) {
				m.globalIlluminationFlags &=
					~MaterialGlobalIlluminationFlags.EmissiveIsBlack;
			}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-emissive-inspector.png) 
![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-emissive.png)  
*实时全局光照下的自发光球体*  

烘焙和实时全局光照之间可见的差异是实时光照贴图的分辨率通常低于烘焙贴图的分辨率。 因此，当不打算动态改变自发光并且仍然在使用烘焙光照时，请充分利用其分辨率更高的优点。

> **设计 EmissiveIsBlack 的目的是?**  
> 这是一个优化，可以跳过部分GI烘焙过程。但是，它是自发光颜色确实是黑色时才能设置的标记。 既然由着色器 GUI 负责设置，就要在通过 Inspector 编辑材质时确定，至少 Unity的标准着色器就是这样做的。因此，如果之后又通过脚本或动画系统更改了自发光颜色，标记不会自动修改。许多人不理解为什么他们的动画自发光不会影响实时全局光照，这就是原因之一。如果您想在运行时更改自发光，那么有一个技巧就是，不要将自发光颜色设置为纯黑色。
> 
> 我们不使用这种方法，而是使用 LightmapEmissionProperty，这样可以提供完全关闭自发光影响全局光的选项。此选项对于用户是含义明确的，没有任何隐藏的行为。 不想使用自发光？确保其全局光照设置为 None 就行了。

## 1.5 自发光动起来

自发光的实时全局光照仅适用于静态对象。虽然物体是静态的，但是它们的材质的自发光是可动的并且将由全局照明系统取用。让我们尝试一下在白色和黑色自发光之间振荡的简单组件。

```c#
using UnityEngine;

public class EmissiveOscillator : MonoBehaviour {

	Material emissiveMaterial;

	void Start () {
		emissiveMaterial = GetComponent<MeshRenderer>().material;
	}
	
	void Update () {
		Color c = Color.Lerp(
			Color.white, Color.black,
			Mathf.Sin(Time.time * Mathf.PI) * 0.5f + 0.5f
		);
		emissiveMaterial.SetColor("_Emission", c);
	}
}
```

将此组件添加到我们的自发光球体上。在播放模式下，其自发光将具有动画效果，但间接光尚未受到影响。我们必须通知实时全局光照系统它有工作要做，调用相应的 MeshRenderer 的 Renderer.UpdateGIMaterials 方法。

```c#
	MeshRenderer emissiveRenderer;
	Material emissiveMaterial;

	void Start () {
		emissiveRenderer = GetComponent<MeshRenderer>();
		emissiveMaterial = emissiveRenderer.material;
	}
	
	void Update () {
		…
		emissiveMaterial.SetColor("_Emission", c);
		emissiveRenderer.UpdateGIMaterials();
	}
```

[*视频：动态自发光的实时全局光照*](https://thumbs.gfycat.com/VillainousFlakyBlueandgoldmackaw-mobile.mp4)  
 

调用 UpdateGIMaterials 会触发物体自发光的完整更新，并使用其 meta pass 渲染它。 当自发光不仅是纯色时，这是很有必要的，比如使用纹理时。如果仅仅是纯色，我们可以使用一个快捷方法，调用 DynamicGI.SetEmissive，参数是自发光颜色，这比使用 meta pass 渲染更快，因此在可以的时候请尽可能利用它的优点。

```c#
//		emissiveRenderer.UpdateGIMaterials();
		DynamicGI.SetEmissive(emissiveRenderer, c);
```

# 2 光照探针代理体

