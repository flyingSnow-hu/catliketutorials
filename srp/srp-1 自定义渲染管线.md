SRP之一 控制渲染
======

[原教程传送门](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/)
---

创建一个渲染管道资产和实例。
渲染摄像机的视图。
执行剔除、过滤和排序。
分离不透明、透明和无效的通道。
使用多台摄像机。
这是关于创建自定义可脚本渲染管道的系列教程的第一部分。它涵盖了一个基础渲染管道的初始创建内容，我们会在后面的教程扩展这个管道。

本系列假设你至少已经学习了对象管理系列和程序化网格教程。

本教程是用Unity 2019.2.6f1制作的。

> **其他SRP系列呢？**
> 我有另一个涵盖可脚本渲染管道(SRP)的教程系列，但那个系列使用实验性的SRP API，它只适用于Unity 2018。这个系列是针对Unity 2019及以后的版本。这个系列采用了不同的、更现代的方法，但会涵盖很多相同的主题。如果你等不到这个系列赶上2018年的系列，那么2018年的系列仍然是有用的。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/tutorial-image.jpg)  
*以一个自定义的渲染管线渲染*


# 1 一个新的渲染管线

要渲染任何东西，Unity必须确定哪些形状需要绘制，在哪里绘制，何时绘制，以及用什么设置绘制。这可能会非常复杂，取决于有多少效果会被涉及：光线、阴影、透明度、图像效果、体积效果等等，都必须按照正确的顺序来处理，才能得到最终的图像。这就是渲染管道的作用。

在过去，Unity只支持一些内置的渲染方式。Unity 2018引入了可脚本的渲染管道（Scriptable Render Pipelines，简称RP），使我们可以随心所欲地做任何事情，同时还能依靠Unity来完成基本的步骤，比如裁剪。Unity 2018还增加了两个用这种新方法制作的实验性RP：Lightweight RP和High Definition RP。在Unity 2019中，LRP不再是实验性的，在Unity 2019.3中被重新命名为通用RP。

通用RP注定要取代目前的传统RP，成为默认的RP。它的设计思想是一个通用的RP，但是相当容易定制。本系列将从头开始创建一个完整的RP，而不是定制该RP。

本教程用一个最小的RP奠定了基础，这个RP使用前向渲染绘制无光照的形状。完成这部分之后，我们可以在以后的教程中扩展管道，添加照明、阴影、不同的渲染方法和更高级的功能。


## 1.1 项目设置

在Unity 2019.2.6或更高版本中创建一个新的3D项目。我们将创建我们自己的管线，所以不要选择已有的RP项目模板。打开项目后，你可以进入包管理器，删除所有你不需要的包。我们在本教程中只用Unity UI包来实验绘制UI，所以这个包你可以保留。

我们专门在线性颜色空间中工作，但Unity 2019.2仍然以伽马空间作为默认值。通过 "Edit/Project Settings "进入播放器设置，然后选择“Player”，然后将 "Other Settings"部分下的 "Color Space"切换为 "Linear"。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/color-space.png)  
*设置线性颜色空间*

在默认场景中填充一些物体，使用标准的、没有光照的、不透明和透明的混合材质。Unlit/Transparent着色器需要一张贴图，所以这里使用了一张球形UV贴图。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/sphere-alpha-map.png)  
*带透明度的球形UV贴图，黑色背景。*

我在测试场景中放了几个立方体，它们都是不透明的。红色的材质使用的是 Standard 着色器，而绿色和黄色的使用 Unlit/Color 。蓝色的球体使用 Standard 着色器，Rendering Mode 设置为 Transparent，而白色的球体则使用 Unlit/Transparent 着色器。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/scene.png)  
*测试场景*


## 1.2 管线资产

