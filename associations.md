# 关联

## 属于 {#bt}
```go
// `User`属于`Profile`, `ProfileID`为外键
type User struct {
  gorm.Model
  Profile   Profile
  ProfileID int
}

type Profile struct {
  gorm.Model
  Name string
}

db.Model(&user).Related(&profile)
//// SELECT * FROM profiles WHERE id = 111; // 111是user的外键ProfileID
```
指定外键
```go
type Profile struct {
    gorm.Model
    Name string
}

type User struct {
    gorm.Model
    Profile      Profile `gorm:"ForeignKey:ProfileRefer"` // 使用ProfileRefer作为外键
    ProfileRefer int
}
```
指定外键和关联外键
```go
type Profile struct {
    gorm.Model
    Refer string
    Name  string
}

type User struct {
    gorm.Model
    Profile   Profile `gorm:"ForeignKey:ProfileID;AssociationForeignKey:Refer"`
    ProfileID int
}
```
## 包含一个 {#ho}
```go
// User 包含一个 CreditCard, UserID 为外键
type User struct {
    gorm.Model
    CreditCard   CreditCard
}

type CreditCard struct {
    gorm.Model
    UserID   uint
    Number   string
}

var card CreditCard
db.Model(&user).Related(&card, "CreditCard")
//// SELECT * FROM credit_cards WHERE user_id = 123; // 123 is user's primary key
// CreditCard是user的字段名称，这意味着获得user的CreditCard关系并将其填充到变量
// 如果字段名与变量的类型名相同，如上例所示，可以省略，如：
db.Model(&user).Related(&card)
```
指定外键
```go
type Profile struct {
  gorm.Model
  Name      string
  UserRefer uint
}

type User struct {
  gorm.Model
  Profile Profile `gorm:"ForeignKey:UserRefer"`
}
```
指定外键和关联外键
```go
type Profile struct {
  gorm.Model
  Name   string
  UserID uint
}

type User struct {
  gorm.Model
  Refer   string
  Profile Profile `gorm:"ForeignKey:UserID;AssociationForeignKey:Refer"`
}
```

## 包含多个 {#hm}
```go
// User 包含多个 emails, UserID 为外键
type User struct {
    gorm.Model
    Emails   []Email
}

type Email struct {
    gorm.Model
    Email   string
    UserID  uint
}

db.Model(&user).Related(&emails)
//// SELECT * FROM emails WHERE user_id = 111; // 111 是 user 的主键
```
指定外键
```go
type Profile struct {
  gorm.Model
  Name      string
  UserRefer uint
}

type User struct {
  gorm.Model
  Profiles []Profile `gorm:"ForeignKey:UserRefer"`
}
```
指定外键和关联外键
```go
type Profile struct {
  gorm.Model
  Name   string
  UserID uint
}

type User struct {
  gorm.Model
  Refer   string
  Profiles []Profile `gorm:"ForeignKey:UserID;AssociationForeignKey:Refer"`
}
```
## 多对多 {#mtm}
```go
// User 包含并属于多个 languages, 使用 `user_languages` 表连接
type User struct {
    gorm.Model
    Languages         []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
    gorm.Model
    Name string
}

db.Model(&user).Related(&languages, "Languages")
//// SELECT * FROM "languages" INNER JOIN "user_languages" ON "user_languages"."language_id" = "languages"."id" WHERE "user_languages"."user_id" = 111
```
指定外键和关联外键
```go
type CustomizePerson struct {
  IdPerson string             `gorm:"primary_key:true"`
  Accounts []CustomizeAccount `gorm:"many2many:PersonAccount;ForeignKey:IdPerson;AssociationForeignKey:IdAccount"`
}

type CustomizeAccount struct {
  IdAccount string `gorm:"primary_key:true"`
  Name      string
}
```
译者注：这里设置好像缺失一部分
## 多种包含 {#p}
支持多种的包含一个和包含多个的关联
```go
type Cat struct {
    Id    int
    Name  string
    Toy   Toy `gorm:"polymorphic:Owner;"`
  }

  type Dog struct {
    Id   int
    Name string
    Toy  Toy `gorm:"polymorphic:Owner;"`
  }

  type Toy struct {
    Id        int
    Name      string
    OwnerId   int
    OwnerType string
  }
```
注意：多态属性和多对多显式不支持，并且会抛出错误。

## 关联模式 {#am}
关联模式包含一些帮助方法来处理关系事情很容易。
```go
// 开始关联模式
var user User
db.Model(&user).Association("Languages")
// `user`是源，它需要是一个有效的记录（包含主键）
// `Languages`是关系中源的字段名。
// 如果这些条件不匹配，将返回一个错误，检查它：
// db.Model(&user).Association("Languages").Error


// Query - 查找所有相关关联
db.Model(&user).Association("Languages").Find(&languages)


// Append - 添加新的many2many, has_many关联, 会替换掉当前 has_one, belongs_to关联
db.Model(&user).Association("Languages").Append([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Append(Language{Name: "DE"})


// Delete - 删除源和传递的参数之间的关系，不会删除这些参数
db.Model(&user).Association("Languages").Delete([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Delete(languageZH, languageEN)


// Replace - 使用新的关联替换当前关联
db.Model(&user).Association("Languages").Replace([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Replace(Language{Name: "DE"}, languageEN)


// Count - 返回当前关联的计数
db.Model(&user).Association("Languages").Count()


// Clear - 删除源和当前关联之间的关系，不会删除这些关联
db.Model(&user).Association("Languages").Clear()
```