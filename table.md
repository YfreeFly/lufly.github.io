以UITableView为例,UICollectionView类似

- 1. 获取当前视图的所有可见cell

```kotlin
open var visibleCells: [UITableViewCell] { get }
```

- 2.获取当前视图中的所有可见cell的IndexPath

```kotlin
open var indexPathsForVisibleRows: [IndexPath]? { get }
```

- 3.根据当前cell的IndexPath获取在tableView的坐标,根据cell的y坐标和tableView的偏移量计算

```swift
open func rectForRow(at indexPath: IndexPath) -> CGRect
```

### 说一下工作中你怎么做性能优化的

```css
答：一般都是说关于tableView的优化处理，

造成tableView卡顿的原因
1.没有使用cell的重用标识符，导致一直创建新的cell
2.cell的重新布局
3.没有提前计算并缓存cell的属性及内容
4.cell中控件的数量过多
5.使用了ClearColor，无背景色，透明度为0
6.更新只使用tableView.reloadData()（如果只是更新某组的话，使用reloadSection进行局部更新）
7.加载网络数据，下载图片，没有使用异步加载，并缓存
8.使用addView 给cell动态添加view
9.没有按需加载cell（cell滚动很快时，只加载范围内的cell）
10.实现无用的代理方法(tableView只遵守两个协议)
11.没有做缓存行高（estimatedHeightForRow不能和HeightForRow里面的layoutIfNeed同时存在，这两者同时存在才会出现“窜动”的bug。
建议是：只要是固定行高就写预估行高来减少行高调用次数提升性能。如果是动态行高就不要写预估方法了，用一个行高的缓存字典来减少代码的调用次数即可）
12.做了多余的绘制工作（在实现drawRect:的时候，它的rect参数就是需要绘制的区域，这个区域之外的不需要进行绘制）
13.没有预渲染图像。（当新的图像出现时，仍然会有短暂的停顿现象。解决的办法就是在bitmap context里先将其画一遍，导出成UIImage对象，然后再绘制到屏幕）

提升tableView的流畅度
*本质上是降低 CPU、GPU 的工作，从这两个大的方面去提升性能。
  1.CPU：对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制
  2.GPU：纹理的渲染

卡顿优化在 CPU 层面
1.尽量用轻量级的对象，比如用不到事件处理的地方，可以考虑使用 CALayer 取代 UIView
2.不要频繁地调用 UIView 的相关属性，比如 frame、bounds、transform 等属性，尽量减少不必要的修改
3.尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性
4.Autolayout 会比直接设置 frame 消耗更多的 CPU 资源
5.图片的 size 最好刚好跟 UIImageView 的 size 保持一致
6.控制一下线程的最大并发数量
7.尽量把耗时的操作放到子线程
8.文本处理（尺寸计算、绘制）
9.图片处理（解码、绘制）

卡顿优化在 GPU层面
1.尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示
2.GPU能处理的最大纹理尺寸是 4096x4096，一旦超过这个尺寸，就会占用 CPU 资源进行处理，所以纹理尽量不要超过这个尺寸
3.尽量减少视图数量和层次
4.减少透明的视图（alpha<1），不透明的就设置 opaque 为 YES
5.尽量避免出现离屏渲染
```

### beginUpdate和endUpdate使用

一般当tableview需要同时执行多个动画时，才会用到beginUpdates函数，它的本质就是建立了`CATransaction这个事务。我们可以通过以下的代码验证这个结论`

```
[CATransaction begin];

[CATransaction setCompletionBlock:^{
    // animation has finished`
}];

[tableView beginUpdates];
// do some work
[tableView endUpdates];

[CATransaction commit];
```

这段代码来自stackoverflow，它的作用就是在tableview的动画结束后，执行需要的操作。这段代码好用的原因就是beginUpdates本质上就是添加了一个动画事务，即CATransaction，当然这个事务可能包含许多操作，比如会重新调整每个cell的高度（但是默认不会重新加载cell）。如果你仅仅更改了UITableView的cell的样式，那么应该试试能否通过调用beginUpdates 和 reloadRowsAtIndexPaths 来实现效果，而不是调用tableview的reloadData方法去重新加载全部的cell！

一个例子，带有动画效果的，重新加载部分cell的代码。

```
[tableView beginUpdates];
[tableView reloadRowsAtIndexPaths:[NSArray arrayWithObject:tmp] withRowAnimation:UITableViewRowAnimationAutomatic];
[tableView endUpdates];
```

下面是一个例子，想实现一个点击cell，cell的高度就变高的效果，就可以在选中方法中设置标志位，来影响代理方法返回的height值，并且在之后调用

```
 [tableView beginUpdates];
  [tableView endUpdates];
```

没错，他们之间没有代码，算然没代码，这2句话会根据代理方法重新调整cell的高度。下面是调用的效果截图：

![061151017595013](/Users/fly/Desktop/Note/iOS/图片/061151017595013.png)
