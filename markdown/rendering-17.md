渲染之十七 混合光照
======

 * 只烘焙间接光
 * 烘焙阴影和实时阴影的混合
 * 处理代码修改和错误
 * 处理减法光照
 
这一篇是渲染教程的第十七部分，上一部分里我们通过光照贴图增加了对静态光照的支持，这次，我们将朝着烘焙和实时光照结合的方向前进。

这篇教程是使用 Unity 5.6.0 制作的。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/tutorial-image.jpg)  
*烘焙和实时光照的混合*  


# 1.烘焙间接光

光照贴图允许我们预计算光照，这会减少 GPU 实时完成的工作量，代价是增加了纹理所占内存，除此之外，它还增加了间接光照。但正如我们上次看到的那样，光照贴图也存在局限性：首先，高光无法烘焙；其次，烘焙光源仅能通过光照探针影响动态物体；第三，烘焙光源不会投射实时阴影。

你可以在下面的屏幕截图中看到完全实时和完全烘焙光照之间的区别。这个场景取自上一节教程，我移动了一些球体位置，并把所有球体设为动态物体，其他物体都是静态的，渲染路径是前向渲染。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/fully-realtime.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/fully-baked.png)  
*完全实时和完全烘焙光照*  

我没有调整光照探针，所以现在它们的位置不太明显，因为静态几何体较少，由此产生的探针光照有点偏，这使得探针在使用时更容易被注意到。

## 1.1 混合模式

间接光照是烘焙拥有的特点而实时照明不具备，因为它需要光照贴图。由于间接光可以为场景增添很多真实感，如果我们可以将它与实时光照结合起来会很好，这是可能的，虽然这意味着阴影变得更加昂贵。将 Mixed Lighting 的 Lighting Mode 设置为 Baked Indirect。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/baked-indirect.png)  
*Mixed Lighting -> Baked Indirect* 

我们已经在上一节教程中切换到了这种模式，但当时我们只使用了完全烘焙。结果混合照明模式没有起作用，要使用混合照明，必须将光源的模式设置为 Mixed。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/mixed-light.png)  
*混合模式的主光源*  

将主方向光修改为混合模式后，会发生两件事：   
首先，Unity 会再次烘焙光照贴图，这次，它只会存储间接光，因此生成的光照贴图会比以前暗很多。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/lightmap-baked.png) 
![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/lightmap-indirect.png)  
*完全烘焙 vs. 只有间接光的光照贴图*

其次，所有物体都会被照亮，如同主光源被设置为实时一样。但有一点不同：向静态物体添加间接光是通过光照贴图，而不是球谐或探针。动态物体仍然通过光照探针来获得间接光。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/mixed-lighting.png)  
*混合光照，实时的直接光加上烘焙的间接光*  

我们不必为了支持它而修改着色器，因为 forward base pass 已经将光照贴图数据和主方向光组合在一起。像往常一样，其余的灯可以获得 additive pass。 使用延迟渲染路径时，主灯也会获得一个 pass。

>**混合光照可以在运行时动态调整吗？**  
>可以，因为实时光照部分还在使用它们，但是，烘焙数据部分是静态的，所以在运行时你只能对光源进行微小调整，比如改变一点强度，过大的变化会使烘焙和实时光照之间的不一致特别明显。

## 1.2 更新着色器

一开始一切似乎都很好。 然而事实证明，阴影衰减不再适用于方向光。只要大幅减少阴影距离，就容易看到阴影被切断了。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/shadows-standard.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/shadows-custom.png)  
*阴影衰减，标准着色器 vs. 我们的着色器*  

Unity 曾经长时间采用一种混合光照模式，但实际上在 Unity 5 中已经无法使用这种模式了。Unity 5.6 中增加了一种新的混合光照模式，这就是我们现在使用的模式。 增加此新模式后，UNITY_LIGHT_ATTENUATION 宏背后的代码已经更改。我们在使用全烘焙或全实时光照时没有注意这一点，但我们现在必须更新代码，以使用新的混合光照。由于这是最近发生的重大变化，我们必须注意 bug。

第一个修改是不再使用 SHADOW_COORDS 宏来定义阴影坐标的插值器，我们必须使用新的 UNITY_SHADOW_COORDS 宏。

```c
struct Interpolators {
	…

//	SHADOW_COORDS(5)
	UNITY_SHADOW_COORDS(5)

	…
};
```

