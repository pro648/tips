在使用手机的过程中，会产生很多交互事件，如触摸屏幕、摇晃、按下按键、使用耳机操控设备等。这些事件都需要系统去响应并作出处理。这篇文章将介绍系统是如何响应、处理这些事件的。

## 1. UIEvent

`UIEvent`描述用户与 app 的单次交互。

应用可以接收不同类型的*事件（event）*，包括触摸事件（touch events）、运动事件（motion events）、远程控制事件（remote-control events）和按下物理按键事件（press events）。触摸事件是最常见的事件，发送给触摸的视图。运动事件由 UIKit 触发，并与 Core Motion framework 运动事件进行区分。远程控制事件允许 responder 对象接收外部配件的指令（如耳机），以便管理音视频播放。Press events 代表与 game controller、AppleTV remote 或其他有物理按键设备的交互。可以使用`type`和`subtype`属性判断事件类型。

Touch event 对象包含与事件相关的 touch。Touch event 对象包含一个或多个 touch，每个 touch 都由`UITouch`对象表示。当发生触摸事件时，系统将事件路由至合适的响应者，并调用`touchesBegan(_:with:)`方法，responder 提取 touch 中的数据，并作出适当响应。

在多点触摸序列中，向 app 传递更新的触摸数据时，UIKit 会复用`UIEvent`对象。因此，不要持有`UIEvent`对象及其中的数据。如果需要在响应者方法之外使用`UIEvent`、`UITouch`数据，应在响应者方法中处理数据，并复制到自定义的数据结构中。

## 2. UITouch

`UITouch`对象表示屏幕上 touch 的位置、大小、移动和压力。

通过传递给 responder 的`UIEvent`来获取`UITouch`对象，`UITouch`提供以下信息：

- touch 发生的 view 或 window。
- touch 位于 view 或 window 的位置。
- touch 大致半径。
- touch 压力大小（支持 3D Touch 或 Apple Pencil 的设备）。

此外，touch 对象使用`timestamp`属性表示 touch 发生的时间，整数类型的`tapCount`属性表示点击屏幕的次数，`UITouch.Phase`属性表示处于began、moved、ended、canceled等阶段，

Touch 对象会在整个多点触控序列中存在。当处理多点触控序列时，可以引用 touch 对象，直到触控结束才释放。如果需要在多点触控序列外使用，需复制 touch 中的数据到自定义数据结构。

`gestureRecognizers`属性包含了当前处理 touch 的手势。

## 3. UIResponder

`UIResponder`类抽象了响应和处理事件的接口。

*响应者（responder）对象*（即`UIResponder`的实例）构成了事件处理的骨干。很多对象都继承自`UIResponder`，如`UIApplication`、`UIViewController`、`UIView`(`UIWindow`继承自`UIView`，进而也继承自`UIResponder`)。事件发生时，UIKit调度事件给 responder 处理。

想要处理特定类型事件，responder 必须重写对应方法。例如，想要处理 touch event，responder 需重写以下方法：

- touchesBegan(_:with:)
- touchesMoved(_:with:)
- touchesEnded(_:with:)
- touchesCancelled(_:with:)

在触摸事件中，responder 使用参数中的 event 信息跟踪 touch 变化，并更新 UI。

除了处理事件，responder 还可以转发未处理的事件。如果指定 responder 未处理事件，它会将事件转发给响应链（responder chain）中的 next responder。UIKit 根据规则，动态管理响应链。

Responder 对象除了处理`UIEvent`，还可以通过`inputView`接收自定义输入，系统的键盘就是一种 input view。用户点击屏幕上的`UITextField`或`UITextView`时，它成为 first responder 并显示 input view，默认展示系统键盘。可以创建自定义 input view 赋值给`inputView`属性，当其成为第一响应者时展示。

## 4. 事件传递

当 iPhone 接收到一个事件时，处理过程如下：

1. 通过动作触发事件，唤醒处于睡眠状态的app。

2. 使用 IOKit.framework 将事件封装为 IOHIDEvent 对象。

   > IOKit.framework 是一个系统框架的集合，用来驱动一些系统事件。IOHIDEvent 中的 HID 代表 Human Interface Device。

