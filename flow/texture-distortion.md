流体之一 纹理变形
======
伪造流体
------

 * 使用流体图调整UV坐标
 * 创建无缝动画循环
 * 控制流体外观
 * 使用导数贴图添加凹凸

这是关于创建流体材质的系列教程第一节。在这一节里，我们使用流体图来扭曲纹理。本教程假设您已经完成了 Basics 系列，再加上 Rendering 系列至少第六节凹凸。

本教程是使用 Unity 2017.4.4f1 制作的。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/tutorial-image.jpg)
*在多个方向上拉伸挤压纹理*

# 1 UV 动起来

不会移动的液体，视觉上与固体无法区分：你在看的是水、果冻还是玻璃？这个静止的水池是冻上了吗？为了确认，需要扰动一下并看它是否会变形。仅仅创造一种看起来像移动中的水的材质是不够的，它实际上必须在移动，否则就像是水的玻璃雕塑，或者是时间冻结的水。这种效果对于照片来说足够了，但不适合电影或者游戏。

大多数时候，我们只是想要一个用水、泥或熔岩制成的表面，或者视觉上像液体的魔法效果。它不需要交互，只是在随便观察时显得可信就行。因此，不需要引用复杂的流体物理模拟，我们所需要的是将一些运动添加到常规材质中，可以通过纹理的UV坐标动画来完成。

本教程中使用的技术首先由 Valve 的 Alex Vlachos 在 SIGGRAPH 2010 介绍 Portal2 的演讲 Water Flow 中详细公开描述。

## 1.1 表面着色器滑起来

关于本教程，您可以开始一个新项目，设置为使用线性颜色空间渲染。 如果您使用的是 Unity 2018，请选择默认的 3D 管道，而不是轻量级或 HD 管道。然后，创建一个新的标准表面着色器。既然我们要通过扭曲纹理映射来模拟流动表面，所以命名为DistortionFlow。下面是一个新的着色器，删除了所有注释和不需要的部分。

```c
Shader "Custom/DistortionFlow" {
	Properties {
		_Color ("Color", Color) = (1,1,1,1)
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		_Glossiness ("Smoothness", Range(0,1)) = 0.5
		_Metallic ("Metallic", Range(0,1)) = 0.0
	}
	SubShader {
		Tags { "RenderType"="Opaque" }
		LOD 200

		CGPROGRAM
		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		sampler2D _MainTex;

		struct Input {
			float2 uv_MainTex;
		};

		half _Glossiness;
		half _Metallic;
		fixed4 _Color;

		void surf (Input IN, inout SurfaceOutputStandard o) {
			fixed4 c = tex2D(_MainTex, IN.uv_MainTex) * _Color;
			o.Albedo = c.rgb;
			o.Metallic = _Metallic;
			o.Smoothness = _Glossiness;
			o.Alpha = c.a;
		}
		ENDCG
	}

	FallBack "Diffuse"
}
```

为了便于查看 UV 坐标是如何变形的，您可以使用[这张测试贴图](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/uv.png)。
![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/uv.png)
*UV 测试图*

创建一个使用上述着色器的材质，将测试纹理作为其反照率贴图。将其平铺设为 4 以便看到纹理是如何重复的。然后使用此材质在场景里添加一个 Quad。为了获得最佳观察效果，请将其围绕 X 轴旋转 90°，使其平放在 XZ 平面内，这样从任何角度查看都比较容易。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/material.png)  
![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/quad.jpg)  
*平面上的扭曲流体材质*  

## 1.2 流动的UV

流动UV坐标的代码是通用的，因此我们将它放在一个单独的 Flow.cginc 文件中。它的内容是一个FlowUV函数，参数是 UV 和时间，返回流动的新 UV 坐标。我们从最直接的位移开始，简单地把时间和坐标加起来。

```c
#if !defined(FLOW_INCLUDED)
#define FLOW_INCLUDED

float2 FlowUV (float2 uv, float time) {
	return uv + time;
}

#endif
```

