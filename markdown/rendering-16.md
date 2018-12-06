渲染之十六 静态光照
======

 * 光照贴图的渲染和采样
 * 在烘焙光照时使用法线
 * 使用光照探针组

这一篇是渲染教程的第十六部分，上一部分里我们渲染了延迟光照，这一部分里，我们研究光照贴图。

这一节教程是基于 Unity 5.6.0

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/tutorial-image.jpg)  
*烘焙光照*


# 1 光照贴图

计算光照是非常昂贵的过程，延迟渲染使我们可以使用很多光源，但是影子方面依然有局限性。如果我们的场景是动态的，我们就无法避免这些计算，但是如果光和物体都不变，我们可以只计算一次然后复用，这样我们就可以在场景上放很多光源，并且不需要在运行时渲染。预烘焙还有一个好处是可以使用区域光源，而实时渲染不可以。

光照的哪些部分要参与预计算，这个不一定，在这个教程里，我们始终会把所有部分都放进光照贴图里，所以场景上完全没有动态光照。

为了演示光照贴图，我创建了一个简单的场景，有一个很简单的结构，带有影子，在结构里面和周围放了一些球体，所有的物体都是 Unity 的默认材质。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/test-scene.png)
*烘焙测试场景*


## 1.1 烘焙光照

为了启用光照贴图，把唯一的光源的模式从 Realtime 改为Baked。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/baked-light.png)
*烘焙主光源*

把主光源设为烘焙之后，这个光源就不包括在动态光源中了，从动态物体的角度看来，这个光源就不存在了，唯一剩下基于主光源的的只有环境光。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/no-realtime-light.png)  
*没有直接光照了*  

为了真正地使用光照贴图，打开 Lighting 窗口，找到 Mixed Lighting 下的 Baked Global Illumination 选项，打开这个选项，把 Lighting Mode 选为 Baked Indirect——尽管叫Indirect，实际上也包括了直接光照——这是向场景里增加间接光照的一般方法。然后，保证 Realtime Global Illumination 选项是关闭的，因为我们现在不支持他。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/baked-indirect.png)  
*Baked Indirect 模式*  


## 1.2 静态物体

现在场景里的物体应该都正常了。这些物体不能移动，把他们标记为 static，以告诉 Unity 这些是静态物体，方法是在 Inspector 面板的右上角找到 Static 开关并勾选上。

> **光源物体需要标记成 static 吗？**  
> 不需要，只要光源的相关设置正确就可以。

静态系统有许多子系统，Static 选项有一个下拉菜单，每个物体在每个子系统中都可以微调成静态或者非静态。此处我们只关心光照贴图，最简单的方式是选择 Everything。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/static-object.png)  
*静态物体*  


一个物体对光照贴图是不是静态，也可以在他的 MeshRenderer 组件的 Inspector 面板上查看和设置。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/lightmap-static.png)  
*光照贴图静态物体*  


现在所有的物体都是静态的了，都会在光照烘焙的过程中起作用。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/baked-scene.png)  
*带有烘焙光的场景*  

注意烘焙出的光照会比实时的光照暗一点，因为高光部分丢失了，只剩下了漫反射。高光取决于观察角度，受相机位置影响，一般来说相机是运动的，所以高光没办法放到光照贴图里，也就是说，光照贴图适合不太亮的光源和比较暗的表面，不适合很强的直接光照和闪亮的表面。如果想要高光，就得使用实时光照，所以一般都会把烘焙光和实时光混合使用。

>**为什么我没看到烘焙光？**  
>确认一下光照贴图确实生成了并且及时更新了，在 Lighting 窗口最下面勾选 Auto Generate，或者手动点击按钮产生新的光照贴图。


## 1.3 光照贴图的设置

Lighting 窗口有一部分是关于光照贴图的设置，这里你可以在质量、体积和烘焙时间之间做一个权衡。你也可以在 Enlighten 和 Progressive 两种烘焙方式之间切换，后者可以渐进式地烘焙贴图，优先烘焙Scene视图里可见的物体，这个特性在编辑器里特别方便，在这篇教程里我使用 Enlighten。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/lightmap-settings.png)  
*默认的光照贴图设置*  

首先，把 Directional Mode 选项设为 Non-Direction ，另外一种模式我们之后再处理。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/non-directional.png)  
*Non-Direction 模式*  

烘焙好的光照信息存储在纹理中，你可以把 Lighting 窗口从 Scene 标签卡切换到  Global Maps 标签卡。基于默认设置，我的测试场景生成了一张1024×1024的贴图。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/lightmap.png)  
*光照贴图*  

