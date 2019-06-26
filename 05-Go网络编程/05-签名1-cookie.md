## 一 cookie使用

由于http是无状态的协议，无法确认用户的状态（如是否登录）。cookie一般存储了用户是否登录的凭证，每次请求都会将该凭证发送给服务器。  

注意：不同域名之间的cookie是互相独立的。  

cookie实现鉴权步骤：
- 用户登录成功后，设置一个cookie：username=lisi
- 每次请求，会自动把cookie发送给服务端
- 服务端处理请求时，从cookie中取出username，就知道是哪个用户了
- 如果没过期，则鉴权通过，过期了，则重定向到登录页

Go中使用Cookie：
```go
expiration := time.Now()
expiration = expiration.AddDate(1, 0, 0)
cookie := http.Cookie{Name: "username", Value: "zs", Expires: expiration}
http.SetCookie(w, &cookie)
```

Go获取Cookie的方式：
```go

//获取cookie方式一
cookie, _ := r.Cookie("username")
fmt.Fprint(w, cookie)

//获取cookie方式二
for _, cookie := range r.Cookies() {
	fmt.Fprint(w, cookie.Name)
}
```

这样做，风险很大，黑客很容易知道cookie中传递的内容，一旦知道用户名，黑客就会在伪造的请求的cookie中加入username。  

这时候就需要session登场了，且看下一章。  