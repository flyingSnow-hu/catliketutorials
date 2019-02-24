渲染之四 第一个光源
======
[原教程传送门](https://catlikecoding.com/unity/tutorials/rendering/part-4/)
---

 * 把法线从物体空间变换到世界空间
 * 方向光
 * 计算漫反射和高光反射
 * 能量守恒
 * 金属度工作流
 * 借鉴 Unity 的 PBS 算法

这一篇是渲染教程的第四部分，上一部分是有关复合纹理贴图，这次，我们要看看如何计算光。

这一篇教程是使用 Unity 5.4.0b17 制作的。


![](https://catlikecoding.com/unity/tutorials/rendering/part-4/tutorial-image.jpg)  
*是时候让光线照亮物体了*  

# 1 法线

我们能看到物体，是由于我们的眼睛可以探测到电磁辐射。最小的一个量子的光被称为光子。我们可以看到的电磁波谱只有一部分，这部分对于我们来说是可见光，其余的部分是我们不可见的。

> **完整的电磁频谱是什么样的？**  
> 光谱被分成光谱带。从低频到高频依次被称为无线电波、微波、红外线、可见光、紫外线、X 射线和伽马射线。  

能够发出光线的物体被称为光源。这些光中的一部分会撞到物体，一部分从物体上反弹回来。如果某束光继续前进击中了我们的眼睛——或者摄像机镜头——那么我们就看到了物体。

为了计算出这整个流程，我们必须知道物体的表面。我们现在已经知道了它的位置，但不知道它的方向。为此，我们需要表面法向量。

## 1.1 使用模型的法线

复制我们的第一个着色器，作为我们的第一个光照着色器。用这个着色器新建一个材质，并把它赋给场景上的立方体和球体。让这些物体具有不同的旋转和缩放，其中一些是非统一缩放，让场景复杂一点。

```c
Shader "Custom/My First Lighting Shader" {
	…
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/objects.png)  
*一些立方体和球体*  

Unity 的立方体和球体包括了顶点法线，我们可以抓过来然后直接传给片段着色器。

```c
			struct VertexData {
				float4 position : POSITION;
				float3 normal : NORMAL;
				float2 uv : TEXCOORD0;
			};
			
			struct Interpolators {
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
				float3 normal : TEXCOORD1;
			};

			Interpolators MyVertexProgram (VertexData v) {
				Interpolators i;
				i.uv = TRANSFORM_TEX(v.uv, _MainTex);
				i.position = mul(UNITY_MATRIX_MVP, v.position);
				i.normal = v.normal;
				return i;
			}
```

现在我们可以在着色器里把法线显示出来。

```c
			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				return float4(i.normal * 0.5 + 0.5, 1);
			}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/batching.png)  
*把法线向量作为颜色显示*

这些是原始法线，直接从模型取出的。立方体的面看起来是平的，因为每一个面都是独立的四边形，四个顶点的法线都朝向同一方向。相反，球体的顶点法线都指向不同的方向，就会导致平滑的插值。

## 1.2 动态合批

立方体的法线还有一点比较奇怪。讲道理来说每一个立方体应该显示的是同样的颜色结构，但是现在不是。更奇怪的是，立方体的颜色还会随着观察角度而变化。

![](https://thumbs.gfycat.com/AfraidRashAlbertosaurus-small.gif)  
*变色的立方体*

这些都是由于动态合批（Dynamic Batching）。Unity 会动态地合并小网格，以减少绘制调用（Draw Call，DC）。球体对于动态合批来说有点大，所以不受影响。立方体就是个不错的目标了。

为了合并网格，物体需要从局部空间变换到世界空间。物体是否参与合并以及如何合并，取决于物体的渲染排序。这个过程也会影响法线，所以我们看到颜色变了。

如果你想的话，可以在 player settings 里关掉动态合批。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/batching-settings.png)  
*合批设置*

除了动态合批，Unity 还有静态合批。不同之处在于它发生在静态几何体上，不过也同样包括了法线变换到世界空间的步骤。它是在运行时发生的。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/no-batching.png)  
*没有动态合批的法线*

虽然你需要了解动态合批，但没有什么可担心的。事实上，我们也要对法线做同样的事情。所以你可以让它保持启用状态。

