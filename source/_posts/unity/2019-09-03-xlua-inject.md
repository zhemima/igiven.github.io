---
title:  "xlua注入源码解读"
---

- **让C#代码支持热更的流程**

1. Generate Code
   这一步主要根据是根据C#类中需要支持热更的方法生成其对应的委托方法，但是并不是每个方法对应一个委托，而是根据调用参数和返回参数公用委托。
2. Hotfix Inject
   这一步主要是对Unity编译出的Dll中的C#类添加判断条件，以此来选择调用Lua中的修复方法还是直接执行C#代码

- **Generate Code 实现和生成结果**
  在Gen代码阶段，对热更生效的部分逻辑，基本就是根据之前在代码中标记了HotfixAttribute的类进行收集，然后使用XLua的模板代码生成器，生成对应名为DelegateBridge类。__Gen_Delegate_Imp这个就是到时候要重复使用到的映射到Lua中function的委托。

```csharp
public void __Gen_Delegate_Imp0()
{
    RealStatePtr L = luaEnv.rawL;
    int errFunc = LuaAPI.pcall_prepare(L, errorFuncRef, luaReference);
    PCall(L, 0, 0, errFunc);
    LuaAPI.lua_settop(L, errFunc - 1);
}
```

生成的代码就是先设置errorFuncRef（异常回调），luaReference（Lua方法）。如果在XLua中设置了热更修复代码，那么就会在C#中生成一个DelegateBridge，而其luaReference的指向就是Lua中的方法，所以这个只能调用指定的__Gen_Delegate_Imp，调用其他会报错。

- **Hotfix Inject**
  这一步是在Unity为C#代码生成完对应dll之后，由XLua再来对dll注入一些判断条件式来完成是否进行Lua调用的行为。
  判断方法很简单，检查对应类静态字段是否有DelegateBridge对象。
  实现如下：

