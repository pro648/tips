这篇文章是我整理的`struct`、`class`的学习笔记，主要用来整理自己的思路。

struct和class用于在app内存储数据。因其有很多相似性，有时难以选择。

## 1. 值类型和引用类型

Swift 中数据类型分为两种：

- 值类型 value type：每个实例都有唯一副本的数据。值类型的数据结构有`struct`、`enum`和tuple（元组），Array和String也是struct的实例。可以以引用方式传递值类型，例如，使用`inout`把值类型函数地址传递给函数参数，但函数参数仍然是值类型。
- 引用类型 reference type：实例之间引用同一份数据。`class`、`function`是引用类型的数据。

> 尽管class实例、function是引用类型的，但传递给函数参数时，如果没有添加`inout`，则依然是值传递（pass by value），不是引用传递（pass by reference）。
>
> 值类型、引用类型与值传递、引用传递是不同的。value type可以进行值传递，也可以引用传递；引用类型可以进行值传递、也可以进行引用传递。
>
> Swift中的传递默认为值传递，进行了copy，即：传递value时，得到了value的拷贝；传递reference时，得到了引用的拷贝，拷贝后的引用与原始引用指向同一实例。Swift对拷贝进行了很多优化，只有在修改值时才进行拷贝。参数默认是不可变的，意味着很多copy不会发生。

下面探讨值类型和引用类型的区别，以及如何选择。

#### 1.1 值类型特点

当赋值、初始化、传参时，value type 会拷贝一份数据，实例之间拥有不同的数据。

```
struct Resolution {
    var width = 0
    var height = 0
}

let hd = Resolution(width: 1920, height: 1080)
var cinema = hd
cinema.width = 2048

print("cinema is now \(cinema.width) pixels wide")
print("hd is still \(hd.width) pixels wide")
```

打印结果为：

```
cinema is now 2048 pixels wide
hd is still 1920 pixels wide
```

显然，值类型更为安全，`let`意味着该实例不可改变。值类型的安全是通过copy一份值来实现的吗？这是否会让传递值变得昂贵起来？

可以说是，也可以说不是。传递值的性能没有想象中的那么差。显示、隐式的`let`已经确保了不可变，也就无需进行copy。即使传递了`var`引用，也不意味着实例会被拷贝。只有在必要时，才会进行拷贝，如修改实例。

#### 1.2 引用类型特点

引用类型赋值时会隐式共享同一实例，即两个指针指向同一个实例。其有以下特点：

- 当赋值、传递给函数参数时，可能多个引用指向同一对象。
- 实例本身是可变的，即使使用`let`引用了实例。
- 修改实例后，影响所有指向该实例的引用。

```
class VideoMode {
    var resolution = Resolution()
    var interlaced = false
    var frameRate = 0.0
    var name: String?
}

let tenEighty = VideoMode()
tenEighty.resolution = hd
tenEighty.name = "1080i"
tenEighty.frameRate = 25.0

let alsoTenEighty = tenEighty
alsoTenEighty.frameRate = 30.0

print("The frameRate property of tenEighty is now \(tenEighty.frameRate)")
```

打印结果为：

```
The frameRate property of tenEighty is now 30.0
```

#### 1.3 引用类型的潜在问题

值类型更容易查找出实例如何被修改的。持有值类型副本无需担心被其它地方的代码修改，特别是多线程的环境，代码更容易调试。

当实例没有可改变数据时，值类型和引用类型是一样的。

你可能会想如果提供一个不可变类型的class，使用`NSObject`将变得容易，同时又拥有了值语义的好处。在Swift中，可以通过在类中只使用不可变的属性，不暴露任何可修改状态的API实现。事实上，许多常见的Cocoa类都被设计为不可变的类，如`NSURL`。然而Swift语言目前没有提供任何机制，像`struct`、`enum`一样禁止类的可变。

## 2. 如何选择

创建新类型实例时，如何选择其类型？以下是选取建议：

- 默认使用structure。
- 当需要与Objective-C互操作时，使用class。
- 需要比较数据是否相同（===）时，使用class。
- 使用结构体和协议来构建继承和共享。

#### 2.1 默认使用structure

优先选择使用structure。Swift中的structure支持了在其它语言中仅限于类可使用的特性，如：存储属性、计算属性、方法。Swift中的structure还可以遵守协议，获得默认实现。Swift标准库和Foundation中很多常见类型都是structure，如：number、string、array、dictionary。

使用structure可以更容易推理代码，无需考虑整个代码执行流程。因为structure是值类型，对其修改不会影响其它部分的值，除非显式同步修改。

#### 2.2 需要与Objective-C互操作时，使用class

如果需使用Objective-C API处理数据，或需要将数据存储到Objective-C Class数据结构中，应使用 class。例如，Objective-C 提供的一些抽象类，必须使用类进行继承。

#### 2.3 需要比较是否相同（===）时，使用class

Swift提供了两种比较是否相等的方式，`==`和`===`。

`==`比较是否相等，相等的标准由实例来实现。例如，`4 == 4`为true，即`==`比较的数值是否相等。系统提供的其它类型也实现了相等比较，如String、bool、double等。

自定义的class和struct默认不能使用`==`比较，需遵守`Equatable`协议才可以使用。

`===`比较实例内存地址是否相同。当两个实例使用相同数据创建，使用`==`比较会认为相同，但`===`会认为是不同的。`===`只可以用来比较class，值类型的struct每个实例都是不同的。

当需要比较是否为同一对象（内存地址相同）使用类。例如，file handle、网络连接。例如，有一个本地数据库的连接，管理数据库访问的代码需要完全控制数据库状态，此类场景适合使用class，但需控制app哪些部分可访问数据库状态。

#### 2.4 使用结构体和协议来构建继承关系

structure和class都支持某种形式的继承，structure和protocol只能遵守protocol，不能继承自class。然而，使用class构建的继承层级结构，也可以使用protocol实现。

如果你正在从零开始构建继承层级，优先选择protocol继承。protocol允许class、structure、enumeration参与继承体系，而class的继承只允许其他class参与。当选择如何对数据进行建模时，应优先考虑使用协议继承来构建数据模型的层级结构，然后在structure中遵守这些协议。

参考资料：

1. [Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing_between_structures_and_classes)
2. [Value and Reference Types](https://developer.apple.com/swift/blog/?id=10)
3. [Is Swift Pass By Value or Pass By Reference](https://stackoverflow.com/questions/27364117/is-swift-pass-by-value-or-pass-by-reference)
