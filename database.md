# 数据库

## 连接数据库 {#dbc}
要连接到数据库首先要导入驱动程序。例如
``` go
import _ "github.com/go-sql-driver/mysql"
```
为了方便记住导入路径，GORM包装了一些驱动。
```go
import _ "github.com/jinzhu/gorm/dialects/mysql"
// import _ "github.com/jinzhu/gorm/dialects/postgres"
// import _ "github.com/jinzhu/gorm/dialects/sqlite"
// import _ "github.com/jinzhu/gorm/dialects/mssql"
```

### MySQL
注：为了处理`time.Time`，您需要包括`parseTime`作为参数。 （[更多支持的参数](https://github.com/go-sql-driver/mysql#parameters)）
``` go
import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/mysql"
)

func main() {
  db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
  defer db.Close()
}
```

### PostgreSQL
``` go
import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/postgres"
)

func main() {
  db, err := gorm.Open("postgres", "host=myhost user=gorm dbname=gorm sslmode=disable password=mypassword")
  defer db.Close()
}
```

### Sqlite3
```
import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/sqlite"
)

func main() {
  db, err := gorm.Open("sqlite3", "/tmp/gorm.db")
  defer db.Close()
}
```

### 不支持的数据库
GORM正式支持上述的数据库，如果您使用的是不受支持的数据库请按照下面的连接编写对应数据库支持文件。
[https://github.com/jinzhu/gorm/blob/master/dialect.go](https://github.com/jinzhu/gorm/blob/master/dialect.go)



## 迁移 {#m}

### 自动迁移
自动迁移模式将保持更新到最新。

<span style="color:red;">警告：</span>自动迁移**仅仅**会创建表，缺少列和索引，并且不会改变现有列的类型或删除未使用的列以保护数据。
``` go
db.AutoMigrate(&User{})

db.AutoMigrate(&User{}, &Product{}, &Order{})

// 创建表时添加表后缀
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```

### 检查表是否存在
``` go
// 检查模型`User`表是否存在
db.HasTable(&User{})

// 检查表`users`是否存在
db.HasTable("users")
```

### 创建表
``` go
// 为模型`User`创建表
db.CreateTable(&User{})

// 创建表`users'时将“ENGINE = InnoDB”附加到SQL语句
db.Set("gorm:table_options", "ENGINE=InnoDB").CreateTable(&User{})
```

### 删除表
``` go
// 删除模型`User`的表
db.DropTable(&User{})

// 删除表`users`
db.DropTable("users")

// 删除模型`User`的表和表`products`
db.DropTableIfExists(&User{}, "products")
```

### 修改列
修改列的类型为给定值
```go
// 修改模型`User`的description列的数据类型为`text`
db.Model(&User{}).ModifyColumn("description", "text")
```

### 删除列
``` go
// 删除模型`User`的description列
db.Model(&User{}).DropColumn("description")
```

### 添加外键
```go
// 添加主键
// 1st param : 外键字段
// 2nd param : 外键表(字段)
// 3rd param : ONDELETE
// 4th param : ONUPDATE
db.Model(&User{}).AddForeignKey("city_id", "cities(id)", "RESTRICT", "RESTRICT")
```

### 索引
``` go
// 为`name`列添加索引`idx_user_name`
db.Model(&User{}).AddIndex("idx_user_name", "name")

// 为`name`, `age`列添加索引`idx_user_name_age`
db.Model(&User{}).AddIndex("idx_user_name_age", "name", "age")

// 添加唯一索引
db.Model(&User{}).AddUniqueIndex("idx_user_name", "name")

// 为多列添加唯一索引
db.Model(&User{}).AddUniqueIndex("idx_user_name_age", "name", "age")

// 删除索引
db.Model(&User{}).RemoveIndex("idx_user_name")
```

