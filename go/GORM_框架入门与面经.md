# GORM 入门与面经

> 面向初学者的 Go ORM 笔记。GORM 用结构体映射表和记录，但 ORM 不会替你理解 SQL、索引和事务边界。

---

## 1. GORM 是什么？如何连接数据库？

GORM 是 Go 的 ORM（对象关系映射）库。它将 Go 结构体映射为表，将方法调用转换成 SQL；遇到性能、索引、锁或复杂查询问题时，仍应查看实际 SQL。

```go
import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

dsn := "user:password@tcp(127.0.0.1:3306)/app?charset=utf8mb4&parseTime=True&loc=Local"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
if err != nil {
    return err
}

sqlDB, err := db.DB()
if err != nil {
    return err
}
sqlDB.SetMaxOpenConns(20)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(time.Hour)
```

GORM 底层使用 `database/sql` 连接池。应用启动时创建一个 `*gorm.DB` 后复用即可，不要每个请求重新连接；连接池大小应结合数据库连接上限、实例数和实际并发评估。

---

## 2. 模型定义与迁移

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"size:64;not null"`
    Email     string         `gorm:"size:128;uniqueIndex;not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

if err := db.AutoMigrate(&User{}); err != nil {
    return err
}
```

- 默认表名通常由结构体名复数化得到；可通过 `TableName()` 或 `db.Table()` 明确指定。
- `ID`、`CreatedAt`、`UpdatedAt` 有常见默认约定，`gorm.DeletedAt` 启用软删除。
- `AutoMigrate` 适合原型和简单增量变更，但线上重要库应使用可审查、可回滚的版本化迁移脚本；删除列、缩小字段和大表 DDL 都需要额外评估。

---

## 3. CRUD：创建、查询、更新、删除

```go
// Create
user := User{Name: "Ada", Email: "ada@example.com"}
if err := db.Create(&user).Error; err != nil { return err }

// Read：First 未找到时返回 gorm.ErrRecordNotFound
var found User
err := db.Where("email = ?", "ada@example.com").First(&found).Error
if errors.Is(err, gorm.ErrRecordNotFound) { /* 返回 404 */ }
if err != nil && !errors.Is(err, gorm.ErrRecordNotFound) { return err }

// Update：明确 Select 字段，避免请求体覆盖不应修改的列
if err := db.Model(&found).Select("Name").Updates(User{Name: "Grace"}).Error; err != nil {
    return err
}

// Delete：模型含 DeletedAt 时是软删除
if err := db.Delete(&found).Error; err != nil { return err }
```

查询列表应显式分页和排序，避免无界 `Find`：

```go
var users []User
err := db.Where("name LIKE ?", "%Ada%").Order("id DESC").Limit(20).Offset(0).Find(&users).Error
```

所有来自外部的值都使用 `?` 占位参数，不要拼接 SQL 字符串。列名、排序字段等无法参数化的标识符，应从固定白名单选择。

---

## 4. 关联、预加载和 N+1 问题

```go
type Order struct {
    ID     uint
    UserID uint
    User   User
}

var orders []Order
err := db.Preload("User").Find(&orders).Error
```

常见关系是 belongs to、has one、has many、many to many。逐条循环查询关联对象会形成 N+1 查询：先查 100 条订单，再查 100 次用户。少量明确关联时可用 `Preload` 批量加载；需要筛选字段、聚合或大数据量时，使用 `Joins` 或手写 SQL，并检查执行计划。

---

## 5. Context、事务与错误处理

数据库调用应接收请求上下文，使超时和客户端取消能传递到驱动：

```go
err := db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&Order{UserID: userID}).Error; err != nil {
        return err // 返回 error 会回滚
    }
    if err := tx.Model(&Inventory{}).
        Where("sku = ? AND stock >= ?", sku, quantity).
        Update("stock", gorm.Expr("stock - ?", quantity)).Error; err != nil {
        return err
    }
    return nil // 返回 nil 才提交
})
```

事务只包裹必须原子化的数据库操作，避免在事务内调用慢 RPC、等待用户输入或执行长时间计算，否则会长时间占用连接和锁。并发扣库存等问题除了事务，还要依赖正确的条件更新、行锁或乐观锁策略。

---

## 6. 面试高频问题

### GORM 的 `*gorm.DB` 能并发使用吗？

通常可以。链式调用会派生会话状态，底层连接由 `database/sql` 连接池管理。不要在共享变量上保存并修改特定查询条件；每次请求从全局 `db` 开始链式构造，或用 `Session` 创建独立会话。

### `First`、`Find` 与 `Take` 有什么区别？

`First` 按主键升序取一条，`Last` 按主键降序取一条，`Take` 不指定排序取一条；它们未找到时会返回 `gorm.ErrRecordNotFound`。`Find` 用于集合查询，空结果通常不是错误。

### `Save` 和 `Updates` 怎么选？

`Save` 倾向保存全部字段，容易把零值和不该更新的字段一起写入。接收 HTTP 请求时更推荐使用 DTO，再通过 `Select`/`Updates(map[string]any{...})` 明确允许修改的列。

### 什么是软删除？如何查已删除数据？

模型带 `gorm.DeletedAt` 时，删除会更新 `deleted_at`，普通查询自动排除该行。需要查看全部记录时用 `Unscoped()`，但恢复、清理和唯一索引策略都要单独设计。

### 如何排查慢查询？

打开适度的 SQL 日志或慢查询日志，记录参数与耗时；将最终 SQL 放入 `EXPLAIN` 查看索引、扫描行数和执行计划。不要只通过“换 ORM 写法”猜测优化，核心仍是 SQL、索引、数据量和访问模式。

---

## 7. 面试速答

> GORM 是建立在 `database/sql` 之上的 ORM，负责把结构体和 CRUD 映射成 SQL，但不替代对 SQL、索引和事务的理解。项目里我会复用全局 DB、为每次调用传入 context、用参数化条件避免注入、为列表做分页，并把多步原子操作包进短事务。关联查询会警惕 N+1，必要时用 Preload、Join 或手写 SQL，并通过日志和 EXPLAIN 验证性能。