同样，TRANSFER_SHADOW 应该换成 UNITY_TRANSFER_SHADOW。

```c
Interpolators MyVertexProgram (VertexData v) {
	…

//	TRANSFER_SHADOW(i);
	UNITY_TRANSFER_SHADOW(i);

	…
}
```

但是，这会产生一个编译器错误，因为这个宏需要一个额外的参数。从 Unity 5.6 开始，只有方向阴影所使用的屏幕空间坐标才会被放入插值器中，点光源和聚光灯的阴影坐标现在都在片段着色器中计算。另外一个新特点是，在某些情况下，光照贴图坐标会被用于阴影遮罩，我们将在后面介绍。为此，必须把第二个 UV 通道的数据提供给宏，也就是光照贴图坐标的数据。

```c
UNITY_TRANSFER_SHADOW(i, v.uv1);
```

这时又会出现编译器错误，发生这种情况是因为 UNITY_SHADOW_COORDS 在某些情况下会错误地创建插值器，即使实际上并不需要它。 在这种情况下，TRANSFER_SHADOW 不会对它进行初始化，从而导致错误。 这个 bug 存在于 5.6.0 版本中，至少在 5.6.2 和 2017.1.0 beta 还未修复。

这个 bug 通常不会遇到，因为 Unity 的标准着色器使用 UNITY_INITIALIZE_OUTPUT 宏来对其插值器结构体进行完全初始化，因为我们没有使用这个宏，所以我们遇到了这个 bug。要解决这个问题，请使用 UNITY_INITIALIZE_OUTPUT 宏来初始化插值器。这样，我们的代码会跳过（或者说带着）这个 bug 编译成功。

```c
Interpolators MyVertexProgram (VertexData v) {
	Interpolators i;
	UNITY_INITIALIZE_OUTPUT(Interpolators, i);
	…
}
```
> **UNITY_INITIALIZE_OUTPUT 都做了什么？**  
> 当支持它时，它只是将 0 赋值给变量，并转换为正确的类型。它不受支持时候什么都不做。
```c
// Initialize arbitrary structure with zero values.
// Not supported on some backends
// (e.g. Cg-based particularly with nested structs).
// hlsl2glsl would almost support it, except with structs that have arrays
// -- so treat as not supported there either :(
#if defined(UNITY_COMPILER_HLSL) || defined(SHADER_API_PSSL) || \
	defined(UNITY_COMPILER_HLSLCC)
	#define UNITY_INITIALIZE_OUTPUT(type,name) name = (type)0;
#else
	#define UNITY_INITIALIZE_OUTPUT(type,name)
#endif
```
> 我更倾向于不要使用这个宏，而是依赖显式地赋值，因为它会隐藏我们刚刚遇到的那种 bug。

## 1.3 自己处理阴影衰减

我们现在正确使用了新的宏，但我们主光源的阴影仍然不会像应有的那样衰减。 事实证明，当同时使用方向阴影和光照贴图时，UNITY_LIGHT_ATTENUATION 不会计算此衰减，主方向光设为混合模式之后情况就是如此， 所以我们必须手动完成。

> **这种情况下为什么不会衰减呢？**  
> UNITY_LIGHT_ATTENUATION 宏曾经是可以独立使用的，但自 Unity 5.6 以来，它被设计为与 Unity 的标准全局光照一起使用，而我们没有按照设计使用，因此它无法工作。
> 
> 至于为何进行此更改，唯一的线索是 AutoLight 中的注释，其中写到：“handles shadows in the depths of the GI function for performance reasons”（出于性能原因，在 GI 函数中处理深度中的阴影）。 由于着色器编译器随意移动代码，这等于什么也没有告诉我们。 即使这个特殊情况有一个充分的理由，也是无从查证，因为 Unity 的着色器代码已经变得非常纠结。所以，我也不知道。

对于我们的延迟光照着色器，我们已经有代码来计算阴影衰减。将相关代码片段从 MyDeferredShading 复制到 My Lighting 中的一个新函数。 唯一真正的区别是我们必须从观察矢量和观察矩阵构造 viewZ，只有 Z 分量是真正需要的，因此我们不需要执行全矩阵乘法。

