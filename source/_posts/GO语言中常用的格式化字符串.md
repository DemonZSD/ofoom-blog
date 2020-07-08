---
title: GO语言中常用的格式化字符串
date: 2018-07-07 09:59:22
categories: golang
tags: [golang, 字符串]
---

<!-- more -->

语言中常用的格式化字符串

在GO 语言中，格式化续延了C语言风格

在示例中，我们可能用到以下数据结构：

```
type User struct {
	name string
	age int
}
// 初始化 User 对象的指针 user
user := &User{
    name: "ofoom",
    age: 22,
}



```

字符串常用的格式化功能

**对象**

|格式化字符|功能|示例|
|-----|-----|-----|
|`%v`|按值的本来值输出|```fmt.Sprintf("my info is %v", user```，格式化结果为：`my info is {ofoom 22}`|
|`%+v`|在`%v`的基础上，将结构体的字段名和值展开|```fmt.Sprintf("my info is %v", user```，格式化结果：`my info is {name:ofoom age:22}`|
|`%#v`|输出 Go 语言语法格式的类型和值|```fmt.Sprintf("my info is %#v", user```，格式化结果为：`my info is main.User{name:"ofoom", age:22}`|
|`%T`|输出Go 语言语法格式的类型|```fmt.Sprintf("my info is %T", user```，格式化结果为：`my info is main.User`，注意区分 `%#v`，方便记忆。|

---

**整型**

|格式化字符|功能|示例|
|-----|-----|-----|
|`%b`|整型以二进制方式显示|```fmt.Sprintf("%b", user.age)```，格式化结果为：`10110`|
|`%o`|整型以八进制方式显示|```fmt.Sprintf("%o", user.age)```，格式化结果为：`26`|
|`%d`|整型以十进制方式显示|```fmt.Sprintf("%d", user.age)```，格式化结果为：`22`|
|`%x`|整型以十六进制方式显示|```fmt.Sprintf("%x", user.age + 9)```，格式化结果为：`1f`|
|`%X`|整型以十六进制方式显示，字母大写|```fmt.Sprintf("%X", user.age + 9)```，格式化结果为：`1F`|
|`%U`|Unicode 字符|```fmt.Sprintf("%U", user.age)```，格式化结果为：`U+0016`|

---

**浮点数**

|格式化字符|功能|示例|
|-----|-----|-----|
|`%f`|浮点数|```fmt.Sprintf("%U", 12.34)```，格式化结果为：`12.340000`|


---

**其他**

|格式化字符|功能|示例|
|-----|-----|-----|
|`%%`|输出 `%` 本体|```fmt.Sprintf("%%")```，格式化结果为：`%`|
|`%p`|指针，十六进制方式显示|```fmt.Sprintf("%p", user)```，格式化结果为：`0xc000004460`|


