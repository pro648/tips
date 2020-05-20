`iOS`编程中，定义属性中的特性有atomic、nonatomic、copy、assign、strong、weak等，一般格式如下：

````
@property (nonatomic, strong) NSString *name; 
````
下面详细解释一下他们的区别。
#### atomic
- 默认属性。
- 当前进程进行到一半，其他线程来访问当前线程，可以保证先执行完毕当前线程。
- 只是保证setter/getter 完整，不是线程安全。

> 例如：对于atomic的对象，线程A调用getter，同时线程B、线程C都调用setter并设不同的值，最后线程A可能得到原来的值、也可能得到线程B、线程C设的值，只能够保证得到的是原始值、B线程设定的值、C线程设定的值三者中一个完整值，没有办法确定最终得到的是哪个值。如果线程D调用release，程序会崩溃。所以atomic只是read/write安全，不是thread安全。

Runtime [objc-accessors.mm](https://opensource.apple.com/source/objc4/objc4-781/runtime/objc-accessors.mm.auto.html)文件包含了属性设值、取值源码，如下所示：

```
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}

static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```

可以看到，`atomic`修饰的设值、取值方法使用了自旋锁，确保线程同步。

虽然设值、取值方法是原子操作，但不代表是线程安全。例如，可变数组使用`atomic`修饰，只是取数组、设置数组是原子操作，但操作数组元素不再是线程安全。

#### nonatomic 
- 非默认属性。
- 两个线程同时访问同一个属性将会导致无法预计的结果。
- 优点是程序运行速度快。

#### copy 

- 是owner，不是reference（引用）。当对象可变时，可设置为copy，用于获取此时值的副本。
- 新的对象引用计数为1，与原始对象引用计数无关，且原始对象引用计数不会改变。
- 使用copy创建的新对象也是强引用，使用完成后需要负责释放该对象。
- copy特性可以减少对象对上下文的依赖。新对象、原始对象中任一对象的值改变不会影响另一对象的值。
- 要想设置该对象的特性为copy，该对象必须遵守`NSCopying`协议，Foundation类默认实现了`NSCopying`协议，所以只需要为自定义的类实现该协议即可。更全面了解`NSCopying`协议，查看 [深复制、浅复制、copy、mutableCopy](https://github.com/pro648/tips/wiki/%E6%B7%B1%E5%A4%8D%E5%88%B6%E3%80%81%E6%B5%85%E5%A4%8D%E5%88%B6%E3%80%81copy%E3%80%81mutableCopy) 一文。

#### assign
- 与copy相反，只是reference，不是owner。只返回指针。
- 用于float、int、BOOL等类型。
- 释放后再发送消息会导致程序崩溃。

#### strong

- 默认属性
- strong = retain iOS引入ARC后，用strong替代了retain。
- 创建一个强引用的指针，引用对象引用计数加1。
- strong特性表示两个对象内存地址相同（建立一个指针，进行指针拷贝），内容会一直保持相同，直到更改一方内存地址，或将其设置为`nil`。
- 如果有多个对象同时引用一个属性，任一对象对该属性的修改都会影响其他对象获取的值。
- 所有实例变量、局部变量默认都是strong。


#### weak
- 只是reference，不是owner。即引用计数不会加1。
- 可将weak对象设为nil，向nil发送消息，什么都不会执行，程序也不会崩溃。
- 代理使用weak。delegate几乎一直own代理对象，所以代理对象应该对代理使用weak，否则会形成循环引用（retain cycle）。但也有例外，如果代理对象的生命周期比代理短，代理对象也可以使用strong。
- IBOutlet常用weak。

>关于strong和weak对比的一个形象[例子](http://stackoverflow.com/questions/9262535/explanation-of-strong-and-weak-storage-in-ios5/9262768#9262768)：
>
> 假设对象是一条小狗，小狗想跑走（be deallocated)。
>
>strong类型就像是拴狗的绳子，只要有一条绳子栓住狗，它就不能跑走，如果有五条绳子拴着同一条狗（五个strong类型指向同一个对象），只有当五条绳子都释放狗才可以跑走。
>
>weak类型就像是小孩子看着小狗说：看这里有小狗。只要还有绳子拴着小狗，小孩子们就可以继续指着小狗说：看这里有小狗。当绳子释放了的时候，不管有多少小孩子依旧在指着小狗说：看这里的小狗。小狗都会跑掉。
>
>当最后一个strong指针不再指向这个对象，这个对象就会被释放，此时，所有指向这个对象的weak指针都将被清空。

#### readonly
- 非默认属性
- 只有可读方法，也就是只有getter方法。

#### readwrite

- 默认属性
- 如果希望一个属性只允许自己读写，而对所有外部文件都是只读的，可以在接口部分声明该属性为readonly类型，最后在私有接口部分重写该属性为readwrite类型。


