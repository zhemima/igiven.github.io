---
title : "c#特性参数"
---



可以通过反射获取这些特性然后做处理

```
class Program
{
    static void Main(string[] args)
    {
        var message = new MessageData {

            Header="header...",
            Body="body....",
            Footer="footer...",
        };

        Type objT = typeof(Program);
        Type fromBodyT = typeof(FromBodyAttribute);
        MethodInfo method = objT.GetMethod("Test");

        ParameterInfo[] paramsInfo = method.GetParameters();
        var parameters= new List<object>(paramsInfo.Length);
        foreach (ParameterInfo parameterInfo in paramsInfo)
        {
            var parameter = new object();
            if (parameterInfo.CustomAttributes.Any(i => i.AttributeType == fromBodyT))
                parameter = message.Body;
            parameters.Add(parameter);
        }


        object result = method.Invoke(null, parameters.ToArray());
        Console.WriteLine(result);


    }
    public class FromBodyAttribute : Attribute
    {
    }
    public static string Test([FromBody] string body)
    {
        return body;
    }
    class MessageData
    {

        public string Body { get; set; }
        public string Header { get; set; }
        public string Footer { get; set; }

    }

}
```

