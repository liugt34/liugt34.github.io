---
title: 在C# 9中使用record关键字声明强类型Id
date: 2023-05-03 14:22:38 +0800
categories: [ASP.NET Core]
tags: [aspnetcore]
mermaid: true
---

![banner](https://raw.githubusercontent.com/liugt34/imagegallery/main/th.jpg)


# 为何要用强类型Id

强类型代码具有很多优势

- 能够享受代码提示功能
- 能够获得重构工具的支持
- 能够在编译期发现更多错误

通常在实体类中，我们通常会使用int, string, int来作为id，例如

```c#
public class Teacher
{
	public int Id { get; set; }
    public string Name { get; set; }
}

public class Student
{
	public int Id { get; set; }
    public string Name { get; set; }
}

public class ClassRoom
{
	public int Id { get; set; }
    public string Name { get; set; }   
}
```

代码看起来很清晰明了，没有什么问题。但是往往会存在一些容易混淆的地方不容易被发现，例如我有一个方法

```c#
public class SchoolManager
{
    public AddTeacherAndStudentToClassRoom(int classId, int studentId, int teacherId); 
}	
    .
    .
    .
//错误的调用方法
AddTeacherAndStudentToClassRoom(studentId, teacherId, classId);
```

这个时候因为三个类型都是int，编译器是无法识别错误的，但是参数却出现了顺序混乱，最终导致数据错误

这个时候强类型Id就可以完美解决上面的问题

# 使用强类型Id

以学生为例，我们先定义一个学生Id的结构体StudentId

```c#
public readonly struct StudentId : IEquatable<StudentId>
{
    public StudentId(int value)
    {
        Value = value;
    }
    
    public int Value { get; }

    public bool Equals(StudentId other) => other.Value == Value;
    public override bool Equals(object obj) => obj is StudentId other && Equals(other);
    public override int GetHashCode() => Value.GetHashCode();
    public override string ToString() => $"StudentId {Value}";
    public static bool operator ==(StudentId a, StudentId b) => a.Equals(b);
    public static bool operator !=(StudentId a, StudentId b) => !a.Equals(b);
}
```

看起来似乎没有啥问题，一个简单的结构体，但是如果Teacher,ClassRoom都要写一遍的话是不是无形中增加了很多工作量呢。那如果有更多的实体类是不是要写到天荒地老，抓狂🤣。解决这个问题推荐两种方式

1. 国外大神的强类型库 [GitHub - andrewlock/StronglyTypedId: A Rosyln-powered generator for strongly-typed IDs](https://github.com/andrewlock/StronglyTypedId)
2. 使用C# 9的新特性 关键词 record

今天主要是介绍第二种 record
C# 9.0 引入了记录类型。 可使用 `record` 关键字定义一个引用类型，以最简的方式创建不可变类型。这种类型是线程安全的，不需要进行线程同步，非常适合并行计算的数据共享。它减少了更新对象会引起各种bug的风险，更为安全。 `System.DateTime`  和 `string`  也是不可变类型非常经典的代表。

上面的代码改造如下：

```c#
public record StudentId(int Value);
```

是的，你没看错，就一句😎, 改造后的学生类

```C#
public class Student
{
	public StudentId Id { get; set; }
    public string Name { get; set; }
}

//同理改造其他类中Id为强类型，此处不再一一举例
public class SchoolManager
{
    public AddTeacherAndStudentToClassRoom(ClassRoomId classId, StudentId studentId, TeacherId teacherId); 
}	
```



# 在Asp.Net Core中使用

新建一个控制器，然后启动

```csharp
[ApiController]
[Route("api/[controller]")]
public class StudentController : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<Student> Get(StudentId id)
    {
        return Ok(id);
    }
}
```

浏览器数据 https://localhost:7015/api/Student/1 结果报错

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-e92e15ae2e45e5d13f0f21c82e67142e-ba406004a318cfc7-00",
  "errors": {
    "$": [
      "The input does not contain any JSON tokens. Expected the input to start with a valid JSON token, when isFinalBlock is true. Path: $ | LineNumber: 0 | BytePositionInLine: 0."
    ],
    "id": [
      "The id field is required."
    ]
  }
}
```

ASP.NET Core不知道如何将整数 1转换成StudentId类型，所以会出现报错，此时我们需要增加一个类型转换器

```c#
public class StudentIdConverter : TypeConverter
{
    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType) =>
        sourceType == typeof(string);
    public override bool CanConvertTo(ITypeDescriptorContext context, Type destinationType) =>
        destinationType == typeof(string);

    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
    {
        return value switch
        {
            string s => new StudentId(int.Parse(s)),
            null => null,
            _ => throw new ArgumentException($"Cannot convert from {value} to StudentId", nameof(value))
        };
    }

    public override object ConvertTo(ITypeDescriptorContext context, CultureInfo culture, object value, Type destinationType)
    {
        if (destinationType == typeof(string))
        {
            return value switch
            {
                StudentId id => id.Value.ToString(),
                null => null,
                _ => throw new ArgumentException($"Cannot convert {value} to string", nameof(value))
            };
        }

        throw new ArgumentException($"Cannot convert {value ?? "(null)"} to {destinationType}", nameof(destinationType));
    }
}
```



同时将该特性添加至 StudentId

```c#
    [TypeConverter(typeof(StudentIdConverter))]
    public record StudentId(int Value);
```

F5再次运行得到输出结果

```json
{
    "id": {
        "value": 1
    },
    "name": "张三"
}
```

但是我们发现了一个问题，就是Id变成了一个json对象，而不是一个数值，这不是我们想要的

# JSON序列化问题

新增一个序列化转换器,逻辑很简单:

+ 对于反序列化,我们读取数值，并使用该数值创建一个StudentId
+ 对于序列化, 我们只需要将StudentId的数值写入即可

```C#
public class StudentIdJsonConverter : JsonConverter<StudentId> 
{
    public override StudentId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        if (reader.TokenType is JsonTokenType.Null)
            throw new ArgumentNullException(nameof(StudentId));

        var value = JsonSerializer.Deserialize<int>(ref reader, options);
        return new StudentId(value);
    }

    public override void Write(Utf8JsonWriter writer, StudentId value, JsonSerializerOptions options)
    {
        if (value is null)
            writer.WriteNullValue();
        else
            JsonSerializer.Serialize(writer, value.Value, options);
    }
}
```

同时，我们需要在启动代码前注入JSON序列化配置

```c#
builder.Services.AddControllers()
                .AddJsonOptions(options =>
                {
                    options.JsonSerializerOptions.Converters.Add(
                        new StudentIdJsonConverter());
                });
```

F5再次运行，序列化已正常

```
{
    "id": 1,
    "name": "张三"
}
```



