---
title : "理解自动内存管理"
---

当创建对象、字符串或数组时，存储它所需的内存将从称为堆的中央池中分配。当项目不再使用时，它曾经占用的内存可以被回收并用于别的东西。在过去，通常由程序员通过适当的函数调用明确地分配和释放这些堆内存块。如今，像Unity的Mono引擎这样的运行时系统会自动为您管理内存。自动内存管理需要比显式分配/释放更少的编码工作，并大大降低内存泄漏（内存被分配但从未随后释放的情况）的可能性。

## 值类型和引用类型

当调用一个函数时，它的参数值将被复制到一个保留特定调用的内存区域。只占用几个字节的数据类型可以非常快速方便地复制。然而，对象、字符串和数组要大得多，如果这些类型的数据被定期复制，那将是非常低效的。幸运的是，这是不必要的；大项目的实际存储空间是从堆中分配的，一个小的“指针”值用来记住它的位置。从那时起，只有指针在参数传递过程中需要被复制。只要运行时系统能够定位指针标识的项，就可以经常使用数据的一个副本。
在参数传递期间直接存储和复制的类型称为值类型。这些包括整数，浮点数，布尔和Unity的结构类型（例如Color和Vector3）。分配在堆上然后通过指针访问的类型称为引用类型，因为存储在变量中的值仅仅是“引用”到真实数据。引用类型的示例包括对象，字符串和数组。

## 内存分配和垃圾收集

内存管理器跟踪它知道未被使用的堆中的区域。当请求一个新的内存块时（例如当一个对象被实例化时），管理器选择一个未使用的区域，从中分配该块，然后从已知的未使用的空间中移除分配的内存。后续请求以相同的方式处理，直到没有足够大的空闲区域分配所需的块大小。在这一点上，从堆中分配的所有内存仍然在使用中是非常不可能的。只要还存在可以找到它的引用变量，就只能访问堆上的引用项。如果对内存块的所有引用都消失了（即，引用变量已被重新分配，或者它们是现在超出范围的局部变量），则它占用的内存可以安全地重新分配。
为了确定哪些堆块不再被使用，内存管理器会搜索所有当前活动的引用变量，并将它们所指的块标记为`live`。在搜索结束时，内存管理器认为这些`live`块之间的任何空间都是空的，并且可用于后续分配。由于显而易见的原因，定位和释放未使用的内存的过程被称为垃圾回收（或简称GC）。

## 优化

垃圾收集对程序员来说是自动的、不可见的，但是收集过程实际上需要大量的CPU时间。如果正确使用，自动内存管理通常会等于或击败手动分配的整体性能。但是，对于程序员来说，重要的是要避免那些比实际需要触发更多次收集器和在执行中引入暂停的错误。有一些臭名昭著的算法，可能是GC噩梦，尽管他们乍一看是无辜的。重复字符串连接是一个典型的例子：

```
//C# script example
using UnityEngine;
using System.Collections;

public class ExampleScript : MonoBehaviour {
    void ConcatExample(int[] intArray) {
        string line = intArray[0].ToString();
        
        for (i = 1; i < intArray.Length; i++) {
            line += ", " + intArray[i].ToString();
        }
        
        return line;
    }
}

//JS script example
function ConcatExample(intArray: int[]) {
    var line = intArray[0].ToString();
    
    for (i = 1; i < intArray.Length; i++) {
        line += ", " + intArray[i].ToString();
    }
    
    return line;
}
```

