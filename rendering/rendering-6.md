
渲染之六 凹凸
======
[原教程传送门](https://catlikecoding.com/unity/tutorials/rendering/part-6/)
---

 * 变换法线模拟凹凸
 * 从高度场计算法线
 * 采样和混合法线贴图
 * 从切线空间转换为世界空间

这一篇是渲染教程的第六部分，上一部分里我们增加了更复杂的光照，这次，我们将制造错觉以模拟更复杂的曲面

这一节教程是基于 5.4.0f3

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tutorial-image.jpg)  
*看起来不再像光滑的球体了*

# 1 凹凸贴图

我们可以使用反照率纹理来创建具有复杂颜色的材质。我们可以使用法线来调整表面的曲率。使用这些工具，我们可以制造出各种表面。但是，单个三角形的表面始终是平滑的。它只能在三个法向量之间进行插值，因此不能代表粗糙或变化的表面。当去掉反照率纹理仅使用纯色时，就变得很明显。

举一个平坦度的好例子。把一个简单的四边形（Quad）添加到场景，绕X轴旋转90°，使其面向上。把我们的光照材质赋给它，没有纹理，色调（tint）为纯白。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/flat.png)  
*纯平的四边形*

由于默认的天空盒非常亮，导致其他光源的作用不明显，所以这节教程里我们把它关掉，方法是在 Lighting 面板里把 Ambient Intensity 设为零。然后只启用主方向灯。在场景视图中找一个好视角，以便在四边形上能看到一些光线变化。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/ambient-intensity.png)  
 ![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/no-ambient.png)  
 *没有环境光 ，只有主方向光*

怎么能让这个四边形看起来不那么平呢？我们可以假造粗糙度，把它画到反照率图上。但是这样得来的阴影是静态的，如果光线发生变化，或者物体移动了，阴影也要跟着变，否则就破坏了光影关系。如果考虑到高光反射，甚至连摄像机都不能动。

我们可以修改法线以制造曲面的感觉，但是每个四边形只有四条法线，每个顶点一个，这样只能造出平滑的过渡。如果我们想使表面变丰富变粗糙，需要更多的法线。

我们也可以把四边形分割成更小的四边形，这样就有了更多的法线。不过实际上，如果我们有很多顶点，可以直接移动顶点，我们就不需要伪造粗糙感了，直接制造一个真粗糙表面就可以了。然而即使这样，每个小四边形仍然有同样的问题，我们继续分割吗？这样下去会制造出巨量的网格和更巨量的三角形。这个思路用来建模可以，但是对实时渲染的游戏不适合。

## 1.1 高度贴图

和平坦的表面相比，粗糙表面的特征是高度不统一。如果我们把这些高度数据存进纹理，就可以逐像素产生法向量，而非逐顶点。这个思路被称为凹凸贴图，最初是由 James Blinn 提出的。

这是一张高度贴图，配合我们的大理石贴图使用。它是一张 RGB 格式的纹理，每个通道的值完全一致。把这张图导入你的工程，使用默认的导入设置即可。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/marble-heights.png)  
*大理石的高度图*

为 My First Lighting 着色器添加一个 _HeightMap 属性。由于它和反照率贴图共用 UV，所以不需要单独设置其缩放和偏移。默认值是什么不太重要，只要是均匀的就行，灰色图就可以。

```c
    Properties {
        _Tint ("Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Albedo", 2D) = "white" {}
        [NoScaleOffset] _HeightMap ("Heights", 2D) = "gray" {}
        [Gamma] _Metallic ("Metallic", Range(0, 1)) = 0
        _Smoothness ("Smoothness", Range(0, 1)) = 0.1
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/heights-inspector.png)  
*带有高度图的材质*

把对应的变量加到 My Lighting 库文件中，这样我们就可以访问纹理了。我们先把他乘以反照率，看看是什么样子。

```c
float4 _Tint;
sampler2D _MainTex;
float4 _MainTex_ST;

sampler2D _HeightMap;

…

float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
    i.normal = normalize(i.normal);

    float3 viewDir = normalize(_WorldSpaceCameraPos - i.worldPos);

    float3 albedo = tex2D(_MainTex, i.uv).rgb * _Tint.rgb;
    albedo *= tex2D(_HeightMap, i.uv);

    …
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/height-as-color.png)  
*把高度图作为颜色*  

