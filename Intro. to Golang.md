### Intro. to Golang

reference：

https://zhuanlan.zhihu.com/p/403114396

#### 代码管理对比（C语言）

C语言通过文件来管理代码，想使用某一个函数时，需要include导入对应的.h文件

Go语言中通过包来管理代码，Go语言没有.h文件的概念, 在Go中想使用某一个函数时, 只需要import导入对应的包即可

C语言中函数、变量公私有管理：通过extern和static实现是否公开函数和变量

Go语言中函数、变量公私有管理：通过函数名称首字母大小写实现是否公开函数，通过变量名称首字母大小写实现是否公开变量

C语言关键字：「图片」

Go语言关键字：「图片」

C语言数据类型：「图片」

Go语言关键字：「图片」

##### 常量变量对比

- C语言定义常量和变量格式

```c
数据类型 变量名称 = 值;
const 数据类型 常量名称 = 值;
```

- Go语言定义常量和变量格式

- - 除了以下标准格式外,Go语言还提供了好几种简单的语法糖

```go
var 变量名称 数据类型 = 值;
const 变量名称 数据类型 = 值;
```

##### 注释对比

- 和C语言一样,Go语言也支持单行注释和多行注释, 并且所有注释的特性都和C语言一样

- - 单行注释 `// 被注释内容`
  - 多行注释 `/* 被注释内容*/`

在Go语言中,官方更加推荐使用单行注释,而非多行注释

##### 运算符对比

- 算数运算符和C语言几乎一样

- - Go语言中++、--运算符不支持前置

  - - 错误写法: ++i; --i;

Go语言中++、--是语句,不是表达式,所以必须独占一行

- - - 错误写法: a = i++; return i++;

位运算符和C语言几乎一样

- 新增一个&^运算符

「图片」

##### 流程控制语句对比

- C语言流程控制中的if、switch、for在Go语言都可以使用

- C语言中的四大跳转语句return、break、continue、goto在Go语言都可以使用

- Go语言除了实现C语言中if、switch、for、return、break、continue、goto的基本功能以外,还对if、switch、for、break、continue进行了增强

- - 例如: if 条件表达式前面可以添加初始化表达式
  - 例如: break、continue可以指定标签
  - 例如: switch语句可以当做if/elseif来使用
  - ... ...

值得注意的是Go语言中没有while循环和dowhile循环, 因为它们能做的Go语言中的for循环都可以做

##### 函数和方法对比

- C语言定义函数格式

```c
返回值类型 函数名称(形参列表) {
		函数体相关语句;
		return 返回值;
}
```

- Go语言定义函数格式

```go
func  函数名称(形参列表)(返回值列表) {
		函数体相关语句;
    return 返回值;
}
```

- C语言中没有方法的概念, 但是Go语言中有方法

- - 对于初学者而言,可以简单的把方法理解为一种特殊的函数

```go
func (接收者 接受者类型)函数名称(形参列表)(返回值列表) {
		函数体相关语句;
		return 返回值;
}
```



##### Go语言程序组成

- 和C语言程序一样，Go语言程序也是由众多函数组成的
- 和C语言程序一样，程序运行时系统会自动调用名称叫做main的函数
- 和C语言程序一样，如果一个程序没有主函数，则这个程序不具备运行能力
- 和C语言程序一样，一个Go语言程序有且只能有一个主函数

##### Go语言主函数定义格式

- Go语言main函数格式

- - func 告诉系统这是一个函数
  - main主函数固定名称
  - 函数左括号必须和函数名在同一行
  - main函数必须在main包中

- ```go
  package main // 告诉系统当前代码属于main这个包
  import "fmt" // 导入打印函数对应的fmt包
  func main() {
          // 通过包名.函数名称的方式, 利用fmt包中的打印函数输出语句
      fmt.Println("Hello World!!!")
  }
  ```



##### Go语言编码风格

- 1.go程序编写在.go为后缀的文件中
- 2.包名一般使用文件所在文件夹的名称
- 2.包名应该简洁、清晰且全小写
- 3.main函数只能编写在main包中
- 4.每一条语句后面可以不用编写分号(推荐)
- 5.如果没有编写分号,一行只能编写一条语句
- 6.函数的左括号必须和函数名在同一行
- 7.导入包但没有使用包编译会报错
- 8.定义局部变量但没有使用变量编译也会报错
- 9.定义函数但没有使用函数不会报错
- 10.给方法、变量添加说明,尽量使用单行注释

##### 标识符

- Go语言中的标识符和C语言中的标识符的含义样, 是指程序员在程序中自己起的名字(变量名称、函数名称等)

- 和C语言一样Go语言标识符也有一套`命名规则`, Go语言标识符的命名规则几乎和C语言一模一样

- - 只能由字母(a~z、 A~Z)、数字、下划线组成
  - 不能包含除下划线以外的其它特殊字符串
  - 不能以数字开头
  - 不能是Go语言中的关键字
  - 标识符严格区分大小写, test和Test是两个不同的标识符

- 和C语言标识符命名规则不同的是