```c
float FadeShadows (Interpolators i, float attenuation) {
	float viewZ =
		dot(_WorldSpaceCameraPos - i.worldPos, UNITY_MATRIX_V[2].xyz);
	float shadowFadeDistance =
		UnityComputeShadowFadeDistance(i.worldPos, viewZ);
	float shadowFade = UnityComputeShadowFade(shadowFadeDistance);
	attenuation = saturate(attenuation + shadowFade);
	return attenuation;
}
```

手动计算衰减必须在调用 UNITY_LIGHT_ATTENUATION 之后。

```c
UnityLight CreateLight (Interpolators i) {
	…

		UNITY_LIGHT_ATTENUATION(attenuation, i, i.worldPos.xyz);
		attenuation = FadeShadows(i, attenuation);

	…
}
```

但是只有当 UNITY_LIGHT_ATTENUATION 决定跳过衰减时才能手动计算衰减。UnityShadowLibrary 文件中定义了 HANDLE_SHADOWS_BLENDING_IN_GI 就是标识这种情况。因此，只有定义了 HANDLE_SHADOWS_BLENDING_IN_GI 时，FadeShadows 才能执行操作。

```c
float FadeShadows (Interpolators i, float attenuation) {
	#if HANDLE_SHADOWS_BLENDING_IN_GI
		// UNITY_LIGHT_ATTENUATION doesn't fade shadows for us.
		float viewZ =
			dot(_WorldSpaceCameraPos - i.worldPos, UNITY_MATRIX_V[2].xyz);
		float shadowFadeDistance =
			UnityComputeShadowFadeDistance(i.worldPos, viewZ);
		float shadowFade = UnityComputeShadowFade(shadowFadeDistance);
		attenuation = saturate(attenuation + shadowFade);
	#endif
	return attenuation;
}
```

最后，我们的阴影又如它应有的样子一样衰减了。

# 2 使用阴影遮罩

烘焙间接光加混合模式光源非常昂贵，它们的运算量与实时光源一样多，还要加上间接光源的光照贴图，而混合模式与全烘焙相比，最重要的就是增加了实时阴影。幸运的是，有一种方法可以将阴影烘焙到光照贴图中，并结合实时阴影，要启用此功能，请将混合光照模式更改为 Shadowmask。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/shadowmask-mode.png)  
*阴影遮罩模式*  

在此模式下，混合光的间接光照和阴影衰减都存储在光照贴图中，而阴影存储在单独的一张贴图中，称为阴影遮罩。当仅使用主方向光时，所有被照亮的东西都会在阴影遮罩中显示为红色，它是红色的，因为阴影信息存储在纹理的 R 通道中。实际上，一张贴图最多可以存储四个光源的阴影，因为它有四个通道。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/shadowmask.png)  
*烘焙的强度图和阴影遮罩*  

Unity创建了阴影遮罩后，静态对象投射的阴影将消失，只有光照探针仍然需要考虑它们。 动态对象的阴影不受影响。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/no-baked-shadows.png)  
*烘焙的阴影消失了*  


## 2.1 对阴影遮罩采样

为了获得烘焙的阴影，我们必须对阴影遮罩采样。Unity 的宏已经为点光源和聚光灯做了这个工作，但我们也必须将它包含在我们的 FadeShadows 函数中。我们可以使用 UnityShadowLibrary 中的 UnitySampleBakedOcclusion 函数。，它需要光照贴图UV坐标和世界位置作为参数。

```c
float FadeShadows (Interpolators i, float attenuation) {
	#if HANDLE_SHADOWS_BLENDING_IN_GI
		…
		float bakedAttenuation =
			UnitySampleBakedOcclusion(i.lightmapUV, i.worldPos);
		attenuation = saturate(attenuation + shadowFade);
	#endif
	return attenuation;
}
```

> UnitySampleBakedOcclusion 都做了什么？
> 它使用光照贴图坐标对阴影遮罩进行采样，然后选择适当的通道。 unity_OcclusionMaskSelector 变量是一个向量，其中的某个分量被设置为1，与当前被着色的光源匹配。
```c
fixed UnitySampleBakedOcclusion (float2 lightmapUV, float3 worldPos) {
    #if defined (SHADOWS_SHADOWMASK)
        #if defined(LIGHTMAP_ON)
            fixed4 rawOcclusionMask = UNITY_SAMPLE_TEX2D_SAMPLER(
				unity_ShadowMask, unity_Lightmap, lightmapUV.xy
			);
        #else
			fixed4 rawOcclusionMask =
				UNITY_SAMPLE_TEX2D(unity_ShadowMask, lightmapUV.xy);
        #endif
        return saturate(dot(rawOcclusionMask, unity_OcclusionMaskSelector));
    #else
        return 1.0;
    #endif
}
```
> 这个函数还处理光照探针代理体的衰减，但我们还没有打算支持那些东西，所以我删除了相关代码，要求世界位置作为参数也是这个原因。

