---
title: 对比 Golang，Nodejs
description: 唯有套路得人心
date: 2021-06-01
slug: compare-go-and-node
image: img/2021/06/Aldeyjarfoss.jpg
categories: [Conclusion]
tags: [Conclusion, Go, Node]
---

5 月份的时候，整了几个小项目出来，其中包括用 Go 和 Node 写的两个后台。现在，回过头看看，发现后台代码都很套路，不外乎配置路由、使用中间件，所以这里做一点总结。

## 简单说明

Go 采用了 Gin 框架，Node 采用了 Express 框架。

地址分别为[iris-backend](https://github.com/AzusaChino/iris-backend) [iris-node](https://github.com/AzusaChino/iris-node)

## 对比 main 入口文件

### Gin

```go
func main() {
    // 定义app
    app := gin.Default()
    app.Static("/assets", "./assets")

    // 配置中间件
    app.Use(cors.Default())
    app.Use(middleware.Logger())

    // 配置路由
    router.ApplyRouter(app)

    // 启动
    app.Run(":7878")
}
```

### Express

```ts
// 定义app
const app = express();
const port = process.env.APP_PORT || 5000;

// 配置中间件
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(logHandler);
app.use(cors());
app.use(errorHandler);

// 配置路由
app.use("/api", AppRouter);

// 启动
app.listen(port, () => console.log(`> Listening on port ${port}`));
```

## 对比中间件

简单的写了一个 logger，仅供参考

### Go-Logger

```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()
        reqUrl := c.Request.URL
        c.Next()
        latency := time.Since(t)
        status := c.Writer.Status()
        log.Printf("%s请求完成, 耗时 %d, 状态 %d\n", reqUrl, latency, status)
    }
}
```

### Node-Logger

```ts
const logHandler = (req: Request, res: Response, next: NextFunction) => {
  let time = Date.now();
  let url = req.url;
  next();
  let latency = Date.now() - time;
  let status = res.status;
  console.log(`${url} 请求完成，耗时 ${latency}，状态 ${status}`);
};
```

## 对比路由

### Go-Router

```go
func ApplyRouter(app *gin.Engine) {
    apiV1 := app.Group("/api/v1")
    {
        apiV1.POST("/login", Login)
    }
}

func Login(c *gin.Context) {
    json := model.User{}

    // 通过 `json:""`，可以使用BindJSON获取参数
    c.BindJSON(&json)
    // 若通过`form:""`, 可以使用表单以及Bind获取参数
    db, cleanFunc, err := utils.GetDb()
    defer cleanFunc()
    if err != nil {
        log.Fatal(err)
    }
    var user model.User
    result := db.Where("username = ? AND password = ?", json.Username, json.Password).Find(&user)
    if result.Error != nil {
        c.JSON(http.StatusOK, common.Error(1001, "登陆失败"))
        return
    }
    c.JSON(http.StatusOK, common.Ok(user))
}

```

### Node-Router

```ts
const LoginRouter = Router();

LoginRouter.post("/login", (req: Request, res: Response) => {
  const loginParam: LoginParam = req.body;
  login(loginParam)
    .then((r) => {
      if (r) {
        res.status(200).json(ok({ data: "token", message: "登陆成功" }));
      } else {
        res.status(500).json(fail({ message: "用户不存在" }));
      }
    })
    .catch(() => {
      res.status(500).json(fail({ message: "登陆失败" }));
    });
});
```

## 简单总结

虽然套路很简单，但这都是因为框架很优秀啊。然后，突然想起来，Spring Webflux 也是类似的套路，所以吃熟这一套可以做很多事。

## 参考链接

- [Gin 文档](https://gin-gonic.com/docs/)
- [Express 官网](https://expressjs.com/)
