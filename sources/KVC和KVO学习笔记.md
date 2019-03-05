在编程中，最常见的就是程序的流程取决于你所使用的各种变量和属性的值，根据变量和属性的值确定后面运行的代码，有时会检查对象是否已加入数组，或是否已被移除，因此，获取类中属性的变化是编程中重要部分。

我们有多种方式获取对象的改变，如[委托、通知](https://github.com/pro648/tips/wiki/%E5%A7%94%E6%89%98%E3%80%81%E9%80%9A%E7%9F%A5%E4%BC%A0%E5%80%BC%E7%9A%84%E7%94%A8%E6%B3%95%E4%B8%8E%E5%8C%BA%E5%88%AB)等。如果需要观察多个属性的变化，为避免产生大量的代码，最好是使用键值观察（Key Value Observing，简称KVO），这也是Apple在自己的软件中大量使用的一种。

使用键值观察跟踪单个属性或集合（如数组）的变化非常高效，它只需要在观察者方法中添加代码，不需要修改被观察文件内的代码，这一点和委托、通知不同。但需要注意的是，键值观察（KVO）是建立在键值编码（Key Value Coding，简称KVC）的基础上，也就是说任何你想使用KVO观察的属性必须符合键值编码。

KVC和KVO提供了一个强大高效的方式来编写代码，学习KVO前必须先掌握KVC，所以下面我们结合demo来学习KVC。在这个demo中所有结果将直接在控制台输出，没有创建用户界面。

## 1. 创建应用

启动Xcode，点击File > New > Project…，选择iOS > Application > Single View Application模板，点击*Next*；在*Product Name*一栏填写`KVC&KVODemo`，点击*Next*；选择文件位置，点击*Create*创建工程。

## 2. 键值编码

假设我们有一个`NSString`类型的`firstName`的属性，我们想把`Donald`赋值给属性，我们可以使用下面两种方式之一。

```
self.firstName = @"Donald";	    // 1
_firstName = @"Donald";			// 2
```

上面的代码我们非常熟悉，1直接为属性赋值，2直接给实例变量赋值。如果使用KVC设值，代码如下：

```
[self setValue:@"Donald" forKey:@"firstName"];
```

你会发现使用KVC和词典中设值或将标量值和结构值转换为`NSValue`非常类似。再举一例，下面代码3使用设值方法设值，4使用KVC模式设值。

```
[someObject.someProperty setText:@"This is a text"];                       // 3
[self setValue:@"This is a text" forKey:@"someObject.someProperty.text"];  // 4
```

在第一个示例中，我们用KVC替代直接赋值，在第二个示例中，我们用KVC替代访问器方法设值。使用KVC时我们只需要将值与`Key`或`KeyPath`匹配就可以，使用字符串间接把值赋给属性。如果需要获取属性的值，可用下面方式：

```
NSLog(@"%@",[self valueForKey:@"firstName"]);
```

键值编码机制是由一个`NSKeyValueCoding`非正式协议定义的，`NSObject`实现了这个协议，所以我们继承`NSObject`才能让我们的类获得KVC能力。理论上，如果你的类遵守`NSKeyValueCoding`协议，也可以自己实现KVC的细节，这样做完全行得通，但这样太不值得了，也太占用时间了。

打开Xcode，点击File  >  New  >  File…，或使用快捷键(⌘+N)创建一个类。在弹出窗口中，选择iOS > Source > Cocoa Touch Class模板，点击*Next*；类名称为`Children`，父类为`NSObject`，点击*Next*；选择文件位置，点击*Create*创建文件。

进入`Children.h`文件，添加两个属性，一个是`firstName`，一个是`age`，我们将使用这两个属性展示KVC的主要特性。更新后代码如下：

```
@interface Children : NSObject

@property (nonatomic, strong) NSString *firstName;
@property (nonatomic, assign) NSUInteger age;

@end
```

进入`Children.m`文件初始化上面两个属性。

```
@implementation Children

- (instancetype)init
{
    self = [super init];
    if (self)
    {
        _firstName = @"";
        _age = 0;
    }
    
    return self;
}

@end
```

进入`ViewController.m`文件，导入`Children.h`，声明两个`Children`类型的属性。代码如下：

```
#import "ViewController.h"
#import "Children.h"

@interface ViewController ()

@property (nonatomic, strong) Children *child1;
@property (nonatomic, strong) Children *child2;

@end
```

在`viewDidLoad`方法中，初始化`child1`对象，使用KVC方法先设值、后取值并输出到控制台。

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    // child1
    self.child1 = [Children new];
    
    // 1.使用KVC设值
    [self.child1 setValue:@"Jr" forKey:@"firstName"];
    [self.child1 setValue:[NSNumber numberWithUnsignedInteger:39] forKey:@"age"];
    
    // 2. 取值 输出到控制台
    NSString *childFirstName = [self.child1 valueForKey:@"firstName"];
    NSUInteger child1Age = [[self.child1 valueForKey:@"age"] unsignedIntegerValue];
    NSLog(@"%@,%lu",childFirstName,child1Age);
}
```

在1中使用`setValue: forKey: `为属性设值。需要注意的是`age`是数字，因此不能直接作为参数，需要转换为`NSNumber`类型，另外键（Key）的字符串必须和属性中的名称一致，否则运行时app会崩溃，弹出*Terminating app due to uncaught exception 'NSUnknownKeyException',*提示。在2中，使用`valueForKey: `取值，输出到控制台。

```
Jr,39
```

目前为止，我们已经学习了如何编写符合KVC的代码，如何使用KVC设值和取值，以及`Key`写错会如何。现在开始学习一下如何使用`KeyPath`。首先进入`Children.h`文件，添加一个`Children`类型的属性。

```
@interface Children : NSObject

