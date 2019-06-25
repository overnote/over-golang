## 一 JSON数据格式

JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，严格的json格式如下所示：
```js
	{
		"name":"lisi",
		"age":"30",
		"address": ["北京","上海"]
	}
```

JSON易于机器解析和生成，在web开发中，数据是这样传递的：
- 客户端：将数据（结构体、map）等序列化成json字符串，发送给服务端
- 服务端：拿到json字符串时，再反序列化承恢复成原来的数据结构，最后进行业务处理，处理完的结构同样需要序列化成json字符串传递给客户端

## 二 JSON序列化与反序列化

#### 2.1 JSON序列化

JSON序列化需要使用`encoding/json`包，如下案例所示：

```go
	type Person struct {
		Name string
		Age int
	}

	p := Person {
		Name: "lisi",
		Age: 50,
	}

	data, _ := json.Marshal(&p)
	fmt.Printf(string(data));	 	//{"Name":"lisi","Age":50}
```

同理，我们也可以使用上述方法对基本数据类型、切片、map等数据进行序列化。   

在结构体序列化时，如果希望序列化后的`key`的名字可以自定义，可以给该结构体指定一个`tag`标签：
```go
	type Person struct {
		Name string `json:"my_name"`
		Age int `json:"my_age"`
	}
	//序列化的结果：{"my_name":"lisi","my_age":50}
```

在定义`struct tag`的时候需要注意的几点是:
- 字段的tag是`"-"`，那么这个字段不会输出到JSON
- tag中如果带有`"omitempty"`选项，那么如果该字段值为空，就不会输出到JSON串中
- 如果字段类型是bool, string, int, int64等，而tag中带有`",string"`选项，那么这个字段在输出到JSON的时候会把该字段对应的值转换成JSON字符串
- JSON对象只支持string作为key，所以要编码一个map，那么必须是map[string]T这种类型(T是Go语言中任意的类型)
- Channel, complex和function是不能被编码成JSON的
- 嵌套的数据是不能编码的，不然会让JSON编码进入死循环
- 指针在编码的时候会输出指针指向的内容，而空指针会输出null

#### 2.2 JSON反序列化
```go
	type Person struct {
		Name string 
		Age int 
	}

	//反序列化json为结构体
	jsonStr := `{"Name":"lisi","Age":50}`	//如果是通过程序直接获取的json字符串，则无需转义

	var p Person
	json.Unmarshal([]byte(jsonStr), &p)
	fmt.Println(p)							//{lisi 50}
```

## 三 解析到interface

在第二章，我们清楚的知道JSON的数据结构，可以直接进行序列化操作，如果不知道JSON具体的结构，就需要解析到interface，因为interface{}可以用来存储任意数据类型的对象。  

JSON包中采用`map[string]interface{}`和`[]interface{}`结构来存储任意的JSON对象和数组。Go类型和JSON类型的对应关系如下：
- bool 代表 JSON booleans,
- float64 代表 JSON numbers,
- string 代表 JSON strings,
- nil 代表 JSON null.

现在我们假设有如下的JSON数据
```go
	jsonStr := `{"Name":"Lisi","Age":6,"Parents":["Lisan","WW"]}`
	jsonBytes := []byte(jsonStr)

	var i interface{}
	json.Unmarshal(jsonBytes, &i)
	fmt.Println(i)		// map[Age:6 Name:Lisi Parents:[Lisan WW]]
```

上述变量`i`存储了存储了一个map类型，key是strig，值存储在空接口内，
如果在我们不知道他的结构的情况下，我们把他解析到interface{}里面，其真实结构如下：

```Go
i = map[string]interface{}{
	"Name": "Lisi",
	"Age":  6,
	"Parents": []interface{}{
		"Lisan",
		"WW",
	},
}
```

由于是空接口类型，无法直接访问，需要使用断言方式：
```go
	m := i.(map[string]interface{})
	for k, v := range m {
		switch r := v.(type) {
		case string:
			fmt.Println(k, " is string ", r)
		case int:
			fmt.Println(k, " is int ", r)
		case []interface{}:
			fmt.Println(k, " is array ", )
			for i, u := range r {
				fmt.Println(i, u)
			}
		default:
			fmt.Println(k, " cannot be recognized")
		}
	}
```	

上面是官方提供的解决方案，操作起来不是很方便，目前bitly公司开源了一个叫做`simplejson`的包，在处理未知结构体的JSON时相当方便。  
地址：https://github.com/bitly/go-simplejson  
示例：
```go
	jsonStr := `{
		"test": {"Name":"Lisi","Age":6,"Parents":["Lisan","WW"]}
	}`
	js, err := simplejson.NewJson([]byte(jsonStr))

	if err != nil {
		fmt.Println(err)
		return
	}

	arr, _ := js.Get("test").Get("Parents").Array()
	str := js.Get("test").Get("Name").MustString()

	fmt.Println("arr: ", arr)		// arr:  [Lisan WW]
	fmt.Println("str: ", str)		// str:  Lisi
```