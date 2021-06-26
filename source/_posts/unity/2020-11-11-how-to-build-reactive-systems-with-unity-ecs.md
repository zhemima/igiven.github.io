---
title : "如何使用Unity ECS打造Reactive System？"
---

### 1. 使用SystemStateComponent

### 原理

所谓State，含义是只能被手动删除的Component，在删除Entity后，依然留下做一些殿后工作，只有被指名要求删除的时候才会被删除。这种特性就让我们可以通过不同的Query去获得结构变化的消息。

### 栗子

1. 假设我们有一个Entity，身上有两个组件，A：IComponentData 与 B: ISystemStateComponentData
2. 当我们添加A组件的时候，通过Filter{ 有A无B }，我们可以在别处得知这个Entity何时被添加。在添加后手动加入B组件。
3. 当我们删除Entity，或者移除A组件的时候，通过Filter { 有B无A }，同理可得知何时这个Entity被移除或是A组件被移除。

更具体的实现可以在查看官方对于ParentSystem的设计。

### 2. 查询ComponentVersion

每当出现某Component相关的结构性变化的时候，该Component的版本就会+1 。

```
EntityManager.GetComponentVersion()
```

------

## 数据变化

顾名思义。方法有三种。

### 1. Chunk检查

### 原理

```csharp
chunk.DidChange(ArchetypeChunkComponentType, version)
```

查看其内部实现

```csharp
if ( ComponentVer > LastSystemVersion ) 
```

什么意思呢？

1. GlobalSystemVersion为记录一个世界所有系统更新信息的版本号。在每一个系统更新**之前**，GSV++。
2. LastSystemVersion为系统记录自己的版本号。在某系统更新**之后**，它会保存GSV，含义是**上次**运行时的版本号，直到下次某系统更新之后，它的版本号不会更改
3. 每一种Component，在System申请写入权限的时候，都会记录该System的LSV 获取方式为chunk.GetArch

因此，if ( ComponentVer > LSV ) 就说明该Component被修改了（有系统获得了写入权限）。翻译成大白话，就是Component是否在提供的版本号之后的时间被更新？在当前系统请求LSV的结果是上一帧该系统的版本号，如果Component被写入时的版本号大于该系统上一帧的版本号，就说明Component在此时至上一帧的某个时间点被写入。

- 由于这个Component只有在遍历的时候申请写入权限时才会记住版本号，因此EntityManager手动更新信息是不会被记录的。
- 这个信息时效性只有一帧，从上次该系统更新后到这次更新后的一帧，因此在这次更新中，修改Component后查询是否改变，答案是True，反之为False。

### 举例

1. 系统的更新顺序为A->B->C->A
2. 那么GSV ：0 -> 1 -> 2 -> 3，每个系统更新之前+1
3. 当数据在B系统被写入，Component就记住了B的GSV = 1
4. 当我们在第二次轮到A系统的时候监测是否Component被改动，DidChange自动使用A系统上次的GSV记录 LSV = 0 与 Component记录的信息CV = 1做对比，发现CV > LSV，得知信息已经被更改了，返回True。

### API

```
chunk.DidChange(InputAType, LastSystemVersion);
```

注意LSV应从EntityManager.LastSystemVersion取得，并传入Job

```csharp
[BurstCompile]
struct UpdateJob : IJobChunk
{
   public ArchetypeChunkComponentType<InputA> InputAType;
   public ArchetypeChunkComponentType<InputB> InputBType;
   [ReadOnly] public ArchetypeChunkComponentType<Output> OutputType;
   public uint LastSystemVersion;

   public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
   {
       var inputAChanged = chunk.DidChange(InputAType, LastSystemVersion);
       var inputBChanged = chunk.DidChange(InputBType, LastSystemVersion);
       if (!(inputAChanged || inputBChanged))
           return;
      //...
}
```

### 2. Query自动检查

在声明Query的时候，特别注明

```
m_Group.SetFilterChanged(new ComponentType{ typeof(InputA), typeof(InputB)});
```

这样Query就会把没被修改的ComponentType排除在外。注意，这种检查是Component层级，而不是单个Entity层级的。

```csharp
EntityQuery m_Group;
protected override void OnCreate()
{
   m_Group = GetEntityQuery(typeof(Output), 
                               ComponentType.ReadOnly<InputA>(), 
                               ComponentType.ReadOnly<InputB>());
   m_Group.SetFilterChanged(new ComponentType{ typeof(InputA), typeof(InputB)});
}
```

