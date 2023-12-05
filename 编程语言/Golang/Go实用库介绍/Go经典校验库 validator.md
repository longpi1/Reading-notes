# Go 经典校验库 validator

## 简介

[validator](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator) 是一个结构体参数验证器。用于对数据进行校验。在 Web 开发中，对用户传过来的数据我们都需要进行严格校验，防止用户的恶意请求。例如日期格式，用户年龄，性别等必须是正常的值。经典的 [gin](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgin-gonic%2Fgin) 框架就是用了 validator 作为默认的校验器。它的能力能够帮助开发者最大程度地减少【基础校验】的代码，你只需要一个 tag 就能完成校验。

## 快速使用

```
package main

import (
	"fmt"

    "github.com/go-playground/validator/v10"
)

type User struct {
	Name string `validate:"min=6,max=10"`
	Age  int    `validate:"min=1,max=100"`
}

func main() {
    //启动时初始化
	validate := validator.New()

	u1 := User{Name: "lidajun", Age: 18}
	err := validate.Struct(u1)
	fmt.Println(err)

	u2 := User{Name: "dj", Age: 101}
	err = validate.Struct(u2)
	fmt.Println(err)
}
```

`validator`在结构体标签（`struct tag`）中定义字段的**约束**。使用`validator`验证数据之前，调用`validator.New()`创建一个**验证器**，这个验证器可以指定选项、添加自定义约束，然后通过调用它的`Struct()`方法来验证各种结构对象的字段是否符合定义的约束。

总的来说只需要三步即可：

1. 调用 `validator.New()` 初始化一个校验器；
2. 将【待校验的结构体】传入我们的校验器的 `Struct` 方法中；
3. 校验返回的 error 是否为 nil 即可。

在上面代码中，我们定义了一个结构体`User`，`User`有名称`Name`字段和年龄`Age`字段。通过`min`和`max`约束，我们设置`Name`的字符串长度为`[6,10]`之间，`Age`的范围为`[1,100]`。

第一个对象`Name`和`Age`字段都满足约束，故`Struct()`方法返回`nil`错误。第二个对象的`Name`字段值为`dj`，长度 2，小于最小值`min`，`Age`字段值为 101，大于最大值`max`，故返回错误：

```
<nil>
Key: 'User.Name' Error:Field validation for 'Name' failed on the 'min' tag
Key: 'User.Age' Error:Field validation for 'Age' failed on the 'max' tag
```

错误信息比较好理解，`User.Name`违反了`min`约束，`User.Age`违反了`max`约束，一眼就能看出问题所在。

## 约束

`validator`提供了非常丰富的约束可供使用，下面依次来介绍。

### 范围约束

我们上面已经看到了使用`min`和`max`来约束字符串的长度或数值的范围，下面再介绍其它的范围约束。范围约束的字段类型有以下几种：

- 对于数值，则约束其值；
- 对于字符串，则约束其长度；
- 对于切片、数组和`map`，则约束其长度。

下面如未特殊说明，则是根据上面各个类型对应的值与参数值比较。

- `len`：等于参数值，例如`len=10`；
- `max`：小于等于参数值，例如`max=10`；
- `min`：大于等于参数值，例如`min=10`；
- `eq`：等于参数值，注意与`len`不同。对于字符串，`eq`约束字符串本身的值，而`len`约束字符串长度。例如`eq=10`；
- `ne`：不等于参数值，例如`ne=10`；
- `gt`：大于参数值，例如`gt=10`；
- `gte`：大于等于参数值，例如`gte=10`；
- `lt`：小于参数值，例如`lt=10`；
- `lte`：小于等于参数值，例如`lte=10`；
- `oneof`：只能是列举出的值其中一个，这些值必须是数值或字符串，以空格分隔，如果字符串中有空格，将字符串用单引号包围，例如`oneof=red green`。



### 跨字段约束

`validator`允许定义跨字段的约束，即该字段与其他字段之间的关系。这种约束实际上分为两种，一种是参数字段就是同一个结构中的平级字段，另一种是参数字段为结构中其他字段的字段。约束语法很简单，要想使用上面的约束语义，只需要稍微修改一下。例如**相等约束**（`eq`），如果是约束同一个结构中的字段，则在后面添加一个`field`，使用`eqfield`定义字段间的相等约束。如果是更深层次的字段，在`field`之前还需要加上`cs`（可以理解为`cross-struct`），`eq`就变为`eqcsfield`。它们的参数值都是需要比较的字段名，内层的还需要加上字段的类型。

### 字符串

`validator`中关于字符串的约束有很多，这里介绍几个：

