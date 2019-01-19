
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

不幸的是，我们不知道这些函数原型是什么，不过我们可以近似求解。我们可以比较纹理上的两个点的高度。例如，极端一点的例子，比较一下 U = 0 和 U = 1。这两个采样点之间的差值就是这一段的变化率，用函数表示，就是 $f(1) - f(0)$. 我们以此为基础构建切向量，$\begin{bmatrix}1 \\\\ f(1) - f(0) \\\\ 0 \end{bmatrix}$

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/tangent-diagram.png)  
*从 $\begin{bmatrix}0 \\\\ f(0) \end{bmatrix}$ 到 $\begin{bmatrix}1 \\\\ f(1) \end{bmatrix}$ 的切向量*

这当然是真实切线向量的粗略近似。它将整个纹理视为线性斜率。为了做得更好，我们可以对两个更近的点采样。例如，U 坐标为 0 和 ½。这两个点之间的变化率是 $f(\frac{1}{2}) - f(0)$ 每半单位 U <font color="red"><sub>这句话的结构类似于“工资增长的幅度是2元每半年”</sub></font>。因为整单位的变化率比较好处理，所以我们把变化率除以点的距离，所以得到 $\frac{f(\frac{1}{2}) - f(0)}{\frac{1}{2}} = 2(f(\frac{1}{2}) - f(0))$. 由此得到切向量 $\begin{bmatrix}1 \\\\ 2(f(\frac{1}{2}) - f(0)) \\\\ 0 \end{bmatrix}$.

通常来说，我们必须相对于每个渲染的片段的 U 坐标执行此运算。设到下一个点的距离为常数 $\delta$，则导函数可以近似为 $f'(u) \approx \frac{f(u+\delta) - f(u)}{\delta}$.

δ 越小，我们的近似值就越接近真实导数值。当然它不能变成零，但是当 δ 趋近于无穷小时，可以得到$f'(u)=\lim_{\delta\to0}\frac{f(u+\delta)-f(u)}{\delta}$. 这个对导数求近似值的方法被称为有限差分。由这个方法，我们可以得到任一点的切向量$\begin{bmatrix}1 \\\\ f'(u) \\\\ 0 \end{bmatrix}$.

## 1.4 从切线到法线

在我们的着色器里可用的 δ 值是什么样的呢？最小的有意义的值就是纹理的一个像素的距离。在着色器里，可以通过一个带有 _TexelSize 后缀的 float4 变量获取这个值。这个变量由 Unity 自动设置，如同 _ST 变量一样。

```c
sampler2D _HeightMap;
float4 _HeightMap_TexelSize;
```

> **_TexelSize 里存的是什么？**  
> 它的前两个分量存放的是纹素尺寸，也就是 U 和 V 的分母 <font color="red"><sub>也就是像素数量的倒数</sub></font>。后两个分量是像素数量。例如一个 256×128 的纹理，对应的分量是 (0.00390625, 0.0078125, 256, 128)。

我们现在可以对这个纹理采样两次，计算高度导数，然后构建切向量，我们直接把他作为法向量使用。
```c
    float2 delta = float2(_HeightMap_TexelSize.x, 0);
    float h1 = tex2D(_HeightMap, i.uv);
    float h2 = tex2D(_HeightMap, i.uv + delta);
    i.normal = float3(1, (h2 - h1) / delta.x, 0);

    i.normal = normalize(i.normal);
```

实际上由于最后我们还是要归一化，所以可以把切向量缩放 δ 倍，这样省掉一次除法而且还提高了精度。

```c
    i.normal = float3(delta.x, h2 - h1, 0);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/tangent.png)  
*把切向量作为法向量*  

我们得到了一个非常锐利的结果。那是因为高度差达到了一个单位的范围，从而产生非常陡峭的斜坡。由于扰动法线实际上并没有真正改变表面，所以我们不希望出现如此巨大的差异，可以把高度加一个任意的缩放。我们这里将高度范围缩小到一个纹素，也就是将高度差乘以δ，或者简单地将切线中的 δ 替换为 1。

```c
    i.normal = float3(1, h2 - h1, 0);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/scaled-height.png)  
*缩放之后的高度*

看起来开始变好了，但光照是错误的，太暗了。那是因为我们直接使用切线作为法线，要将其转换为朝向上的法向量，我们必须围绕Z轴把切线旋转 90°。

```c
    i.normal = float3(h1 - h2, 1, 0);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/rotated.png)  
*使用真正的法线*  

> 向量旋转的原理是？  
> 要把一个 2D 向量逆时针旋转 90°，需要把 XY 分量交换位置，然后把新的 X 分量取相反数。最后得到 $\begin{bmatrix}-f'(u) \\\\ 1 \\\\ 0 \end{bmatrix}$.  
> ![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/vector-rotation.png)  
> *2D 向量旋转 90°*  

## 1.5 中心差分

我们使用有限差分来近似创建法向量。具体地，通过使用前向差分方法。我们取一个点，然后向一个方向看以确定斜率。最后法线会偏向该方向。为了更好地逼近法线，我们可以在两个方向上移动采样点。这使线性近似在当前点上居中，称为中心差分法。这样的导数函数变为 $f'(u)=\lim_{\delta\to0}\frac{f(u+\frac{\delta}{2})-f(u-\frac{\delta}{2})}{\delta}$.

```c
    float2 delta = float2(_HeightMap_TexelSize.x * 0.5, 0);
    float h1 = tex2D(_HeightMap, i.uv - delta);
    float h2 = tex2D(_HeightMap, i.uv + delta);
    i.normal = float3(h1 - h2, 1, 0);