## 1.3 世界空间下的法线

除了动态合批的物体外，我们所有的法线都在物体空间中。但我们必须知道世界空间的表面方向，所以必须将法线从物体空间转换到世界空间。我们需要物体的变换矩阵。

Unity 将对象的所有变换层次折叠成单个变换矩阵，就像我们在[第 1 节](https://catlikecoding.com/unity/tutorials/rendering/part-1)中所做的那样。我们可以将其写为 $O=T_{1}T_{2}T_{3}...$ 其中 $T$ 是单独的转换，$O$ 是组合转换。该矩阵称为物体到世界矩阵。

Unity 提供 float4x4 unity_ObjectToWorld 变量以在着色器中获取此矩阵，该变量在 UnityShaderVariables 中定义。在顶点着色器中将此矩阵与物体空间的法线相乘，以将物体空间法线转换到世界空间。因为法线代表方向，应该忽略位移。所以第四个齐次坐标分量必须为零。

```c
			Interpolators MyVertexProgram (VertexData v) {
				Interpolators i;
				i.position = mul(UNITY_MATRIX_MVP, v.position);
				i.normal = mul(unity_ObjectToWorld, float4(v.normal, 0));
				i.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return i;
			}
```

或者，我们可以仅使用矩阵的 3×3 部分。不过编译后的代码结果完全相同，因为编译器将消去与常数零相乘的内容。

```c
				i.normal = mul((float3x3)unity_ObjectToWorld, v.normal);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/object-to-world.png)  
*从物体空间到世界空间*  

现在法线到了世界空间，不过有些法线看起来比较亮，这是因为它们也被缩放了。所以我们需要在变换之后重新归一化。

```c
			i.normal = mul(unity_ObjectToWorld, float4(v.normal, 0));
			i.normal = normalize(i.normal);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/normalized.png)  
*归一化之后的法线*

我们重新把法线归一化了，不过在三轴没有统一缩放的物体上看起来有点奇怪。这是因为，当一个表面在某一轴向上被拉伸的时候，其法线和顶点拉伸的方式不同。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/scaling-incorrect.png)  
*X 轴缩放，顶点和法线都乘以 ½*  

当缩放不是三轴统一的时候，法线的缩放实际上是相反的，这样才能匹配缩放后的表面形状。即便是统一缩放也应当这样计算。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/scaling-correct.png)  
*X 轴缩放，顶点乘以 ½，法线乘以 2*

不过缩放需要逆行，旋转还是要正向进行，这如何是好？

上文我们把一个物体的变换矩阵描述为  $O=T_{1}T_{2}T_{3}...$ ，不过其实可以更加深入一步。我们知道每一层级都结合了三种变换：缩放、旋转和平移。所以这里的每一个 $T$ 都可以拆解为 $SRP$.

也就是说 $O=S_{1}R_{1}P_{1}S_{2}R_{2}P_{2}S_{3}R_{3}P_{3}...$，不过我们这里简单考虑 $O=S_{1}R_{1}P_{1}S_{2}R_{2}P_{2}$.

由于法线是方向向量，我们不关心其平移，所以进一步简化为 $O=S_{1}R_{1}S_{2}R_{2}$，并且只考虑矩阵的 3×3 部分。

我们需要的是逆转其缩放，而保持旋转不变。所以我们所求的新变换矩阵是 $N=S_{1}^{-1}R_{1}S_{2}^{-1}R_{2}$

