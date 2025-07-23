# ET框架ExcelExporter配置系统详解

## 概述

**ExcelExporter是ET框架的配置数据转换工具**，它将Excel配置表转换为游戏可以直接使用的代码和数据文件。这个工具实现了从策划Excel表到游戏运行时高效配置系统的完整转换流程。

## 访问方式

### Unity编辑器菜单
```
ET -> BuildTool -> ExcelExporter
```

### 命令行调用
```bash
# Windows
Tool.exe --AppType=ExcelExporter --Console=1

# Linux/macOS  
./Tool --AppType=ExcelExporter --Console=1
```

## 主要功能

### 1. Excel to C# Class Generation（Excel转C#类生成）

ExcelExporter会根据Excel表结构自动生成强类型的C#配置类：

```csharp
// 自动生成的配置类示例
[Config]
public partial class UnitConfigCategory : Singleton<UnitConfigCategory>, IMerge
{
    [BsonElement]
    [BsonDictionaryOptions(DictionaryRepresentation.ArrayOfArrays)]
    private Dictionary<int, UnitConfig> dict = new();

    public UnitConfig Get(int id)
    {
        this.dict.TryGetValue(id, out UnitConfig item);
        if (item == null)
        {
            throw new Exception($"配置找不到，配置表名: {nameof(UnitConfig)}，配置id: {id}");
        }
        return item;
    }

    public bool Contain(int id) => this.dict.ContainsKey(id);
    public Dictionary<int, UnitConfig> GetAll() => this.dict;
}

public partial class UnitConfig: ProtoObject, IConfig
{
    /// <summary>Id</summary>
    public int Id { get; set; }
    /// <summary>名字</summary>
    public string Name { get; set; }
    /// <summary>位置</summary>
    public int Position { get; set; }
    /// <summary>身高</summary>
    public int Height { get; set; }
    /// <summary>体重</summary>
    public int Weight { get; set; }
}
```

### 2. Excel to JSON Conversion（Excel转JSON数据）

将Excel数据转换为结构化的JSON格式：

```json
{
  "dict": [
    [1001, {"_t":"UnitConfig","_id":1001,"Type":1,"Name":"米克尔","Position":1,"Height":178,"Weight":68}],
    [1002, {"_t":"UnitConfig","_id":1002,"Type":1,"Name":"米克尔2","Position":2,"Height":278,"Weight":78}],
    [1003, {"_t":"UnitConfig","_id":1003,"Type":1,"Name":"米克尔3","Position":1,"Height":178,"Weight":68}]
  ]
}
```

### 3. JSON to Binary Protobuf（JSON转二进制数据）

最终生成二进制.bytes文件供游戏运行时快速加载：

```csharp
// 生成二进制配置文件
string path = Path.Combine(dir, $"{protoName}Category.bytes");
using FileStream file = File.Create(path);
file.Write(final.ToBson());
```

## Excel文件格式规范

### 文件命名规则

```
格式: ConfigName[@cs].xlsx

示例:
- UnitConfig.xlsx           # 客户端服务端共享配置
- StartMachineConfig@s.xlsx # 仅服务端配置  
- PlayerConfig@c.xlsx       # 仅客户端配置
- ItemConfig@cs.xlsx        # 显式指定共享配置
```

**命名约定**:
- 无后缀: 默认为`cs`（共享）
- `@c`: 仅客户端使用
- `@s`: 仅服务端使用  
- `@cs`: 客户端服务端共享

### Excel表格结构标准

```
| Row | Col A | Col B | Col C    | Col D      | Col E        | Col F       |
|-----|-------|-------|----------|------------|--------------|-------------|
| 1   |       |       | cs       | cs         | cs           | cs          |
| 2   |       |       | Id       | 类型       | 名字         | 位置        |
| 3   |       |       | Id       | Type       | Name         | Position    |
| 4   |       |       | int      | int        | string       | int         |
| 5   |       |       |          |            |              |             |
| 6   |       | cs    | 1001     | 1          | 米克尔       | 1           |
| 7   |       | cs    | 1002     | 1          | 米克尔2      | 2           |
```

**行定义**:
- **Row 1**: CS字段标识（c/s/cs），决定该字段在哪些端使用
- **Row 2**: 字段中文说明（用于生成注释）
- **Row 3**: 字段英文名称（生成C#属性名）
- **Row 4**: 字段数据类型（int/string/float等）
- **Row 5**: 空行分隔符
- **Row 6+**: 实际配置数据