3. 系统通过 mach port 将 IOHIDEvent 对象转发给 SpringBoard.app。

4. SpringBoard.app 是 iOS 系统桌面 app，只接收按键、触摸、加速、接近传感器等几种 event。SpringBoard.app 找到可以响应这个事件的 app，并通过 mach port 将 IOHIDEvent 对象转发给 app 。

5. app 主线程 RunLoop 接收到 SpringBoard.app 转发的消息，触发对应 mach port 的 source1 回调 __IOHIDEventSystemClientQueueCallback()。

6. Source1 回调内部触发 Source0 回调 __UIApplicationHandleEventQueue()。

7. Source0 回调内部，将 IOHIDEvent 对象转化为`UIEvent`。

8. Source0 回调内部调用 UIApplication 的`sendEvent(_:)`方法，将`UIEvent`发给`UIWindow`。

`UIWindow`接收到事件后，开始传递事件。

## 5. 第一响应者 First Responder

下图显示了包含`UILabel`、`UITextField`、`UIButton`、`UIView`、`UIViewController`、`UIWindow`等视图的事件传递。

![ResponderChain](images/ResponderChain.png)

如果文本框没有处理 event，UIKit 转发事件给文本框的父视图`UIView`，随后是控制器的根视图、视图控制器、window。如果 window 也没有处理 event，UIKit 转发 event 至 UIApplication。如果 app delegate 是`UIResponder`子类，且未处于 responder chain，则也可能转发给 app delegate。

UIKit 根据事件类型指定第一响应者，事件类型如下：

| 事件类型     | 第一响应者              |
| ------------ | ----------------------- |
| 触摸事件     | 触摸的视图              |
| 按压事件     | 焦点对象                |
| 晃动事件     | 开发者或UIKit指定的对象 |
| 远程控制事件 | 开发者或UIKit指定的对象 |
| 编辑按钮信息 | 开发者或UIKit指定的对象 |

> 与加速计、陀螺仪、磁力仪相关的运动事件，不遵守 responder chain，Core Motion 直接将事件发送给指定对象。

触摸事件传递大致分为以下三个阶段： 

1. 寻找触摸对象 hit-testing。
2. 响应手势 recognize gesture。
3. 触摸事件传递 response chain。

#### 5.1 Hit-testing

Hit-testing 是查找 touch point 是否位于指定视图上的过程。iOS 使用 hit-testing 查找触摸事件最前的视图（即视图数组中 index 最大的视图），hit-testing 使用逆序深度优先遍历算法（reverse pre-order depth-first traversal algorithm）查找视图。

在介绍 hit-testing 如何工作前，先看下从手指触摸屏幕到抬起的单次触摸流程：

![hit-test-touch-event-flow](images/ResponderChainhit-test-touch-event-flow.png)

如上图所示，每次触摸屏幕的时候都会调用 hit-testing，并且是在视图、gesture recognizer 收到`UIEvent`之前。

Hit-testing 结束后，触摸位置下最前端视图被选为 first responder，它被关联到`UITouch`对象，并且 touch event 的所有阶段都会关联此视图。除了 hit test 视图，添加到 hit test 视图、父视图上的手势，都会关联到`UITouch`对象。最后，hit test 视图开始接受触摸系列事件。

> 即使手指已经滑动到 hit test view 边界之外，到了另一个视图上，hit test view 也会继续接收`UITouch`，直到这次 touch event 序列结束。

像前面说到的，hit-testing 使用逆序深度优先遍历算法（先查找到根视图，然后从高 index 开始遍历子视图）。子视图永远渲染在父视图之上；子视图数组中，兄弟视图（sibling view）中高 index 对象更大概率渲染在低 index 对象之上，多个视图覆盖在一起时，数组中高 index 对象更大概率是最前端视图。因此，逆序深度优先遍历算法可以减少遍历次数。

> 子视图会覆盖部分或全部父视图。父视图使用数组存储子视图，在数组中顺序决定子视图的可见性。如果两个子视图覆盖在一起，后添加子视图覆盖在先添加的子视图之上。

下图显示了视图和对应视图层级树，视图树从左至右反映了视图添加到父视图的顺序。

![hit-test-view-hierachy](images/ResponderChainhit-test-view-hierarchy.png)