Unity 的默认物体都有一套为光照贴图准备的 UV 坐标，对于导入的模型，你可以使用你自己的 UV 坐标，也可以让 Unity 为你生成，烘焙之后，在光照贴图上可以看到物体的展开（unwraps），其所占的空间取决于物体在场景中的尺寸以及光照贴图的分辨率。如果一张贴图不够用，Unity 会创建多张贴图。


![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/low-quality.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/higher-quality.png)  
*不同分辨率的光照贴图差别巨大*   

什么样的设置最好，取决于你的项目，你可能要一直调整，直到达到一个不错的平衡点。注意显示质量也很大程度上受光照贴图展开的质量影响，无缝贴图会造成明显的瑕疵，Unity 默认的球体就是个好例子——它不适合光照贴图。


## 1.4 间接光照

虽然烘焙意味着失去了高光，但是我们得到了间接光照，间接光照的意思是，这束光在到达我们的眼睛之前在不同的表面反弹过很多次。因为光线会在角落里来回反弹，所以本来应该是阴影的区域也有了一点光照，这个特性是实时演算时无法模拟的，但是烘焙可以。

为了清楚地看到实时光照和烘焙的区别，把环境光的强度设为0，这样消除了天空盒的影响，所以现在所有的光都是来自方向光。


![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/no-environmental-realtime.png)  

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/no-environmental-lightmapped.png)  
*没有环境光的时候，实时 vs. 光照贴图*   

一个光子每一次经过反弹，都会丢失一点能量，并且被表面染上一点颜色。Unity 在烘焙间接光的时候也考虑了这些性质，其结果就是，每个物体会被临近的物体染上一点颜色。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/green-floor-realtime.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/green-floor-lightmapped.png)  
*绿色地板，实时 vs. 光照贴图*  


自发光的表面也会影响烘焙，成为间接光的光源。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/emissive-realtime.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/emissive-lightmapped.png)  
*自发光地板，实时 vs. 光照贴图*  


有关间接光照的一个特别设置是环境光遮蔽——Ambient Occlusion (AO) ——意思是间接光在角落和转折处产生的阴影。这属于一种人工增强，可以增强深度感。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/ambient-occlusion-settings.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/ambient-occlusion.png)  
*使用环境光遮蔽*  


环境光遮蔽完全取决于一个表面有多隐蔽，而不真正考虑光线从哪个方向来，有时候也会产生问题，比如和自发光表面一起使用的时候。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/ambient-occlusion-incorrect.png)  
*明显错了的 AO*  


## 1.5 透明

光照贴图可以——在一定程度上——处理半透明表面——光线可以透过表面，但是颜色不会受到影响。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/semitransparent-roof.png)  
*半透明的屋顶*  

断截（Cutout）材质也可以。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/cutout-roof.png)  
*断截材质*  


但是，这只对封闭的表面起作用，当使用像 Quad 一类的单面模型时，光线在不存在的一面也会被遮挡。如果另一面没有物体还好，否则在使用单向的半透明表面时就会出问题。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/quad-errors.png)  
*Quad 造成的错误*  


为了正确处理这种情况，你需要告诉烘焙器像透明表面一样对待他们，具体可以通过自定义光照贴图设置。选择 *Asset/Create/Lightmap Parameters* 创建一个资源，这个资源允许你针对每个物体自定义光照贴图设置。在这个例子里，我们只关心如何标记透明物体，所以开启 Is Transparent 选项，虽然他是 Precomputed Realtime GI 设置的一部分，也会影响所有的光照烘焙。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/lightmap-parameters.png)  
*指定透明物体*  


为了启用这些设置，在物体的 Mesh Renderer 的 Inspector 面板上选择一下，你的自定义资源的名字会出现在 *Lightmap Parameters* 旁边的下拉列表里。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/using-custom-parameters.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/lightmapping/quad-correct.png)  
*为透明的面使用自定义参数*   


把一个物体标记为透明会改变其对间接光的行为，透明物体会让间接光透过，不透明物体则不会。


# 2 使用光照贴图

现在我们知道了光照贴图如何工作，我们可以对我们的第一个光照着色器添加支持了，第一步是对光照贴图采样：修改场景上所有球体的设置，使用一个白色的材质，用我们自己的着色器。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/using-lightmaps/spheres-custom.png)  
*使用我们自己的白色材质的球体*  


## 2.1 光照贴图着色器的变体

