# CRUD:读写数据
<!-- toc -->
## 创建 {#c}
### 创建记录
```go
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}

db.NewRecord(user) // => 主键为空返回`true`

db.Create(&user)

db.NewRecord(user) // => 创建`user`后返回`false`
```
### 默认值
您可以在gorm tag中定义默认值，然后插入SQL将忽略具有默认值的这些字段，并且其值为空，并且在将记录插入数据库后，gorm将从数据库加载这些字段的值。
```go
type Animal struct {
    ID   int64
    Name string `gorm:"default:'galeone'"`
    Age  int64
}

var animal = Animal{Age: 99, Name: ""}
db.Create(&animal)
// INSERT INTO animals("age") values('99');
// SELECT name from animals WHERE ID=111; // 返回主键为 111
// animal.Name => 'galeone'
```
### 在Callbacks中设置主键
如果要在BeforeCreate回调中设置主字段的值，可以使用scope.SetColumn，例如：
```go
func (user *User) BeforeCreate(scope *gorm.Scope) error {
  scope.SetColumn("ID", uuid.New())
  return nil
}
```
### 扩展创建选项
```go
// 为Instert语句添加扩展SQL选项
db.Set("gorm:insert_option", "ON CONFLICT").Create(&product)
// INSERT INTO products (name, code) VALUES ("name", "code") ON CONFLICT;
```
## 查询 {#q}
```go
// 获取第一条记录，按主键排序
db.First(&user)
//// SELECT * FROM users ORDER BY id LIMIT 1;

// 获取最后一条记录，按主键排序
db.Last(&user)
//// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// 获取所有记录
db.Find(&users)
//// SELECT * FROM users;

// 使用主键获取记录
db.First(&user, 10)
//// SELECT * FROM users WHERE id = 10;
```
### Where查询条件 (简单SQL)
```go
// 获取第一个匹配记录
db.Where("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name = 'jinzhu' limit 1;

// 获取所有匹配记录
db.Where("name = ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu';

db.Where("name <> ?", "jinzhu").Find(&users)

// IN
db.Where("name in (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)

db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
```
### Where查询条件 (Struct & Map)
注意：当使用struct查询时，GORM将只查询那些具有值的字段
```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 LIMIT 1;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// 主键的Slice
db.Where([]int64{20, 21, 22}).Find(&users)
//// SELECT * FROM users WHERE id IN (20, 21, 22);
```
### Not条件查询
```go
db.Not("name", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name <> "jinzhu" LIMIT 1;

// Not In
db.Not("name", []string{"jinzhu", "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");

// Not In slice of primary keys
db.Not([]int64{1,2,3}).First(&user)
//// SELECT * FROM users WHERE id NOT IN (1,2,3);

db.Not([]int64{}).First(&user)
//// SELECT * FROM users;

// Plain SQL
db.Not("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE NOT(name = "jinzhu");

// Struct
db.Not(User{Name: "jinzhu"}).First(&user)
//// SELECT * FROM users WHERE name <> "jinzhu";
```

### 带内联条件的查询
注意：使用主键查询时，应仔细检查所传递的值是否为有效主键，以避免SQL注入
```go
// 按主键获取
db.First(&user, 23)
//// SELECT * FROM users WHERE id = 23 LIMIT 1;

// 简单SQL
db.Find(&user, "name = ?", "jinzhu")
//// SELECT * FROM users WHERE name = "jinzhu";

db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)
//// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// Struct
db.Find(&users, User{Age: 20})
//// SELECT * FROM users WHERE age = 20;

// Map
db.Find(&users, map[string]interface{}{"age": 20})
//// SELECT * FROM users WHERE age = 20;
```

