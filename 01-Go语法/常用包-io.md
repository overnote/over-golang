## 一 文件操作

#### 1.1 目录操作

文件操作的大多数函数都是在os包里面，下面列举了几个目录操作的：

- func Mkdir(name string, perm FileMode) error

	创建名称为name的目录，权限设置是perm，例如0777
	
- func MkdirAll(path string, perm FileMode) error

	根据path创建多级子目录
	
- func Remove(name string) error

	删除名称为name的目录，当目录下有文件或者其他目录时会出错

- func RemoveAll(path string) error

	根据path删除多级子目录，如果path是单个名称，那么该目录下的子目录全部删除。

实例：
```Go

package main

import (
	"fmt"
	"os"
)

func main() {
	os.Mkdir("test", 0777)
	os.MkdirAll("test/test1/test2", 0777)
	err := os.Remove("test")
	if err != nil {
		fmt.Println(err)
	}
	os.RemoveAll("test")
}

```

#### 1.2 新建文件

新建文件可以通过如下两个方法

- func Create(name string) (file *File, err Error)

	根据提供的文件名创建新的文件，返回一个文件对象，默认权限是0666的文件，返回的文件对象是可读写的。

- func NewFile(fd uintptr, name string) *File
	
	根据文件描述符创建相应的文件，返回一个文件对象


#### 1.3 打开文件

- func Open(name string) (file *File, err Error)

	该方法打开一个名称为name的文件，但是是只读方式，内部实现其实调用了OpenFile。

- func OpenFile(name string, flag int, perm uint32) (file *File, err Error)	

	打开名称为name的文件，flag是打开的方式，只读、读写等，perm是权限		

#### 1.4 写文件
写文件函数：

- func (file *File) Write(b []byte) (n int, err Error)

	写入byte类型的信息到文件

- func (file *File) WriteAt(b []byte, off int64) (n int, err Error)

	在指定位置开始写入byte类型的信息

- func (file *File) WriteString(s string) (ret int, err Error)

	写入string信息到文件
	
写文件的示例代码
```Go

package main

import (
	"fmt"
	"os"
)

func main() {
	userFile := "test.txt"
	fout, err := os.Create(userFile)		
	if err != nil {
		fmt.Println(userFile, err)
		return
	}
	defer fout.Close()
	for i := 0; i < 10; i++ {
		fout.WriteString("Just a test!\r\n")
		fout.Write([]byte("Just a test!\r\n"))
	}
}

```
带缓冲的写入：
```go
file, err := os.Openfile(path, O_WRONLY | O_CREATE, 0666)
if err != nil {
	fmt.Printf("%v", err)
	return
}
defer file.Close()
writer := bufio.NewWriter(file)
for l := 0; i < 5; i++ {
	writer.Writetring("hello\n")
}

writer.Flush()
```
#### 1.5 读文件
读文件函数：

- func (file *File) Read(b []byte) (n int, err Error)

	读取数据到b中

- func (file *File) ReadAt(b []byte, off int64) (n int, err Error)

	从off开始读取数据到b中

读文件的示例代码:
```Go

package main

import (
	"fmt"
	"os"
)

func main() {
	userFile := "test.txt"
	fl, err := os.Open(userFile)		
	if err != nil {
		fmt.Println(userFile, err)
		return
	}
	defer fl.Close()					//当程序退出时，defer，需要关闭文件，否则容易产生内存泄漏
	buf := make([]byte, 1024)
	for {
		n, _ := fl.Read(buf)
		if 0 == n {
			break
		}
		os.Stdout.Write(buf[:n])
	}
}

```
带缓冲的大文件读取：
```go
	userFile := "test.txt"
	fl, err := os.Open(userFile)		
	if err != nil {
		fmt.Println(userFile, err)
		return
	}
	defer fl.Close()

	reader := bufio.NewReader(file)
	for {
		str, err := reader.ReadString("\n")		//读到换行就结束一次
		if err != io.EOF {						//io.EOF表示问价末尾
			break
		}
		//输出内容
		fmt.Print(str)
	}

```
一次性读取小型文件到内存中，该方法内部封装了open和close：
```
file := "d:/test.txt"
content, err := ioutil.ReadFile(file)
if err != nil {
	fmt.Printf("%v",err)
}
fmt.Prinf("%v",string(content))
```