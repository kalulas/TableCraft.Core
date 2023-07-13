# TableCraft.Core

[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)

## Overview

TableCraft.Core is a .NET library that can extract key information (such as configuration file names and field names) from different types of configuration files based on rules. It also uses additional description files to supplement information about field types and labels, and outputs complete configuration file information in the specified JSON format.

In addition, TableCraft.Core can use this configuration file information to generate any text you want based on the template you provide (which is very useful for generating code files with repeated rules).

## Features

* Declare available field value types and collection types
* Add labels to fields for special handling when generating text (such as primary keys in tables)
* Support configuration files/data sources of different file types (currently only CSV is supported)
* Support data description of different file types (currently only JSON is supported)
* Text template rendering based on [T4 Text Templates](https://learn.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates?view=vs-2022), generate business code in any language using the API provided by TableCraft.Core, modify generation rules without recompiling
* Provide version control support (currently only Perforce)

## Example

Assuming we have a Students.csv file:

| ID   | ClassID | Name   | Age  | Courses                 |
| ---- | ------- | ------ | ---- | ----------------------- |
| 0    | 1       | Edward | 24   | CS61A;MIT18.01;MIT18.06 |
| 1    | 1       | Alex   | 24   | UCB CS61B;MIT 6.006     |

By supplementing field types, tags, etc. through an additional JSON file:

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

Completing such a JSON configuration file on your own may be frustrating. You can develop your own visualization tool and integrate it with TableCraft.Core to generate the JSON file; or you can take a look at [TableCraft.Editor](https://github.com/kalulas/TableCraft), which is a visual tool example based on AvaloniaUI.

Now by completing a text template (.tt) to simply describe your generation rules:

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

Congratulations! TableCraft.Core will now output the following results for you:

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

## Configuration

### libenv.json

To use the TableCraft.Core runtime library, you need to configure the `libenv.json` file and pass it in through the `TableCraft.Core.Configuration.ReadConfigurationFromJson` method for initialization before use.

```jsonc
{
    // Define data value types
    "DataValueType": ["int", "uint", "float", "boolean", "string"],
    // Define collection types
    "DataCollectionType": ["none", "array"],
    // Define available field tags
    "AttributeTag": ["primary", "label1", "label2"],
    // TableCraft uses UTF8 encoding by default. Specify whether a BOM header is needed here.
    "UTF8BOM": false,
     // For csv type data source files, specify the row number where field names are located and where comments are located (if they do not exist, fill in -1).
     "CsvSource":{
        "HeaderLineIndex": 0,
        "CommentLineIndex": 1
     },
     // Specify various export code methods.
    "ConfigUsageType":{
        "usage0":{
            // T4 template file used to generate code. This file needs to be placed in the Templates directory at the same level as the executable file.
            "CodeTemplateName":"usage0-template.tt",
            // The type of generated file. In this example, c# code is generated.
            "TargetFileType":".cs",
            // The format string of the generated file name, if this string contains a file type, it will be replaced by TargetFileType
            "OutputFormat": "{0}_base"
        }
     },
    // Specify group to support exporting multiple files for each usage
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