目前，Unity使用的是默认的渲染管线。要用自定义的渲染管线来代替它，我们首先要为它创建一个资产类。我们将使用与 Unity 使用的通用 RP 大致相同的文件夹结构。创建一个 Custom RP 资产文件夹，其中有一个 Runtime 子文件夹。在那里为 CustomRenderPipelineAsset 类型新建一个C#脚本。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/folder-structure.png)  
*文件夹结构*

资产类必须继承  UnityEngine.Rendering 名字空间下的 RenderPipelineAsset 类。

```C#
using UnityEngine;
using UnityEngine.Rendering;

public class CustomRenderPipelineAsset : RenderPipelineAsset {}
```

RP 资产的主要目的是让 Unity 有办法连接负责渲染的管线对象实例。资产本身只是一个句柄和一个存储设置的地方。我们还没有任何设置，所以我们要做的就是给 Unity 提供一个获取管线对象实例的方法。这是通过覆盖抽象的 CreatePipeline 方法来完成的，该方法应该返回一个 RenderPipeline 实例。但我们还没有定义一个自定义的 RP 类型，所以先返回 null。

CreatePipeline 方法是用 protected 访问修饰符定义的，这意味着只有定义该方法的类——也就是 RenderPipelineAsset ——和那些扩展该方法的类才能访问它。

```C#
	protected override RenderPipeline CreatePipeline () {
		return null;
	}
```

现在我们需要在项目中添加一个这种类型的资产。要做到这一点，请为 CustomRenderPipelineAsset 添加一个 CreateAssetMenu 属性。

```C#
[CreateAssetMenu]
public class CustomRenderPipelineAsset : RenderPipelineAsset { … }
```

这样 *Asset/Create* 菜单中就增加了一个条目。让我们搞整洁一点：把它放在 *Rendering* 子菜单中。我们将属性的 menuName 属性设置为 *Rendering/Custom Render Pipeline*。这个属性可以直接设置在其类型之后的圆括号内。

```C#
[CreateAssetMenu(menuName = "Rendering/Custom Render Pipeline")]
public class CustomRenderPipelineAsset : RenderPipelineAsset { … }
```

使用新菜单项将资产添加到项目中，然后找到 *Project Settings/Graphics*，并在 *Scriptable Render Pipeline Settings* 下选择它。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/srp-settings.png)  
*选择了自定义RP*

取代默认 RP 之后一些事情改变了。首先，图形设置中的很多选项都消失了，这些在一个信息面板中会提到。其次，我们已经禁用了默认的RP，而没有提供有效的替代物，所以不会有任何东西被渲染。游戏窗口、场景窗口和材质预览都不再有功能。如果你打开帧调试器—— Window/Analysis/Frame Debugger——并启用它，你会看到游戏窗口中确实没有绘制任何东西。

## 1.3 渲染管线实例

创建一个 CustomRenderPipeline 类，并将其脚本文件放在与 CustomRenderPipelineAsset 同一文件夹中。这是我们的资产返回的RP实例所使用的类型，因此它必须继承 RenderPipeline。

```C#
using UnityEngine;
using UnityEngine.Rendering;

public class CustomRenderPipeline : RenderPipeline {}
```

RenderPipeline 定义了一个 protected abstract Render 方法，我们必须覆盖它，以创建一个具体的管线。它有两个参数：一个 ScriptableRenderContext 和一个 Camera 数组。现在暂时将该方法留空。

```C#
	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {}
```

让 CustomRenderPipelineAsset.CreatePipeline 返回一个新的 CustomRenderPipeline 实例。这将使我们得到一个合法且有功能的管线，尽管它还不能渲染任何东西。

```C#
	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline();
	}
```


# 2 渲染

每一帧，Unity 都会在 RP 实例上调用 Render。它传递了一个上下文结构，该结构提供了与本地引擎的连接，我们可以使用它进行渲染。它还传递了一个摄像机数组，因为场景中可能有多个活动摄像机。RP 的责任是按照提供的顺序渲染所有这些摄像机。

## 2.1 CameraRenderer