UnitySampleBakedOcclusion 在使用阴影遮罩时返回烘焙阴影的衰减，在其他情况下返回 1。现在我们必须将它与我们已经具有的衰减相结合，以淡化我们的阴影，这些工作可以用 UnityMixRealtimeAndBakedShadows 函数完成。
```c
float bakedAttenuation =
			UnitySampleBakedOcclusion(i.lightmapUV, i.worldPos);
//		attenuation = saturate(attenuation + shadowFade);
		attenuation = UnityMixRealtimeAndBakedShadows(
			attenuation, bakedAttenuation, shadowFade
		);
```

> **UnityMixRealtimeAndBakedShadows 是如何工作的？**
> 它也是UnityShadowLibrary中的一个函数，删除了涉及光照探针代理体和一些与我们无关的边界情况的代码。
```c
inline half UnityMixRealtimeAndBakedShadows (
	half realtimeShadowAttenuation, half bakedShadowAttenuation, half fade
) {
	#if !defined(SHADOWS_DEPTH) && !defined(SHADOWS_SCREEN) && \
		!defined(SHADOWS_CUBE)
		return bakedShadowAttenuation;
	#endif

	#if defined (SHADOWS_SHADOWMASK)
		#if defined (LIGHTMAP_SHADOW_MIXING)
			realtimeShadowAttenuation =
				saturate(realtimeShadowAttenuation + fade);
			return min(realtimeShadowAttenuation, bakedShadowAttenuation);
		#else
			return lerp(
				realtimeShadowAttenuation, bakedShadowAttenuation, fade
			);
		#endif
	#else //no shadowmask
		return saturate(realtimeShadowAttenuation + fade);
	#endif
}
```
> 如果没有动态阴影，则返回烘焙的衰减，这意味着不计算动态对象的阴影，以及光照贴图对象的烘焙阴影。
> 当不使用阴影遮罩时，它会执行和我们以前一样的衰减，否则要看我们是否正在进行阴影混合，我们将在稍后介绍。 现在，它只是在实时和烘焙衰减之间进行插值。


![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/mixed-shadows.png)  
*实时阴影和阴影遮罩*  

我们现在可以在静态对象上获得实时和烘焙阴影，并把它们正确混合。实时阴影仍会在阴影距离之外淡出，但烘焙阴影则不会。

## 2.2 添加一个阴影遮罩到 G 缓存

阴影遮罩现在可以在前向渲染下使用，如果想要在延迟渲染下使用，我们还需要做一些工作。 具体来说，我们必须在需要时将阴影遮罩信息添加为额外的 G 缓存，因此，在定义 SHADOWS_SHADOWMASK 时，将另一个缓冲区添加到 FragmentOutput 结构中。

```c
struct FragmentOutput {
	#if defined(DEFERRED_PASS)
		float4 gBuffer0 : SV_Target0;
		float4 gBuffer1 : SV_Target1;
		float4 gBuffer2 : SV_Target2;
		float4 gBuffer3 : SV_Target3;

		#if defined(SHADOWS_SHADOWMASK)
			float4 gBuffer4 : SV_Target4;
		#endif
	#else
		float4 color : SV_Target;
	#endif
};
```

这是我们的第五个 G 缓存，数量已经不少了。 并非所有平台都支持它，仅当有足够的渲染目标可用时，Unity 才支持阴影遮罩，我们也应该这样做。

```c
#if defined(SHADOWS_SHADOWMASK) && (UNITY_ALLOWED_MRT_COUNT > 4)
	float4 gBuffer4 : SV_Target4;
#endif
```

我们现在只需将采样的阴影遮罩数据存储在 G 缓存中，因为此时我们还没有处理特定的光源。 我们可以使用 UnityGetRawBakedOcclusions 函数，它的工作方式与 UnitySampleBakedOcclusion 类似，区别在于前者不只使用一个通道。

