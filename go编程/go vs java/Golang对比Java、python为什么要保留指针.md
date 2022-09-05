# Golang对比Java、python为什么要保留指针

## 为什么要用指针？

平时我们在Golang使用指针一般是为了以下的情况：

- **方法直接修改原来对象**
- **保证参数传递的自由，可以在传递重量级对象时使用指针**

但Go 保留指针不仅仅是为了解决传递参数的问题，还跟它的语言特性有密不可分的联系。

## 值语义

Go 里面的变量是**值语义**，这个跟 C/C++是一脉相承的。比如一个结构体变量赋值给另外一个变量就是一次内存拷贝，而不是只拷贝一个指针，因此需要指针来表达引用语义，关于拷贝的具体实现可以了解[直接值部与间接值部的实现](https://gfw.go101.org/article/value-part.html)。

关于值语义：**值语义(value semantics)**指的是对象的拷贝与原对象无关，就像拷贝 int 一样。C++ 的内置类型(bool/int/double/char)都是值语义，标准库里的 complex<> 、pair<>、vector<>、map<>、string 等等类型也都是值语意，拷贝之后就与原对象脱离关系。同样，Java 语言的 primitive types 也是值语义。

## 优点

**复杂的高级类型占用的内存往往相对较大，存储在 [heap](https://www.zhihu.com/search?q=heap&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A1665421830}) 中，GC 回收频率相对较低，代价也较大，因此传引用/指针可以避免进行成本较高的复制操作，并且节省内存，提高程序运行效率。**

为什么要保留值语义，而不是像 Java 或者 Python 一样让复合类型默认都是指针类型呢？因为值语义带来了如下好处：

- **[结构体](https://www.zhihu.com/search?q=结构体&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2242103027})可以直接用来比较相等，而非比较指针，Java 里面的 == 操作符除了基本类型有用，其他类型几乎没用。**
- **与 C 语言更好地交互。Go 可以通过 cgo 与 C 语言无缝交互。Go 里面的结构体基本上不用特殊处理就能传递给 C 的函数使用。主要得益于 Go 的结构体和 C 的一样都是值类型。**
- **开发者能更好的掌控内存布局。一个结构体数组就是一段连续内存，而不是一个指针数组。**
- **减轻 GC 压力。紧凑的内存布局减少了 GC 对象的个数，比如一个100w 长度的结构体数组就是一个 GC 对象，而不是100w 个。**
- **减轻堆内存的分配压力。函数通过传值的方式传递参数后，原变量不会发生逃逸，可以被分配在栈上**

Go 为了内存安全，虽然有指针，但不支持指针算数，但结合 unsafe.Pointer 也可以完成一些非常规情景下的精细内存操作。比如结合 [mmap](https://www.zhihu.com/search?q=mmap&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2242103027}) 实现堆外内存管理，runtime 里面的内存管理就是这么来的，完全不用另外用 C 语言来实现。 这也是可以使用 Go 语言来写操作系统（[eggos](https://link.zhihu.com/?target=https%3A//github.com/icexin/eggos)）的原因。

## 总结：

**Go 的指针一方面提供了引用语义，另一方面像 C 语言一样给了开发者灵活管理内存的能力。**

参考链接：樊冰心：https://www.zhihu.com/question/399589293/answer/2242103027