在我们的着色器中引用此文件，并以主纹理坐标和当前时间为参数调用 FlowUV，Unity 提供的时间在内置变量 _Time.y中。最后使用新的 UV 坐标对纹理进行采样。

```c
		#include "Flow.cginc"

		sampler2D _MainTex;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			float2 uv = FlowUV(IN.uv_MainTex, _Time.y);
			fixed4 c = tex2D(_MainTex, uv) * _Color;
			o.Albedo = c.rgb;
			o.Metallic = _Metallic;
			o.Smoothness = _Glossiness;
			o.Alpha = c.a;
		}
```

[视频：沿对角线流动的 UV](https://thumbs.gfycat.com/HonoredJealousChinchilla-mobile.mp4)

因为我们对两个坐标增加了相同的量，所以纹理会对角滑动。 因为我们和时间做加法，所以它从右上角滑到左下角。因为我们正在使用纹理的默认封装模式，所以动画每秒都会循环。

动画仅在时间值增长时可见，编辑器处于播放模式时就是这种情况，但您也可以通过 Scene 窗口的工具栏启用 Animated Materials，在编辑模式下启用时间进度。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/animated-materials.png)  
*启用 Animated Materials*  

实际上，每次编辑器重绘场景时，材质使用的时间值都会增长。因此，即使 Animated Materials 被禁用，每次编辑时也会看到纹理滑动一点。Animated Materials 只是强制编辑器一直重绘场景，所以应该只在需要时才开启它。

## 1.3 流动方向

您可以使用速度向量来控制流动的方向和速度，而不是始终沿同一方向流动。此向量可以作为属性添加到材质中，然而，这样仍然局限于对整个材质使用相同的向量，这看起来像一个刚性的表面在滑动。为了使某些东西看起来像流动的液体，除了整体移动之外，它必须随着时间的推移而有局部变化。

我们可以添加另一个速度向量来削弱静止感，使用它来二次采样纹理，并把两个值混合。当使用两个微有不同的向量时，我们可以得到一个变形纹理。 然而，我们仍然局限于以相同的方式流动整个表面。通常开阔水域或直流这样足矣，对于更复杂的情况则不太合适。

为了支持更有趣的流体，我们必须以某种方式把材料表面的流向量复杂化。最直接的方法是通过流体贴图。这是一个包含 2D 向量的纹理，R 通道存放向量的 U 分量，G 通道存放向量 V 分量。这张纹理不需要很大，因为我们不需要急剧的突然变化，我们可以依靠双线性过滤来保持平滑。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/flowmap.png)  
*流体贴图*  

这个纹理是用卷曲噪声创建的，在 [Noise Derivatives 教程](https://catlikecoding.com/unity/tutorials/noise-derivatives/)中有解释，创建细节不在此详叙。它包含多个顺时针和逆时针的旋转流，没有任何源或槽。确保将其导入为非 sRGB 的常规 2D 纹理，因为它不包含任何颜色数据。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/texture-import.png)  
*以非 sRGB 格式导入*

将流体贴图的属性添加到我们的材质中。它不需要单独设置平铺和偏移，因此请为其指定 NoScaleOffset 属性。 默认值为一张黑色纹理，也就是没有流的情况。

```c
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		[NoScaleOffset] _FlowMap ("Flow (RG)", 2D) = "black" {}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/material-flowmap.png)  
*加了流体贴图的材质*

为流体贴图添加变量并对其进行采样以获取流向量。然后暂时以它作为反照率，将它显示出来。

```c
		sampler2D _MainTex, _FlowMap;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			float2 flowVector = tex2D(_FlowMap, IN.uv_MainTex).rg;
			float2 uv = FlowUV(IN.uv_MainTex, _Time.y);
			fixed4 c = tex2D(_MainTex, uv) * _Color;
			o.Albedo = c.rgb;
			o.Albedo = float3(flowVector, 0);
			…
		}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/tiled-flow.jpg)  
*平铺的流向量*  