···
@property (nonatomic, strong) Children *child;

@end
```

返回到`ViewController.m`，在`viewDidLoad`方法中，初始化`child2`并设值，最后初始化`child`属性。

```
- (void)viewDidLoad
{
    ···
    // child2
    self.child2 = [Children new];
    [self.child2 setValue:@"Ivanka" forKey:@"firstName"];
    [self.child2 setValue:[NSNumber numberWithUnsignedInteger:35] forKey:@"age"];
    self.child2.child = [Children new];    
}
```

现在使用`setValue: forKeyPath: `为`child`属性设值，这里的键是一个使用点语法的字符串`@"child.firstName"`。

```
- (void)viewDidLoad
{
    ...
    [self.child2 setValue:@"Eric" forKeyPath:@"child.firstName"];
    [self.child2 setValue:[NSNumber numberWithUnsignedInteger:33] forKeyPath:@"child.age"];
    
    NSLog(@"%@,%lu",self.child2.child.firstName, self.child2.child.age);
}
```

最后使用`NSLog`测试设值是否成功。输出是：

```
Eric,33
```

> `valueForKey:`和`valueForKeyPath:`是在`NSKeyValueCoding`非正式协议中定义的方法，两者默认由根类`NSObject`实现，是KVC框架的一部分。
>
> `objectForKey:`是由`NSDictionary`提供提取对应键的值的方法。
>
> 虽然在词典中使用`valueForKey:`也可以提取到值，但当`key`字符串以`@`开头时会遇到问题。所以在词典中使用`objectForKey:`，在KVC中使用`valueForKey:`和`valueForKeyPath:`。

## 3. 键值观察

我们已经掌握了KVC，现在开始学习KVO。以下是实现KVO的步骤：

1. 使用`addObserver: forKeyPath: options: context: `方法注册为观察者，用于观察其他类的属性。
2. 观察者必须实现`observerValueForKeyPath: ofObject: change: context: `方法以接收属性变化通知。
3. 使用`removeObserver: forKeyPath: context: `方法移除观察者。

### 3.1 观察单个属性

进入`ViewController.m`，在实现部分添加`viewWillAppear:`方法，在`viewWillAppear:`方法内添加`firstName`和`age`属性为观察对象。

```
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    
    [self.child1 addObserver:self forKeyPath:@"firstName" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:NULL];
    [self.child1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:NULL];
}
```

上面添加观察者方法中各参数含义如下：

- `addObserver:` 注册成为观察者，以便接收KVO通知，通常为`self`，该对象必须实现`observerValueForKeyPath: ofObject: change: context: `以接收属性变化通知。
- `keyPath:`要观察的属性字符串，必须和属性一致，不能为空。
- `options:`用来指定通知词典中应包含值类型。如果参数是`NSKeyValueObservingOptionNew`，词典包含新产生的值；如果参数是`NSKeyValueObservingOptionOld`，词典包含变化前的值；如果参数是数字`0`，词典不包括任何值；如果需要`change`词典同时包括新产生值和变化前的旧值，可以像上面代码一样使用` |`，即或运算符；任何时候都可以使用`[object valueForKey:<Key>]`方法获取属性变化产生的新值。
- `context:`这是一个指针，可用做我们观察到的属性更改的唯一标志符，经常设置为`NULL`，后面会详细说明。

现在我们已经可以观察到`firstName`和`age`两个属性的的变化，KVO观察到每一个观察对象的变化都会调用`observerValueForKeyPath: ofObject: change: context: `方法，如果观察多个属性的变化，观察方法内`if`语句可能很长，下面实现`observerValueForKeyPath: ofObject: change: context: `方法。

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    if ([keyPath isEqualToString:@"firstName"])
    {
        NSLog(@"The name of the child was changed.\n %@",change);
    }
    else if ([keyPath isEqualToString:@"age"])
    {
        NSLog(@"The new value is %@,The old value is %@",[change valueForKey:NSKeyValueChangeNewKey],[change valueForKey:NSKeyValueChangeOldKey]);
    }
}
```

