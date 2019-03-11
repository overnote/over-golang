## 一 cookie介绍

由于http是无状态的协议，无法确认用户的状态（如是否登录）。cookie一般存储了用户是否登录的凭证，每次请求都会将该凭证发送给服务器。  

注意：不同域名之间的cookie是互相独立的。

cookie实现鉴权步骤：
- 用户登录成功后，设置一个cookie：username=lisi
- 每次请求，会自动把cookie发送给服务端
- 服务端处理请求时，从cookie中取出username，就知道是哪个用户了
- 如果没过期，则鉴权通过，过期了，则重定向到登录页
  
这样做，风险很大，黑客很容易知道cookie中传递的内容，一旦知道用户名，黑客就会在伪造的请求的cookie中加入username。  

改进，使用sesion机制：
- 生成的cookie不再存储username，而是一个随机生成的字符串，比如32位的uuid，服务端存储一个uuid与用户名的映射
- 用户再次请求时，会自动把cookie中的uuid带入给服务器
- 服务器使用uuid进行鉴权

注意：一般上述的uuid在cookie中存储的键都是sid

## 二 Go中的cookie对象

Go语言中通过net/http包中的SetCookie来设置：
```Go
http.SetCookie(w ResponseWriter, cookie *Cookie)
```
Go中的cookie对象：
```Go

type Cookie struct {
	Name       string
	Value      string
	Path       string
	Domain     string
	Expires    time.Time
	RawExpires string

// MaxAge=0 means no 'Max-Age' attribute specified.
// MaxAge<0 means delete cookie now, equivalently 'Max-Age: 0'
// MaxAge>0 means Max-Age attribute present and given in seconds
	MaxAge   int
	Secure   bool
	HttpOnly bool
	Raw      string
	Unparsed []string // Raw text of unparsed attribute-value pairs
}

```
## 三 Go操作cookie
```Go
//设置cookie
expiration := time.Now()
expiration = expiration.AddDate(1, 0, 0)
cookie := http.Cookie{Name: "username", Value: "zs", Expires: expiration}
http.SetCookie(w, &cookie)

//获取cookie方式一
cookie, _ := r.Cookie("username")
fmt.Fprint(w, cookie)

//获取cookie方式二
for _, cookie := range r.Cookies() {
	fmt.Fprint(w, cookie.Name)
}

```