- `contains=`：包含参数子串，例如`contains=email`；
- `containsany`：包含参数中任意的 UNICODE 字符，例如`containsany=abcd`；
- `containsrune`：包含参数表示的 rune 字符，例如`containsrune=☻`；
- `excludes`：不包含参数子串，例如`excludes=email`；
- `excludesall`：不包含参数中任意的 UNICODE 字符，例如`excludesall=abcd`；
- `excludesrune`：不包含参数表示的 rune 字符，`excludesrune=☻`；
- `startswith`：以参数子串为前缀，例如`startswith=hello`；
- `endswith`：以参数子串为后缀，例如`endswith=bye`。

### 唯一性

使用`unqiue`来指定唯一性约束，对不同类型的处理如下：

- 对于数组和切片，`unique`约束没有重复的元素；
- 对于`map`，`unique`约束没有重复的**值**；
- 对于元素类型为结构体的切片，`unique`约束结构体对象的某个字段不重复，通过`unqiue=field`指定这个字段名。

### 特殊

有一些比较特殊的约束：

- `-`：跳过该字段，不检验；
- `|`：使用多个约束，只需要满足其中一个，例如`rgb|rgba`；
- `required`：字段必须设置，不能为默认值；
- `omitempty`：如果字段未设置，则忽略它。

### 其他

`validator`提供了大量的、各个方面的、丰富的约束，如`ASCII/UNICODE`字母、数字、十六进制、十六进制颜色值、大小写、RBG 颜色值，HSL 颜色值、HSLA 颜色值、**JSON 格式**、**文件路径**、URL、base64 编码串、**ip 地址**、ipv4、ipv6、UUID、经纬度等等。想看完整的建议参考[文档](https://link.juejin.cn/?target=https%3A%2F%2Fpkg.go.dev%2Fgithub.com%2Fgo-playground%2Fvalidator%2Fv10%23hdr-Baked_In_Validators_and_Tags) 以及仓库 [README](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator)



## VarWithValue方法

在一些很简单的情况下，我们仅仅想对两个变量进行比较，如果每次都要先定义结构和`tag`就太繁琐了。`validator`提供了`VarWithValue()`方法，我们只需要传入要验证的两个变量和约束即可

## 自定义约束

除了使用`validator`提供的约束外，还可以定义自己的约束。例如现在有个奇葩的需求，产品同学要求用户必须使用回文串作为用户名，我们可以自定义这个约束：

首先定义一个类型为`func (validator.FieldLevel) bool`的函数检查约束是否满足，可以通过`FieldLevel`取出要检查的字段的信息。然后，调用验证器的`RegisterValidation()`方法将该约束注册到指定的名字上。最后我们就可以在结构体中使用该约束。上面程序中，第二个对象不满足约束`palindrome`，输出：

## 错误处理

Golang 的 error 是个 interface，默认其实只提供了 Error() 这一个方法，返回一个字符串，能力比较鸡肋。同样的，validator 返回的错误信息也是个字符串：

```
Key: 'User.Name' Error:Field validation for 'Name' failed on the 'min' tag
```

这样当然不错，但问题在于，线上环境下，很多时候我们并不是【人工地】来阅读错误信息，这里的 error 最终是要转化成错误信息展现给用户，或者打点上报的。

其实，我们可以进行更精准的处理。`validator`返回的错误实际上只有两种，一种是参数错误，一种是校验错误。参数错误时，返回`InvalidValidationError`类型；校验错误时返回`ValidationErrors`，它们都实现了`error`接口。而且`ValidationErrors`是一个错误切片，它保存了每个字段违反的每个约束信息：

所以`validator`校验返回的结果只有 3 种情况：

- `nil`：没有错误；
- `InvalidValidationError`：输入参数错误；
- `ValidationErrors`：字段违反约束。

我们可以在程序中判断`err != nil`时，依次将`err`转换为`InvalidValidationError`和`ValidationErrors`以获取更详细的信息：

```
func processErr(err error) {
  if err == nil {
    return
  }

  invalid, ok := err.(*validator.InvalidValidationError)
  if ok {
    fmt.Println("param error:", invalid)
    return
  }

  validationErrs := err.(validator.ValidationErrors)
  for _, validationErr := range validationErrs {
    fmt.Println(validationErr)
  }
}

func main() {
  validate := validator.New()

  err := validate.Struct(1)
  processErr(err)

  err = validate.VarWithValue(1, 2, "eqfield")
  processErr(err)
}
```

## 总结

`validator`功能非常丰富，使用较为简单方便。本篇文章介绍的约束只是其中的冰山一角，想看完整的建议参考[文档](https://link.juejin.cn/?target=https%3A%2F%2Fpkg.go.dev%2Fgithub.com%2Fgo-playground%2Fvalidator%2Fv10%23hdr-Baked_In_Validators_and_Tags) 以及仓库 [README](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator)

## 参考链接

1. validator GitHub：https://github.com/go-playground/validator
2. Go 每日一库 GitHub：https://github.com/darjun/go-daily-lib
3. 解析 Golang 经典校验库 validator 用法：https://juejin.cn/post/7135803728916905997