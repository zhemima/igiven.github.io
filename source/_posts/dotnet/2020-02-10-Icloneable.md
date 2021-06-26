---
title : "C# 深浅复制"
---

关于深浅复制大家可参考MSDN：https://msdn.microsoft.com/zh-cn/library/system.object.memberwiseclone.aspx 

所谓深浅复制可解读为：

浅复制：C#语言种的MemberwiseClone方法仅仅是再内存种创建一个新对象，然后将原有对象的所有字段直接复制过去，无论是值类型还是引用类型，是值类型的就复制值类型，是应用类型的就复制引用本身（而不会复制所引用的对象），因此，是浅拷贝。

需要注意的是:  在利用MemberwiseClone()创建新对象的时候，不会像利用new的时候一样，也就是不会先执行字段的创建和构造方法条用的过程。

深复制：如果字段是值类型的，则对字段执行逐位复制，如果字段是引用类型的，则把引用类型的对象指向**一个全新的对象！**



## ICloneable接口

ICloneable接口包含一个Clone方法，可以用来创建当前对象的拷贝。

```
public interface ICloneable 
{ 
    object Clone(); 
}
```


ICloneable的问题是Clone方法并不会显式地指定是执行浅拷贝或深拷贝，因此调用者将无法确定实际情况。因此，有一些关于把ICloneable从.NET框架中淘汰的讨论。MSDN文档似乎暗示Clone方法是进行的深拷贝，但是文档没有明确的说明：

ICloneable接口包含一个成员方法，Clone，意在支持超过MemberWiseClone所提供的功能... MemberWiseClone进行的是浅拷贝...

类型安全的克隆

ICloneable的另一个缺点是Clone方法返回的是一个对象，因此每次调用Clone都要进行一次强制类型转换。

```
Person joe = new Person(); 
joe.Name = "Joe Smith"; 
Person joeClone = (Person)joe.Clone();
```


一种可以避免进行强制类型转换的方式是提供你自己的类型安全的Clone方法。注意，你依然要提供ICloneable.Clone方法的以满足iCloneable接口的要求。

```
public class Person : ICloneable 
{ 
    public string Name; 
    object ICloneable.Clone() 
    { 
        return this.Clone(); 
    } 
    public Person Clone() 
    { 
        return (Person)this.MemberwiseClone(); 
    } 
}
```

1. 手工克隆

一个能够保证对象完全按照你所想的那样进行克隆的方式是手工克隆对象的每一个域（field）。这种方式的缺点是麻烦而且容易出错：如果你在类中增 加了一个域，你很可能会忘记更新Clone方法。还要在克隆引用对象指向原始对象的时候，注意避免无限循环引用。下面是一个进行深拷贝的简单例子：

```
public class Person : ICloneable 
{ 
    public string Name; 
    public Person Spouse; 
    public object Clone() 
    { 
        Person p = new Person(); 
        p.Name = this.Name; 
        if (this.Spouse != null) 
            p.Spouse = (Person)this.Spouse.Clone(); 
        return p; 
    } 
}
```

2. 使用MemberWiseClone方法

MemberWiseClone是Object类的受保护方法，能够通过创建一个新对象，并把所有当前对象中的非静态域复制到新对象中，从而创建一 个浅拷贝。对于值类型的域，进行的是按位拷贝。对于引用类型的域，引用会被赋值而引用的对象则不会。因此，原始对象及其克隆都会引用同一个对象。注意，这 种方法对派生类都是有效的，也就是说，你只需在基类中定义一次Clone方法。下面是一个简单的例子：

```
public class Person : ICloneable 
{ 
    public string Name; 
    public Person Spouse; 
    public object Clone() 
    { 
        return this.MemberwiseClone(); 
    } 
}
```

3. 用反射进行克隆

用反射进行克隆是使用Activator.CreateInstance方法来创建一个相同类型的新对象，然后用反射对所有域进行浅拷贝。这种方法 的优点是它是全自动的，不需要在对象中添加或删除成员的时候修改克隆方法。另外它也能被写成提供深拷贝的方法。缺点是使用了反射，因此会比较慢，而且在部 分受信任的环境中是不可用的。示例代码