在上面代码中，我们根据参数`keyPath`判断哪一个属性改变了，之后输出。在输出时，可以直接输出词典`change`，也可以用`valueForKey:`获取词典中的值，此处的键用`NSKeyValueChangeNewKey`或`NSKeyValueChangeOldKey`，前者获取新产生的值，后者获取改变前的旧值。现在在`viewwillAppear: `底部添加下面代码来验证是否可以观察到属性变化。

```
- (void)viewWillAppear:(BOOL)animated
{
    ...
    // 添加观察者后改变值 验证是否可以观察到值变化
    [self.child1 setValue:@"Tiffany" forKey:@"firstName"];
    [self.child1 setValue:[NSNumber numberWithUnsignedInteger:23] forKey:@"age"];
}
```

运行，输出内容为：

```
The name of the child was changed.
 {
    kind = 1;
    new = Tiffany;
    old = Jr;
}

The new value is 23,The old value is 39
```

你可以从`change`词典中提取你需要的值，有了KVO观察属性变化变的如此简单。现在在`viewWillAppear: `方法中添加观察者，观察`child2`对象的属性变化，随后为`age`设值。

```
- (void)viewWillAppear:(BOOL)animated
{
    ...    
    // 观察child2属性变化 设值
    [self.child2 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:NULL];
    [self.child2 setValue:[NSNumber numberWithUnsignedInteger:64] forKey:@"age"];
}
```
运行后输出如下：

```
The name of the child was changed.
 {
    kind = 1;
    new = Tiffany;
    old = Jr;
}

The new value is 23,The old value is 39

The new value is 64,The old value is 35
```

正如看到的一样，我们收到两个`age`属性的改变通知，尽管我们自己可以区分出每一个通知来自哪一个对象属性的改变，但在程序中，目前我们无法对此进行区分。为解决这个问题，我们将使用`addObserver: forKeyPath: options: context: `方法中的`context`参数。`context`参数一般使用下面声明方法。

```
static void *XXContext = &XXContext;
```
表示一个静态变量存放着它自己的指针，也就是它自己什么也没有。因为要在`addObserver: forKeyPath: options: context: `和`observerValueForKeyPath: ofObject: change: context: `两个方法中使用`context`，这里的`context`声明为静态全局变量。

在`ViewController.m`实现前添加下面两个声明。

