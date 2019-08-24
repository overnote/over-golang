## 一 session介绍

#### 1.0 cookie登录的改进

改进方案：
- 生成的cookie不再存储username，而是一个随机生成的字符串，比如32位的uuid，服务端存储一个uuid与用户名的映射
- 用户再次请求时，会自动把cookie中的uuid带入给服务器
- 服务器使用uuid进行鉴权

注意：一般上述的uuid在cookie中存储的键都是sid，也就是常说的session方案。

#### 1.1 session原理 

session和cookie的目的相同，都是为了克服http协议无状态的缺陷，但完成的方法不同。session通过cookie，在客户端保存session_id，而将用户的其他会话消息保存在服务端的session对象中，与此相对的，cookie需要将所有信息都保存在客户端。因此cookie存在着一定的安全隐患，例如本地cookie中保存的用户名密码被破译，或cookie被其他网站收集。  

#### 1.2 session创建过程

session的基本原理是由服务器为每个会话维护一份信息数据，客户端和服务端依靠一个全局唯一的标识来访问这份数据，以达到交互的目的。当用户访问Web应用时，服务端程序会随需要创建session，这个过程可以概括为三个步骤：

- 生成全局唯一标识符（sessionid）；
- 开辟数据存储空间。一般会在内存中创建相应的数据结构，但这种情况下，系统一旦掉电，所有的会话数据就会丢失，如果是电子商务类网站，这将造成严重的后果。所以为了解决这类问题，你可以将会话数据写到文件里或存储在数据库中，当然这样会增加I/O开销，但是它可以实现某种程度的session持久化，也更有利于session的共享；
- 将session的全局唯一标示符发送给客户端。

以上三个步骤中，最关键的是如何发送这个session的唯一标识这一步上。考虑到HTTP协议的定义，数据无非可以放到请求行、头域或Body里，所以一般来说会有两种常用的方式：
- 1.Cookie：服务端通过设置Set-cookie头就可以将session的标识符传送到客户端，而客户端此后的每一次请求都会带上这个标识符
- 2.URL重写：在返回给用户的页面里的所有的URL后面追加session标识符，这样用户在收到响应之后，无论点击响应页面里的哪个链接或提交表单，都会自动带上session标识符，从而就实现了会话的保持。虽然这种做法比较麻烦，但是，如果客户端禁用了cookie的话，此种方案将会是首选。

#### 1.3 session创建因素

- 全局session管理器
- 保证sessionid 的全局唯一性
- 为每个客户关联一个session
- session 的存储(可以存储到内存、文件、数据库等)
- session 过期处理

## 二 实现session对象

