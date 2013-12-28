# 起因

一年前接触texture packer的时候，就一直有做一个类似工具的想法，当时想了一下，可以从以下几个方面入手：

- 界面与功能分离，也就是提供命令行功能，界面直接通过命令行调用实现相关功能。界面可以考虑使用qt或者wxwidget。
- 实现一个高效的packer算法
- 图像处理问题，读入不同格式的文件，输出设定格式的文件，这个可以使用GraphicsMagick来处理。

那么首要实现的其实就是一个好的packer算法。而对于图像处理，以及界面展示，这些我觉得都可以在后期考虑。

虽然当时很有想做一个的冲动，但因为很多原因，停滞了下来。不过心里总有一个坎，直到现在又重新接触了cocos2d-x，立马觉得要实现这个packer了。

# bin packing problem

我们需要实现的功能就是把一批小的texture合并到一张大的texture上面，而这个就是bin packing problem需要解决的问题。

参考了很多资料，譬如[这个](http://codeincomplete.com/posts/2011/5/7/bin_packing/)，[这个](http://www.codeproject.com/Articles/210979/Fast-optimizing-rectangle-packing-algorithm-for-bu)，还有[这个](http://www.blackpawn.com/texts/lightmaps/default.html)。

理解了基本算法之后，自己也实现了很简单的[packer](https://gist.github.com/4168873)。大概原理也很简单:

- 将所有texture从大到小排序，可以按照maxsize，area，width，height等。
- 取出第一个texture，初始化root bin的大小为该texture大小。
- 取出第二个texutre，通过root查找是否有bin能将其放下，如果能放下，则放入，并将该bin切分。
- 如果没有可以放下的bin，则扩容root bin，并将原先root bin作为扩大bin之后的子节点，扩容bin设为root bin
- 重复执行上述3，4，直到所有的texture都放入

具体流程可以参考代码。

# 总结

总的来说，实现一个简单的bin packer是比较简单的，当然还有很多复杂的实现，只是对于一般应用来说，足够了。这里还需要赞一下python，快速开发原型实在是太方便了。