```c
	FragmentOutput output;
	#if defined(DEFERRED_PASS)
		#if !defined(UNITY_HDR_ON)
			color.rgb = exp2(-color.rgb);
		#endif
		output.gBuffer0.rgb = albedo;
		output.gBuffer0.a = GetOcclusion(i);
		output.gBuffer1.rgb = specularTint;
		output.gBuffer1.a = GetSmoothness(i);
		output.gBuffer2 = float4(i.normal * 0.5 + 0.5, 1);
		output.gBuffer3 = color;

		#if defined(SHADOWS_SHADOWMASK) && (UNITY_ALLOWED_MRT_COUNT > 4)
			output.gBuffer4 =
				UnityGetRawBakedOcclusions(i.lightmapUV, i.worldPos.xyz);
		#endif
	#else
		output.color = ApplyFog(color, i);
	#endif
```

要在没有光照贴图的情况下进行编译，请在光照贴图不可用时将光照贴图坐标设为 0。  

## 2.3 使用阴影遮罩 G 缓存

要使我们的着色器和默认的延迟渲染着色器配合使用，这样就够了，但要使其与我们的自定义着色器一起使用，我们必须调整 MyDeferredShading 。第一步是为额外的 G 缓存添加变量。

```c
sampler2D _CameraGBufferTexture0;
sampler2D _CameraGBufferTexture1;
sampler2D _CameraGBufferTexture2;
sampler2D _CameraGBufferTexture4;
```

接下来，创建一个函数来获取阴影衰减。如果我们有一个阴影遮罩，可以对纹理进行采样，并和 unity_OcclusionMaskSelector 求点积再做 saturate 来完成。该向量定义在 UnityShaderVariables 中，用于指示当前正在渲染的灯光的通道。

```c
float GetShadowMaskAttenuation (float2 uv) {
	float attenuation = 1;
	#if defined (SHADOWS_SHADOWMASK)
		float4 mask = tex2D(_CameraGBufferTexture4, uv);
		attenuation = saturate(dot(mask, unity_OcclusionMaskSelector));
	#endif
	return attenuation;
}
```

在 CreateLight 中，我们还得在阴影遮罩的情况下淡化阴影，即使当前光源没有实时阴影。

```c
UnityLight CreateLight (float2 uv, float3 worldPos, float viewZ) {
	…

	#if defined(SHADOWS_SHADOWMASK)
		shadowed = true;
	#endif

	if (shadowed) {
		…
	}

	…
}
```

要正确包含烘焙阴影，请再次使用 UnityMixRealtimeAndBakedShadows 而不是旧的淡入淡出计算。

```c
	if (shadowed) {
		float shadowFadeDistance =
			UnityComputeShadowFadeDistance(worldPos, viewZ);
		float shadowFade = UnityComputeShadowFade(shadowFadeDistance);
//		shadowAttenuation = saturate(shadowAttenuation + shadowFade);
		shadowAttenuation = UnityMixRealtimeAndBakedShadows(
			shadowAttenuation, GetShadowMaskAttenuation(uv), shadowFade
		);

		…
	}
```

我们现在使用自定义的延迟着色器也获得了正确的烘焙阴影。只有一点：当我们的优化分支被执行的时候，会跳过阴影混合，所以使用阴影遮罩时无法使用这个捷径。

```c
	if (shadowed) {
		…

		#if defined(UNITY_FAST_COHERENT_DYNAMIC_BRANCHING) && defined(SHADOWS_SOFT)
			#if !defined(SHADOWS_SHADOWMASK)
				UNITY_BRANCH
				if (shadowFade > 0.99) {
					shadowAttenuation = 1;
				}
			#endif
		#endif
	}
```

## 2.4 距离阴影遮罩模式

虽然阴影遮罩模式为静态对象提供了良好的烘焙阴影，但动态对象无法从中受益，动态对象只能接收实时阴影和光照探针数据。 如果我们想在动态对象上获得良好的阴影，那么静态对象也必须投射实时阴影，这就是 Distance Shadowmask 混合光照模式的用途。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/distance-shadowmask-mode.png)  
*Distance Shadowmask 模式*  

> **我没有 Distance Shadowmask 选项？**
> 在 Unity 2017 里，使用哪种阴影遮罩模式是通过质量设置面板控制的。

使用距离阴影遮罩模式时，所有内容都使用实时阴影，乍一看，它似乎与 Baked Indirect 模式完全相同。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/realtime-shadows.png)  
*所有物体都是实时阴影*

