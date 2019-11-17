## 一 数据交互的格式

常见的数据交互格式有：
- JSON：JavaScript Object Notation，轻量级的数据交换格式，如：`{"name":"lisi","address": ["北京","上海"]}`
- XML：工业开发中常用的数据交互标准格式

## 二 JSON方式

#### 2.1 JSON序列化

JSON序列化与反序列化需要使用`encoding/json`包，如下案例所示：

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

	str :=  `{"Name":"lisi","Age":50}`

	// 反序列化json为结构体
	type Person struct {
		Name string 
		Age int 
	}

	var p Person
	json.Unmarshal([]byte(str), &p)
	fmt.Println(p)							//{lisi 50}
```

#### 2.3 解析到interface

2.1和2.2的案例中，我们知道json的数据结构，可以直接进行序列化操作，如果不知道JSON具体的结构，就需要解析到interface，因为interface{}可以用来存储任意数据类型的对象。  

JSON包中采用`map[string]interface{}`和`[]interface{}`结构来存储任意的JSON对象和数组。Go类型和JSON类型的对应关系如下：
- bool 代表 JSON booleans,
- float64 代表 JSON numbers,
- string 代表 JSON strings,
- nil 代表 JSON null

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

上面是官方提供的解决方案，操作起来不是很方便，推荐使用第三方包有：
- https://github.com/bitly/go-simplejson
- https://github.com/thedevsaddam/gojsonq

## 三 XML方式

#### 3.1 解析XML

现在有如下`books.xml`示例：
```xml
<?xml version="1.0" encoding="utf-8"?>
<books version="1">
	<book>
		<bookName>离散数学</bookName>
		<bookPrice>120</bookPrice>
	</book>
	<book>
		<bookName>人月神话</bookName>
		<bookPrice>75</bookPrice>
	</book>
</books>
```

通过xml包的`Unmarshal`函数来解析：
```go
package main

import (
	"encoding/xml"
	"fmt"
	"io/ioutil"
	"os"
)

type BookStore struct {
	XMLName     xml.Name `xml:"books"`
	Version     string   `xml:"version,attr"`
	Store       []book	 `xml:"book"`
	Description string   `xml:",innerxml"`
}

type book struct {
	XMLName    	xml.Name `xml:"book"`
	BookName 	string   `xml:"bookName"`
	BookPrice   string   `xml:"bookPrice"`
}

func main() {

	file, err := os.Open("books.xml") 		
	if err != nil {
		fmt.Printf("error: %v", err)
		return
	}
	defer file.Close()
	data, err := ioutil.ReadAll(file)
	if err != nil {
		fmt.Printf("error: %v", err)
		return
	}

	v := BookStore{}
	err = xml.Unmarshal(data, &v)
	if err != nil {
		fmt.Printf("error: %v", err)
		return
	}

	fmt.Println(v)
}

```

#### 3.2 生成XML

xml包中的`Marshal`和`MarshalIndent`两个函数，可以用来生成xml。这两个函数主要的区别是第二个函数会增加前缀和缩进，函数的定义如下所示：
```Go
package main

import (
	"encoding/xml"
	"fmt"
	"os"
)

type BookStore struct {
	XMLName     xml.Name `xml:"books"`
	Version     string   `xml:"version,attr"`
	Store       []book	 `xml:"book"`
}

type book struct {
	BookName 	string   `xml:"bookName"`
	BookPrice   string   `xml:"bookPrice"`
}

func main() {

	bs := &BookStore{Version: "1"}
	bs.Store = append(bs.Store, book{"离散数学", "120"})
	bs.Store = append(bs.Store, book{"人月神话", "75"})

	output, err := xml.MarshalIndent(bs, "  ", "    ")
	if err != nil {
		fmt.Printf("error: %v\n", err)
	}

	// 生成正确xml头
	os.Stdout.Write([]byte(xml.Header))
	os.Stdout.Write(output)
}