每个摄像机都是独立渲染的。因此，与其让CustomRenderPipeline渲染所有的摄像机，不如将这一责任转交给一个新的类，专门用于渲染一个摄像机。将其命名为CameraRenderer，并给它一个带有上下文和摄像机参数的公共渲染方法。为了方便起见，我们将这些参数存储在字段中。

```C#
using UnityEngine;
using UnityEngine.Rendering;

public class CameraRenderer {

	ScriptableRenderContext context;

	Camera camera;

	public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;
	}
}
```

当 CustomRenderPipeline 创建时，让它创建一个渲染器的实例，然后用它来循环渲染所有摄像机。

```C#
	CameraRenderer renderer = new CameraRenderer();

	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(context, camera);
		}
	}
```

我们的相机渲染器大致相当于通用 RP 的可编程渲染器。这种方法将来会使支持每个摄像机的不同渲染方法变得简单：例如一个用于第一人称视角，一个用于3D地图叠加，或者前向与延迟渲染。但目前我们会用同样的方式来渲染所有的摄像机。

## 2.2 绘制天空盒

CameraRenderer.Render 的工作是绘制其相机能看到的所有几何体。为了清晰起见，我们将这个特定的任务隔离在一个单独的 DrawVisibleGeometry 方法中。我们首先让它绘制默认的天空盒，这可以通过调用上下文中的 DrawSkybox 作为参数来完成。

```C#
	public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;

		DrawVisibleGeometry();
	}

	void DrawVisibleGeometry () {
		context.DrawSkybox(camera);
	}
```

这样还不能显示天空盒。这是因为我们向上下文发出的命令是有缓冲的。我们必须通过调用上下文的 Submit 方法，把工作提交到队列，以便执行。让我们在 DrawVisibleGeometry 之后调用一次 Submit 方法。

```C#
	public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;

		DrawVisibleGeometry();
		Submit();
	}

	void Submit () {
		context.Submit();
	}
```

天空盒终于出现在游戏和场景窗口中。当你启用它时，你还可以在帧调试器中看到它的一个条目。它被列为 Camera.RenderSkybox。它的下面有一个单一的 Draw Mesh 项，代表了实际的绘制调用。这对应的是游戏窗口的渲染。帧调试器不会报告其他窗口的绘制情况。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox.png)  
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox-debugger.png)  
*绘制了天空盒*

请注意，目前摄像机的方向并不会影响天空盒的渲染方式。我们将摄像机传递给 DrawSkybox，只是用来读取相机的清除标志，以决定是否应该绘制天空盒。

为了正确地渲染天空盒和整个场景，我们必须设置视图-投影矩阵。这个变换矩阵将摄像机的位置和方向（视图矩阵）与摄像机的透视或正射投影（投影矩阵）结合起来。它在着色器中被称为 unity_MatrixVP，是几何体绘制时使用的着色器属性之一。当选择绘制调用时，你可以在帧调试器的 ShaderProperties 部分检查这个矩阵。

目前，unity_MatrixVP 矩阵始终是相同的。我们必须通过 SetupCameraProperties 方法，将摄像机的属性应用到上下文中。这将设置矩阵以及其他一些属性。在调用 DrawVisibleGeometry 之前，请在一个单独的 Setup 方法中进行。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox-aligned.png)  
*方向正确的天空盒*

## 2.3 指令缓冲

上下文将实际的渲染延迟，直到我们发出提交指令。在此之前，我们可以对它进行配置，为它添加命令，方便以后执行。有些任务（比如绘制天空盒）可以通过一个专门的方法发出，但其他命令必须通过一个单独的命令缓冲区间接发出。我们需要这样一个缓冲区来绘制场景中的其他几何体。

为了得到一个缓冲区，我们必须创建一个新的 CommandBuffer 对象实例。我们只需要一个缓冲区，所以默认为 CameraRenderer 创建一个缓冲区，并用一个字段存储其引用。同时给缓冲区起一个名字，这样我们就可以在帧调试器中识别它。 “Render Camera” 就蛮好。