实际上仍然有一个阴影遮罩，在此模式下，超出阴影距离的地方使用烘焙阴影和光照探针。所以这是最昂贵的模式，等于在阴影距离以内使用烘焙间接光，超出该范围使用阴影遮罩。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/realtime-shadowmask-blending.png)  
*近处实时光，远处使用阴影遮罩和探针*  

我们已经支持这种模式，因为我们使用的是 UnityMixRealtimeAndBakedShadows。为了在完全实时和烘焙阴影之间正确混合，函数会像往常一样衰减实时阴影，然后采用实时阴影和烘焙阴影两者的较小值。

## 2.5 多光源

由于阴影遮罩有四个通道，因此一次最多可支持四个重叠的光体。例如，这是一个使用光照贴图另带三个聚光灯的场景截图，我降低了主光源的强度，使聚光灯更明显。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/four-lights.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/four-lights-maps.png)  
*四盏灯的混合*   

## 2.6 支持多个带遮罩的方向光

不幸的是，事实证明，阴影遮罩只支持一个混合模式的方向灯。当有多个方向灯时，阴影衰减会出错。至少在使用前向渲染路径时如此，而延迟渲染正常工作。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/two-directional-incorrect.png)  
*两盏灯时错误的衰减*  

Unity的标准着色器也有这个问题，至少在 5.6.2 和 2017.1.0f1 版本还是如此。但是，这并不是由于光照贴图引擎的固有限制，这是使用 UNITY_LIGHT_ATTENUATION 的新方法的时的疏忽。Unity 通过 UNITY_SHADOW_COORDS 定义了阴影插值器，既用它来存储方向阴影的屏幕空间坐标，也用来存储具有阴影遮罩的其他光源的光照贴图坐标。

使用阴影遮罩的方向灯也需要光照贴图坐标。在 forward base pass 里，插值器会包含这些坐标，因为 LIGHTMAP_ON 会在需要时定义。但是，LIGHTMAP_ON 永远不会在 additive pass 中定义，这意味着附加的方向灯不具有可用的光照贴图坐标。 事实证明，在这种情况下 UNITY_LIGHT_ATTENUATION 只简单地返回 0，导致光照贴图采样错误。

所以我们不能在使用阴影遮罩的附加方向灯上依赖 UNITY_LIGHT_ATTENUATION。我们很容易识别出这种情况，这里假设我们正在使用屏幕空间方向阴影，实际在某些平台上并非如此。

```c
#if defined(FOG_LINEAR) || defined(FOG_EXP) || defined(FOG_EXP2)
	…
#endif


#if !defined(LIGHTMAP_ON) && defined(SHADOWS_SCREEN)
	#if defined(SHADOWS_SHADOWMASK) && !defined(UNITY_NO_SCREENSPACE_SHADOWS)
		#define ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS 1
	#endif
#endif
```

接下来，当我们有额外的带遮罩方向阴影时，我们还必须包括光照贴图坐标。

```c
struct Interpolators {
	…

	#if defined(LIGHTMAP_ON) || ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
		float2 lightmapUV : TEXCOORD6;
	#endif
};

…

Interpolators MyVertexProgram (VertexData v) {
	…

	#if defined(LIGHTMAP_ON) || ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
		i.lightmapUV = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
	#endif

	…
}
```

有了光照贴图坐标，我们可以再次使用 FadeShadows 函数来计算衰减。

```c
float FadeShadows (Interpolators i, float attenuation) {
	#if HANDLE_SHADOWS_BLENDING_IN_GI || ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
		…
	#endif

	return attenuation;
}
```

但这还是不对，因为我们正在给它提供有问题的衰减数据。我们必须绕过 UNITY_LIGHT_ATTENUATION 并且仅仅获取烘焙的衰减，在这种情况下我们可以通过 SHADOW_ATTENUATION 宏来完成。

