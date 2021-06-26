---
title : "Converting between Structs and Byte Arrays"
---

In object-oriented code bases, we tend to express most of not all of our data in highly semantic and contextual ways – that is, we use classes that contain both data and behaviour, and often even more information through inheritance, attributes, and more.

However, sometimes we need to extract the data contained in these types – for example for sending network messages, or saving to disk. In this post we will look into converting between structs and byte arrays, to make exactly this possible.

We will compare different ways of doing so, and analyse them for performance and easy of use.



## Why byte arrays?

The reason we will be looking into serialising our data into byte arrays is because these are essentially the most fundamental data storage format. We can easily write them to a network buffer or stream, or to a file.

There are many alternatives and none of them will be right for every single use case. For example, I like to use JSON files to store settings, scripts, and text-based assets, as well as asset meta data.

In those cases performance is not the most important consideration. Instead it is more valuable to be able to edit and review files easily inside a text editor.

In situations where performance is important however – such as the mentioned networking or compact file storage – serialising only the relevant data itself and skipping the encoding and formatting inherent to clear text files can be key.

## Why structures?

There are several reasons for why we are talking about structures specifically.

First, I want to make a clear distinction between data and behaviour, by using a type that contains exactly the data we are interested in serialising.

Secondly, structures are much more reliable and controllable when it comes to binary data layout. We will see how this is important for our last method of serialisation.

Overall, we can use structures to directly represent the data that will be written into our byte array. For our example of networking this means that we have a clear one to one correspondence between our structures and our network messages.

## `BinaryFormatter`

