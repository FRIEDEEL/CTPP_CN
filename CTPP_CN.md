# 写给Python程序员的Cairo教程
## 译序
原文为[Cairo Tutorial for Python Programmers](https://www.tortall.net/mu/wiki/CairoTutorial#cairo-tutorial-for-python-programmers)，原作者为*Michael Urman*。这篇也算是[PyCairo官方网站](https://pycairo.readthedocs.io/en/latest/resources.html)推荐的教程。
翻了一圈好像没有中文翻译，就自己一边读来学，一边翻译来玩玩。

\[by *FRIEDEEl* 231215\]

# 序言：写给Python程序员的Cairo教程
[Cairo](https://cairographics.org/)是一个强大的2D图形库。
本文档将为你介绍cairo如何工作，也会介绍以及许多你可能用得上的函数。

想要跟上本教程，你可能需要在电脑上安装以下几项：
- [Cairo](https://cairographics.org/snapshots/) 本身
- [python](https://www.python.org/downloads/) 用来运行教程中的代码片段
- [PyCairo](https://cairographics.org/snapshots/) 用于讲以上二者结合

相对地，如果你准备好接受挑战了，那么仅需要上面的cairo一项，就可以把这些例子转换到自己更熟悉的语言和本地环境。
*Nis Martensen* 已经贴心地在[C语言环境下](http://cairographics.org/tutorial/) 干了这件事。

> 注意：所有代码示例中都依赖 cairo 1.2.0 或更高版本来支持 `cairo.SVGSurface` 。
> 另外，有几个例子需要 `push_group()` 和 `pop_group()` ，以及辐射梯度 (radial gradients) 要在 1.4 及以上版本才能正确渲染。
> 必要时，第一项要求可以通过将 `cairo.SVGSurface(filename + '.svg', width, height)` 替换为 `cairo.ImageSurface(cairo.FORMAT_ARGB32, width, height)` 来正确渲染。
> 但说真的，还是强烈建议你升级一下。

## 目录
- 序言：写给Python程序员的Cairo教程
  - 目录
- Cairo 的绘图模型
  - 名词
  - 动作

# Cairo的绘图模型
为了解释Cairo接下来的一系列操作，我们先对Cairo绘图的模型一探究竟。
这里只会涉及少数几个概念，它们会在接下来的方法中反复地被应用。
首先我会介绍*名词(nouns)*：
[目标层(destination)]()、
[源(source)]()、
[遮罩(mask)]()、
以及[语境(context)]。 
在这之后介绍*动作(verbs)*，它们会提供操纵这些*名词*的方法，并让你画出你想画的图形。
生成示意图的代码会放在[这里]()，但先别急着去读。

> 如果你觉得这些介绍有些啰嗦，*Don Ingle*做了个把所有东西整合在一起的SVG[一图流示意图]()。
> 这些示意图可能需要你用[Inkscape]()之类的东西查看，并且可能需要2个合适的字体才能正确显示。
> 在学习过程中根据进度放大每页就好。
> 如果你觉得有用，*Don*本人请你务必下载并分享这些示意图。

## 名词(Nouns)
Cairo使用的名词概念都有点抽象。
为了让它们具体一些，这里用了一些示意图来描画这些东西之间是怎么相互作用的。
前三个名词是本章里你将看到的示意图中展示的三个不同的*层(layer)*。
第四个名词，*路径(Path)*，当它被提到的时候会画在示意图的中间层上。
而最后一个名词，*上下文(Context)*，则没有在示意图中展示。

### 目标层(Destination)
> 【译注】这个翻译不怎么准确，但如果用“目标”感觉会和"target"这个词混淆，所以暂译。

*目标层*就是你绘画所在的*表面(Surface)*。
它可能会像在PyGTK教程中的一样，和一些像素数组(pixel array)绑定，
也可能和某个SVG或者PDF文件绑定，也可能是其他东西。
这个表面把图形元素集中在一起，
这样就可以组合出更复杂的作品，就像在canvas\*上作图那样。

![diagram](destination.svg)

> \*【译注】这里canvas似乎是指html的canvas作图而非现实中的画布

### 源(Source)
所谓*源*就是绘图将围绕的“画”本身。
在某些例子会用纯黑色来表示源，但为了能看到下面的其他层，我会把它用一个半透明的层来展示。
和现实的绘画不同，源不一定非得是纯色的，它可以是某种*图案(Pattern)*甚至之前创建的其他表面。
此外还和现实不同的一点是，它可以包含一些关于透明度的信息——Alpha通道。

![diagram](source.svg)

### 遮罩(Mask)
*遮罩*可能是最重要的部分：它控制了你的*源*是如何被*应用\**到*目标层*上的。
在这里我们用黄色带镂空的层将其标出，*源*可以从镂空的地方印到*目标层*上。
当你应用某个绘画动作(drawing verb)的时候，它有点像你从*源*上镂印到*目标层*上。
在遮罩镂空的地方，*源*被复制到*目标层上*；
而在遮罩层阻挡的地方，则什么都不会发生。

![diagram](the-mask.svg)

> *【译注】原文用的是apply，直译水平有限不太好翻译，你可以理解为喷漆涂鸦时候所用的遮罩，或者PS里所用的图层蒙版。

### 路径(Path)
*路径*的概念介于*上下文*的一部分和*遮罩*的一部分之间。
我会用<font color=#00FF00>绿色</font>的线在mask层上将其标出。
路径被路径动作(path verbs)控制，并被绘画动作(drawing verb)所使用。

### 上下文(Context)
每个*上下文(context)* 追踪 *动作(verb)* 影响到的一切。
它会追踪一个source、一个destination和一个mask。
它也会追踪几个辅助性的参数，比如你的线宽和样式，你的字模和尺寸之类的。
最重要的是，它会追踪一个path，而path会被绘画指令(drawing verb)转换成一个mask。