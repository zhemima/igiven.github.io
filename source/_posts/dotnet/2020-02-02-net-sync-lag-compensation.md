---
title:  "快节奏多人游戏同步(4)-延时补偿"
---

## PART 1 概述

之前三篇文章主要解释了关于 client-server 游戏架构，总结起来大概就是以下这些：

- 服务器从客户端收到带有时间戳的输入信息；
- 服务器处理输入并且更新世界状态；
- 服务器向所有客户端发送游戏世界的快照
- 客户端发送输入并且模拟游戏的结果；
- 客户端获取世界更新
  - 将自身预测的状态和服务器发送来的状态进行同步；
  - 将其他客户端控制的实体插值到过去的状态

从玩家的角度来看，以上行为会导致两个重要的结果：

- 玩家看到 **自己** 处于 **现在**
- 玩家看到 **其他玩家** 处于 **过去**

这其实并没有什么大不了的问题，但是对于时间和空间非常敏感的事件就会造成很大的问题；比如在射击游戏中爆掉敌人的头！

## PART 2 延时补偿

假设你正用狙击枪完美的瞄准目标的头部，此时射击绝对万无一失。

然而却没打到。。。

为什么会发生这种事情。。

因为我们之前解释过的 client-server 架构，你瞄准的是 100ms 之前的玩家的头，而不是开枪的时候的玩家的头。。。

在某种程度上相当于你在一个光速非常非常慢的宇宙中进行游戏，你瞄准的是敌人过去的位置，当你扣下扳机的时候他早就走远了。。

比较幸运的是有一个相对简单的解决方案，对几乎所有的玩家都是友好的，下面来解释一下它的工作流程：

- 开火的时候，客户端发送开火指令到服务器，同时包含开火的一瞬间确切的时间和方向。
- 这是关键的一步。由于服务器获取所有带有时间戳的输入，因此它可以在过去的任何时刻重构世界。特别是，它可以在任何时间点按照任何客户端眼中的样子重建世界。
- 这意味着服务器可以准确地知道你开枪的那一刻你的武器瞄准了什么。这是你的敌人过去的头部位置，但服务器知道这是他的头部在你当前客户端所在的位置。
- 服务器在该时间点处理快照，并更新客户端。

于是皆大欢喜～

服务器很开心是因为他是服务器，他永远都很开心。。。哦好冷啊

你很开心是因为你瞄准目标头部并射击，完成了一记漂亮的爆头

你的敌人可能是唯一不完全开心的哪个，如果他站在原地被你爆头那就是他的问题，但是如果他在移动的话，只能说明你是特别厉害的狙击手。

但是如果他在掩体附近，然后移动到掩体内部的安全位置后才被命中了呢？

好吧这的确有可能发生，但这就是你要为此付出的代价，因为你可以射击「过去的他」，他可能在进入掩体后几毫秒被射击。

从某种程度上来说这是不公平的，但这是大家接受程度最高的解决方案了，明明瞄准开枪最后却 miss 问题更大～

## PART 3 总结

这篇文章是快节奏多人游戏同步这个系列的最后一篇了，虽然这类问题很难得到完美解答，但是对相关概念有了清晰理解以后再看也并不是那么困难。

虽然本文的读者都是游戏开发者，但依然有着另一部分读者对此很感兴趣，那就是玩家们。对玩家来说去理解诸如此类问题依然是一件很有趣的事情。

### 扩展阅读

以下是一些参考资料包括文章和源码之类的，可以帮助大家更方便的理解相关概念。

与本文相关性最高的文章如下 

http://www.gabrielgambetta.com/lag-compensation.html

http://www.gabrielgambetta.com/client-side-prediction-live-demo.html

[What Every Programmer Needs to Know About Game Networking](http://gafferongames.com/networking-for-game-programmers/what-every-programmer-needs-to-know-about-game-networking/)

 [Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization).





https://link.springer.com/article/10.1007/s00530-012-0271-3#Sec17

https://github.com/search?l=C%23&q=lag+compensation&type=Repositories

https://github.com/search?l=C%23&p=1&q=Fast-Paced+Multiplayer&type=Repositories

https://github.com/JoaoBorks/unity-fastpacedmultiplayer

https://github.com/gamestdio/timeline