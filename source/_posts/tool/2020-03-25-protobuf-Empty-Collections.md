---
title : "protobuf序列化的一些问题"
---

# Base / Derived Classes

Each derived class must have its base class marked with [ProtoInclude(<num>, typeof(ProtoBuff-Derived-Class))]. If not, all values will be NULL.

```
[ProtoContract]
[ProtoInclude(100, typeof(HomeFolders))]
[ProtoInclude(200, typeof(PublicFolders))]
public class Folders
{
  [ProtoMember(1)]
  public int ProtoMember1 { get; set; }

  [ProtoMember(2)]
  public int ProtoMember2 { get; set; }
}

[ProtoContract]
public class HomeFolders : Folders
{
  [ProtoMember(1)]
  public int ProtoMember4 { get; set; }
}

[ProtoContract]
public class PublicFolders : Folders
{
  [ProtoMember(1)]
  public int ProtoMember5 { get; set; }
}
```

# Avoid duplicate property tags

Using the same number for ProtoInclude and ProtoMember will generate an error about duplicate property tags. The example below is **NOT** correct.

```
[ProtoContract]
[ProtoInclude(1, typeof(PublicFolders))]
public class Folders
{
  [ProtoMember(1)]
  public int ProtoMember1 { get; set; }
}
```

So you need to use a different number for ProtoInclude. Corrected example:

```
[ProtoContract]
[ProtoInclude(100, typeof(PublicFolders))]
public class Folders
{
  [ProtoMember(1)]
  public int ProtoMember1 { get; set; }
}
```

# Null vs. Empty Collections

ProtoBuf does not understand the difference between a collection (List, IEnumerable etc) being null versus empty (zero count). For example, if you put these objects into the cache,

```
List<int> list1 = new List<int>();
List<int> list2 = null;
```

after deserialization, both the lists will have the same value—that is NULL. There are two ways to solve this:

1. Using a private field (we are using this):

   ```
   [ProtoMember(12, OverwriteList = true)]
   private List _publicFolders;
   public List publicFolders
   {
     get
     {
       if (_publicFolders == null)
       {
         _publicFolders = new List();
       }
       return _publicFolders;
     }
     set
     {
       _publicFolders = value;
     }
   }
   ```

2. Using the OnDeserialized attribute:

   ```
   [ProtoMember(2, OverwriteList = true)]
   private PublicFolder[] publicFolders;
   [ProtoMember(3, OverwriteList = true)]
   private PrivateFolder[] privateFolder;
   [ProtoMember(4, OverwriteList = true)]
   private SecureFolder[] secureFolder;
   
   [OnDeserialized]
   private void HandleSerializationMismatch(StreamingContext context)
   {
     publicFolders = publicFolders ?? new PublicFolders[0];
     privateFolder = privateFolder ?? new PrivateFolder[0];
     secureFolder = secureFolder ?? new SecureFolder[0];
   }
   ```

# Things to Remember

ProtoBuf ignores properties if the class inherits from a collection and the Items property for that collection is null. Example:

```
public class Folders : List
{
  public int value1 { get; set; }

  public int value2 { get; set; }
}

Folders folders = new Folders() { value1 = 5; value2 = 6; };
```

After deserialization, the value of the Folders object will be NULL, because the count of items on is 0.

Classes that inherit from special collections are also not supported.

```
public class Folders : ReadOnlyCollection
{
  public int value1 { get; set; }

  public int value2 { get; set; }
}

Folders folders = new Folders() { value1 = 5; value2 = 6; };
```

# AllowParseableTypes

AllowParseableTypes is a global switch that determines whether types with “.ToString()” and “Parse(string)” methods should be serialized as strings. We can use this setting for types that can’t be marked in the ProtoContract but can be parseable.

```
static ProtoBufClient()
{
  RuntimeTypeModel.Default.AllowParseableTypes = true;
}
```

For example, to solve the serialization problem with the Version type:

