---
title : "理解托管堆"
---

另一个Unity开发者面对的普遍问题是托管堆的意外扩展。在Unity中，托管堆的扩张比收缩更加容易。进一步说，Unity的垃圾回收策略倾向于碎片化内存，这可以防止收缩出来一个大的堆。

------

## 技术细节：托管堆怎样运行以及为什么它会扩张

​    托管堆是内存中被项目脚本运行时（Mono或者IL2CPP）的内存管理器自动管理的部分。在托管代码中创造的所有对象必须被分配在托管堆上（请注意：严格的说，所有非空引用类型的对象和所有被装箱的值类型对象都必须被分配到托管堆上）。

![img](../../assets/images/2020-02-18-BestPracticeUnderstandingPerformanceInUnity/UnderstandingPerformanceinUnity-AssetAuditingSection_image_0.png)

​    在上面的图表中，白色的方块表示分配到托管堆的内存量，它里面有颜色的方块表示托管堆内存空间中存储的数据值。当另外的值被需要时，将会从托管堆中分配更多空间。

​    垃圾收集器周期运行（请注意：具体时间取决于不同的平台）。会清扫堆上的所有对象，检查那些不再被引用的对象并标记。然后删除没有被引用的对象，释放内存。

​    至关重要的是，Unity的垃圾收集是，采用[Boehm GC算法](https://en.wikipedia.org/wiki/Boehm_garbage_collector)，是非世代和非压缩的。非世代意味着当执行一次收集的时候，GC必须清扫全部的托管堆，并且性能会随着堆的扩张而降低。非压缩意味着内存中的对象不会重新移动位置来使对象间的空隙关闭。

![img](../../assets/images/2020-02-18-BestPracticeUnderstandingPerformanceInUnity/UnderstandingPerformanceinUnity-AssetAuditingSection_image_1.png)

​    上面的图显示了一个内存片段的例子。当一个对象被释放时，其内存也会被清空。但是被释放的空间不会变成一个大的可用内存池的一部分。被释放对象两边的对象仍然被使用。由于这个原因，被释放的内存变成了其他内存段中间的空隙（这个空隙在上图中由红圈标明）。这个新释放的空间只能用于存储与它完全相同或者更小大小的对象。 

​    当分配一个对象时，请记住在内存中对象必须总是占用一段连续的空间。

​    这导致了内存碎片的核心问题，虽然堆中的总可用空间是非常大的，但是可能一部分或者所有这些可用空间是被分配对象间的小空隙。在这种情况下，尽管可能会有足够的总空间来容纳一个确定大小的分配，但是托管堆找不到一块足够大的连续内存来适配这次分配。

![img](../../assets/images/2020-02-18-BestPracticeUnderstandingPerformanceInUnity/UnderstandingPerformanceinUnity-AssetAuditingSection_image_2.png)

​    然而，如果一个大的对象被分配，并且没有足够的连续空间来容纳这个对象，如同上图所示，那么Unity的内存管理器将执行两个操作。

​    首先，如果垃圾收集器还没有运行，那么它就会运行。尝试释放足够的空间来填充内存分配的需求。

​    如果，GC运行完毕，仍然没有可以满足需求的连续内存空间，那么托管堆将会被扩大。托管堆扩大的具体大小由平台决定；然而，大多数Unity平台将托管堆扩大一倍。

------

## 托管堆的关键问题

​    托管堆的核心问题是其扩张是以两倍进行的：

​    ·当托管堆扩张时，Unity并不总是释放分配到托管堆的内存页，它采取一个优化策略，保持扩张的托管堆，即使托管堆大部分是空的。这用来避免当有更大的内存分配发生时需要重新扩大托管堆。

​    ·在大多数平台上，Unity最终会将托管堆上那部分空的内存页释放回操作系统。这个发生的间隔时间是不能保证的，并且也不能被依靠。

​    ·用于托管堆的地址空间（栈）不再会被返还给操作系统。

​    ·对于32位程序来说，如果托管堆扩张收缩多次，这会导致地址空间（栈）耗尽。如果一个程序可用内存的地址空间（栈）被耗尽，操作系统将会终止这个程序。

​    ·对于64位程序来说，地址空间（栈）足够大，这对于运行时间不超过人类平均寿命的程序来说基本不会发生。

------

## 临时分配

​    许多Unity项目都发现每帧有几十或几百kb的临时数据分配到托管堆处理。这通常对项目性能是非常有害的。考虑下面的计算：

​    如果一个程序每帧分配1kb的临时内存，运行在每秒60帧的情况下，那么它每秒就会分配60kb的内存。在一分钟之内，这在内存中增加了3.6mb的垃圾。每秒执行一次垃圾回收器会降低性能，但是尝试在低内存设备上运行时每分钟分配3.6mb内存是有问题的。

​    进一步说，考虑到加载操作。如果在一个繁重的资源加载过程中生成了大量的临时对象，并且这些对象被引用到该操作完成，那么垃圾回收器则不能释放这些临时对象并且托管堆需要扩张——即使托管堆中包含的许多这些对象马上就会被释放。

![img](../../assets/images/2020-02-18-BestPracticeUnderstandingPerformanceInUnity/UnderstandingPerformanceinUnity-AssetAuditingSection_image_3.png)

​    保持对托管内存分配的追踪非常简单。在Unity的CPU Profiler中，概述写着“GC Alloc”那一列。这一行显示着在某一帧分配到托管堆的比特数。（请注意：这个数字与在制定帧分配的临时比特数不相同。分析器显示的事在特定帧被分配的比特数量，即使这些被分配内存的一些或全部在随后的帧中被复用。）将“Deep Profiling”选项打开，这就可以追踪这些内存分配是在哪些函数中发生的。

​    Unity Profiler不会追踪那些没有发生在主线程的内存分配。因此，“GC Alloc”列就不能用来测量在用户自己创建的线程中的托管内存分配。可以把其他线程中的代码切换到主线程中执行来进行调试，或使用BeginThreadProfiling这个API来在Profiler的TimeLine选项卡中来显示样本。

​    在目标设备上使用开发模式构建可以分析托管内存分配。

​    请注意：一些脚本函数在Editor中运行会导致内存分配，但是当项目构建后就不会再产生内存分配。GetComponent就是最普遍的例子，这个函数在Editor中运行时总是会产生内存分配，但是在构建好的项目中就不会。

​    总体来说，我们强烈的建议所有开发者当项目在交互状态时最小化托管堆内存分配。如果内存分配发生在没有交互操作的时候，比如说场景加载时，问题较少。

​    Jetbrains Resharper这个Visual Studio插件可以帮助定位代码中的内存分配。

​    使用Unity的深度分析模式来定位托管内存分配的特定原因。在深度分析模式，所有的函数的调用都被独立记录，在函数调用树中提供一个清晰界面来显示托管内存分配是在哪里发生的。请注意，深度分析模式不止在编辑器中，也可以使用命令行参数-deepprofiling在安卓和桌面上使用。在分析过程中深度分析器按钮保持灰色。

------

## 基础内存维护

​    有一些相对简单的技术来使托管堆内存分配降低。

------

#### 容器和数组重用

​    当使用C#的容器类或者数组时，如果可能考虑重用或者缓存分配的容器或者数组。容器类暴露了一个Clear函数，当清除容器的值时不会释放分配给容器的内存。

> void Update() {
>
>   List<float> nearestNeighbors = new List<float>();
>
>   findDistancesToNearestNeighbors(nearestNeighbors);
>
>   nearestNeighbors.Sort();
>
>   // … use the sorted list somehow …
>
> }

​    当给一个复杂计算分配临时的“帮助性”容器时尤其好用。下面的代码是一个非常简单的例子：

​    在这个例子中，nearestNeighbors列表每帧被分配一次，用于存储一些数据点。可以把这个列表非常简单的从这个函数中提出来放入函数所在的类中，这避免了每帧分配一个新的列表：

> List<float> m_NearestNeighbors = new List<float>();
>
> void Update() {
>
>   m_NearestNeighbors.Clear();
>
>   findDistancesToNearestNeighbors(NearestNeighbors);
>
>   m_NearestNeighbors.Sort();
>
>   // … use the sorted list somehow …
>
> }

​    在这个版本中，每帧列表的内存会被保持和重用。只有当列表需要被扩大时内存才会被分配。

------

#### 闭包和匿名函数

​    当使用闭包和匿名函数时，有两点需要考虑。

​    首先，所有函数引用在C#中都是引用类型，因此分配到了托管堆。临时内存分配可以通过作为一个参数传递函数引用很简单的创建。这个内存分配不管是使用匿名函数或者是预先定义好的函数来传递都会发生。

​    其次，转换一个匿名函数为闭包显著增加了需要传递闭包到函数接收的内存量。

​    考虑下面的代码：

> List<float> listOfNumbers = createListOfRandomNumbers();
>
> listOfNumbers.Sort( (x, y) =>
>
> (int)x.CompareTo((int)(y/2))
>
> );

​    这段代码使用了一个简单的匿名函数来控制第一行创建的列表成员的排序方式。可是，如果一个程序员想要重用这段代码，可以考虑使用一个局部变量来代替常量2，像下面这样：

> List<float> listOfNumbers = createListOfRandomNumbers();
>
> int desiredDivisor = getDesiredDivisor();
>
> listOfNumbers.Sort( (x, y) =>
>
> (int)x.CompareTo((int)(y/desiredDivisor))
>
> );

​    现在匿名函数需要在此函数作用范围之外获取一个变量的状态，所以变成了一个闭包。desiredDivisor变量必须通过某种方式传进闭包中，以便被闭包中的实际代码所使用。

​    为了实现这个需求，C#生成了一个匿名类，用来保存闭包需要的超过作用范围的对象。当闭包被传入Sort函数时，将生成一个此类的副本，并且这个类的副本通过整数desiredDivisor的值来初始化。

​    由于执行闭包需要实例化一个生成它的类的拷贝，在C#中所有类都是引用类型，那么执行一个闭包需要在托管堆中分配一个对象。

​    总体来说，如果可能最好避免在C#中使用闭包。在性能敏感的代码中应该尽量最小化使用匿名函数和函数引用，尤其是在基于每帧执行的代码中。

------

#### IL2CPP下的匿名函数

​    目前，检查通过IL2CPP生成的代码，显示简单的声明以及指定一个System.Function类型的变量会分配一个新的对象。无论这个变量是显式（在一个函数或类中声明）或是隐式（作为一个函数的参数声明）都会这样。

​    所以，在IL2CPP脚本后端下所有使用匿名函数的情况都会分配托管堆内存。Mono脚本后端不是这种情况。

​    进一步说，在IL2CPP中，由于函数参数声明方式的不同，会有显著不同的托管堆内存分配量级的不同。正如预期的那样，闭包在每次调用中分配最多的内存。

​    不直观的说，在IL2CPP脚本后端下，当作为一个参数传递预定义的函数时，分配近似于闭包的内存量。匿名函数在托管堆上生成最少量的暂时垃圾，这是通过一个或多个量级的指令实现的。

​    所以，如果一个项目想要在IL2CPP脚本后端上发布，有三的关键点需要注意：

​    ·采取不需要传递函数作为参数的的编码风格

​    ·当这种情况不可避免时，采用匿名函数而不是预定义函数

​    ·避免闭包，不管用什么脚本后端

------

## 装箱

​    装箱是Unity项目中最普遍的意识不到的临时内存分配的原。它发生在当一个值类型被当做引用类型使用时；这通常发生在传递原始的值类型变量（比如int和float）到对象类型的函数中。

​    在这个极度简单的例子中，为了要传递到object.Equals函数中，整数x被装箱，由于object的Equals函数要求传入的是一个object作为参数。

> int x = 1;
>
> object y = new object();
>
> y.Equals(x);

​    C#的IDE和编译器通常不会对装箱发出警告，即使其导致了无意识的内存分配。这是因为C#语言是在小的临时内存分配会被世代的垃圾回收器和分配大小敏感的内存池有效率处理的假设下开发的。

​    由于Unity的内存分配器使用不区分内存分配大小的内存池，并且Unity的垃圾回收器也不是世代的，因此其不能有效率的清除由装箱带来的小的，频率的临时内存分配。

​    在Unity运行时使用C#编码时，应该尽量避免装箱。

------

#### 识别装箱

​    基于正在使用的脚本后端，装箱在CPU跟踪数据中显示为一些函数的调用。它们通常采用下面的这些形式之一，<some class>是一些其他类或者结构体的名字，...是一些参数的数量：

​    ·<some class>::Box(…)

​    ·Box(…)

​    ·<some class>_Box(…)

​    它可以通过搜索反编译器或者IL查看器的输出来定位，例如ReSharper中内置的IL查看器或者dotPeek反编译器，IL指令是“box”。

------

#### 字典和枚举

​    一个普遍的造成装箱的问题是使用枚举类型来作为字典的键。声明一个枚举创建了一个新的值类型，在后台被作为像一个整数来对待，但是在编译时强制执行类型安全原则。

​    默认情况下，调用Dictionary.add(key, value)的结果是调用Object.getHashCode(Object)。这个函数用于为字典的键获得合适的散列码，并且用于所有接受key的函数：Dictionary.tryGetValue, Dictionary.remove等等。

​    Object.getHashCode函数是一个引用类型，但是枚举值始终是一个值类型。因此，对于枚举作为键的字典来说，每次函数调用都会至少一次对键进行装箱。

​    下面的代码片段举出了一个简单的例子展示了装箱的问题：

> enum MyEnum { a, b, c };
>
> var myDictionary = new Dictionary<MyEnum, object>();
>
> myDictionary.Add(MyEnum.a, new object());

​    要解决这个问题，有必要写一个自定义的类实现IEqualityComparer接口，并且传递这个类的实例到字典的比较器中（请注意：这个对象通常是无状态的，所里可以被不同的字典重用以节省内存）。

​    下面的是一个对上面代片段实现IEqualityComparer的简单例子。

> public class MyEnumComparer : IEqualityComparer<MyEnum> {
>
>   public bool Equals(MyEnum x, MyEnum y) {
>
> ​    return x == y;
>
>   }
>
>   public int GetHashCode(MyEnum x) {
>
> ​    return (int)x;
>
>   }
>
> }

​    可以将上面类的实例传递给字典的构造函数。

------

#### Foreach循环

​    在Unity版本的Mono的C#编译器中，使用foreach循环会在每次循环结束时强制Unity去装箱一个值（请注意：在每次循环整个结束的时候这个值会被装箱一次。在这个循环中每次迭代不会装箱，所以不论循环两次还是两百次内存使用都是相同的）。这是因为通过Unity的C#编译器生成的IL构造了一个通用的值类型枚举器以迭代值类型的容器。

​    这个枚举器实现的IDisposable接口，其肯定会在循环终止时被调用。然而，在值类型对象（比如结构体和枚举器）上调用接口函数需要把它们装箱。

​    检查下面非常简单的代码例子：

> int accum = 0;
>
> foreach(int x in myList) {
>
>   accum += x;
>
> }

​    上面的代码，当通过Unity的C#编译器运行时，产生下面的中间语言：

> .method private hidebysig instance void
>
>   ILForeach() cil managed
>
>  {
>
>   .maxstack 8
>
>   .locals init (
>
>    [0] int32 num,
>
>    [1] int32 current,
>
>    [2] valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32> V_2
>
>   )
>
>   // [67 5 - 67 16]
>
>   IL_0000: ldc.i4.0  
>
>   IL_0001: stloc.0   // num
>
>   // [68 5 - 68 74]
>
>   IL_0002: ldarg.0   // this
>
>   IL_0003: ldfld    class [mscorlib]System.Collections.Generic.List`1<int32> test::myList
>
>   IL_0008: callvirt   instance valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<!0/*int32*/> class [mscorlib]System.Collections.Generic.List`1<int32>::GetEnumerator()
>
>   IL_000d: stloc.2   // V_2
>
>   .try
>
>   {
>
>    IL_000e: br      IL_001f
>
>   // [72 9 - 72 41]
>
>    IL_0013: ldloca.s   V_2
>
>    IL_0015: call     instance !0/*int32*/ valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32>::get_Current()
>
>    IL_001a: stloc.1   // current
>
>   // [73 9 - 73 23]
>
>    IL_001b: ldloc.0   // num
>
>    IL_001c: ldloc.1   // current
>
>    IL_001d: add     
>
>    IL_001e: stloc.0   // num
>
>   // [70 7 - 70 36]
>
>    IL_001f: ldloca.s   V_2
>
>    IL_0021: call     instance bool valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32>::MoveNext()
>
>    IL_0026: brtrue    IL_0013
>
>    IL_002b: leave    IL_003c
>
>   } // end of .try
>
>   finally
>
>   {
>
>    IL_0030: ldloc.2   // V_2
>
>    IL_0031: box     valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32>
>
>    IL_0036: callvirt   instance void [mscorlib]System.IDisposable::Dispose()
>
>    IL_003b: endfinally 
>
>   } // end of finally
>
>   IL_003c: ret     
>
>  } // end of method test::ILForeach
>
> } // end of class test

