# Gin 框架入门与面经

> 面向第一次接触 Go Web 开发的学习者。示例基于 Gin 的常用 API，重点是理解请求如何经过路由、中间件和处理函数。

---

## 1. Gin 是什么？一次请求怎样流转？

Gin 是 Go 的 HTTP Web 框架，基于标准库 `net/http`，提供路由匹配、参数绑定、中间件和 JSON 响应等常用能力。典型链路是：

```text
HTTP 请求 -> Router -> 全局/路由组中间件 -> Handler -> 响应
```

最小程序：

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default() // Logger + Recovery 两个默认中间件
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    _ = r.Run(":8080")
}
```

`gin.Default()` 适合快速开始；若要自行决定中间件，使用 `gin.New()`。生产环境应显式配置日志、恢复、超时、CORS 和鉴权，而不是只依赖默认配置。

---

## 2. 路由、参数和路由组

```go
r := gin.New()

r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")          // 路径参数：/users/42
    page := c.DefaultQuery("page", "1") // 查询参数：?page=2
    c.JSON(200, gin.H{"id": id, "page": page})
})

api := r.Group("/api/v1")
api.Use(authMiddleware())
api.POST("/users", createUser)
```

- `GET`、`POST`、`PUT`、`PATCH`、`DELETE` 对应 HTTP 方法；不要为了方便把所有接口都写成 `GET`。
- 路由组用于共享前缀和中间件，例如 `/api/v1`、管理员接口、需要登录的接口。
- 业务 Handler 应尽量薄：解析输入、调用 service、映射响应；数据库访问和复杂业务规则放到 service/repository 层。

---

## 3. 请求绑定与校验

绑定将 JSON、表单或 URL 参数转换为结构体。`ShouldBindJSON` 会返回错误，适合自己统一返回格式；不要忽略绑定错误。

```go
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=2,max=32"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"gte=0,lte=150"`
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusCreated, gin.H{"name": req.Name})
}
```

常用方法：

- `ShouldBindJSON`：JSON 请求体。
- `ShouldBindQuery`：查询参数。
- `ShouldBindUri`：路径参数。
- `ShouldBind`：根据 `Content-Type` 自动选择；接口行为明确时，优先显式方法。

校验只解决输入格式，不能替代权限校验、唯一性检查、库存检查等业务校验。

---

## 4. 中间件、错误处理与 Context

中间件是签名为 `func(*gin.Context)` 的函数。`c.Next()` 会执行后续中间件和 Handler；`c.Abort()` 会阻止后续链路继续执行。

```go
func authMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        c.Set("userID", "u-123") // 仅存本次请求范围的数据
        c.Next()
    }
}
```

`*gin.Context` 是 Gin 的请求上下文，提供参数、响应和中间件控制；需要调用数据库、RPC 等标准库接口时，传递 `c.Request.Context()`。请求超时或客户端断开后，下游应监听该标准 `context.Context` 的取消信号。

错误处理建议统一：输入错误返回 400，未认证 401，无权限 403，不存在 404，冲突 409，未知服务端错误 500；对外不要返回 SQL、堆栈或内部实现细节。`gin.Recovery()` 只能防止 panic 让进程崩溃，不能替代正常的 `error` 处理和告警。

---

## 5. 新手常见问题

### Gin 和 `net/http` 的关系？

Gin 建立在 `net/http` 之上，最终仍是一个 `http.Handler`。Gin 减少了路由、绑定和中间件的样板代码，但 Go 标准库的 `Request`、`ResponseWriter`、`context` 等概念仍然适用。

### `gin.Default()` 和 `gin.New()` 有什么区别？

`Default()` 等价于创建引擎后挂上 Logger 和 Recovery；`New()` 只有空引擎，适合需要精确控制日志和中间件的项目。

### 中间件的执行顺序？

按照注册顺序进入，`c.Next()` 后再按相反方向返回。因此可在 `Next()` 前记录开始时间，在后面记录状态码和耗时。路由组中间件只影响该组及其子组。

### Handler 里能不能启动 goroutine？

可以，但不能在请求结束后继续直接使用 `*gin.Context`，它会被复用。需要的字段先复制出来，并传递 `c.Request.Context()` 或项目定义的后台任务 context；后台任务还要有明确超时和退出条件。

### 如何测试 Gin 接口？

用 `httptest.NewRecorder()` 构造响应记录器，用 `httptest.NewRequest()` 构造请求，再调用 `router.ServeHTTP(recorder, request)`；断言状态码、响应 JSON 和副作用即可，无需真实监听端口。

---

## 6. 面试速答

> Gin 是基于 `net/http` 的高性能 Web 框架。请求先匹配路由，再按顺序通过中间件，最后进入 Handler。Handler 负责绑定和校验输入、调用业务层、返回响应；鉴权、日志、Recover、链路追踪等横切逻辑放中间件。数据库和 RPC 调用使用 `c.Request.Context()`，使请求取消和超时能够向下游传播。
