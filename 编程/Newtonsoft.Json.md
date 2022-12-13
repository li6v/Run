[TOC]

------------

# JSON相关的推荐网站

[What is JSON (w3schools.com)](https://www.w3schools.com/whatis/whatis_json.asp)

[JSON](https://www.json.org/json-en.html)

[在线JSON校验格式化工具（Be JSON）](https://www.bejson.com/)

http://tomeko.net/software/JSONedit/

https://www.newtonsoft.com/json

# 前言

在C#中，创建一个实例可以用new，也可以使用字面值法，如1，"1",true,null等。这有点Javascript的感觉。

# JSON跨语言

编程语言的类型都可以映射到Json的某种类型，通过JSON这个媒介，可以实现不同语言的类型转换和沟通。如C++的`unsigned int`到C#的`short`，C#的`List`到C#的`DataTable`，C++的`string`到C#的`DateTime`。
一个类就是一堆数据的集合，C#的类序列化成JSON再转成C++的类，这样就可以实现采用不同语言编写的应用程序间的交互。

# JSON有哪些类型

JSON类型：数字，字符串，布尔，对象，数组，null

## 数字

JSON中的数字可以是小数，整数，正数，负数，可以使用科学计数法。

`-289`

`22.59`

`5.972e+24`

文本文件test.json

```txt
0.618
```

```c#
JsonConvert.DeserializeObject<double>(File.ReadAllText("test.json"));
```

源码字面值

```csharp
JsonConvert.DeserializeObject<double>("0.618");
```

## 字符串

JSON的字符串必须且只能用双引号包裹，不能使用单引号包裹

JSON字符串内部的双引号需要加转义字符表示` \"`

JSON字符串内部使用`\\`表示`\`

JSON字符串内部的控制字符仍旧采用转义字符表示，如换行符`\n` 回车符`\r`制表符`\t` 空字符`\0` 响铃`\a` 退格`\b`

文本文件 test.json

```tex
"0.618"
```

```csharp
JsonConvert.DeserializeObject<string>(File.ReadAllText("test.json"));
```

源码字面值

```csharp
JsonConvert.DeserializeObject<double>("\"0.618\"");
```

https://www.bejson.com/   可利用该网站提供的JSON转义工具进行转义。

原始表达：John Say "Bye"

Json文本形式："John Say \\"Bye\\"" 

C#字面值形式：`"\"John Say \\\"Bye\\\"\""`（Json文本形式经过转义得到的Json代码形式）

注意：使用双引号包裹Json文本形式经过转义后得到的字符串，便可以得到C#字面值形式。

## 对象

对象类型是使用逗号分隔的名称-值对构成的集合，并使用花括号{ }包裹。

使用json表示对象时，属性名(key)必须用双引号包裹，对于Json本身而言，key名可以是包含空格，#，% 等的字符串，但是为了可移植性兼容更多的语言，尽量不要使用特殊字符。

value如果是字符串类型，可以是任何形式任何字符。 value并不需要总被双引号包裹，value是字符串时，必须使用双引号,如果是数字，布尔值，数组，对象，null，这些都不应被双引号包裹。

```csharp
Button btn = JsonConvert.DeserializeObject<Button>("{\"Tag\":2021}");
btn = JsonConvert.DeserializeObject<Button>("{}");
btn = JsonConvert.DeserializeObject<Button>("null");
```

## 数组

数组是值的集合，每个值都可以是字符串，数字，布尔值，对象和数组中的任何一种，数组必须被方括号[ ]包裹，且值与值之间用逗号隔开。

["str1","str2",null, 5.0] 这种表示对于json是合法的，但是我们应该尽量避免这样做，因为这种json 数组可以被动态语言正确解析成列表数据类型，但是像C#，java，C++这种强类型语言无法解析成泛型数组或泛型列表数据类型。

```csharp
ArrayList list = JsonConvert.DeserializeObject<ArrayList>("[1,true,\"123\",{},[1,2]]");
foreach (var item in list)
{
    Console.WriteLine(item.GetType().Name);
}
/*
Int64
Boolean
String
JObject
JArray
*/
```

## null

null表示一个没有值的值，必须是小写。
文本文件 test.josn
```tex
null
```
```c#
JsonConvert.DeserializeObject<int?>(File.ReadAllText("test.json"));
```
C#源码
```C#
JsonConvert.DeserializeObject<int?>("null");
```

# 构建一个合法JSON的步骤

合法的JSON文本形式只有以下7种形式，其它情况都是错误的。（在记事本中的展现）
（1）1
（2）"1"
（3）true
（4）false
（5）null
（6）{...}
（7）[...]

转换成C#源码字符串形式：

（1）"1"
（2）"/"1/""
（3）"true"
（4）"false"
（5）"null"
（6）"{...}"
（7）"[...]"

源代码文件，其实本身就是文本，就是字符序列，就是字符串。编译器在编译时，会把"1"解释成字符串1，把'1'解释成字符1，把1解释成数字1.

# JSON转义 & 去除转义网站

[在线JSON校验格式化工具（Be JSON）](https://www.bejson.com/)
JSON是一种文本，或者说是字符序列。构建Json文本时，要在记事本中书写构建，构建完成后程序以流读取文本文件即可获取Json文本。如果不采用流读取文件进行使用，那便是在源码中以字符串字面值表达。
int a 表示一个整数变量,string s 表示一个字符串变量，在源码中，1 被解释成一个整数(类型)，"1" 被解释成一个字符串(类型)。所以，int a = 1合法，int a = "1"是非法的；string s = "1"是合法的，string s = 1是非法的。
在源码中，使用"xxx"表示一个(文本xxx)字符串时，如果xxx中含有双引号，编译器便会无法得知哪个 " 表示字符串结束，所以，xxx中的双引号使用 \" 表示。那么又引入了新的问题，如何表示\呢？\\表示\ 。控制字符使用 \ + 显示字符表示。这样，"xxx"可以表示所有的文本(控制字符和显示字符)了。
生活中提及的文本，在源码中便是字符串。在记事本中编辑好的JSON文本，使用C#的字符串表达时，那势必是 "记事本中的JSON文本" 。我们知道，JSON中可能含有双引号，所以我们想要原封不动的在源码中表示JSON文本，必须先转义。
我们将记事本中的JSON文本先转义,然后，在源码中，**<u>"转义后的JSON文本"</u>** 便完全等价于记事本中的JSON文本了。
记事本中的【JSON文本】   等价于    源码中的【"转义后的JSON文本"】    等价于   去除转义后的【转义后的JSON文本】。
![image](https://img2020.cnblogs.com/blog/1037641/202111/1037641-20211114021623574-85822372.gif)


## JSON,字面值表示法

1. 类型系统的分类

基础类：数字，字符串，二元

简单类：成员都是基础类，重要的信息是属性

复杂类：成员至少包含一个简单类，也可以包含基础类，重要的信息是属性

集合类：重要的信息是存储的元素（基础类，简单类，复杂类，集合），而非属性，它的属性取决于它盛放的元素。

2. 字面值

只有基础类才有字面值形式。

C#的const关键字就是用具名变量表示字面值，消灭魔法数，魔法字符串，编译后const变量都会被替换成字面值。

C#的const只能修饰基元类型。

推出结论：任何类型都是由基础类扩展组成，那么基于字面值可以表示所有类型。JSON就是这种表示法。

# 序列化和反序列化
面向过程编程，由变量表示的数据在整个程序中非常零散，面向对象编程通过封装思想让这些零散的变量组织成对象的属性或字段，即属性=数据，这样更有利于组织和理解大型程序。在应用程序开发中，我们经常将数据持久化再在某个时机恢复这些数据，如果用面向对象去理解这个过程，便是保存对象的各个属性以及根据先前保存的属性值实例化一个对象。
将内存（断电即失）中的对象（变量）的状态（值）存储起来持久化（硬盘，数据库）称作序列化serialization，反之称作反序列化deserialization。

内存 --> 硬盘   序列化
硬盘 --> 内存   反序列化

# Newtonsoft.Json

## 简介

Newtonsoft.json.dll，也称Json.NET，是C#进行JSON序列化的类库，第三方私人贡献的类库，非微软官方类库。开源，跨平台，功能丰富，性能强悍，简单易用。该库极受欢迎，成为下载量第一的nuget包，是.NET平台优先选择的Json序列化类库。

官网：https://www.newtonsoft.com/json

## Newtonsoft.Json和System.Text.Json
Newtonsoft.Json功能更丰富更强大，但是性能略低于System.Text.Json.但个人认为这点性能算是微不足道，如果对性能十分敏感，推荐使用二进制的ProtoBuff.Newtonsoft.Json最致命的缺点是需要下载，版本不断迭代，可能带来应用程序引用不同版本的Newtonsoft.Json.dll麻烦.

# 最简单的序列化

```csharp
class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

Person person = new Person()
{
    Name = "Jack",
    Age = 17
};
```

```csharp
string json = JsonConvert.SerializeObject(person);
Console.WriteLine(json);
```

输出：

`{"Name":"Jack","Age":17}`

# 序列化时设置缩进

```csharp
string json = JsonConvert.SerializeObject(person,Formatting.Indented);
Console.WriteLine(json);
```

输出：
`{
     "Name": "Jack",
     "Age": 17
}`

# 缩进JSON字符串

```csharp
string jsonIndented = JToken.Parse(jsonNotIndented).ToString(Formatting.Indented);
```

# 缩进的优缺点

优点：缩进后的Json排版好，更易读。
缺点：添加了大量空白字符，导致序列化后得到的Json长度变长，那么存储的成本会变高，CPU处理的时间会变长，网络发送占用的带宽更高，耗时更久。
建议：因为网络资源成本比较高，存储成本与CPU成本相对较低，所以不要缩进用于网络发送的Json，进程可以缩进接收到的Json后再打印日志和储存。

# 非集合类型的序列化

```C#
JsonConvert.SerializeObject(2021);      // 2021
JsonConvert.SerializeObject('A');       // "A"
JsonConvert.SerializeObject(true);      // true
JsonConvert.SerializeObject("2021");    // "2021"
JsonConvert.SerializeObject(null);      // null
JsonConvert.SerializeObject(DateTime.Now); // "2021-02-01T11:04:11.4245398+08:00"
JsonConvert.SerializeObject(TimeSpan.FromSeconds(1024)); // "00:17:04"
JsonConvert.SerializeObject(new Uri(@"C:\floder")); // "C:\\floder"
JsonConvert.SerializeObject(System.Guid.NewGuid()); // "c711acc0-b87b-48fe-809d-82647d8a79df"
JsonConvert.SerializeObject(new { });   // {}
```

# 集合类型的序列化

集合类型序列化时，并不是为了储存其属性信息，而是其包含的元素的信息。这是合理的，想一想，当我们恢复容器元素后，容器的属性自然也相应的恢复。容器类型有价值的信息是其包含的元素。

## 数组

```csharp
string jsonArray = JsonConvert.SerializeObject(new int[] { });
string jsonList = JsonConvert.SerializeObject(new List<double> { 3.14, 9.12, 0.67 });
Console.WriteLine(jsonArray);
Console.WriteLine(jsonList);
```

输出：

```csharp
[]
[3.14,9.12,0.67]
```

## 字典

```csharp
Dictionary<string, int> points = new Dictionary<string, int>
{
  { "James", 9001 },
  { "Jo", 3474 },
  { "Jess", 11926 }
};
string jsonDictionary = JsonConvert.SerializeObject(points, Formatting.Indented);
Console.WriteLine(jsonDictionary);
```

```csharp
{
      "James": 9001,
      "Jo": 3474,
      "Jess": 11926
}
```

总结：字典会被序列化成对象，key.ToString()是字段名，value是字段值。

## DataTable

```csharp
DataTable table = new DataTable();
table.Columns.Add(new DataColumn("Name", typeof(string)));
table.Columns.Add(new DataColumn("Age", typeof(int)));
table.Rows.Add("Jack", 20);
JsonConvert.SerializeObject(table);
```

输出：[{"Name":"Jack","Age":20}]
总结：DataTable其实就是实例的集合，实例的字段名就是列名，实例的类型就是列的类型。

## DataSet

```csharp
DataSet dataSet = new DataSet("dataSet");
DataTable table = new DataTable();
DataColumn idColumn = new DataColumn("id", typeof(int));
idColumn.AutoIncrement = true;
DataColumn itemColumn = new DataColumn("item");
table.Columns.Add(idColumn);
table.Columns.Add(itemColumn);
dataSet.Tables.Add(table);
for (int i = 0; i < 2; i++)
{
  DataRow newRow = table.NewRow();
  newRow["item"] = "item " + i;
  table.Rows.Add(newRow);
}
dataSet.AcceptChanges();
string json = JsonConvert.SerializeObject(dataSet, Formatting.Indented);
Console.WriteLine(json);
```

输出：

```json
{
    "Table1": [{
            "id": 0,
            "item": "item 0"
        },
        {
            "id": 1,
            "item": "item 1"
        }
    ]
}
```

# 处理枚举

```csharp
enum Status
{
    Idel,
    Active,
    Unkonown
}

// 枚举序列化成字符串形式
string jsonEnumStr = JsonConvert.SerializeObject(new List<Status>() { Status.Active, Status.Idel, Status.Unkonown });
Console.WriteLine(jsonEnumStr);
var ret = JsonConvert.DeserializeObject<List<Status>>(jsonEnumStr, new StringEnumConverter());
// 枚举序列化成整数形式
string jsonEnumInt = JsonConvert.SerializeObject(new List<Status>() { Status.Active, Status.Idel, Status.Unkonown }, new StringEnumConverter());
Console.WriteLine(jsonEnumInt);
ret = JsonConvert.DeserializeObject<List<Status>>(jsonEnumStr);
```

输出：

`[1,0,2]
["Active","Idel","Unkonown"]`

# 剔除属性

我们可以不序列化我们不感兴趣的属性，这样可以简化JSON，这样可以减少流量的消耗。

1. 增加私有字段 或 删除公开属性
2. 剔除值为null的属性
3. 设置属性的默认值，并剔除值是默认值的属性
4. 剔除指定类型的属性
5. 剔除不满足指定表达式的属性

```csharp
JsonConvert.SerializeObject(new TestClass(), new JsonSerializerSettings()
{
    ContractResolver = new ConcreteContractResolver()
});
```

```csharp
internal class ConcreteContractResolver : DefaultContractResolver
{
    protected override JsonContract CreateContract(Type objectType)
    {
        return base.CreateContract(objectType);
    }

    protected override IList<JsonProperty> CreateProperties(Type type, MemberSerialization memberSerialization)
    {
        IList<JsonProperty> properties = base.CreateProperties(type, memberSerialization);
        foreach (var item in properties)
        {
            item.PropertyName = "_" + item.PropertyName;
        }
        return properties;
    }

    protected override List<MemberInfo> GetSerializableMembers(Type objectType)
    {
        List<MemberInfo> members = base.GetSerializableMembers(objectType);
        if (objectType.Name == typeof(TestClass).Name)
        {
            FieldInfo fieldInfo = objectType.GetField("_param", BindingFlags.NonPublic | BindingFlags.Instance);
            // 添加私有字段
            members.Add(fieldInfo);
            // 删除属性
            members.RemoveAll(elem => elem.Name == "PropertyRemoved");
        }
        return members;
    }

    protected override JsonProperty CreateProperty(MemberInfo member, MemberSerialization memberSerialization)
    {
        JsonProperty jp = base.CreateProperty(member, memberSerialization);

        // 私有属性的Readable必须设置成true才能序列化
        if (jp.PropertyName == "_param")
        {
            jp.Readable = true;
        }

        // 忽略值为默认值的属性 | TestClass的成员Key的值若是DELETE则不序列化
        if (member.DeclaringType == typeof(TestClass))
        {
            if (member.Name == "Key")
            {
                // 设置属性的默认值
                jp.DefaultValue = "DELETE";
                // 忽略值为默认值的属性
                jp.DefaultValueHandling = DefaultValueHandling.Ignore;
            }
        }

        // 忽略值为null的属性
        jp.NullValueHandling = NullValueHandling.Ignore;
        // 忽略值为默认值的属性
        jp.DefaultValueHandling = DefaultValueHandling.Ignore;
        // 忽略特定名称的属性
        if (member.Name == "Id")
        {
            jp.Ignored = true;
        }

        // 忽略指定类型的属性
        if (jp.PropertyType == typeof(bool))
        {
            jp.Ignored = true;
        }

        // 忽略满足指定条件的属性
        jp.ShouldSerialize = new Predicate<object>((obj) =>
        {
            if (member.DeclaringType == typeof(TestClass))
            {
                TestClass @params = obj as TestClass;
                if (@params.Id == 3)
                {
                    return false;
                }
                return true;
            }
            return true;
        });

        return jp;
    }
}
```

# 自定义属性名称

JSON是跨语言跨平台的，经常用于进程间信息交互。常见的B-S模型，前端是Javascript，后端是PHP，C#，python等，如果前后端通过Json格式消息交互，但是不同的语言有着不同的命名习惯，导致序列化后得到的Json字段名称各不相同。比如C#序列化后的Json字段是大驼峰命名法，Json消息发送到前端，Javascript的字段都是小写字母，前端收到后无法正常解析，这时，能够指定序列化的Json字段名就非常有必要了。

### 小驼峰

```csharp
string json = JsonConvert.SerializeObject(new Message() { Params = new Params() }, new JsonSerializerSettings()
{
 	ContractResolver = new DefaultContractResolver() { NamingStrategy = new CamelCaseNamingStrategy() },
});
Console.WriteLine(json);
```

输出：

{"id":0,"key":null,"executeResult":false,"params":{"param1":"\u0000","param2":0.0}}

### 蛇形

```csharp
string json = JsonConvert.SerializeObject(new Message() { Params = new Params() }, new JsonSerializerSettings()
{
 	ContractResolver = new DefaultContractResolver() { NamingStrategy = new SnakeCaseNamingStrategy() },
});
Console.WriteLine(json);
```

输出：

{"id":0,"key":null,"execute_result":false,"params":{"param1":"\u0000","param2":0.0}}

### 单独设置某个属性的名称

单独设置的某个属性的名称，优先级较高，会屏蔽NamingStrategy。

```csharp
internal class ConcreteContractResolver : DefaultContractResolver

{
    protected override JsonProperty CreateProperty(MemberInfo member, MemberSerialization memberSerialization)

    {
        // 注意：如果通过NamingStrategy设置了小驼峰或蛇形，此时的jp的PropertyName已经不是C#的大驼峰，而是小驼峰或蛇形了

        JsonProperty jp = base.CreateProperty(member, memberSerialization);

        if (member.DeclaringType == typeof(Params))

        {
            if (member.Name == "Param1")

            {
                jp.PropertyName = "param_one";
            }
        }

        return jp;
    }
}
```

```csharp
string json = JsonConvert.SerializeObject(new Message() { Params = new Params() }, new JsonSerializerSettings()
{
	ContractResolver = new ConcreteContractResolver() { NamingStrategy = new SnakeCaseNamingStrategy() },
});
```

输出：

{"id":0,"key":null,"execute_result":false,"params":{"param_one":"\u0000","param2":0.0}}

# 处理默认值

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public Person Partner { get; set; }
    public decimal? Salary { get; set; }
}

Person person = new Person();

string jsonIncludeDefaultValues = JsonConvert.SerializeObject(person, Formatting.Indented);

Console.WriteLine(jsonIncludeDefaultValues);
// {
//   "Name": null,
//   "Age": 0,
//   "Partner": null,
//   "Salary": null
// }

string jsonIgnoreDefaultValues = JsonConvert.SerializeObject(person, Formatting.Indented, new JsonSerializerSettings
{
    DefaultValueHandling = DefaultValueHandling.Ignore
});

Console.WriteLine(jsonIgnoreDefaultValues);
// {}
```



# 处理未知属性

```csharp
public class Account
{
    public string FullName { get; set; }
    public bool Deleted { get; set; }
}

string json = @"{
  'FullName': 'Dan Deleted',
  'Deleted': true,
  'DeletedDate': '2013-01-20T00:00:00'
}";

try
{
    JsonConvert.DeserializeObject<Account>(json, new JsonSerializerSettings
    {
        MissingMemberHandling = MissingMemberHandling.Error
    });
}
catch (JsonSerializationException ex)
{
    Console.WriteLine(ex.Message);
    // Could not find member 'DeletedDate' on object of type 'Account'. Path 'DeletedDate', line 4, position 23.
}
```



# 处理null

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public Person Partner { get; set; }
    public decimal? Salary { get; set; }
}

Person person = new Person
{
    Name = "Nigal Newborn",
    Age = 1
};

string jsonIncludeNullValues = JsonConvert.SerializeObject(person, Formatting.Indented);

Console.WriteLine(jsonIncludeNullValues);
// {
//   "Name": "Nigal Newborn",
//   "Age": 1,
//   "Partner": null,
//   "Salary": null
// }

string jsonIgnoreNullValues = JsonConvert.SerializeObject(person, Formatting.Indented, new JsonSerializerSettings
{
    NullValueHandling = NullValueHandling.Ignore
});

Console.WriteLine(jsonIgnoreNullValues);
// {
//   "Name": "Nigal Newborn",
//   "Age": 1
// }
```





## 多态

```csharp
public class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }
}

