在app的生命周期中，有时需要保存、读取一些临时数据。根据需求不同，可能保存到磁盘上，也可能只存储在内存中。Apple提供的`NSCache`类提供了使用key-value将对象缓存到内存的方案。

`NSCache`有以下优点：

- 只在内存存储数据。如果app被杀死，占用的内存会被释放，不会持久化到磁盘。
- key-value对机制与`Dictionary`非常像，可以很方便的读取、保存数据。与`Dictionary`不同的是key不会被拷贝，更为高效。
- 可以设置自动删除缓存的策略，也可以主动驱逐缓存中对象。
- 线程安全。

只有下面一点不够完美之处：

- `NSCache`是Objective-C API，Key和Value都必须是Class类型。Swift中的`String`是struct类型，ObjC中的`NSString`是Class类型，在Swift中使用`NSCache`时，key不能时String类型。

`NSCache`用来缓存创建成本比较高，但在需要时可以重新创建的对象。例如，向用户展示的图片。下载、解码图片耗费时间和流量，再次展示同一张图片时如果再下载一次体验会很差，这时可以用缓存存储图片。内存告急时清理掉缓存中图片，再次展示时通过网络下载图片。

## 1. 创建NSCache对象

`NSCache`的构建方法接收两个范型对象，一个是key，一个是value。

```
let cache = NSCache<NSString, UIImage>()
cache.name = "ImageCache"
```

`NSCache`是Objective-C API，范型类型必须继承自`AnyObject`，即不能使用Swift的struct，必须使用class。因此，也不能使用String类型，需将其转换为NSString。这个示例中，key为`NSString`类型，value为`UIImage`类型。

## 2. 存储对象

将对象添加到缓存时，只需调用`setObject（_:forKey:)`方法：

```
        guard let img = UIImage(named: "pro648") else { return }
        cache.setObject(img, forKey: "imgKey")
```

`setObject(_:forKey:cost:)`方法也可以添加对象到缓存，其额外增加了key value占用缓存大小参数。

`cost`参数用来计算整体缓存大小。当内存不足，或缓存大小到达阀值，`NSCache`会开始驱逐程序，移除缓存中的一些对象，但移除的对象是无序的。因此，如果你想通过操控`cost`值实现特定行为，最终结果会与预期不符。`cost`是value占用字节大小。如果不了解value对象字节大小，则应直接使用`setObject（_:forKey:)`方法，或给`cost`参数传递0。与`NSMutableDictionary`不同的是，`NSCache`不会拷贝key。

## 3. 读取缓存数据

使用`object(forKey:)`读取缓存中数据，返回值类型是`ObjectType`。如果缓存中没有存储过、或已经被移除了，该方法会返回`nil`。

```
        guard let image = cache.object(forKey: "imgKey") else {
            print("image went away")
            return
        }
        
        print("The image is still cached")
```

## 4. 移除缓存数据

删除缓存中数据不会增加缓存实现复杂度。清除单个对象使用`removeObject(forKey:)`方法：

```
cache.removeObject(forKey: "imgKey")
```

移除所有缓存对象使用`removeAllObjects()`方法：

```
cache.removeAllObjects()
```

## 5. 自动清理缓存

手动清理缓存是非常重要的。手动清理能够满足大多case，但`NSCache`提供了根据设定的条件自动清理缓存。可以设置缓存对象数量和缓存大小。

即使没有设置任何清理缓存的条件，内存不足时`NSCache`也会清理缓存对象。因此，缓存中数据要能够再次创建，不应对其生命周期做出假设。

#### 5.1 设置缓存对象数量

给`countLimit`属性设置大于0的值就可以限制缓存对象数量，零为不限制数量。

```
cache.countLimit = 10
```

根据缓存数据类型决定缓存数量大小。如果缓存的是大图，可以设置较低的缓存数量；如果缓存的是文本等比较小的对象，则可以设置较高的数量。

`countLimit`并不是严格限制。清除缓存由`NSCache`来实现，其根据设备当前可用缓存决定何时清除缓存。因此，超过缓存数量，`NSCache`可能立即清理缓存，也可能稍后清理，也可能不清理缓存。