```c
float FadeShadows (Interpolators i, float attenuation) {
	#if HANDLE_SHADOWS_BLENDING_IN_GI || ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
		// UNITY_LIGHT_ATTENUATION doesn't fade shadows for us.
		#if ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
			attenuation = SHADOW_ATTENUATION(i);
		#endif
		…
	#endif

	return attenuation;
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/using-a-shadowmask/two-directional-correct.png)  
*两个方向光下正确的衰减*  

> **依赖 UNITY_LIGHT_ATTENUATION 是一个好主意吗？**
> 这个宏的代码很久以来一直是稳定的，它一直是自定义着色器引用 Unity 的照明设置的最佳方式。直到 Unity 5.6.0 开始发生了变化，一套新方法被强制塞进旧的宏结构。
> Unity在 2017.3 中再次改变了附加光源的计算方法，因此支持额外的方向光，但这会给我们的解决方案和未来的光照计算带来麻烦，最快的解决方案是禁用我们的解决方案。

```c
// #define ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS 1
```

> 不幸的是，Unity的最新方法是一种打补丁，它新引入了对 clip 空间的 w 坐标的依赖——基本是唯一需要依赖的地方。 它对所有 LOD 交叉淡入的组合都不起作用，因此不过是一个 bug 代替了另一个bug。当我讲到新的脚本化渲染流水线时，我可能不会依赖 UNITY_LIGHT_ATTENUATION。

# 3 减法阴影

混合照明很不错，但它并不像纯烘焙那么便宜。如果您的目标是低性能硬件，那么混合照明就不可行了。烘焙可以工作，但您可能真的需要动态对象来在静态对象上投射阴影。 在这种情况下，您可以使用减法混合照明模式。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/subtractive-mode.png)  
*减法模式*  

切换到减法模式后，场景将变得更加明亮。这是因为静态对象现在同时使用完全烘焙的光照贴图和直接光照。与往常一样，动态物体仍然使用光照探针和直接光照。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/too-bright.png)  
*静态物体计算了两倍的光照*  

减法模式仅适用于前向渲染。使用延迟渲染时，相关对象将退化回前向渲染，就像透明对象一样。

## 3.1 减法光照

减法模式的思路是通过光照贴图照亮静态对象，同时将动态阴影也包括进去。这是通过降低阴影区域中光照贴图的强度来完成的。为此，着色器需要同时访问光照贴图和实时阴影，它还需要使用实时光来确定光照贴图必须调暗多少，这就是我们在切换到这种模式后获得双倍照明的原因。

减法光是一种近似，仅适用于单个方向光，因此仅支持主方向光的阴影。此外，我们必须以某种方式知道动态阴影区域中的间接光信息，由于我们使用的是完全烘焙的光照贴图，因此我们没有此信息。 Unity 使用了均匀的颜色来模拟环境光，而非另外采用一张间接光的光照贴图——这就是 *Realtime Shadow Color* 选项的含义，你可以在 Mixed Lighting 选项组里找到它。

在着色器中，如我们所知道，应该在定义了 LIGHTMAP_ON，SHADOWS_SCREEN 和 LIGHTMAP_SHADOW_MIXING ，而没有定义 SHADOWS_SHADOWMASK 关键字时计算减法光照。在这种情况下，让我们定义 SUBTRACTIVE_LIGHTING，以更容易使用。

```c
#if !defined(LIGHTMAP_ON) && defined(SHADOWS_SCREEN)
	#if defined(SHADOWS_SHADOWMASK) && !defined(UNITY_NO_SCREENSPACE_SHADOWS)
		#define ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS 1
	#endif
#endif

#if defined(LIGHTMAP_ON) && defined(SHADOWS_SCREEN)
	#if defined(LIGHTMAP_SHADOW_MIXING) && !defined(SHADOWS_SHADOWMASK)
		#define SUBTRACTIVE_LIGHTING 1
	#endif
#endif
```

在我们开始其他事情之前，必须先去掉双倍照明，这可以通过关闭动态光来完成，如在延迟渲染时候一样。

```c
UnityLight CreateLight (Interpolators i) {
	UnityLight light;

	#if defined(DEFERRED_PASS)  || SUBTRACTIVE_LIGHTING
		light.dir = float3(0, 1, 0);
		light.color = 0;
	#else
		…
	#endif

	return light;
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/baked-static.png)  
*静态物体只有烘焙光*

## 3.2  烘焙光的阴影

要应用减法阴影，让我们创建一个函数来调整间接光，通常来说它什么都不做。

```c
void ApplySubtractiveLighting (
	Interpolators i, inout UnityIndirect indirectLight
) {}
```

这个函数应该在我们获取了光照贴图数据之后调用：

