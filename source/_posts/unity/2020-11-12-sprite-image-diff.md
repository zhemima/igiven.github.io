---
title : "Unity中Sprite和UI Image的区别"
---

coffeecato写在前面：
本文确实不错，作者用以说明自动生成网格的示图非常具有代表性，从drawcall的生成过程分析性能开销的重点，引出了overdraw和达到GPU像素填充率限制的原因，从中也可以看出作者对这个主题的理解颇有深度。查看作者的个人自述，居然是个2012年毕业的小伙子，后生可畏啊！翻译本文对自己也是个考验。
英文水平捉急，如果错误请多多指正。

*原文：[UNITY SPRITES: SPRITERENDERER VS. CANVASRENDERER (UI IMAGE)](https://rubentorresbonet.wordpress.com/2016/05/26/unity-sprites-spriterenderer-vs-canvasrenderer-ui-image/)*
翻译已征得原作者同意：
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171116103038856)

当在一个公司项目工作时，我被问到关于sprites(SpriteRenderer)和UI image(CanvasRenderer)的区别。我没找到多少相关的信息，所以我决定在公司准备一场介绍来帮助大家把两者的区别搞清楚。本文中你将会看到一个比当时的介绍更完整的版本。运行环境是Unity5.3.4f1.

Sprites本质上是半透明texture，其中texture是在导入时被设置导入为sprites的。它们不是直接被应用于meshes,而是会应用于长方形/多边形（最后，它们始终是meshes，因为没有那么大的区别）。Sprites就是被渲染到2d/3d场景或者其他界面中的图片.

**1.用法**
在Unity中使用sprites很简单。只需将目标图片移动到assets文件夹下然后点击打开inspector settings.将texture type改为sprite(2D and UI),如下图：
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114233841299)
现在你该决定将图片当成sprite使用还是作为UI Image.但是如果考虑到渲染，你可能并不确定你想要使用哪种方式。我们将会在下一节描述两种方式间的区别；现在我们大概描述一下如何在Unity中创建它们。

如果你想使用SpriteRenderer,将sprite从Project窗口移动到Hierarchy窗口或者Scene窗口。成功创建后的窗口应该像这样：
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114233856654)
如果你想创建UI Image,在Hierarchy中右键然后create new UI–>Image.这个控件需要canvas,如果没有canvas会自动创建一个。最后，你将看到：
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114233911906)

**2.对比：SpriteRenderer vs CanvasRenderer**

在Hierarchy窗口,你可以把sprites随便放在哪。然而，UI Images不得不放在canvas下面。你可以通过transform移动其他obejcts那样移动sprites,但是images使用RectTransform来在界面系统中移动。

使用默认材质时，Sprites是在”Queue” = “Transparent” “RenderType” = “Geometry”(原文： transparent geometry queue)模式下渲染的。UI Images也是在这种模式下渲染的，除非你使用了Overlay 模式渲染（coffeecato补充：Canvas的Render Mode）,这种情况下它将会通过Canvas.RenderOverlay渲染。你可能会猜到，这样在移动设备上的开销很大。我们稍后会讨论到。

sprites和images的一个最重要的区别在于sprites支持网格的自动构造，而UI Image的网格始终由矩形构成。构造网格的原因将会在下一节讲到；我们将会看到它的重要性及它对性能的重大影响。

最后，两种方式都可以通过使用sprite atlases来减少draw calls.

下面的例子将会帮助看到二者之间的区别：
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114233924452)
上图中可以看到，UI Image创造了一个紧密的矩形来包裹sprite,而SpriteRenderer创造了一个能更好匹配将要渲染的sprite的网格。看看另一个例子：
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114233938619)
同样的情况出现这个例子中。但是网格这次看起来更复杂了，为什么呢？Unity尝试去为sprites构造最佳的网格来避免引入太多的多边形。可能有人会说这样的权衡到底是利是弊。

如果我们导入一张拥有孤岛（coffeecato补充：原文是islands）的png，一张图片包含被透明区域分隔开的图形会发生什么情况？
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114233951951)