public class Employee : Person
{
    public string Department { get; set; }
    public string JobTitle { get; set; }
}

public class PersonConverter : CustomCreationConverter<Person>
{
    public override Person Create(Type objectType)
    {
        return new Employee();
    }
}

string json = @"{
  'Department': 'Furniture',
  'JobTitle': 'Carpenter',
  'FirstName': 'John',
  'LastName': 'Joinery',
  'BirthDate': '1983-02-02T00:00:00'
}";

Person person = JsonConvert.DeserializeObject<Person>(json, new PersonConverter());

Console.WriteLine(person.GetType().Name);
// Employee

Employee employee = (Employee)person;

Console.WriteLine(employee.JobTitle);
// Carpenter
```



```csharp
public abstract class Business
{
    public string Name { get; set; }
}

public class Hotel : Business
{
    public int Stars { get; set; }
}

public class Stockholder
{
    public string FullName { get; set; }
    public IList<Business> Businesses { get; set; }
}


Stockholder stockholder = new Stockholder
{
    FullName = "Steve Stockholder",
    Businesses = new List<Business>
    {
        new Hotel
        {
            Name = "Hudson Hotel",
            Stars = 4
        }
    }
};

string jsonTypeNameAll = JsonConvert.SerializeObject(stockholder, Formatting.Indented, new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.All
});

