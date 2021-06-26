---
title : "commandbuff的一些问题"
---



component

```
public struct State:IComponentData
{
	public int a;
} 
```

a系统

```

 var PostUpdateCommands = World.GetExistingSystem<AbilityUpdateCommandBufferSystem>().CreateCommandBuffer();
Entities
    .ForEach((Entity entity) =>
    {
    	var state = EntityManager.GetComponent<State>(entity);
    	state.a = 1111;
    	PostUpdateCommands.SetComponent(entity,state);
    })
    .WithoutBurst()
    .Run();


```

b系统

```
Entities
    .ForEach((Entity entity) =>
    {
    	var state = EntityManager.GetComponent<State>(entity);
    	state.a = 2222;
    	EntityManager.SetComponentData(entity,state);
    })
     .WithStructuralChanges()
    .Run();
```

执行顺序

```
A-->B-->AbilityUpdateCommandBufferSystem
```

这里的结果,我们以为是2222.其实是1111.

所以要么全是使用commandbuff.否则容易造成数据错误

