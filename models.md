# 模型

## 模型定义 {#md}
``` go
type User struct {
    gorm.Model
    Birthday     time.Time
    Age          int
    Name         string  `gorm:"size:255"`       // string默认长度为255, 使用这种tag重设。
    Num          int     `gorm:"AUTO_INCREMENT"` // 自增

    CreditCard        CreditCard      // One-To-One (拥有一个 - CreditCard表的UserID作外键)
    Emails            []Email         // One-To-Many (拥有多个 - Email表的UserID作外键)

    BillingAddress    Address         // One-To-One (属于 - 本表的BillingAddressID作外键)
    BillingAddressID  sql.NullInt64

    ShippingAddress   Address         // One-To-One (属于 - 本表的ShippingAddressID作外键)
    ShippingAddressID int

    IgnoreMe          int `gorm:"-"`   // 忽略这个字段
    Languages         []Language `gorm:"many2many:user_languages;"` // Many-To-Many , 'user_languages'是连接表
}

type Email struct {
    ID      int
    UserID  int     `gorm:"index"` // 外键 (属于), tag `index`是为该列创建索引
    Email   string  `gorm:"type:varchar(100);unique_index"` // `type`设置sql类型, `unique_index` 为该列设置唯一索引
    Subscribed bool
}

type Address struct {
    ID       int
    Address1 string         `gorm:"not null;unique"` // 设置字段为非空并唯一
    Address2 string         `gorm:"type:varchar(100);unique"`
    Post     sql.NullString `gorm:"not null"`
}

type Language struct {
    ID   int
    Name string `gorm:"index:idx_name_code"` // 创建索引并命名，如果找到其他相同名称的索引则创建组合索引
    Code string `gorm:"index:idx_name_code"` // `unique_index` also works
}

type CreditCard struct {
    gorm.Model
    UserID  uint
    Number  string
}
```
## 约定 {#c}
### `gorm.Model` 结构体
基本模型定义`gorm.Model`，包括字段`ID`，`CreatedAt`，`UpdatedAt`，`DeletedAt`，你可以将它嵌入你的模型，或者只写你想要的字段
``` go
// 基本模型的定义
type Model struct {
  ID        uint `gorm:"primary_key"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt *time.Time
}

// 添加字段 `ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt`
type User struct {
  gorm.Model
  Name string
}

// 只需要字段 `ID`, `CreatedAt`
type User struct {
  ID        uint
  CreatedAt time.Time
  Name      string
}
```
### 表名是结构体名称的复数形式
```go
type User struct {} // 默认表名是`users`

// 设置User的表名为`profiles`
func (User) TableName() string {
  return "profiles"
}

func (u User) TableName() string {
    if u.Role == "admin" {
        return "admin_users"
    } else {
        return "users"
    }
}

// 全局禁用表名复数
db.SingularTable(true) // 如果设置为true,`User`的默认表名为`user`,使用`TableName`设置的表名不受影响

```
### 更改默认表名
您可以通过定义`DefaultTableNameHandler`对默认表名应用任何规则。
```go
gorm.DefaultTableNameHandler = func (db *gorm.DB, defaultTableName string) string  {
    return "prefix_" + defaultTableName;
}
```

### 列名是字段名的蛇形小写
```go
type User struct {
  ID uint             // 列名为 `id`
  Name string         // 列名为 `name`
  Birthday time.Time  // 列名为 `birthday`
  CreatedAt time.Time // 列名为 `created_at`
}

// 重设列名
type Animal struct {
    AnimalId    int64     `gorm:"column:beast_id"`         // 设置列名为`beast_id`
    Birthday    time.Time `gorm:"column:day_of_the_beast"` // 设置列名为`day_of_the_beast`
    Age         int64     `gorm:"column:age_of_the_beast"` // 设置列名为`age_of_the_beast`
}
```

### 字段`ID`为主键
```go
type User struct {
  ID   uint  // 字段`ID`为默认主键
  Name string
}

// 使用tag`primary_key`用来设置主键
type Animal struct {
  AnimalId int64 `gorm:"primary_key"` // 设置AnimalId为主键
  Name     string
  Age      int64
}
```

### 字段`CreatedAt`用于存储记录的创建时间
创建具有`CreatedAt`字段的记录将被设置为当前时间
```go
db.Create(&user) // 将会设置`CreatedAt`为当前时间

// 要更改它的值, 你需要使用`Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

### 字段`UpdatedAt`用于存储记录的修改时间
保存具有`UpdatedAt`字段的记录将被设置为当前时间
```go
db.Save(&user) // 将会设置`UpdatedAt`为当前时间
db.Model(&user).Update("name", "jinzhu") // 将会设置`UpdatedAt`为当前时间
```

### 字段`DeletedAt`用于存储记录的删除时间，如果字段存在
删除具有`DeletedAt`字段的记录，它不会冲数据库中删除，但只将字段`DeletedAt`设置为当前时间，并在查询时无法找到记录，请参阅[软删除](crud.md#sd)