```C#
	const string bufferName = "Render Camera";

	CommandBuffer buffer = new CommandBuffer {
		name = bufferName
	};
```

> **那实例初始化器的语法是怎么用的呢？**  
> 就像我们在调用构造函数后，把 ```buffer.name = bufferName;``` 写成了一条单独的语句。但是在创建一个新对象的时候，你可以在构造函数的调用中附加一个代码块。然后在代码块中设置对象的字段和属性，而不必显式引用对象实例。它明确了只有在设置了这些字段和属性之后才能使用实例。除此以外，它使得只使用单条语句初始化成为可能（例如我们在这里使用的字段初始化）而不需要具有许多参数变体的构造函数。
>
> 请注意，我们省略了调用构造函数时的空参数列表，这在使用对象初始化器语法时是允许的。

我们可以使用命令缓冲区来注入 profiler 采样，这些采样将同时显示在 profiler 和帧调试器中。其方法是在适当的时间点调用 BeginSample 和 EndSample。在我们的例子中，就是在 Setup 和 Submit 的开头。这两个方法必须提供相同的采样名字，这里我们将使用缓冲区的名字。

```C#
	void Setup () {
		buffer.BeginSample(bufferName);
		context.SetupCameraProperties(camera);
	}

	void Submit () {
		buffer.EndSample(bufferName);
		context.Submit();
	}
```

要执行缓冲区，请在上下文中调用 ExecuteCommandBuffer，并将缓冲区作为参数。这会从缓冲区中复制命令，但并不清除命令。如果我们想复用它，就必须在事后明确地调用清除。因为执行和清除总是一起进行的，所以添加一个同时进行这两项工作的方法是比较方便的。

```C#
	void Setup () {
		buffer.BeginSample(bufferName);
		ExecuteBuffer();
		context.SetupCameraProperties(camera);
	}

	void Submit () {
		buffer.EndSample(bufferName);
		ExecuteBuffer();
		context.Submit();
	}

	void ExecuteBuffer () {
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
```

Camera.RenderSkyBox 采样现在内含于 Render Camera 了。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/render-camera-sample.png)  
*Render Camera 采样*

## 2.4 清理渲染目标

无论我们绘制什么，最终都会渲染到摄像机的渲染目标上。默认是帧缓冲区，但也可能是渲染纹理。无论之前绘制到该目标上的是什么，都还在那里，这可能会干扰我们现在正在渲染的图像。为了保证正确的渲染，我们必须清除渲染目标，以摆脱它的旧内容。这要通过调用命令缓冲区上的 ClearRenderTarget 来完成，它属于Setup 方法。

CommandBuffer.ClearRenderTarget 至少需要三个参数。前两个参数表示是否清除深度和颜色数据，这两个参数我们都设为 true。第三个参数是用来清除的颜色，这个参数我们使用 Color.clear。