纹理在场景中显得比较亮，因为它是线性空间的数据。这没关系，因为我们不应该把它用作颜色。由于表面着色器的主 UV 使用的是主纹理的平铺和偏移，因此我们的流体贴图也会平铺，而我们不需要平铺，因此将主纹理的平铺也设成1。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/material-no-tiling.png)  
![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/flow-no-tiling.jpg)  
*没有平铺的流向量*

## 1.4 定向滑动

现在我们有了流向量，我们可以为 FlowUV 函数添加对它的支持。为函数添加一个参数，然后将它们与时间相乘，再做减法。减法是因为这会使流体流向向量的方向。

```c
float2 FlowUV (float2 uv, float2 flowVector, float time) {
	return uv - flowVector * time;
}
```

将流向量传递给函数，但在此之前要确保向量有意义。和法线贴图一样，矢量可以指向任何方向，因此可能包含负分量。因此，矢量的编码方式与法线贴图相同。我们必须对其手动解码。此外，恢复原来的反照率。

```c
			float2 flowVector = tex2D(_FlowMap, IN.uv_MainTex).rg * 2 - 1;
			float2 uv = FlowUV(IN.uv_MainTex, flowVector, _Time.y);
			fixed4 c = tex2D(_MainTex, uv) * _Color;
			o.Albedo = c.rgb;
//			o.Albedo = float3(flowVector, 0);
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/distortion.jpg)  
*过于扭曲*  

我们得到了一张过于扭曲的纹理。发生这种情况是因为纹理在多个方向上移动，随着时间的推移，拉伸和挤压累积越来越强。为了防止它变得一团糟，我们必须在某个时候重置动画。最简单的方法是仅使用动画的一小部分时间。这样，时间刻度正常地从 0 上升到 1，但随后会重置为 0，形成锯齿图案。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/sawtooth.png)  
*锯齿状的时间刻度*

这是特别为流动画设计而非通常的时间，在 FlowUV 中创建锯齿级数。
```c
float2 FlowUV (float2 uv, float2 flowVector, float time) {
	float progress = frac(time);
	return uv - flowVector * progress;
}
```

[视频：每秒重置动画进度](https://thumbs.gfycat.com/BlissfulSoupyCornsnake-mobile.mp4)

我们现在可以看到纹理确实在不同方向和速度下发生扭曲。除了突然重置之外，最明显的是纹理随着扭曲的累积而迅速出现方块。这是由流体贴图的压缩引起的。 默认压缩设置使用DXT1格式，这是产生块的原因。使用比较均一的纹理时，这些误差通常不明显，但在对界限清晰的图案（如我们的测试纹理）变形时会非常明显。所以我在本教程的所有截图和视频中使用的是一个未压缩的流体贴图。

[视频：未压缩的贴图](https://thumbs.gfycat.com/BreakableAbleIzuthrush-mobile.mp4)

> 为什么不使用高分辨率的流体贴图？
> 这是可以的，但流体贴图通常会覆盖一大片面积，因此最终的有效分辨率还是比较低。只要你没有使用极端的扭曲就没有问题。本教程中的扭曲非常强，所以非常明显。

[unitypackage](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/animating-uv.unitypackage)

# 2 无缝循环

现在为止，我们设置了非统一的流动画，但它会每秒重置一次。为了使其循环而没有间断，我们必须以某种方式将 UV 恢复到其原始值，然后重新扭曲。时间只会前进，所以我们不能把失真简单反演，这样会导致水流反复来回而不是朝一个方向前进，我们必须找到另一种方式解决。

## 2.1 权重混合

我们无法避免重置失真，但我们可以尝试隐藏它。当我们接近最大失真时，我们可以将纹理淡化为黑色。如果我们也从一开始就以黑色淡入纹理开始，那么当整个表面都是黑色时会发生突然重置。虽然这种重置非常明显，但至少没有视觉上的突变。

为了实现衰减，让我们在 FlowUV 函数的输出中添加一个混合权重，将函数重命名为FlowUVW。权重放在第三个分量中，目前实际上是 1，那我们就从 1 开始。

```c
float3 FlowUVW (float2 uv, float2 flowVector, float time) {
	float progress = frac(time);
	float3 uvw;
	uvw.xy = uv - flowVector * progress;
	uvw.z = 1;
	return uvw;
}
```

将纹理与返回的权重相乘以淡化纹理。

```c
			float3 uvw = FlowUVW(IN.uv_MainTex, flowVector, _Time.y);
			fixed4 c = tex2D(_MainTex, uvw.xy) * uvw.z * _Color;
