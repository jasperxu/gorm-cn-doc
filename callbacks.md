# Callbacks

您可以将回调方法定义为模型结构的指针，在创建，更新，查询，删除时将被调用，如果任何回调返回错误，gorm将停止未来操作并回滚所有更改。

## 创建对象
创建过程中可用的回调
```go
// begin transaction 开始事物
BeforeSave
BeforeCreate
// save before associations 保存前关联
// update timestamp `CreatedAt`, `UpdatedAt` 更新`CreatedAt`, `UpdatedAt`时间戳
// save self 保存自己
// reload fields that have default value and its value is blank 重新加载具有默认值且其值为空的字段
// save after associations 保存后关联
AfterCreate
AfterSave
// commit or rollback transaction 提交或回滚事务
```

## 更新对象
更新过程中可用的回调
```go
// begin transaction 开始事物
BeforeSave
BeforeUpdate
// save before associations 保存前关联
// update timestamp `UpdatedAt` 更新`UpdatedAt`时间戳
// save self 保存自己
// save after associations 保存后关联
AfterUpdate
AfterSave
// commit or rollback transaction 提交或回滚事务
```
## 删除对象
删除过程中可用的回调
```go
// begin transaction 开始事物
BeforeDelete
// delete self 删除自己
AfterDelete
// commit or rollback transaction 提交或回滚事务
```
## 查询对象 {#querying-an-object}
查询过程中可用的回调
```go
// load data from database 从数据库加载数据
// Preloading (edger loading) 预加载（加载）
AfterFind
```

## 回调示例
```go
func (u *User) BeforeUpdate() (err error) {
    if u.readonly() {
        err = errors.New("read only user")
    }
    return
}

// 如果用户ID大于1000，则回滚插入
func (u *User) AfterCreate() (err error) {
    if (u.Id > 1000) {
        err = errors.New("user id is already greater than 1000")
    }
    return
}
```
gorm中的保存/删除操作正在事务中运行，因此在该事务中所做的更改不可见，除非提交。 如果要在回调中使用这些更改，则需要在同一事务中运行SQL。 所以你需要传递当前事务到回调，像这样：
```go
func (u *User) AfterCreate(tx *gorm.DB) (err error) {
    tx.Model(u).Update("role", "admin")
    return
}
```
```go
func (u *User) AfterCreate(scope *gorm.Scope) (err error) {
  scope.DB().Model(u).Update("role", "admin")
    return
}
```