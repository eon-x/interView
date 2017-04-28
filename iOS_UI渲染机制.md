## iOS 视图，动画渲染机制探究

iOS 为开发者提供了丰富的 Framework（UIKit，Core Animation，Core Graphic，OpenGL 等等）来满足开发从上层到底层各种各样的需求。

先来一张官方文档里面的图

![](http://7xkym9.com1.z0.glb.clouddn.com/1.png)

可以看出iOS渲染视图的核心是 Core Animation。从底层到上层依此是 GPU->(OpenGL、Core Graphic) -> Core Animation -> UIKit。

在 iOS上，动画和视图的渲染其实是在另外一个进程做的（下面我们叫这个进程 render server），在 iOS 5 以前这个进程叫 SpringBoard，在 iOS 6 之后叫 BackBoard。

##### iOS 上视图或者动画渲染的各个阶段：

###### 在 APP 内部的有4个阶段：

- 布局：在这个阶段，程序设置 View / Layer 的层级信息，设置 layer 的属性，如 frame，background color 等等。

- 创建 backing image：在这个阶段程序会创建 layer 的 backing image，无论是通过 setContents 将一个 image 传給 layer，还是通过 [drawRect:] 或 [drawLayer: inContext:] 来画出来的。所以 [drawRect:] 等函数是在这个阶段被调用的。

- 准备：在这个阶段，Core Animation 框架准备要渲染的 layer 的各种属性数据，以及要做的动画的参数，准备传递給 render server。同时在这个阶段也会解压要渲染的 image。（除了用 imageNamed：方法从 bundle 加载的 image 会立刻解压之外，其他的比如直接从硬盘读入，或者从网络上下载的 image 不会立刻解压，只有在真正要渲染的时候才会解压）。

- 提交：在这个阶段，Core Animation 打包 layer 的信息以及需要做的动画的参数，通过 IPC（inter-Process Communication）传递給 render server。

###### 在 APP 外部的2个阶段：

当这些数据到达 render server 后，会被反序列化成 render tree。然后 render server 会做下面的两件事：

- 根据 layer 的各种属性（如果是动画的，会计算动画 layer 的属性的中间值），用  OpenGL  准备渲染。

- 渲染这些可视的 layer 到屏幕。

如果做动画的话，最后的两个步骤会一直重复知道动画结束。

我们都知道 iOS 设备的屏幕刷新频率是 60HZ。如果上面的这些步骤在一个刷新周期之内无法做完（1/60s），就会造成掉帧。

##### 哪些操作可能会过度消耗 CPU 或者 GPU，从而造成掉帧。

- 视图上有太多的 layer 或者几何形状：如果视图的层级结构太复杂的话，当某些视图被渲染或者 frame 被修改的话，CPU 会花比较多得时间去重新计算 frame。尤其如果用 autolayout 的话，会更消耗 CPU。同时过多的几何结构会大大增多需要渲染的 OpenGL
	triangles 以及栅格化的操作（将 OpenGL 的 triangles 转化成像素）

- 太多的 overdraw：overdraw 是指一个像素点被多次地用颜色填充。这个主要是由于一些半透明的 layer 相互重叠造成的。GPU 的 fill-rate（用颜色填充像素的速率）是有限的。如果 overdraw 太多的话，势必会降低 GPU 的性能。

- 视图的延后载入：iOS 只有在展示 viewcontroller 的 view 或者访问 viewcontroller 的 view，比如说 someviewcontroller.view 的时候才会加载view。如果在用户点击了某个 button，并且在 button 的响应函数里做了很多消耗 cpu 的工作，这个时候如果 present 某个 viewcontroller 的话，会容易卡顿，尤其是如果viewcontroller 要从 database 里获取数据，或者从 nib 文件初始化 view 或者加载图片会更卡。

- 离屏的绘制：离屏的绘制有两种情况：1. 有些效果（如 rounded corners，layer masks，drop shadows 和 layer rasterization）不能直接的绘制到屏幕上，必须先绘制到一个 offscreen 的 image context 上，这种操作会引入额外的内存和 CPU 消耗。2. 实现了 drawRect 或者 drawLayer:inContext:，为了支持任意的绘制，core graphic 会创建一个大小跟要画的 view 一样的 backing image。并且当画完的以后要传输到 render server 上渲染。所以没事不要重载 drawRect 等函数却什么都不做。

- 图片解压：用 imageNamed：从 bundle 里加载会立马解压。一般的情况是在赋值给 UIImageView 的 image 或者 layer 的 contents 或者画到一个 core graphic context 里才会解压。

##### 渲染性能优化的注意点：

- 隐藏的绘制：catextlayer 和 uilabel 都是将 text 画入 backing image 的。如果改了一个包含 text 的 view 的 frame 的话，text 会被重新绘制。

- Rasterize：当使用 layer 的 shouldRasterize 的时候（记得设置适当的 laye r的 rasterizationScale），layer 会被强制绘制到一个 offscreen image 上，并且会被缓存起来。这种方法可以用来缓存绘制耗时（比如有比较绚的效果）但是不经常改的 layer，如果 layer 经常变，就不适合用。

- 离屏绘制： 使用 Rounded corner， layer masks， drop shadows 的效果可以使用 stretchable images。比如实现 rounded corner，可以将一个圆形的图片赋值于 layer 的 content 的属性。并且设置好 contentsCenter 和 contentScale 属性。

- Blending and Overdraw ：如果一个 layer 被另一个 layer 完全遮盖，GPU 会做优化不渲染被遮盖的 layer，但是计算一个 layer 是否被另一个 layer 完全遮盖是很耗 cpu 的。将几个半透明的 layer 的 color 融合在一起也是很消耗的。

我们要做的：

1.设置 view 的 backgroundColor 为一个固定的，不透明的 color。

2.如果一个 view 是不透明的，设置 opaque 属性为 YES。（直接告诉程序这个是不透明的，而不是让程序去计算）

这样会减少 blending 和 overdraw。

如果使用 image 的话，尽量避免设置 image 的 alpha 为透明的，如果一些效果需要几个图片融合而成，就让设计用一张图画好，不要让程序在运行的时候去动态的融合。

好了，介绍完这些渲染优化需要注意的点，让我们用 instrument 的 Core Animation 和 GPU Driver 来看看一些具体的例子。

Core Animation：

Color Blended Layers：看半透明 layer 的遮盖情况。从绿到红，越红遮盖越大。不是说半透明的 layer 很多，范围很大就要优化，要参看 GPU Driver 的测量情况看，下面会介绍；

Color Hits Green and Misses Red：当使用shouldRasterize的时候，layer drawing 会被缓存起来，如果 rasterized 的 layer 需要被重新绘制，会标示红。

Color Offscreen-Rendered Yellow：如果要做 offscreen drawing 的话，会标成黄色。

Core Animation Template 只是能让开发者直观地看到哪些地方有可能需要优化，但是到底要不要优化，还是要看 GPU Driver 的表现。

GPU Driver

Renderer Utilization ——如果这个值大于50%的话，表示 GPU 的性能受到 fill-rate 的限制，可能有太多的 Offscreen rendering，overdraw，blending。

Tiler Utilization ——如果这个值大于50%，表示可能有太多的 layers。

Bugly是腾讯内部产品质量监控平台的外发版本，支持iOS和Android两大主流平台,其主要功能是App发布以后，对用户侧发生的crash以及卡顿现象进行监控并上报，让开发同学可以第一时间了解到app的质量情况，及时修改。目前腾讯内部所有的产品，均在使用其进行线上产品的崩溃监控。

http://www.tuicool.com/articles/nyyA32m