```
@end

static void *child1Context = &child1Context;
static void *child2Context = &child2Context;

@implementation ViewController
```

你也可以把`context`声明为属性，但声明为全局变量更为简单。现在修改`viewWillAppear:`方法中的添加观察者方法，将`context`参数中的`NULL`替换为刚声明的全局变量。

```
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    
    [self.child1 addObserver:self forKeyPath:@"firstName" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:child1Context];
    [self.child1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:child1Context];
    
    // 添加观察者后改变值 验证是否可以观察到值变化
    [self.child1 setValue:@"Tiffany" forKey:@"firstName"];
    [self.child1 setValue:[NSNumber numberWithUnsignedInteger:23] forKey:@"age"];
    
    // 观察child2属性变化 设值
    [self.child2 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:child2Context];
    [self.child2 setValue:[NSNumber numberWithUnsignedInteger:64] forKey:@"age"];
}
```

最后修改`observerValueForKeyPath: ofObject: change: context: `方法，以便区分出通知来自哪一个对象属性的变化。

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    ...
    // 使用context后
    if (context == child1Context)
    {
        if ([keyPath isEqualToString:@"firstName"])
        {
            NSLog(@"The name of the FIRST child was changed.\n %@",change);
        }
        else if ([keyPath isEqualToString:@"age"])
        {
            NSLog(@"The new value of the FIRST child is %@,The new value of the FIRST child is %@",[change valueForKey:NSKeyValueChangeNewKey],[change valueForKey:NSKeyValueChangeOldKey]);
        }
    }
    else if (context == child2Context)
    {
        if ([keyPath isEqualToString:@"age"])
        {
            NSLog(@"The new value of the SECOND child is %@,The new value of the SECOND child is %@",[change valueForKey:NSKeyValueChangeNewKey],[change valueForKey:NSKeyValueChangeOldKey]);
        }
    }
}
```

### 3.2 注册相互影响的键

在许多情况下，一个属性的值取决另一个对象中的一个或多个其他属性的值。如果一个属性的值改变，那么派生属性的值也应该改变。例如：姓名由姓和名两个属性组成，当其中任何一个属性变化时,姓名属性都要得到改变的通知。

进入`Children.h`，添加`NSString`类型的`fullName`属性和`lastName`两个属性。

```
@interface Children : NSObject

...
@property (nonatomic, strong) NSString *fullName;
@property (nonatomic, strong) NSString *lastName;

@end
```

进入`Children.m`，初始化刚声明的属性，`fullName`由`firstName`和`lastName`组成。

```
- (instancetype)init
{
    self = [super init];
    if (self)
    {
        ...
        _lastName = @"";
    }
    
    return self;
}

- (NSString *)fullName
{
    return [NSString stringWithFormat:@"%@ %@",self.firstName,self.lastName];
}
```

当`firstName`和`lastName`属性变化时，必须通知`fullName`属性的应用程序，因为它们会影响`fullName`属性的值。可以通过实现类方法`keyPathsForValuesAffecting<key>`来获取哪些属性会影响`<key>`属性，这里的`<key>`为`fullName`，首字母要大写。在`Children.m`中添加以下类方法：

```
+ (NSSet *)keyPathsForValuesAffectingFullName
{
    return [NSSet setWithObjects:@"firstName",@"lastName", nil];
}
```

进入`ViewController.m`文件，在`viewWillAppear:`方法中添加观察者，观察`fullname`属性，之后修改`lastName`和`firstName`属性。

```
- (void)viewWillAppear:(BOOL)animated
{
    ...
    // 添加观察者 观察fullName属性 修改firstName lastName
    [self.child1 addObserver:self forKeyPath:@"fullName" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:child1Context];
    self.child1.lastName = @"Trump";
    self.child1.firstName = @"Ivana";
}
```

在`observerValueForKeyPath: ofObject: change: context: `方法中，观察到`fullname`属性变化时进行输出。

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    ...
    // 使用context后
    if (context == child1Context)
    {
        ...
        else if ([keyPath isEqualToString:@"fullName"])
        {
            NSLog(@"The full name of First child was change.\n %@",change);
        }
    }
    ...
}
```

