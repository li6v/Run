# 值元组 ValueTuple

## 什么是ValueTuple?

​		可以认为ValueTuple是一种可以存放多个不同类型的元素的集合类型。

##  创建ValueTuple

```csharp
ValueTuple<int, double> vt = new ValueTuple<int, double>(1, 3.1415);

ValueTuple<int, double> vt = ValueTuple.Create<int, double>(1, 3.1415);

var vt = (sum: 1, pai : 3.1415);

(int sum, double pai) vt = (1, 3.14);
```

显然第3和4种方式最好，不仅语法简洁，而且可以利用有意义的名称访问元组内的成员。

## 访问值元组成员

1. ItemX

   ```csharp
   ValueTuple<int, double> vt = new ValueTuple<int, double>(1, 3.1415);
   vt.Item1 = 2;
   vt.Item2 = 3.14;
   ```

2. 有意义的名称

   ```csharp
   var vt = (sum: 1, pai : 3.1415);
   vt.sum = 2;
   vt.pai = 3.14;
   
   (int sum, double pai) vt = (1, 3.14);
   vt.sum = 83;
   vt.pai = 3.1415926;
   ```

   显然方式2采用有意义的名称，程序的可读性更好，建议永远使用方式2使用值元组。

## 最多8个成员的限制

​        ValueTuple最多支持8个泛型，超出后，第8个成员可以定义成ValueTuple，来定义更多数量的元素。

```csharp
var vt = new ValueTuple<int, int, int, int, int, int, int, ValueTuple<int, int, int>>(1, 2, 3, 4, 5, 6, 7, (8, 9, 10));
Console.WriteLine(vt.Item10);    // 访问第10个元素 -- 10
Console.WriteLine(vt.Rest.Item3); // 访问第10个元素 -- 10
```

## 常见的应用场景

1. 方法有多个返回值

==现有实现方法和缺点==

```csharp
// 方法1：out
public void DoSomething1(out int id, out string message) { id = 10; message = "Hi"; }
// 方法2：额外类
public WorkResult DoSomething2() => new WorkResult { Code = 10, Message = "Hi" };
// 方法3：dynamic
public dynamic DoSomething3() => new { Code = 10, Message = "Hi" };
// 方法4：object
public object DoSomething4() => new { Code = 10, Message = "Hi" };
```

4种方法的缺点

方法1：out不支持async/await，await异步方法后，并不会将输出以返回值形式返回。

方法2：定义额外的类，让项目充满了许多无关紧要的小类，看起来冗杂。

方法3：需要以字符串形式访问返回值的各个成员，没有智能提示，易出错。

方法4：需要通过反射访问返回值的各个成员，更加复杂。

==利用值元组返回多个值==

```csharp
 static (string, int, double) GetStudentInfo(int id)
 {
     return ("Jack", 20, 182);
 }
 
 var studentInfo = GetStudentInfo(1);
 // 通过ItemX访问成员
 Console.WriteLine($"姓名：{studentInfo.Item1}");
 Console.WriteLine($"年龄：{studentInfo.Item2}");
 Console.WriteLine($"身高：{studentInfo.Item3}");
```

输出

```te
姓名：Jack
年龄：20
身高：182
```

上述缺点，需要用Item1，Item2，Item3来访问值元组的成员，可以利用值元组的别名，通过有意义的标识符访问其成员，只需要定义值元组时指定别名即可。

```csharp
static (string name, int age, double height) GetStudentInfo(int id)
{
    return ("Jack", 20, 182);
}

var studentInfo = GetStudentInfo(1);
// 通过别名访问成员
Console.WriteLine($"姓名：{studentInfo.name}");
Console.WriteLine($"年龄：{studentInfo.age}");
Console.WriteLine($"身高：{studentInfo.height}");
```

2. 泛型集合

   放在泛型集合中的元素必须是同一种类型，但是没若干个不同类型甚至同一类型成员成员代表同一个含义。

   可以拿一次性获取Ini文件的多个Key的值为例。

   ```csharp
   string[] GetValue(params (string section, string key)[] keys) => null;
   
   GetValue(("plc", "ip"),("plc","port"));
   ```

## 拆包和_

```csharp
(string name, int age, double height)  = GetStudentInfo(1);

// 丢弃不需要的返回值, 可以使用 _ 拆包，表示不会使用的返回值
(_, int age, _) = GetStudentInfo(1);
```

## 迷雾重重

难点：区分( )语法表示 ①拆包②批量赋值还是③创建值元组。

```csharp
static (string, int, double) GetStudentInfo(int id) =>("",0,1.1);

public static void Main(string[] args)
{
    (string name, int age, double height) = GetStudentInfo(1);
}
```

以上代码会被编译成：

```csharp
var info = GetStudentInfo(1);
string name = info.Item1;
int age = info.Item2;
double height = info.Item3;
```

-----

```csharp
(string name, int age, double height) info = GetStudentInfo(1);
```

以上代码会被编译成：

```csharp
ValueTuple<string,int,double> info = GetStudentInfo(0);
info.Item1;info.Item2;info.Item3;
```

