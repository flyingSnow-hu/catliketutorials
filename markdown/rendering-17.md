渲染之十七 混合光照
======

 * 只烘焙间接光
 * 烘焙阴影和实时阴影的混合
 * 处理代码修改和错误
 * 处理减色光源
 
这一篇是渲染教程的第十七部分，上一部分里我们通过光照贴图增加了对静态光照的支持，这次，我们将朝着烘焙和实时光照结合的方向前进。

这篇教程是使用 Unity 5.6.0 制作的。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/tutorial-image.jpg)  
*烘焙和实时光照的混合*  


# 1.烘焙间接光

光照贴图允许我们预计算光照，这会减少 GPU 必须实时完成的工作量，代价是增加了纹理所占内存，除此之外，它还增加了间接光照。但正如我们上次看到的那样，光照贴图也存在局限性：首先，高光无法烘焙；其次，烘焙光源影响动态物体仅能通过光照探针。第三，烘焙光源不会投射实时阴影。

你可以在下面的屏幕截图中看到完全实时和完全烘焙光照之间的区别。这个场景取自上一节教程，我移动了一些球体位置，并把所有球体设为动态物体， 其他物体都是静态的，渲染路径是前向渲染。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/fully-realtime.png)  

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/fully-baked.png)  
*完全实时和完全烘焙光照*  

我没有调整光照探针，所以现在它们的位置不太明显，因为静态几何体较少，由此产生的探针光照有点偏，这使得在使用时更容易注意到。

## 1.1 混合模式

间接光照是烘焙拥有的特点而实时照明不具备，因为它需要光照贴图。由于间接光可以为场景增添很多真实感，如果我们可以将它与实时光照结合起来会很好，这是可能的，虽然这意味着阴影变得更加昂贵。将 Mixed Lighting 的 Lighting Mode 设置为 Baked Indirect。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/baked-indirect.png)  
*Mixed Lighting -> Baked Indirect* 

我们已经在上一节教程中切换到了这种模式，但当时我们只使用了完全烘焙。结果混合照明模式没有起作用，要使用混合照明，必须将光源的模式设置为 Mixed。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/mixed-light.png)  
*混合模式的主光源*  

将主方向光修改为混合模式后，会发生两件事。  
首先，Unity 会再次烘焙光照贴图，这次，它只会存储间接光，因此生成的光照贴图会比以前暗很多。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/lightmap-baked.png)  

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/lightmap-indirect.png)  
*完全烘焙 vs. 只有间接光的光照贴图*

其次，所有物体都会被照亮，如同主光源被设置为实时一样。但有一点不同：向静态物体添加间接光是通过光照贴图，而不是球谐或探针。动态物体仍然通过光照探针来获得间接光。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/mixed-lighting.png)  
*混合光照，实时的直接光加上烘焙的间接光*  

我们不必为了支持它而修改着色器，因为 forward base pass 已经将光照贴图数据和主方向光组合在一起。像往常一样，其余的灯可以获得 additive pass。 使用延迟渲染路径时，主灯也会获得一个 pass。

>**混合光照可以在运行时动态调整吗？**  
>可以，因为它们用于实时光照，但是，他们的烘焙数据部分是静态的。所以在运行时你只能对光源进行微小调整，比如改变一点强度。过大的变化将使烘焙和实时光照之间的不同步特别明显。

## 1.2 更新着色器

一开始一切似乎都很好。 然而事实证明，阴影衰减不再适用于方向光。只要大辐减少阴影距离，就容易看到阴影被切断了。

![](https://catlikecoding.com/unity/tutorials/rendering/part-17/baking-indirect-light/shadows-custom.png)  
*阴影衰减，标准着色器 vs. 我们的着色器*  

Unity 曾经长时间采用一种混合光照模式，但实际上在 Unity 5 中已经无法使用这种模式了。Unity 5.6 中增加了一种新的混合光照模式，这就是我们现在使用的模式。 增加此新模式后，UNITY_LIGHT_ATTENUATION 宏背后的代码已经更改。我们在使用全烘焙或全实时光照时没有注意这一点，但我们现在必须更新代码，以使用新的混合光照。由于这是最近发生的重大变化，我们必须注意 bug。

第一个要修改的是不再使用 SHADOW_COORDS 宏来定义阴影坐标的插值器，我们必须使用新的 UNITY_SHADOW_COORDS 宏。

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
> 至于为何进行此更改，唯一的线索是 AutoLight 中的注释，其中写到：“handles shadows in the depths of the GI function for performance reasons”（出于性能原因，在 GI 函数中处理深度中的阴影）。 由于着色器编译器随意移动代码，这等于什么也没有告诉我们。 即使这个特殊情况有一个充分的理由，也是无从查证，因为 Unity 的着色器代码已经变得非常纠结。所以，我不知道。

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

## 2.2 添加一个阴影遮罩到 G 缓冲区

阴影遮罩现在可以在前向渲染下使用，如果想要在延迟渲染下使用，我们还需要做一些工作。 具体来说，我们必须在需要时将阴影遮罩信息添加为额外的 G 缓冲区，因此，在定义 SHADOWS_SHADOWMASK 时，将另一个缓冲区添加到 FragmentOutput 结构中。