现在你可以运行下，可以输出`fullname`属性的变化。

```
The full name of First child was change.
 {
    kind = 1;
    new = "Tiffany Trump";
    old = "Tiffany ";
}

The full name of First child was change.
 {
    kind = 1;
    new = "Ivana Trump";
    old = "Tiffany Trump";
}

The name of the FIRST child was changed.
 {
    kind = 1;
    new = Ivana;
    old = Tiffany;
}
```


### 3.3 观察数组

`NSArray`是KVC和KVO中的一种特殊情况，想要观察到数组的变化需要做一些额外的工作。事实上，有很多关于数组的细节，但在这里，我们只讲解一些基础的、重要的内容。因为数组不符合KVC，因此观察数组不像观察上面示例中的属性那么简单。我们要实现一些关于数组的方法以便使数组符合KVC，进而可以使用KVO观察数组的变化。

这里我们将讨论可变数组，不可变数组与可变数组类似，只是需要实现的方法少一些。假设我们有一个可变数组`myArray`，这里需要实现的方法与数组的插入、移除、计数类似，不同之处在于数组的名称，需要实现方法如下：

- countOfMyArray
- objectInMyArrayAtIndex:
- insertObject:inMyArrayAtIndex:
- removeObjectFromMyArrayAtIndex:

这些方法都很熟悉，不同的是我们用数组的名称替换里面名称。如果是不可变数组，只需要取消实现最后两个方法。

让数组符合KVC有好的一方面，也有不利的一方面。好处是Xcode会对数组名建议补全；坏的一方面是类中每一个想使用KVC观察的数组都要实现这些方法，会产生大量代码。为了避免产生大量重复代码，我们可以创建一个新的类，类内只包含一个可变数组，在这个数组内实现这些方法让这个数组符合KVC，这样在其他类中使用这个类的实例对象。这样的好处是：让数组符合KVC，必须实现的方法只需要实现一次，这个类可以重复使用。你可以理解为这是一个高级版本的数组。

现在添加一个新类，点击File > New > File…，选取iOS > Source > Cocoa Touch Class模板，点击*Next*；类名称为`KVCMutableArray`，父类为`NSObject`，点击*Next*；选择文件位置，点击*Create*创建文件。

进入`KVCMutableArray.h`文件，声明一个可变数组及一些方法以便数组符合KVC。

```
@interface KVCMutableArray : NSObject

@property (nonatomic, strong) NSMutableArray *array;

- (NSUInteger)countOfArray;
- (id)objectInArrayAtIndex:(NSUInteger)index;
- (void)insertObject:(id)object inArrayAtIndex:(NSUInteger)index;
- (void)removeObjectFromArrayAtIndex:(NSUInteger)index;
- (void)replaceObjectInArrayAtIndex:(NSUInteger)index withObject:(id)object;

@end
```

在上面`insertObject: inArrayAtIndex: `方法中，`object`对象这里设定为`id`类型，以便其他类可以使用。进入`KVCMutableArray.m`，添加`init`方法，初始化`array`,实现头文件中声明的方法。

```
@implementation KVCMutableArray

- (instancetype)init
{
    self = [super init];
    if (self)
    {
        _array = [NSMutableArray new];
    }
    
    return self;
}

- (NSUInteger)countOfArray
{
    return self.array.count;
}

- (id)objectInArrayAtIndex:(NSUInteger)index
{
    return [self.array objectAtIndex:index];
}

- (void)insertObject:(id)object inArrayAtIndex:(NSUInteger)index
{
    [self.array insertObject:object atIndex:index];
}

- (void)removeObjectFromArrayAtIndex:(NSUInteger)index
{
    [self.array removeObjectAtIndex:index];
}

- (void)replaceObjectInArrayAtIndex:(NSUInteger)index withObject:(id)object
{
    [self.array replaceObjectAtIndex:index withObject:object];
}

@end
```

到目前我们已经创建了一个符合KVC的数组。

