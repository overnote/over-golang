## 一 Go操作MySQL
Go中支持MySQL的驱动目前比较多，有如下几种，有些是支持database/sql标准，而有些是采用了自己的实现接口。  

推荐使用：
- https://github.com/go-sql-driver/mysql （Go编写，维护频繁，支持Gosql接口，底层支持keepalive）
- https://github.com/jinzhu/gorm
- https://github.com/jmoiron/sqlx（推荐）

代码示例：

```Go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	
	db, err := sql.Open("mysql", "root:123456@/mydata?charset=utf8")
	checkErr(err)

	//插入数据
	sql1 := "INSERT INTO user SET name=?,age=?"
	stmt, err := db.Prepare(sql1)
	checkErr(err)
	res, err := stmt.Exec("zs", "30",)
	checkErr(err)

	id, err := res.LastInsertId()
	checkErr(err)
	fmt.Println("插入id=", id)


	//更新数据
	sql2 := "UPDATE user SET name=? WHERE id=?";
	stmt, err = db.Prepare(sql2)
	checkErr(err)
	res, err = stmt.Exec("lisi", id)
	checkErr(err)
	affect, err := res.RowsAffected()
	checkErr(err)
	fmt.Println("更新行数=", affect)

	//查询数据
	rows, err := db.Query("SELECT * FROM user")
	checkErr(err)
	for rows.Next() {
		var id int
		var name string
		var age int
		err = rows.Scan(&id, &name, &age)
		checkErr(err)
		fmt.Println("id=", id)
		fmt.Println("name=", name)
		fmt.Println("age=", age)
	}

	//删除数据
	// stmt, err = db.Prepare("DELETE FROM user WHERE uid=?")
	// checkErr(err)
	// res, err = stmt.Exec(id)
	// checkErr(err)
	// affect, err = res.RowsAffected()
	// checkErr(err)
	// fmt.Println(affect)

	db.Close()

}

func checkErr(err error) {
	if err != nil {
		panic(err)
	}
}
```
sql.Open()函数用来打开一个注册过的数据库驱动，第二个参数格式：
```
	user@unix(/path/to/socket)/dbname?charset=utf8
	user:password@tcp(localhost:5555)/dbname?charset=utf8
	user:password@/dbname
	user:password@tcp([de:ad:be:ef::ca:fe]:80)/dbname
```
db.Prepare()函数用来返回准备要执行的sql操作，然后返回准备完毕的执行状态。

db.Query()函数用来直接执行Sql返回Rows结果。

stmt.Exec()函数用来执行stmt准备好的SQL语句

我们可以看到我们传入的参数都是=?对应的数据，这样做的方式可以一定程度上防止SQL注入。