```c
UnityIndirect CreateIndirectLight (Interpolators i, float3 viewDir) {
	…

	#if defined(FORWARD_BASE_PASS) || defined(DEFERRED_PASS)
		#if defined(LIGHTMAP_ON)
			indirectLight.diffuse =
				DecodeLightmap(UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightmapUV));
			
			#if defined(DIRLIGHTMAP_COMBINED)
				…
			#endif

			ApplySubtractiveLighting(i, indirectLight);
		#else
			indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
		#endif
		…
	#endif

	return indirectLight;
}
```

如果有减色光照，那么我们必须获取阴影衰减，我们可以简单地从 CreateLight 复制一点代码。

```c
void ApplySubtractiveLighting (
	Interpolators i, inout UnityIndirect indirectLight
) {
	#if SUBTRACTIVE_LIGHTING
		UNITY_LIGHT_ATTENUATION(attenuation, i, i.worldPos.xyz);
		attenuation = FadeShadows(i, attenuation);
	#endif
}
```

接下来，如果我们使用实时照明，必须弄清楚应该接收多少光。我们假设此信息与光照贴图中的信息相匹配。 由于光照贴图仅包含漫反射，因此我们可以计算定向光的兰伯特项。

```c
	#if SUBTRACTIVE_LIGHTING
		UNITY_LIGHT_ATTENUATION(attenuation, i, i.worldPos.xyz);
		attenuation = FadeShadows(i, attenuation);

		float ndotl = saturate(dot(i.normal, _WorldSpaceLightPos0.xyz));
	#endif
```

为了得到阴影光强度，我们必须计算 Lambert 项与衰减相乘。 但是我们已知的是完全没有遮蔽的烘焙光。因此，我们估算一下阴影会阻挡多少光线。

```c
		float ndotl = saturate(dot(i.normal, _WorldSpaceLightPos0.xyz));
		float3 shadowedLightEstimate =
			ndotl * (1 - attenuation) * _LightColor0.rgb;
```

通过从烘焙的光中减去该估计值，我们最终得到调整后的光。

```c
		float3 shadowedLightEstimate =
			ndotl * (1 - attenuation) * _LightColor0.rgb;
		float3 subtractedLight = indirectLight.diffuse - shadowedLightEstimate
		indirectLight.diffuse = subtractedLight;
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/subtracted-light.png)  
*减法光*

无论环境光如何，这种算法都会产生纯黑色阴影。为了更好地匹配场景，我们可以使用减色阴影颜色，通过 unity_ShadowColor 获得。 阴影区域不应该比这种颜色更暗，但它们可能更亮,因此取最大计算光和阴影颜色。

```c
		float3 subtractedLight = indirectLight.diffuse - shadowedLightEstimate;
		subtractedLight = max(subtractedLight, unity_ShadowColor.rgb);
		indirectLight.diffuse = subtractedLight;
```

我们还必须考虑阴影强度低于1的可能性。要应用阴影强度，请根据 _LightShadowData 的X分量在阴影和非阴影光之间进行插值。  

```c
		subtractedLight = max(subtractedLight, unity_ShadowColor.rgb);
		subtractedLight =
			lerp(subtractedLight, indirectLight.diffuse, _LightShadowData.x);
		indirectLight.diffuse = subtractedLight;
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/colored-shadows.png)  
*带颜色的阴影*

因为我们的场景的环境光强度为零，所以默认阴影颜色与场景不匹配。但这样可以很容易地看出减法阴影，所以我没有调整它。 另外，带颜色的阴影区域现在明显覆盖了所有烘焙阴影，这不是我们想要的，它应该只影响接收动态阴影的区域，而不能使烘焙阴影变亮。要正确执行此操作，请取减法光照和烘焙光照的最小值。

```c
//		indirectLight.diffuse = subtractedLight;
		indirectLight.diffuse = min(subtractedLight, indirectLight.diffuse);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/subtractive-shadows.png)  
*正确的减法阴影*

我们现在只要使用适当的阴影颜色，就可以获得正确的减法阴影。但请记住，它只是一个近似值，并且不适用于多个灯光。例如，其他烘焙光源的阴影是错误的。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/subtractive-shadows/incorrect-subtraction.png)  
*其他光源的错误减法阴影*

下一课是**实时全局光、探针体和LOD组**

---
[原教程传送门]（https://catlikecoding.com/unity/tutorials/rendering/part-17/）