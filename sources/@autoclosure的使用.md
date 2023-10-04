Swift的闭包（Closure）与ObjC中的block很像，但Swift闭包语法更简洁，也新增了许多新特性。本文是我学习@autoclosure的记录。

## 1. 什么是@autoclosure

当前是morning时，下述方法会打印出*Good morning*欢迎语：

``` 
func goodMorning(morning: Bool, whom: String) {
    if morning {
        print("Good Morning, \(whom)")
    }
}

goodMorning(morning: true, whom: "Pavel")
goodMorning(morning: false, whom: "John")
```

打印内容为：

```
Good Morning, Pavel
```

对于John，第一个参数为false，不会打印欢迎语。

下面新增一个欢迎提示：

```
func giveAname() -> String {
    print("giveAname() is called")
    return "Robert"
}

goodMorning(morning: true, whom: giveAname())
```

这一次`goodMorning(morning:whom:)`方法的第二个参数是另一个函数`giveAname()`，`giveAname`函数返回值类型为String。

运行后输出如下：

```
giveAname() is called
Good Morning, Robert
```

`goodMorning(morning:whom:)`第一个参数传false，如下所示：

```
func giveAname() -> String {
    print("giveAname() is called")
    return "Robert"
}

goodMorning(morning: false, whom: giveAname())
```

控制台打印如下：

```
giveAname() is called
```

当`goodMorning(morning:whom:)`第一个参数为false时，第二个参数不会被使用，但`giveAname()`函数仍然进行了调用。如果`giveAname()`函数不希望被调用，可以使用闭包（closure）：

```
func goodMorning(morning: Bool, whom: () -> String) {
    if morning {
        print("Good morning, \(whom())")
    }
}

func giveAname() -> String {
    print("giveAname() is called")
    return "Robert"
}

goodMorning(morning: false, whom: giveAname)
```

现在，闭包作为第二个参数，`giveAname`不会在`goodMorning(morning:whom:)`前调用，且只有在用到时才会调用。

## 2. 为什么要有@autocolsure

现在第二个参数是闭包类型，不接受String常量和变量。`@autoclosure`可以解决这一问题。

自动闭包（autoclosure）是自动创建的闭包，用于包装作为参数传递给函数的表达式，其不接受参数，调用时返回表达式的值。autoclosure的语法便利性让开发者直接编写普通表达式即可，而非显式闭包，也可以省略参数周围的大括号。

使用自动闭包后如下：

```
func goodMorning(morning: Bool, whom: @autoclosure () -> String) {
    if morning {
        print("Good morning, \(whom())")
    }
}

func giveAname() -> String {
    print("giveAname() is called")
    return "Robert"
}

goodMorning(morning: true, whom: giveAname())
goodMorning(morning: false, whom: giveAname())
goodMorning(morning: true, whom: "Pavel")
goodMorning(morning: false, whom: "Jhon")
```

控制台打印如下：

```
giveAname() is called
Good morning, Robert
Good morning, Pavel
```

`goodMorning(morning:whom:)`的第二个参数是自动闭包，传入的函数、变量、常量会被自动转换为闭包，且只有在调用到的时候才执行。

## 3. 什么时候使用@autoclosure

通常，调用函数参数可能是自动闭包类型，开发者编写方法不太常设置为自动闭包类型。例如，`assert(condition:message:file:line:)`的condition、message参数都是auto closure，condition只在playground、非优化（即非release模式）模式生效，message只在condition为false时触发。

```
/// Performs a traditional C-style assert with an optional message.
///
/// Use this function for internal sanity checks that are active during testing
/// but do not impact performance of shipping code. To check for invalid usage
/// in Release builds, see `precondition(_:_:file:line:)`.
///
/// * In playgrounds and `-Onone` builds (the default for Xcode's Debug
///   configuration): If `condition` evaluates to `false`, stop program
///   execution in a debuggable state after printing `message`.
///
/// * In `-O` builds (the default for Xcode's Release configuration),
///   `condition` is not evaluated, and there are no effects.
///
/// * In `-Ounchecked` builds, `condition` is not evaluated, but the optimizer
///   may assume that it *always* evaluates to `true`. Failure to satisfy that
///   assumption is a serious programming error.
///
/// - Parameters:
///   - condition: The condition to test. `condition` is only evaluated in
///     playgrounds and `-Onone` builds.
///   - message: A string to print if `condition` is evaluated to `false`. The
///     default is an empty string.
///   - file: The file name to print with `message` if the assertion fails. The
///     default is the file where `assert(_:_:file:line:)` is called.
///   - line: The line number to print along with `message` if the assertion
///     fails. The default is the line number where `assert(_:_:file:line:)`
///     is called.
public func assert(_ condition: @autoclosure () -> Bool, _ message: @autoclosure () -> String = String(), file: StaticString = #file, line: UInt = #line)
```

autoclosure可以延迟执行，只有在调用时才执行。对于有副作用或比较昂贵的操作，延迟执行会很有用。

> 滥用autoclosure可能导致代码难以阅读，上下文环境或函数名应标记出延迟执行。

## 总结

函数参数可以使用返回值类型相同的无参闭包替换。这种类型的闭包可以使用`@autoclosure`标记。使用`@autoclosure`标记后，闭包前后不需要添加大括号，自动闭包会自动添加大括号。使用`@autoclosure`标记后，传入的函数不会立即执行，会延迟到函数体内调用才执行。

Demo名称：autoclosure  
源码地址：<https://github.com/pro648/BasicDemos-iOS/tree/master/Autoclosure>

参考资料：

1. [@autoclosure what, why and when](https://medium.com/ios-os-x-development/https-medium-com-pavelgnatyuk-autoclosure-what-why-and-when-swift-641dba585ece)
2. [Autoclosures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/#Autoclosures)

