#                          Golang的语言特性与鸭子类型

## 前言

### 什么是鸭子类型？

> *Suppose you see a bird walking around in a farm yard. This bird has no label that says 'duck'. But the bird certainly looks like a duck. Also, he goes to the pond and you notice that he swims like a duck. Then he opens his beak and quacks like a duck. Well, by this time you have probably reached the conclusion that the bird is a duck, whether he's wearing a label or not.*

上述是对于**鸭子类型**的最有名的阐述。意思是对于事物类型的判断， 不取决于事物本身预设的标签（label）， 而取决于判断者判断时需要用到的条件， 如果事物拥有符合条件的属性，那么在判断者眼中它就是那种类型。

在程序设计中，鸭子类型（英语：Duck typing）是动态类型和某些静态语言的一种对象推断风格。这种风格适用于动态语言(比如PHP、Python、Ruby、Typescript、Lua、JavaScript、Java、Groovy、C#等)和静态语言(比如Golang来说，静态类型语言在编译时便已确定了变量的类型，但是Golang的实现是：在编译时推断变量的类型)，支持"鸭子类型"的语言的解释器/编译器将会在解析(Parse)或编译时，推断对象的类型。

动态语言和静态语言的差别在此就有所体现。**静态语言在编译期间就能发现类型不匹配的错误，不像动态语言，必须要运行到那一行代码才会报错**。静态语言要求程序员在编码阶段就要按照规定来编写程序，为每个变量规定数据类型，这在某种程度上，加大了工作量，也加长了代码量。动态语言则没有这些要求，可以让人更专注在代码逻辑上。

Go 语言作为一门现代静态语言，是有后发优势的。它引入了动态语言的便利，同时又会进行静态语言的类型检查。Go 采用了折中的做法：**不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。**

## 动态语言(python)的duck typing

假如有个叫`say_quack`的`Python`函数, 它接受一个接口参数，参数的类型不固定，只要有`quack`方法就可以啦。

```python
def say_quack(duck)
    duck.quack()

class RealDcuk:
    def quack(self):
        print("quack quack")

class ToyDuck:
    def quack(self):
        print("squee squee")

duck = RealDuck()
say_quack(duck)

toyDuck = ToyDuck()
say_quack(duck)
```

可以看出动态语言的duck typing非常灵活方便，类型的检测和使用不依赖于编译器的静态检测，而是依赖文档、清晰的代码和测试来确保正确使用。这样其实是牺牲了安全性来换取灵活性。 假设你没有认真看文档，不知道`say_quack`方法的`duck`参数是需要`quack`方法， 你编写了一个其他类，它只有一个`run`方法， 你把它的对象当成参数给`say_quack`编译时也是不会报错的。只有在运行时才会报错， 这样就存在很大的安全隐患。

所以，有没有一种折中（tradeoff）， 兼顾这种duck typing的灵活性和静态检测的安全性呢？

## go语言接口的隐式实现

假如你有个golang的接口叫Duck：

```go
type Duck interface {
    quack()
}
```

任何拥有`quack`方法的类型， 都隐式地（implicitly）实现了`Duck`接口， 并能当做Duck接口使用。

```go
package main
import (
    "fmt"
)

type Duck interface {
    quack()
}

type RealDuck struct {
}

func (d RealDuck) quack() {
    fmt.Println("quack quack")
}

type ToyDuck struct {
}

func (d ToyDuck) quack() {
    fmt.Println("squee squee")
}

func sayQuack(d Duck) {
    d.quack()
}
func main() {
    realDuck := RealDuck{}
    toyDuck := ToyDuck{}

    sayQuack(realDuck)

    sayQuack(toyDuck)

}
```

如果你有一个`Dog`类型， 它没有quack方法， 当你用它做sayQuack参数时， 编译时就会报错。另外来说， 如果接口使用者定义了一个新的接口也拥有quack方法， 那上面的`RealDuck`和`ToyDuck`也可以当做新的接口来使用。

这样就达到了一个灵活性和安全性的平衡。因为go对接口的实现是隐式的， 所以它的接口类型在使用之前是不固定的， 它可以灵活的变成各种接口类型，只要它满足使用者的对接口的要求。 又因为使用者使用接口时在编译时就对接口实现者有没有满足接口需求进行了检测，所以又兼顾了安全性。



## 参考链接

   1.[go语言与鸭子类型的关系](https://golang.design/go-questions/interface/duck-typing/)

2. [Golang 的 interface 及 duck typing 鸭子类型](https://my.oschina.net/chinaliuhan/blog/3123026)

   