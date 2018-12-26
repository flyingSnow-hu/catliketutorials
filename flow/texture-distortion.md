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

## 1.1 表面着色器动起来

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