如上图所示，View A 和 View B，及其子视图 View A.2 和 View B.1覆盖在一起，但由于 View B 的 index 大于 View A，View B 和它的子视图渲染到了 View A 和它的子视图上面。因此，当触摸重叠的 View B.1 区域时，hit-testing 返回 View B.1 视图。

逆序深度优先遍历算法查找过程如下：

![hit-test-depth-first-traversal](images/ResponderChainhit-test-depth-first-traversal.png)

`UIWindow`是视图层级结构的根视图，查找时先向`UIWindow`发送`hitTest(_:with:)`消息，该方法会返回包含触摸位置的视图。

下图显示了查找逻辑：

![hit-test-flowchart](images/ResponderChainhit-test-flowchart.png)

下面代码演示了`hitTest(_:event:)`方法可能的实现方式：

```
    func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        // 只有允许用户交互、没有隐藏、可见度大于 0.01 时，才允许接收手势。
        guard isUserInteractionEnabled, !isHidden, alpha > 0.01 else { return nil }
        
        // 点击point不在视图中时，直接返回 nil。
        guard self.point(inside: point, with: event) else {
            return nil
        }
        
        // 逆序遍历子视图
        for subView in subviews.reversed() {
            let convertedPoint = subView.convert(point, from: self)
            let hitTestView = subView.hitTest(convertedPoint, with: event)
            
            // 首个非空子视图即为 first responder
            if let hitView = hitTestView {
                return hitView
            }
        }
        
        // 如果所有子视图都没有响应 hit-testing，则视图本身为 first responder。
        return self
    }
```

使用 runtime 的 method swizzling 交换系统的`hitTest(_:event:)`方法，每次调用时输出当前类，查找到 first responder 时输出 first responder。如下所示：

```
extension UIView{
    public class func initializeMethod() {
        
        // hit-testing
        let originalHitSelector = #selector(UIView.hitTest(_:with:))
        let swizzleHitSelector = #selector(UIView.pr_hitTest(_:with:))
        
        guard let originalHitMethod = class_getInstanceMethod(self, originalHitSelector) else { return }
        guard let swizzleHitMethod = class_getInstanceMethod(self, swizzleHitSelector) else { return }
        
        let didAddHitMethod = class_addMethod(self, originalHitSelector, method_getImplementation(swizzleHitMethod), method_getTypeEncoding(swizzleHitMethod))
        if didAddHitMethod {
            class_replaceMethod(self, swizzleHitSelector, method_getImplementation(originalHitMethod), method_getTypeEncoding(swizzleHitMethod))
        } else {
            method_exchangeImplementations(originalHitMethod, swizzleHitMethod)
        }
        
        // pointInside
        let originalInsideSelector = #selector(UIView.point(inside:with:))
        let swizzleInsideSelector = #selector(UIView.pr_point(inside:with:))
        
        guard let originalInsideMethod = class_getInstanceMethod(self, originalInsideSelector) else { return }
        guard let swizzleInsideMethod = class_getInstanceMethod(self, swizzleInsideSelector) else { return }
        
        let didAddInsideMethod = class_addMethod(self, originalInsideSelector, method_getImplementation(swizzleInsideMethod), method_getTypeEncoding(swizzleInsideMethod))
        if didAddInsideMethod {
            class_replaceMethod(self, swizzleInsideSelector, method_getImplementation(originalInsideMethod), method_getTypeEncoding(swizzleInsideMethod))
        } else {
            method_exchangeImplementations(originalInsideMethod, swizzleInsideMethod)
        }
    }
    
    // 交换系统的hitTest(_:event:)方法，hit-Testing 时可以查看查找顺序。
    @objc public func pr_hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        print(NSStringFromClass(type(of: self)) + "  " + #function)
        let result = pr_hitTest(point, with: event)
        if result != nil {
            print((NSStringFromClass(type(of: self))) + " hitTesting return:" + NSStringFromClass(type(of: result!)))
        }
        
        return result
    }
    
    @objc public func pr_point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        print(NSStringFromClass(type(of: self)) + " --- pointInside")
        let result = pr_point(inside: point, with: event)
        print(NSStringFromClass(type(of: self)) + " pointInside +++ return: \(result)")
        return result
    }
}
```