> **逆矩阵是什么？**  
> 矩阵$M$的逆矩阵是$M^{-1}$。其含义是当这两个矩阵相乘的时候，其作用会相互抵消，它们互为逆矩阵。也就是$MM^{-1}=M^{-1}M=I$.  
>   
> 如果需要抵消一串操作，需要以相反次序依次乘逆矩阵，其思维模型类似于先穿袜子就要先脱鞋。也就是说 ${(AB)}^{-1}=B^{-1}A^{-1}$.  
>   
> 对于一个标量 $x$，它的逆是$\frac{1}{x}$，因为$\frac{x}{x}=1$。这也暗示了 0 不能求逆，同样，不是每个矩阵都有逆矩阵。  
>   
> 我们这里遇到的矩阵都是缩放、旋转、平移矩阵，由于我们不会缩放到 0，所以所有的矩阵都是可逆的。  
>   
> 平移矩阵的逆是把其第四列的 XYZ 取相反数。  
> $\begin{bmatrix} 1&0&0&x\\\\ 0&1&0&y\\\\ 0&0&1&z\\\\ 0&0&0&1 \end{bmatrix}^{-1} = \begin{bmatrix}1&0&0&-x\\\\ 0&1&0&-y\\\\ 0&0&1&-z\\\\ 0&0&0&1 \end{bmatrix}$  
>   
> 缩放矩阵的逆是对其对角线元素求倒数，只需考虑其 3×3 部分。
> $\begin{bmatrix} x&0&0\\\\ 0&y&0\\\\ 0&0&z\\\\ \end{bmatrix}^{-1} = \begin{bmatrix} \frac{1}{x}&0&0\\\\ 0&\frac{1}{y}&0\\\\ 0&0&\frac{1}{z}\\\\ \end{bmatrix}$  
>   
> 旋转矩阵可以按轴向分解考虑，比如绕着 Z 轴旋转。旋转 
$z$ 弧度的反操作是旋转 $-z$ 弧度。如果你了解正弦波和余弦波，就应该知道 $sin(-z) = -sin(z)$，$cos(-z) = cos(z)$，所以问题就简单了。  
> $\begin{bmatrix} cos(z)&-sin(z)&0\\\\ sin(z)&cos(z)&0\\\\ 0&0&1\\\\ \end{bmatrix}^{-1} = \begin{bmatrix} cos(z)&sin(z)&0\\\\ -sin(z)&cos(z)&0\\\\ 0&0&1\\\\ \end{bmatrix}$  
>   
> 注意旋转矩阵的逆等同于沿对角线翻转，只有 sin 分量的符号变了。

除了物体转世界矩阵，Unity 还提供了一个世界转物体矩阵，这两个矩阵互为逆矩阵。于是我们可以直接得到 $O^{-1}=R_{2}^{-1}S_{2}^{-1}R_{1}^{-1}S_{1}^{-1}$.

这个矩阵提供了我们所需的逆缩放矩阵，不过旋转也是逆的，顺序也是反的。不过幸运的是，我们可以转置这个矩阵，正好可以清除所有的副作用。于是可以得到 ${(O^{-1})}^{T}=N$.

> **矩阵转置又是啥？**  
> 矩阵 $M$ 的转置写作 $M^T$，意思是沿主对角线翻转它。它的行变成了列，列变成了行。注意主对角线上的元素不能变。  
> $\begin{bmatrix}1 & 2 & 3\\\\ 4 & 5 & 6\\\\ 7 & 8 & 9 \end{bmatrix}^{T} = \begin{bmatrix}1 & 4 & 7\\\\ 2 & 5 & 8\\\\ 3 & 6 & 9 \end{bmatrix}$  
> 和求逆一样，一串矩阵乘法的转置也是反转其顺序：$(AB)^{T} = B^{T}A^{T}$。这个式子对于非方形矩阵也是成立的，否则会得到一个无效的矩阵乘法。通常来说这是正确的，你可以找找证据。  
> 显然，对一个矩阵翻转两次就变回来了，也就是 $(M^T)^T=M$  
  
> **为什么转置矩阵就可以得到想要的结果？**  
> 首先，注意到 $R^{-1}=R^T$，如上文所证。  
> 所以得出： $O^{-1} = R_2^{-1} S_2^{-1} R_1^{-1} S_1^{-1} = R_2^T S_2^{-1} R_1^T S_1^{-1}$.  
> 然后求转置： $(O^{-1})^T = (S_1^{-1})^T (R_1^T)^T (S_2^{-1})^T (R_2^T)^T = (S_1^{-1})^T R_1 (S_2^{-1})^T R_2$.  
> 然后注意到 $S^T=S$，因为缩放矩阵除了对角线之外都是零。  
> 得到：$ (O^{-1})^T = S_1^{-1} R_1 S_2^{-1} R_2 = N$.  

所以我们把世界转物体矩阵转置，然后乘以顶点法线。

```c
				i.normal = mul(
					transpose((float3x3)unity_WorldToObject),
					v.normal
				);
				i.normal = normalize(i.normal);
```
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/correct-normals.png)  
*正确的世界空间法线*  