​    最有关系的代码是靠近最后的__finally { … }__语句块。callvirt指令在执行这个函数前在内存中发现IDisposable.Dispose方法的定位，并且要求这个枚举器被装箱。

​    总体来说，在Unity中应该避免使用foreach循环。不全是因为装箱，还有是通过枚举器实现的容器迭代的函数调用消耗比常规的for或者while循环的迭代要慢的多。

​    请注意在Unity5.5中C#编译器有了重大升级，增强了Unity生成IL的能力。特别是，装箱操作已经从foreach循环中消除。然而，由于函数调用的开销，与基于数组的等价代码相比CPU性能差距依旧存在。

------

#### 有数组值的Unity API(*译者已查，在Unity官方文档中，会明确表明这些API有临时内存分配*)

​    一个更有害且难于发现的是由于重复使用Unity返回数组的API造成的虚拟的数组内存分配。所有返回数组的Unity的API都会在每次它们被访问时返回一个该数组新的拷贝。在没有必要的情况下经常访问有数组值的Unity的API效果非常不理想。

​    举个例子，下面的代码在vertices数组的每次迭代中虚拟的创造了四份拷贝。内存分配在每次.vertices属性被访问时发生。

> for(int i = 0; i < mesh.vertices.Length; i++)
>
> {
>
>   float x, y, z;
>
>   x = mesh.vertices[i].x;
>
>   y = mesh.vertices[i].y;
>
>   z = mesh.vertices[i].z;
>
>   // ...
>
>   DoSomething(x, y, z); 
>
> }