```

这种方法轻微地移动了凹凸，使其和高度场更好地对齐。除此之外并没有修改其形状。

## 1.6 两个维度都加上

目前为止，我们创建的法线仅考虑了 U 的变化。我们一直在使用函数 $f(u,v)$ 对 $u$ 的偏导数，也就是 $f_{u}^{'}(u,v)$，简写为 $f_{u}^{'}$. 我们也可以沿 V 方向创建法线，也就是 $f_{v}^{'}$. 这样的话，切向量就是 $\begin{bmatrix}0 \\\\ f_{v}^{'} \\\\ 1 \end{bmatrix}$. 法向量是 $\begin{bmatrix}0 \\\\ 1 \\\\ -f_{v}^{'} \end{bmatrix}$.
```c
    float2 du = float2(_HeightMap_TexelSize.x * 0.5, 0);
    float u1 = tex2D(_HeightMap, i.uv - du);
    float u2 = tex2D(_HeightMap, i.uv + du);
    i.normal = float3(u1 - u2, 1, 0);

    float2 dv = float2(0, _HeightMap_TexelSize.y * 0.5);
    float v1 = tex2D(_HeightMap, i.uv - dv);
    float v2 = tex2D(_HeightMap, i.uv + dv);
    i.normal = float3(0, 1, v1 - v2);

    i.normal = normalize(i.normal);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/other-dimension.png)  
*V 方向的法线*

我们现在可以获取 U 和 V 切线了。这些向量一起描述了片段上高度场的表面。通过计算它们的叉积，我们可以得到 2D 高度场的法向量。

```c
    float2 du = float2(_HeightMap_TexelSize.x * 0.5, 0);
    float u1 = tex2D(_HeightMap, i.uv - du);
    float u2 = tex2D(_HeightMap, i.uv + du);
    float3 tu = float3(1, u2 - u1, 0);

    float2 dv = float2(0, _HeightMap_TexelSize.y * 0.5);
    float v1 = tex2D(_HeightMap, i.uv - dv);
    float v2 = tex2D(_HeightMap, i.uv + dv);
    float3 tv = float3(0, v2 - v1, 1);

    i.normal = cross(tv, tu);
    i.normal = normalize(i.normal);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/normals.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/normals-details.png)  
*完整的法线*  

> **叉积是个啥？**   
> (略略略)

计算切向量的叉积，可以得到 $\begin{bmatrix}0 \\\\ f_{v}^{'} \\\\ 1 \end{bmatrix} × \begin{bmatrix}0 \\\\ f_{u}^{'} \\\\ 1 \end{bmatrix} = \begin{bmatrix}-f_{u}^{'} \\\\ 1 \\\\ -f_{v}^{'} \end{bmatrix}$. 所以我们可以直接构造法向量，而不需要依赖 cross 函数。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    float2 du = float2(_HeightMap_TexelSize.x * 0.5, 0);
    float u1 = tex2D(_HeightMap, i.uv - du);
    float u2 = tex2D(_HeightMap, i.uv + du);
//  float3 tu = float3(1, u2 - u1, 0);

    float2 dv = float2(0, _HeightMap_TexelSize.y * 0.5);
    float v1 = tex2D(_HeightMap, i.uv - dv);
    float v2 = tex2D(_HeightMap, i.uv + dv);
//  float3 tv = float3(0, v2 - v1, 1);

//  i.normal = cross(tv, tu);
    i.normal = float3(u1 - u2, 1, v1 - v2);
    i.normal = normalize(i.normal);
}
```

[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-mapping/bump-mapping.unitypackage)

# 2 法线贴图

凹凸贴图运行时，我们必须执行多次纹理采样和有限差分计算。这似乎是一种浪费，因为法线应该是始终相同的。 为什么要每一帧做重复工作？我们可以只做一次并将法线存储在纹理中。

> 那么纹理过滤还有效么？
> 双线性和三线性过滤将在法线向量之间进行混合，就像法线在三角形之间插值一样。所以我们必须把采样的法线重新归一化。
>  
> 您还必须确保每个 mipmap 包含法线是有效的。不能简单地像颜色数据一样对纹理进行向下采样。得到的向量也必须归一化。这一点 Unity 会保证。

这意味着我们需要一个法线贴图。我可以提供一个，但也可以让 Unity 为我们做一个。将高度贴图的 纹理类型更改为*Normal Map*，Unity会自动切换纹理使用三线性过滤，并假设我们要使用灰度图像数据来生成法线贴图。这正是我们想要的，但请将 Bumpiness 更改为更低的值，如0.05。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/inspector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/preview.png)  
*从高度图生成法线*  

应用导入设置后，Unity 将计算出法线贴图。原始的高度贴图仍然存在，但 Unity 内部会使用生成的贴图。

如同我们将法线可视化为颜色时一样，法线的分量必须调整到 0-1 范围内。所以它们实际存放的是 $\frac{N+1}{2}$。也就是说，比较平的地方会显示为绿色，然而法线贴图看起来是蓝色的，这是因为法线贴图常见的一个约定是将向上方向存储在 Z 分量中。从 Unity 的角度来看，Y 和 Z 坐标互相交换了。

## 2.1 采样法线贴图

法线贴图和高度贴图非常不同，所以我们先正一下名字。

```c
    Properties {
        _Tint ("Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Albedo", 2D) = "white" {}
        [NoScaleOffset] _NormalMap ("Normals", 2D) = "bump" {}
//      [NoScaleOffset] _HeightMap ("Heights", 2D) = "gray" {}
        [Gamma] _Metallic ("Metallic", Range(0, 1)) = 0
        _Smoothness ("Smoothness", Range(0, 1)) = 0.1
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/material-with-normal-map.png)  
*改成法线贴图*  

我们可以删掉所有有关高度贴图的代码，然后换成一次采样加一次归一化。

```c
sampler2D _NormalMap;

//sampler2D _HeightMap;
//float4 _HeightMap_TexelSize;

…

void InitializeFragmentNormal(inout Interpolators i) {
    i.normal = tex2D(_NormalMap, i.uv).rgb;
    i.normal = normalize(i.normal);
}
```

当然，我们还要把法线转回原始的 [-1, 1] 区间: 计算 $2N-1$.

```c
    i.normal = tex2D(_NormalMap, i.uv).xyz * 2 - 1;
```

别忘了交换 Y 和 Z

```c
    i.normal = tex2D(_NormalMap, i.uv).xyz * 2 - 1;
    i.normal = i.normal.xzy;
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/normals.png)  
*使用法线贴图*  