### Or条件查询
```go
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
//// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';

// Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';

// Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2"}).Find(&users)
```
### 查询链
Gorm有一个可链接的API，你可以这样使用它
```go
db.Where("name <> ?","jinzhu").Where("age >= ? and role <> ?",20,"admin").Find(&users)
//// SELECT * FROM users WHERE name <> 'jinzhu' AND age >= 20 AND role <> 'admin';

db.Where("role = ?", "admin").Or("role = ?", "super_admin").Not("name = ?", "jinzhu").Find(&users)
```
### 扩展查询选项
```go
// 为Select语句添加扩展SQL选项
db.Set("gorm:query_option", "FOR UPDATE").First(&user, 10)
//// SELECT * FROM users WHERE id = 10 FOR UPDATE;
```
### FirstOrInit
获取第一个匹配的记录，或者使用给定的条件初始化一个新的记录（仅适用于struct，map条件）
```go
// Unfound
db.FirstOrInit(&user, User{Name: "non_existing"})
//// user -> User{Name: "non_existing"}

// Found
db.Where(User{Name: "Jinzhu"}).FirstOrInit(&user)
//// user -> User{Id: 111, Name: "Jinzhu", Age: 20}
db.FirstOrInit(&user, map[string]interface{}{"name": "jinzhu"})
//// user -> User{Id: 111, Name: "Jinzhu", Age: 20}
```
### Attrs
如果未找到记录，则使用参数初始化结构
```go
// Unfound
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrInit(&user)
//// SELECT * FROM USERS WHERE name = 'non_existing';
//// user -> User{Name: "non_existing", Age: 20}

db.Where(User{Name: "non_existing"}).Attrs("age", 20).FirstOrInit(&user)
//// SELECT * FROM USERS WHERE name = 'non_existing';
//// user -> User{Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "Jinzhu"}).Attrs(User{Age: 30}).FirstOrInit(&user)
//// SELECT * FROM USERS WHERE name = jinzhu';
//// user -> User{Id: 111, Name: "Jinzhu", Age: 20}
```
### Assign
将参数分配给结果，不管它是否被找到
```go
// Unfound
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrInit(&user)
//// user -> User{Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "Jinzhu"}).Assign(User{Age: 30}).FirstOrInit(&user)
//// SELECT * FROM USERS WHERE name = jinzhu';
//// user -> User{Id: 111, Name: "Jinzhu", Age: 30}
```
### FirstOrCreate
获取第一个匹配的记录，或创建一个具有给定条件的新记录（仅适用于struct, map条件）
```go
// Unfound
db.FirstOrCreate(&user, User{Name: "non_existing"})
//// INSERT INTO "users" (name) VALUES ("non_existing");
//// user -> User{Id: 112, Name: "non_existing"}

// Found
db.Where(User{Name: "Jinzhu"}).FirstOrCreate(&user)
//// user -> User{Id: 111, Name: "Jinzhu"}
```
### Attrs
如果未找到记录，则为参数分配结构
```go
// Unfound
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'non_existing';
//// INSERT INTO "users" (name, age) VALUES ("non_existing", 20);
//// user -> User{Id: 112, Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "jinzhu"}).Attrs(User{Age: 30}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'jinzhu';
//// user -> User{Id: 111, Name: "jinzhu", Age: 20}
```
### Assign
将其分配给记录，而不管它是否被找到，并保存回数据库。
```go
// Unfound
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'non_existing';
//// INSERT INTO "users" (name, age) VALUES ("non_existing", 20);
//// user -> User{Id: 112, Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "jinzhu"}).Assign(User{Age: 30}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'jinzhu';
//// UPDATE users SET age=30 WHERE id = 111;
//// user -> User{Id: 111, Name: "jinzhu", Age: 30}
```
### Select
指定要从数据库检索的字段，默认情况下，将选择所有字段;
```go
db.Select("name, age").Find(&users)
//// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
//// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
//// SELECT COALESCE(age,'42') FROM users;
```
### Order
在从数据库检索记录时指定顺序，将重排序设置为`true`以覆盖定义的条件
```go
db.Order("age desc, name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// Multiple orders
db.Order("age desc").Order("name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// ReOrder
db.Order("age desc").Find(&users1).Order("age", true).Find(&users2)
//// SELECT * FROM users ORDER BY age desc; (users1)
//// SELECT * FROM users ORDER BY age; (users2)
```
### Limit
指定要检索的记录数
```go
db.Limit(3).Find(&users)
//// SELECT * FROM users LIMIT 3;

// Cancel limit condition with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
//// SELECT * FROM users LIMIT 10; (users1)
//// SELECT * FROM users; (users2)
```