Console.WriteLine(jsonTypeNameAll);
// {
//   "$type": "Newtonsoft.Json.Samples.Stockholder, Newtonsoft.Json.Tests",
//   "FullName": "Steve Stockholder",
//   "Businesses": {
//     "$type": "System.Collections.Generic.List`1[[Newtonsoft.Json.Samples.Business, Newtonsoft.Json.Tests]], mscorlib",
//     "$values": [
//       {
//         "$type": "Newtonsoft.Json.Samples.Hotel, Newtonsoft.Json.Tests",
//         "Stars": 4,
//         "Name": "Hudson Hotel"
//       }
//     ]
//   }
// }

string jsonTypeNameAuto = JsonConvert.SerializeObject(stockholder, Formatting.Indented, new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.Auto
});

Console.WriteLine(jsonTypeNameAuto);
// {
//   "FullName": "Steve Stockholder",
//   "Businesses": [
//     {
//       "$type": "Newtonsoft.Json.Samples.Hotel, Newtonsoft.Json.Tests",
//       "Stars": 4,
//       "Name": "Hudson Hotel"
//     }
//   ]
// }

// for security TypeNameHandling is required when deserializing
Stockholder newStockholder = JsonConvert.DeserializeObject<Stockholder>(jsonTypeNameAuto, new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.Auto
});