## 2.2 DXT5nm

我们的法线肯定有问题。那是因为 Unity 最终编码法线的方式和我们预想的不一样。虽然纹理预览显示的是 RGB 编码，但 Unity 实际上使用的是 DXT5nm。

DXT5nm 格式仅存储法线的 X 和 Y 分量。 其 Z 分量被丢弃。 Y 分量如预想的一样存储在G通道中。然而 X 分量存储在 A 通道中。R 和 B 通道不使用。

> **为什么要这样存储 X 和 Y 分量呢？**
> 
> 四通道纹理仅使用两个通道似乎很浪费。如果使用的是未压缩的纹理，确实如此。然而 DXT5nm 格式的设计意图是与 DXT5 纹理压缩一起使用。Unity默认执行此操作。
> 
> DXT5 通过对 4×4 像素的块进行分组，并用两种颜色和查找表来近似压缩像素。用于颜色的位数因通道而异。R 和 B 各 5 位，G 得 6 位，而 A 得 8 位。这就是把 X 坐标存放到 A 通道的原因之一。另一个原因是 RGB 通道共用一个查找表，A 单独占用一个查找表。这样一套设计可以把 X 和 Y 分量隔离开。
> 
> 压缩是有损的，但对于法线贴图其程度是可接受的。与未压缩的 8 位 RGB 纹理相比，可以获得 3：1 的压缩比。
> 
> 无论您是否实际压缩它们，Unity 都会使用 DXT5nm 格式编码所有法线​​贴图。但是，在移动平台上并非如此，因为它们不支持 DXT5。这时 Unity 将使用常规 RGB 编码。

所以由于 DXT5nm 格式，我们只能获得法线的前两个分量。

```c
    i.normal.xy = tex2D(_NormalMap, i.uv).wy * 2 - 1;
```

第三个分量需要根据前两个分量推断出来。由于法线都是单位向量，$||N||=||N||^{2}=N_{x}^{2}+N_{y}^{2}+N_{z}^{2}=1$，所以$N_{z}=\sqrt{1-N_{x}^{2}-N_{y}^{2}}$.

```c
    i.normal.xy = tex2D(_NormalMap, i.uv).wy * 2 - 1;
    i.normal.z = sqrt(1 - dot(i.normal.xy, i.normal.xy));
    i.normal = i.normal.xzy;
```

理论上说，其结果应该等于原始 Z 分量。但是由于纹理的精度有限，再加上纹理过滤，结果通常会有所不同。但也足够接近了。

此外，由于精度限制，$N_{x}^{2}+N_{y}^{2}$可能超出界限。 通过限制点积来确保这种情况不会发生。

```c
    i.normal.z = sqrt(1 - saturate(dot(i.normal.xy, i.normal.xy)));
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/dxt5nm.png)  
*解码 DXT5nm 法线*

## 2.3 缩放凹凸

既然我们已经将法线烘焙到纹理中，所以我们就不能在片段着色器中缩放它们了——真的是这样吗？

我们可以在计算 Z 之前缩放法线的X和Y分量。如果我们减小 X 和 Y，那么 Z 将变大，从而产生更平坦的表面。 如果增加 X 和 Y 则相反。 所以可以通过这种方式调整凹凸。 由于我们已经对 X 和 Y 的平方加了限制，所以永远不会得到无效的法线。

让我们为着色器添加一个凹凸缩放属性，如同 Unity 的标准着色器一样。

```c
    Properties {
        _Tint ("Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Albedo", 2D) = "white" {}
        [NoScaleOffset] _NormalMap ("Normals", 2D) = "bump" {}
        _BumpScale ("Bump Scale", Float) = 1
        [Gamma] _Metallic ("Metallic", Range(0, 1)) = 0
        _Smoothness ("Smoothness", Range(0, 1)) = 0.1
    }
```

将这个比例系数纳入计算。

```c
sampler2D _NormalMap;
float _BumpScale;

…

void InitializeFragmentNormal(inout Interpolators i) {
    i.normal.xy = tex2D(_NormalMap, i.uv).wy * 2 - 1;
    i.normal.xy *= _BumpScale;
    i.normal.z = sqrt(1 - saturate(dot(i.normal.xy, i.normal.xy)));
    i.normal = i.normal.xzy;
    i.normal = normalize(i.normal);
}
```

为了与使用高度图时的强度大致相同，将比例缩小到 0.25。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/bump-scale.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/scaled.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/scaled-details.png)  
*缩放过的凹凸*   

UnityStandardUtils 包含了 UnpackScaleNormal 函数。它会自动对法线贴图正确解码，并进行缩放。所以让我们利用这个方便的函数吧。

```c
void InitializeFragmentNormal(inout Interpolators i) {
//  i.normal.xy = tex2D(_NormalMap, i.uv).wy * 2 - 1;
//  i.normal.xy *= _BumpScale;
//  i.normal.z = sqrt(1 - saturate(dot(i.normal.xy, i.normal.xy)));
    i.normal = UnpackScaleNormal(tex2D(_NormalMap, i.uv), _BumpScale);
    i.normal = i.normal.xzy;
    i.normal = normalize(i.normal);
}
```

> **UnpackScaleNormal 是什么样子的？**

> Unity 针对不支持 DXT5nm 的平台定义了 UNITY_NO_DXT5nm 关键字。此时该函数切换到RGB格式，不支持法线缩放。 由于指令限制，它在 Shader Model 2 环境下也不支持缩放。因此，在移动平台时不要依赖于凹凸缩放。
>
```c
    half3 UnpackScaleNormal (half4 packednormal, half bumpScale) {
        #if defined(UNITY_NO_DXT5nm)
            return packednormal.xyz * 2 - 1;
        #else
            half3 normal;
            normal.xy = (packednormal.wy * 2 - 1);
            #if (SHADER_TARGET >= 30)
                // SM2.0: instruction count limitation
                // SM2.0: normal scaler is not supported
                normal.xy *= bumpScale;
            #endif
            normal.z = sqrt(1.0 - saturate(dot(normal.xy, normal.xy)));
            return normal;
        #endif
    }