## 1.2 调整法线

由于我们的逐像素法线会变得非常复杂，把初始化代码移到一个独立的函数里。同时删掉高度图的测试代码。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    i.normal = normalize(i.normal);
}

float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
    InitializeFragmentNormal(i);

    float3 viewDir = normalize(_WorldSpaceCameraPos - i.worldPos);

    float3 albedo = tex2D(_MainTex, i.uv).rgb * _Tint.rgb;
//  albedo *= tex2D(_HeightMap, i.uv);

    …
}
```

因为我们现在的四边形是平行于 XZ 平面，所以其法向量恒定为(0,1,0)。我们可以暂时忽略顶点数据，用一个常量法线代替，后边再处理不同表面朝向的问题。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    i.normal = float3(0, 1, 0);
    i.normal = normalize(i.normal);
}
```

我们怎么在这应用高度数据呢？一个简单的办法是把高度用作法线的 Y 分量，再归一化。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    float h = tex2D(_HeightMap, i.uv);
    i.normal = float3(0, h, 0);
    i.normal = normalize(i.normal);
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/height-as-normal.png)  
*以高度为法线*  

没起啥作用，因为归一化把每一点的法向量又拉回了(0,1,0)。黑线代表此处高度为零，导致归一化失败，我们需要一些其他方法。

## 1.3 有限差分

我们现在处理的是二维的纹理数据，有 U 和 V 两个维度，高度可以认为是方向向上的第三维。所以我们也可以换一个说法，纹理代表了一个函数 $f(u,v) = h$。一开始我们只考虑 U 维度，函数变成 $f(u) = h$ 。我们能从中解出法线吗？

如果我们知道函数的斜率，那么可以用它来计算任何一点的法线。 斜率由 $h$ 的变化率定义，也就是其导数 $h'$。由于 $h$ 是某一个函数的值，所以 $h'$ 也是某一个函数的值。所以我们有导数函数 $f'(u) = h'$。

不幸的是，我们不知道这些函数原型是什么，不过我们可以近似求解。我们可以比较纹理上的两个点的高度。例如，极端一点的例子，比较一下 U = 0 和 U = 1。这两个采样点之间的差值就是这一段的变化率，用函数表示，就是 $f(1) - f(0)$. 我们以此为基础构建切向量，$\begin{bmatrix}1 \\ f(1) - f(0) \\ 0 \end{bmatrix}$

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/tangent-diagram.png)  
*从 $\begin{bmatrix}0 \\ f(0) \end{bmatrix}$ 到 $\begin{bmatrix}1 \\ f(1) \end{bmatrix}$ 的切向量*

这当然是真实切线向量的粗略近似。它将整个纹理视为线性斜率。为了做得更好，我们可以对两个更近的点采样。例如，U 坐标为 0 和 ½。这两个点之间的变化率是 $f(\frac{1}{2}) - f(0)$ 每半单位 U<font color=red><sup>这句话的结构类似于“工资增长的幅度是2元每半年”</sup></font>。因为整单位的变化率比较好处理，所以我们把变化率除以点的距离，所以得到 $\frac{f(\frac{1}{2}) - f(0)}{\frac{1}{2}} = 2(f(\frac{1}{2}) - f(0))$. 由此得到切向量 $\begin{bmatrix}1 \\ 2(f(\frac{1}{2}) - f(0)) \\ 0 \end{bmatrix}$.

通常来说，我们必须相对于每个渲染的片段的 U 坐标执行此运算。设到下一个点的距离为常数 $\delta$，则导函数可以近似为 $f'(u) \approx \frac{f(u+\delta) - f(u)}{\delta}$.

δ 越小，我们的近似值就越接近真实导数值。当然它不能变成零，但是当 δ 趋近于无穷小时，可以得到$f'(u)=\lim_{\delta\to0}\frac{f(u+\delta)-f(u)}{\delta}$. 这个对导数求近似值的方法被称为有限差分。由这个方法，我们可以得到任一点的切向量$\begin{bmatrix}1 \\ f'(u) \\ 0 \end{bmatrix}$.

## 1.4 




  
下一课是[影子](https://catlikecoding.com/unity/tutorials/rendering/part-7/)。  
[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/tangents.unitypackage)



