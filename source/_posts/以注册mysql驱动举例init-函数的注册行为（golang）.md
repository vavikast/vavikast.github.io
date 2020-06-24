---
title: 以注册mysql驱动举例init()函数的注册行为（golang）
catalog: true
date: 2020-06-24 10:58:59
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- go
catagories:
- go
updateDate: 2020-06-24 10:58:59
---

Go语言中的database/sql包提供了保证SQL或类SQL数据库的泛用接口，并不提供具体的数据库驱动。使用database/sql包时必须注入（至少）一个数据库驱动。

​		但是import _ "github.com/go-sql-driver/mysql" 的意义不是很好理解。

​		为此，我通过自己写的三个简单程序，演示init()的注册行为。

​		

代码的目录组织结构如下：

```
awesomeProject/talkischeap/main.go  //主函数
awesomeProject/talkischeap/showmecode/main.go  // 类型定义函数
awesomeProject/talkischeap/showyourcode/main.go //init()函数
```



主函数

```
package main

import "awesomeProject/talkischeap/showmecode"  //类比database/sql
import _ "awesomeProject/talkischeap/showyourcode" //类比github.com/go-sql-driver/mysql
//import _ "github.com/go-sql-driver/mysql"
//import "database/sql"
func main()  {
	// 程序运行的顺序 (1)引入的包 (2) 当前包中的常量 (3) 当前包中的变量（4）当前包的init （5）main函数
	//本例解释如下
	//1. 导入showmecode包时，导入了showmecode的包中的变量driver，这是个引用类型。
	//2. 导入showyourcode包时,导入了init函数，init函数调用了showmecode中的Register函数。
	//3.driver被赋值。影响了全局的driver。
	//4.main函数中调用showmecode.printDriver()函数，把driver的内容打印出来.

	//直接打印driver的值
	showmecode.PrintDriver()
}

```

```
运行结果如下：
map[Driver :  yes,It's it!]
```



showmecode代码包

```
package showmecode

import "fmt"

//声明map类型数据driver
var driver = make(map[string]string)

//注册函数： 注册driver
func Register(name string, value string) {
	driver[name] = value
}

//打印函数： 打印driver
func PrintDriver() {
	fmt.Println(driver)
}

```

showyourcode代码包

```
package showyourcode

import (
	"awesomeProject/talkischeap/showmecode"
)
//init()当作注册使用，调用了showcode的代码
func init()  {
	showmecode.Register("Driver ","  yes,It's it!")
}


```

​		

​		简单梳理了init函数的注册行为，但是mysql的驱动注册并没有那么简单，只是抛砖引玉，后续深入理解后再做详细说明。

​	

参考连接：

[5分钟理解golang的init函数](https://zhuanlan.zhihu.com/p/34211611)

[golangmysql驱动的解析](https://hacpai.com/article/1573025534337)

[导入包的截个说明](https://www.cnblogs.com/shengulong/p/10230644.html)