session.go文件：
```go
package session

import (
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"sync"
	"time"
)

//session存储方式接口
type Provider interface {
	//初始化一个session，sid根据需要生成后传入
	SessionInit(sid string) (Session, error)
	//根据sid,获取session
	SessionRead(sid string) (Session, error)
	//销毁session
	SessionDestroy(sid string) error
	//回收
	SessionGC(maxLifeTime int64)
}

//Session操作接口
type Session interface {
	Set(key, value interface{}) error
	Get(key interface{}) interface{}
	Delete(ket interface{}) error
	SessionID() string
}

type Manager struct {
	cookieName  string
	lock        sync.Mutex //互斥锁
	provider    Provider   //存储session方式
	maxLifeTime int64      //有效期
}

//实例化一个session管理器
func NewSessionManager(provideName, cookieName string, maxLifeTime int64) (*Manager, error) {
	provide, ok := provides[provideName]
	if !ok {
		return nil, fmt.Errorf("session: unknown provide %q ", provideName)
	}
	return &Manager{cookieName: cookieName, provider: provide, maxLifeTime: maxLifeTime}, nil
}

//注册 由实现Provider接口的结构体调用
func Register(name string, provide Provider) {
	if provide == nil {
		panic("session: Register provide is nil")
	}
	if _, ok := provides[name]; ok {
		panic("session: Register called twice for provide " + name)
	}
	provides[name] = provide
}

var provides = make(map[string]Provider)

//生成sessionId
func (manager *Manager) sessionId() string {
	b := make([]byte, 32)
	if _, err := io.ReadFull(rand.Reader, b); err != nil {
		return ""
	}
	//加密
	return base64.URLEncoding.EncodeToString(b)
}

//判断当前请求的cookie中是否存在有效的session，存在返回，否则创建
func (manager *Manager) SessionStart(w http.ResponseWriter, r *http.Request) (session Session) {
	manager.lock.Lock() //加锁
	defer manager.lock.Unlock()
	cookie, err := r.Cookie(manager.cookieName)
	if err != nil || cookie.Value == "" {
		//创建一个
		sid := manager.sessionId()
		session, _ = manager.provider.SessionInit(sid)
		cookie := http.Cookie{
			Name:     manager.cookieName,
			Value:    url.QueryEscape(sid), //转义特殊符号@#￥%+*-等
			Path:     "/",
			HttpOnly: true,
			MaxAge:   int(manager.maxLifeTime),
			Expires:  time.Now().Add(time.Duration(manager.maxLifeTime)),
			//MaxAge和Expires都可以设置cookie持久化时的过期时长，Expires是老式的过期方法，
			// 如果可以，应该使用MaxAge设置过期时间，但有些老版本的浏览器不支持MaxAge。
			// 如果要支持所有浏览器，要么使用Expires，要么同时使用MaxAge和Expires。
		}
		http.SetCookie(w, &cookie)
	} else {
		sid, _ := url.QueryUnescape(cookie.Value) //反转义特殊符号
		session, _ = manager.provider.SessionRead(sid)
	}
	return session
}

//销毁session 同时删除cookie
func (manager *Manager) SessionDestroy(w http.ResponseWriter, r *http.Request) {
	cookie, err := r.Cookie(manager.cookieName)
	if err != nil || cookie.Value == "" {
		return
	} else {
		manager.lock.Lock()
		defer manager.lock.Unlock()
		sid, _ := url.QueryUnescape(cookie.Value)
		manager.provider.SessionDestroy(sid)
		expiration := time.Now()
		cookie := http.Cookie{
			Name:     manager.cookieName,
			Path:     "/",
			HttpOnly: true,
			Expires:  expiration,
			MaxAge:   -1}
		http.SetCookie(w, &cookie)
	}
}

func (manager *Manager) GC() {
	manager.lock.Lock()
	defer manager.lock.Unlock()
	manager.provider.SessionGC(manager.maxLifeTime)
	time.AfterFunc(time.Duration(manager.maxLifeTime), func() { manager.GC() })
}

```