```C#
	void Setup () {
		buffer.BeginSample(bufferName);
		buffer.ClearRenderTarget(true, true, Color.clear);
		ExecuteBuffer();
		context.SetupCameraProperties(camera);
	}
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-nested-sample.png)  
*清除，附带嵌套采样*

帧调试器现在为清除动作显示了一个 Draw GL 条目，该条目嵌套在 Render Camera 的一个额外级别中。之所以会出现这种情况，是因为 ClearRenderTarget 以命令缓冲区的名字包装其采样。我们可以在开始我们自己的采样之前调用清除，以摆脱多余的嵌套。这样就会产生两个相邻的 Render Camera 采样域，然后这两个域会被合并。

```C#
	void Setup () {
		buffer.ClearRenderTarget(true, true, Color.clear);
		buffer.BeginSample(bufferName);
		//buffer.ClearRenderTarget(true, true, Color.clear);
		ExecuteBuffer();
		context.SetupCameraProperties(camera);
	}
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-one-sample.png)  
*清除，没有嵌套采样*

 *Draw GL* 条目代表用 Hidden/InternalClear 着色器绘制了一个全屏四边形，该着色器会写入渲染目标，但这并不是最有效的清除方式。之所以使用这种方法，是因为我们是在设置相机属性之前进行清除。如果我们把这两步的顺序互换一下，就可以得到一种高效的清除方式。

 ```C#
 	void Setup () {
		context.SetupCameraProperties(camera);
		buffer.ClearRenderTarget(true, true, Color.clear);
		buffer.BeginSample(bufferName);
		ExecuteBuffer();
		//context.SetupCameraProperties(camera);
	}
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-correct.png)  
*正确的清除*

现在我们看到 *Clear (color+Z+stencil)*，这表示颜色和深度缓冲区都被清除了。Z代表深度缓冲区，模板数据也是同一缓冲区的一部分。

## 2.5 剔除

我们目前看到了天空盒，但没有看到我们放在场景中的物体。我们不会绘制每一个对象，而只会渲染那些摄像机可见的对象。我们的做法是搜索场景中所有带有渲染器组件的对象，剔除那些不在摄像机视野范围内的对象。

要想知道哪些对象可以被剔除，我们需要跟踪多个摄像机设置和矩阵，为此我们可以使用 ScriptableCullingParameters 结构。我们可以在摄像机上调用 TryGetCullingParameters，而不是自己填充它。它返回参数是否能被成功检索，因为对于不正常的摄像机设置，它可能会失败。为了获得参数数据，我们必须将其作为输出参数，在其前面加一个 out。添加一个单独的Cull方法，其返回值表示成功或失败。

```C#
	bool Cull () {
		ScriptableCullingParameters p
		if (camera.TryGetCullingParameters(out p)) {
			return true;
		}
		return false;
	}
```

> **out 是做甚么的？**（翻译略）

变量作为输出参数时，可以将声明内嵌在参数列表里面，我们就这样做。

```C#
	bool Cull () {
		//ScriptableCullingParameters p
		if (camera.TryGetCullingParameters(out ScriptableCullingParameters p)) {
			return true;
		}
		return false;
	}
```

在 Render 方法里，Setup 之前，调用 Cull，如果失败则放弃。

```C#
	public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;

		if (!Cull()) {
			return;
		}

		Setup();
		DrawVisibleGeometry();
		Submit();
	}
```

实际的剔除工作是通过调用上下文的 Cull 方法来完成的，这个方法会返回一个 CullingResults 结构。所以如果剔除成功的话，就在我们的 Cull 中调用上下文的 Cull，并将结果存储在一个字段中。在这种情况下，我们必须将 culling 作为一个引用参数传递，在它前面写上ref。

> **ref 又是做甚么的？**（翻译略）

## 2.6 绘制几何图形

一旦我们知道哪些物体是可见的，我们就可以继续渲染它们。这是通过调用上下文的 DrawRenderers 方法来完成的，第一个参数是剔除结果，以告诉它要使用哪些 renderer。除此之外，我们还要提供绘制设置和过滤设置。这两个都是结构——DrawingSettings和FilteringSettings——我们目前使用它们的默认构造函数。这两个结构都必须通过引用来传递。调用的时机是 DrawVisibleGeometry 中，绘制天空盒之前。

```C#
	void DrawVisibleGeometry () {
		var drawingSettings = new DrawingSettings();
		var filteringSettings = new FilteringSettings();

		context.DrawRenderers(
			cullingResults, ref drawingSettings, ref filteringSettings
		);

		context.DrawSkybox(camera);
	}
