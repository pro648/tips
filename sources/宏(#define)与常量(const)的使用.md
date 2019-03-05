iOS开发中经常用到`#define`进行文本替换，`const`修饰数据类型。下面说一下他们的使用细节。
#### #define宏
- 一般`#define`放在程序开始，在`#import`之后，也可以放在其他任何位置，但是必须先定义后引用。
- 预定义的名称和变量的行为模式不同，没有局部定义之类的说法。在一个方法内定义，就可以在这个方法之后的任何位置使用。
- 定义如果需要两行，那么上一行的最后一个字符必须是反斜线符号`\`，用以告诉预处理程序这里存在一个后续。
- 预定义名称不是变量，因此不能为他赋值，除非替换指定值的结果实际上是一个变量。为了和变量进行区分，一般定义名称都是大小字母组合。
- 宏是进行完整字符替换，如果不是特别确定需要分号，不要在定义语句末尾添加分号。
- 没有类型，不进行类型检查。
- 对同一个宏在多个地方进行引用，每个引用都会开辟一块内存。

#### const常量
- 只分配一次内存空间。项目中用到多次，也不会多次分配内存空间。
- 编译时会进行类型检查，及时提醒错误。

另外说一下`NSString const *var` 和 `NSString * const var`的区别。经常遇到的有

``` 
const NSString *var	
NSString const *var // 与上一个意义相同
NSString * const var 
```
`const NSString *var`和`NSString const *var`没有区别。因为`NSString`本身就是不可变的，此处的`const`没有任何用途。`const`右侧的为定义的常量，无法修改。

1. 下面的`var`不能做任何修改。

```
static NSString * const var;       // a
static NSString const * const var; // 与a相同，第一个const没有任何用途
static const NSString * const var; // 与a相同，第一个const没有任何用途
```

2. 下面的`var`对象的值不能修改，但可以修改指针指向。

```
static NSString * var;              // b
static NSString const * var;        // 与b相同，const没有任何用途
static const NSString * var;        // 与b相同，const没有任何用途
```

3. 下面的`var`对象的值可以改变，指针指向不能改变。

```
static NSMutableString * const var; // c
```

4. 下面的`pro648`对象的值和指针都可以改变。

```
static NSMutableString * pro648; // d
```

下面代码代表的是`pro648`指针指向地址被改变了，即开始时指向`@"a"`，后来指向`@"b"`，而不是字符串`@"a"`改变了。

```
pro648 = @"a";
pro648 = @"b";
```

> 事实上不能修改`NSString`对象的内容，`NSMutableString`对象可以使用`appendString:`方法修改。

如果声明`pro648`变量方法是：

```
NSString const *pro648
```

最后`pro648`会指向`@"b"`。如果声明`pro648`变量方法是：

```
NSString * const pro648
```

当尝试为`pro648`再次赋值时，即修改其指针指向时，编译器会发出*Can not assign to variable 'var' with const-qualified type 'NSString * const-strong'*的警告。

参考资料：

1. [What is a better practice when defining a const NSString object in objective-c](https://stackoverflow.com/questions/30000641/what-is-a-better-practice-when-defining-a-const-nsstring-object-in-objective-c)

2. [#define vs const in Objective-C](https://stackoverflow.com/questions/11153156/define-vs-const-in-objective-c)
