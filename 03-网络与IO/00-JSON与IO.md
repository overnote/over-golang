## 一 JSON数据格式
JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，如下所示：
```js
{
    "name":"lisi",
    "age":"30",
    "address": ["北京","上海"]
}
```
JSON易于机器解析和生成，程序在网络传输时会先将数据（结构体、map）等序列化成json字符串，接收方拿到json字符串时，再反序列化承恢复成原来的数据结构。
## 二 JSON序列化与反序列化
#### 2.1 JSON序列化
```go
package main
import (
	"fmt"
	"encoding/json"
)

type Person struct {
	Name string
	Age int
}

func main() {

	//序列化结构体
	p := Person {
		Name: "lisi",
		Age: 50,
	}
	data, _ := json.Marshal(&p)
	fmt.Printf(string(data));	 //{"Name":"lisi","Age":50}
}
```
同理，我们也可以使用上述方法对基本数据类型、切片、map等数据进行序列化。 
对于结构体的序列化，如果我们希望序列化后的 key 的名字，又我们自己重新制定，那么可以给 struct
指定一个 tag 标签：
```go
type Person struct {
	Name string `json:"my_name"`
	Age int `json:"my_age"`
}
//输出：{"my_name":"lisi","my_age":50}
```
#### 2.2 JSON反序列化
```go
package main
import (
	"fmt"
	"encoding/json"
)

type Person struct {
	Name string 
	Age int 
}

func main() {

	//反序列化json为结构体
	jsonStr := `{"Name":"lisi","Age":50}`	//如果是通过程序直接获取的json字符串，则无需转义

	var p Person
	json.Unmarshal([]byte(jsonStr), &p)
	fmt.Println(p)			//{lisi 50}
}
```