这里的关键细节是，新的部分不会被一个接一个地添加到字符串中。实际情况是，每次循环`line`变量的前一个内容都会变死——一个完整的新字符串被分配到包含原来的部分，再在最后加上新的部分。由于字符串随着`i`值的增加而变得更长，所以所消耗的堆空间数量也增加了，因此每次调用这个函数时都很容易消耗数百字节的空闲堆空间。如果你需要连接多个字符串，那么一个更好的选择是Mono库的[System.Text.StringBuilder](https://msdn.microsoft.com/en-gb/library/system.text.stringbuilder.aspx)类。然而，即使反复连接也不会引起太多麻烦，除非它被频繁调用，而在Unity中通常意味着帧更新。就像是：

```
//C# script example
using UnityEngine;
using System.Collections;

public class ExampleScript : MonoBehaviour {
    public GUIText scoreBoard;
    public int score;
    
    void Update() {
        string scoreText = "Score: " + score.ToString();
        scoreBoard.text = scoreText;
    }
}


//JS script example
var scoreBoard: GUIText;
var score: int;

function Update() {
    var scoreText: String = "Score: " + score.ToString();
    scoreBoard.text = scoreText;
}
```

...每次调用Update将分配新字符串，并不断生成的新垃圾。大多数情况下，只有当分数变化时才更新文本：

```
//C# script example
using UnityEngine;
using System.Collections;

public class ExampleScript : MonoBehaviour {
    public GUIText scoreBoard;
    public string scoreText;
    public int score;
    public int oldScore;
    
    void Update() {
        if (score != oldScore) {
            scoreText = "Score: " + score.ToString();
            scoreBoard.text = scoreText;
            oldScore = score;
        }
    }
}


//JS script example
var scoreBoard: GUIText;
var scoreText: String;
var score: int;
var oldScore: int;

function Update() {
    if (score != oldScore) {
        scoreText = "Score: " + score.ToString();
        scoreBoard.text = scoreText;
        oldScore = score;
    }
}
```

当函数返回数组值时，会发生另一个潜在的问题：

```
//C# script example
using UnityEngine;
using System.Collections;

public class ExampleScript : MonoBehaviour {
    float[] RandomList(int numElements) {
        var result = new float[numElements];
        
        for (int i = 0; i < numElements; i++) {
            result[i] = Random.value;
        }
        
        return result;
    }
}


//JS script example
function RandomList(numElements: int) {
    var result = new float[numElements];
    
    for (i = 0; i < numElements; i++) {
        result[i] = Random.value;
    }
    
    return result;
}
```

当创建一个充满值的新数组时，这种函数非常优雅和方便。但是，如果反复调用，那么每次都会分配新的内存。由于数组可能非常大，可用空间可能会迅速消耗，从而导致垃圾收集频繁。避免这个问题的一个方法是利用数组是引用类型的事实。作为参数传递给函数的数组可以在该函数内修改，结果将在函数返回后保留。
像上面这样的功能通常可以被替换成：

```
//C# script example
using UnityEngine;
using System.Collections;

public class ExampleScript : MonoBehaviour {
    void RandomList(float[] arrayToFill) {
        for (int i = 0; i < arrayToFill.Length; i++) {
            arrayToFill[i] = Random.value;
        }
    }
}


//JS script example
function RandomList(arrayToFill: float[]) {
    for (i = 0; i < arrayToFill.Length; i++) {
        arrayToFill[i] = Random.value;
    }
}
```

这只是用新值替换数组的现有内容。虽然这需要在调用代码中完成数组的初始分配（这似乎有些不雅），但是在调用该函数时不会产生任何新的垃圾。

## 主动请求垃圾收集

如上所述，最好尽量避免分配。然而，鉴于它们不能被完全消除，您可以使用两种主要策略来最大限度地减少其入侵游戏：

### 小堆垃圾收集快速可频繁收集

这个策略通常最适合长期游戏的游戏，其中平滑的帧速率是主要的关注点。这样的游戏通常会频繁地分配小块，但这些块将仅在短时间内使用。在iOS上使用此策略时，典型的堆大小约为200KB，iPhone 3G上的垃圾收集大约需要5ms。如果堆增加到1MB，则收集大约需要7ms。因此，有时候可以以规则的帧间隔请求垃圾回收。这通常会使垃圾收集发生的次数比严格的需要的更多，但是它们将被快速处理，对游戏的影响最小：

```
if (Time.frameCount % 30 == 0)
{
   System.GC.Collect();
}
```

但是，您应该谨慎使用此技术，并检查profiler统计信息，以确保它真正减少了游戏的收集时间。

### 大堆垃圾收集缓慢且不可频繁收集

这个策略对于分配（和因此收集）相对不频繁并可以在游戏暂停期间处理的游戏最适用。对于堆来说，尽可能大，而不是因为系统内存太少而导致操作系统杀死你的应用程序。但是，如果可能，Mono运行时会自动避免扩展堆。您可以通过在启动期间预先分配一些占位符空间来手动扩展堆（即，您实例化一个纯粹用于对内存管理器产生影响的“无用”对象）：

```
//C# script example
using UnityEngine;
using System.Collections;

public class ExampleScript : MonoBehaviour {
    void Start() {
        var tmp = new System.Object[1024];
        
        // make allocations in smaller blocks to avoid them to be treated in a special way, which is designed for large blocks
        for (int i = 0; i < 1024; i++)
            tmp[i] = new byte[1024];
        
        // release reference
        tmp = null;
    }
}


//JS script example
function Start() {
    var tmp = new System.Object[1024];

    // make allocations in smaller blocks to avoid them to be treated in a special way, which is designed for large blocks
        for (var i : int = 0; i < 1024; i++)
        tmp[i] = new byte[1024];

    // release reference
        tmp = null;
}
```

一个足够大的堆不应该在游戏中的暂停期间完全被填满，这样可以容纳一次收集。当发生这样的暂停时，您可以显式地请求垃圾收集：

```
System.GC.Collect();
```

同样，在使用此策略时应该小心，并注意Profiler统计数据，而不是仅仅假定它具有所期望的效果。

## 可重复使用的对象池

很多情况下，只要减少创建和销毁对象的数量，就可以避免生成垃圾。游戏中存在着某些类型的物体，如抛射体，尽管一次只会有少量的物体在游戏中，但它们可能会被反复地遇到。在这种情况下，常常可以重用对象，而不是破坏旧对象，并用新的对象替换它们。

## 更多信息

内存管理是一个微妙而复杂的课题，它已经投入了大量的学术研究。如果您有兴趣了解更多信息，那么[memorymanagement.org](http://www.memorymanagement.org/)是一个很好的资源，列出了许多出版物和在线文章。有关对象池的更多信息可以在[维基百科页面](https://en.wikipedia.org/wiki/Object_pool_pattern)和[Sourcemaking.com](https://sourcemaking.com/design_patterns/object_pool)上找到。

> 原文链接：[Understanding Automatic Memory Management](https://docs.unity3d.com/Manual/UnderstandingAutomaticMemoryManagement.html)