----

```csharp
public class Point
{
    public int X { get; set; }
    public int Y { get; set; }

    public Point(int x, int y)
    {
        (X, Y) = (x, y);
    }
}
```

以上代码会被编译成：

```csharp
public class Point
{
    public int X { get; set; }
    public int Y { get; set; }

    public Point(int x, int y)
    {
        this.X = x;
        this.Y = y;
    }
}
```

-----

```csharp
public class Point
{
    public int X { get; set; }
    public int Y { get; set; }

    public Point(int x, int y)
    {
        (X, Y) A = (x, y);  // 编译失败，语法错误
    }
}
```

-----

总结

```csharp
(int age,string name) 要么是声明值元组，要么是拆包
int age; string name; (age,name) 要么是分配元组内存，要么是拆包，要么是批量赋值
(1,"Jack") 要么是分配元组内存，要么是批量赋值
(age:1,name:"Jack") 肯定是分配元组内存
```





## 值元组和序列化

能够给元素命名，方便使用和记忆，这里需要注意虽然命名了，但是实际上value tuple没有定义这样名字的属性或者字段，真正的名字仍然是ItemX，所有的元素名字都只是设计和编译时用的，不是运行时用的（因此注意对该类型的序列化和反序列化操作）；

函数返回的ValueTuple 各元素的名字记录在TupleElementNamesAttribute里，所以运行时也不是没有办法获取到的。

# 原始字符串

## 转义字符

字符分为可显示字符(如A,B,C,1,2,3)和控制字符(回车符CR,换行符LF,空NUL,EOT传输结束,退格BS)。在编辑代码时，可显示字符可使用键盘直接输入，但是控制字符无法输入和表示，所以IT行业规定，利用 `\` + `可视化字符`表示一个控制字符，如回车符 `\r`，换行符 `\n`。由于 \被用来转义，所以`\`用`\\`表示。

==综上，控制字符用 `\` + `可视化字符`表示，可视化字符直接表示，但`\`这个可视化字符由于被用来转义，所以要特殊对待它，用`\\`表示`\`。==

由于在代码中，字符串类型的字面值需要用`""`包裹，来让编译器知道此字面值是字符串类型，所以除了特殊对待`控制字符`，`\` ，还要特殊对待` "`。如果字符串中包含`"`，用`\"`表示`"`，这样可以让编译器能够区分出` "`是作为字符串的内容还是结束标志。

* 如何在C#中书写下面的字符串？

`12/3\ab"cCRLF`  【共计11个字符】

```csharp
string str = "12/3\\ab\"c\r\n";
```

* C#语法可以用@禁止编译器把`\`当转义，如果字符串中含有很多`\`，这样就很省事，常用于路径字符串。

`‪C:\Program Files\Adobe\Acrobat DC\Resource\Font\CourierStd.otf`

不带@

```csharp
string str = "‪C:\\Program Files\\Adobe\Acrobat DC\\Resource\\Font\\CourierStd.otf";
```
带@
```csharp
string str = @"‪C:\Program Files\Adobe\Acrobat DC\Resource\Font\CourierStd.otf";
```

==由于带@时，禁止\转义，所以当我们要输入的字符串含有控制字符时，无法用@这个语法糖。另外，如果字符串含有`"`,不再用`\"`表示，而是用`""`==.

```csharp
string str = @"12/3\ab""c";
```

* 原始字符串
  1. 用"""包裹的字符串，肉眼看到什么，就被解析成什么。
  2. 原始字符串字面量以至少三个双引号 (""") 字符开头， 它以相同数量的双引号字符结尾。
  3. 左引号之后、右引号之前的换行符不包括在最终内容中。
  4. 右双引号左侧的任何空格都将从字符串字面量中删除。
  5. 原始字符串和@一样，禁止转义，所以无法表示带有控制字符的字符串，但是原始字符串排版比@好，也不用特殊处理`"`.
  6. 特别适合表示JSON字符串

```csharp
string longMessage = """
            This is a long message.
            It has several lines.
                Some are indented
                        more than others.
            Some should start at the first column.
            Some have "quoted text" in them.
            """;
```

输出

```tex
This is a long message.
It has several lines.
    Some are indented
            more than others.
Some should start at the first column.
Some have "quoted text" in them.
```
## 如何处理内插变量

`{"level":"info"}`

```csharp
string level = "error";

string msg = $"{{\"level\":\"{level}\"}}";

msg = $@"{{""level"":""{level}""}}";

msg = $$"""
    {"level":"{{level}}"}
    """;
```

==字符串未加$时，`{` 和 `}`是简单的可显示字符。加了$后，`{` 和 `}`用于包裹变量，有了其他专门用途，所以想要表示`{`自身时，用`{{`，同理，`}`用`}}`表示。==



==原始字符串至少用2个 `$` 表示有多少个连续的大括号开始和结束内插。==

```csharp
int x = 100;
string str =$$"""
{{{x}}}
""";
// 输出 {100}
    
str = $$$"""
{{{x}}}
    """;
// 输出 100
```

# 类型转换