```csharp
bool injectMethod(MethodDefinition method, HotfixFlagInTool hotfixType)
{
            var type = method.DeclaringType;
            bool isFinalize = (method.Name == "Finalize" && method.IsSpecialName);
            //__Gen_Delegate_Imp 方法引用
            MethodReference invoke = null;
            int param_count = method.Parameters.Count + (method.IsStatic ? 0 : 1);
            //根据返回值和参数个数类型，查找对应的委托方法
            if (!findHotfixDelegate(method, out invoke, hotfixType))
            {
                Error("can not find delegate for " + method.DeclaringType + "." + method.Name + "! try re-genertate code.");
                return false;
            }

            if (invoke == null)
            {
                throw new Exception("unknow exception!");
            }
            
            invoke = injectAssembly.MainModule.Import(invoke);
            //插入的类静态字段，用来标记对应的方法是否有对应的Lua注入
            FieldReference fieldReference = null;
            //方法中的变量定义
            VariableDefinition injection = null;
            bool isIntKey = hotfixType.HasFlag(HotfixFlagInTool.IntKey) && !type.HasGenericParameters && isTheSameAssembly;
            //isIntKey = !type.HasGenericParameters;

            if (!isIntKey)
            {
                //新建变量，加入方法体的变量组中
                injection = new VariableDefinition(invoke.DeclaringType);
                method.Body.Variables.Add(injection);
                //获取这个方法对应的委托名，因为有重载方法存在，所以之前已经注入的过的方法会在这边获取时候计数加1，
                //比如第一个重载获取的是__Hotfix0，那么下一个重载会是__Hotfix1，判断是否注入就是是否设置对应FieldReference。
                var luaDelegateName = getDelegateName(method);
                if (luaDelegateName == null)
                {
                    Error("too many overload!");
                    return false;
                }
                //创建对应的静态Field名字就是上面取到的luaDelegateName 
                FieldDefinition fieldDefinition = new FieldDefinition(luaDelegateName, Mono.Cecil.FieldAttributes.Static | Mono.Cecil.FieldAttributes.Private,
                    invoke.DeclaringType);
                type.Fields.Add(fieldDefinition);
                fieldReference = fieldDefinition.GetGeneric();
            }

            bool ignoreValueType = hotfixType.HasFlag(HotfixFlagInTool.ValueTypeBoxing);
            //IL插入位置，现在定位的是方法体的第一行
            var insertPoint = method.Body.Instructions[0];
            //获取IL处理器
            var processor = method.Body.GetILProcessor();
            //构造函数的处理逻辑先跳过这边不做分析
            if (method.IsConstructor)
            {
                insertPoint = findNextRet(method.Body.Instructions, insertPoint);
            }

            Dictionary<Instruction, Instruction> originToNewTarget = new Dictionary<Instruction, Instruction>();
            HashSet<Instruction> noCheck = new HashSet<Instruction>();
            
            while (insertPoint != null)
            {
                //isIntKey这边用到的是Xlua中的AutoIdMap，这边只对最基础的功能做分析，这边就分析基础的注入了。
                Instruction firstInstruction;
                if (isIntKey)
                {
                    firstInstruction = processor.Create(OpCodes.Ldc_I4, bridgeIndexByKey.Count);
                    processor.InsertBefore(insertPoint, firstInstruction);
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Call, hotfixFlagGetter));
                }
                else
                {
                    //创建第一条IL语句，获取类的静态Field压入方法栈中，其实就是之前luaDelegateName获取的字段
                    firstInstruction = processor.Create(OpCodes.Ldsfld, fieldReference);
                    //插入insertPoint之前
                    processor.InsertBefore(insertPoint, firstInstruction);
                    //创建并插入IL，获取栈顶的值并压入到对应的变量中，injection就是我们之前创建的新建变量
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Stloc, injection));
                    //创建并插入IL，压入变量体中的值到栈
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ldloc, injection));
                }
                //创建跳转语句，为false时候直接跳转insertPoint,
                //这边OpCodes.Brfalse看起来是布尔值判断，其实也会判断是否为null
                var jmpInstruction = processor.Create(OpCodes.Brfalse, insertPoint);
                processor.InsertBefore(insertPoint, jmpInstruction);

                if (isIntKey)
                {
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ldc_I4, bridgeIndexByKey.Count));
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Call, delegateBridgeGetter));
                }
                else
                {
                    //创建并插入IL,再次压入变量的值，因为上面做完判断后，栈顶的值就会被弹出，所以这边要再次压入
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ldloc, injection));
                }

                for (int i = 0; i < param_count; i++)
                {
                    if (i < ldargs.Length)
                    {
                        processor.InsertBefore(insertPoint, processor.Create(ldargs[i]));
                    }
                    else if (i < 256)
                    {
                        processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ldarg_S, (byte)i));
                    }
                    else
                    {
                        processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ldarg, (short)i));
                    }
                    if (i == 0 && !method.IsStatic && type.IsValueType)
                    {
                        processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ldobj, type));
                        
                    }
                    //对值类型进行Box
                    if (ignoreValueType)
                    {
                        TypeReference paramType;
                        if (method.IsStatic)
                        {
                            paramType = method.Parameters[i].ParameterType;
                        }
                        else
                        {
                            paramType = (i == 0) ? type : method.Parameters[i - 1].ParameterType;
                        }
                        if (paramType.IsValueType)
                        {
                            processor.InsertBefore(insertPoint, processor.Create(OpCodes.Box, paramType));
                        }
                    }
                }
                //创建并插入IL，调用invoke方法，因为之前已经压入injection的值，DelegateBridge的对象
                processor.InsertBefore(insertPoint, processor.Create(OpCodes.Call, invoke));
                //普通方法，加入返回操作
                if (!method.IsConstructor && !isFinalize)
                {
                    processor.InsertBefore(insertPoint, processor.Create(OpCodes.Ret));
                }

                if (!method.IsConstructor)
                {
                    break;
                }
                else
                {
                    originToNewTarget[insertPoint] = firstInstruction;
                    noCheck.Add(jmpInstruction);
                }
                insertPoint = findNextRet(method.Body.Instructions, insertPoint);
            }

            if (method.IsConstructor)
            {
                fixBranch(processor, method.Body.Instructions, originToNewTarget, noCheck);
            }

            if (isFinalize)
            {
                if (method.Body.ExceptionHandlers.Count == 0)
                {
                    throw new InvalidProgramException("Finalize has not try-catch? Type :" + method.DeclaringType);
                }
                method.Body.ExceptionHandlers[0].TryStart = method.Body.Instructions[0];
            }
            if (isIntKey)
            {
                bridgeIndexByKey.Add(method);
            }
            return true;
}
static string getDelegateName(MethodDefinition method)
{
            string fieldName = method.Name;
            if (fieldName.StartsWith("."))
            {
                fieldName = fieldName.Substring(1);
            }
            string ccFlag = method.IsConstructor ? "_c" : "";
            string luaDelegateName = null;
            var type = method.DeclaringType;
            for (int i = 0; i < MAX_OVERLOAD; i++)
            {
                string tmp = ccFlag + "__Hotfix" + i + "_" + fieldName;
                if (!type.Fields.Any(f => f.Name == tmp)) // injected
                {
                    luaDelegateName = tmp;
                    break;
                }
            }
            return luaDelegateName;
}
```

- **xlua.hotfix**在完成生成代码和注入后，只要在Lua中调用xlua.hotfix或util.hotfix*ex方法就可以实现C#代码热更了。*hotfix和hotfixex的区别在与是否可以调用原C#代码，其实ex的实现也是调用了hotfix，在下面将分析hotfix和hotfix_ex的实现原理。
  先分析下hotfix的Lua代码，代码在第一篇文章中的实例化lua中：