实际上，UnityCG 有一个好用的函数 UnityObjectToWorldNormal，做的工作就如我们上面所做。所以我们可以直接使用这个函数。它的内部使用的是显式的乘法而非[transpose](http://developer.download.nvidia.com/cg/transpose.html)，这样更有利于代码编译。  

> **UnityObjectToWorldNormal长什么样子？**  
> 这就是。inline 关键字不会执行任何操作，如果你想知道的话。
```c
// Transforms normal from object to world space
inline float3 UnityObjectToWorldNormal( in float3 norm ) {
	// Multiply by transposed inverse matrix,
	// actually using transpose() generates badly optimized code
	return normalize(
		unity_WorldToObject[0].xyz * norm.x +
		unity_WorldToObject[1].xyz * norm.y +
		unity_WorldToObject[2].xyz * norm.z
	);
}
```

## 1.4 重新归一化

在顶点着色器里传入了正确的法线之后，它们被传入插值器。不幸的是，单位向量之间的线性插值结果不会保持单位长度，它会变短。

所以我们在片段着色器里需要把法线重新归一化。

```c
			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				i.normal = normalize(i.normal);
				return float4(i.normal * 0.5 + 0.5, 1);
			}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/renormalized.png)  
*重归一化的法线*

虽然这样做结果更好，但是不这样做的结果误差也不是很大。你也可以出于性能考虑，不在片段着色器里重新归一化。在移动平台上这是一种普遍的优化手段。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/interpolation-error.png)  
*夸大了的误差*  

[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/normals.unitypackage)  

# 2 漫反射

一个非光源的物体能被我们看见，是因为它会反射光线。反射有很多方式，我们首先研究的是漫反射。

发生漫反射的原因是一束光线没有马上从物体表面反弹出来，而是进入了物体，在内部来回反弹并且几次被分裂，最后才离开物体表面。实际上，光子和原子之间的作用非常复杂，不过我们不需要对真实世界的物理了解得那么详细。

有多少光线会被表面漫反射取决于光撞击平面的角度。正对着表面，也就是光线和法线成 0° 入射时，反射光最多，随着角度增大，反射光减少。90° 时，没有光子撞击到物体表面，所以表面应该是暗的。漫反射光的数量和光线与法线的夹角余弦成正比。这称为兰伯特余弦法则（Lambert's cosine law）。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/lambert.png)  
*漫反射*  

要计算兰伯特反射系数，就要知道表面法线和光线方向的点积。我们已经有了法线，但是光线方向还不知道。我们先从一个固定方向开始，假设光从正上方射入。

```c
			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				i.normal = normalize(i.normal);
				return dot(float3(0, 1, 0), i.normal);
			}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/lit-from-above-gamma.png)![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/lit-from-above-linear.png)  
*正上方射入光线，伽马空间和线性空间*  

> **点积是啥？**  
> （略略略）  

## 2.1 限制光线范围

