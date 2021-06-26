---
title : "unity的addressables"
---



## Addressables.InitializeAsync

初始化并填充Addressables.ResourceLocators







## IResourceLocator

资源目录,可以通过 代码加载额外的locator

```
var  requestCatalog=Addressables.LoadContentCatalogAsync("http://192.168.100.100:51879/catalog_extra.json.json", providerSuffix);
var locator = await requestCatalog.Task;
Addressables.AddResourceLocator(locator);
```



存储了一个IResourceLocation列表..根据不同的key(label,path)进行存储

比如: 有个资源是ui.prefab,同时又lable`enter`,`main`可能就会有三个IResourceLocation

- enter为key的IResourceLocation
- main为key的IResourceLocation
- `Assets/prefabs/ui/ui.prefab`为key的IResourceLocation







## IResourceLocation









### MergeMode

**MergeMode**是什么呢？翻译过来是合并模式。

```text
public enum MergeMode
{
    None = 0,
    UseFirst = 0,
    Union,
    Intersection
}
```

 会去先查询每一个地址/标签对应的资源，然后再根据MergeMode进行最终结果的计算。

举个栗子：

比如传入的参数是`new List<object>{"cube", "red"}`，根据cube查询出来的资源有A、B、D，根据red查询出来的资源有C、D、E。

那么MergeMode是Node或UseFirst时，会取第一个key查询到的资源：A、B、D；

MergeMode是Union时，会取所有key查询到的资源的**并集**：A、B、C、D、E；

MergeMode是Intersection时，会取所有key查询到的资源的**交集**：D。











[Addressables.LoadAsset(s)Async | Addressables | 1.15.1 (unity3d.com)](https://docs.unity3d.com/Packages/com.unity.addressables@1.15/manual/LoadingAddressableAssets.html)