**列定义**:
- **Col A**: 保留列
- **Col B**: 行标识（cs/c/s），决定该行数据在哪些端使用
- **Col C+**: 配置字段数据

### 支持的数据类型

```csharp
// 基础类型
int, uint, int32, int64, long, float, double, string

// 数组类型
int[], uint[], int32[], long[], string[]

// 复杂数组
int[][] (二维数组)

// 类型转换示例
private static string Convert(string type, string value)
{
    switch (type)
    {
        case "int[]":
            return $"[{value}]";        // 1,2,3 -> [1,2,3]
        case "string":
            return $"\"{value}\"";      // hello -> "hello"
        case "int":
            return value == "" ? "0" : value; // 空值转0
        default:
            throw new Exception($"不支持此类型: {type}");
    }
}
```

## 配置类型分类系统

### ConfigType枚举

```csharp
public enum ConfigType
{
    c = 0,    // Client Only - 仅客户端
    s = 1,    // Server Only - 仅服务端  
    cs = 2,   // Client Server - 客户端服务端共享
}
```

### 生成目录结构

```
Unity/Assets/Scripts/Model/Generate/
├── Client/Config/           # 客户端专用配置类
│   └── PlayerUIConfig.cs
├── Server/Config/           # 服务端专用配置类  
│   └── StartMachineConfig.cs
└── ClientServer/Config/     # 共享配置类
    ├── UnitConfig.cs
    └── ItemConfig.cs
```

### 字段级别控制

Excel中可以精确控制每个字段的使用范围：

```
| cs字段标识 | s字段标识  | c字段标识  |
|------------|------------|------------|
| Id         | ServerId   | UIIcon     |
| Name       | InternalId | DisplayName|
| Type       | AdminOnly  | ClientTip  |
```

## 核心处理流程

### 1. 文件扫描和过滤

```csharp
public static void Export()
{
    // 扫描Excel目录
    List<string> files = FileHelper.GetAllFiles(excelDir);
    
    foreach (string path in files)
    {
        string fileName = Path.GetFileName(path);
        
        // 过滤规则
        if (!fileName.EndsWith(".xlsx") ||      // 必须是Excel文件
            fileName.StartsWith("~$") ||        // 排除临时文件
            fileName.Contains("#"))             // 排除注释文件
        {
            continue;
        }
        
        ProcessExcelFile(path);
    }
}
```

### 2. Excel表头解析

```csharp
static void ExportSheetClass(ExcelWorksheet worksheet, Table table)
{
    const int row = 2;
    for (int col = 3; col <= worksheet.Dimension.End.Column; ++col)
    {
        // 跳过注释表（表名以#开头）
        if (worksheet.Name.StartsWith("#"))
            continue;

        // 读取表头信息
        string fieldCS = worksheet.Cells[row, col].Text.Trim().ToLower();     // Row 2: cs标识
        string fieldDesc = worksheet.Cells[row + 1, col].Text.Trim();        // Row 3: 字段说明
        string fieldName = worksheet.Cells[row + 2, col].Text.Trim();        // Row 4: 字段名
        string fieldType = worksheet.Cells[row + 3, col].Text.Trim();        // Row 5: 字段类型

        // 跳过注释字段（包含#号）
        if (fieldCS.Contains("#"))
        {
            table.HeadInfos[fieldName] = null;
            continue;
        }

        // 存储字段信息
        table.HeadInfos[fieldName] = new HeadInfo(fieldCS, fieldDesc, fieldName, fieldType, ++table.Index);
    }
}
```

### 3. C#类代码生成

```csharp
static void ExportClass(string protoName, Dictionary<string, HeadInfo> classField, ConfigType configType)
{
    // 根据模板生成类代码
    StringBuilder sb = new StringBuilder();
    
    foreach ((string _, HeadInfo headInfo) in classField)
    {
        if (headInfo == null) continue;
        
        // 检查字段是否适用于当前配置类型
        if (configType != ConfigType.cs && !headInfo.FieldCS.Contains(configType.ToString()))
            continue;

        // 生成字段定义
        sb.Append($"\t\t/// <summary>{headInfo.FieldDesc}</summary>\n");
        sb.Append($"\t\tpublic {headInfo.FieldType} {headInfo.FieldName} {{ get; set; }}\n");
    }

    // 使用模板替换生成最终代码
    string content = template
        .Replace("(ConfigName)", protoName)
        .Replace("(Fields)", sb.ToString());
    
    // 写入文件
    string exportPath = Path.Combine(GetClassDir(configType), $"{protoName}.cs");
    File.WriteAllText(exportPath, content);
}
```

