---
title: 使用泛型转换强类型Id
date: 2023-05-03 14:22:38 +0800
categories: [ASP.NET Core]
tags: [aspnetcore]
mermaid: true
---

![banner](https://raw.githubusercontent.com/liugt34/imagegallery/main/beautiful-park-1.jpg)

在上一章中，我们介绍了如何使用强类型Id。在使用的过程中也发现了一些问题，比如在类型转换时需要对每个强类型都需要进行转换。今天我们来学习一下如何使用泛型，将类型转换做成一个通用类。

# 类型通用转换

使用 **record** 创建一个泛型类型, 然后使 ***StudentId, ClassRoomId, TeacherId*** 继承该类

```C#
public abstract record StronglyTypedId<TValue>(TValue Value) where TValue : notnull
{
    public override string ToString() => Value.ToString();
}

public record StudentId(int Value) : StronglyTypedId<int>(Value);
```

现在我们有了一个通用的基类，这样我们就可以编写一个通用转换器了。它将比仅针对StudentIdConvert的内容更复杂，但我们只需要写一次。

首先我们需要创建一个助手类，主要用途如下：

+ 检查类型是否为强类型id，并获取值的类型

+ 创建并缓存委托，以根据值创建强类型id的实例

```C#
public static class StronglyTypedIdHelper
{
    //缓存委托
    private static readonly ConcurrentDictionary<Type, Delegate> StronglyTypedIdFactories = new();

    public static Func<TValue, object> GetFactory<TValue>(Type stronglyTypedIdType) where TValue : notnull
    {
        return (Func<TValue, object>)StronglyTypedIdFactories.GetOrAdd(stronglyTypedIdType, CreateFactory<TValue>);
    }

    //根据强类型Type返回相应的实例
    private static Func<TValue, object> CreateFactory<TValue>(Type stronglyTypedIdType) where TValue : notnull
    {
        if (!IsStronglyTypedId(stronglyTypedIdType))
        {
            throw new ArgumentException($"Type '{stronglyTypedIdType}' is not a strongly-typed id type", nameof(stronglyTypedIdType));
        }

        var ctor = stronglyTypedIdType.GetConstructor(new[] { typeof(TValue) });
        if (ctor is null)
        {
            throw new ArgumentException($"Type '{stronglyTypedIdType}' doesn't have a constructor with one parameter of type '{typeof(TValue)}'", nameof(stronglyTypedIdType));
        }

        var param = Expression.Parameter(typeof(TValue), "value");
        var body = Expression.New(ctor, param);
        var lambda = Expression.Lambda<Func<TValue, object>>(body, param);
        return lambda.Compile();
    }

    //判断是否是强类型
    public static bool IsStronglyTypedId(Type type) => IsStronglyTypedId(type, out _);

    public static bool IsStronglyTypedId(Type type, [NotNullWhen(true)] out Type idType)
    {
        if (type is null)
        {
            throw new ArgumentNullException(nameof(type));
        }

        if (type.BaseType is Type baseType && baseType.IsGenericType && baseType.GetGenericTypeDefinition() == typeof(StronglyTypedId<>))
        {
            idType = baseType.GetGenericArguments()[0];
            return true;
        }

        idType = null;
        return false;
    }
}
```

编写类型转换器

```c#
public class StronglyTypedIdConverter<TValue> : TypeConverter where TValue : notnull
{
    private static readonly TypeConverter IdValueConverter = GetIdValueConverter();
    //获取类型转换器
    private static TypeConverter GetIdValueConverter()
    {
        var converter = TypeDescriptor.GetConverter(typeof(TValue));

        if (!converter.CanConvertFrom(typeof(string)))
        {
            throw new InvalidOperationException($"Type '{typeof(TValue)}' doesn't have a converter that can convert from string");
        }

        return converter;
    }

    private readonly Type _type;
    public StronglyTypedIdConverter(Type type)
    {
        _type = type;
    }

    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
    {
        return sourceType == typeof(string) || sourceType == typeof(TValue) || base.CanConvertFrom(context, sourceType);
    }

    public override bool CanConvertTo(ITypeDescriptorContext context, Type destinationType)
    {
        return destinationType == typeof(string) || destinationType == typeof(TValue) || base.CanConvertTo(context, destinationType);
    }

    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
    {
        if (value is string s)
        {
            value = IdValueConverter.ConvertFrom(s);
        }

        if (value is TValue idValue)
        {
            //判断是不是泛型强类型
            var factory = StronglyTypedIdHelper.GetFactory<TValue>(_type);
            return factory(idValue);
        }

        return base.ConvertFrom(context, culture, value);
    }

    public override object ConvertTo(ITypeDescriptorContext context, CultureInfo culture, object value, Type destinationType)
    {
        if (value is null)
        {
            throw new ArgumentNullException(nameof(value));
        }

        var stronglyTypedId = (StronglyTypedId<TValue>)value;
        
        TValue idValue = stronglyTypedId.Value;
        if (destinationType == typeof(string))
        {
            return idValue.ToString()!;
        }

        if (destinationType == typeof(TValue))
        {
            return idValue;
        }

        return base.ConvertTo(context, culture, value, destinationType);
    }
}
```

这个转换器可以转换为字符串和TValue，这应该满足我们的需求。此时我们将其添加到泛型强类型的特性上

```C#
[TypeConverter(typeof(StronglyTypedIdConverter))]
public abstract record StronglyTypedId<TValue>(TValue Value) where TValue : notnull
{
    public override string ToString() => Value.ToString();
}
```

Oooops!!! 报错了，这是因为转换器类型不能是开放的泛型类型。因此我们需要一个非通用的中间转换器，它将创建实际的转换器并委托给它！

```C#
public class StronglyTypedIdConverter : TypeConverter
{
    //转换器缓存
    private static readonly ConcurrentDictionary<Type, TypeConverter> ActualConverters = new();

    private readonly TypeConverter _innerConverter;

    public StronglyTypedIdConverter(Type stronglyTypedIdType)
    {
        _innerConverter = ActualConverters.GetOrAdd(stronglyTypedIdType, CreateActualConverter);
    }

    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType) =>
        _innerConverter.CanConvertFrom(context, sourceType);
    public override bool CanConvertTo(ITypeDescriptorContext context, Type destinationType) =>
        _innerConverter.CanConvertTo(context, destinationType);
    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value) =>
        _innerConverter.ConvertFrom(context, culture, value);
    public override object ConvertTo(ITypeDescriptorContext context, CultureInfo culture, object value, Type destinationType) =>
        _innerConverter.ConvertTo(context, culture, value, destinationType);

	//创建转换器
    private static TypeConverter CreateActualConverter(Type stronglyTypedIdType)
    {
        if (!StronglyTypedIdHelper.IsStronglyTypedId(stronglyTypedIdType, out var idType))
            throw new InvalidOperationException($"The type '{stronglyTypedIdType}' is not a strongly typed id");
		//使用反射创建转换器实例
        var actualConverterType = typeof(StronglyTypedIdConverter<>).MakeGenericType(idType);
        return (TypeConverter)Activator.CreateInstance(actualConverterType, stronglyTypedIdType)!;
    }
}
```

OK，现在我们把之前的转换器都删除，重新 F5运行测试成功。

```c#
{
    "id": 1,
    "name": "张三"
}
```



# JSON通用转换

解决了类型转换问题，下一步就是JSON的通用转换了，当前我们还是每一个类型都专门写了JSON转换器来进行处理。

现在我们移除**StudentIdJsonConverter**类，新建**StronglyTypedIdJsonConverter** 泛型转换类

```C#
public class StronglyTypedIdJsonConverter<TStronglyTypedId, TValue> : JsonConverter<TStronglyTypedId>
    where TStronglyTypedId : StronglyTypedId<TValue>
    where TValue : notnull
{
    public override TStronglyTypedId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        if (reader.TokenType is JsonTokenType.Null)
            return null;

        var value = JsonSerializer.Deserialize<TValue>(ref reader, options);
        //利用上一个步骤中的强类型帮助类动态创建类型
        var factory = StronglyTypedIdHelper.GetFactory<TValue>(typeToConvert);
        return (TStronglyTypedId)factory(value);
    }

    public override void Write(Utf8JsonWriter writer, TStronglyTypedId value, JsonSerializerOptions options)
    {
        if (value is null)
            writer.WriteNullValue();
        else
            JsonSerializer.Serialize(writer, value.Value, options);
    }
}
```

然后在services中将注入类修改为如下

```C#
services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.Converters.Add(new StronglyTypedIdJsonConverter<ProductId, int>());
    });
```

但是我们发现这里还是要为每一个类型都添加一个泛型的转换器，这不是我们想要的结果。我们需要构建一个工厂类继承**JsonConverterFactory** ， 重写转换器创建方法，根据转换器类型自动创建JSON转换器。

```C#
public class StronglyTypedIdJsonConverterFactory : JsonConverterFactory
{
    private static readonly ConcurrentDictionary<Type, JsonConverter> Cache = new();

    public override bool CanConvert(Type typeToConvert)
    {
        return StronglyTypedIdHelper.IsStronglyTypedId(typeToConvert);
    }

    public override JsonConverter CreateConverter(Type typeToConvert, JsonSerializerOptions options)
    {
        return Cache.GetOrAdd(typeToConvert, CreateConverter);
    }

    private static JsonConverter CreateConverter(Type typeToConvert)
    {
        if (!StronglyTypedIdHelper.IsStronglyTypedId(typeToConvert, out var valueType))
            throw new InvalidOperationException($"Cannot create converter for '{typeToConvert}'");

        var type = typeof(StronglyTypedIdJsonConverter<,>).MakeGenericType(typeToConvert, valueType);
        return (JsonConverter)Activator.CreateInstance(type);
    }
}
```

修改一下注入内容

```
services.AddControllers()
        .AddJsonOptions(options =>
        {
            options.JsonSerializerOptions.Converters.Add(new StronglyTypedIdJsonConverterFactory());
        });
```

重写运行，成功返回我们想要的结果。

```
{
    "id": 1,
    "name": "张三"
}
```

# 使用Newtonsoft

深吐一口气，好复杂！ 那有没有简单的的方法呢，那必须有。如果你的项目中使用了**Newtonsoft.Json** 那么恭喜你，序列化这些你完全不需要去管他，**Newtonsoft** 会自动为你寻找相对于的TypeConvert。你只需将AddJsonOptions修改为AddNewtonsoftJson即可。😊