当表面面向光时，计算点积是没有问题的。但是当表面背向光时，逻辑上它应该在自己的阴影里，接受不到光照。此时光线与法线的夹角大于 90°，所以其余弦也就是点积会变成负数。我们不想要负数的光，就需要对结果做一点限制。这是可以使用标准的 [max](http://developer.download.nvidia.com/cg/max.html) 函数。

```c
				return max(0, dot(float3(0, 1, 0), i.normal));
```

除了 max 函数，常见的还有 [saturate](http://developer.download.nvidia.com/cg/saturate.html) 函数。这个标准函数的作用是把结果限制在 0 和 1 之间。

```c
				return saturate(dot(float3(0, 1, 0), i.normal));
```

这似乎是不必要的，因为我们知道我们的点积将永远不会大于 1。但是在某些情况下，saturate 实际上更高效，具体取决于硬件。不过我们现在不应该操心这种微优化。事实上，我们可以将它委托给 Unity 的家伙们。

UnityStandardBRDF 包含文件定义了方便的 DotClamped 函数。它会计算点积，并确保它永远不会为负数。这正是我们所需要的。这个文件还包含许多其他光照函数，还引用了其他有用的文件，稍后我们将需要它们。 所以我们就用它吧！

```c
			#include "UnityCG.cginc"
			#include "UnityStandardBRDF.cginc"

			…

			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				i.normal = normalize(i.normal);
				return DotClamped(float3(0, 1, 0), i.normal);
			}
```

> **DotClamped 长什么样子？**  
> 这就是。显然，他们觉得在低端硬件和 PS3 上 saturate 性能更好。
```c
inline half DotClamped (half3 a, half3 b) {
	#if (SHADER_TARGET < 30 || defined(SHADER_API_PS3))
		return saturate(dot(a, b));
	#else
		return max(0.0h, dot(a, b));
	#endif
}
```
> 这个着色器用的是半精度浮点数，不过目前不需要精度问题，因为只会在移动设备上有区别。

因为 UnityStandardBRDF 已经包含了 UnityCG 和其他一些文件，所以我们不必再明确地包含它了。 不删掉它也没有错，但我们想保持简短。

```c
//			#include "UnityCG.cginc"
			#include "UnityStandardBRDF.cginc"
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/include-files.png)  
*从 UnityStandardBRDF 开始的文件继承链*

## 2.2 光源

现在我们应该去掉硬编码的光源，采用场景设置的光源方向了。默认情况下，每一个 Unity 场景都会有一个代表太阳的光源。这个光源是方向光，也就是说它被认为是无限远的，所以它发出的所有光线方向都完全相同。当然在现实生活中并非如此，但太阳如此遥远，所以这样近似基本差不多。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/default-scene-light.png)  
*Default scene light, moved out of the way.*  

UnityShaderVariables 定义了一个 float4 \_WorldSpaceLightPos0 变量。它代表了当前光源的位置，或者如果是方向光的话代表光线方向。因为它是齐次坐标，所以有四个分量。所以，对于我们的方向光源，第四个分量为 0。

```c
				float3 lightDir = _WorldSpaceLightPos0.xyz;
				return DotClamped(lightDir, i.normal);
```

## 2.3 光照模式

如果想得到正确的结果，我们需要告诉 Unity 我们需要什么光照数据。方法是给着色器的 pass 加一个 LightMode 标签。

使用哪种光照模式取决于我们如何渲染场景。我们可以使用的有前向渲染或者延迟渲染路径。另外还有两种更古老的渲染模式，但是我们不关心。你可以在 player rendering 设置里找到他，就在颜色空间选项的上边，我们使用前向渲染（Forward）模式，就是默认的那种。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/rendering-path.png)  
*渲染路径选项*

我们的 pass 使用 ForwardBase 模式，这种 pass 是前向渲染某个物体时第一个使用的 pass。这样我们才可以获取到场景的主光源，还有一些其他设置我们后边再说。

```c
		Pass {
			Tags {
				"LightMode" = "ForwardBase"
			}

			CGPROGRAM

			…

			ENDCG
		}
```

![](https://thumbs.gfycat.com/CleverSpiffyBernesemountaindog-small.gif)  
*漫反射*

## 2.4 光线颜色

显然光线不总是白色，每一个光源都有自己的颜色。我们可以通过 fixed4 \_LightColor0 变量获取。它的定义在 UnityLightingCommon 文件里。

> **fixed4 是什么？**  
> 它是一种低精度数字，在移动设备上以精度换取速度。在桌面平台，fixed 等同于 float。精度优化是后面的主题。

此变量包含了灯光的颜色乘以强度。虽然它有四个分量，但我们只需要 RGB。

```c
			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				i.normal = normalize(i.normal);
				float3 lightDir = _WorldSpaceLightPos0.xyz;
				float3 lightColor = _LightColor0.rgb;
				float3 diffuse = lightColor * DotClamped(lightDir, i.normal);
				return float4(diffuse, 1);
			}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/light-color.png)  
*有颜色的光*

## 2.5 反照率

大多数材质都会吸收一部分电磁波谱。这样它们就有了颜色。例如，如果所有可见的红色频率都被吸收掉，对应的物体将显示为青色。

> **没逃掉的光线到哪去了呢？**  
> 这部分光的能量通常转化为热能存储在物体中。这就是黑色东西一般比白色东西更温暖的原因。

