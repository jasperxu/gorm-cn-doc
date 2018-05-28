# 高级用法
## 错误处理 {#eh}
执行任何操作后，如果发生任何错误，GORM将其设置为`*DB`的`Error`字段
```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
    // 错误处理...
}

// 如果有多个错误发生，用`GetErrors`获取所有的错误，它返回`[]error`
db.First(&user).Limit(10).Find(&users).GetErrors()

// 检查是否返回RecordNotFound错误
db.Where("name = ?", "hello world").First(&user).RecordNotFound()

if db.Model(&user).Related(&credit_card).RecordNotFound() {
    // 没有信用卡被发现处理...
}
```
## 事务 {#t}
要在事务中执行一组操作，一般流程如下。
```go
// 开始事务
tx := db.Begin()

// 在事务中做一些数据库操作（从这一点使用'tx'，而不是'db'）
tx.Create(...)

// ...

// 发生错误时回滚事务
tx.Rollback()

// 或提交事务
tx.Commit()
```
### 一个具体的例子
```go
func CreateAnimals(db *gorm.DB) err {
  tx := db.Begin()
  // 注意，一旦你在一个事务中，使用tx作为数据库句柄

  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
     tx.Rollback()
     return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
     tx.Rollback()
     return err
  }

  tx.Commit()
  return nil
}
```
## SQL构建 {#sb}
### 执行原生SQL
```go
db.Exec("DROP TABLE users;")
db.Exec("UPDATE orders SET shipped_at=? WHERE id IN (?)", time.Now, []int64{11,22,33})

// Scan
type Result struct {
    Name string
    Age  int
}

var result Result
db.Raw("SELECT name, age FROM users WHERE name = ?", 3).Scan(&result)
```
### sql.Row & sql.Rows
获取查询结果为`*sql.Row`或`*sql.Rows`
```go
row := db.Table("users").Where("name = ?", "jinzhu").Select("name, age").Row() // (*sql.Row)
row.Scan(&name, &age)

rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    ...
    rows.Scan(&name, &age, &email)
    ...
}

// Raw SQL
rows, err := db.Raw("select name, age, email from users where name = ?", "jinzhu").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    ...
    rows.Scan(&name, &age, &email)
    ...
}
```
### 迭代中使用sql.Rows的Scan
```go
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()

for rows.Next() {
  var user User
  db.ScanRows(rows, &user)
  // do something
}
```
## 通用数据库接口sql.DB {#g}
从`*gorm.DB`连接获取通用数据库接口[*sql.DB](http://golang.org/pkg/database/sql/#DB)
```go
// 获取通用数据库对象`*sql.DB`以使用其函数
db.DB()

// Ping
db.DB().Ping()
```
### 连接池
```go
db.DB().SetMaxIdleConns(10)
db.DB().SetMaxOpenConns(100)
```

## 复合主键 {#cpk}
将多个字段设置为主键以启用复合主键
```go
type Product struct {
    ID           string `gorm:"primary_key"`
    LanguageCode string `gorm:"primary_key"`
}
```
## 日志 {#l}
Gorm有内置的日志记录器支持，默认情况下，它会打印发生的错误
```go
// 启用Logger，显示详细日志
db.LogMode(true)

// 禁用日志记录器，不显示任何日志
db.LogMode(false)

// 调试单个操作，显示此操作的详细日志
db.Debug().Where("name = ?", "jinzhu").First(&User{})
```
### 自定义日志
参考GORM的默认记录器如何自定义它[https://github.com/jinzhu/gorm/blob/master/logger.go](https://github.com/jinzhu/gorm/blob/master/logger.go)
```go
db.SetLogger(gorm.Logger{revel.TRACE})
db.SetLogger(log.New(os.Stdout, "\r\n", 0))
```