### 4. 动态编译验证

```csharp
private static Assembly DynamicBuild(ConfigType configType)
{
    string classPath = GetClassDir(configType);
    List<SyntaxTree> syntaxTrees = new List<SyntaxTree>();
    
    // 收集所有生成的C#文件
    foreach (string classFile in Directory.GetFiles(classPath, "*.cs"))
    {
        syntaxTrees.Add(CSharpSyntaxTree.ParseText(File.ReadAllText(classFile)));
    }

    // 收集程序集引用
    List<PortableExecutableReference> references = new List<PortableExecutableReference>();
    foreach (Assembly assembly in AppDomain.CurrentDomain.GetAssemblies())
    {
        if (!assembly.IsDynamic && assembly.Location != "")
        {
            references.Add(MetadataReference.CreateFromFile(assembly.Location));
        }
    }

    // 编译验证
    CSharpCompilation compilation = CSharpCompilation.Create(null,
        syntaxTrees.ToArray(),
        references.ToArray(),
        new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary));

    using MemoryStream memSteam = new MemoryStream();
    EmitResult emitResult = compilation.Emit(memSteam);
    
    if (!emitResult.Success)
    {
        StringBuilder errorMsg = new StringBuilder();
        foreach (Diagnostic error in emitResult.Diagnostics)
        {
            errorMsg.Append($"{error.GetMessage()}\n");
        }
        throw new Exception($"动态编译失败:\n{errorMsg}");
    }

    // 返回编译后的程序集
    memSteam.Seek(0, SeekOrigin.Begin);
    return Assembly.Load(memSteam.ToArray());
}
```

### 5. 数据导出和序列化

```csharp
// JSON数据导出
static void ExportExcelJson(ExcelPackage package, string name, Table table, ConfigType configType, string relativeDir)
{
    StringBuilder sb = new StringBuilder();
    sb.Append("{\"dict\": [\n");
    
    foreach (ExcelWorksheet worksheet in package.Workbook.Worksheets)
    {
        if (worksheet.Name.StartsWith("#")) continue;
        
        // 从第6行开始读取数据
        for (int row = 6; row <= worksheet.Dimension.End.Row; ++row)
        {
            string rowPrefix = worksheet.Cells[row, 2].Text.Trim();
            
            // 跳过注释行和不匹配的配置类型
            if (rowPrefix.Contains("#")) continue;
            if (rowPrefix == "") rowPrefix = "cs";
            if (configType != ConfigType.cs && !rowPrefix.Contains(configType.ToString())) continue;

            // 生成JSON条目
            string id = worksheet.Cells[row, 3].Text.Trim();
            if (id == "") continue;

            sb.Append($"[{id}, {{\"_t\":\"{name}\"");
            
            // 处理各字段数据
            for (int col = 3; col <= worksheet.Dimension.End.Column; ++col)
            {
                string fieldName = worksheet.Cells[4, col].Text.Trim();
                if (!table.HeadInfos.ContainsKey(fieldName)) continue;
                
                HeadInfo headInfo = table.HeadInfos[fieldName];
                if (headInfo == null) continue;
                if (configType != ConfigType.cs && !headInfo.FieldCS.Contains(configType.ToString())) continue;

                string jsonFieldName = headInfo.FieldName == "Id" ? "_id" : headInfo.FieldName;
                string cellValue = worksheet.Cells[row, col].Text.Trim();
                
                sb.Append($",\"{jsonFieldName}\":{Convert(headInfo.FieldType, cellValue)}");
            }
            
            sb.Append("}],\n");
        }
    }
    
    sb.Append("]}\n");
    
    // 写入JSON文件
    string jsonPath = Path.Combine(string.Format(jsonDir, configType.ToString(), relativeDir), $"{name}.txt");
    Directory.CreateDirectory(Path.GetDirectoryName(jsonPath));
    File.WriteAllText(jsonPath, sb.ToString());
}

// 二进制数据导出
private static void ExportExcelProtobuf(ConfigType configType, string protoName, string relativeDir)
{
    Assembly assembly = GetAssembly(configType);
    Type categoryType = assembly.GetType($"ET.{protoName}Category");
    
    // 创建配置类实例
    IMerge finalConfig = Activator.CreateInstance(categoryType) as IMerge;

    // 读取并合并所有相关JSON文件
    string jsonDir = Path.Combine(string.Format(jsonDir, configType, relativeDir));
    string[] jsonFiles = Directory.GetFiles(jsonDir, $"{protoName}*.txt");
    
    foreach (string jsonPath in jsonFiles.OrderBy(x => x))
    {
        string json = File.ReadAllText(jsonPath);
        try
        {
            object deserializedConfig = BsonSerializer.Deserialize(json, categoryType);
            finalConfig.Merge(deserializedConfig);
        }
        catch (Exception e)
        {
            throw new Exception($"JSON文件解析失败: {jsonPath}", e);
        }
    }

    // 序列化为二进制文件
    string binaryPath = Path.Combine(GetProtoDir(configType, relativeDir), $"{protoName}Category.bytes");
    Directory.CreateDirectory(Path.GetDirectoryName(binaryPath));
    
    using FileStream file = File.Create(binaryPath);
    file.Write(finalConfig.ToBson());
}
```

