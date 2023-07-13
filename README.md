# TableCraft.Core

[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)

[README.en.md](README.en.md)

## 简介

TableCraft.Core是一个.NET库，它可以根据规则，从不同类型的配置文件提取出关键信息（配置文件名，字段名等），并使用额外的描述文件进行信息补充（字段类型，字段标签），以规定的JSON格式输出一份配置文件的完整信息。

同时，TableCraft.Core还能够使用这份配置文件信息，根据你提供的模板生成任何你想要的文本（这在生成一些规则重复的代码文件时非常有用）。

## 特性

* 声明可用的字段数值类型，集合类型
* 为字段添加标签，以便在生成文本时进行特殊处理（例如表格的主键）
* 可支持不同文件类型的配置文件/数据源（目前仅支持csv）
* 可支持不同文件类型的数据描述（目前仅支持JSON）
* 基于[T4 Text Templates](https://learn.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates?view=vs-2022)的文本模板渲染，使用TableCraft.Core提供的API生成任意语言的业务代码，修改生成规则无需重新编译
* 可提供版本控制支持（目前仅支持perforce）

## 示例

现有 Students.csv：

| ID   | ClassID | Name   | Age  | Courses                 |
| ---- | ------- | ------ | ---- | ----------------------- |
| 0    | 1       | Edward | 24   | CS61A;MIT18.01;MIT18.06 |
| 1    | 1       | Alex   | 24   | UCB CS61B;MIT 6.006     |

通过额外的一份JSON文件加以描述，补充字段类型，标签等：

```jsonc
{
    "ConfigName" : "Students",
    "Usage"      : {
        "usage0" : {
            "ExportName" : "StudentsTable"
        }
    },
    "Attributes" : [
        {
            "AttributeName" : "ID",
            "Comment"       : "Identifier",
            "ValueType"     : "int",
            "DefaultValue"  : "",
            "CollectionType" : "none",
            "Usages"         : [
                {
                    "Usage" : "usage0",
                    "FieldName" : "ID"
                }
            ],
            "Tag"            : [
                "primary"
            ]
        },
        // ...
    ]
}
```

自己完成这样一份JSON文件的配置可能让人比较沮丧，你可以开发自己的可视化工具，接入TableCraft.Core，完成JSON文件的生成；或者你可以看一眼[TableCraft.Editor](https://github.com/kalulas/TableCraft)，一个基于AvaloniaUI的可视化工具示例。

现在通过完成一份文本模板（.tt）来简单描述你的生成规则：

```text
<#@ template debug="true" hostspecific="true" language="C#" #>
<#@ parameter type="System.String" name="CurrentUsage"#>
<#
    var configInfoObject = Host.GetHostOption("CurrentConfigInfo");
    var configInfo = configInfoObject as ConfigInfo;
    if (configInfo == null) throw new Exception("null CurrentConfigInfo received, exit!");
#>
namespace Foo {
    public class <#= configInfo.GetExportName(CurrentUsage) #> {
<# 
    foreach( var info in configInfo.AttributeInfos ){ 
        if(info.IsValid() && info.HasUsage(CurrentUsage)){
#>
            public <#= info.ValueType #> <#= info.GetUsageFieldName(CurrentUsage) #> { get; private set; }
<#
        }
    }
#>
    }
}
```

大功告成，现在TableCraft.Core将会为你输出以下结果：

```csharp
namespace Foo {
    public class StudentsTable {
            public int ID { get; private set; }
            public uint ClassID { get; private set; }
            public string Name { get; private set; }
            public uint Age { get; private set; }
            public string Courses { get; private set; }
    }
}
```

## 配置方式

### libenv.json

使用运行时库TableCraft.Core需要配置`libenv.json`文件，并在使用前将文件通过方法`TableCraft.Core.Configuration.ReadConfigurationFromJson`传入，进行初始化

```jsonc
{
    // 规定数值类型
    "DataValueType": ["int", "uint", "float", "boolean", "string"],
    // 规定集合类型
    "DataCollectionType": ["none", "array"],
    // 规定可用的字段标签
    "AttributeTag": ["primary", "label1", "label2"],
    // TableCraft默认使用UTF8编码，在此指定是否需要BOM头
    "UTF8BOM": false,
    // 对csv类型数据源文件，规定字段名所在的行数，与注释所在的行数（若不存在则填-1）
    "CsvSource":{
        "HeaderLineIndex": 0,
        "CommentLineIndex": 1
    },
    // 规定各种导出的代码途径
    "ConfigUsageType":{
        // 将使用可执行文件同级"Templates"目录下的"usage0-template.tt"进行文本能生成，
        // 且生成文件使用"{0}_base"为格式字符串，使用".cs"后缀
        "usage0":{
            "CodeTemplateName": "usage0-template.tt",
            "TargetFileType": ".cs",
            "OutputFormat": "{0}_base"
        }
    },
    // 导出途径组，用于同时导出多种途径
    "ConfigUsageGroup":{
        "group0":[
            "usage0",
            "usage1"
        ]
    }
}
```

## License

[MIT](http://opensource.org/licenses/MIT)

Copyright (c) 2023 - Boming Chen