#### 5.2 设定最大Cost

缓存对象`cost`的定义是抽象的，因缓存类型而异。

回到上面缓存图片的示例中。缓存图片时`cost`可以是图片字节大小，图片越大占用内存越多。`cost`大小也可以是图片尺寸，即宽、高。

缓存字符串时，`cost`可以是字符数量。例如`pro648`包含六个字符，`github.com/pro648`包含17个字符。

给`totalCostLimit`属性赋值设置最大缓存。图片缓存设置50Mb：

```
cache.totalCostLimit = 50_000_000
```

与`countLimit`类似，当到达最大缓存值时是否开始清理缓存与系统当前状态有关。

## 6. NSDiscardableContent

当对象有子组件，当子组件不使用时可以先释放，这一点可以通过遵守`NSDiscardableContent`协议来实现。

假设有以下`Person`类：

```
class Person {
    let firstName: String
    let lastName: String
    var avatar: UIImage? = nil
    
    init(firstName: String, lastName: String, avatar: UIImage?) {
        self.firstName = firstName
        self.lastName = lastName
        self.avatar = avatar
    }
}
```

缓存`Person`对象时，`firstName`和`lastName`属性占用内存可以忽略不计，`avatar`头像可能很大。当系统清理缓存时，我们希望只清理图片。即`Person`是要缓存的对象，`avatar`是可以被清理的子组件。

`NSDiscardableContent`与变量协同。当图片内存正在被读取或使用，计数为1；未被使用时，计数为0；创建`NSDiscardableContent`时，默认值为1。

如下所示：

```
class Person {
	...
    
    // Our counter variable
    var accessCounter = true
}

extension Person: NSDiscardableContent {
    // 开始读取内容时调用
    func beginContentAccess() -> Bool {
        if avatar != nil {
            accessCounter = true
        } else {
            accessCounter = false
        }
        return accessCounter
    }
    
    // 不再需要内容时调用
    func endContentAccess() {
        accessCounter = false
    }
    
    // 如果内容不再使用，释放掉。
    func discardContentIfPossible() {
        avatar = nil
    }
    
    // 如果已经释放了，返回true。
    func isContentDiscarded() -> Bool {
        return avatar == nil
    }
}
```

`NSCache`的`evictsObjectsWithDiscardedContent`属性决定清除整个缓存对象，还是只清除子组件。默认情况下，`NSCache`会清除整个缓存对象，而非缓存对象的子组件。将`evictsObjectsWithDiscardedContent`设置为`false`开启只清除子组件。

```
personCache.evictsObjectsWithDiscardedContent = false
```

## 7. NSCacheDelegate协议

`NSCacheDelegate`协议只有一个方法`cache(_:willEvictObject)`，释放对象时会调用该方法。

```
extension DiscardableViewController: NSCacheDelegate {
    func cache(_ cache: NSCache<AnyObject, AnyObject>, willEvictObject obj: Any) {
        guard let person = obj as? Person else {
            return
        }
        
        print("willEvictObject: \(person.firstName)")
    }
}
```

对象从缓存中移除前会`cache(_:willEvictObject)`方法，可以在该方法中进行一些操作。

## 总结

`NSCache`用来在内存中缓存内容。既可以手动管理缓存，也可以设置条件自动释放内存。

`NSCache`是Objective-C API，key、value都必须时class类型。Struct和enum类型不能进行缓存，如果需要在`NSCache`中缓存，则可以再封装一下`NSCache`，具体可以查看这篇文章：[Caching in Swift](https://www.swiftbysundell.com/articles/caching-in-swift/)

Demo名称：NSCache  
源码地址：https://github.com/pro648/BasicDemos-iOS/tree/master/NSCache

参考资料：

1. [Caching Content With NSCache](https://www.andyibanez.com/posts/caching-content-with-nscache/)
2. [Caching with NSCache in iOS](https://blog.devgenius.io/caching-with-nscache-in-ios-2e97be8d6b53)
3. [NSCache](https://nshipster.com/nscache/)
4. [Caching in Swift](https://www.swiftbysundell.com/articles/caching-in-swift/)