如果一个着色器支持光照贴图，Unity会去寻找带有 LIGHTMAP_ON 关键字的变体，所以我们对这个关键字添加一条 multi-compile 指令。当使用前向渲染时，光照贴图只在 base pass 里采样。

```c
#pragma multi_compile _ SHADOWS_SCREEN
#pragma multi_compile _ VERTEXLIGHT_ON
#pragma multi_compile _ LIGHTMAP_ON
#pragma multi_compile_fog 
```

当使用光照贴图时，Unity 不会启用顶点光照，所以这两个关键字是互斥的，我们不需要同时支持 VERTEXLIGHT_ON 和 LIGHTMAP_ON。

```c
#pragma multi_compile _ SHADOWS_SCREEN
// #pragma multi_compile _ VERTEXLIGHT_ON
// #pragma multi_compile _ LIGHTMAP_ON
#pragma multi_compile _ LIGHTMAP_ON VERTEXLIGHT_ON
#pragma multi_compile_fog 
```

延迟渲染也支持光照贴图，所以把关键字也加入延迟渲染的pass。

```c
#pragma multi_compile _ UNITY_HDR_ON 
#pragma multi_compile _ LIGHTMAP_ON 
```


## 2.2 光照贴图的坐标

对光照贴图采样的坐标存放在第二张纹理坐标——uv1——里，所以把这个管道加入 VertexData 结构体。

```c
struct VertexData {
	float4 vertex : POSITION;
	float3 normal : NORMAL;
	float4 tangent : TANGENT;
	float2 uv : TEXCOORD0;
	float2 uv1 : TEXCOORD1;
};
```

光照贴图坐标也需要插值，因为他和顶点光照是互斥的，所以可以同时使用TEXCOORD6.

```c
struct Interpolators {
	…

	#if defined(VERTEXLIGHT_ON)
		float3 vertexLightColor : TEXCOORD6;
	#endif

	#if defined(LIGHTMAP_ON)
		float2 lightmapUV : TEXCOORD6;
	#endif
};
```

顶点结构体里的坐标定义了用于光照贴图的网格的纹理展开，但是没有告诉我们光照贴图在哪里，也没有告诉我们其大小。我们还需要缩放和偏移这些坐标，来获取最终的纹理坐标，这个工作和一般的贴图坐标的变换是类似的，除非这个变换是物体特有的，而非材质定义的。光照贴图定义在 UnityShaderVariables 中，名字是 unity_Lightmap

```c
Interpolators MyVertexProgram (VertexData v) {
	…

	i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
	i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);

	#if defined(LIGHTMAP_ON)
		i.lightmapUV = TRANSFORM_TEX(v.uv1, unity_Lightmap);
	#endif

	…
}
```


不幸的是，我们不能方便地使用宏 TRANSFORM_TEX，因为它是基于一个假设——光照贴图的变换定义为 unity_Lightmap_ST——然而实际上是 **unity_LightmapST**，因此我们得手工做这件事。

```c
i.lightmapUV = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
```

## 2.3 光照贴图采样

由于光照贴图数据属于间接光照，我们在 CreateIndirectLight 方法中对其进行采样，当光照贴图可用时，我们用他作为间接光照的来源，以代替球谐函数。

```c
UnityIndirect CreateIndirectLight (Interpolators i, float3 viewDir) {
        …
        #if defined(VERTEXLIGHT_ON)
                indirectLight.diffuse = i.vertexLightColor;
        #endif

        #if defined(FORWARD_BASE_PASS) || defined(DEFERRED_PASS)
                #if defined(LIGHTMAP_ON)
                        indirectLight.diffuse = 0;
                #else
                        indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
                #endif
                float3 reflectionDir = reflect(-viewDir, i.normal);
                …
        #endif
        return indirectLight;
}
```

>**为何对indirectLight.diffuse直接赋值，而不是加法？**  
>这暗示了，光照贴图永不与顶点光照为伍。

unity_Lightmap 的解压形式取决于平台，他在声明的时候使用的是 UNITY_DECLARE_TEX2D(unity_Lightmap)，采样的时候，我们也会使用宏 **UNITY_SAMPLE_TEX2D**，而非 tex2D，一会儿我们会看到其原因。

```c
indirectLight.diffuse = UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightmapUV);
```

我们现在得到了烘焙的间接光，但看起来不太对，因为光照贴图数据已经过编码，颜色值以RGBM格式或一半强度存储，以支持高强度光。 UnityCG 的 DecodeLightmap 函数负责为我们解码。

```c
indirectLight.diffuse = DecodeLightmap(UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightmapUV));
```


# 3.创建光照贴图