Console.WriteLine(newStockholder.Businesses[0].GetType().Name);
// Hotel
```



```csharp
This sample deserializes JSON with MetadataPropertyHandling set to ReadAhead so that metadata properties do not need to be at the start of an object.
    
string json = @"{
  'Name': 'James',
  'Password': 'Password1',
  '$type': 'MyNamespace.User, MyAssembly'
}";

object o = JsonConvert.DeserializeObject(json, new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.All,
    // $type no longer needs to be first
    MetadataPropertyHandling = MetadataPropertyHandling.ReadAhead
});

User u = (User)o;

Console.WriteLine(u.Name);
// James
```



# 构造方法

```csharp
public class Website
{
    public string Url { get; set; }

    private Website()
    {
    }

    public Website(Website website)
    {
        if (website == null)
        {
            throw new ArgumentNullException(nameof(website));
        }

        Url = website.Url;
    }
}

string json = @"{'Url':'http://www.google.com'}";

try
{
    JsonConvert.DeserializeObject<Website>(json);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
    // Value cannot be null.
    // Parameter name: website
}

Website website = JsonConvert.DeserializeObject<Website>(json, new JsonSerializerSettings
{
    ConstructorHandling = ConstructorHandling.AllowNonPublicDefaultConstructor
});

Console.WriteLine(website.Url);
// http://www.google.com
```





```csharp
public class UserViewModel
{
    public string Name { get; set; }
    public IList<string> Offices { get; private set; }