材质的漫反射的颜色称为反照率 Albedo。Albedo 在拉丁语的意思是白色程度。它描述了有多少红色，绿色和蓝色通道被漫反射，其余的都被吸收。我们可以使用材质的纹理和色调来设置它。

```c
			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				i.normal = normalize(i.normal);
				float3 lightDir = _WorldSpaceLightPos0.xyz;
				float3 lightColor = _LightColor0.rgb;
				float3 albedo = tex2D(_MainTex, i.uv).rgb * _Tint.rgb;
				float3 diffuse =
					albedo * lightColor * DotClamped(lightDir, i.normal);
				return float4(diffuse, 1);
			}
```

让我们也把主纹理在 Inspector 的标签改成 Albedo。

```c
	Properties {
		_Tint ("Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Albedo", 2D) = "white" {}
	}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/albedo-inspector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/albedo-gamma.png)![](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/albedo-linear.png)  
*带有反照率的漫反射，在伽马和线性空间下*

[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-4/diffuse-shading/diffuse-shading.unitypackage)  

# 3 高光着色

除了漫反射，还有一种高光反射。它发生的条件是光在撞击到平面后没有被漫反射，而是直接从表面反弹，反弹的角度等于撞击平面的角度。这种效应导致了你在镜子里看到的反射。

不同于漫反射，在高光反射中观察者的位置很重要。只有反射角度直接朝向你的光才能被看到，其余的都散失到其他地方了，所以没有被你看到。

所以我们需要知道从表面到观察者的方向，所以要知道表面和摄像机的世界坐标。

我们可以在顶点着色器里，通过物体转世界矩阵计算表面的世界坐标，然后传给片段着色器。

```c
			struct Interpolators {
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
				float3 normal : TEXCOORD1;
				float3 worldPos : TEXCOORD2;
			};

			Interpolators MyVertexProgram (VertexData v) {
				Interpolators i;
				i.position = mul(UNITY_MATRIX_MVP, v.position);
				i.worldPos = mul(unity_ObjectToWorld, v.position);
				i.normal = UnityObjectToWorldNormal(v.normal);
				i.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return i;
			}
```

摄像机的位置可以通过 float3 _WorldSpaceCameraPos 获取，它定义在 UnityShaderVariables 里。我们用它减去表面的位置，然后归一化，得到观察方向。

```c
			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				i.normal = normalize(i.normal);
				float3 lightDir = _WorldSpaceLightPos0.xyz;
				float3 viewDir = normalize(_WorldSpaceCameraPos - i.worldPos);
				
				float3 lightColor = _LightColor0.rgb;
				float3 albedo = tex2D(_MainTex, i.uv).rgb * _Tint.rgb;
				float3 diffuse =
					albedo * lightColor * DotClamped(lightDir, i.normal);
				return float4(diffuse, 1);
			}
```

> **Unity 的着色器不会对观察方向插值么？**  
> 会插值，Unity 的着色器在顶点着色器里计算观察方向然后对其插值。在片段着色器里归一化，或者为低端硬件考虑在顶点着色器里归一化。哪个都可以。

## 3.1 反射光

为了知道反射光的角度，我们可以使用 [reflect](http://developer.download.nvidia.com/cg/reflect.html) 函数。它需要的参数是光的入射方向和法线方向，以求出反射方向。所以我们需要把光线方向翻转。

```c
				float3 reflectionDir = reflect(-lightDir, i.normal);

				return float4(reflectionDir * 0.5 + 0.5, 1);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/reflection-directions.png)  
*反射方向*

> **反射向量是如何求的？**  
> 给定入射方向 $D$，法线 $N$，反射方向是 $D-2N(N \cdot D)$.  

对于完美的光滑镜面，我们只能看到角度恰好的一支反射光，其余部分的高光反射由于角度不对，看起来都是黑的。不过物体不会是绝对光滑的，都有微小的凹凸，导致表面法线实际上会有差异。

所以我们可以看到一部分高光反射，即使视角不是那么恰好匹配反射角。我们离反射角越远，看到的就越少。我们可以再一次用限制点积来指出我们能看到多少光。

