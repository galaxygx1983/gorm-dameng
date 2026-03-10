---
name: gorm-dameng
description: 达梦数据库(DM8) GORM 驱动开发指南。用于 Go 项目集成达梦数据库。TRIGGER 当用户需要：(1) 在 Go 项目中使用达梦数据库 (2) 配置 GORM 连接达梦 (3) 处理达梦特有的 SQL 语法问题 (4) 解决达梦数据库迁移问题 (5) 实现达梦的 Upsert/Insert On Conflict 功能 (6) 设置 VARCHAR 字符长度而非字节长度
---

# 达梦数据库 GORM 驱动

## 概述

`gorm-dameng` 是达梦数据库(DM8)的 GORM v2 方言包，基于达梦官方 Go 驱动二次开发，提供开箱即用的 ORM 支持。

**仓库地址**: https://github.com/godoes/gorm-dameng

**许可证**: MIT (非GPL)

**当前驱动版本**: go-20250513

## 快速开始

### 安装

```bash
go get -d github.com/godoes/gorm-dameng
```

### 基本连接

```go
import (
    "github.com/godoes/gorm-dameng"
    "gorm.io/gorm"
)

func main() {
    // 方式1：使用 BuildUrl 构建连接字符串
    options := map[string]string{
        "schema":         "SYSDBA",
        "appName":        "MyApp",
        "connectTimeout": "30000",
    }
    dsn := dameng.BuildUrl("user", "password", "127.0.0.1", 5236, options)
    db, err := gorm.Open(dameng.Open(dsn), &gorm.Config{})

    // 方式2：直接使用连接字符串
    // dm://user:password@host:port?schema=SYSDBA
    db, err := gorm.Open(dameng.Open("dm://SYSDBA:password@127.0.0.1:5236"), &gorm.Config{})
}
```

## 核心配置

### VARCHAR 字符长度模式

达梦默认 VARCHAR 长度按字节计算，可配置为按字符计算：

```go
// 字符长度模式（推荐用于中文）
db, err := gorm.Open(dameng.New(dameng.Config{
    DSN: dsn,
    VarcharSizeIsCharLength: true, // VARCHAR(100 CHAR) 而非 VARCHAR(100)
}), &gorm.Config{})

// 字节长度模式（默认）
db, err := gorm.Open(dameng.New(dameng.Config{
    DSN: dsn,
    VarcharSizeIsCharLength: false,
}), &gorm.Config{})
```

### 连接参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `schema` | 登录后的当前模式 | 用户默认模式 |
| `appName` | 应用名称 | - |
| `connectTimeout` | 连接超时(ms) | - |

## 数据类型映射

| Go 类型 | 达梦类型 |
|---------|---------|
| `bool` | `BIT` |
| `int8/uint8` | `TINYINT` |
| `int16/uint16` | `SMALLINT` |
| `int32/uint32` | `INT` |
| `int64/uint64` | `BIGINT` |
| `float32/float64` | `DOUBLE` / `DECIMAL(p,s)` |
| `string` | `VARCHAR` / `CLOB` (超长) |
| `time.Time` | `TIMESTAMP WITH TIME ZONE` |
| `[]byte` | `VARBINARY` / `BLOB` (超长) |

## 迁移功能

### 自动迁移

```go
type User struct {
    ID        uint   `gorm:"primaryKey;autoIncrement"`
    Name      string `gorm:"size:100;not null"`
    Email     string `gorm:"size:255;unique"`
    CreatedAt time.Time
}

// 迁移并设置表注释
db.Set("gorm:table_comments", "用户信息表").AutoMigrate(&User{})
```

### 列注释

```go
type Product struct {
    ID    uint   `gorm:"primaryKey" gorm:"comment:主键ID"`
    Name  string `gorm:"size:100;comment:产品名称"`
    Price int    `gorm:"comment:价格(分)"`
}
```

### 布尔类型默认值

达梦使用 `BIT` 类型表示布尔值，需注意：

```go
type Order struct {
    IsActive bool `gorm:"default:true"`  // 自动转换为 default:1
    Status   bool `gorm:"default:false"` // 自动转换为 default:0
}
```

## Upsert (MERGE INTO)

达梦不支持 `INSERT ... ON CONFLICT`，gorm-dameng 使用 `MERGE INTO` 语法实现：

```go
// GORM Clauses 方式
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    DoUpdates: clause.AssignmentColumns([]string{"name", "updated_at"}),
}).Create(&user)

// 生成的 SQL:
// MERGE INTO users USING (SELECT ...) AS "excluded"
// ON (users.email = excluded.email)
// WHEN MATCHED THEN UPDATE SET ...
// WHEN NOT MATCHED THEN INSERT ...
```

## 自增主键插入

```go
type Product struct {
    ID   uint `gorm:"primaryKey;autoIncrement"`
    Name string
}

// 手动指定 ID 时自动设置 IDENTITY_INSERT
product := Product{ID: 100, Name: "Special"}
db.Create(&product) // 自动执行 SET IDENTITY_INSERT ... ON/OFF
```

## 动态服务名（集群）

```go
// 连接达梦集群
dsn := "dm://user:password@GroupName?GroupName=(host1:port1,host2:port2,host3:port3)"
db, err := gorm.Open(dameng.Open(dsn), &gorm.Config{})
```

## 常见问题

### VARCHAR 长度超限

**问题**: `VARCHAR` 长度超过 32767 报错

**解决**: 使用 `CLOB` 类型或限制长度在 32767 以内

```go
type Article struct {
    Content string `gorm:"type:CLOB"` // 长文本使用 CLOB
}
```

### 字符长度 vs 字节长度

**问题**: 中文存储时长度不够

**解决**: 启用 `VarcharSizeIsCharLength: true`

```go
// VARCHAR(100) 按字节 = 25个中文字符
// VARCHAR(100 CHAR) 按字符 = 100个中文字符
db, _ := gorm.Open(dameng.New(dameng.Config{
    DSN: dsn,
    VarcharSizeIsCharLength: true,
}), &gorm.Config{})
```

### 模式(Schema)切换

```go
// 连接时指定模式
dsn := dameng.BuildUrl("user", "password", "host", 5236, map[string]string{
    "schema": "MY_SCHEMA",
})

// 运行时切换
db.Exec("SET SCHEMA MY_SCHEMA")
```

## 相比官方驱动的优化

| 功能 | 官方驱动 | gorm-dameng |
|------|---------|-------------|
| 安装方式 | 手动复制源码 | `go get` 直接安装 |
| GORM 支持 | 需自行实现 | 完整方言包 |
| 数据库迁移 | 不支持 | 完整 Migrator |
| Upsert | 不支持 | MERGE INTO |
| 列注释 | 不支持 | 支持 |
| 字符长度模式 | 不支持 | 可配置 |
| 动态服务名 | 不支持 | 支持 |

## 参考文档

- 达梦官方文档: https://eco.dameng.com/document/dm/zh-cn/pm/
- GO DM 接口: https://eco.dameng.com/document/dm/zh-cn/app-dev/go_dm.html
- GORM 文档: https://gorm.io/zh_CN/docs/