​    这可以通过普通的重构来使其只有一次数组内存分配，不论循环迭代多少次。这是通过在进入循环前存储vertices数组来实现的。

> var vertices = mesh.vertices;
>
> for(int i = 0; i < vertices.Length; i++)
>
> {
>
>   float x, y, z;
>
>   x = vertices[i].x;
>
>   y = vertices[i].y;
>
>   z = vertices[i].z;
>
>   // ...
>
>   DoSomething(x, y, z); 
>
> }

​    虽然CPU的消耗在访问一个属性一次时并不是非常高，在持续的循环中重复的访问它们会导致性能热点。进一步来讲，没必要的重复访问会导致托管堆的扩张。

​    这个问题再移动设备上非常普遍，因为Input.touches这个API的行为与上述类似。这在包含下面的类似代码的项目中也非常普遍，内存分配发生在每次访问.touches属性时。

> for ( int i = 0; i < Input.touches.Length; i++ )
>
> {
>
>   Touch touch = Input.touches[i];
>
>   // …
>
> }

​    然而，现在有很多Unity的API有不会造成内存分配的新版本。当使用这些时，应该是更有利的。

> int touchCount = Input.touchCount;
>
> for ( int i = 0; i < touchCount; i++ )
>
> {
>
>   Touch touch = Input.GetTouch(i);
>
>   // …
>
> }

​    转换上面的例子到Touch API 无内存分配的版本是非常简单的：

​    请注意：这个属性的访问(Input.touchCount)仍旧要放在循环条件的外面，这是为了节省CPU在执行属性get方法时的消耗。

------

#### 空数组重用

​    一些团队在一个数组返回值的函数需要返回一个空值时，倾向于用返回一个空数组来代替null。这种编码模式在许多托管语言中非常常见，尤其是C#和Java。

​    总体来说，当一个函数返回一个长度为0的数组时，返回一个预先分配好的长度为0的数组的单例实例比重复创造空数组被认为更有效率。（请注意：当然，一个例外情况要考虑那就是当返回的数组需要改变长度时）







- https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity4-1.html

