---
title: 使用dotnet core创建一个可执行的命令行程序
date: 2023-07-06 15:37:00 +0800
categories: [C#]
tags: [dotnet]
mermaid: true
---

# 为何要做命令行程序

对于大多数开发者而言，图形化工具（GUI）更简单更直观，比如比如图形化的Git工具，各类IDE等等。那为什么还需要做命令行程序呢？

我个人认为有以下几点。

1. 命令行程序脱离了GUI，更节省系统消耗
2. 兼容不同的操作系统，跨平台可移植性强
3. 可以方便不同的开发语言调用，无需再去开发烦人的API



# 案例

今天突然接到客户的临时需求，说是要将一个文件夹下的图片全部合并成一个PDF文件。之前这个系统是用nodejs写的，然后nodejs上并没有太好的解决方案。本来是想着用dotnet core写一个API服务，但是API一是要去访问nodejs运行目录下的文件，二是开发人员又不想花太多精力去做API代理。于是就想到做一个控制它命令，让nodejs直接调用命令行来处理。



# 解决方案

## 第一步：创建项目并安装System.CommandLine

微软推出了一款命令行库  [NuGet Gallery | System.CommandLine](https://www.nuget.org/packages/System.CommandLine) 用来解析命令行中所有的参数。 我们新建一个dotnet 6.0项目，然后引用该库

当前还是预览版，所以需要添加版本号

```bash
dotnet add package System.CommandLine --version 2.0.0-beta4.22272.1
```

创建项目的时候注意 **勾选 Do not use top-level statements** ，否则无法生成main函数。

## 第二步：定义命令

定义我们想用的命令格式

```bash
pdftool merge -d E:\test -n=test.pdf
```

1. pdftool 是命令名称
2. merge  是子命令
3. -d 是参数 --dir 的简写，表示图片所在文件夹
4. -n 是参数 --name的简写，表示导出pdf文件的名称

## 第三步：编写代码

```c#
static async Task<int> Main(string[] args)
{
    // 定义 Root command
    var command = new RootCommand("This is a tool for pdf. has functions like merge images to pdf,import pdf page to images");
    
    // 定义子命令
    var mergeCommand = new Command("merge", "Merge Images to pdf,support jpg and png only");
    
    // 定义参数
    var dirOption = new Option<string>("--dir", "The images directory path");
    dirOption.IsRequired = true;
    //参数简写
    dirOption.AddAlias("-d"); 
    
    //定义参数校验，例如:检验文件夹是否存在
    dirOption.AddValidator((option) =>
    {
        var dir = option.GetValueForOption<string>(dirOption);

        if (!Directory.Exists(dir))
        {
            option.ErrorMessage = $"The images directory '{dir}' doesn't exist";
        }
    });
    mergeCommand.AddOption(dirOption);
    
    //同理添加参数 name
    ...
        
    //添加该命令处理方法
    mergeCommand.SetHandler(
        (dir, name) =>  
        {
            //处理方法，这里如何处理PDF大家可以自定义
            MergeImagesInDirectory(dir, name);
        },
        dirOption, nameOption);

    command.AddCommand(mergeCommand);
    
    //等待调用
    return await command.InvokeAsync(args);
}
```



完整代码已上传github   [liugt34/pdftool (github.com)](https://github.com/liugt34/pdftool)