虽然光照贴图似乎已经能配合我们的着色器工作，但这仅适用于我们简单的测试场景，目前，烘焙器总是将我们的对象视为不透明且纯白色，即使它们并非如此。我们必须对着色器进行一些调整，甚至添加另一个 pass 以便完全支持光照贴图。

从现在开始，场景中的所有物体使用我们自己的着色器，不再使用默认材质。

## 3.1 半透明阴影

烘焙器不使用实时渲染管线，因此它不使用我们的着色器来完成其工作，当我们尝试使用半透明阴影时，问题就会浮现出来。现在给屋顶立方体设置一个材质，使其颜色的 alpha 分量小于1，从而变为半透明。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/semitransparent-roof-incorrect.png)  
*半透明屋顶，不正确*  

烘焙器仍然对待这个屋顶如同实心物体，这是错误的。他使用材质的渲染类型来决定如何对待这些表面，实际上，他确实知道屋顶是半透明的，只是如同不透明物体一样对待，这是因为他使用材质球的 \_Color 属性的 alpha 分量和主纹理来决定透明度，但是我们没有这个属性，我们的属性叫 \_Tint！

不幸的是，没有什么方法告诉烘焙器来使用哪个属性，所以为了使光照贴图起作用，我们只能把 \_Tint 改名叫 \_Color。首先修改我们的着色器属性名字：

```c
Properties {
//              _Tint ("Tint", Color) = (1, 1, 1, 1)
                _Color ("Tint", Color) = (1, 1, 1, 1)
                …
        }
```

然后，为了着色器逻辑正确，我们还要修改 *My Lighting* 中的变量名字：

```c
//float4 _Tint;
float4 _Color;
…
float3 GetAlbedo (Interpolators i) {
        float3 albedo = tex2D(_MainTex, i.uv.xy).rgb * _Color.rgb;
        …
}
float GetAlpha (Interpolators i) {
        float alpha = _Color.a;
        …
}
```

*My Shadows*里也同样

```c
//float4 _Tint;
float4 _Color;
…
float GetAlpha (Interpolators i) {
        float alpha = _Color.a;
        …
}
```

还要修改 MyLightingShaderGUI. 

```c#
void DoMain () {
                GUILayout.Label("Main Maps", EditorStyles.boldLabel);
                MaterialProperty mainTex = FindProperty("_MainTex");
                editor.TexturePropertySingleLine(
                        MakeLabel(mainTex, "Albedo (RGB)"), mainTex, FindProperty("_Color")
                );

                …
        }
```


## 3.2 断截阴影

断截阴影也有类似的问题。烘焙器希望 alpha 断截阈值存储在  \_Cutoff 属性中，但我们使用 \_AlphaCutoff。因此，它使用默认阈值 1。


解决方案还是采用Unity的命名约定，所以属性换个名字。

```c
Properties {
                …
//              _AlphaCutoff ("Alpha Cutoff", Range(0, 1)) = 0.5
                _Cutoff ("Alpha Cutoff", Range(0, 1)) = 0.5
                …
        }
```

修改 *My Lighting* 配合新名字

```c
//float _AlphaCutoff;
float _Cutoff;
…
FragmentOutput MyFragmentProgram (Interpolators i) {
        float alpha = GetAlpha(i);
        #if defined(_RENDERING_CUTOUT)
                clip(alpha - _Cutoff);
        #endif
        …
}
```

*My Shadows* 同样

```c
//float _AlphaCutoff;
float _Cutoff;

…

float4 MyShadowFragmentProgram (Interpolators i) : SV_TARGET {
        float alpha = GetAlpha(i);
        #if defined(_RENDERING_CUTOUT)
                clip(alpha - _Cutoff);
        #endif

        …
}
```

MyLightingShaderGUI 同样

```c#
void DoAlphaCutoff () {
                MaterialProperty slider = FindProperty("_Cutoff");
                EditorGUI.indentLevel += 2;
                editor.ShaderProperty(slider, MakeLabel(slider));
                EditorGUI.indentLevel -= 2;
        }
```


## 3.3 添加元通道

下一步是确保烘焙器使用了正确的反照率和自发光。目前所有东西都是白色实心物体，你可以把地板调成绿色看看，应该会制造出绿色间接光，但是现在仍然是白色。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/green-floor-incorrect.png)  
*绿地板，不正确*  


为了确定物体表面的颜色，烘焙器会寻找一个 Meta 模式的 pass，这个 pass 只会被烘焙器使用，不会导出到最终构建里。现在我们来加一个这样的 pass，他是一个基本的 pass，不能启用剔除，把这些代码放进一个新的 *MyLightmapping* 文件里。

