
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

  

---
  
下一课是[影子](https://catlikecoding.com/unity/tutorials/rendering/part-7/)。  
[unitypackage](https://catlikecoding.com/unity/tutorials/rendering/part-6/tangents/tangents.unitypackage)