```

## 2.4 把反照率和凹凸合起来

现在我们的法线贴图功能完成了，可以检查效果了。当只使用大理石反照率贴图时，我们的四边形看起来像完美抛光的石头。给他加上法线贴图，它会变成一个更有意思的表面。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/without-bumps.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/with-bumps.png)  
*没有法线和有法线的对比*

[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-6/normal-mapping/normal-mapping.unitypackage)


# 3 凹凸的细节

在第三节“合并纹理”中，我们创建了一个带有细节纹理的着色器。我们为反照率做了细节贴图，现在也可以为凹凸也做一个。首先，还是要为 My First Lighting 添加对反照率细节的支持。

```c
    Properties {
        _Tint ("Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Albedo", 2D) = "white" {}
        [NoScaleOffset] _NormalMap ("Normals", 2D) = "bump" {}
        _BumpScale ("Bump Scale", Float) = 1
        [Gamma] _Metallic ("Metallic", Range(0, 1)) = 0
        _Smoothness ("Smoothness", Range(0, 1)) = 0.1
        _DetailTex ("Detail Texture", 2D) = "gray" {}
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/detail-inspector.png)  
*加上反照率细节贴图*

我们不为细节 UV 单独添加插值器，而是在单个插值器中手动打包主 UV 和细节 UV。主 UV 放入 XY，细节 UV 放入 ZW。  

```c
struct Interpolators {
    float4 position : SV_POSITION;
//  float2 uv : TEXCOORD0;
    float4 uv : TEXCOORD0;
    float3 normal : TEXCOORD1;
    float3 worldPos : TEXCOORD2;

    #if defined(VERTEXLIGHT_ON)
        float3 vertexLightColor : TEXCOORD3;
    #endif
};
```

添加所需的变量并在顶点代码中填充插值器。

```c
sampler2D _MainTex, _DetailTex;
sampler2D _MainTex, _DetailTex;
float4 _MainTex_ST, _DetailTex_ST;

…

Interpolators MyVertexProgram (VertexData v) {
    Interpolators i;
    i.position = mul(UNITY_MATRIX_MVP, v.position);
    i.worldPos = mul(unity_ObjectToWorld, v.position);
    i.normal = UnityObjectToWorldNormal(v.normal);
    i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
    i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);
    ComputeVertexLightColor(i);
    return i;
}
```

现在当需要主 UV 时我们应该使用 i.uv.xy 而不是 i.uv。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    i.normal = UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
    i.normal = i.normal.xzy;
    i.normal = normalize(i.normal);
}

float4 MyFragmentProgram (Interpolators i) : SV_TARGET {
    InitializeFragmentNormal(i);

    float3 viewDir = normalize(_WorldSpaceCameraPos - i.worldPos);

    float3 albedo = tex2D(_MainTex, i.uv.xy).rgb * _Tint.rgb;

    …
}
```

把细节纹理和反照率相乘。

```c
    float3 albedo = tex2D(_MainTex, i.uv.xy).rgb * _Tint.rgb;
    albedo *= tex2D(_DetailTex, i.uv.zw) * unity_ColorSpaceDouble;
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/without-bumps.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/with-bumps.png)  
*加了细节的反照率，不带法线和带有法线*  

## 3.1 法线细节

既然我们的大理石材质的细节纹理是灰度图，我们可以使用它来生成法线贴图。复制它并将其导入类型更改为法线贴图。将其凹凸减少到 0.1 之类，并保留所有其他设置。

当 mipmap 淡出时，颜色渐渐变为灰色。结果就是 Unity 生成的法线细节贴图逐渐变平。最后它们一起淡出。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/detail-normals-inspector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/detail-normals-preview.png)  
*法线细节纹理*  

将法线细节贴图的属性添加到着色器。给它一个凹凸缩放。

```c
    Properties {
        _Tint ("Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Albedo", 2D) = "white" {}
        [NoScaleOffset] _NormalMap ("Normals", 2D) = "bump" {}
        _BumpScale ("Bump Scale", Float) = 1
        [Gamma] _Metallic ("Metallic", Range(0, 1)) = 0
        _Smoothness ("Smoothness", Range(0, 1)) = 0.1
        _DetailTex ("Detail Texture", 2D) = "gray" {}
        [NoScaleOffset] _DetailNormalMap ("Detail Normals", 2D) = "bump" {}
        _DetailBumpScale ("Detail Bump Scale", Float) = 1
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/detail-bumps-inspector.png)  
*法线细节贴图和缩放*  

添加所需的变量并获取法线细节贴图，如同主法线贴图一样。在我们动手合并之前，暂时只显示细节法线。

```c
sampler2D _NormalMap, _DetailNormalMap;
float _BumpScale, _DetailBumpScale;

…

void InitializeFragmentNormal(inout Interpolators i) {
    i.normal = UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
    i.normal =
        UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
    i.normal = i.normal.xzy;
    i.normal = normalize(i.normal);
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/detail-normals.png)  
*凹凸细节*

## 3.2 混合法线

为了合并，我们将主反照率和细节反照率相乘。 法线不能这样做，因为它们是矢量。 但我们可以对它们求平均，在归一化之前。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    float3 mainNormal =
        UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
    float3 detailNormal =
        UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
    i.normal = (mainNormal + detailNormal) * 0.5;
    i.normal = i.normal.xzy;
    i.normal = normalize(i.normal);
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/averaged-normals.png)  
*平均化的法线*

结果不是很好。 主凹凸和细节凹凸都变平了。理想情况下，如果其中之一是平的，它就不应该影响另一个。

我们在这里试图做的是合并两个高度场。平均没有意义。叠加它们更有意义。当两个高度函数叠加时，它们的斜率——也就是它们的导数——也被叠加。我们可以从法线中提取导数吗？

