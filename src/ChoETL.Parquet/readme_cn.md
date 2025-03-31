# ChoETL.Parquet 使用文档

## 目录

- [简介](#简介)
- [安装](#安装)
- [基础用法](#基础用法)
    - [读取 Parquet 文件](#读取-parquet-文件)
    - [写入 Parquet 文件](#写入-parquet-文件)
- [配置选项](#配置选项)
    - [通用配置](#通用配置)
    - [Reader 特有配置](#reader-特有配置)
    - [Writer 特有配置](#writer-特有配置)
- [高级用法](#高级用法)
    - [自定义类型转换](#自定义类型转换)
    - [异步操作](#异步操作)
    - [处理大文件](#处理大文件)
- [常见问题与解决方案](#常见问题与解决方案)
- [完整示例](#完整示例)

## 简介

ChoETL.Parquet 是 Cinchoo ETL 框架的一部分，专门用于处理 Apache Parquet 格式的数据文件。Parquet 是一种列式存储格式，非常适合于大数据处理场景，具有高效的压缩和编码方案 [1](https://github.com/Cinchoo/ChoETL)。

ChoETL.Parquet 提供了简单易用的 API 来读取和写入 Parquet 格式的数据，支持直接映射到 .NET POCO 对象，具有高度的灵活性和可配置性。

## 安装

### .NET Framework 版本

通过 NuGet Package Manager 安装：

```
PM> Install-Package ChoETL.Parquet
```

### .NET Standard / .NET Core 版本

通过 NuGet Package Manager 安装：

```
PM> Install-Package ChoETL.NETStandard
```

然后引入命名空间：

```csharp
using ChoETL;
```

## 基础用法

### 读取 Parquet 文件

使用 `ChoParquetReader` 从 Parquet 文件读取数据：

```csharp
// 定义一个与 Parquet 文件结构匹配的 POCO 类
public class Employee
{
    public int ID { get; set; }
    public string Name { get; set; }
    public string Department { get; set; }
    public double Salary { get; set; }
    public DateTime JoinDate { get; set; }
}

// 从 Parquet 文件读取数据
using (var reader = new ChoParquetReader<Employee>("employees.parquet"))
{
    foreach (var employee in reader)
    {
        Console.WriteLine($"{employee.ID}, {employee.Name}, {employee.Department}, {employee.Salary}, {employee.JoinDate}");
    }
}
```

### 写入 Parquet 文件

使用 `ChoParquetWriter` 将数据写入 Parquet 文件：

```csharp
// 准备数据
var employees = new List<Employee>
{
    new Employee { ID = 1, Name = "张三", Department = "研发", Salary = 15000, JoinDate = new DateTime(2020, 1, 15) },
    new Employee { ID = 2, Name = "李四", Department = "市场", Salary = 12000, JoinDate = new DateTime(2019, 5, 10) },
    new Employee { ID = 3, Name = "王五", Department = "人事", Salary = 10000, JoinDate = new DateTime(2021, 3, 20) }
};

// 写入 Parquet 文件
using (var writer = new ChoParquetWriter<Employee>("employees.parquet"))
{
    writer.Write(employees);
}
```

## 配置选项

### 通用配置

ChoETL.Parquet 提供了多种配置选项来自定义读取和写入行为：

```csharp
// 通过 fluent API 设置配置
var configuration = new ChoParquetRecordConfiguration<Employee>()
    .Configure(c => c.Culture = CultureInfo.InvariantCulture)
    .Configure(c => c.ErrorMode = ChoErrorMode.IgnoreAndContinue)
    .Map(m => m.Property(p => p.Salary).FieldName("AnnualSalary"));
    
// 读取时使用配置
using (var reader = new ChoParquetReader<Employee>("employees.parquet", configuration))
{
    // ...
}

// 写入时使用配置
using (var writer = new ChoParquetWriter<Employee>("employees.parquet", configuration))
{
    // ...
}
```

### Reader 特有配置

```csharp
var readerConfig = new ChoParquetRecordConfiguration<Employee>()
    .Configure(c => c.IgnoreFieldValueMode = ChoIgnoreFieldValueMode.Empty)
    .Configure(c => c.ThrowAndStopOnMissingField = false)
    .Configure(c => c.MaxScanRows = 100); // 最多扫描100行来推断架构
```

### Writer 特有配置

```csharp
var writerConfig = new ChoParquetRecordConfiguration<Employee>()
    .Configure(c => c.UseDefaultValueForMissingFields = true)
    .Configure(c => c.NullValueHandling = ChoNullValueHandling.Empty)
    .Configure(c => c.CompressionMethod = ChoParquetCompressionMethod.Snappy); // 使用 Snappy 压缩算法
```

## 高级用法

### 自定义类型转换

```csharp
var config = new ChoParquetRecordConfiguration<Employee>()
    .Map(m => m.Property(p => p.JoinDate)
        .ValueConverter(o => o == null ? DateTime.MinValue : DateTime.Parse(o.ToString()))
    );
```

### 异步操作

```csharp
// 异步读取
using (var reader = new ChoParquetReader<Employee>("employees.parquet"))
{
    await foreach (var employee in reader.AsAsync())
    {
        // 处理每条记录
    }
}

// 异步写入
using (var writer = new ChoParquetWriter<Employee>("employees.parquet"))
{
    await writer.WriteAsync(employees);
}
```

### 处理大文件

```csharp
// 使用批处理模式处理大文件
var config = new ChoParquetRecordConfiguration<Employee>()
    .Configure(c => c.BatchSize = 1000); // 每批处理1000条记录

using (var reader = new ChoParquetReader<Employee>("large_employees.parquet", config))
{
    reader.BatchReceived += (sender, e) =>
    {
        // 每批数据处理
        Console.WriteLine($"Received batch of {e.Items.Count} records");
    };
    
    reader.Read();
}
```

## 常见问题与解决方案

### 问题：读取时字段映射不正确

**解决方案**：确保 POCO 类的属性名与 Parquet 文件中的列名匹配，或者使用 FieldName 映射：

```csharp
var config = new ChoParquetRecordConfiguration<Employee>()
    .Map(m => m.Property(p => p.Name).FieldName("EmployeeName"))
    .Map(m => m.Property(p => p.Department).FieldName("Dept"));
```

### 问题：处理大文件时内存不足

**解决方案**：使用批处理模式和适当的批处理大小：

```csharp
var config = new ChoParquetRecordConfiguration<Employee>()
    .Configure(c => c.BatchSize = 500)
    .Configure(c => c.LazyLoad = true);
```

### 问题：日期时间格式不正确

**解决方案**：使用 ValueConverter 或设置全局文化信息：

```csharp
var config = new ChoParquetRecordConfiguration<Employee>()
    .Configure(c => c.Culture = new CultureInfo("zh-CN"))
    .Map(m => m.Property(p => p.JoinDate).ValueConverter(o => DateTime.ParseExact(o.ToString(), "yyyy-MM-dd", CultureInfo.InvariantCulture)));
```

## 完整示例

以下是一个完整的示例，演示了 ChoETL.Parquet 的主要功能：

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using ChoETL;

namespace ParquetDemo
{
    public class Product
    {
        public int ProductID { get; set; }
        public string ProductName { get; set; }
        public decimal Price { get; set; }
        public int StockQuantity { get; set; }
        public DateTime LastUpdated { get; set; }
        public bool IsAvailable { get; set; }
    }

    class Program
    {
        static void Main(string[] args)
        {
            // 准备示例数据
            var products = new List<Product>
            {
                new Product { ProductID = 101, ProductName = "笔记本电脑", Price = 6999.99M, StockQuantity = 50, LastUpdated = DateTime.Now, IsAvailable = true },
                new Product { ProductID = 102, ProductName = "智能手机", Price = 4999.99M, StockQuantity = 100, LastUpdated = DateTime.Now.AddDays(-1), IsAvailable = true },
                new Product { ProductID = 103, ProductName = "蓝牙耳机", Price = 599.99M, StockQuantity = 200, LastUpdated = DateTime.Now.AddDays(-2), IsAvailable = true },
                new Product { ProductID = 104, ProductName = "平板电脑", Price = 3599.99M, StockQuantity = 0, LastUpdated = DateTime.Now.AddDays(-3), IsAvailable = false }
            };

            // 配置 Parquet 写入选项
            var writerConfig = new ChoParquetRecordConfiguration<Product>()
                .Configure(c => c.Culture = new CultureInfo("zh-CN"))
                .Configure(c => c.CompressionMethod = ChoParquetCompressionMethod.Snappy)
                .Map(m => m.Property(p => p.ProductName).FieldName("Name"))
                .Map(m => m.Property(p => p.LastUpdated).FieldName("UpdateTime"));

            // 写入 Parquet 文件
            Console.WriteLine("正在写入 Parquet 文件...");
            using (var writer = new ChoParquetWriter<Product>("products.parquet", writerConfig))
            {
                writer.Write(products);
            }
            Console.WriteLine("Parquet 文件写入完成!");

            // 配置 Parquet 读取选项
            var readerConfig = new ChoParquetRecordConfiguration<Product>()
                .Configure(c => c.Culture = new CultureInfo("zh-CN"))
                .Map(m => m.Property(p => p.ProductName).FieldName("Name"))
                .Map(m => m.Property(p => p.LastUpdated).FieldName("UpdateTime"));

            // 读取 Parquet 文件
            Console.WriteLine("\n正在读取 Parquet 文件:");
            using (var reader = new ChoParquetReader<Product>("products.parquet", readerConfig))
            {
                foreach (var product in reader)
                {
                    Console.WriteLine($"ID: {product.ProductID}, 名称: {product.ProductName}, 价格: {product.Price:C}, " +
                        $"库存: {product.StockQuantity}, 更新时间: {product.LastUpdated}, 是否可用: {product.IsAvailable}");
                }
            }

            Console.WriteLine("\n按任意键退出...");
            Console.ReadKey();
        }
    }
}
```

---

通过这份文档，您应该能够快速上手使用 ChoETL.Parquet 组件处理 Parquet 格式文件。更多信息和示例可以参考 ChoETL 的[官方 Wiki](https://github.com/Cinchoo/ChoETL/wiki) [2](https://github.com/cinchoo/choetl/wiki)。