```

## 2.2 锯齿波

现在我们需要一个权重函数 w(p)，使 w(0) = w(1) = 0，在一半的地方达到最强，也就是 w(1/2) = 1. 最简单的符合要求的函数就是三角波 w(p) = 1 - |1 - 2p|. 试着把这个函数赋给权重。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/seamless-looping/triangle.png)  
*循环往复的三角波*  

```c
	uvw.z = 1 - abs(1 - 2 * progress);
```
[视频：三角波调制](https://thumbs.gfycat.com/ExcellentGlitteringGrub-mobile.mp4)

> **为什么不用一个更平滑的函数？**
> 你也可以试着用正弦波或者 smoothstep 函数，不过这些函数会使着色器变复杂，但不影响最终效果，一个三角波就够了。

## 2.3 时间偏移

虽然从技术上讲我们已经消除了视觉上的突变，但我们引入了黑色脉冲。脉冲非常明显，因为它在所有位置同时发生。如果我们可以在时间维度将其稀释，可能就不那么明显了。我们可以在表面上施加不同的时间偏移以实现这一点，低频的 Perlin 噪声非常适用于这种情况。我们不想再加一张纹理，而是将噪声打包在流体贴图中。[这里](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/seamless-looping/flowmap.png)是一张与之前相同的流体贴图，但现在其 A 通道中存放着噪声，噪声与流向量无关。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/seamless-looping/flowmap.png)  
*A 通道里存了噪声的流体图*  

为了明确指明流体贴图里的噪声，我们更新一下属性的标签。

```c
[NoScaleOffset] _FlowMap ("Flow (RG, A noise)", 2D) = "black" {}
```

对噪声采样，加到时间上，然后传给 FlowUVW。

```c
			float2 flowVector = tex2D(_FlowMap, IN.uv_MainTex).rg * 2 - 1;
			float noise = tex2D(_FlowMap, IN.uv_MainTex).a;
			float time = _Time.y + noise;
			float3 uvw = FlowUVW(IN.uv_MainTex, flowVector, time);
```

![](https://thumbs.gfycat.com/TotalCaringDog-mobile.mp4)  
*带偏移量的时间*

> **为什么采样了两次？**
> 为了演示一下：着色器的编译器会把他优化成一次采样。

黑色脉冲还在，但已经变成了一种比较协调的在表面上扩散的波，与均匀脉冲相比，更容易混淆。此外还有个意外收获，时间偏移还使扭曲的进度不均匀，导致整体扭曲更加富于变化。

## 2.4 把两种扭曲合起来

如果不想使用黑色，我们也可以与其他东西混合，例如原始的无扭曲纹理。但这样我们会看到固定的纹理淡入淡出，会破坏流动的错觉。可以与另一个扭曲纹理混合以解决这个问题，这要求我们对纹理进行两次采样，每次采样具有不同的 UVW 数据。

因此我们最终得到两个脉冲，A和B. 当A的权重为0时，B应为1，反之亦然。这样就去掉了黑色脉冲。这是通过将 B 的相位移动半个周期来完成的，也就是加0.5。但这都属于 FlowUVW 的工作细节，所以先添加一个布尔参数来指示它返回 UVW 的 A 变体还是 B 变体。

```c
float3 FlowUVW (float2 uv, float2 flowVector, float time, bool flowB) {
	float phaseOffset = flowB ? 0.5 : 0;
	float progress = frac(time + phaseOffset);
	float3 uvw;
	uvw.xy = uv - flowVector * progress;
	uvw.z = 1 - abs(1 - 2 * progress);
	return uvw;
}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/seamless-looping/double-triangle.png)  
*A 和 B 的权重和总是 1*  

