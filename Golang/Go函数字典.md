## Go函数字典

#### Parse()

**Package:**flag

**Example:**

```Go
var configFile = flag.String("f", "etc/ai.json", "the config file")
//String函数接受三个参数，
//第一个为选项名字：即可以指定如 go run main.go -f [参数] 如果参数为空，则默认为etc/ai.json。
//第二个参数为默认值
//第三个参数是在--help时显示的释义
flag.Parse()
```

**Explain**:

​	这个函数主要是用来获取运行命令行时获取使用者指定的参数的。



#### ReadFile()

**Package:**ioutil

**Example:**

```go
ioutil.ReadFile("helloWorld.txt")
//这个函数有且仅有一个参数，用来获取目标文件的内容
```

**Explain:**

​	这个函数主要是读取当前目录下的文件内容。



#### Fatal()

**Package:**log

**Example:**

```go
log.Fatal("Oh,Some Error happened!")
//打印信息并退出程序，注意可以接受多个参数
```

**Explain:**

​	这个函数你可以理解为打印并退出(os.Exit(1))

## GO语法字典

#### ...

**Example:**

```go
ar1:=[]string{"1","2"}
ar2:=[]string{"3","4","5"}
ar3:=append(ar1,ar2...)
//它将ar2的元素一个一个追加到ar1上

func test(args ...string){
  return
}
//这个函数可以接受多个string参数
```

**Explain:**

​	它可以接受一个或者多个参数。