上图中我们看到的情况很有意思，SpriteRenderer创造了两个子网格，一个对应一个孤岛；然后UI Image只是通过扩展矩形来覆盖整个图片。

**3.性能**

你可能会猜到，上面例子显示的不同处理方式可能会导致性能的差异。当渲染很多对象时，差别会更明显（比如地形中的草，或者粒子特效）。下面我们来分析一下其中的原因。

当渲染texture时，设置好顶点，索引，uv坐标，纹理数据和shader参数，然后向GPU发送数据，这个过程就是著名的draw call.随后，在图片最终显示之前，一些乱七八糟的事情在GPU发生。一个简单的渲染管线通常包括：
1. CPU 发送一个draw命令到GPU
2. GPU获取到绘制所需的所有信息
3. 几何图形通过顶点着色器和光栅化被转化为像素
4. 每个像素通过片元着色器被转化然后被写入到帧缓存一次或数次
5. 当一帧结束时，图形会显示在你的显示器上

回到主题，SpriteRenderer和UI Image之间的区别是什么？显而易见，sprites的开销更大，因为它的几何数据更复杂。但是如果我告诉你顶点操作通常比片元操作开销小的多呢？尤其对于移动设备和半透明对象。

在很多引擎包括Unity中，半透明材质是由后向前渲染的。那意味着，最远的物体（从camera出发）最先渲染，这样alpha混合操作才能像预期那样工作。对于不透明材质，渲染正好反过来这样便于我们剔除不可见物体。

像素着色器会被渲染sprite中的每一个像素都执行，因此，当存在较大的图形时（相对屏幕尺寸），片元着色器将会在很多像素上执行。问题在于，当透明物体在视锥体内时没有很有效的方法将它们剔除，因此你将会渲染所有的半透明物体即使其中的大多数最终都不可见。所以你会发现同一个像素会渲染多次，在帧缓存中也会重写多次。这个问题通常被称为**overdraw**.同样地，由于这种现象带来了**内存带宽**的浪费，会很快达到GPU **像素填充率**的限制，这种情况是移动设备应当极力避免的。这就是问题的关键。

如果你确实理解了上面一段，你将会弄明白SpriteRenderer和CanvasRenderer是多么的不同。前者通过构造网格清除了不必要的透明像素（因此，避免了执行开销巨大的片元着色器，从而避免了overdraw），然而UI Image创建了一个简单的网格很可能会引起很多overdraw。你需要在复杂的几何图形和更多的片操作之间做一个权衡。

你应该会想到使用sprite atlases，因为spritest通常数量很大同时尺寸很小。这会导致绘制sprites有很多drawcall.同样地，对于较大的图形，图形压缩也是不错的方法。
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114234009365)

你可以通过Atlas Packer很方便的创建sprite atlas.同时，有时自动构造的网格性能上并不好你也无法控制它，因此你可能会使用一些性能更好的插件比如ShoeBox 或者 TexturePacker.
![这里写图片描述](../../assets/images/2020-11-12-sprite-image-diff/20171114234059641)

**4.结论**

当下次再遇到sprites时，不妨考虑下面的建议：
1. 如果sprites的数量不多，想用什么用什么。如果拥有上百个sprites，重新读读本文。
2. 使用profiler和frame debugger来搞清楚发生的状况。
3. 避免使用透明，尽量使用不透明的物体替代透明物体。
4. 避免在屏幕上渲染尺寸较大的sprites，这会引起更多的overdraw。你可以通过在Scene View中选择rendering mode为Overdraw来查看overdraw的情况。这对于粒子特效很关键。
5. 选择更复杂的几何体而不是更多的像素，尤其对于移动设备。可以通过选择Scene View中的Shading Mode为shaded wireframe来查看。
6. 如果需要对界面进行较多的位置操作（比如content fitter, vertical groups等）选择UI Images.
7. 减少渲染区域的分辨率来查看性能有没有实质的提升，通过这种方法来判断是否达到了像素填充率的限制。