现在我们要在`Children.h`中添加一个可变数组，数组内包括姓名，用KVO观察数组内容的变化。在这里我们应该使用`KVCMutableArray`类型的属性，而不是系统提供的默认数组，进入`Children.h`，导入`KVCMutableArray.h`文件，添加新的属性。

```
#import <Foundation/Foundation.h>
#import "KVCMutableArray.h"

@interface Children : NSObject

...
@property (nonatomic, strong) KVCMutableArray *cousins;

@end
```

在`Children.m`中初始化刚声明的对象。

```
- (instancetype)init
{
    self = [super init];
    if (self)
    {
        ...
        _cousins = [KVCMutableArray new];
    }
    
    return self;
}
```

进入`ViewController.m`文件，在`viewWillAppear:`方法底部添加如下代码：

```
- (void)viewWillAppear:(BOOL)animated
{
    ...
    // 对数组进行观察
    [self.child1 addObserver:self forKeyPath:@"cousins.array" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:NULL];
    [self.child1.cousins insertObject:@"Antony" inArrayAtIndex:0];
    [self.child1.cousins insertObject:@"Julia" inArrayAtIndex:1];
    [self.child1.cousins replaceObjectInArrayAtIndex:0 withObject:@"Ben"];
}
```

在`observerValueForKeyPath: ofObject: change: context: `方法中处理接收到通知。

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    ...
    else if ([keyPath isEqualToString:@"cousins.array"] && [object isKindOfClass:[Children class]])
    {
        NSLog(@"cousins.array %@",change);
    }
    else
    {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}
```

这里除了使用`keyPath`进行判断还可以额外进行类判断，当父类也在观察属性时会有帮助。最后,如果没有合适的`context`或`keyPath`，把它交给父类来处理，可能是父类也在观察同一个属性。

运行app，输出结果证明观察数组成功。

```
cousins.array {
    indexes = "<_NSCachedIndexSet: 0x6000000394e0>[number of indexes: 1 (in 1 ranges), indexes: (0)]";
    kind = 2;
    new =     (
        Antony
    );
}

cousins.array {
    indexes = "<_NSCachedIndexSet: 0x600000039500>[number of indexes: 1 (in 1 ranges), indexes: (1)]";
    kind = 2;
    new =     (
        Julia
    );
}

cousins.array {
    indexes = "<_NSCachedIndexSet: 0x6000000394e0>[number of indexes: 1 (in 1 ranges), indexes: (0)]";
    kind = 4;
    new =     (
        Ben
    );
    old =     (
        Antony
    );
}
```

### 3.4 手动发送通知

默认情况下，KVO观察到属性变化系统会自动发送通知，但在某些情况下，你可能需要控制何时发送通知。例如：在某些情况下不需要发送通知，或将多个改变合并为一个通知发送。手动发送通知提供了执行此操作的方法。

手动和自动通知并不互斥，已经存在自动通知的类内也可以添加手动通知。你可以通过重写由`NSObject`实现的`automaticallyNotifiesObserversForKey: `类方法来控制特定属性的通知发送，这个方法的参数`key`就是想要手动控制通知的属性，这个方法返回值类型是`BOOL`类型，想要手动控制通知的属性在重写这个类方法时返回`NO`，其他属性由超类来处理。

假设我们现在不想接收`firstName`属性的变化，进入`Children.m`文件，在实现部分添加下面类方法。

```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    BOOL automatic = NO;
    if ([key isEqualToString:@"firstName"])
    {
        automatic = NO;
    } else {
        automatic = [super automaticallyNotifiesObserversForKey:key];
    }
    return automatic;
}
```

上面的方法非常简单，暂停`firstName`属性的自动通知；在`else`部分，使用超类调用相同方法，以便让iOS处理所有未在上面显式添加的属性。

现在运行app，你会发现所有`firstName`属性的变化都没有输出。现在只是能够停止特定键对应属性变化的通知，还不能称为手动发送通知。

想要手动发送通知，需要添加`willChangeValueForKey: `和`didChangeValueForKey: `方法。步骤如下：

1. 调用`willChangeValueForKey: `方法。
2. 修改所观察属性的值。
3. 调用`didChangeValueForKey: `方法。

进入`ViewController.m`文件，在`viewWillAppear: `方法中找到`[self.child1 setValue:@"Tiffany" forKey:@"firstName"];`这一行代码，并用下面三行代码替换。

```
- (void)viewWillAppear:(BOOL)animated
{
    ...
    // 添加观察者后改变值 验证是否可以观察到值变化
    [self.child1 willChangeValueForKey:@"firstName"];
    [self.child1 setValue:@"Tiffany" forKey:@"firstName"];
    [self.child1 didChangeValueForKey:@"firstName"];
    ...
}