- - Go语言中_单独作为标识符出现时, 代表`空标识符`, 它对应的值会被忽略

```go
package main

import "fmt"

func main() {
 // 将常量10保存到名称叫做num的变量中
 var num int = 10
 fmt.Println("num = ", num)

 // 忽略常量20,不会分配存储空间,也不会保存常量20
 //var _ int = 20
 //fmt.Println("_ = ", _) // cannot use _ as value

 // Go语言中如果定义了变量没有使用, 那么编译会报错(sub declared and not used)
 // 所以如果我们只使用了sum,没有使用sub会报错
 // 为了解决这个问题, 我们可以使用_忽略sub的值
 //var sum, sub int = calculate(20, 10)
 var sum, _ int = calculate(20, 10)
 fmt.Println("sum = ", sum)

}

func calculate(a, b int)(int, int)  {
 var sum int = a + b
 var sub int = a - b
 return sum, sub
}
```

- 和C语言一样,标识符除了有`命名规则`以外,还有标识符`命名规范`

- - 规则必须遵守, 规范不一定要遵守, 但是建议遵守

  - Go语言的命名规范和C语言一样, 都是采用驼峰命名, 避免采用_命名

  - - 驼峰命名: sendMessage / sayHello
    - _命名: send_message / say_hello

##### Go语言中定义变量有三种格式

Go语言中变量的概念和C语言中也一样

```go
// 标准格式
var 变量名称 数据类型 = 值;
// 自动推到类型格式
var 变量名称 = 值;
// 简短格式(golang官方推荐格式)
变量名称 := 值;
package main
import "fmt"
func main() {
    var num1 int // 先定义
    num1 = 10 // 后赋值
    fmt.Println("num1 = ", num1)

    var num2 int = 20 // 定义的同时赋值
    fmt.Println("num2 = ", num2)

    var num3  = 30 // 定义的同时赋值, 并省略数据类型
    fmt.Println("num3 = ", num3)
    
    num4  := 40 // 定义的同时赋值, 并省略关键字和数据类型
    /*
    num4  := 40 等价于
    var num4 int
    num4 = 40
    */
    fmt.Println("num4 = ", num4)
}
```

- 一定不要把:=当做赋值运算符来使用

```go
package main
import "fmt"
var num = 10 // 定义一个全局变量
func main() {
    num := 20 // 定义一个局部变量
    fmt.Println("num = ", num)
        test()
}
func test() {
    fmt.Println("num = ", num) // 还是输出10
}
```

- :=只能用于定义局部变量,不能用于定义全局变量

```go
package main
import "fmt"
num := 10 // 编译报错
func main() {
    fmt.Println("num = ", num)
}
```

- 使用:=定义变量时,不能指定var关键字和数据类型

```go
package main
import "fmt"
func main() {
    //var num int := 10 // 编译报错
    //var num := 10 // 编译报错
    num int := 10 // 编译报错
    fmt.Println("num = ", num)
    fmt.Println("num = ", num)
}
```

- 变量组中不能够使用:=

```go
package main
import "fmt"
func main() {
    var(
        num := 10 // 编译报错
    )
    fmt.Println("num = ", num)
}
```

- 通过:=同时定义多个变量, 必须给所有变量初始化

```go
package main
import "fmt"
func main() {
    //num1, num2 := 666, 888 // 正确
    num1, num2 := 666 // 报错
    fmt.Printf("%d, %d\n", num1, num2)
}
```

- C语言中全局变量没有赋值,那么默认初始值为0, 局部变量没有赋值,那么默认初始值是随机值

- Go语言中无论是全局变量还是局部变量,只要定义了一个变量都有默认的0值

- - int/int8/int16/int32/int64/uint/uint8/uint16/uint32/uint64/byte/rune/uintptr的默认值是0
  - float32/float64的默认值是0.0
  - bool的默认值是false
  - string的默认值是""
  - pointer/function/interface/slice/channel/map/error的默认值是nil
  - 其它复合类型array/struct默认值是内部数据类型的默认值

```go
package main
import "fmt"
func main() {
    var intV int // 整型变量
    var floatV float32 // 实型变量
    var boolV bool // 布尔型变量
    var stringV string // 字符串变量
    var pointerV *int // 指针变量
    var funcV func(int, int)int // function变量
    var interfaceV interface{} // 接口变量
    var sliceV []int // 切片变量
    var channelV chan int // channel变量
    var mapV map[string]string // map变量
    var errorV error // error变量

    fmt.Println("int = ", intV) // 0
    fmt.Println("float = ", floatV) // 0
    fmt.Println("bool = ", boolV) // false
    fmt.Println("string = ", stringV) // ""
    fmt.Println("pointer = ", pointerV) // nil
    fmt.Println("func = ", funcV) // nil
    fmt.Println("interface = ", interfaceV) // nil
    fmt.Println("slice = ", sliceV) // []
    fmt.Println("slice = ", sliceV == nil) // true
    fmt.Println("channel = ", channelV) // nil
    fmt.Println("map = ", mapV) // map[]
    fmt.Println("map = ", mapV == nil) // true
    fmt.Println("error = ", errorV) // nil

    var arraryV [3]int // 数组变量
    type Person struct{
        name string
        age int
    }
    var structV Person // 结构体变量
    fmt.Println("arrary = ", arraryV) // [0, 0, 0]
    fmt.Println("struct = ", structV) // {"" 0}
}
```