我们现在必须调用两次 FlowUVW ，一次使用 false、一次使用 true 作为其最后一个参数。然后对纹理进行两次采样，分别乘以权重再相加以得到最终的反照率。

```c
			float time = _Time.y + noise;

			float3 uvwA = FlowUVW(IN.uv_MainTex, flowVector, time, false);
			float3 uvwB = FlowUVW(IN.uv_MainTex, flowVector, time, true);

			fixed4 texA = tex2D(_MainTex, uvwA.xy) * uvwA.z;
			fixed4 texB = tex2D(_MainTex, uvwB.xy) * uvwB.z;

			fixed4 c = (texA + texB) * _Color;
```

![](https://thumbs.gfycat.com/AbleIdioticLeopard-mobile.mp4)  
*混合两个相位*

黑色脉冲已经消失了，它作为波仍然存在，但现在作为两个阶段之间的过渡，就远没那么明显了。

两个模式之间混合的副作用是它们的周期实际上减半，这使我们动画的总时间减半——它现在每秒循环两次。但是其实不必两次使用相同的模式，我们可以将 B 的 UV 坐标偏移半个单位，就能使图案不一样——而且还是使用相同的纹理——而不引入任何方向性偏移。

```c
uvw.xy = uv - flowVector * progress + phaseOffset;
```

![](https://thumbs.gfycat.com/GrimUntimelyCaecilian-mobile.mp4)  
*A 和 B 的 UV 不同*

因为我们使用规律的测试图案，所以 A 和 B 的白色网格线会重叠，但是他们的方块的颜色是不同的，结果是动画在两种颜色配置之间交替，并且每过一秒钟重复一次。

## 2.5 UV 跳跃

除了将 A 和 B 的 UV 恒定偏移半个单位外，还可以按照相位偏移 UV。这将导致动画随时间变化，因此要循环回完全相同的状态需要更长的时间。

我们可以根据时间简单地滑动 UV 坐标，但这会导致整个动画滑动，引入方向性偏移。另外我们可以在每个阶段保持 UV 偏移恒定，并在相位变化时跳跃式偏移 UV 以避免视觉上的滑动。换句话说，我们每次权重为 0 时都会进行 UV 跳跃。方法是把 UV 加上时间的整数部分乘跳跃偏移。修改 FlowUVW 以支持此功能，并添加新参数以指定跳转向量。

```c
float3 FlowUVW (
	float2 uv, float2 flowVector, float2 jump, float time, bool flowB
) {
	float phaseOffset = flowB ? 0.5 : 0;
	float progress = frac(time + phaseOffset);
	float3 uvw;
	uvw.xy = uv - flowVector * progress + phaseOffset;
	uvw.xy += (time - progress) * jump;
	uvw.z = 1 - abs(1 - 2 * progress);
	return uvw;
}
```

给着色器添加两个属性以控制跳跃。我们使用两个 float 而非一个向量，目的是使用滑杆控制范围。因为参与混合的两个模式相位相差 1/2，我们的动画就已经在每 1 个周期里包括了 0 → 1/2 的 UV 偏移。UV 跳跃以此为基础叠加，也就是说如果跳跃值为二分之一，就会在每 2 个周期内形成 0 → 1/2 → 1/2 → 0 的进行，这不是我们想要的。跳跃值最多不超过四分之一，也就是在 4 个周期内构成 0 → 1/2 → 1/4 → 3/4 → 1/2 → 0 → 3/4 → 1/4 的进行。负值也可以，同样不能低于负四分之一。跳跃值为 -0.25 时，序列是 0 → 1/2 → 3/4 → 1/4 → 1/2 → 0 → 1/4 → 3/4.

```c
		[NoScaleOffset] _FlowMap ("Flow (RG, A noise)", 2D) = "black" {}
		_UJump ("U jump per phase", Range(-0.25, 0.25)) = 0.25
		_VJump ("V jump per phase", Range(-0.25, 0.25)) = 0.25
```

继续把 float 变量加到 shader 里，使用这两个值构造跳跃值向量，传给 FlowUVW.

```c
		sampler2D _MainTex, _FlowMap;
		float _UJump, _VJump;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			float2 flowVector = tex2D(_FlowMap, IN.uv_MainTex).rg * 2 - 1;
			float noise = tex2D(_FlowMap, IN.uv_MainTex).a;
			float time = _Time.y + noise;
			float2 jump = float2(_UJump, _VJump);

			float3 uvwA = FlowUVW(IN.uv_MainTex, flowVector, jump, time, false);
			float3 uvwB = FlowUVW(IN.uv_MainTex, flowVector, jump, time, true);

			…
		}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/seamless-looping/material.png)  
[视频](https://thumbs.gfycat.com/PowerlessDenseArrowworm-mobile.mp4)  
**跳跃值最大时的材质**

在跳跃值最大时，我们每个循环序列有八个 UV 偏移。因为每个阶段经历两个偏移，且每个阶段长一秒，所以我们的动画现在每四秒循环一次。

## 2.6 对跳跃值的分析

为了更好地了解 UV 跳跃的工作原理，您可以将流向量设置为零，这样就可以专注于偏移。首先考虑没有任何跳跃的动画，即原始的交替模式。

[视频：跳跃 0 ，时长 1 秒](https://thumbs.gfycat.com/TightRelievedGoa-mobile.mp4) 

可以看到每个方块在两种颜色之间交替。还可以看到交替的纹理偏移量是统一的 1/2，但并不是很明显，并且没有方向性偏移。接下来看一下两个维度都是最大跳跃值的动画。

[视频：跳跃 0.25 ，时长 4 秒](https://thumbs.gfycat.com/FakeUnknownDassierat-mobile.mp4)

结果看起来和之前不同，因为跳跃四分之一会导致我们的测试纹理的网格线移动，在正方形和十字形之间交替。 白线仍然没有显示出方向性偏移，但色块已经有了方向性偏移。图案在对角移动，但不是特别明显。 它向前迈出半步，然后向后退四分之一，然后重复。如果我们使用 -0.25 的最小跳跃，那么它将向前迈出半步，然后向前迈出四分之一，然后重复。要使方向性偏移更明显，请使用非对称的跳跃，例如0.2。

[视频：跳跃 0.2 ，时长 2.5 秒](https://thumbs.gfycat.com/RequiredIckyEnglishsetter-mobile.mp4)

在这种情况下，白色网格线也似乎在移动。但是因为我们仍在使用相当接近于对称的大跳跃，所以可以将移动解释为多个方向，具体取决于您对图像如何看。如果你改变了看图的思路，就很容易丢掉你之前认为它流动的方向。

因为我们使用 0.2 的跳跃，动画在五个阶段后重复，所以总共五秒钟。 但是，因为我们在两个偏移相位之间进行混合，所以在每个相位的正中间存在一个潜在的交叉点。如果动画在奇数个阶段之后循环，那么它由于阶段跨越中途，实际上循环了两次。所以在这种情况下，持续时间仅为2.5秒。

U 和 V 的跳跃不一定相同。除了影响方向性偏移，每个维度使用不同的跳跃值也会影响循环持续时间。例如，考虑 U 跳跃为0.25，V 跳跃为0.1。U每四个阶段一循环，而 V 每十个阶段一循环。因此，在四个阶段之后，U 已经循环一轮而 V 还没有，所以总的动画也没有完成一轮循环。只有当 U 和 V 在同一阶段结束时同时完成一个循环，我们才能到达动画的结尾。当使用有理数进行跳跃时，循环持续时间等于其分母的最小公倍数。在 0.25 和 0.1 的情况下，分母为 4 和 10，所以最小公倍数是20。

选择跳跃向量，没有统一的方法以达到一个很长的循环持续时间。例如，如果我们用 0.25 和 0.2 代替 0.25 和 0.1，持续时间会更长或更短吗？由于 4 和 5 的最小公倍数也是 20 ，因此持续时间是相同的。而且，虽然你可以提出理论上需要很长时间甚至永远不循环的值，但大多数实际上并不实用。 我们无法察觉太小的变化，更何况还有数值精度的限制，这可能导致理论上良好的跳跃值在随机观察下完全没有变化，或者循环比预期来的更快。

我认为好的跳跃值——当然除了 0 ——应该在 0.2 到 0.25 之间，正的负的都可以。我曾经提出过 6/25 = 0.24 和 5/24 ≈ 0.2083333 这一对值非常合乎标准。前者在 25 个阶段内完成 6 个循环，后者在 24 个周期内完成 5 个循环，总的循环阶段数理论上在 600 以上，按每秒钟一阶段算需要至少十分钟。

在本节教程的后面部分，为了使循环时间更短，我会把跳跃值设为 0。

[unitypackage](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/seamless-looping/seamless-looping.unitypackage)

# 3 调整动画

现在我们有了一个基本的流动画，让我们再为它添加一些配置选项，这样就可以对它的外观进行一些调整。

## 3.1 平铺

首先，让我们可以平铺扭曲用的纹理。我们不能只依赖表面着色器的主平铺和偏移，因为它们也会影响流体贴图。所以，我们需要单独的纹理平铺配置。既然是用于扭曲，通常只有方形纹理才有意义，因此我们只需要一个平铺值。

为了不使平铺影响流体贴图，我们必须在采样流体贴图之后再将其用于 UV ，不过要在添加阶段 B 的偏移之前，因此必须在 FlowUVW 中完成，这意味着我们的函数需要一个平铺参数。

```c
float3 FlowUVW (
	float2 uv, float2 flowVector, float2 jump,
	float tiling, float time, bool flowB
) {
	…
//	uvw.xy = uv - flowVector * progress + phaseOffset;
	uvw.xy = uv - flowVector * progress;
	uvw.xy *= tiling;
	uvw.xy += phaseOffset;
	…
}
```

给我们的着色器添加一个平铺属性，默认值为1。

```c
		_UJump ("U jump per phase", Range(-0.25, 0.25)) = 0.25
		_VJump ("V jump per phase", Range(-0.25, 0.25)) = -0.25
		_Tiling ("Tiling", Float) = 1
```

然后添加变量并将其传递给 FlowUVW。

```c
		float _UJump, _VJump, _Tiling;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			…

			float3 uvwA = FlowUVW(
				IN.uv_MainTex, flowVector, jump, _Tiling, time, false
			);
			float3 uvwB = FlowUVW(
				IN.uv_MainTex, flowVector, jump, _Tiling, time, true
			);

			…
		}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animation-tweaks/material-tiling.png)