```c
				return DotClamped(viewDir, reflectionDir);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/reflections.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/phong.png)  
*高光反射*

## 3.2 光滑度

基于上述效应产生的高光面积大小取决于材质的粗糙度。光滑材料能更好地聚集光线，导致高光面积更小。我们可以把光滑度设置为材质的一个属性。这个值通常会设定为 0 到 1 之间，所以我们为它做个滑杆。

```c
	Properties {
		_Tint ("Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Texture", 2D) = "white" {}
		_Smoothness ("Smoothness", Range(0, 1)) = 0.5
	}
		
		…

			float _Smoothness;
```

点积的幂次越高，高光就越小。我们用光滑度来控制幂次，不过想达到理想的效果，1 是远远不够的，我们把它乘以 100.

```c
				return pow(
					DotClamped(viewDir, reflectionDir),
					_Smoothness * 100
				);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/smoothness-slider.png)
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/smoothness.png)  
相当光滑  

## 3.3 Blinn-Phong

我们目前计算反射光的依据是 Blinn 反射模型。不过其实最常用的模型叫 Blinn-Phong。它使用了光线方向和观察方向的中间值，称为半向量。高光的贡献度是由法线和半向量的夹角决定。

```c
//				float3 reflectionDir = reflect(-lightDir, i.normal);
				float3 halfVector = normalize(lightDir + viewDir);

				return pow(
					DotClamped(halfVector, i.normal),
					_Smoothness * 100
				);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/halfway-vector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/blinn-phong.png)  
*Blinn-Phong 高光*

这种算法的高光面积更大，但是可以通过提高光滑度来抵消。虽然两种算法都是近似，但 Blinn-Phong 的结果在视觉上比 Phong 更接近现实。不过它有一个很大的局限性，就是它可能会为从背面照亮的物体产生错误的高光。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/specular-error.png)  
*错误的高光，光滑度为 0.01*  

当光滑度比较低的时候，这种瑕疵就比较明显。想要掩饰这种瑕疵可以通过阴影，或者基于光线角度对高光进行衰减。Unity 的旧版 Shader 也有这个问题，所以我们一点不需要担心。而且我们很快就会转到另外一种光照方法。

## 3.4 高光颜色

显然高光反射的颜色应该和光线的颜色一致，所以让我们把它考虑进来。

```c
				float3 halfVector = normalize(lightDir + viewDir);
				float3 specular = lightColor * pow(
					DotClamped(halfVector, i.normal),
					_Smoothness * 100
				);

				return float4(specular, 1);
```

不过这还没完，反射的颜色还取决于材质。这一点上和反照率不同的是，金属往往反照率非常微弱，但是具有强烈且通常带有颜色的高光反射率。相比之下，非金属倾向于具有明显的反照率，而高光反射率较弱而不带有颜色。

我们可以添加纹理和色调以定义高光颜色，如同反照率一样。不过让我们先不用纹理，只用一个色调来解决。

```c
	Properties {
		_Tint ("Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Albedo", 2D) = "white" {}
		_SpecularTint ("Specular", Color) = (0.5, 0.5, 0.5)
		_Smoothness ("Smoothness", Range(0, 1)) = 0.1
	}

	…

			float4 _SpecularTint;
			float _Smoothness;

			…

			float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
				…

				float3 halfVector = normalize(lightDir + viewDir);
				float3 specular = _SpecularTint.rgb * lightColor * pow(
					DotClamped(halfVector, i.normal),
					_Smoothness * 100
				);

				return float4(specular, 1);
			}
```

我们可以用一个颜色属性同时控制高光反射的颜色和强度。

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/specular-color.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/colored-specular.png)  
*调了颜色的高光反射*
  
> **可以用色调的 a 通道存光滑度嘛？**  
> 当然可以，在使用纹理存放高光颜色和光滑度的时候也可以这么干。  
  
## 3.5 漫反射和高光

漫反射和高光就像光照拼图的两块。我们可以将它们加到一起，使我们的图像更加完整。

```c
				return float4(diffuse + specular, 1);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/diffuse-specular-gamma.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/diffuse-specular-linear.png)  
*漫反射加高光，在伽马和线性空间下*  

[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-4/specular-shading/specular-shading.unitypackage)  

# 4 能量守恒

---
  
下一课是[多光源](https://catlikecoding.com/unity/tutorials/rendering/part-5/)。  
[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-4/physically-based-shading/physically-based-shading.unitypackage)