```text
     init_xlua.lua
     xlua.hotfix = function(cs, field, func)
                //判空
                if func == nil then func = false end
                //检查并且统一转化为table
                //因为在Xlua中可以一次传一个方法，或者一次传一组方法
                local tbl = (type(field) == 'table') and field or {[field] = func}
                //遍历需要hotfix的代码，key是方法名，v是对应的func
                for k, v in pairs(tbl) do
                    //构造函数的hotfix，这边不做分析了，原理一样
                    local cflag = ''
                    if k == '.ctor' then
                        cflag = '_c'
                        k = 'ctor'
                    end
                    //检查v的类型
                    local f = type(v) == 'function' and v or nil
                    //调用access函数，其在初始化注册，最终实现在C#中下文解析实现
                    xlua.access(cs, cflag .. '__Hotfix0_'..k, f) -- at least one
                    //尝试给重载方法也添加上function如果有重载的话
                    pcall(function()
                        for i = 1, 99 do
                            xlua.access(cs, cflag .. '__Hotfix'..i..'_'..k, f)
                        end
                    end)
                end
                //设置私有访问
                xlua.private_accessible(cs)
              end
```

**XLuaAccess在C#中的实现：**

```text
        //xlua.access(cs, cflag .. '__Hotfix0_'..k, f)

        public static int XLuaAccess(RealStatePtr L)
        {
            try
            {
                
                ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
                //获取对应的CS类Type
                Type type = getType(L, translator, 1);
                object obj = null;
                if (type == null && LuaAPI.lua_type(L, 1) == LuaTypes.LUA_TUSERDATA)
                {
                    obj = translator.SafeGetCSObj(L, 1);
                    if (obj == null)
                    {
                        return LuaAPI.luaL_error(L, "xlua.access, #1 parameter must a type/c# object/string");
                    }
                    type = obj.GetType();
                }

                if (type == null)
                {
                    return LuaAPI.luaL_error(L, "xlua.access, can not find c# type");
                }
                //将cflag .. '__Hotfix0_'..k 转为fieldName，这个字段就是之前Inject时候创建的类静态字段名
                string fieldName = LuaAPI.lua_tostring(L, 2);

                BindingFlags bindingFlags = BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.Static;
                //这边访问一定是Set所以后面就不分析了，这边就是反射获取对应的FieldInfo，重点在于translator.GetObject方法。
                if (LuaAPI.lua_gettop(L) > 2) // set
                {
                    var field = type.GetField(fieldName, bindingFlags);
                    if (field != null)
                    {
                        field.SetValue(obj, translator.GetObject(L, 3, field.FieldType));
                        return 0;
                    }
                    var prop = type.GetProperty(fieldName, bindingFlags);
                    if (prop != null)
                    {
                        prop.SetValue(obj, translator.GetObject(L, 3, prop.PropertyType), null);
                        return 0;
                    }
                }
                else
                {
                    ...
                }
                return LuaAPI.luaL_error(L, "xlua.access, no field " + fieldName);
            }
            catch (Exception e)
            {
                return LuaAPI.luaL_error(L, "c# exception in xlua.access: " + e);
            }
        }
        //为了减少篇幅，只展示必要代码
        public object GetObject(RealStatePtr L, int index, Type type)
        {
            int udata = LuaAPI.xlua_tocsobj_safe(L, index);

            if (udata != -1)
            {
                ...
            }
            else
            {
                //一些特殊值类型的返回
                ...
                return (objectCasters.GetCaster(type)(L, index, null));
            }
        }
        //为了减少篇幅，只展示必要代码
        public ObjectCast GetCaster(Type type)
       {
            ...
            ObjectCast oc;
            //缓存行为，继续分析第一次获取的caster委托
            if (!castersMap.TryGetValue(type, out oc))
            {
                oc = genCaster(type);
                castersMap.Add(type, oc);
            }
            return oc;
        }
        //为了减少篇幅，只展示必要代码
        private ObjectCast genCaster(Type type)
        {
            
            ObjectCast fixTypeGetter = (RealStatePtr L, int idx, object target) =>
            {
                if (LuaAPI.lua_type(L, idx) == LuaTypes.LUA_TUSERDATA)
                {
                    object obj = translator.SafeGetCSObj(L, idx);
                    return (obj != null && type.IsAssignableFrom(obj.GetType())) ? obj : null;
                }
                return null;
            }; 
            if ...
            //在Inject代码时候我们已经知道其字段类型就是DelegateBridge，所以会返回的就是这个表达式
            else if (typeof(DelegateBridgeBase).IsAssignableFrom(type))
            {
                return (RealStatePtr L, int idx, object target) =>
                {
                    //缓存行为，分析CreateDelegateBridge
                    object obj = fixTypeGetter(L, idx, target);
                    if (obj != null) return obj;

                    if (!LuaAPI.lua_isfunction(L, idx))
                    {
                        return null;
                    }

                    return translator.CreateDelegateBridge(L, null, idx);
                };
            }
        }
        //为了减少篇幅，只展示必要代码
        public object CreateDelegateBridge(RealStatePtr L, Type delegateType, int idx)
        {
            //这边只是查找是否这个方法已经有缓存了，有的话直接返回，没有就要Create，我们继续分析Create的事。
            LuaAPI.lua_pushvalue(L, idx);
            LuaAPI.lua_rawget(L, LuaIndexes.LUA_REGISTRYINDEX);
            //不为null就是已经生成过，就直接从缓存获取
            if (!LuaAPI.lua_isnil(L, -1))
            {
                retrun ...
            }
            else
            {
                //弹出刚刚查询过的nil值
                LuaAPI.lua_pop(L, 1);
            }
            //开始生成对应的DelegateBridge
            //压入idx对应的值，idx对应的值是Lua中的function
            LuaAPI.lua_pushvalue(L, idx);
            //获取function对应的引用id
            int reference = LuaAPI.luaL_ref(L);
            //再次压入idx对应的值，idx对应的值是Lua中的function
            LuaAPI.lua_pushvalue(L, idx);
            //压入function对应的引用id
            LuaAPI.lua_pushnumber(L, reference);
            //将栈顶的两个值以字典形式存在全局变量表中，这个的作用就是上面的代码查询是否已经在lua中缓存。
            LuaAPI.lua_rawset(L, LuaIndexes.LUA_REGISTRYINDEX);
            DelegateBridgeBase bridge;
            try
            {
                //创建DelegateBridge，注意这边的reference，这个就是对应了lua中修复的lua函数，
                //我们在Inject时候call的方法会使用到这个参数。
                bridge = new DelegateBridge(reference, luaEnv);
            }
            catch(Exception e)
            {
                //异常情况下的，清场操作
                LuaAPI.lua_pushvalue(L, idx);
                LuaAPI.lua_pushnil(L);
                LuaAPI.lua_rawset(L, LuaIndexes.LUA_REGISTRYINDEX);
                LuaAPI.lua_pushnil(L);
                LuaAPI.xlua_rawseti(L, LuaIndexes.LUA_REGISTRYINDEX, reference);
                throw e;
            }
            //因为hotfix时候，delegateType传进来是个null，后续代码不会被调用到，就不做分析了。
            if (delegateType == null)
            {
                //缓存到字典中，注意这里是弱引用所以会被回收，被回收后，需要从lua中查询到对应引用值，然后再生产。
                delegate_bridges[reference] = new WeakReference(bridge);
                return bridge;
            }
            ...
        }
```