```c
Pass {
    Tags {
        "LightMode" = "Meta"
    }

    Cull Off

    CGPROGRAM
    #pragma vertex MyLightmappingVertexProgram
    #pragma fragment MyLightmappingFragmentProgram

    #include "My Lightmapping.cginc"

    ENDCG
}


现在我们来确定反照率，高光颜色，光滑度和自发光，把所需的变量和函数从 *MyLighting* 拷贝到 *MyLightmapping*。我们只需要顶点位置和 uv 坐标，法线和切线不需要，但是在顶点着色器里需要光照贴图 uv。

```c
#if !defined(MY_LIGHTMAPPING_INCLUDED)
#define MY_LIGHTMAPPING_INCLUDED

#include "UnityPBSLighting.cginc"

float4 _Color;
sampler2D _MainTex, _DetailTex, _DetailMask;
float4 _MainTex_ST, _DetailTex_ST;

sampler2D _MetallicMap;
float _Metallic;
float _Smoothness;

sampler2D _EmissionMap;
float3 _Emission;

struct VertexData {
        float4 vertex : POSITION;
        float2 uv : TEXCOORD0;
        float2 uv1 : TEXCOORD1;
};

struct Interpolators {
        float4 pos : SV_POSITION;
        float4 uv : TEXCOORD0;
};

float GetDetailMask (Interpolators i) {
        …
}

float3 GetAlbedo (Interpolators i) {
        …
}

float GetMetallic (Interpolators i) {
        …
}

float GetSmoothness (Interpolators i) {
        …
}

float3 GetEmission (Interpolators i) {
        …
}

#endif
```

我们还是照旧使用这些函数，除了GetEmission，之前这个方法只在 forwards base pass 和 deferred pass 起作用，现在我们可以把这些限制通通删掉。

```c
float3 GetEmission (Interpolators i) {
//      #if defined(FORWARD_BASE_PASS) || defined(DEFERRED_PASS)
        #if defined(_EMISSION_MAP)
                return tex2D(_EmissionMap, i.uv.xy) * _Emission;
        #else
                return _Emission;
        #endif
//      #else
//              return 0;
//      #endif
}
```

这些函数只在对应的关键字下起作用，所以给pass加上 shader_feature

```c

    #pragma vertex MyLightmappingVertexProgram
    #pragma fragment MyLightmappingFragmentProgram

    #pragma shader_feature _METALLIC_MAP
    #pragma shader_feature _ _SMOOTHNESS_ALBEDO _SMOOTHNESS_METALLIC
    #pragma shader_feature _EMISSION_MAP
    #pragma shader_feature _DETAIL_MASK
    #pragma shader_feature _DETAIL_ALBEDO_MAP

    #include "My Lightmapping.cginc"
```

## 3.4 顶点代码

这个pass的顶点部分很简单，转换一下位置以及变换一下纹理坐标。

```c
Interpolators MyLightmappingVertexProgram (VertexData v) {
        Interpolators i;
    i.pos = UnityObjectToClipPos(v.vertex);

        i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
        i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);
        return i;
}
```

但是，我们实际上并没有在为相机渲染，我们正在渲染光照贴图，我们正在关联颜色与对象在光照贴图中展开的纹理。 要进行此映射，我们必须使用光照贴图坐标而非顶点位置，并使用正确的变换。

```c
    Interpolators i;
    v.vertex.xy = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
    v.vertex.z = 0;
    i.pos = UnityObjectToClipPos(v.vertex);
```

事实证明，要使这些代码在所有机器上起作用，顶点的z坐标必须有值，即使我们不用他。Unity的着色器提供了一个虚值，我们也可以这么做。

```c
        Interpolators i;
        v.vertex.xy = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
        v.vertex.z = v.vertex.z > 0 ? 0.0001 : 0;
    i.pos = UnityObjectToClipPos(v.vertex);
```


## 3.5 片段着色器

在这个片段代码里，我们要输出反照率和自发光，烘焙器会调用这个 pass 两次，每次取一个值。为了简单点，我们可以使用 UnityMetaPass 文件里的 UnityMetaFragment 函数，这个函数要求一个 UnityMetaInput 类型的参数，这个函数会自动决定输出哪个值以及如何编码。

UnityMetaInput 还包括了高光颜色——虽然光照贴图里没有——用来计算某些编辑器效果，我们现在先忽略。

```c
#include "UnityPBSLighting.cginc"
#include "UnityMetaPass.cginc"

