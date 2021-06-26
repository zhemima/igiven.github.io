---
title : "Orleans 最佳实践"
---

 



 # 重入

grain激活体是单线程的，默认情况下，激活体会自始至终地处理完成每个请求后，才会处理下一个请求。
可重入的激活体，可以在上一个请求尚未完成处理的情况下，开始执行另一个请求。执行仍然限于单个线程，因此激活体仍然一次执行一个回合，并且每个回合仅代表激活体的一个请求执行。
可重入的grain代码永远不会并行运行多段grain代码（grain代码的执行将始终是单线程的），但是，可重入的grain可能会看到不同请求交错执行的代码。也就是说，来自不同请求的延续回合，是交错执行的。

因为访问grain一般是需要跨服务器的，所以可重入是很有必要的。。







# 无状态Grain

grain永远是运行在服务端的,包括无状态的.









# time

time是单线程的..



## 错误



```
        public override async Task OnActivateAsync()
        {

            if (this.TimerDisposable == null)
            {
                this.TimerDisposable = RegisterTimer(
                            this.SaveChanges,
                            "timer",
                            TimeSpan.FromMilliseconds(0),
                            TimeSpan.FromMilliseconds(1)
                            );
            }

            await base.OnActivateAsync();
        }



        private Task SaveChanges(object arg)
        {
            this.persisitent.State.num++;

            return this.persisitent.WriteStateAsync();
        }

    	public async Task StartUp()
        {
            this.persisitent.State.num = 0;
            await this.WriteStateAsync();
        }
        
        
           var grain = this.GrainFactory.GetGrain<ITimeTestGrain>(1000);

            await grain.StartUp();
        
```

以上代码当duetime设置为0的时候,会有类型的错误

```
Orleans.Storage.InconsistentStateException: ETag mismatch - tried with ETag: a62ea0e1-a40d-43c3-a508-1e8615351443
```

原因猜测,grain未激活完成的时候,如果为0的时候,这个时候time的callback(SaveChanges)还没有加入到任务队列,就调用了state,而之后也立即同时调用了state这个时候就会报错.

解决方案:

- 不要设置duetime为0
- 或者不要在OnActivateAsync中RegisterTimer,在Activate后再RegisterTimer



# 相关链接 

- https://dotnet.github.io/orleans/Documentation/resources/Best_Practices.html