memory.go文件：Provider 和Session 接口的具体实现
```go
package memory

import (
	"container/list"
	"example/example/public/session"
	"sync"
	"time"
)

var pder = &FromMemory{list: list.New()}

func init() {
	pder.sessions = make(map[string]*list.Element, 0)
	//注册  memory 调用的时候一定有一致
	session.Register("memory", pder)
}

//session实现
type SessionStore struct {
	sid              string                      //session id 唯一标示
	LastAccessedTime time.Time                   //最后访问时间
	value            map[interface{}]interface{} //session 里面存储的值
}

//设置
func (st *SessionStore) Set(key, value interface{}) error {
	st.value[key] = value
	pder.SessionUpdate(st.sid)
	return nil
}

//获取session
func (st *SessionStore) Get(key interface{}) interface{} {
	pder.SessionUpdate(st.sid)
	if v, ok := st.value[key]; ok {
		return v
	} else {
		return nil
	}
	return nil
}

//删除
func (st *SessionStore) Delete(key interface{}) error {
	delete(st.value, key)
	pder.SessionUpdate(st.sid)
	return nil
}
func (st *SessionStore) SessionID() string {
	return st.sid
}

//session来自内存 实现
type FromMemory struct {
	lock     sync.Mutex               //用来锁
	sessions map[string]*list.Element //用来存储在内存
	list     *list.List               //用来做 gc
}

func (frommemory *FromMemory) SessionInit(sid string) (session.Session, error) {
	frommemory.lock.Lock()
	defer frommemory.lock.Unlock()
	v := make(map[interface{}]interface{}, 0)
	newsess := &SessionStore{sid: sid, LastAccessedTime: time.Now(), value: v}
	element := frommemory.list.PushBack(newsess)
	frommemory.sessions[sid] = element
	return newsess, nil
}

func (frommemory *FromMemory) SessionRead(sid string) (session.Session, error) {
	if element, ok := frommemory.sessions[sid]; ok {
		return element.Value.(*SessionStore), nil
	} else {
		sess, err := frommemory.SessionInit(sid)
		return sess, err
	}
	return nil, nil
}

func (frommemory *FromMemory) SessionDestroy(sid string) error {
	if element, ok := frommemory.sessions[sid]; ok {
		delete(frommemory.sessions, sid)
		frommemory.list.Remove(element)
		return nil
	}
	return nil
}

func (frommemory *FromMemory) SessionGC(maxLifeTime int64) {
	frommemory.lock.Lock()
	defer frommemory.lock.Unlock()
	for {
		element := frommemory.list.Back()
		if element == nil {
			break
		}
		if (element.Value.(*SessionStore).LastAccessedTime.Unix() + maxLifeTime) <
			time.Now().Unix() {
			frommemory.list.Remove(element)
			delete(frommemory.sessions, element.Value.(*SessionStore).sid)
		} else {
			break
		}
	}
}
func (frommemory *FromMemory) SessionUpdate(sid string) error {
	frommemory.lock.Lock()
	defer frommemory.lock.Unlock()
	if element, ok := frommemory.sessions[sid]; ok {
		element.Value.(*SessionStore).LastAccessedTime = time.Now()
		frommemory.list.MoveToFront(element)
		return nil
	}
	return nil
}


```

## 三 使用session

```go
package main

import (
	_ "example/example/public/memory"  //这里修改成你存放menory.go相应的目录
	"example/example/public/session" //这里修改成你存放session.go相应的目录
	"fmt"
	"github.com/aliyun/alibaba-cloud-sdk-go/sdk/log"
	"net/http"
)

var globalSessions *session.Manager

func init() {
	var err error
	globalSessions, err = session.NewSessionManager("memory", "goSessionid", 3600)
	if err != nil {
		fmt.Println(err)
		return
	}
	go globalSessions.GC()
	fmt.Println("fd")
}

func sayHelloHandler(w http.ResponseWriter, r *http.Request) {

	cookie, err := r.Cookie("name")
	if err == nil {
		fmt.Println(cookie.Value)
		fmt.Println(cookie.Domain)
		fmt.Println(cookie.Expires)
	}
	//fmt.Fprintf(w, "Hello world!\n") //这个写入到w的是输出到客户端的
}
func login(w http.ResponseWriter, r *http.Request) {
	sess := globalSessions.SessionStart(w, r)
	val := sess.Get("username")
	if val != nil {
		fmt.Println(val)
	} else {
		sess.Set("username", "jerry")
		fmt.Println("set session")
	}
}
func loginOut(w http.ResponseWriter, r *http.Request) {
	//销毁
	globalSessions.SessionDestroy(w, r)
	fmt.Println("session destroy")
}

func main() {
	http.HandleFunc("/", sayHelloHandler) //	设置访问路由
	http.HandleFunc("/login", login)
	http.HandleFunc("/loginout", loginOut) //销毁
	log.Fatal(http.ListenAndServe(":8080", nil))
}

```

## 四 预防session劫持