## 在游戏中使用配置

### 基本使用方式

```csharp
// 获取单个配置项
UnitConfig unitConfig = UnitConfigCategory.Instance.Get(1001);
Log.Debug($"Unit Name: {unitConfig.Name}, Height: {unitConfig.Height}");

// 检查配置是否存在
if (UnitConfigCategory.Instance.Contain(1001))
{
    // 配置存在，安全使用
}

// 获取所有配置
Dictionary<int, UnitConfig> allUnits = UnitConfigCategory.Instance.GetAll();
foreach (var kvp in allUnits)
{
    Log.Debug($"ID: {kvp.Key}, Name: {kvp.Value.Name}");
}

// 获取任意一个配置（通常用于获取默认值）
UnitConfig defaultUnit = UnitConfigCategory.Instance.GetOne();
```

### 高级使用场景

```csharp
// 配置查询和过滤
public class ConfigHelper
{
    // 根据条件查找配置
    public static List<UnitConfig> FindUnitsByType(int type)
    {
        return UnitConfigCategory.Instance.GetAll()
            .Values
            .Where(unit => unit.Type == type)
            .ToList();
    }
    
    // 配置缓存
    private static Dictionary<int, List<UnitConfig>> typeCache = new();
    
    public static List<UnitConfig> GetUnitsByTypeCached(int type)
    {
        if (!typeCache.TryGetValue(type, out var units))
        {
            units = FindUnitsByType(type);
            typeCache[type] = units;
        }
        return units;
    }
    
    // 配置验证
    public static bool ValidateUnitConfig(int unitId)
    {
        if (!UnitConfigCategory.Instance.Contain(unitId))
        {
            Log.Error($"Unit config not found: {unitId}");
            return false;
        }
        
        var config = UnitConfigCategory.Instance.Get(unitId);
        if (string.IsNullOrEmpty(config.Name))
        {
            Log.Error($"Unit name is empty: {unitId}");
            return false;
        }
        
        return true;
    }
}
```

## 输出文件结构

ExcelExporter执行后会生成以下文件结构：

```
项目根目录/
├── Unity/Assets/Scripts/Model/Generate/
│   ├── Client/Config/                    # 客户端配置类
│   │   ├── PlayerUIConfig.cs
│   │   └── ItemDisplayConfig.cs
│   ├── Server/Config/                    # 服务端配置类
│   │   ├── StartMachineConfig.cs
│   │   ├── StartProcessConfig.cs
│   │   └── ServerOnlyConfig.cs
│   └── ClientServer/Config/              # 共享配置类
│       ├── UnitConfig.cs
│       ├── ItemConfig.cs
│       └── AIConfig.cs
├── Config/
│   ├── Json/                            # JSON数据文件
│   │   ├── c/                           # 客户端JSON数据
│   │   │   ├── PlayerUIConfig.txt
│   │   │   └── ItemDisplayConfig.txt
│   │   ├── s/                           # 服务端JSON数据
│   │   │   ├── StartMachineConfig.txt
│   │   │   └── ServerOnlyConfig.txt
│   │   └── cs/                          # 共享JSON数据
│   │       ├── UnitConfig.txt
│   │       ├── ItemConfig.txt
│   │       └── AIConfig.txt
│   └── Excel/                           # 二进制数据文件
│       ├── c/                           # 客户端二进制数据
│       │   ├── PlayerUIConfigCategory.bytes
│       │   └── ItemDisplayConfigCategory.bytes
│       ├── s/                           # 服务端二进制数据
│       │   ├── StartMachineConfigCategory.bytes
│       │   └── ServerOnlyConfigCategory.bytes
│       └── cs/                          # 共享二进制数据
│           ├── UnitConfigCategory.bytes
│           ├── ItemConfigCategory.bytes
│           └── AIConfigCategory.bytes
└── Unity/Assets/Bundles/Config/         # 客户端打包用配置（从c/复制）
    ├── PlayerUIConfigCategory.bytes
    └── ItemDisplayConfigCategory.bytes
```