之前，我们构造法线的时候，是通过归一化$\begin{bmatrix}-f_{u}^{'} \\\\ 1 \\\\ -f_{v}^{'} \end{bmatrix}$. 我们的贴图中的法线都是这个形式，除了 Y 和 Z 分量交换了位置，即$\begin{bmatrix}-f_{u}^{'} \\\\ -f_{v}^{'} \\\\ 1 \end{bmatrix}$。但是实际上的法线是经过归一化的，也就是$\begin{bmatrix}-sf_{u}^{'}  \\\\ -sf_{v}^{'} \\\\ s\end{bmatrix}$，其中 s 为任意的缩放因子。这样一来，Z 分量就恰好等于这个因子，用 X 和 Y 分别除以 Z 就可以得到偏导数。只有一种情况除外——Z 等于零，也就是一个完全垂直的斜率。但是我们的凹凸远没有那么陡峭，所以不需要担心。

既然我们有了导数，就可以把导数求和以得到合并高度场的导数，然后再反求法向量。归一化之前的法向量是$\begin{bmatrix}\frac{M_{x}}{M_{z}}+\frac{D_{x}}{D_{z}} \\\\ \frac{M_{y}}{M_{z}}+\frac{D_{y}}{D_{z}} \\\\ 1 \end{bmatrix}$.

```c
void InitializeFragmentNormal(inout Interpolators i) {
    float3 mainNormal =
        UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
    float3 detailNormal =
        UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
    i.normal =
        float3(mainNormal.xy / mainNormal.z + detailNormal.xy / detailNormal.z, 1);
    i.normal = i.normal.xzy;
    i.normal = normalize(i.normal);
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/added-derivatives.png)  
*导数相加*

这样看起来好多了！对于基本是平的法线贴图，合并的效果很好。但是叠加比较陡的斜率还是会损失细节。一个略微有改进的方法称为 whiteout 混合。首先把新法线乘以$M_{z}D_{z}$. 这样毫无问题因为之后还会归一化。我们由此得到$\begin{bmatrix}M_{x}D_{z}+D_{x}M_{z} \\\\ M_{y}D_{z}+D_{y}M_{z} \\\\ M_{z}D_{z} \end{bmatrix}$. 再去掉 X 和 Y 的缩放，得到$\begin{bmatrix}M_{x}+D_{x} \\\\ M_{y}+D_{y} \\\\ M_{z}D_{z} \end{bmatrix}$.

这种调整夸大了 X 和 Y 分量，因此在陡坡会产生更明显的凹凸。但优点是当其中一个法线持平时，另一个法线不会改变。

> **这种方法为何叫 whiteout 混合？**  
> 这种方法首先由 Christopher Oat 在 SIGGRAPH'07 公开提出。它被用在 AMD 的 Ruby：Whiteout 演示中，因此得名。

```c
    i.normal =
        float3(mainNormal.xy + detailNormal.xy, mainNormal.z * detailNormal.z);
//      float3(mainNormal.xy / mainNormal.z + detailNormal.xy / detailNormal.z, 1);
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/mixed-normals.png)  
*Whiteout 算法混合的法线，加上反照率贴图*

UnityStandardUtils 包含了 BlendNormals 函数，该函数也使用 whiteout 混合。我们就用它吧。它还把结果归一化了，因此我们不必再自己做了。

> **BlendNormals 方法长什么样子？**  
> 它做的事就是我们之前做的。
```c
half3 BlendNormals (half3 n1, half3 n2) {
    return normalize(half3(n1.xy + n2.xy, n1.z * n2.z));
}
```

[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-6/bump-details/bump-details.unitypackage)

# 4 切线空间

到目前为止，我们始终假设我们正在对平行于 XZ 平面的平面进行着色。但是如果试图任意使用这种技术，它必须适用于任意几何体。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/mesh-normals.png)![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/mapped-normals.png)  
*立方体和球体上的错误凹凸贴图*

我们可以对齐立方体的一个面，使其符合之前的假设。 也可以通过交换和翻转坐标轴来支持其他面。但这都假设了一个轴对齐的立方体。当立方体具有任意旋转角度时，问题就变得更加复杂。我们必须变换凹凸贴图的计算结果，使其与面的真实方向相匹配。

我们能知道一个面的朝向吗？为此，我们需要定义 U 轴和 V 轴的方向矢量。这两个矢量再加上法线向量，定义了一个符合我们假设的3D空间。一旦我们拥有了这个空间，我们就可以用它来将凹凸转化为世界空间。

既然我们已经有了法向量 N，我们只需要再来一个向量。这两个向量的叉积就可以定义第三个向量。

这另一个向量一般作为顶点数据的一部分，由网格提供。因为它位于由曲面法线定义的平面中，所以被称为切向量 T. 按照惯例，此向量匹配 U 轴，指向右侧。

第三个向量称为 B，副切线 bitangent 或者副法线 binormal。Unity 将其称为副法线，因此我也这么称呼。此向量定义 V 轴，指向前方。 导出副法线的标准方法是通过$B = N \times T$。 然而，这样算出的向量是向后而非向前的向量。要纠正此问题，结果必须乘以因子 -1。该因子存储为 T 的额外第四个分量。

> **为什么在切线向量中存储-1？**  
> 创建具有轴对称性的 3D 模型（如人和动物）时，一个常用技术是把网格左右镜像。这意味着您只需编辑网格的一侧。而且只需要一半纹理数据。这意味着法线和切线向量也是镜像的。但是，副法线不应该镜像！为了支持这一点，镜像部分切线的第四个分量将存储为 1 而非 -1。所以这个数据实际上是可变的。所以必须明确提供。

因此，我们可以使用顶点法线和切线来构造与网格表面相匹配的3D空间。此空间称为切线空间、切线基或者 TBN 空间。在立方体上，每个面的切线空间是一致的。 在球体上，切线空间在其表面环绕。

如果要构造这个空间，网格必须包含切线向量。幸运的是，Unity 的默认网格包含这些数据。将网格导入 Unity 时，可以导入自己的切线，也可以让 Unity 为你生成切线。

## 4.1 把切线空间描出来

为了了解切线空间的工作原理，让我们编写一套快速将切线空间可视化的代码。创建一个名为 TangentSpaceVisualizer 的组件，为其添加 OnDrawGizmos 方法。