[视频](https://thumbs.gfycat.com/AdorableHonorableAmericanbobtail-mobile.mp4) 
*平铺设为 2，时长还是 1 秒*

当平铺设置为 2 时，动画的流速似乎是之前的两倍，原因仅仅是纹理被缩放过。当没有 UV 跳跃时，动画循环时长仍然是一秒。

## 3.2 动画速度

缩放时间可以直接控制动画的速度。这会影响整个动画，也影响其持续时间。为着色器添加一个速度属性以支持此功能。

```c
		_Tiling ("Tiling", Float) = 1
		_Speed ("Speed", Float) = 1
```

只需将 _Time.y 乘以相应的变量即可，然后再加噪声值，因此时间偏移不受影响。

```c
		float _UJump, _VJump, _Tiling, _Speed;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			float2 flowVector = tex2D(_FlowMap, IN.uv_MainTex).rg * 2 - 1;
			float noise = tex2D(_FlowMap, IN.uv_MainTex).a;
			float time = _Time.y * _Speed + noise;
			…
		}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animation-tweaks/material-speed.png)  
<iframe src='https://gfycat.com/ifr/OrderlyPoorIndochinahogdeer' frameborder='0' scrolling='no' allowfullscreen width='640' height='284'></iframe><p> <a href="https://gfycat.com/OrderlyPoorIndochinahogdeer">via Gfycat</a></p>
*速度 0.5，时长现在是 2 秒*

## 3.3 流的强度

流速由流体贴图决定。我们可以通过调整动画速度来加快或减慢速度，但这也会影响阶段和动画的时长。改变表观流速的另一种方法是缩放流向量。通过调节流量的强度，我们可以加快减慢速度，甚至逆转它，而不会影响时间。这也改变了扭曲量。为达到此目的，向着色器添加 Flow Strength 属性。

```c
		_Speed ("Speed", Float) = 1
		_FlowStrength ("Flow Strength", Float) = 1
```

只需将流向量乘以相应的变量即可。

```c
		float _UJump, _VJump, _Tiling, _Speed, _FlowStrength;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			float2 flowVector = tex2D(_FlowMap, IN.uv_MainTex).rg * 2 - 1;
			flowVector *= _FlowStrength;
			…
		}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animation-tweaks/material-flow-strength.png)  
