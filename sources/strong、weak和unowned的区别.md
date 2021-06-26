编写代码时需注意是否产生了循环引用，因此就产生了什么时候使用`weak`、`unowned`问题？这篇文章将介绍 Swift 中的`strong`、`weak`、`unowned`的区别。

## 1. ARC

自动引用计数（即 Automated Reference Count，简称 ARC）是 Xcode 4.2版本的新特性，其与手动管理内存使用了相同的计数系统。不同点在于：系统在编译时会帮助我们插入合适的内存管理方法，保留和释放都会自动进行，避免了手动管理引用计数的一些潜在问题。

Swift 使用自动引用计数跟踪、管理app的内存。通常情况下，这意味着ARC会自动管理内存，开发者无需关注内存管理。当类的实例不再使用时，ARC会自动释放其占用的内存。

为帮助管理内存，ARC 有时需了解类之间的关系。在 Swift 中使用 ARC 与在 Objective-C 中使用 ARC 类似。

引用计数只适用类的实例。结构体和枚举是值类型，不是引用类型，存储和传递的时候并非使用引用。

## 2. strong

`strong`指针通过增加指向对象的引用计数，保护被指向对象不被ARC释放。即，只要有一个强指针指向该对象，它就不会被释放。

Swift 中声明的属性默认是`strong`。当对象间引用关系是线性时，使用`strong`指针不会产生问题。

当两个实例使用强指针指向彼此时，两个实例引用计数都不会变为零，即产生循环引用（strong reference cycle）。

下面是一个循环引用的示例：

```
class Person {
    let name: String
    init(name: String) {
        self.name = name
    }
    
    var apartment: Apartment?
    deinit {
        print("\(name) is being deinitizlized")
    }
}

class Apartment {
    let unit: String
    init(unit: String) {
        self.unit = unit
    }
    
    var tenant: Person?
    deinit {
        print("Apartment \(unit) is being deinitialized")
    }
}
```

上面定义了两个类：`Person`和`Apartment`，代表住户和公寓。两个类都实现了`deinitializer`方法，当类的实例销毁时进行打印，方便观察实例占用的内存是否释放了。

下面代码定义了两个可选类型的变量，初始值为nil。并为其分配两个新的实例：

```
var john: Person?
var unit4A: Apartment?

john = Person(name: "John Appleseed")
unit4A = Apartment(unit: "4A")
```

目前，`john`变量强引用`Person`实例，`unit4A`变量强引用`Apartment`实例，如下所示：

![ReferenceCycle](images/21/UnownedReferenceCycle01.png)

现在连接两个实例。`person`持有`apartment`，`apartment`持有`person`。

```
john?.apartment = unit4A
unit4A?.tenant = john
```

连接两个实例后，引用关系如下：

![ReferenceCycle](images/21/UnownedReferenceCycle02.png)

`Person`的实例强引用了`Apartment`的实例，`Apartment`的实例强引用了`Person`的实例，即产生了循环引用。当移除`john`、`unit4A`的引用时，实例的引用计数不会变为零，也就是实例内存不会被ARC释放。

```
john = nil
unit4A = nil
```

设置`john`、`unit4A`变量为`nil`后，引用关系如下：

![ReferenceCycle](images/21/UnownedReferenceCycle03.png)

`Person`和`Apartment`实例之间的强引用无法破除。

## 3. 解决循环引用

Swift 提供了两种解决循环引用的方案：`weak`和`unowned`。`weak`和`unowned`引用其它实例时不会产生强引用，引用计数不会加一。因此，不会产生循环引用。

当一个实例的生命周期短于另一个时（即一个实例可以先被销毁），使用`weak`引用。在上面公寓的示例中可能出现公寓没有住户的情况。因此，可以使用`weak`解决循环引用问题。当另一个实例生命周期与当前实例相同，或长于当前实例时，使用`unowned`引用。

#### 3.1 weak引用

