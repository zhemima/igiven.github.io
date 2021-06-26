---
title:  "DatetimeOffset和Datetime的区别"
---


```
  [Fact]
        public void TestDatetimeoffset2()
        {
            var a = DateTimeOffset.Now;
            var b = a.ToLocalTime();
            var c = a.ToUniversalTime();

            Assert.Equal(b, c);
        }
```

上面的abc都是一个值



```
    [Fact]
        public void TestDatetimeoffset3()
        {
            var a = DateTime.Now;
            var b = a.ToLocalTime();
            var c = a.ToUniversalTime();

            Assert.Equal(b, c);
        }
```

上面的值是不等的



datetimeoffset使用efcore存储到datetime字段都是0时区的...取出来后如果显示再web可以toLocalTime

如果是游戏中使用,除非是发送给用户显示..否则不需要toLocalTime..



1，DateTime

表示时间上的一刻，通常以日期和当天时间来表示。

2， DateTimeOffset

表示一个时间点，通常以相对于协调世界时（UTC）的日期和时间来表示



 https://docs.microsoft.com/en-us/dotnet/standard/datetime/performing-arithmetic-operations 



```

using System;

public enum TimeComparison
{
   EarlierThan = -1,
   TheSameAs = 0,
   LaterThan = 1
}

public class DateManipulation
{
   public static void Main()
   {
      DateTime localTime = DateTime.Now;
      DateTime utcTime = DateTime.UtcNow;
      
      Console.WriteLine("Difference between {0} and {1} time: {2}:{3} hours", 
                        localTime.Kind.ToString(), 
                        utcTime.Kind.ToString(), 
                        (localTime - utcTime).Hours, 
                        (localTime - utcTime).Minutes);
      Console.WriteLine("The {0} time is {1} the {2} time.", 
                        localTime.Kind.ToString(), 
                        Enum.GetName(typeof(TimeComparison), localTime.CompareTo(utcTime)), 
                        utcTime.Kind.ToString());  
   }
}
// If run in the U.S. Pacific Standard Time zone, the example displays 
// the following output to the console:
//    Difference between Local and Utc time: -7:0 hours
//    The Local time is EarlierThan the Utc time.      


public class DateTimeOffsetManipulation
{
   public static void Main()
   {
      DateTimeOffset localTime = DateTimeOffset.Now;
      DateTimeOffset utcTime = DateTimeOffset.UtcNow;
      
      Console.WriteLine("Difference between local time and UTC: {0}:{1:D2} hours", 
                        (localTime - utcTime).Hours, 
                        (localTime - utcTime).Minutes);
      Console.WriteLine("The local time is {0} UTC.", 
                        Enum.GetName(typeof(TimeComparison), localTime.CompareTo(utcTime)));  
   }
}
// Regardless of the local time zone, the example displays 
// the following output to the console:
//    Difference between local time and UTC: 0:00 hours.
//    The local time is TheSameAs UTC.


```

 从实例中可以看出，DateTimeOffset是取相对于UTC的日期和时间来表示的，所以DateTimeOffset.Now和DateTimeOffset.UtcNow的值是一样的。而DateTime不同，是以日期和当前时间来显示的。 



就是说..如果你用DateTime进行加减运算要么只用DateTime.Now..要么只用DateTime.UtcNow..混用会出现不可预期的错误



而DateTimeOffset的无论是用now还是utcnow加减都一样,因为他代表的是一个时间点...如果是为了客户端显示使用可以转换成本地时间再tostring



