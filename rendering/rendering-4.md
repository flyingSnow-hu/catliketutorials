渲染之四 第一盏灯
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

## 1.1 世界空间下的法线

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

![从物体空间到世界空间](https://catlikecoding.com/unity/tutorials/rendering/part-4/normals/object-to-world.png)  
**  

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
> $\begin{bmatrix} 1&0&0&x\\ 0&1&0&y\\ 0&0&1&z\\ 0&0&0&1 \end{bmatrix}^{-1} = \begin{bmatrix}1&0&0&-x\\ 0&1&0&-y\\ 0&0&1&-z\\ 0&0&0&1 \end{bmatrix}$  
>   
> 缩放矩阵的逆是对其对角线元素求倒数，只需考虑其 3×3 部分。
> $\begin{bmatrix} x&0&0\\ 0&y&0\\ 0&0&z\\ \end{bmatrix}^{-1} = \begin{bmatrix} \frac{1}{x}&0&0\\ 0&\frac{1}{y}&0\\ 0&0&\frac{1}{z}\\ \end{bmatrix}$  
>   
> 旋转矩阵可以按轴向分解考虑，比如绕着 Z 轴旋转。旋转 
$z$ 弧度的反操作是旋转 $-z$ 弧度。如果你了解正弦波和余弦波，就应该知道 $sin(-z) = -sin(z)$，$cos(-z) = cos(z)$，所以问题就简单了。  
> $\begin{bmatrix} cos(z)&-sin(z)&0\\ sin(z)&cos(z)&0\\ 0&0&1\\ \end{bmatrix}^{-1} = \begin{bmatrix} cos(z)&sin(z)&0\\ -sin(z)&cos(z)&0\\ 0&0&1\\ \end{bmatrix}$  
>   
> 注意旋转矩阵的逆等同于沿对角线翻转，只有 sin 分量的符号变了。

除了物体转世界矩阵，Unity 还提供了一个世界转物体矩阵，这两个矩阵互为逆矩阵。于是我们可以直接得到 $O=R_{2}^{-1}S_{2}^{-1}R_{1}^{-1}S_{1}^{-1}$.

这个矩阵提供了我们所需的逆缩放矩阵，不过旋转也是逆的，顺序也是反的。不过幸运的是，我们可以转置这个矩阵，正好可以清除所有的副作用。于是可以得到 ${(O^{-1})}^{T}=N$.

> **矩阵转置又是啥？**  
> 矩阵 $M$ 的转置写作 $M^T$，意思是沿主对角线翻转它。它的行变成了列，列变成了行。注意主对角线上的元素不能变。  
> $\begin{bmatrix}1 & 2 & 3\\ 4 & 5 & 6\\ 7 & 8 & 9 \end{bmatrix}^{T} = \begin{bmatrix}1 & 4 & 7\\ 2 & 5 & 8\\ 3 & 6 & 9 \end{bmatrix}$  
> 和求逆一样，一串矩阵乘法的转置也是反转其顺序：$(AB)^{T} = B^{T}A^{T}$。这个式子对于非方形矩阵也是成立的，否则会得到一个无效的矩阵乘法。通常来说这是正确的，你可以找找证据。  
> 显然，对一个矩阵翻转两次就变回来了，也就是 $(M^T)^T=M$  
  
> **为什么转置矩阵就可以得到想要的结果？**  
> 首先，注意到 $R^{-1}=R^T$，如上文所证。  
> 所以得出： $O^-1 = R_2^-1 S_2^-1 R_1^-1 S_1^-1 = R_2^T S_2^-1 R_1^T S_1^-1$.  
> 然后求转置： $(O^-1)^T = (S_1^-1)^T (R_1^T)^T (S_2^1)^T (R_2^T)^T = (S_1^-1)^T R_1 (S_2^-1)^T R_2$.  
> 然后注意到 $S^T=S$，因为缩放矩阵除了对角线之外都是零。  
> 得到：$ (O^-1)^T = S_1^-1 R_1 S_2^-1 R_2 = N$.  

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

 

---
  
下一课是[多盏灯](https://catlikecoding.com/unity/tutorials/rendering/part-5/)。  
[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-4/physically-based-shading/physically-based-shading.unitypackage)