```c
using UnityEngine;

public class TangentSpaceVisualizer : MonoBehaviour {

    void OnDrawGizmos () {
    }
}
```

每次触发 OnDrawGizmos 时，从物体的 Mesh Filter 中抓取网格，并用它来显示其切线空间。当然，这仅在网格存在时才有效。要抓取 shadedMesh，而不是 mesh。前者会直接引用网格资源，后者会创建一份副本。

> **为什么 MeshFilter.mesh 会创建一份副本？**  
> 假设有一个使用网格资源的物体，你想在运行时调整该物体的网格。这时你需要为该网格资源创建一个本地副本而非直接修改资源。因此 MeshFilter.mesh 会创建副本。

```c
    void OnDrawGizmos () {
        MeshFilter filter = GetComponent<MeshFilter>();
        if (filter) {
            Mesh mesh = filter.sharedMesh;
            if (mesh) {
                ShowTangentSpace(mesh);
            }
        }
    }

    void ShowTangentSpace (Mesh mesh) {
    }
```

首先我们要显示的是法线向量。从网格中获取顶点位置和法线，并以此绘制线段。我们必须将其转换为世界空间，以便在场景中匹配几何体。由于法线对应于切线空间中的向上方向，我们用绿色绘制。

```c
    void ShowTangentSpace (Mesh mesh) {
        Vector3[] vertices = mesh.vertices;
        Vector3[] normals = mesh.normals;
        for (int i = 0; i < vertices.Length; i++) {
            ShowTangentSpace(
                transform.TransformPoint(vertices[i]),
                transform.TransformDirection(normals[i])
            );
        }
    }

    void ShowTangentSpace (Vector3 vertex, Vector3 normal) {
        Gizmos.color = Color.green;
        Gizmos.DrawLine(vertex, vertex + normal);
    }
```

> **每一帧取一遍网格数据是不是效率不够好？**  
> 是的。不过既然我们是快速可视化，我们就不纠结于这里的性能优化吧。

把这个组件挂到带有网格的物体上，查看其顶点法线。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/gizmos-normal.png)  
*显示法线*

线段的合理长度是多少？这取决于物体形状。所以我们添加一个可配置的比例系数。我们还要支持一个偏移配置，它将线段推离表面，这样可以更轻松地检查重叠的顶点。

```c
    public float offset = 0.01f;
    public float scale = 0.1f;

    void ShowTangentSpace (Vector3 vertex, Vector3 normal) {
        vertex += normal * offset;
        Gizmos.color = Color.green;
        Gizmos.DrawLine(vertex, vertex + normal * scale);
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/inspector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/gizmos-offset-scale.png)  
*偏移和缩放*

下一步是切向量。虽然它们是 4 维矢量，用法就和普通矢量一样。切线指向切线空间的右侧，所以用红色绘制。

```c
    void ShowTangentSpace (Mesh mesh) {
        Vector3[] vertices = mesh.vertices;
        Vector3[] normals = mesh.normals;
        Vector4[] tangents = mesh.tangents;
        for (int i = 0; i < vertices.Length; i++) {
            ShowTangentSpace(
                transform.TransformPoint(vertices[i]),
                transform.TransformDirection(normals[i]),
                transform.TransformDirection(tangents[i])
            );
        }
    }

    void ShowTangentSpace (Vector3 vertex, Vector3 normal, Vector3 tangent) {
        vertex += normal * offset;
        Gizmos.color = Color.green;
        Gizmos.DrawLine(vertex, vertex + normal * scale);
        Gizmos.color = Color.red;
        Gizmos.DrawLine(vertex, vertex + tangent * scale);
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/gizmos-tangents.png)  
*显示法线和切线*

最后用蓝色绘制副法线。

```c
    void ShowTangentSpace (Mesh mesh) {
        …
        for (int i = 0; i < vertices.Length; i++) {
            ShowTangentSpace(
                transform.TransformPoint(vertices[i]),
                transform.TransformDirection(normals[i]),
                transform.TransformDirection(tangents[i]),
                tangents[i].w
            );
        }
    }

    void ShowTangentSpace (
        Vector3 vertex, Vector3 normal, Vector3 tangent, float binormalSign
    ) {
        …
        Vector3 binormal = Vector3.Cross(normal, tangent) * binormalSign;
        Gizmos.color = Color.blue;
        Gizmos.DrawLine(vertex, vertex + binormal * scale);
    }
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/gizmos-binormal.png)  
*显示完整的切线空间*

可以看到，在默认的立方体上，切线空间在面和面之间不同，但在面的内部各顶点是一致的。在球体上，切线空间在每一个顶点都不一致，结果就是切线空间在三角形上要进行插值，形成曲面。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/sphere-tangent-space.png)  
*默认球体周围的切线空间*

在球体周围围绕切线空间是有问题的。Unity 的默认球体使用经纬度纹理布局。这就像在球上缠绕一张纸，形成一个圆筒。然后再把圆筒的顶部和底部弄皱，使它们与球体匹配。因此球体的两极非常混乱。而 Unity 的默认球体的顶点布局是立方体式的，这更加重了问题。它们适用于快速原型，但不要指望默认网格能够产生高质量的结果。

## 4.2 着色器里的切线空间

要在着色器里获取切线，我们需要把它加入 VertexData 结构体。

```c
struct VertexData {
    float4 position : POSITION;
    float3 normal : NORMAL;
    float4 tangent : TANGENT;
    float2 uv : TEXCOORD0;
};
```

插值器里也必须包含它们。插值器的顺序无关紧要，但我喜欢将法线和切线放在一起。

```c
struct Interpolators {
    float4 position : SV_POSITION;
    float4 uv : TEXCOORD0;
    float3 normal : TEXCOORD1;
    float4 tangent : TEXCOORD2;
    float3 worldPos : TEXCOORD3;

