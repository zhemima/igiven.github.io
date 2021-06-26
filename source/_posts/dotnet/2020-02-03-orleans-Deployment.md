---
title:  "orleans的部署模式"
---



# 蓝绿部署（Blue/Green Deployment）

过去的 10 年里，很多公司都在使用蓝绿部署（发布）来实现热部署，这种部署方式具有安全、可靠的特点。蓝绿部署虽然算不上“ Sliver Bullet”，但确实很实用。
蓝绿部署是最常见的一种0 downtime部署的方式，是一种以可预测的方式发布应用的技术，目的是减少发布过程中服务停止的时间。蓝绿部署原理上很简单，就是通过冗余来解决问题。通常生产环境需要两组配置（蓝绿配置），一组是active的生产环境的配置（绿配置），一组是inactive的配置（蓝绿配置）。用户访问的时候，只会让用户访问active的服务器集群。在绿色环境（active）运行当前生产环境中的应用，也就是旧版本应用version1。当你想要升级到version2 ，在蓝色环境（inactive）中进行操作，即部署新版本应用，并进行测试。如果测试没问题，就可以把负载均衡器／反向代理／路由指向蓝色环境了。随后需要监测新版本应用，也就是version2 是否有故障和异常。如果运行良好，就可以删除version1 使用的资源。如果运行出现了问题，可以通过负载均衡器指向快速回滚到绿色环境。
蓝绿部署的优点：
这种方式的好处在你可以始终很放心的去部署inactive环境，如果出错并不影响生产环境的服务，如果切换后出现问题，也可以在非常短的时间内把再做一次切换，就完成了回滚。而且同时在线的只有一个版本。蓝绿部署无需停机，并且风险较小。
(1) 部署版本1的应用（一开始的状态），所有外部请求的流量都打到这个版本上。
(2) 部署版本2的应用，版本2的代码与版本1不同(新功能、Bug修复等)。
(3) 将流量从版本1切换到版本2。
(4) 如版本2测试正常，就删除版本1正在使用的资源（例如实例），从此正式用版本2。
从过程不难发现，在部署的过程中，应用始终在线。并且，新版本上线的过程中，并没有修改老版本的任何内容，在部署期间，老版本的状态不受影响。这样风险很小，并且，只要老版本的资源不被删除，理论上，可以在任何时间回滚到老版本。
蓝绿部署的弱点：
使用蓝绿部署需要注意的一些细节包括：
1、当切换到蓝色环境时，需要妥当处理未完成的业务和新的业务。如果数据库后端无法处理，会是一个比较麻烦的问题。
2、有可能会出现需要同时处理“微服务架构应用”和“传统架构应用”的情况，如果在蓝绿部署中协调不好这两者，还是有可能导致服务停止；
3、需要提前考虑数据库与应用部署同步迁移/回滚的问题。
4、蓝绿部署需要有基础设施支持。
5、在非隔离基础架构（ VM 、 Docker 等）上执行蓝绿部署，蓝色环境和绿色环境有被摧毁的风险。
6、另外，这种方式不好的地方还在于冗余产生的额外维护、配置的成本，以及服务器本身运行的开销。
蓝绿部署适用的场景：
1、不停止老版本，额外搞一套新版本，等测试发现新版本OK后，删除老版本。
2、蓝绿发布是一种用于升级与更新的发布策略，部署的最小维度是容器，而发布的最小维度是应用。
3、蓝绿发布对于增量升级有比较好的支持，但是对于涉及数据表结构变更等等不可逆转的升级，并不完全合适用蓝绿发布来实现，需要结合一些业务的逻辑以及数据迁移与回滚的策略才可以完全满足需求。

 

# 滚动发布（rolling update）

滚动发布，一般是取出一个或者多个服务器停止服务，执行更新，并重新将其投入使用。周而复始，直到集群中所有的实例都更新成新版本。这种部署方式相对于蓝绿部署，更加节约资源——它不需要运行两个集群、两倍的实例数。我们可以部分部署，例如每次只取出集群的20%进行升级。
这种方式也有很多缺点，例如：
(1) 没有一个确定OK的环境。使用蓝绿部署，我们能够清晰地知道老版本是OK的，而使用滚动发布，我们无法确定。
(2) 修改了现有的环境。
(3) 如果需要回滚，很困难。举个例子，在某一次发布中，我们需要更新100个实例，每次更新10个实例，每次部署需要5分钟。当滚动发布到第80个实例时，发现了问题，需要回滚。此时，脾气不好的程序猿很可能想掀桌子，因为回滚是一个痛苦，并且漫长的过程。
(4) 有的时候，我们还可能对系统进行动态伸缩，如果部署期间，系统自动扩容/缩容了，我们还需判断到底哪个节点使用的是哪个代码。尽管有一些自动化的运维工具，但是依然令人心惊胆战。
并不是说滚动发布不好，滚动发布也有它非常合适的场景。


![img](../../assets/images/2020-02-03-orleans-Deployment/20190516171224267.png)



# orleans支持 *蓝绿部署模型*以及*滚动部署模型*

- ClusterId：这是Orleans集群的唯一ID。使用此ID的所有客户端和Silo将能够直接相互通信。但是，您可以选择ClusterId对不同的部署使用不同的名称。

- ServiceId：这是您的应用程序的唯一ID，将由某些提供程序（例如持久性提供程序）使用。此ID应该保持稳定，并且在整个部署中都不应更改。

Orleans has both `ClusterId` & `ServiceId` to support the *blue/green deployment model*.

In this model, each deployment slot will have a distinct `ClusterId` (eg, the values could be "blue-slot" & "green-slot") but they will always have the same `ServiceId` (eg, "my-service"). The "blue-slot" silos will only talk to other "blue-slot" silos.

*However* **Grain A** in the blue cluster and **Grain A** in the green cluster will still share the same storage - if they are both active then one will see a conflict when writing to the state if the other activation has already written it.

This allows for the state in the database to remain consistent when multiple clusters are active (which is usually a short period of time - during the upgrade).

If you do not use blue/green deployments then you can set `ClusterId` & `ServiceId` to the same value.

To say this in a different way:

- `ClusterId` + `ServiceId` are used for cluster membership
- `ServiceId` is used for storage

https://github.com/dotnet/orleans/issues/5696#issuecomment-503595998