### Offset
指定在开始返回记录之前要跳过的记录数
```go
db.Offset(3).Find(&users)
//// SELECT * FROM users OFFSET 3;

// Cancel offset condition with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
//// SELECT * FROM users OFFSET 10; (users1)
//// SELECT * FROM users; (users2)
```

### Count
获取模型的记录数
```go
db.Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Find(&users).Count(&count)
//// SELECT * from USERS WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (users)
//// SELECT count(*) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (count)

db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)
//// SELECT count(*) FROM users WHERE name = 'jinzhu'; (count)

db.Table("deleted_users").Count(&count)
//// SELECT count(*) FROM deleted_users;
```

### Group & Having
```go
rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Rows()
for rows.Next() {
    ...
}

rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Rows()
for rows.Next() {
    ...
}

type Result struct {
    Date  time.Time
    Total int64
}
db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Scan(&results)
```

### Join
指定连接条件
```go
rows, err := db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Rows()
for rows.Next() {
    ...
}

db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&results)

// 多个连接与参数
db.Joins("JOIN emails ON emails.user_id = users.id AND emails.email = ?", "jinzhu@example.org").Joins("JOIN credit_cards ON credit_cards.user_id = users.id").Where("credit_cards.number = ?", "411111111111").Find(&user)
```

### Pluck
将模型中的单个列作为地图查询，如果要查询多个列，可以使用[Scan](#Scan)
```go
var ages []int64
db.Find(&users).Pluck("age", &ages)

var names []string
db.Model(&User{}).Pluck("name", &names)

db.Table("deleted_users").Pluck("name", &names)

// 要返回多个列，做这样：
db.Select("name, age").Find(&users)
```

### Scan {#Scan}
将结果扫描到另一个结构中。
```go
type Result struct {
    Name string
    Age  int
}

var result Result
db.Table("users").Select("name, age").Where("name = ?", 3).Scan(&result)

// Raw SQL
db.Raw("SELECT name, age FROM users WHERE name = ?", 3).Scan(&result)
```

### Scopes {#Scopes}
将当前数据库连接传递到`func(*DB) *DB`，可以用于动态添加条件
```go
func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
    return db.Where("amount > ?", 1000)
}

func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
}

func PaidWithCod(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
}

func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
    return func (db *gorm.DB) *gorm.DB {
        return db.Scopes(AmountGreaterThan1000).Where("status in (?)", status)
    }
}

db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)
// 查找所有信用卡订单和金额大于1000

db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
// 查找所有COD订单和金额大于1000

db.Scopes(OrderStatus([]string{"paid", "shipped"})).Find(&orders)
// 查找所有付费，发货订单
```
### 指定表名
```go
// 使用User结构定义创建`deleted_users`表
db.Table("deleted_users").CreateTable(&User{})

var deleted_users []User
db.Table("deleted_users").Find(&deleted_users)
//// SELECT * FROM deleted_users;

db.Table("deleted_users").Where("name = ?", "jinzhu").Delete()
//// DELETE FROM deleted_users WHERE name = 'jinzhu';
```

## 预加载 {#p}
```go
db.Preload("Orders").Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4);

db.Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4) AND state NOT IN ('cancelled');

db.Where("state = ?", "active").Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
//// SELECT * FROM users WHERE state = 'active';
//// SELECT * FROM orders WHERE user_id IN (1,2) AND state NOT IN ('cancelled');

db.Preload("Orders").Preload("Profile").Preload("Role").Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4); // has many
//// SELECT * FROM profiles WHERE user_id IN (1,2,3,4); // has one
//// SELECT * FROM roles WHERE id IN (4,5,6); // belongs to
```
### 自定义预加载SQL
您可以通过传递`func(db *gorm.DB) *gorm.DB`（与[Scopes](#Scopes)的使用方法相同）来自定义预加载SQL，例如：
```go
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("orders.amount DESC")
}).Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4) order by orders.amount DESC;
```
### 嵌套预加载
```go
db.Preload("Orders.OrderItems").Find(&users)
db.Preload("Orders", "state = ?", "paid").Preload("Orders.OrderItems").Find(&users)
```

## 更新 {#u}
### 更新全部字段
`Save`将包括执行更新SQL时的所有字段，即使它没有更改
```go
db.First(&user)

user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)

//// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
```

### 更新更改字段
如果只想更新更改的字段，可以使用`Update`, `Updates`
```go
// 更新单个属性（如果更改）
db.Model(&user).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

// 使用组合条件更新单个属性
db.Model(&user).Where("active = ?", true).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;

// 使用`map`更新多个属性，只会更新这些更改的字段
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello', age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// 使用`struct`更新多个属性，只会更新这些更改的和非空白字段
db.Model(&user).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// 警告:当使用struct更新时，FORM将仅更新具有非空值的字段
// 对于下面的更新，什么都不会更新为""，0，false是其类型的空白值
db.Model(&user).Updates(User{Name: "", Age: 0, Actived: false})
```

### 更新选择的字段
如果您只想在更新时更新或忽略某些字段，可以使用`Select`, `Omit`
```go
db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

### 更新更改字段但不进行Callbacks
以上更新操作将执行模型的`BeforeUpdate`, `AfterUpdate`方法，更新其`UpdatedAt`时间戳，在更新时保存它的`Associations `，如果不想调用它们，可以使用`UpdateColumn`, `UpdateColumns`
```go
// 更新单个属性，类似于`Update`
db.Model(&user).UpdateColumn("name", "hello")
//// UPDATE users SET name='hello' WHERE id = 111;

// 更新多个属性，与“更新”类似
db.Model(&user).UpdateColumns(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18 WHERE id = 111;
```

### Batch Updates 批量更新
`Callbacks`在批量更新时不会运行
```go
db.Table("users").Where("id IN (?)", []int{10, 11}).Updates(map[string]interface{}{"name": "hello", "age": 18})
//// UPDATE users SET name='hello', age=18 WHERE id IN (10, 11);

// 使用struct更新仅适用于非零值，或使用map[string]interface{}
db.Model(User{}).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18;

// 使用`RowsAffected`获取更新记录计数
db.Model(User{}).Updates(User{Name: "hello", Age: 18}).RowsAffected
```

### 使用SQL表达式更新
```go
DB.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))
//// UPDATE "products" SET "price" = price * '2' + '100', "updated_at" = '2013-11-17 21:34:10' WHERE "id" = '2';

DB.Model(&product).Updates(map[string]interface{}{"price": gorm.Expr("price * ? + ?", 2, 100)})
//// UPDATE "products" SET "price" = price * '2' + '100', "updated_at" = '2013-11-17 21:34:10' WHERE "id" = '2';

DB.Model(&product).UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
//// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = '2';

DB.Model(&product).Where("quantity > 1").UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
//// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = '2' AND quantity > 1;
```

### 在Callbacks中更改更新值
如果要使用`BeforeUpdate`, `BeforeSave`更改回调中的更新值，可以使用`scope.SetColumn`，例如
```go
func (user *User) BeforeSave(scope *gorm.Scope) (err error) {
  if pw, err := bcrypt.GenerateFromPassword(user.Password, 0); err == nil {
    scope.SetColumn("EncryptedPassword", pw)
  }
}
```

### 额外更新选项
```go
// 为Update语句添加额外的SQL选项
db.Model(&user).Set("gorm:update_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Update("name, "hello")
//// UPDATE users SET name='hello', updated_at = '2013-11-17 21:34:10' WHERE id=111 OPTION (OPTIMIZE FOR UNKNOWN);
```

## 删除/软删除 {#d}
**警告** 删除记录时，需要确保其主要字段具有值，GORM将使用主键删除记录，如果主要字段为空，GORM将删除模型的所有记录
```go
// 删除存在的记录
db.Delete(&email)
//// DELETE from emails where id=10;

// 为Delete语句添加额外的SQL选项
db.Set("gorm:delete_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Delete(&email)
//// DELETE from emails where id=10 OPTION (OPTIMIZE FOR UNKNOWN);
```
### 批量删除
删除所有匹配记录
```go
db.Where("email LIKE ?", "%jinzhu%").Delete(Email{})
//// DELETE from emails where email LIKE "%jinhu%";

db.Delete(Email{}, "email LIKE ?", "%jinzhu%")
//// DELETE from emails where email LIKE "%jinhu%";
```
### 软删除
如果模型有`DeletedAt`字段，它将自动获得软删除功能！ 那么在调用`Delete`时不会从数据库中永久删除，而是只将字段`DeletedAt`的值设置为当前时间。
```go
db.Delete(&user)
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// 批量删除
db.Where("age = ?", 20).Delete(&User{})
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// 软删除的记录将在查询时被忽略
db.Where("age = 20").Find(&user)
//// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;

// 使用Unscoped查找软删除的记录
db.Unscoped().Where("age = 20").Find(&users)
//// SELECT * FROM users WHERE age = 20;

// 使用Unscoped永久删除记录
db.Unscoped().Delete(&order)
//// DELETE FROM orders WHERE id=10;
```

## 关联 {#a}
默认情况下，当创建/更新记录时，GORM将保存其关联，如果关联具有主键，GORM将调用Update来保存它，否则将被创建。
```go
user := User{
    Name:            "jinzhu",
    BillingAddress:  Address{Address1: "Billing Address - Address 1"},
    ShippingAddress: Address{Address1: "Shipping Address - Address 1"},
    Emails:          []Email{
                                        {Email: "jinzhu@example.com"},
                                        {Email: "jinzhu-2@example@example.com"},
                   },
    Languages:       []Language{
                     {Name: "ZH"},
                     {Name: "EN"},
                   },
}

db.Create(&user)
//// BEGIN TRANSACTION;
//// INSERT INTO "addresses" (address1) VALUES ("Billing Address - Address 1");
//// INSERT INTO "addresses" (address1) VALUES ("Shipping Address - Address 1");
//// INSERT INTO "users" (name,billing_address_id,shipping_address_id) VALUES ("jinzhu", 1, 2);
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu@example.com");
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu-2@example.com");
//// INSERT INTO "languages" ("name") VALUES ('ZH');
//// INSERT INTO user_languages ("user_id","language_id") VALUES (111, 1);
//// INSERT INTO "languages" ("name") VALUES ('EN');
//// INSERT INTO user_languages ("user_id","language_id") VALUES (111, 2);
//// COMMIT;

db.Save(&user)
```
参考[Associations](associations.md)更多详细信息

### 创建/更新时跳过保存关联
默认情况下保存记录时，GORM也会保存它的关联，你可以通过设置`gorm:save_associations`为`false`跳过它。
```go
db.Set("gorm:save_associations", false).Create(&user)

db.Set("gorm:save_associations", false).Save(&user)
```

### tag设置跳过保存关联
您可以使用tag来配置您的struct，以便在创建/更新时不会保存关联
```go
type User struct {
  gorm.Model
  Name      string
  CompanyID uint
  Company   Company `gorm:"save_associations:false"`
}

type Company struct {
  gorm.Model
  Name string
}
```