```
private static TOut TransReflection<TIn, TOut>(TIn tIn)
        {
            TOut tOut = Activator.CreateInstance<TOut>();
            var tInType = tIn.GetType();
            foreach (var itemOut in tOut.GetType().GetProperties())
            {
                var itemIn = tInType.GetProperty(itemOut.Name); ;
                if (itemIn != null)
                {
                    itemOut.SetValue(tOut, itemIn.GetValue(tIn));
                }
            }
            return tOut;
        }
```



**调用一百万次耗时：2464毫秒**

4. 使用序列化进行克隆

克隆一个对象的最简单的方法是将它序列化并立刻反序列化为一个新对象。和反射方法一样，序列化方法是自动的，无需在对对象成员进行增删的时候做出修 改。缺点是序列化比其他方法慢，甚至比用反射还慢，所有引用的对象都必须是可序列化的（Serializable）。另外，取决于你所使用的序列化的类型 （XML，SOAP，二进制）的不同，私有成员可能不能像期望的那样被克隆。示例代码在这里，这里和这里。

```
StudentSecond ss= JsonConvert.DeserializeObject<StudentSecond>(JsonConvert.SerializeObject(s));
```

**调用一百万次耗时：**2984毫秒



http://wiki.unity3d.com/index.php/ObjectCopier

```
using System;
using System.Collections;
using System.Runtime.Serialization;
using System.Runtime.Serialization.Formatters.Binary;
using System.IO;
 
/// <summary>
/// Reference Article http://www.codeproject.com/KB/tips/SerializedObjectCloner.aspx
/// 
/// Provides a method for performing a deep copy of an object.
/// Binary Serialization is used to perform the copy.
/// </summary>
public static class ObjectCopier
{
    /// <summary>
    /// Perform a deep Copy of the object.
    /// </summary>
    /// <typeparam name="T">The type of object being copied.</typeparam>
    /// <param name="source">The object instance to copy.</param>
    /// <returns>The copied object.</returns>
    public static T Clone<T>(this T source)
    {
        if (!typeof(T).IsSerializable)
        {
            throw new ArgumentException("The type must be serializable.", "source");
        }
 
        // Don't serialize a null object, simply return the default for that object
        if (Object.ReferenceEquals(source, null))
        {
            return default(T);
        }
 
        IFormatter formatter = new BinaryFormatter();
        Stream stream = new MemoryStream();
        using (stream)
        {
            formatter.Serialize(stream, source);
            stream.Seek(0, SeekOrigin.Begin);
            return (T)formatter.Deserialize(stream);
        }
     }
}
```



5. 使用IL进行克隆

一种罕见的解决方案是使用IL（中间语言）来进行对象克隆。这种方式创建一个动态方法（DynamicMethod），获取中间语言生成器 （ILGenerator），向方法中注入代码，把它编译成一个委托，然后执行这个委托。委托会被缓存，因此中间语言只在初次克隆的时候才会生成，后续的 克隆都不会重新生成一遍。尽管这种方法比使用反射快，但是这种方法难以理解和维护。示例代码

```
　 public static class TransExpV2<TIn, TOut>
    {

        private static readonly Func<TIn, TOut> cache = GetFunc();
        private static Func<TIn, TOut> GetFunc()
        {
            ParameterExpression parameterExpression = Expression.Parameter(typeof(TIn), "p");
            List<MemberBinding> memberBindingList = new List<MemberBinding>();

            foreach (var item in typeof(TOut).GetProperties())
            {
　　　　　　　　　if (!item.CanWrite)
　　　　　　　　　　    continue;

                MemberExpression property = Expression.Property(parameterExpression, typeof(TIn).GetProperty(item.Name));
                MemberBinding memberBinding = Expression.Bind(item, property);
                memberBindingList.Add(memberBinding);
            }

            MemberInitExpression memberInitExpression = Expression.MemberInit(Expression.New(typeof(TOut)), memberBindingList.ToArray());
            Expression<Func<TIn, TOut>> lambda = Expression.Lambda<Func<TIn, TOut>>(memberInitExpression, new ParameterExpression[] { parameterExpression });

            return lambda.Compile();
        }

        public static TOut Trans(TIn tIn)
        {
            return cache(tIn);
        }

    }
```