    public UserViewModel()
    {
        Offices = new List<string>
        {
            "Auckland",
            "Wellington",
            "Christchurch"
        };
    }
}

string json = @"{
  'Name': 'James',
  'Offices': [
    'Auckland',
    'Wellington',
    'Christchurch'
  ]
}";

UserViewModel model1 = JsonConvert.DeserializeObject<UserViewModel>(json);

foreach (string office in model1.Offices)
{
    Console.WriteLine(office);
}
// Auckland
// Wellington
// Christchurch
// Auckland
// Wellington
// Christchurch

UserViewModel model2 = JsonConvert.DeserializeObject<UserViewModel>(json, new JsonSerializerSettings
{
    ObjectCreationHandling = ObjectCreationHandling.Replace
});

foreach (string office in model2.Offices)
{
    Console.WriteLine(office);
}
// Auckland
// Wellington
// Christchurch
```

```csharp
JsonConvert.DeserializeObject<object>("{"A":"aaa","B":123}");
```

```csharp
class Message
{
    public string MsgType{get;set;}
    public string Command{get;set;}
    public object MsgParam{get;set;}
}
```

```csharp
Message msg = JsonConvert.DeserializeObject<Message>("MsgType":"REQ","Command":"Add","MsgParam":{"Argument1":111,"Argument2":222});
```

msg.MsgParam的类型不是Object，二是JObject，这是Newtonsoft.Json的特殊处理。

## 循环引用

查看Json.NET官网的Samples。

## 日期

Newtonsoft.Json会将DateTime类型作为字符串输出，默认情况下时间格式是ISO（世界标准时间格式）。

```csharp
string json = JsonConvert.SerializeObject(DateTime.Now);
Console.WriteLine(json);
// "2022-10-31T21:52:04.1221217+08:00"
```

可自定义DateTime的输出格式。

```csharp
string json = JsonConvert.SerializeObject(DateTime.Now, new JsonSerializerSettings()
{
    DateFormatString = "yyyy年M月d日"
});
string json = JsonConvert.SerializeObject(DateTime.Now);
Console.WriteLine(json);
// "2022年10月31日"
```

全局设置

```csharp
JsonConvert.DefaultSettings = new Func<JsonSerializerSettings>(() =>
{
    JsonSerializerSettings settings = new JsonSerializerSettings();
    settings.DateFormatString = "yyyy年-M月-d日";
    return settings;
});
string json = JsonConvert.SerializeObject(DateTime.Now);
Console.WriteLine(json);
// "2022年-10月-31日"
```

## 默认情况下不区分大小写的反序列化

JSON的妙用
DataGridView控件的DataSource绑定DataTable比较好，如果我们有一个现成的List<T>，可以经过序列化再反序列化成DataTable，最后绑定到DataGridView

# 理解JsonConverter



StringEnumConverter和DateTimeFormatConverter源码可以更好的理解JsonConverter。



`Newtonsoft.Json` 是 .NET 下最受欢迎 JSON 操作库，原为 `JSON.Net` 后改名为 `Newtonsoft.Json`，之前一直推荐大家使用，除了性能好之外，主要是功能丰富，基本满足所有的可能用到的场景（不区分小写，现在还不行，，）。

遇到这样一个需求，全局使用一种时间格式，某些属性使用特殊的时间格式，这里以一个日期为例



解决办法：自定义一个 Converter，针对某一个属性使用，[DateTimeFormatConverter源码](https://github.com/WeihanLi/WeihanLi.Common/blob/dev/src/WeihanLi.Common/Json/DateTimeFormatConverter.cs):

```csharp
using Newtonsoft.Json.Converters;