这样在进行调用hotfix后，对应的要修复的类的静态字段就会被设置上对应的DelegateBridge对象，然后在C#代码执行到对应的需要热更修复的方法时候，会先执行我们注入的IL代码，检查是否有对应的DelegateBridge。那么就是调用DelegateBridge中对应的方法，方法中包含的reference就是Lua对应的function，这样就执行到了lua中去，实现了热更。

- **util.hotfix_ex的实现**
  其实现直白的来讲就是在调用util.hotfix_ex(functionB)时候，真正设置的是一个中间函数A，它被设置为对应方法的热更修复函数。
  在调用A进行热更时候，它先设置这个方法的热更方法为空，然后调用原先设置的functionB，当functionB调用完后，然后再设置回热更方法为A，那么就能实现在热更修复方法functionB中调用原先的方法。
  因为设置这些参数都是带反射的，所以在高频场景是有性能消耗的。
  代码实现如下：

```lua
local function hotfix_ex(cs, field, func)
    --断言，检查参数
    assert(type(field) == 'string' and type(func) == 'function', 'invalid argument: #2 string needed, #3 function needed!')
    --创建中间函数，就是上文提到的A
    local function func_after(...)
        --先将需要热更修复的方法设置为nil，那么再调用方法时候会执行的就是之前方法
        xlua.hotfix(cs, field, nil)
        --执行func，就是上文提到的functionB
        local ret = {func(...)}
        ---重新将需要热更修复的方法设置为中间函数A
        xlua.hotfix(cs, field, func_after)
        return unpack(ret)
    end
    --设置需要热更修复为中间函数A
    xlua.hotfix(cs, field, func_after)
end
```

- **结束语**整个Hotfix的实现也分析完了，后续的文章将继续分析，XLua中的各种优化技巧实现，比如无GC传值，模板生成技术。