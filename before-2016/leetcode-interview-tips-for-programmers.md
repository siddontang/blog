## 起因

写在开头，脑袋铁定秀逗了，历时20多天，刷完了leetcode上面151道题目（当然很多是google的），感觉自己对算法和数据结构算是入门了，但仍然还有很多不清楚的地方，于是有了对于每道题目写分析的冲动。不过在看到leetcode上面的文章之后，决定先从翻译入手，顺带再写写自己做题的心得体会。今天是第一篇：程序员面试技巧。

如果你主修计算机科学，那么在你工作的时候会碰到很多有难度的编程问题。当你去找工作的时候，你会有很多的面试，而面试官通常很喜欢问你很多技术性的问题，以下就是三类主要的题型：

+ 算法/数据结构
+ 特定编程语言/通用的计算机知识
+ 脑筋急转弯

后续我将详细的讨论上面这几点：

## 算法/数据结构

你必须深刻的理解以下几种数据结构：数组，链表，二叉树，哈希表等。你必须明确地知道什么时候该使用数组或者链表，譬如在实现一个列表翻转的时候。

面试官通常会问你一些算法问题用以检验你的编程解决问题能力，一些算法问题如下：

+ 在一个句子里面反转单词。
+ 在一个单词里面反转字母。（尝试使用迭代或者递归解决）
+ 如何在一个列表里面找到重复的数字。
+ 生成数字n的全排列。（小提示：递归）
+ 找到两个排好序数组的中间值。（小提示：分治法）
+ 如何进行拼写检查并验证单词。（小提示：编辑距离）

（译者吐槽，说句实话，上面几个问题如果没做题大部分我还真答不出来。）

## 特定编程语言/通用的计算机知识

如果你应聘的是C++职位，需要准备好应对很多C++的问题，其它编程语言也一样。如果你应聘的职位不需要特定的编程语言，那么你可能会被问到通用的计算机知识，譬如在堆和栈上面创建对象的区别。

我经常碰到的一些C++问题如下：

+ 什么是虚函数？怎么实现的？使用虚函数会有啥性能问题？
+ 什么是虚析构函数？为什么需要他们？如果你不实现成虚析构会出现什么问题？
+ 什么是拷贝构造函数？它什么时候被调用？
+ malloc和new的区别是什么？

（译者吐槽，上面这些感觉还算靠谱，至少没问虚拟继承是啥！我面试的时候通常还会问很多STL的东西，毕竟这在C++里面已经是很基础的了。Template这些的就算了，有时候都能把自己绕晕，还是别坑面试者了。）

## 脑筋急转弯

一些面试官很喜欢用一个脑筋急转弯来结束面试。这个其实很坑爹的，不过还是老老实实的准备一些吧。

一些经典的问题：

+ 两个鸡蛋问题：
    
    你有两个相同的鸡蛋，然后还有100层楼等你去爬。你需要知道最高到哪一层扔下鸡蛋，鸡蛋不会摔碎。（鸡蛋语录：为啥受伤的总是蛋蛋？）。很有可能第一层就碎了，也可能到了100层才碎。你需要知道最多尝试几次就能找到答案。
    
+ 尽可能快的在一个byte里面反转bit。

+ 你有5个相同的罐子，都装着球，其中有4个里面每个球都是重10g，另一个灌每个球重9g，你有一台电子称，只称一次，找到那个重9g球的罐子。

（译者吐槽，上面这些脑筋急转弯问题感觉就是算法问题，哪里是脑筋急转弯，明显的忽悠！）

## 总结

当然这是译者自己的，原文在上面就结束了，可以看到，作为一个程序员，我们其实是幸运的，因为我们能够接触如此多的挑战，并不断提升自己。可我们同时也是郁闷的，很多时候往往都有这样的错觉，数据结构和算法在很多工作中并不用到，但是为啥偏偏就面这个，linus貌似说过：“Bad programmers worry about the code. Good programmers worry about data structures and their relationships”，而在《大教堂与集市》这本书里面，作者直接说明“Smart data structures and dumb code works a lot better than the other way around”。有时候当写程序写多了，自然就发现算法和数据结构的重要性了。

另外，现在无论哪个公司，除非你是大牛级别的，面试几乎不会脱离上面那些东西，所以还是老老实实的学习吧，骚年。