```

现在运行app，`firstName`属性的改变会在控制台输出，也就是我们已经成功手动发送通知。

```
The name of the FIRST child was changed.
 {
    kind = 1;
    new = Tiffany;
    old = Jr;
}
```

事实上，通知是在调用`didChangeValueForKey: `方法后发送的。如果不想在改变属性值后立即发送通知，可以在改变属性后任何想要发送通知的位置调用`didChangeValueForKey: `方法。例如这个demo中，你可以把`[self.child1 didChangeValueForKey:@"firstName"];`放在程序行代码的最底部，控制台内容输出顺序将会发生变化，这里不再详细说明。

> 如果单个操作导致多个键改变，则必须嵌套更改通知。如下：
>
> ``` 
>   [self.child1 willChangeValueForKey:@"firstName"];
>   [self.child1 willChangeValueForKey:@"property"];
>   self.child1.firstName = @"First";       // 1 不能观察到
>   self.child1.firstName = @"Second";      // 2 可以观察到
>   self.child1.property = @"xxx";
>   [self.child1 didChangeValueForKey:@"firstName"];
>   [self.child1 didChangeValueForKey:@"property"];
> ```
>
> 可以把多个手动通知嵌套在一起，每个手动通知只能观察到键最新一次的改变。如上面代码，只有2可以观察到改变，1的改变不能观察到。

最后一定要记得移除观察者。如果视图控制器释放前没有移除观察者，释放时app会崩溃。一般添加观察者在`viewDidLoad`方法、`viewWillAppear:`中，移除观察者可以在`observerValueForKeyPath: ofObject: change: context: `处理完通知后，或`viewWillDisappear:`方法中，也可以在`dealloc`方法中。在这个demo中，我们在`viewWillDisappear:`移除观察者。

```
- (void)viewWillDisappear:(BOOL)animated
{
    [super viewWillDisappear:animated];
    
    // 移除所有观察者
    [self.child1 removeObserver:self forKeyPath:@"firstName" context:child1Context];
    [self.child1 removeObserver:self forKeyPath:@"age" context:child1Context];
    [self.child1 removeObserver:self forKeyPath:@"fullName" context:child1Context];
    [self.child1 removeObserver:self forKeyPath:@"cousins.array" context:NULL];
    [self.child2 removeObserver:self forKeyPath:@"age" context:child2Context];
}
```

每一个`addObserver: forKeyPath: options: context: `必须对应一个`removeObserver: forKeyPath: context: `，KVO没有办法判断当前控制器是否被注册为观察者，并且移除不存在的观察者，app也会崩溃。

## 总结

键值观察提供了一种允许对象在其他类属性变化时获得通知的机制。对于应用程序中模型层和控制器层通信特别有用。控制器对象通常用来观察模型对象的属性，并且视图对象也可以通过控制器对象观察模型对象的属性。此外，模型对象可以观察其他模型对象，也可以观察自身。

键值观察和键值编码都是一种帮助建立更强大、更灵活、更高效的应用，可能刚接触时觉得很奇特，最后你会感觉这些很容易掌握。

文件名称：KVC&KVODemo    
源码地址：<https://github.com/pro648/BasicDemos-iOS>

参考资料：

1. [Key-Value Observing Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)
2. [Understanding Key-Value Observing and Coding](http://www.appcoda.com/understanding-key-value-observing-coding/)
3. [KVO Considered Harmful](http://khanlou.com/2013/12/kvo-considered-harmful/)