> 如果你对 runtime 还不了解，可以查看我的文章：[Runtime从入门到进阶一](https://github.com/pro648/tips/blob/master/sources/Runtime%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E8%BF%9B%E9%98%B6%E4%B8%80.md)和[Runtime从入门到进阶二](https://github.com/pro648/tips/blob/master/sources/Runtime%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E8%BF%9B%E9%98%B6%E4%BA%8C.md)。

点击 ViewB1:

![MethodSwizzling](images/ResponderChainMethodSwizzling.png)





输出如下：

```
UITextEffectsWindow  pr_hitTest(_:with:)
UITextEffectsWindow --- pointInside
UITextEffectsWindow pointInside +++ return: true
UIInputSetContainerView  pr_hitTest(_:with:)
UIEditingOverlayGestureView --- pointInside
UIEditingOverlayGestureView pointInside +++ return: true
UIEditingOverlayGestureView  pr_hitTest(_:with:)
UIEditingOverlayGestureView --- pointInside
UIEditingOverlayGestureView pointInside +++ return: true
UIEditingOverlayGestureView hitTesting return:UIEditingOverlayGestureView
UIInputSetHostView  pr_hitTest(_:with:)
UIInputSetContainerView hitTesting return:UIInputSetContainerView
UITextEffectsWindow hitTesting return:UITextEffectsWindow

// hit-testing 时会调用point inside。
UIWindow  pr_hitTest(_:with:)
UIWindow --- pointInside
UIWindow pointInside +++ return: true

UITransitionView  pr_hitTest(_:with:)
UITransitionView --- pointInside
UITransitionView pointInside +++ return: true
UIDropShadowView  pr_hitTest(_:with:)
UIDropShadowView --- pointInside
UIDropShadowView pointInside +++ return: true
UILayoutContainerView  pr_hitTest(_:with:)
UILayoutContainerView --- pointInside
UILayoutContainerView pointInside +++ return: true
UINavigationBar  pr_hitTest(_:with:)
UINavigationBar --- pointInside
UINavigationBar pointInside +++ return: false
UINavigationTransitionView  pr_hitTest(_:with:)
UINavigationTransitionView --- pointInside
UINavigationTransitionView pointInside +++ return: true
UIViewControllerWrapperView  pr_hitTest(_:with:)
UIViewControllerWrapperView --- pointInside
UIViewControllerWrapperView pointInside +++ return: true

// UIView、ViewC、ViewB、ViewB2、ViewB1依次调用 hitTest。
UIView  pr_hitTest(_:with:)
UIView --- pointInside
UIView pointInside +++ return: true
ResponderChain.ViewC  pr_hitTest(_:with:)
ResponderChain.ViewC --- pointInside
ResponderChain.ViewC pointInside +++ return: false
ResponderChain.ViewB  pr_hitTest(_:with:)
ResponderChain.ViewB --- pointInside
ResponderChain.ViewB pointInside +++ return: true
ResponderChain.ViewB2  pr_hitTest(_:with:)
ResponderChain.ViewB2 --- pointInside
ResponderChain.ViewB2 pointInside +++ return: false
ResponderChain.ViewB1  pr_hitTest(_:with:)
ResponderChain.ViewB1 --- pointInside
ResponderChain.ViewB1 pointInside +++ return: true
ResponderChain.ViewB1 hitTesting return:ResponderChain.ViewB1
ResponderChain.ViewB hitTesting return:ResponderChain.ViewB1
UIView hitTesting return:ResponderChain.ViewB1
UIViewControllerWrapperView hitTesting return:ResponderChain.ViewB1
UINavigationTransitionView hitTesting return:ResponderChain.ViewB1
UILayoutContainerView hitTesting return:ResponderChain.ViewB1
UIDropShadowView hitTesting return:ResponderChain.ViewB1
UITransitionView hitTesting return:ResponderChain.ViewB1

// 再次执行 hit-Testing
UIWindow hitTesting return:ResponderChain.ViewB1
UIWindow  pr_hitTest(_:with:)
UIWindow --- pointInside
UIWindow pointInside +++ return: true
UITransitionView  pr_hitTest(_:with:)
UITransitionView --- pointInside
UITransitionView pointInside +++ return: true
UIDropShadowView  pr_hitTest(_:with:)
UIDropShadowView --- pointInside
UIDropShadowView pointInside +++ return: true
UILayoutContainerView  pr_hitTest(_:with:)
UILayoutContainerView --- pointInside
UILayoutContainerView pointInside +++ return: true
UINavigationBar  pr_hitTest(_:with:)
UINavigationBar --- pointInside
UINavigationBar pointInside +++ return: false
UINavigationTransitionView  pr_hitTest(_:with:)
UINavigationTransitionView --- pointInside
UINavigationTransitionView pointInside +++ return: true
UIViewControllerWrapperView  pr_hitTest(_:with:)
UIViewControllerWrapperView --- pointInside
UIViewControllerWrapperView pointInside +++ return: true
UIView  pr_hitTest(_:with:)
UIView --- pointInside
UIView pointInside +++ return: true
ResponderChain.ViewC  pr_hitTest(_:with:)
...
```

从日志中可以看到，首先是`UIWindow`开始调用 hitTest，然后是导航控制器视图、根视图，之后是ViewC，ViewC返回 false后，开始遍历ViewB，ViewB返回 ture 后，先遍历 ViewB2，ViewB2 返回 false 后才遍历 ViewB1，最终返回 ViewB1。

> 可以看到，一次点击会调用两次`hitTest(_:event:)`，这是因为系统在调用 hit test 时，可能微调点击位置，`hitTest(_:event:)`只是单纯的函数，没有其它副作用。

## 6. 用途

通过重写`hitTest(_:event:)`、`pointInside(point:event:)`方法，可以将事件转发给其它视图处理。

> 由于先执行`hitTest(_:event:)`、后发送 event，重写`hitTest(_:event:)`、`pointInside(point:event:)`方法后，转发的事件是完整事件，即包含`UITouch.Phase.began`、`UITouch.Phase.moved`、`UITouch.Phase.ended`等所有阶段的事件。

#### 6.1 扩大可点击区域

一个常见用途就是扩大可点击区域，使它大于视图`bounds`。下图的按钮大小为 20*20，不太方便点击，可以使用`pointInside(point:event:)`扩大其可点击区域。

![hit-test-increase-touch-area](images/ResponderChainhit-test-increase-touch-area.png)

为了使其更具有通用性，为`UIButton`增加分类，在分类方法中实现改变可点击区域的功能，这样后续想要使用时，只需调用分类方法即可。

下面使用runtime为分类添加属性的方式，实现扩大`UIButton`可点击区域的功能：

```
// 修改 UIButton 可点击区域

struct AssociateKeys {
    static var topKey: UInt8 = 0
    static var leftKey: UInt8 = 1
    static var bottomKey: UInt8 = 2
    static var rightKey: UInt8 = 3
}

protocol HitTestSlopProtocol {
    func expand(edgeInset hitTestSlop: UIEdgeInsets)
}

extension UIButton: HitTestSlopProtocol {
    func expand(edgeInset hitTestSlop: UIEdgeInsets) {
        objc_setAssociatedObject(self, &AssociateKeys.topKey, hitTestSlop.top, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        objc_setAssociatedObject(self, &AssociateKeys.leftKey, hitTestSlop.left, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        objc_setAssociatedObject(self, &AssociateKeys.bottomKey, hitTestSlop.bottom, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        objc_setAssociatedObject(self, &AssociateKeys.rightKey, hitTestSlop.right, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
    }

    private func slop() -> UIEdgeInsets {
        guard let topValue = objc_getAssociatedObject(self, &AssociateKeys.topKey) as? CGFloat,
              let leftValue = objc_getAssociatedObject(self, &AssociateKeys.leftKey) as? CGFloat,
              let bottomValue = objc_getAssociatedObject(self, &AssociateKeys.bottomKey) as? CGFloat,
              let rightValue = objc_getAssociatedObject(self, &AssociateKeys.rightKey) as? CGFloat else { return .zero}
        
        return UIEdgeInsets(top: topValue, left: leftValue, bottom: bottomValue, right: rightValue)
    }
    
    open override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        let insets = slop()
        
        if insets == .zero {
            // Safer to use UIView's point(inside:with:) if we can.
            return super.point(inside: point, with: event)
        } else {
            return bounds.inset(by: insets).contains(point)
        }
    }
}
```

后续想要改变可点击区域时，只需调用`expand(edgeInset:)`方法即可：

```
        button.expand(edgeInset: UIEdgeInsets(top: -30, left: -30, bottom: -30, right: -30))
```

点击 button 之外，也可以触发 button 事件，如下所示：

![TouchArea](images/ResponderChainTouchArea.gif)

> 通过重写`hitTest(_:event:)`，可以实现 tabbar 中部突出部分响应手势的需求。与上面实现有些类似，但不必写成分类。

#### 6.2 事件集中处理

假设视图控制器中有一个 table view，cell 上有两个按钮 firstButton、secondButton，点击按钮、cell本身都会触发事件。以前我们一般直接处理事件，或使用delegate、closure等回调给视图控制器处理，现在我们可以使用 nextResponder 将所有响应都传递到控制器处理，这样代码逻辑会更清晰，业务逻辑也变得更简单。

```
extension UIResponder {
    
    /// 将事件转发给下一 responder
    /// - Parameters:
    ///   - event: 事件名称
    ///   - userInfo: 事件附带的额外信息
    @objc func routerEvent(with event: String, userInfo: [String:String]) {
        print(NSStringFromClass(type(of: self)) + "  " + #function)
        
        self.next?.routerEvent(with: event, userInfo: userInfo)
    }
}

    // Cell 按钮的点击事件
    @objc private func firstButtonTapped(_: UIButton) {
        print(NSStringFromClass(type(of: self)) + "  " + #function)
        
        // 路由给控制器处理
        routerEvent(with: "firstButton", userInfo: [:])
    }
    
    @objc private func secondButtonTapped(_: UIButton) {
        print(NSStringFromClass(type(of: self)) + "  " + #function)
        
        routerEvent(with: "secondButton", userInfo: [:])
    }
    
extension RouterEventVC {
    // 统一处理所有事件
    override func routerEvent(with event: String, userInfo: [String : String]) {
        if event == "firstButton" {
            print("firstButton Clicked")
        } else if event == "secondButton" {
            print("secondButton Clicked")
        } else {
            print("Something else Clicked")
        }
    }
}
```

这样就可以将点击firstButton、secondButton的响应方法集中在 RouterEventVC 中处理。点击 firstButton 时，触发`routerEvent(with: "firstButton", userInfo: [:])`方法，此时将事件转发给`UITableViewCell`；由于 cell 没有处理事件，cell 将事件转发给`UITableView`处理；由于`UITableView`没有处理事件，table view 将事件转发给 RouterEventVC 的根视图`UIView`；由于`UIView`没有处理事件，它将事件转发给 RouterEventVC，RouterEventVC 已经处理了事件，不再进行转发。最后，也就由视图控制器统一处理。

## 总结

事件响应链和传递链完全相反。最有机会处理事件的就是通过事件传递找到的 first responder；如果 first responder 没有进行处理，就会沿着事件响应链传递给*下一个响应者 next responder*，一直追溯到最上层的 UIApplication。若都没有进行处理，就丢弃事件。

Demo名称：ResponderChain  
源码地址：<https://github.com/pro648/BasicDemos-iOS/tree/master/ResponderChain>

参考资料：

1. [Hit-Testing in iOS](http://smnh.me/hit-testing-in-ios/)

2. [Using Responders and the Responder Chain to Handle Events](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events)

3. [hitTest:withEvent: called twice?](https://lists.apple.com/archives/cocoa-dev/2014/Feb/msg00118.html)

4. [Stored Properties In Swift Extensions](https://marcosantadev.com/stored-properties-swift-extensions/)

5. [iOS 中的事件响应与处理](https://blog.boolchow.com/2018/03/25/iOS-Event-Response/)

6. [事件响应机制](https://yeziahehe.com/2020/01/19/responder_chain/)

7. [iOS事件传递及响应链](https://limeng99.club/learning/2019/12/30/iOS%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92%E5%8F%8A%E5%93%8D%E5%BA%94%E9%93%BE.html)

8. [iOS响应链](https://xiaozhuanlan.com/topic/8796435021)

   