调用：**StudentSecond ss= TransExpV2.Trans(s);**

**调用一百万次耗时：107毫秒\**

6. 使用扩展方法进行克隆

Havard Stranden用扩展方法（extention method）创建了一个自定义的克隆框架。这个框架能够创建对象及其引用的对象的深拷贝，不管对象结构有多复杂。缺点是，这是一个不提供源代码的自定义 框架（更新：现在已经包括源代码了，参见本文评论），并且它不能在不使用无参数构造器的时候，拷贝由私有方法创建的对象。另一个问题，也是所有自动化的深 克隆方法共有的问题是，深拷贝通常需要灵活地处理不能进行简单自动化特殊情况（例如未受管理的资源）。



### 一个测试类

```

using Microsoft.VisualStudio.TestTools.UnitTesting;
using System.Collections.Generic;
using System.Linq;
using System;
namespace dotnet_samples_test
{

    public class Person : ICloneable 
    {
        public IdInfo IdInfo;
        public int Age { get; set; }
        public string Name { get; set; }
        public string Address { get; set; }
        public List<string> Phones { get; set; } = new List<string>();
        public object Clone()
        {
            return this.MemberwiseClone();
        }

        public Person ShallowCopy()
        {
            return (Person)this.Clone();
        }

        public Person DeepCopy()
        {
            var result =  (Person)this.Clone();
            result.IdInfo = result.IdInfo.DeepCopy();
            result.Phones = result.Phones.ToList();
            return result;
        }
    }


    public class IdInfo
    {
        public int IdNumber;
        public IdInfo(int IdNumber)
        {
            this.IdNumber = IdNumber;
        }

        public object Clone()
        {
            return this.MemberwiseClone();
        }

        public IdInfo ShallowCopy()
        {
            return (IdInfo)this.Clone();
        }

        public IdInfo DeepCopy()
        {
            var result =  (IdInfo)this.Clone();

            return result;
        }

    }





    [TestClass]
    public class CloneTest1
    {
        [TestMethod]
        public void TestMethod1()
        {
            var person1 = new Person
            {
                Name = "長澤雅美",
                Age = 30,
                Address = "日本静岡縣磐田市",
                Phones = new List<string> { "9", "1", "1" },
                IdInfo = new IdInfo(1)
            };
            var person2 = person1.ShallowCopy();
            person2.IdInfo.IdNumber = 2;
            person2.Phones.RemoveAt(0);
            person2.Name="冈本伊朗";
            Console.WriteLine("person1的名字:"+person1.Name);
            Console.WriteLine("person1第一个数字是:"+person1.Phones[0]);
            Console.WriteLine("person1id是:"+person1.IdInfo.IdNumber);

                      Console.WriteLine("person2的名字:"+person2.Name);
            Console.WriteLine("person2第一个数字是:"+person2.Phones[0]);
            Console.WriteLine("person2id是:"+person2.IdInfo.IdNumber);
            Assert.AreEqual(person1.IdInfo.IdNumber,person2.IdInfo.IdNumber);
        }

        [TestMethod]
        public void TestMethod2()
        {
            var person1 = new Person
            {
                Name = "長澤雅美",
                Age = 30,
                Address = "日本静岡縣磐田市",
                Phones = new List<string> { "9", "1", "1" },
                IdInfo = new IdInfo(1)
            };
            var person2 = person1.DeepCopy();
            person2.IdInfo.IdNumber = 2;
            person2.Phones.RemoveAt(0);
            
            Console.WriteLine("person1的名字:"+person1.Name);
            Console.WriteLine("person1第一个数字是:"+person1.Phones[0]);
            Console.WriteLine("person1id是:"+person1.IdInfo.IdNumber);

            Console.WriteLine("person2的名字:"+person2.Name);
            Console.WriteLine("person2第一个数字是:"+person2.Phones[0]);
            Console.WriteLine("person2id是:"+person2.IdInfo.IdNumber);

            Assert.AreEqual(person1.IdInfo.IdNumber,person2.IdInfo.IdNumber);
        }
    }
}

```