    #if defined(VERTEXLIGHT_ON)
        float3 vertexLightColor : TEXCOORD4;
    #endif
};
```

在顶点代码里将切线变换到世界空间，使用 UnityCG 中的 UnityObjectToWorldDir 函数。当然这个变换只对切线的 XYZ 分量有效，其 W 分量需要原样传入插值器。

```c
Interpolators MyVertexProgram (VertexData v) {
    Interpolators i;
    i.position = mul(UNITY_MATRIX_MVP, v.position);
    i.worldPos = mul(unity_ObjectToWorld, v.position);
    i.normal = UnityObjectToWorldNormal(v.normal);
    i.tangent = float4(UnityObjectToWorldDir(v.tangent.xyz), v.tangent.w);
    i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
    i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);
    ComputeVertexLightColor(i);
    return i;
}
```

> **UnityObjectToWorldDir 长什么样子？**  
> 只是一个简单的方向变换，使用了世界-模型矩阵。
```c
// Transforms direction from object to world space
inline float3 UnityObjectToWorldDir( in float3 dir ) {
    return normalize(mul((float3x3)unity_ObjectToWorld, dir));
}
```

我们现在可以在片段着色器中获取法线和切线了。这样就可以在 InitializeFragmentNormal 函数中构造副法线。但是，我们必须注意一定要用原始的法线，而非凹凸贴图的法线。凹凸贴图的法线存在于切线空间中，请务必区分清楚。

```c
void InitializeFragmentNormal(inout Interpolators i) {
    float3 mainNormal =
        UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
    float3 detailNormal =
        UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
    float3 tangentSpaceNormal = BlendNormals(mainNormal, detailNormal);
    tangentSpaceNormal = tangentSpaceNormal.xzy;

    float3 binormal = cross(i.normal, i.tangent.xyz) * i.tangent.w;
}
```

> **法线和切线不需要归一化么？**  
> 如果我们想确保使用的是单位向量，那么确实应该这样做。实际上，如果要创建正确的 3D 空间，我们还应该确保法线和切线之间的角度为 90°。
> 
> 但是，我们不打算在这纠结。原因在下一节说明。

现在我们可以把贴图中获取的法线从切线空间转为世界空间了。

```c
    float3 binormal = cross(i.normal, i.tangent.xyz) * i.tangent.w;

    i.normal = normalize(
        tangentSpaceNormal.x * i.tangent +
        tangentSpaceNormal.y * i.normal +
        tangentSpaceNormal.z * binormal
    );
```

交换 YZ 分量这一步也可以去掉了，和空间转换合并成一步。

```c
//  tangentSpaceNormal = tangentSpaceNormal.xzy;
    
    float3 binormal = cross(i.normal, i.tangent.xyz) * i.tangent.w;

    i.normal = normalize(
        tangentSpaceNormal.x * i.tangent +
        tangentSpaceNormal.y * binormal +
        tangentSpaceNormal.z * i.normal
    ;
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/tangent-to-world.png)  
*转换过后的法线*

构造副法线时，还有一个额外的细节：假设一个物体的缩放为(-1,1,1)。这意味着它是镜像变换的。在这种情况下，我们必须翻转副法线，以正确地镜像切线空间。实际上，当有奇数个维度为负数时，就必须这样做。UnityShaderVariables 定义了 float4 unity_WorldTransformParams 变量以帮助我们。当我们需要翻转副法线时，它的第四个分量为 -1，否则为 1。

```c
    float3 binormal = cross(i.normal, i.tangent.xyz) *
        (i.tangent.w * unity_WorldTransformParams.w);
```

> **unity_WorldTransformParams 的其他分量是做什么用的？**  
> 我不知道。至少目前好像没什么用。

## 4.3 同步切线空间

当 3D 艺术家创建细节模型时，通常的方法是构建一个非常高分辨率的模型。所有细节都是实际的 3D 几何体。然后为了能在游戏中使用，生成模型的低分辨率版本。把细节烘焙到此模型的纹理中。

高分辨率模型的法线被烘焙到法线贴图中，此时法线从世界空间转换为切线空间。在游戏中渲染低分辨率模型时，做相反的转换。

只要两个转换使用相同的算法和相同的切线空间，此过程就毫无问题。若非如此，游戏中的渲染结果将是错误的。这样 3D 艺术家会非常痛苦。因此，必须确保法线贴图生成器，Unity的网格导入过程和着色器都已同步。这称为切线空间工作流的同步。

> **法线贴图怎么样？**  
> 我们法线贴图由高度场生成。因此它们的参考系是平坦的，并且它们的切线空间是规则的。因此，当它们被应用于具有弯曲切线空间的物体时，最终的法线将会失真，和高度场不一致。不过这样也没什么问题，因为大理石的精确外观并不重要。

Unity 从 5.3 开始使用 mikktspace 算法。 因此，请确保在生成法线贴图时也使用 mikktspace。导入网格时，你可以允许 Unity 为您生成切线向量，因为它使用 mikktspace 。或者，自己导出 mikktspace 切线供 Unity 使用。

> **mikktspace 是什么？**  
> 它是一种生成切线空间和法线的标准，由 Morten Mikkelsen 提出。这个名字是 Mikkelsen's tangent space 的简写。
> 
> 对于与 mikktspace 同步的着色器，它必须在顶点程序中接收归一化的法线和切向量。然后对这些矢量进行插值，而不在每个片段重新归一化。通过计算 cross(normal.xyz，tangent.xyz) * tangent.w 得到副法线。这样我们的着色器就与 mikktspace 同步，Unity 的标准着色器也是如此。
> 
> 请注意，mikktspace 不保证是正交基，法线和切线之间的角度可以自由变化。只要失真不会太大，这不是问题。因为我们只使用它来转换法线，所以只有一致性是重要的。

使用 mikktspace 时，有一个可选项：副法线可以在片段程序中构建——就像我们一样——或者在顶点程序中构建——就像 Unity 一样。两种方法会产生略微不同的副法线。

![](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/binormal-difference.png)  
*夸大的副法线的差异*  

因此，在为 Unity 生成法线贴图时，请使用按顶点计算副法线的设置。或者随意并假设它们是按片段计算的，并使用片段计算副法线的着色器。