##### 数据类型转换

- C语言中数据可以隐式转换或显示转换, 但是Go语言中数据`只能显示转换`

- Go语言数值类型之间转换

- - 格式: `数据类型(需要转换的数据)`
  - 注意点: 和C语言一样数据可以从大类型转换为小类型, 也可以从小类型转换为大类型. 但是大类型转换为小类型可能会丢失精度

```go
package main
import "fmt"
func main() {
 var num0 int = 10
 var num1 int8 = 20
 var num2 int16
 //num2 = num0 // 编译报错, 不同长度的int之间也需要显示转换
 //num2 = num1 // 编译报错, 不同长度的int之间也需要显示转换
 num2 = int16(num0)
 num2 = int16(num1)
 fmt.Println(num2)

 var num3 float32 = 3.14
 var num4 float64
 //num4 = num3 // 编译报错, 不同长度的float之间也需要显示转换
 num4 = float64(num3)
 fmt.Println(num4)

 var num5 byte = 11
 var num6 uint8 // 这里不是隐式转换, 不报错的原因是byte的本质就是uint8
 num6 = num5
 fmt.Println(num6)

 var num7 rune = 11
 var num8 int32
 num8 = num7 // 这里不是隐式转换, 不报错的原因是byte的本质就是int32
 fmt.Println(num8)
}
```

##### Go语言常量

- Go语言自定义常量注意点

- - 定义的局部变量或者导入的包没有被使用, 那么编译器会报错,无法编译运行
  - 但是定义的常量没有被使用,编译器不会报错, 可以编译运行

```go
package main
import "fmt"
func main() {
		// 可以编译运行
		const PI float32 = 3.14
}
```

- - 在常量组中, 如果上一行常量有初始值,但是下一行没有初始值, 那么下一行的值就是上一行的值

```go
package main
import "fmt"
func main() {
	const (
		num1 = 998
		num2 // 和上一行的值一样
		num3 = 666
		num4 // 和上一行的值一样
		num5 // 和上一行的值一样
	)
	fmt.Println("num1 = ", num1) // 998
	fmt.Println("num2 = ", num2) // 998
	fmt.Println("num3 = ", num3) // 666
	fmt.Println("num4 = ", num4) // 666
	fmt.Println("num5 = ", num5) // 666

	const (
		num1, num2 = 100, 200
		num3, num4 // 和上一行的值一样, 注意变量个数必须也和上一行一样
	)
	fmt.Println("num1 = ", num1)
	fmt.Println("num2 = ", num2)
	fmt.Println("num3 = ", num3)
	fmt.Println("num4 = ", num4)
}
```

- 枚举常量

- - C语言中枚举类型的本质就是整型常量 
  - Go语言中没有C语言中明确意义上的enum定义, 但是可以借助iota标识符来实现枚举类型



- C语言枚举格式:

```go
 enum　枚举名　{
    枚举元素1,
    枚举元素2,
    … …
 };
C语言枚举中,如果没有指定初始值,那么从0开始递增


#include <stdio.h>
int main(int argc, const char * argv[])
{
 enum Gender{
 male,
 female,
 yao,
    };
//    enum Gender g = male;
//    printf("%d\n", g); // 0
//    enum Gender g = female;
//    printf("%d\n", g); // 1
 enum Gender g = yao;
 printf("%d\n", g); // 2
 return 0;
}
```

- - C语言枚举中, 如果指定了初始值,那么从指定的数开始递增

```c
#include <stdio.h>
int main(int argc, const char * argv[])
{
 enum Gender{
 male = 5,
 female,
 yao,
    };
//    enum Gender g = male;
//    printf("%d\n", g); // 5
//    enum Gender g = female;
//    printf("%d\n", g); // 6
 enum Gender g = yao;
 printf("%d\n", g); // 7
 return 0;
}
```



- Go语言实现枚举格式

```go
const(
  枚举元素1 = iota
  枚举元素2 = iota
  ... ...
)
利用iota标识符标识符实现从0开始递增的枚举

package main
import "fmt"
func main() {
 const (
  male = iota
  female = iota
  yao = iota
 )
 fmt.Println("male = ", male) // 0
 fmt.Println("male = ", female) // 1
 fmt.Println("male = ", yao) // 2
}
```

- iota注意点:

- - 在同一个常量组中,iota从0开始递增, `每一行递增1`
  - 在同一个常量组中,只要上一行出现了iota,那么后续行就会自动递增

```go
 package main
 import "fmt"
 func main() {
 const (
  male = iota // 这里出现了iota
  female // 这里会自动递增
  yao
 )
 fmt.Println("male = ", male) // 0
 fmt.Println("male = ", female) // 1
 fmt.Println("male = ", yao) // 2
 }
```