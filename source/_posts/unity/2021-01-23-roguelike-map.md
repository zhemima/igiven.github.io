---
title : "roguelike地牢生成算法"
---

#### 引言

文章作者：[Mike Anderson](http://www.roguebasin.com/index.php?title=Mike_Anderson)

随机生成的地图是 [Roguelike](https://indienova.com/tag/roguelike/) 类游戏最独特的一点，它让游戏变得很有乐趣，因为玩家永远要面对新的挑战。

但是随机地图却不是那么容易生成的。在传统的游戏中，一般你都会有一个地图编辑器，可以自由的创建地图。在任何一款称得上是“Roguelike”的游戏中，开发者都要自己创造一个“虚拟地图编辑器”，这样才能随机创建无限的动态地图，从而让玩家在其中流连忘返。

在这篇文章里，我会将自己在开发一款名为 [Tyrant](http://sourceforge.net/projects/tyrant/) 的 Roguelike 游戏中使用的方法记录下来。我怀疑这可能只能算是一个原型，但是我之前也没有见过什么一本正经讲述生成 Roguelike 地图算法的文章。而且，它工作得还是比较令人满意的，所以，我愿意将它分享给大家。

#### 这款算法的目标

在写任何代码之前，了解自己的目标总是很重要的，这对编程很有帮助，哪怕你随后会做无数的修改。

一个地牢（[Dungeon](https://indienova.com/tag/dungeon/)）应该包含以下要点：

- 一组相互连通的房间、门和通道
- 一个入口（向上走的楼梯）
- 一个出口（向下走的楼梯）
- 所有的空间必须能够到达

最后一点尤其重要。要知道，你的玩家在契而不舍的努力之后，应该能够顺利通过这一层，不要让他们失望。另外，如果放了某个物品到地图上的某个空间，它应该不会被藏在无法到达的地方。

#### 计划

在我写 Tyrant 的时候，我尝试了很多种不同的算法来生成地图，这里所讲的是我能做到的最好的一个，也是目前游戏中使用的那个。

我的灵感来自于此：“如果我是地下城的一个居民，那么我该怎么去建设我的地牢呢？”

显然，我并不会将我的地下城建造成一个一个看起来不错的小房间，然后在中间用长长的通道连接起来。所以，当我需要为我的小怪物们提供更多空间的时候，我应该是拿起我的斧头，挖一个更大一些的洞。这样当他们有所需要的时候就会增加一些新房间——尽管它们看起来可能杂乱无章。

有些地下城主可能想要用吊桥呀、陷阱呀什么的来守护比较“有趣”的房间，但是这些需求都异曲同工。由一个小的地牢开始，慢慢向四周扩散，直到整个地牢形成。这就是我们的计划。

#### 算法

在这个算法里面，“元素”代表着某种地图元素，比如：大房间、小房间、通道、圆形竞技场、保险柜等等。

1. 将整个地图填满土
2. 在地图中间挖一个房间出来
3. 选中某一房间（如果有多个的话）的墙壁
4. 确定要修建某种新元素
5. 查看从选中的墙延伸出去是否有足够的空间承载新的元素
6. 如果有的话继续，不然就返回第 3 步
7. 从选中的墙处增加新的元素
8. 返回第 3 步，直到地牢建设完成
9. 在地图的随机点上安排上楼和下楼的楼梯
10. 最后，放进去怪兽和物品

第 1、2 步很简单。只要你创建好地图就可以去做到。我发现，写一个 `fillRect` 指令用来填充一个区域是比较有效的做法。

第 3 步麻烦一些。你不能随意的寻找一个方块区域去添加你的元素，因为规则是要将元素添加到当前的地牢当中。这样会使得连接看起来比较不错，也确保了所有的区域都可以到达。Tyrant 的做法是：在地图上随机选择一个方块，直到找到横向或者纵向毗邻一个干净的方块那个。这样做的好处是：它给了你一个近乎公平的方式去选择某一面墙。

第 4 步不太困难。我自己写了一个随机方法来决定建造哪一种元素。你可以自己定义它们，调整某些元素出现的权重，这会让你的地牢有自己的特点和侧重点。一个规划比较好的地牢会有很多规矩的房间，中间有长而且直的走廊连接。而洞穴则可能有一堆打洞以及曲折的小道等等。

第 5 步更复杂一些，而且也是整个算法的核心。针对每一种元素，你需要知道它会占用的空间大小。然后你要去判断它是否和已经有的元素相交。Tyrant 使用了相对简单的一种方法：它会先得到要创建的元素所占用的空间大小，得到这个空间的数据，然后检查是否这个空间由土填满。

第 6 步决定是否创建这个元素。如果这个待确定的空间包含有除了土之外的内容，那么就回到第 3 步继续。注意，大部分元素在这步都会被打回。不过这不是个问题，因为处理时间可以忽略。Tyrant 尝试着将某个元素加入 300 次左右到地牢中去，一般只有 40 次左右会通过这步。

第 7 步会将新元素添加到地图上去。在这步，你还可以增加一些有趣的元素，比如动物、居民、秘道门和财宝什么的。

第 8 步返回去创建更多的房间。确切的次数跟你地牢的尺寸以及其它参数有关。

第 9 步要看个人喜好了。最简单的方法就是随机的去查找方块，直到找到一个空的位置去放置楼梯。

第 10 步就是随机的创建怪兽。Tyrant 在这一步才加入游戏中大多数的怪兽，由少量的特殊怪兽或者生物会在生成房间的时候添加进去。

就这样啦，这里所说的只是算法的规则，具体还要您自己去实现啦。

#### 例子

好了，在看了半天算法之后，我们来一个例子吧：

Key:

```
# = 地板
D = 门
W = 正在考查中的墙
C = 宝箱
```

\1. 第一个房间

```
#####
#####
#####
```

\2. 随机选择一面墙

```
#####
#####W
#####
```

\3. 为新的通道元素进行区域搜索（包括两边的空间）

```
#####**********
#####W*********
#####**********
```

\4. 是空的，可以添加元素

```
#####
#####D########
#####
```

\5. 选择另外一面墙

```
#####     W
#####D########
#####
```

\6. 扫描寻找新的房间所占用空间：

```
       ******
       ******
       ******
       ******
       ******
#####  ***W**
#####D########
#####
```

\7. 这个地区也可以，那就添加一个新房间，再往里面扔一个宝箱 C（Chest）：

```
        ####
        ###C
        ####
        ####
#####     D  
#####D########
#####
```

\8. 跟前面做法一样，我们增加一个新的通道元素

```
             #
             #
        #### #
        ###C #
        #### #
        #### #
#####     D  #
#####D########
#####
```

\9. 这一次，我们试着为第二个房间增加一个通道元素

```
             #
             #
        #### #
        ###C*******
        ####W******
        ####*******
#####     D  #
#####D########
#####
```

\10. 扫描失败了，已经被占用

```
             #
             #
        #### #
        ###C #
        #### #
        #### #
#####     D  #
#####D########
#####
```

\11. 比较特别的元素，一个菱形的房间

```
             #
             #   ###
        #### #  #####
        ###C # #######
        #### #D#######
        #### # #######
#####     D  #  #####
#####D########   ###
#####
```

\12. 添加一个隐藏的暗门，以及充满陷阱的通道：

```
             #
             #   ###
        #### #  #####
        ###C # #######S###T##TT#T##
        #### #D#######
        #### # #######
#####     D  #  #####
#####D########   ###
#####
```

\13. 继续……

#### 总结

好了，这就是我的算法，我希望它对你有用，或者从一个有趣的角度去看如何解决一个问题。

#### 代码实现

**Java 代码实现**
[Java 代码实现](http://www.roguebasin.com/index.php?title=Java_Example_of_Dungeon-Building_Algorithm)
你可以通过 [Open Processing](http://openprocessing.org/visuals/?visualID=18822) 在浏览器里面运行它（需要做一些小修改）。它会创建一个图形化的地牢。

**Python Curses 代码实现**
[Python Curses 代码实现](http://www.roguebasin.com/index.php?title=Python_Curses_Example_of_Dungeon-Building_Algorithm)

**C++ 代码实现**
[C++ 代码实现](http://www.roguebasin.com/index.php?title=C%2B%2B_Example_of_Dungeon-Building_Algorithm)

**C# 代码实现**
[C# 代码实现](http://www.roguebasin.com/index.php?title=CSharp_Example_of_a_Dungeon-Building_Algorithm)

原文地址：[链接](http://www.roguebasin.com/index.php?title=Dungeon-Building_Algorithm)