> **切线空间很麻烦，不用它行吗？**  
> 由于切线空间环绕物体表面，因此物体的确切形状就不重要了，所以您可以将任何一张切线空间的法线贴图应用于物体。您也可以像我们一样平铺贴图。此外，当网格因动画而变形时，切线空间——以及法线贴图——也会随之变形。
> 
> 如果取消切线空间，则必须使用模型空间法线贴图。这些贴图不会粘在表面上，因此不能平铺，不能应用于不同的形状，并且不能与网格一起变形。此外，它们还不适用于纹理压缩。
> 
> 所以使用切线空间有着充足的理由。话虽如此，也有一些方法可以处理切线空间法线，而无需明确提供切线向量。这些技术依赖着色器的导数指令，我们将在以后的教程中介绍。但这并不能消除对同步工作流的需求。

## 4.4 按顶点或是按片段求副法线

如果想与 Unity 的标准着色器保持一致，我们必须计算每个顶点的副法线。这样做的好处是不必在片段着色器中计算叉积。缺点是需要一个额外的插值器。

如果您不确定应该使用哪种方法，可以始终支持这两种方法。假设当定义了 BINORMAL_PER_FRAGMENT时，按片段计算副法线，否则按顶点计算。那么在前一种情况下，我们保留 float4 tangent 插值器。在后一种情况下，我们需要两个 float3 插值器。

```c
struct Interpolators {
    float4 position : SV_POSITION;
    float4 uv : TEXCOORD0;
    float3 normal : TEXCOORD1;

    #if defined(BINORMAL_PER_FRAGMENT)
        float4 tangent : TEXCOORD2;
    #else
        float3 tangent : TEXCOORD2;
        float3 binormal : TEXCOORD3;
    #endif

    float3 worldPos : TEXCOORD4;

    #if defined(VERTEXLIGHT_ON)
        float3 vertexLightColor : TEXCOORD5;
    #endif
};
```

> **我们是跳过了一个插值器吗？**  
> 我们只在需要一个副法线插值器时使用了 TEXCOORD3。因此，当定义 BINORMAL_PER_FRAGMENT 时，我们跳过了此插值器索引。这样也是允许的，我们可以使用任何想要的插值器索引，只要不超过最大值。

我们将副法线的计算定义成自己的函数，就可以任意在顶点或片段着色器中使用它。

```c
float3 CreateBinormal (float3 normal, float3 tangent, float binormalSign) {
    return cross(normal, tangent.xyz) *
        (binormalSign * unity_WorldTransformParams.w);
}

Interpolators MyVertexProgram (VertexData v) {
    Interpolators i;
    i.position = mul(UNITY_MATRIX_MVP, v.position);
    i.worldPos = mul(unity_ObjectToWorld, v.position);
    i.normal = UnityObjectToWorldNormal(v.normal);

    #if defined(BINORMAL_PER_FRAGMENT)
        i.tangent = float4(UnityObjectToWorldDir(v.tangent.xyz), v.tangent.w);
    #else
        i.tangent = UnityObjectToWorldDir(v.tangent.xyz);
        i.binormal = CreateBinormal(i.normal, i.tangent, v.tangent.w);
    #endif
        
    i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
    i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);
    ComputeVertexLightColor(i);
    return i;
}

…

void InitializeFragmentNormal(inout Interpolators i) {
    float3 mainNormal =
        UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
    float3 detailNormal =
        UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
    float3 tangentSpaceNormal = BlendNormals(mainNormal, detailNormal);

    #if defined(BINORMAL_PER_FRAGMENT)
        float3 binormal = CreateBinormal(i.normal, i.tangent.xyz, i.tangent.w);
    #else
        float3 binormal = i.binormal;
    #endif
    
    i.normal = normalize(
        tangentSpaceNormal.x * i.tangent +
        tangentSpaceNormal.y * binormal +
        tangentSpaceNormal.z * i.normal
    );
}
```

由于没有定义 BINORMAL_PER_FRAGMENT，我们的着色器现在将按顶点计算副法线。如果要为按片段计算，则必须在某处定义 BINORMAL_PER_FRAGMENT。您可以将它视为库文件的配置选项。因此，在包含 My Lighting 之前，在 My First Lighting Shader 中定义它是有作用的。

因为所有的 pass 应该使用相同的设置，我们必须同时在 base pass 和 additive pass 中定义它。但我们也可以将它放在我们着色器顶部的 CGINCLUDE 块中。该块的内容会包含在所有 CGPROGRAM 块中。

```c
    Properties {
        …
    }

    CGINCLUDE

    #define BINORMAL_PER_FRAGMENT

    ENDCG

    SubShader {
        …
    }
```

从编译过的着色器代码中可以看出区别，比如这是 D3D11 所使用的插值器，没有定义 BINORMAL_PER_FRAGMENT。

> // Output signature:
> //
> // Name                 Index   Mask Register SysValue  Format   Used
> // -------------------- ----- ------ -------- -------- ------- ------
> // SV_POSITION              0   xyzw        0      POS   float   xyzw
> // TEXCOORD                 0   xyzw        1     NONE   float   xyzw
> // TEXCOORD                 1   xyz         2     NONE   float   xyz 
> // TEXCOORD                 2   xyz         3     NONE   float   xyz 
> // TEXCOORD                 3   xyz         4     NONE   float   xyz 
> // TEXCOORD                 4   xyz         5     NONE   float   xyz 

下面是定义了 BINORMAL_PER_FRAGMENT 的：

> // Output signature:
> //
> // Name                 Index   Mask Register SysValue  Format   Used
> // -------------------- ----- ------ -------- -------- ------- ------
> // SV_POSITION              0   xyzw        0      POS   float   xyzw
> // TEXCOORD                 0   xyzw        1     NONE   float   xyzw
> // TEXCOORD                 1   xyz         2     NONE   float   xyz 
> // TEXCOORD                 2   xyzw        3     NONE   float   xyzw
> // TEXCOORD                 4   xyz         4     NONE   float   xyz 

---
  
下一课是[影子](https://catlikecoding.com/unity/tutorials/rendering/part-7/)。  
[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/tangents.unitypackage)