![](https://thumbs.gfycat.com/BabyishColorlessKiskadee-small.gif)
*流强度 0.25，时长 2s*  

## 3.4 流偏移

另一个可能的调整是控制动画开始的位置。到目前为止，我们在每个阶段开始时都处于零扭曲状态，进展到最大扭曲。由于相位的权重在中间点达到 1 ，所以当扭曲为一半强度时，模式最清晰。 因此，我们大多数时候看到的是半扭曲的纹理。这样配置通常很好，但并非总是如此。例如，在 Portal 2 中，漂浮的废墟纹理经常呈现的是其未扭曲状态。这是通过扭曲 UV 坐标时将流量偏移 -0.5 来完成的。

我们也来支持这个，向 FlowUVW 添加 flowOffset 参数。把 progress 偏移 flowOffset 之后再与流向量相乘。

```c
float3 FlowUVW (
	float2 uv, float2 flowVector, float2 jump,
	float flowOffset, float tiling, float time, bool flowB
) {
	float phaseOffset = flowB ? 0.5 : 0;
	float progress = frac(time + phaseOffset);
	float3 uvw;
	uvw.xy = uv - flowVector * (progress + flowOffset);
	uvw.xy *= tiling;
	uvw.xy += phaseOffset;
	uvw.xy += (time - progress) * jump;
	uvw.z = 1 - abs(1 - 2 * progress);
	return uvw;
}
```

接下来，添加一个属性来控制着色器的流偏移量。实践中常用值为 0 和 -0.5，也可以尝试其他值。

```c
		_FlowStrength ("Flow Strength", Float) = 1
		_FlowOffset ("Flow Offset", Float) = 0
```

把相关变量传给 FlowUVW.

```c
		float _UJump, _VJump, _Tiling, _Speed, _FlowStrength, _FlowOffset;

		…

		void surf (Input IN, inout SurfaceOutputStandard o) {
			…

			float3 uvwA = FlowUVW(
				IN.uv_MainTex, flowVector, jump,
				_FlowOffset, _Tiling, time, false
			);
			float3 uvwB = FlowUVW(
				IN.uv_MainTex, flowVector, jump,
				_FlowOffset, _Tiling, time, true
			);

			…
		}
```

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animation-tweaks/material-flow-offset.png)  
![](https://thumbs.gfycat.com/ElementaryOptimisticGavial-small.gif)
*流偏移设为 -0.5*  

流量偏移为-0.5时，每个阶段的的峰值点时没有扭曲。但由于时间偏移的存在，整体结果仍然是扭曲的。

[unitypackage](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animation-tweaks/animation-tweaks.unitypackage)

# 4 纹理