```

我们还是没有看到任何东西，因为我们还需要指明允许哪种着色器通道。由于我们在本课中只打算支持无光照的着色器，所以我们需要为 SRPDefaultUnlit pass 获取着色器标签ID。我们可以只做一次，并将其缓存在一个静态字段中。

```C#
static ShaderTagId unlitShaderTagId = new ShaderTagId("SRPDefaultUnlit");
```

将它作为 DrawingSettings 构造函数的第一个参数，同时提供一个新的 SortingSettings 结构体。将摄像机传递给 SortingSettings 的构造函数，用于确定是否适用于正交或基于距离的排序。

```C#
	void DrawVisibleGeometry () {
		var sortingSettings = new SortingSettings(camera);
		var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		);
		…
	}
```

除此之外，我们还必须指明哪些渲染队列是允许的。将 RenderQueueRange.all 传给 FilteringSettings 的构造函数，这样我们就把所有东西都包含进去了。

```C#
var filteringSettings = new FilteringSettings(RenderQueueRange.all);
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/drawing-unlit.png)  
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/drawing-unlit-debugger.png)  
*绘制无光照物体*

只有使用无光照着色器的可见对象才会被绘制。所有的绘制调用都在帧调试器中列出，归入 RenderLoop.Draw 下。有一些奇怪的事情发生在透明对象身上，不过我们首先关注一下对象的绘制顺序。这是由帧调试器显示的，你可以通过单选或者方向键来逐步查看绘制调用。

[视频：单步跟踪帧调试器](https://gfycat.com/bothanyindianskimmer)

绘制顺序是杂乱无章的。但我们可以通过设置 SortingSettings 的 candards 属性来强制规定特定的绘制顺序。我们使用 SortingCriteria.CommonOpaque。

[视频](https://gfycat.com/rawimpishalpinegoat)  
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/sorting.png)  
*通用的透明度排序*

物体现在可以大体上从前向后绘制，这对于不透明的物体来说非常理想。如果某个东西最终被绘制在其他东西后面，其隐藏的片段可以被跳过，从而加快渲染速度。CommonOpaque 选项还考虑了一些其他原则，包括渲染队列和材质。

## 2.7 分开绘制不透明和透明的几何体

帧调试器向我们展示了透明对象已被绘制，但天空盒会覆盖在所有不存在不透明对象的片段上。因为天空盒在所有不透明几何体之后绘制，因此它的所有隐藏片段都会被跳过，但会覆盖透明几何体。这是因为透明着色器不会写入深度缓冲区。它们不会隐藏它们之后的任何东西，因为我们可以看穿它们。解决的办法是先绘制不透明的物体，然后是天空盒，然后才是透明物体。

我们可以切换到 RenderQueueRange.opaque ，以排除初始 DrawRenderers 调用中的透明对象。

```C#
var filteringSettings = new FilteringSettings(RenderQueueRange.opaque);
```

然后在绘制完天空盒后再次调用 DrawRenderers。但在此之前将渲染队列范围改为 RenderQueueRange.transparent 。同时将排序标准改为 SortingCriteria.CommonTransparent，以再次设置绘制设置的排序。这样就把透明对象的绘制顺序反过来了。

```C#
	context.DrawSkybox(camera);

	sortingSettings.criteria = SortingCriteria.CommonTransparent;
	drawingSettings.sortingSettings = sortingSettings;
	filteringSettings.renderQueueRange = RenderQueueRange.transparent;

	context.DrawRenderers(
		cullingResults, ref drawingSettings, ref filteringSettings
	);
```

[视频](https://gfycat.com/disloyalremarkableavocet)  
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/opaque-skybox-transparent.png)  
*不透明物体，然后是天空盒，然后是透明物体*

> **为什么绘制顺序要反过来？**
> 由于透明对象不会写入深度缓冲区，因此对它们进行从前到后排序对性能没有好处。但是，当透明对象最终在视觉上有交叠时，它们必须从后到前地绘制才能正确混合。
>
> 不幸的是，从后到前排序并不能保证混合正确，因为排序是针对每个对象的，而且只基于对象本身的位置。相交的和大的透明对象仍然会产生不正确的结果。有时可以通过将几何体切割成较小的部分来解决这个问题。


# 3 编辑器中的渲染