…

float4 MyLightmappingFragmentProgram (Interpolators i) : SV_TARGET {
        UnityMetaInput surfaceData;
        surfaceData.Emission = 0;
        surfaceData.Albedo = 0;
        surfaceData.SpecularColor = 0;
        return UnityMetaFragment(surfaceData);
}
```

>**UnityMetaFragment长什么样？**
>unity_MetaFragmentControl 变量包括了一些标记，来告诉函数是应该输出反照率还是自发光，这里还有一些编辑器效果的变体，不过我这里删掉了因为与主题无关。

```c
half4 UnityMetaFragment (UnityMetaInput IN) {
        half4 res = 0;
        if (unity_MetaFragmentControl.x) {
                res = half4(IN.Albedo,1);

                // d3d9 shader compiler doesn't like NaNs and infinity.
                unity_OneOverOutputBoost = saturate(unity_OneOverOutputBoost);

                // Apply Albedo Boost from LightmapSettings.
                res.rgb = clamp(
                        pow(res.rgb, unity_OneOverOutputBoost), 0, unity_MaxOutputValue
                );
        }
        if (unity_MetaFragmentControl.y) {
                half3 emission;
                if (unity_UseLinearSpace)
                        emission = IN.Emission;
                else
                        emission = GammaToLinearSpace (IN.Emission);

                res = UnityEncodeRGBM(emission, EMISSIVE_RGBM_SCALE);
        }
        return res;
}
```


![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/no-indirect.png)  
*间接光设为零*  

为了获取自发光颜色，我们只需要调用 GetEmission，获取反照率可以再请出 DiffuseAndSpecularFromMetallic，这个方法需要输出高光和反射，所以我们需要提供相关变量，虽然我们不使用。我们可以使用surfaceData.SpecularColor来获取高光颜色。

```c
float4 MyLightmappingFragmentProgram (Interpolators i) : SV_TARGET {
        UnityMetaInput surfaceData;
        surfaceData.Emission = GetEmission(i);
        float oneMinusReflectivity;
        surfaceData.Albedo = DiffuseAndSpecularFromMetallic(
                GetAlbedo(i), GetMetallic(i),
                surfaceData.SpecularColor, oneMinusReflectivity
        );
//      surfaceData.SpecularColor = 0;
        return UnityMetaFragment(surfaceData);
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/indirect-light.png)  
*带颜色的间接光*  

间接光现在起作用了，但是自发光还是可能不起作用，因为烘焙器有时候不包括带有自发光的 pass，材质需要表明自己有自发光以参与烘焙过程。这是通过 Material.globalIlluminationFlags 属性完成的，现在让我们表明一下自发光需要烘焙。

```c#
        void DoEmission () {
                …
                if (EditorGUI.EndChangeCheck()) {
                        if (tex != map.textureValue) {
                                SetKeyword("_EMISSION_MAP", map.textureValue);
                        }

                        foreach (Material m in editor.targets) {
                                m.globalIlluminationFlags =
                                        MaterialGlobalIlluminationFlags.BakedEmissive;
                        }
                }
        }
```


## 3.6 粗糙金属

现在我们的着色器看起来正常了，不过好像还跟标准着色器差一点，在使用带颜色的粗糙金属时比较明显。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/metal-standard.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/metal-custom.png)  
*粗糙绿色金属，标准shader vs. 我们的shader* 

我们的想法是，非常粗糙的金属应该比我们目前的计算结果产生更多的间接光。标准着色器把部分高光颜色加到反照率上，以补偿这一点。它使用 UnityStandardBRDF 的 SmoothnessToRoughness 函数来基于平滑度算出粗糙度值，将粗糙度减半并用来缩放高光颜色。

```c
        float roughness = SmoothnessToRoughness(GetSmoothness(i)) * 0.5;
        surfaceData.Albedo += surfaceData.SpecularColor * roughness;

        return UnityMetaFragment(surfaceData);
```