## 优势和特性

### 1. 类型安全
- **编译时检查**: 字段类型错误在编译时发现
- **IntelliSense支持**: IDE提供完整的代码提示
- **重构安全**: 字段名修改会自动更新所有引用

### 2. 高性能
- **二进制序列化**: BSON格式加载速度快
- **内存优化**: 数据结构紧凑，减少内存占用
- **惰性加载**: 支持按需加载配置

### 3. 多端架构支持
- **精确控制**: 字段级别的客户端/服务端分离
- **共享配置**: 减少重复数据，保证版本一致性
- **独立配置**: 敏感服务端数据不会泄露到客户端

### 4. 开发效率
- **策划友好**: 策划可直接编辑Excel，无需编程知识
- **自动化流程**: 一键转换，减少手工错误
- **热重载支持**: 配置修改后可快速测试
- **版本控制**: Excel文件和生成的代码都可以版本控制

### 5. 扩展性
- **模板化**: 通过Template.txt可以自定义生成的代码结构
- **多格式支持**: 同时生成JSON和二进制格式
- **插件机制**: 可以扩展支持更多数据类型和转换规则

## 最佳实践

### 1. Excel表设计规范
```
建议:
- 使用有意义的字段名，避免缩写
- 为每个字段添加详细的中文说明
- 合理使用cs标识，避免不必要的数据冗余
- 使用#注释不需要的字段或行

避免:
- 空的字段名或类型定义
- 字段类型与实际数据不匹配
- 在数据行中混用不同的cs标识
```

### 2. 配置管理策略
```csharp
// 配置预加载
public class ConfigManager : Singleton<ConfigManager>
{
    public void PreloadConfigs()
    {
        // 预加载常用配置，提高运行时性能
        _ = UnitConfigCategory.Instance;
        _ = ItemConfigCategory.Instance;
        _ = AIConfigCategory.Instance;
    }
    
    public void ValidateAllConfigs()
    {
        // 游戏启动时验证所有配置的完整性
        ValidateUnitConfigs();
        ValidateItemConfigs();
        // ... 其他配置验证
    }
}
```

### 3. 错误处理
```csharp
public static class SafeConfigHelper
{
    public static UnitConfig GetUnitConfigSafe(int id, UnitConfig defaultConfig = null)
    {
        try
        {
            return UnitConfigCategory.Instance.Get(id);
        }
        catch (Exception ex)
        {
            Log.Error($"Failed to get unit config {id}: {ex.Message}");
            return defaultConfig;
        }
    }
}
```

## 常见问题和解决方案

### 1. 编译错误
**问题**: 动态编译失败
**原因**: Excel中字段类型定义错误或字段名不符合C#命名规范
**解决**: 检查Excel表格式，确保字段名为有效的C#标识符

### 2. 数据不匹配
**问题**: 配置加载时数据类型错误
**原因**: Excel中数据格式与定义的字段类型不匹配
**解决**: 统一Excel数据格式，数值型字段不要包含非数字字符

### 3. 中文乱码
**问题**: 生成的配置文件中中文显示乱码
**原因**: Excel文件编码问题
**解决**: 确保Excel文件保存为UTF-8编码

### 4. 性能问题
**问题**: 配置加载缓慢
**原因**: 配置文件过大或频繁访问磁盘
**解决**: 使用配置预加载和内存缓存策略

**总结**: ExcelExporter是ET框架配置管理的核心工具，它通过标准化的Excel格式和自动化的代码生成，实现了高效、类型安全、多端兼容的游戏配置系统，极大提升了开发效率和配置管理的便利性。