# GORM 中文文档

[https://gorm.book.jasperxu.com/](https://jasperxu.github.io/gorm-zh/)

Golang写的，开发人员友好的ORM库。

## 概述

* 全功能ORM（几乎）
* 关联（包含一个，包含多个，属于，多对多，多种包含）
* Callbacks（创建/保存/更新/删除/查找之前/之后）
* 预加载（急加载）
* 事务
* 复合主键
* SQL Builder
* 自动迁移
* 日志
* 可扩展，编写基于GORM回调的插件
* 每个功能都有测试
* 开发人员友好

## 安装

```
go get -u github.com/jinzhu/gorm
```

## 升级到V1.0

* [更新日志](changelog.md)

## 快速开始

```go
package main

import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/sqlite"
)

type Product struct {
  gorm.Model
  Code string
  Price uint
}

func main() {
  db, err := gorm.Open("sqlite3", "test.db")
  if err != nil {
    panic("连接数据库失败")
  }
  defer db.Close()

  // 自动迁移模式
  db.AutoMigrate(&Product{})

  // 创建
  db.Create(&Product{Code: "L1212", Price: 1000})

  // 读取
  var product Product
  db.First(&product, 1) // 查询id为1的product
  db.First(&product, "code = ?", "L1212") // 查询code为l1212的product

  // 更新 - 更新product的price为2000
  db.Model(&product).Update("Price", 2000)

  // 删除 - 删除product
  db.Delete(&product)
}
```



