---
title : "unity world time system"
---





```

  [Preserve]
    [UpdateInGroup(typeof(InitializationSystemGroup))]
    public class UpdateWorldTimeSystem : ComponentSystem
    {
        private bool hasTickedOnce = false;

        protected override void OnUpdate()
        {
            var currentElapsedTime = Time.ElapsedTime;
            var deltaTime = math.min(UnityEngine.Time.deltaTime, World.MaximumDeltaTime);
            World.SetTime(new TimeData(
                elapsedTime: hasTickedOnce ? (currentElapsedTime + deltaTime) : currentElapsedTime,
                deltaTime: deltaTime
            ));
            hasTickedOnce = true;
        }
    }
```





```
    internal struct WorldTime : IComponentData
    {
        public TimeData Time;
    }

    internal struct WorldTimeQueue : IBufferElementData
    {
        public TimeData Time;
    }
```





World.cs

```

        protected Entity TimeSingleton
        {
            get
            {
                if (m_TimeSingletonQuery.IsEmptyIgnoreFilter)
                {
        #if UNITY_EDITOR
                    var entity = EntityManager.CreateEntity(typeof(WorldTime), typeof(WorldTimeQueue));
                    EntityManager.SetName(entity , "WorldTime");
        #else
                    EntityManager.CreateEntity(typeof(WorldTime), typeof(WorldTimeQueue));
        #endif
                }

                return m_TimeSingletonQuery.GetSingletonEntity();
            }
        }

        public void SetTime(TimeData newTimeData)
        {
            EntityManager.SetComponentData(TimeSingleton, new WorldTime() {Time = newTimeData});
            this.Time = newTimeData;
        }

        public void PushTime(TimeData newTimeData)
        {
            var queue = EntityManager.GetBuffer<WorldTimeQueue>(TimeSingleton);
            queue.Add(new WorldTimeQueue() { Time = this.Time });
            SetTime(newTimeData);
        }

        public void PopTime()
        {
            var queue = EntityManager.GetBuffer<WorldTimeQueue>(TimeSingleton);

            Assert.IsTrue(queue.Length > 0, "PopTime without a matching PushTime");

            var prevTime = queue[queue.Length - 1];
            queue.RemoveAt(queue.Length - 1);
            SetTime(prevTime.Time);
        }

```

.

ComponentSystemBase

```
        /// <summary>
        /// The World in which this system exists.
        /// </summary>
        /// <value>The World of this system.</value>
        public World World => m_StatePtr != null ? (World)m_StatePtr->m_World.Target : null;

        /// <summary>
        /// The current Time data for this system's world.
        /// </summary>
        public ref readonly TimeData Time => ref World.Time;

```







