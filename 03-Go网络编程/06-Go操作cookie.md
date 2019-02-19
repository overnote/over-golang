## 一 cookie介绍
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