### 3. IJobForEach中使用 [ChangeFilter]

与Query的排除效果类似。

### 示例

```csharp
public struct ProcessTendency : IJobForEachWithEntity<HumanState, HumanStock>
{
    public void Execute(Entity entity, int index, [ChangedFilter] ref State state)
    {
```

## Best Practice

对自己的系统做了一些单元测试后，我觉得有几点应该记录下来。

- 如果要做一套响应式系统循环触发，慎用EntityManager获取信息。因为EntityManager的Get系列API会直接获取写入权限。
- Job有延迟，因此单元测试的时候，如果涉及Version，应该做一些操作“等待”正确的版本号刷新，比如Debug.Log()。
- GetEntityQuery会在系统注册，因此在Query里SetFilterChanged即可，毋需在Execute()中再声明一遍。
- 在主线程获取DynamicBuffer的长度时，Query与EM都不能做到以只读方式做到。因此正确的做法是

```csharp
var query    = GetEntityQuery(ComponentType.ReadOnly<T>());
var entities  = query.ToEntityArray(Allocator.TempJob);
var entity    = entities[0];
var length   = EntityManager.GetChunk(entity).GetBufferAccessor(GetArchetypeChunkBufferType<Tendency>(true))[0].Length;
entities.Dispose();
```

## Reference

[https://gametorrahod.com/designing-an-efficient-system-with-version-numbers/](https://link.zhihu.com/?target=https%3A//gametorrahod.com/designing-an-efficient-system-with-version-numbers/)

[Using IJobChunk | Package Manager UI website](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/Packages/com.unity.entities%400.0/manual/chunk_iteration_job.html)

[Coping with Change in Unity3D ECS | by Maxim Zaks | Medium](https://medium.com/@icex33/coping-with-change-in-unity3d-ecs-45422fff8dda#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6ImQ5NDZiMTM3NzM3Yjk3MzczOGU1Mjg2YzIwOGI2NmU3YTM5ZWU3YzEiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2MDUxMTkzMDksImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjExNTYwNDIyOTcxMDAwNDczMDgwOCIsImVtYWlsIjoiemhlcGFtYUBnbWFpbC5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXpwIjoiMjE2Mjk2MDM1ODM0LWsxazZxZTA2MHMydHAyYTJqYW00bGpkY21zMDBzdHRnLmFwcHMuZ29vZ2xldXNlcmNvbnRlbnQuY29tIiwibmFtZSI6IuadjuS5neS7mSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vLXlfSnhUTXJkQWk4L0FBQUFBQUFBQUFJL0FBQUFBQUFBQUFBL0FNWnV1Y25xLVNRSGVVR1FRUENfUjJ0Zk93RGh2QzRHbHcvczk2LWMvcGhvdG8uanBnIiwiZ2l2ZW5fbmFtZSI6IuS5neS7mSIsImZhbWlseV9uYW1lIjoi5p2OIiwiaWF0IjoxNjA1MTE5NjA5LCJleHAiOjE2MDUxMjMyMDksImp0aSI6IjQyM2E3NTM3NmE0YTZkNzI3ODgzYmYyZDIyMDljMGNjZTg2NDRjMTUifQ.eicZfv00n72yCqIDVFxy8XOMlWkcjychu8aPN6q9Cj61MvcJfIAAVgdREk1J5t4dpTxqN10FOZnfFxRZBnkbHFjjVLI0Lu1lYr7bE5NhxdRnjaNwbs-WQbuMHAKBeMvdKzIIeqRAswuHNlwtd_ATfON4V0v3aobOlVPOFUz3mgstb5cFhoH2oJIGTfnYj108-1_IVJRUoTo80Eii2gTEsC4frSGweQNZlu2XMnfsfRz3724NqinN7csfXXQ9zZ8vh29zHBgEUYnVh7JNOONr8A9wQY9RxI1v73HEC2YNFekuB87qCjGRpgUKGGFBaUwClpOifr9W-Z30U3t_TSgENw)

[How to build reactive systems with Unity ECS: Part 1 • Effective Unity](https://www.effectiveunity.com/ecs/06-how-to-build-reactive-systems-with-unity-ecs-part-1/)

https://www.effectiveunity.com/ecs/07-how-to-build-reactive-systems-with-unity-ecs-part-2/