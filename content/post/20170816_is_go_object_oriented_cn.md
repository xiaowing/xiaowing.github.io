+++
title = "[译文]GO语言是面向对象的吗？"
author = "xiaowing"
date = 2017-08-16T21:48:52+08:00
draft = false
tags =  [
    "golang",
    "译作",
    ]
topics = [
    "技术",
]
+++

周一在微信上收到了[Go中国](http://www.weixinyidu.com/a_277786)公众号推送的一篇文章 [Is GO object oriented](https://flaviocopes.com/golang-is-go-object-oriented/) ,读完以后感觉观点还是很有意思的，与我的所见有很多相似之处，所以就饶有兴趣地把它翻译成中文，也算是作为"友军"的一点贡献吧。

<!--more-->

## GO语言是面向对象的吗？

原题: IS GO OBJECT ORIENTED?

作者: Flavio Copes

链接: [https://flaviocopes.com/page/about/](https://flaviocopes.com/page/about/)

有时候我会读到文章宣称"GO语言是面向对象的语言"，有时候我又会读到一篇持相反观点的文章宣称"GO语言无法做到面向对象编程因为它根本没有'类'".

有鉴于此，我写了这篇文章来阐述这个话题: **GO语言到底是不是面向对象的语言**？

如果你习惯了从某种特定编程语言的视角来思考问题，那么在这个话题上，你可能会由于自己所惯于使用的语言不同而得出截然相反的答案。比如说，如果你之前习惯了使用C语言，那么很显然GO语言拥有太多面向对象的特性；但是如果你之前习惯了使用Java，那么GO语言看起来就不那么"面向对象"了。

因此关于这个话题，你首先要做的是摒弃其他语言带给你的先入为主的观念并学会用GO语言的思维模式来思考。

然后我就可以给出我的答案了: YES. GO语言是一门面向对象的语言, 而且是以一种很清爽的方式实现了面向对象.

在[GO语言官方文档的FAQ](https://golang.org/doc/faq#Is_Go_an_object-oriented_language)中也表述过下述内容

> Is Go an object-oriented language?
> 
> Yes and no. Although Go has types and methods and 
> allows an object-oriented style of programming, there 
> is no type hierarchy. The concept of “interface” in Go
> provides a different approach that we believe is easy 
> to use and in some ways more general. There are also 
> ways to embed types in other types to provide something
> analogous—but not identical—to subclassing. Moreover,
> methods in Go are more general than in C++ or Java:
> they can be defined for any sort of data, even built-in
> types such as plain, “unboxed” integers. They are not
> restricted to structs (classes).
> Also, the lack of a type hierarchy makes “objects” in
> Go feel much more lightweight than in languages such as 
> C++ or Java.

### 集众家所长

GO语言从[过程式编程语言](https://zh.wikipedia.org/wiki/%E8%BF%87%E7%A8%8B%E5%BC%8F%E7%BC%96%E7%A8%8B), [函数式编程语言](https://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B8%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80)以及[面向对象编程语言](https://zh.wikipedia.org/wiki/%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)中都汲取了一些概念并将它们组装在一起后，将那些剩余的概念舍弃从而创造了一门极具特性但又十分地道的编程语言。

### 要结构体,不要类

GO语言中并没有传统面向对象编程语言中的"类"(`class`)的概念，取而代之的是使用了"结构体"(`struct`)。但是GO语言的结构体可比C语言中的同名前辈要强大得多。在GO语言中,结构体以及结构体的方法(`method`)发挥了与传统意义上的"类"相同的作用，同时在概念上又清晰分明——结构体只负责管理状态，不管理行为;而结构体的方法则负责定义结构体的行为，比如说允许他们更改状态.

### 封装的奥义

我认为GO语言中最好的一个特性就是，直接明了地通过字段名, 方法名以及函数名的首字母大写来保证它们的访问可见性为**public**。其他的那些以小写字母开头的字段等则默认为包(`package`)内私有(private)并且无法被导出至包外。这个特性使得开发者们一眼就可以看出来哪些字段/方法/属性是public的，哪些是private的。另外，由于GO语言中没有**继承**这一概念，所以GO语言中的访问修饰也就根本没有`protected`的概念.

### 无继承

GO语言中没有[继承](https://zh.wikipedia.org/wiki/%E7%BB%A7%E6%89%BF_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6)),这一点在[GO语言官方文档的FAQ](https://golang.org/doc/faq#Why_is_there_no_type_inheritance)也有明确说明:

> Object-oriented programming, at least in the best-known
> languages, involves too much discussion of the 
> relationships between types, relationships that often 
> could be derived automatically. Go takes a different 
> approach.

### 用组合代替继承

程序设计理念中最广为认知的准则之一即是 [用组合代替继承](https://en.wikipedia.org/wiki/Composition_over_inheritance)。这一准则在[四人帮](https://en.wikipedia.org/wiki/Design_Patterns)的那本极富盛名的《设计模式》也屡有提及。而在GO语言中，这一准则被发挥得淋漓尽致。

当我们在定义一个结构体时，我们可以追加类型为另一个结构体的匿名字段。这样一来，我们定义的这个结构体也就同时拥有了另一个结构体的所有字段以及方法。这种技法被称之为**Struct Embedding**。

````
package main

import (
	"fmt"
)

type Dog struct {
	Animal
}
type Animal struct {
	Age int
}

func (a *Animal) Move() {
	fmt.Println("Animal moved")
}
func (a *Animal) SayAge() {
	fmt.Printf("Animal age: %d\n", a.Age)
}
func main() {
	d := Dog{}
	d.Age = 3
	d.Move()
	d.SayAge()
}
````
[执行结果](https://play.golang.org/p/IxMoeByWp5) (需要科学上网)

### 接口

忘记Java或PHP风格的接口概念吧！GO语言中的接口是截然不同的，而其中最关键的一个特性即是: **接口是隐式实现的**——不需要在定义类型时去显式声明一下类型实现了哪些接口。

还是摘抄自[GO语言官方文档的FAQ](https://golang.org/doc/faq#Why_is_there_no_type_inheritance):

> Rather than requiring the programmer to declare ahead 
> of time that two types are related, in Go a type 
> automatically satisfies any interface that specifies a 
> subset of its methods.

通常，接口的定义都很短，甚至有可能至包含一个方法声明。在地道的GO语言实践中，你不应该看到一个接口拥有长长的方法列表。

借助这样的接口就可以很优雅地实现[多态](https://zh.wikipedia.org/wiki/%E5%A4%9A%E5%9E%8B_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6)): 一个方法如果接受某一个接口，那么就意味着这个方法接受了任何实现了该接口的对象.

### 方法

GO语言中的类型往往都拥有方法，但是这些方法的定义是独立于类型定义而存在的。GO语言在语法层面通过一个类似Javascript中prototype方法的特性来实现了这种定义的独立性。

````javascript
function Person(first, last) {
    this.firstName = first;
    this.lastName = last;
}
Person.prototype.name = function() {
    return this.firstName + " " + this.lastName;
};
p = new Person("Flavio", "Copes")
p.name() // Flavio Copes
````

在GO语言中，代码应该写成这样:

````
package main

import (
	"fmt"
)
type Person struct {
	firstName string
	lastName  string
}
func (p Person) name() string {
	return p.firstName + " " + p.lastName
}
func main() {
	p := Person{"Flavio", "Copes"}
	fmt.Println(p.name())
}
````

### 关联方法与类型

方法可以被关联至任何类型,甚至是"曲线救国"式地关联到GO语言中的基础类型。由于方法只能被关联至在同一个包中定义的类型,所以我们无法直接"扩展"基础类型。但事实上，我们可以通过对基础数据类型定义别名从而达到扩展的目的。

````
package main

import (
	"fmt"
)

type Amount int

func (a *Amount) Add(add Amount) {
	*a += add
}

func main() {
	var a Amount
	a = 1
	a.Add(2)
	fmt.Println(a)
}
````
[显示结果](https://play.golang.org/p/ONlUmue1jA) (需要科学上网)

### 函数

让我们回想一下传统的面向对象编程语言吧，比如Java。想想看你曾经定义过多少次包含的全是static方法的名为"Utils"的类?

这样的现象来源与传统的面向对象变成语言中的一个观念: **一切皆是对象**，因此函数定义必须放在一个类里面。所幸的是，这种现象不会发生在GO语言中，因为GO语言有真正的"函数"这一概念。在真实世界中，并非所有事物都必须是一个对象。"类"和"对象"的概念很有用，但也不能到处都用。

在GO语言中，并非所有东西都是对象(若严格从技术角度而言，GO语言没有东西是对象。但通常人们会将一个类型的实例或者变量称之为"对象"),方法仅仅是指那些被关联至某一类型的函数。但GO语言同时又允许函数脱离于对象而独立存在,就如同C语言的函数一样。

所以，GO语言既允许方法存在，也允许函数存在。而且，函数是第一优先的(函数类型可用作定义结构体字段,函数可作为参数传递至其他函数，函数也可作为返回值被函数或方法返回)。

### 大道至简

综上所述，GO语言对于面向对象的实现非常灵活且直接。不用再纠结于类和继承，你可以减少对源码模板文件的依赖，而且不用再一点一点地推敲类与类之间的理想层级结构，从此你只需根据需求自由地对类型进行组合或拆解即可。