在session技术中，客户端和服务端通过session的标识符来维护会话， 但这个标识符很容易就能被嗅探到，从而被其他人利用。它是中间人攻击的一种类型。  

制作一个count计数器：
```Go

func count(w http.ResponseWriter, r *http.Request) {
	sess := globalSessions.SessionStart(w, r)
	ct := sess.Get("countnum")
	if ct == nil {
		sess.Set("countnum", 1)
	} else {
		sess.Set("countnum", (ct.(int) + 1))
	}
	t, _ := template.ParseFiles("count.gtpl")
	w.Header().Set("Content-Type", "text/html")
	t.Execute(w, sess.Get("countnum"))
}

```
count.gtpl的代码如下所示：
```Go

Hi. Now count:{{.}}
```
访问：localhost:9090/count，不断刷新浏览器，count数字将不断增长，当数字显示为6时，查看开发者工具中的value。将这些内容在另外一个浏览器中重新模拟发送请求，我们发现另外一个浏览器同样可以得到结果。
可以看到虽然换了浏览器，但是我们却获得了sessionID，然后模拟了cookie存储的过程。这个例子是在同一台计算机上做的，不过即使换用两台来做，其结果仍然一样。此时如果交替点击两个浏览器里的链接你会发现它们其实操纵的是同一个计数器。不必惊讶，此处firefox盗用了chrome和goserver之间的维持会话的钥匙，即gosessionid，这是一种类型的“会话劫持”。在goserver看来，它从http请求中得到了一个gosessionid，由于HTTP协议的无状态性，它无法得知这个gosessionid是从chrome那里“劫持”来的，它依然会去查找对应的session，并执行相关计算。与此同时 chrome也无法得知自己保持的会话已经被“劫持”。
通过上面session劫持的简单演示可以了解到session一旦被其他人劫持，就非常危险，劫持者可以假装成被劫持者进行很多非法操作。那么如何有效的防止session劫持呢？

其中一个解决方案就是sessionID的值只允许cookie设置，而不是通过URL重置方式设置，同时设置cookie的httponly为true,这个属性是设置是否可通过客户端脚本访问这个设置的cookie，第一这个可以防止这个cookie被XSS读取从而引起session劫持，第二cookie设置不会像URL重置方式那么容易获取sessionID。

第二步就是在每个请求里面加上token，实现类似前面章节里面讲的防止form重复递交类似的功能，我们在每个请求里面加上一个隐藏的token，然后每次验证这个token，从而保证用户的请求都是唯一性。
```Go

h := md5.New()
salt:="astaxie%^7&8888"
io.WriteString(h,salt+time.Now().String())
token:=fmt.Sprintf("%x",h.Sum(nil))
if r.Form["token"]!=token{
	//提示登录
}
sess.Set("token",token)

```

还有一个解决方案就是，我们给session额外设置一个创建时间的值，一旦过了一定的时间，我们销毁这个sessionID，重新生成新的session，这样可以一定程度上防止session劫持的问题。
```Go

createtime := sess.Get("createtime")
if createtime == nil {
	sess.Set("createtime", time.Now().Unix())
} else if (createtime.(int64) + 60) < (time.Now().Unix()) {
	globalSessions.SessionDestroy(w, r)
	sess = globalSessions.SessionStart(w, r)
}
```
session启动后，我们设置了一个值，用于记录生成sessionID的时间。通过判断每次请求是否过期(这里设置了60秒)定期生成新的ID，这样使得攻击者获取有效sessionID的机会大大降低。

上面两个手段的组合可以在实践中消除session劫持的风险，一方面，	由于sessionID频繁改变，使攻击者难有机会获取有效的sessionID；另一方面，因为sessionID只能在cookie中传递，然后设置了httponly，所以基于URL攻击的可能性为零，同时被XSS获取sessionID也不可能。最后，由于我们还设置了MaxAge=0，这样就相当于session cookie不会留在浏览器的历史记录里面。