>**SmoothnessToRoughness怎么计算的？**
>计算方法是1减平滑值再平方,从平滑到粗糙的平方映射能产生比线性转换更好的结果。
```c
// Smoothness is the user facing name
// it should be perceptualSmoothness
// but we don't want the user to have to deal with this name
half SmoothnessToRoughness(half smoothness) {
        return (1 - smoothness) * (1 - smoothness);
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/creating-lightmaps/adjusted-albedo.png)  
*调整过的反照率*  


# 4 带方向的光照贴图

烘焙器只考虑了顶点的几何数据，没有考虑法线贴图，光照贴图的分辨率太低，没法包括一般法线贴图的细节内容，所以静态光照只能是平的，当使用了带有法线的材质时尤其明显。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/directional-lightmaps/normal-maps-realtime.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-16/directional-lightmaps/normal-maps-lightmapped.png)  
*法线贴图，实时 vs. 光照贴图*  

当从实时光照切换到烘焙光时，法线效果几乎消失了，你还能看到法线效果只是因为还有环境反射在使用他。


## 4.1 方向性

要使法线和烘焙光一起使用也可以，把 *Directional Mode* 切回 *Directional*。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/directional-lightmaps/directional-mode.png)  
*再次开启带方向的光照贴图*  

当使用带方向的光照贴图时，Unity 会创建两张图而不是一张图。第一张图和平常一样包括了光照信息，称为强度图，第二张图称为方向图，记录了大部分烘焙光射来的方向。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/directional-lightmaps/intensity-directionality.png)  
*强度图和方向图*  

有了方向图之后，我们就可以根据烘焙光做简单的漫反射计算，使法线贴图成为可能。注意，因为只能记录一个光线方向，所以着色计算是近似的，只要在光线强烈时能够存在一个主导的光线方向，结果就会显得还不错。


## 4.2 对方向采样

如果提供了方向图，Unity 会搜索一个同时带有 LIGHTMAP_ON 和 DIRLIGHTMAP_COMBINED 关键字的着色器变体。我们不用自己添加 multi-compile 指令，而是在 forward base pass 里使用 #pragma multi_compile_fwdbase。他会处理所有光照贴图方面的关键字以及 VERTEXLIGHT_ON 关键字。

```c
//                      #pragma multi_compile _ SHADOWS_SCREEN
//                      #pragma multi_compile _ LIGHTMAP_ON VERTEXLIGHT_ON
                        #pragma multi_compile_fwdbase
                        #pragma multi_compile_fog
```

我们也可以在 deferred pass 做同样的事情，不过我们现在使用 #pragma multi_compile_prepassfinal 指令，他会处理所有光照贴图方面的关键字以及HDR关键字。

>**prepass final是什么？**
>Unity 4 使用的延迟渲染技术和之后的版本不一样，在 Unity 5 里改称为传统延迟渲染。这种方法的 pass 比较多，Prepass final是从那时起提出的术语。#pragma multi_compile_prepassfinal 也沿用到现在的延迟渲染中，而没有引入新指令。

```c
//                      #pragma multi_compile _ UNITY_HDR_ON
//                      #pragma multi_compile _ LIGHTMAP_ON
                        #pragma multi_compile_prepassfinal
```


在 CreateIndirectLight 函数里，取得了烘焙光之后，就需要光的方向信息，方向图是通过 unity_LightmapInd 变量提供的。

```c
    #if defined(LIGHTMAP_ON)
        indirectLight.diffuse = DecodeLightmap(UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightmapUV));
                        
        #if defined(DIRLIGHTMAP_COMBINED)
            float4 lightmapDirection = UNITY_SAMPLE_TEX2D(
                        unity_LightmapInd, i.lightmapUV
                    );
        #endif
    #else
        indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
    #endif
```

但是，这样会导致一个编译错误。这是因为一个纹理实际上由两部分组成，一部分是纹理资源，另一部分是采样状态，采样状态决定了纹理是如何被采样的，包括滤波器和限制模式。通常来说，每一张纹理都有这两个部分，但是不是所有平台都如此，这两部分也可以分开，我们可以定义一个采样状态来配合多张纹理。

因为强度图和方向图是用同一种方式采样的，Unity就尽可能让二者共用一个采样状态，这就是为什么我们对强度图进行采样的时候需要使用 UNITY_SAMPLE_TEX2D 宏。方向图没有对应的采样器定义，要对他采样，我们需要使用 UNITY_SAMPLE_TEX2D_SAMPLER 宏，来指定采样器。

```c
    float4 lightmapDirection = UNITY_SAMPLE_TEX2D_SAMPLER(
            unity_LightmapInd, unity_Lightmap, i.lightmapUV
        );
```


## 4.3 使用方向

为了使用方向，我们首先需要对其解码，然后才能和法线计算点积从而求出漫反射系数，进而计算出颜色。但是方向图里存储的并不是真的单位向量，它包含了更多东西，我们可以使用 UnityCG 中的 DecodeDirectionalLightmap 函数来解码方向同时计算漫反射。

```c
    float4 lightmapDirection = UNITY_SAMPLE_TEX2D_SAMPLER(
            unity_LightmapInd, unity_Lightmap, i.lightmapUV
        );
    indirectLight.diffuse = DecodeDirectionalLightmap(
            indirectLight.diffuse, lightmapDirection, i.normal
        );
