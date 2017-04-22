# 开发
## 架构 {#a}
Gorm使用可链接的API，`*gorm.DB`是链的桥梁，对于每个链API，它将创建一个新的关系。
```go
db, err := gorm.Open("postgres", "user=gorm dbname=gorm sslmode=disable")

// 创建新关系
db = db.Where("name = ?", "jinzhu")

// 过滤更多
if SomeCondition {
    db = db.Where("age = ?", 20)
} else {
    db = db.Where("age = ?", 30)
}
if YetAnotherCondition {
    db = db.Where("active = ?", 1)
}
```
当我们开始执行任何操作时，GORM将基于当前的`*gorm.DB`创建一个新的`*gorm.Scope`实例
```go
// 执行查询操作
db.First(&user)
```
并且基于当前操作的类型，它将调用注册的`creating`, `updating`, `querying`, `deleting`或`row_querying`回调来运行操作。

对于上面的例子，将调用`querying`，参考[查询回调](callbacks.md#querying-an-object)

## 写插件 {#w}
GORM本身由`Callbacks`提供支持，因此您可以根据需要完全自定义GORM
### 注册新的callback
```go
func updateCreated(scope *Scope) {
    if scope.HasColumn("Created") {
        scope.SetColumn("Created", NowFunc())
    }
}

db.Callback().Create().Register("update_created_at", updateCreated)
// 注册Create进程的回调
```
### 删除现有的callback
```go
db.Callback().Create().Remove("gorm:create")
// 从Create回调中删除`gorm:create`回调
```

### 替换现有的callback
```go
db.Callback().Create().Replace("gorm:create", newCreateFunction)
// 使用新函数`newCreateFunction`替换回调`gorm:create`用于创建过程
```

### 注册callback顺序
```go
db.Callback().Create().Before("gorm:create").Register("update_created_at", updateCreated)
db.Callback().Create().After("gorm:create").Register("update_created_at", updateCreated)
db.Callback().Query().After("gorm:query").Register("my_plugin:after_query", afterQuery)
db.Callback().Delete().After("gorm:delete").Register("my_plugin:after_delete", afterDelete)
db.Callback().Update().Before("gorm:update").Register("my_plugin:before_update", beforeUpdate)
db.Callback().Create().Before("gorm:create").After("gorm:before_create").Register("my_plugin:before_create", beforeCreate)
```

### 预定义回调
GORM定义了回调以执行其CRUD操作，在开始编写插件之前检查它们。
* [Create callbacks](https://github.com/jinzhu/gorm/blob/master/callback_create.go)
* [Update callbacks](https://github.com/jinzhu/gorm/blob/master/callback_update.go)
* [Query callbacks](https://github.com/jinzhu/gorm/blob/master/callback_query.go)
* [Delete callbacks](https://github.com/jinzhu/gorm/blob/master/callback_delete.go)
* Row Query callbacks
Row Query callbacks将在运行`Row`或`Rows`时被调用，默认情况下没有注册的回调，你可以注册一个新的回调：

```go
func updateTableName(scope *gorm.Scope) {
  scope.Search.Table(scope.TableName() + "_draft") // append `_draft` to table name
}

db.Callback().RowQuery().Register("publish:update_table_name", updateTableName)
```