`weak`引用不会强持有引用的实例，也就不会阻止ARC释放实例。通过在声明属性、变量前添加`weak`关键字的方式使用弱引用。

当实例被销毁时，ARC 会自动设置弱指针为`nil`。由于弱指针在运行时可能被设置为`nil`，弱指针应被声明为可选类型的变量，而非常量。

和其它可选类型一样，可以检查弱引用值是否存在，这样就不会得到一个无效实例。

> 设置弱引用为`nil`时，不会调用属性观察器。

使用`weak`修饰之前实例`Apartment`中的`tenant`属性，更新后如下：

```
class Person {
    let name: String
    init(name: String) {
        self.name = name
    }
    
    var apartment: Apartment?
    deinit {
        print("\(name) is being deinitizlized")
    }
}

class Apartment {
    let unit: String
    init(unit: String) {
        self.unit = unit
    }
    
    // 使用weak修饰
    weak var tenant: Person?
    deinit {
        print("Apartment \(unit) is being deinitialized")
    }
}
```

下图是实例间引用关系：

![WeakReference](images/21/UnownedWeakReference01.png)

`Person`实例强引用`Apartment`实例，但`Apartment`实例没有强引用`Person`实例。当移除`john`实例对`Person`的强引用，`Person`实例就没有被强引用了，也就可以被销毁了。

#### 3.2 unowned

与`weak`一样，`unowned`指针也不会对指向的对象产生强引用，但`unowned`用在另一个实例生命周期一样或更长的情况。通过在声明属性、变量前添加`unowned`关键字的方式使用`unowned`。

与`weak`不同，`unowned`修饰的引用永远不为空。因此，标记为`unowned`的值不是可选类型，ARC 也不会将`unowned`引用设置为`nil`。

> 只有确信引用不会被释放的时候才使用`unowned`，使用`unowned`修饰的对象被销毁后再次访问会产生运行时错误。

现在定义两个类：`Customer`和`CreditCard`。`Customer`是银行的客户，`CreditCard`是该客户的银行卡。`Customer`和`CreditCard`类都有一个属性持有彼此，这种持有关系会产生强引用。

`Customer`和`CreditCard`的关系与`Person`和`Apartment`的关系稍有不同。`Customer`可能持有`CreditCard`，也可能不持有`CreditCard`；但`CreditCard`不会脱离`Customer`而存在。即`Customer`有一个可选类型的`card`属性，`CreditCard`有一个 unowned 的`customer`属性。

```
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) is being deinitialized")
    }
}

class CreditCard {
    let number: UInt64
    unowned let customer: Customer
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    
    deinit {
        print("Card #\(number) is being deinitialized")
    }
}
```

下面创建`Customer`实例，并使用该实例创建`CreditCard`，如下所示：

```
var john: Customer?
john = Customer(name: "John Appleseed")
john!.card = CreditCard(number: 1234_5678_9012_3456, customer: john!)
```

其引用关系如下：

![unowned](images/21/UnownedUnownedReference01.png)

`Customer`实例强引用`CreditCard`实例，`CreditCard`实例 unowned `Customer`实例。

当`john`变量取消对实例的强引用后，就没有强引用指向该实例，该实例就会被销毁。该实例销毁后，没有强引用指向`CreditCard`，其也会被销毁。

> 上面的示例介绍了如何使用 safe unowned 引用，Swift 同时提供了 unsafe unowned 引用，其可以避免 runtime 的安全检查，提高性能。使用 unsafe 相关操作时，开发者需自行检查其是否存在，确保安全。
>
> 使用`unowned(unsafe)`标记 unsafe unowned 引用。当实例销毁后，再次访问实例会直接访问销毁前的内存地址。

参考资料：

1. [What is the difference in Swift between 'unowned(safe)' and 'unowned(unsafe)'?](https://stackoverflow.com/questions/26553924/what-is-the-difference-in-swift-between-unownedsafe-and-unownedunsafe)
2. [Automatic Reference Counting](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)