```


![](https://catlikecoding.com/unity/tutorials/rendering/part-16/directional-lightmaps/using-directional-lightmaps.png)  
*使用带方向的光照贴图*  

>**DecodeDirectionalLightmap做了啥？**
>它实际上并没有正确地计算漫反射系数，而是使用了半兰伯特算法，这种方法在表面周围有效地弯折光线，被照亮的阴影区域超出了应有的范围。这是必需的，因为烘焙的光线实际上不是来自同一方向。
```c
inline half3 DecodeDirectionalLightmap (
        half3 color, fixed4 dirTex, half3 normalWorld
) {
    // In directional (non-specular) mode Enlighten bakes dominant light
        // direction in a way, that using it for half Lambert and then dividing
        // by a "rebalancing coefficient" gives a result close to plain diffuse
        // response lightmaps, but normalmapped.

    // Note that dir is not unit length on purpose. Its length is
        // "directionality", like for the directional specular lightmaps.

    half halfLambert = dot(normalWorld, dirTex.xyz - 0.5) + 0.5;

    return color * halfLambert / max(1e-4h, dirTex.w);
}
```
这里的代码注释提到了高光贴图，这是一种支持高光的光照贴图，但是需要更多的纹理，代价也更高，大部分情况下效果也不佳，所以从Unity5.6开始就移除了。


# 5 光照探针

光照贴图只对静态物体起作用，对动态物体无效，结果就是，场景上的动态物体和烘焙的光照格格不入，尤其是在场景上没有实时光的时候特别明显。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/light-probes/unlit-dynamic-objects.png)  
*动态物体非常明显*  

为了更好的混合静态物体和动态物体，我们必须以某种方式将烘焙的光照应用于动态对象，Unity 为此提供了光照探针。光照探针是空间中的点，带有该位置处的光照信息，它使用球谐函数而非纹理来存储这些信息。一旦有了探针，他们将被用于动态对象，以覆盖全局环境光信息。所以我们要做的就是创建一些探针，等着他们烘焙，然后我们的着色器就会自动使用他们。

## 5.1 创建一组光照探针

通过 *GameObject/Light/Light Probe Group* 向场景添加一组光照探针，这会创建一个新的 GameObject，其中八个探针形成一个立方体，在对动态对象着色时他们将立即起作用。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/light-probes/light-probe-group.png)  
*一组新的光照探针*  

通过 Inspector 面板，您可以在启用 *Edit Light Probes* 模式后编辑光源探针组。启用后，您可以选择单个探针并在场景视图中移动它们，或通过 Inspector 进行调整，也可以操纵，复制和删除单个探测器，如同 GameObject 一样。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/light-probes/inspector.png)  
*光照探针组的 Inspector*  

不必明确启用编辑模式，在Scene视图中选择探针组就可以开始编辑探针，要停止编辑它们，取消选择该组就可以。

## 5.2 放置光照探针

光探针组将其包围的体积分割成四面体，每四个探针定义一个四面体，这些探针之间会进行插值以确定最终用于动态对象的球谐函数，具体取决于对象在四面体内的位置。换句话说，动态对象被视为单个点，因此它仅适用于比较小的对象。

编辑探针时会自动生成四面体，您不需要知道它们的配置，但可视化界面可以帮助您查看探针的相对位置。

放置光照探针是一个麻烦事儿，你得一直调整，直到得到一个可以接受的结果，就像摆弄光照贴图设置一样。首先，标出能囊括所有动态对象的区域。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/light-probes/area.png)  
*囊括整片区域*  

然后根据光照条件的变化添加更多探针，切记不要将它们放在静态物体内，也不要将它们放在不透明的单面几何体的错误一侧。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/light-probes/placing-probes.png)  
*放置更多探针*  

继续添加和移动探针，直到所有区域的光照都是合理的，并且它们之间的过渡是可接受的。

![](https://catlikecoding.com/unity/tutorials/rendering/part-16/light-probes/tweaking-probes.png)  
*调整探针*  

您可以通过移动动态对象来测试探针，选中动态对象时，也会同时显示当前能影响它的探针。探针将显示其光照信息，而非黄色小球。您还可以看到用于动态对象的插值数据。
  
  
下一课是[混合光照](https://catlikecoding.com/unity/tutorials/rendering/part-17/)。

