# gfyy-token
基于GoFrame框架的token插件，通过服务端验证方式实现token认证

## Requirements

```shell script
gf version >= v2.0.0
```

## Installation

```shell script
go get -u github.com/jason6688/gfyy-token
```

## Usage

### Import
```go
import "github.com/jason6688/gfyy-token"
```

### demo

```go

type User struct {
    UserData string      // 用户数据
    Data    interface{} // 其他需要携带的数据
}

func main() {
    /**
    注意事项:
    1、token存活时间 = 超时时间 + 缓存刷新时间
    2、处理携带token的请求时当前时间大于超时时间并小于缓存刷新时间时token将自动刷新即重置token存活时间
    3、每创建一个gfToken实例时CacheKey必须不相同
    4、GenerateToken函数参数的User.UserKey为用户唯一标识，必须且唯一
    */
    gft := gftoken.NewGfToken(
        gftoken.WithCacheKey("gfToken_"),
        gftoken.WithTimeout(60),
        gftoken.WithMaxRefresh(50),
        gftoken.WithMultiLogin(false),
        gftoken.WithExcludePaths(g.SliceStr{"/excludeDemo"}),
        gftoken.WithGRedisConfig(&gredis.Config{
            Address: "127.0.0.1:6379",
            Db:      9,
        }))
    s := g.Server()
    s.Group("/", func(group *ghttp.RouterGroup) {
        group.GET("/login", func(r *ghttp.Request) {
            userId := r.GetQuery("id").String()
            token, err := gft.GenerateToken(r.GetCtx(), gmd5.MustEncrypt(userId), User{
                UserData: userId,
                Data:    "myData",
            })
    
            if err != nil {
                g.Log().Error(r.GetCtx(), err)
            }
    
            r.Response.Write(token)
        })
    
        gft.Middleware(group)
        group.GET("/user", func(r *ghttp.Request) {
            token, err := gft.GetToken(r)
            if err != nil {
                r.Response.Write(err)
                return
            }
            data, err := gft.ParseToken(token.JwtToken)
            if err != nil {
                r.Response.Write(err)
                return
            }
            r.Response.Write(data)
        })
        group.GET("/excludeDemo", func(r *ghttp.Request) {
            r.Response.Write("Exclude path anyone can access")
        })
    })
    s.SetPort(8080)
    s.Run()
}
```