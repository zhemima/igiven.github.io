---
title : "ConcurrentDictionary并发字典"
---

ConcurrentDictionary主要用于要从多个线程（或异步任务）修改字典的情景中。如果来自单个线程，则可以根据需要使用尽可能多的代码使用标准Dictionary；）

如果您查看ConcurrentDictionary上的方法，则会发现一些有趣的方法，例如TryAdd，TryGetValue，TryUpdate和TryRemove。

例如，若使用普通的Dictionary类时可能会看到下面的代码
```
// There are better ways to do this... but we need an example ;)
if (!dictionary.ContainsKey(id))
    dictionary.Add(id, value);
    
```
多线程中同时调用该段代码,并且使用相同的id来调用Add,它将引发异常。

ConcurrentDictionary方法TryAdd为您处理该问题，并将返回true/false，告诉您是否已添加它（或该键是否已在字典中）。

因此，除非您在代码的多线程部分中进行工作，否则您可能仅可以使用标准的Dictionary类。