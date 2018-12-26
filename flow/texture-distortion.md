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

不会移动的液体，视觉上与固体无法区分：你在看的是水、果冻还是玻璃？这个静止的水池是冻上了吗？为了确认，扰动一下并看它是否变形。仅仅创造一种看起来像移动的水的材质是不够的，它实际上必须移动，否则就像是水的玻璃雕塑，或者是时间冻结的水。这种对于照片来说足够了，但不适合电影或者游戏。

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

您可以使用速度向量来控制流动的方向和速度，而不是始终沿同一方向流动。此向量可以作为属性添加到材质中，然而，我们仍然局限于对整个材质使用相同的向量，这看起来像一个刚性的表面在滑动。为了使某些东西看起来像流动的液体，除了整体移动之外，它必须随着时间的推移而有局部变化。

我们可以添加另一个速度向量来削弱静止感，使用它来二次采样纹理，并把两个值混合。当使用两个微有不同的向量时，我们可以得到一个变形纹理。 然而，我们仍然局限于以相同的方式流动整个表面。通常开阔水域或直流这样足矣，但对于更复杂的情况则不太合适。

为了支持更有趣的流体，我们必须以某种方式把材料表面的流向量复杂化。最直接的方法是通过流体贴图。这是一个包含 2D 向量的纹理，R 通道存放向量的 U 分量，G 通道存放向量 V 分量。这张纹理不需要很大，因为我们不需要急剧的突然变化，我们可以依靠双线性过滤来保持平滑。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/flowmap.png)  
*流体贴图*  

这个纹理是用卷曲噪声创建的，在 [Noise Derivatives 教程](https://catlikecoding.com/unity/tutorials/noise-derivatives/)中有解释，创建细节不在此详叙。它包含多个顺时针和逆时针的旋转流，没有任何源或槽。确保将其导入为非 sRGB 的常规 2D 纹理，因为它不包含任何颜色数据。

![](https://catlikecoding.com/unity/tutorials/flow/texture-distortion/animating-uv/texture-import.png)  
*以非 sRGB 格式导入*

将流体贴图的属性添加到我们的材质中。它不需要单独设置平铺和偏移，因此请为其指定 NoScaleOffset 属性。 默认值情况下，没有流，对应一张黑色纹理。

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

纹理在场景中显得比较亮，因为它是线性空间的数据。这没关系，因为我们不应该把它用作颜色。由于表面着色器的主 UV 使用的是主纹理的平铺和偏移，因此我们的流图也会平铺，而我们不需要平铺流图，因此将主纹理的平铺也设成1。

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

# 2 无缝循环