In our first method of converting between structs and byte arrays, we will make use of .NET’s [BinaryFormatter](https://msdn.microsoft.com/en-us/library/system.runtime.serialization.formatters.binary.binaryformatter(v=vs.110).aspx) class.

The entire purpose of that class is to serialise an object into binary format (i.e. a byte array) – as well as deserialising the same back into objects.

The class offers a lot of functionality – most of which we are not interested in here. Of interest to us are only two methods: `Serialize()` and `Deserialize()`.

These methods allow us to read/write our data to any stream. In many cases we could use this to write to a network – or file – buffer or stream directly. For our purpose – and for ease of testing, we will use the `MemoryStream` class which is little more than a stream wrapper around a byte array in the first place.

Here are two generic methods that do exactly this:

```
public static byte[] Serialize<T>(T data)
    where T : struct
{
    var formatter = new BinaryFormatter();
    var stream = new MemoryStream();
    formatter.Serialize(stream, data);
    return stream.ToArray();
}
public static T Deserialize<T>(byte[] array)
    where T : struct
{
    var stream = new MemoryStream(array);
    var formatter = new BinaryFormatter();
    return (T)formatter.Deserialize(stream);
}
```

These methods can now be easily used like this:

```
[Serializable] // BinaryFormatter needs this attribute
struct MyStruct
{
    // some fields here
}

var data = new MyStruct();
var bytes = Serialize(data);
var data2 = Deserialize<MyStruct>(bytes);
// data and data2 now contain the same values
```

This looks great!

It seems that we have found a solution that is both easy to use, and requires almost no work if we want to expand it. Allowing for the conversion of new structs simple requires the addition of the `Serializable` attribute, while we have to do nothing at all if we modify our structures to include more, less, or different data.

### Performance

How about performance?

I wrote a little test that both serialises and deserialises hundreds of thousands of times, and repeats that process several times to make sure we get accurate results. You can find the full code of it [on my GitHub](https://github.com/amulware/genericgamedev-tests/tree/master/src/StructByteArrayConversion)

Here are the results:

Using `BinaryFormatter`,
– converting a 16 byte structs to an array one million times takes 4.86 seconds;
– converting an array to a 16 byte struct one million times takes 3.85 seconds.

This means that a single call to either of our methods takes less than 5 microseconds.

That is pretty good!

With this performance we can easily write and read thousands of networking messages per second before we will notice the performance impact. That is easily enough for most games and other real-time applications.

There is another kind of performance measurement that is important however – and especially so when it comes to networking: Bandwidth.

The struct I ran the tests with consisted of exactly 16 bytes. That means that in principle we should be able to write it into a byte array with length 16. The BinaryFormatter however – and this is related to the other features it has – writes a total of 218 bytes to the array.

Suffice it to say: That is a whole lot more.

The advantage of this is, that the object that is deserialised will actually be of the correct type. I merely made the method generic so that we could perform the cast and return the structure boxed in the result of `BinaryFormatter.Deserialize()`.

There are cases were we care less about the amount of data, and prefer to handle our data in this way. For this post however, I want to find a method that results in an array as small as possible.

## `BinaryWriter`/`BinaryReader`

Our second case study will be two other .NET classes: [BinaryWriter](https://msdn.microsoft.com/en-us/library/system.io.binaryreader(v=vs.110).aspx) and [BinaryReader](https://msdn.microsoft.com/en-us/library/system.io.binarywriter(v=vs.110).aspx)

These classes are much simpler. They do little more than allowing us to write and read primitive types like integers and booleans to and from an arbitrary stream.

This means that we cannot write the entire structure to our stream with just a single line of code any more. Instead we need to write and read all fields manually:

```
struct MyStruct
{
    int anInteger;
    float aFloat;
    long aLong;

    public byte[] ToArray()
    {
        var stream = new MemoryStream();
        var writer = new BinaryWriter(stream);

        writer.Write(this.anInteger);
        writer.Write(this.aFloat);
        writer.Write(this.aLong);

        return stream.ToArray();
    }

    public static MyStruct FromArray(byte[] bytes)
    {
        var reader = new BinaryReader(new MemoryStream(bytes));

        var s = default(MyStruct);

        s.anInteger = reader.ReadInt32();
        s.aFloat = reader.ReadSingle();
        s.aLong = reader.ReadInt64();

        return s;
    }
}
```

Using these methods is similarly easy to the ones above:

```
var data = new MyStruct();
var array = data.ToArray();
var data2 = MyStruct.FromArray(array);
```

In this case, the returned array is indeed exactly 16 bytes long.

### Performance

But how does this manual approach measure up in performance?

Very well!

In my test, the times for serialising and deserialising went from 4.86 and 3.85 down to 0.50 and 0.20 seconds respectively (again for one million conversions each).

It turns out this approach is not only space efficient, but it is also around ten times faster than the previous one – seemingly no reason to look back!

In fact, there is an optimisation we can make to increase performance even further: We do not have to create new `MemoryStream`s and `BinaryReader`/`BinaryWriter`s for each method call. Instead we can reuse them – either by having static ones (watch out for thread-safety!) or by keeping them in whatever object manages for example our network traffic.

Doing so drops my measured time down to 0.14 and 0.11 seconds respectively.

Note that the same optimisation can be applied to the first method. However – while positive – the performance increase is much less than in this case, relative to the overall much worse time.

### Some concerns

If we do look back to the code however, note how if we add another structure that we would like to serialise, we have to add the two methods to it, and adapt them to its fields.

Further, if we change one of our structures, we have to make sure to reflect that change in both of these methods. We are bound to forget – especially when adding a new field – which could easily result in a small debugging nightmare.

Ideally we can find a solution that is fast, uses little space, and does not require us to continuously maintain our serialisation code.

## Marshalling

The last approach we will take a look at is that of marshalling.

Marshalling refers to using both managed and unmanaged data and the transfer between them. By default, any object created in C# lives in managed memory, which has a lot of advantages – such as automatic garbage collection. Using unmanaged memory on the other hand is more difficult in C#, and requires us to allocate and free space manually. If we forget to do so, we may cause memory leaks that will eventually cause our application to crash.

All of the functionality we are interested in can be found in the static [Marshal](https://msdn.microsoft.com/en-us/library/system.runtime.interopservices.marshal(v=vs.100).aspx) class.

Specifically, we will use:

- `Marshal.SizeOf()`
  to determine the byte size of our structs;
- `Marshal.AllocHGlobal()`
  to allocate unmanaged memory;
- `Marshal.StructureToPtr()`
  to marshal (copy) our structure to the allocated unmanaged memory;
- `Marshal.PtrToStructure()`
  to marshal (copy) from unmanaged memory back to our structure;
- `Marshal.Copy()`
  to copy between the unmanaged memory and our byte array;
- `Marshal.FreeHGlobal()`
  to free the allocated memory;

Using these methods we can construct the following methods:

```
public static byte[] Serialize<T>(T s)
    where T : struct
{
    var size = Marshal.SizeOf(typeof(T));
    var array = new byte[size];
    var ptr = Marshal.AllocHGlobal(size);
    Marshal.StructureToPtr(s, ptr, true);
    Marshal.Copy(ptr, array, 0, size);
    Marshal.FreeHGlobal(ptr);
    return array;
}

public static T Deserialize<T>(byte[] array)
    where T : struct
{
    var size = Marshal.SizeOf(typeof(T));
    var ptr = Marshal.AllocHGlobal(size);
    Marshal.Copy(array, 0, ptr, size);
    var s = (T)Marshal.PtrToStructure(ptr, typeof(T));
    Marshal.FreeHGlobal(ptr);
    return s;
}
```

Note that due to the unsafety of using unmanaged memory, we may want to use a [try – finally](https://msdn.microsoft.com/en-us/library/zwc8s4fz.aspx) block to make sure the memory will always be freed, even if something goes wrong. For brevity, this is left out here.

When testing our code, which again is as easy to use as before – in fact the method signatures are exactly the same as our first pair – we see that it indeed works as we hope.

Without any code inside our structures and completely generic methods there is no need to write or maintain any code when adding or modifying structures.

Further the resulting array is the expected 16 bytes long.

### Performance

When running these methods through the tests, converting structs to byte arrays and vice versa takes a mere 0.47 and 0.60 seconds respectively (again for one million calls).

## Comparison

Here is a table with the results from the performance tests:

| 16 byte struct                     | struct to array | array to struct |
| :--------------------------------- | :-------------- | :-------------- |
| binary formatter (218 byte array!) | 4.86s           | 3.85s           |
| binary writer/reader               | 0.50s           | 0.20s           |
| **binary w/r (singleton)**         | **0.14s**       | **0.11s**       |
| marshalling                        | 0.47s           | 0.60s           |

Clearly, if what we care about most is performance, writing and reading our data manually, using shared `BinaryWriter` and `BinaryReader` objects is the fastest method.

On the other hand, the `BinaryFormatter` and `Marshal` methods allow for much easier reuse and make our code significantly more robust to change since there is no code to update and maintain.

To provide some more data, here are the results from the same test, but this time with a 128 byte structure:

| 128 byte struct                    | struct to array | array to struct |
| :--------------------------------- | :-------------- | :-------------- |
| binary formatter (218 byte array!) | 17.32s          | 14.47s          |
| binary writer/reader               | 1.48s           | 0.66s           |
| **binary w/r (singleton)**         | **0.78s**       | **0.56s**       |
| marshalling                        | 0.84s           | 0.75s           |

We can see that the relative ordering of the different measurements is still the same. However, note how the `BinaryWriter` and `BinaryReader` measurements are getting significantly closer to the `Marshal` ones.

While I would not necessarily encourage structures of this size, I would argue that marshalling is the best approach for large structures in almost every case. The slightly slower performance is easily justified by the much more maintainable code.

In fact, I would go as far and say that even for small structures the ease of using marshalling is still top advantageous despite the lower performance.

In the end, binary serialization is unlikely to ever be a bottleneck, and unless it is, we should choose the option that fulfils our requirements of small array size and ease of use.

## Conclusion

We took a look at three – and a half – different ways of converting between structs and byte arrays.

Judging by memory usage, performance, and ease of use, there is no clear winner – only a clear loser unless we specifically need the additional functionality of `BinaryFormatter`.

However, in marshalling we found a method that is reasonably fast, while acing our other requirements.

While in extremely performance critical code we may want to write our data manually, marshalling is likely the best alternative in the vast majority of cases.

Feel free to let me know if you agree with this analysis, or if you have other methods of achieving the same result that may be worth looking into.

Enjoy the pixels!