```
[Serializable]
[ProtoContract(SkipConstructor = true)]
[ProtoInclude(100, typeof(PrivateFolder))]
[ProtoInclude(200, typeof(PublicFolder))]
[ProtoInclude(300, typeof(SecureFolder))]
public abstract class FolderBase : Folder
{
  ...

  [ProtoMember(3)]
  private string name;
  [ProtoMember(4)]
  private Owner owner;

  ...
}
```

# Protobuf-net Generics on Unity3D IL2CPP.

```
public class CustomCollectionBase<TCollection, TElement, TKey>
{
	protected TCollection _collection;

	protected SortedDictionary<TKey, bool> _removed;

	public CustomCollectionBase(TCollection collection)
	{
		_collection = collection;
	}

	protected byte[] _cBytes;

	protected byte[] _rBytes;

	protected void __PreSerialize()
	{
		using (MemoryStream stream = new MemoryStream())
		{
			Serializer.Serialize(stream, this._collection);
			this._cBytes = stream.ToArray();
		}

		using (MemoryStream stream = new MemoryStream())
		{
			Serializer.Serialize(stream, this._removed);
			this._rBytes = stream.ToArray();
		}
	}

	protected void __PostDeserialize()
	{
		using (MemoryStream stream = new MemoryStream(this._cBytes))
		{
			this._collection = Serializer.Deserialize<TCollection>(stream);
		}

		using (MemoryStream stream = new MemoryStream(this._rBytes))
		{
			this._removed = Serializer.Deserialize<SortedDictionary<TKey, bool>>(stream);
		}
	}
}

[ProtoContract]
public class CustomList<TElement> : CustomCollectionBase<List<TElement>, TElement, int>
{
	public CustomList() : base( new List<TElement>())
	{
	}

	[ProtoMember(1)]
	private byte[] _cProto
	{
		get { return base._cBytes; }
		set { base._cBytes = value; }
	}

	[ProtoMember(2)]
	private byte[] _rProto
	{
		get { return base._rBytes; }
		set { base._rBytes = value; }
	}

	[ProtoBeforeSerialization]
	private void __PreSerialize()
	{
		base.__PreSerialize();
	}

	[ProtoAfterDeserialization]
	private void __PostDeserialize()
	{
		base.__PostDeserialize();
	}
}

[ProtoContract]
public class CustomIntDictionary<TElement> : CustomCollectionBase<Dictionary<int, TElement>, TElement, int>
{
	public CustomIntDictionary() : base(new Dictionary<int, TElement>())
	{
	}

	[ProtoMember(1)]
	private byte[] _cProto
	{
		get { return base._cBytes; }
		set { base._cBytes = value; }
	}

	[ProtoMember(2)]
	private byte[] _rProto
	{
		get { return base._rBytes; }
		set { base._rBytes = value; }
	}

	[ProtoBeforeSerialization]
	private void __PreSerialize()
	{
		base.__PreSerialize();
	}

	[ProtoAfterDeserialization]
	private void __PostDeserialize()
	{
		base.__PostDeserialize();
	}
}

[ProtoContract]
public class CustomStringDictionary<TElement> : CustomCollectionBase<Dictionary<string, TElement>, TElement, string>
{
	public CustomStringDictionary() : base(new Dictionary<string, TElement>())
	{
	}

	[ProtoMember(1)]
	private byte[] _cProto
	{
		get { return base._cBytes; }
		set { base._cBytes = value; }
	}

	[ProtoMember(2)]
	private byte[] _rProto
	{
		get { return base._rBytes; }
		set { base._rBytes = value; }
	}

	[ProtoBeforeSerialization]
	private void __PreSerialize()
	{
		base.__PreSerialize();
	}

	[ProtoAfterDeserialization]
	private void __PostDeserialize()
	{
		base.__PostDeserialize();
	}
}
```

So basically there are two methods that are defined.

1. __PreSerialize – Converts the collection in to a byte array which becomes the proto member.
2. __PostDeserialize – Converts the byte array back to the collection.

We can completely avoid defining run time types for this generic type. Instead of protobuf-net being responsible of creating the type, we create the type by ourselves using the same protobuf Serializer.

This kind of technique can be used for other generic types as well. Please comment your ideas about this approach.