namespace WeihanLi.Common.Json
{
  public class DateTimeFormatConverter : IsoDateTimeConverter
  {
    public DateTimeFormatConverter(string format)
    {
      DateTimeFormat = format;
    }
  }
}
```

在需要设置格式的属性上设置 Converter https://github.com/WeihanLi/ActivityReservation/blob/dev/ActivityReservation.Helper/ViewModels/ReservationViewModel.cs#L8

```csharp
[Display(Name = "预约日期")]
[JsonConverter(typeof(DateTimeFormatConverter), "yyyy-MM-dd")]
public DateTime ReservationForDate { get; set; }
```

请求 api 地址 https://reservation.weihanli.xyz/api/Reservation?pageNumber=1&pageSize=5,返回的数据如下所示：

```json
{
    "Data": [
        {
            "ReservationForDate": "2019-06-10",
            "ReservationForTime": "08:00~09:50",
            "ReservationPersonPhone": "123****0112",
            "ReservationPersonName": "儿**",
            "ReservationUnit": "51",
            "ReservationPlaceName": "多媒体工作室",
            "ReservationActivityContent": "62",
            "ReservationId": "f7ab9128-0977-4fd8-9b1a-92648228b397",
            "ReservationTime": "2019-06-09 05:19:11",
            "ReservationStatus": 1
        },
        {
            "ReservationForDate": "2019-06-12",
            "ReservationForTime": "10:00-12:00",
            "ReservationPersonPhone": "133****3541",
            "ReservationPersonName": "试**",
            "ReservationUnit": "ss",
            "ReservationPlaceName": "多媒体工作室",
            "ReservationActivityContent": "ss",
            "ReservationId": "6c145aea-dc14-4ed9-a47f-48c0b79f7601",
            "ReservationTime": "2019-06-11 12:45:14",
            "ReservationStatus": 0
        },
        {
            "ReservationForDate": "2019-06-17",
            "ReservationForTime": "14:00-16:00",
            "ReservationPersonPhone": "138****3883",
            "ReservationPersonName": "大**",
            "ReservationUnit": "1",
            "ReservationPlaceName": "多媒体工作室",
            "ReservationActivityContent": "1",
            "ReservationId": "cebea7bf-44b1-4565-8cdd-78b6156c5f4d",
            "ReservationTime": "2019-06-10 02:52:18",
            "ReservationStatus": 1
        },
        {
            "ReservationForDate": "2019-06-17",
            "ReservationForTime": "08:00-10:00",
            "ReservationPersonPhone": "132****4545",
            "ReservationPersonName": "冷**",
            "ReservationUnit": "技术部",
            "ReservationPlaceName": "多媒体工作室",
            "ReservationActivityContent": "技术部培训",
            "ReservationId": "07f6f8fd-f232-478e-9a94-de0f5fa9b4e9",
            "ReservationTime": "2019-06-10 01:44:52",
            "ReservationStatus": 2
        },
        {
            "ReservationForDate": "2019-06-22",
            "ReservationForTime": "10:00~11:50",
            "ReservationPersonPhone": "132****3333",
            "ReservationPersonName": "测**",
            "ReservationUnit": "测试",
            "ReservationPlaceName": "多媒体工作室",
            "ReservationActivityContent": "测试",
            "ReservationId": "27d0fb7a-ce14-4958-8636-dd10e5526083",
            "ReservationTime": "2019-06-18 10:57:06",
            "ReservationStatus": 1
        }
    ],
    "PageNumber": 1,
    "PageSize": 5,
    "TotalCount": 18,
    "PageCount": 4,
    "Count": 5
}
```

可以看到 `ReservationForDate` 序列化之后返回的格式如我们指定的格式了~

# 全局序列化设置

在序列化时，可以通过一个JsonSerializerSettings实例指定采用的序列化规则。

```csharp
JsonSerializerSettings settings = new JsonSerializerSettings();
settings.Formatting = Formatting.Indented; 
settings.NullValueHandling = NullValueHandling.Ignore; 
settings.DateFormatHandling = DateFormatHandling.MicrosoftDateFormat;
settings.Converters.Add(new StringEnumConverter());
string json = JsonConvert.SerializeObject(DateTime.Now, settings);
```

未指明JsonSerializerSettings的`JsonConvert.SerializeObject(object obj)`使用的是默认的JsonSerializerSettings实例，可通过`JsonConvert.DefaultSettings`指定默认的JsonSerializerSettings实例。

```csharp
JsonConvert.DefaultSettings = new Func<JsonSerializerSettings>(() =>
{
    JsonSerializerSettings settings = new JsonSerializerSettings();
    settings.Formatting = Formatting.Indented;
    settings.Converters.Add(new StringEnumConverter());
    settings.NullValueHandling = NullValueHandling.Ignore;
    settings.DateFormatHandling = DateFormatHandling.MicrosoftDateFormat;
    return settings;
});
```

这样，在整个应用程序中的`JsonConvert.SerializeObject(object obj);`都会采用上面的JsonConvert.DefaultSettings。这就是Newtonsoft.Json的全局设置功能，一次设置，全局生效。

我们仍旧可以通过`JsonConvert.SerializeObject(object? value, JsonSerializerSettings? settings`定制序列化规则来覆盖掉全局的settings。
