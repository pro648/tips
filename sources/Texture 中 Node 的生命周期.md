> 这是 Texture 文档系列翻译，其中结合了自己的理解和工作中的使用体会。如果哪里有误，希望指出。
>
> 1. [Texture 核心概念](https://github.com/pro648/tips/wiki/Texture%20%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5)
>
> 2. [Texture 布局 Layout](https://github.com/pro648/tips/wiki/Texture%20%E5%B8%83%E5%B1%80%20Layout)
>
> 3. [Texture 便捷方法](https://github.com/pro648/tips/wiki/Texture%20%E4%BE%BF%E6%8D%B7%E6%96%B9%E6%B3%95)
>
> 4. [Texture 性能优化](https://github.com/pro648/tips/wiki/Texture%20%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
>
> 5. [Texture 容器 Node Containers](https://github.com/pro648/tips/wiki/Texture%20%E5%AE%B9%E5%99%A8%20Node%20Containers)
>
> 6. [Texture 基本控件 Node](https://github.com/pro648/tips/wiki/Texture%20%E5%9F%BA%E6%9C%AC%E6%8E%A7%E4%BB%B6%20Node)
> 7. [Texture 中 Node 的生命周期](https://github.com/pro648/tips/wiki/Texture%20%E4%B8%AD%20Node%20%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)

Texture 使用 ARC（Automatic Reference Counting）管理内存。因此，一旦对象不被强引用，会尽快将其释放。当涉及到`ASDisplayNode`及其子类时，不同种类的 node 具有不同的生命周期。了解其生命周期、加载状态、何时进入页面有助于深入了解 Texture。

## 被 node container 管理的 node

Node container 管理其管理 node 的生命周期。通常，node container 根据需要尽快为 node 分配内存，并在不需要时释放他们。Texture 假定 node container 独自管理其 node，并且希望开发者不要引用或修改 node 的生命周期。例如，开发者不应保存`ASCollectionNode`、`ASTableNode`、`ASPagerNode`创建的`ASCellNode`，也不要复用`ASCellNode`。

一旦将`ASCellNode`添加到`ASCollectionNode`或`ASTableNode`，就会为其`ASCellNode`分配内存。例如，刷新数据、插入数据。`UICollectionView`和`UITableView`会对 cell 进行复用，但`ASCollectionNode`和`ASTableNode`不会对 cell 进行复用，因此`ASCellNode`数量和`ASCollectionNode`、`ASTableNode`当前 row/item 数一致。

目前，`ASCollectionNode`和`ASTableNode`插入数据时会立即分配内存。如果批量拉取了100条数据，就会在 batch update 时分配100个 cell node。也会计算每个 cell 的布局。最终，node container 会管理100个或更多 cell，并立即计算出每个 cell 的布局。

Node container 处理大量 cell 时，需要一定时间来处理上述操作。因此推荐使用 node block，以便 cell node 可以在后台线程并发创建，而非在主线程顺序创建。如果想要进一步提升性能，可以使用 batch fetching API 预拉取数据。

可以在`ASDataController`查看实现细节，`ASDataController`是唯一在任一时间均对所有 cell 进行强引用的类。

#### ASCollectionLayout

为了解决一次分配太多 cell 的问题，为`ASCollectionNode`增加了新 API，以便在用户滑动时初始化 cell、计算布局。但由于某些限制，该功能只能用于已知 cell 大小的布局。例如，相册、电子书等。

具体可以查看`ASCollectionLayout`、`ASCollectionGalleryLayoutDelegate`和`ASCollectionFlowLayoutDelegate`。

#### 释放 ASCellNode

由于`ASCellNode`不会进行复用，其生命周期会比`UICollectionViewCell`、`UITableViewCell`长。当`ASCellNode`不再使用，并从 container node 移除后会被释放。例如，下拉刷新、删除 cell，或 container node 被移除。

Container node 被移除后，其 cell node 并不会立即释放。这是因为 table node 和 collection node 可能持有大量 cell node，同时释放会造成明显卡顿。为避免这一问题，`ASCollectionNode`和`ASTableNode`释放其对象时（特别是`ASDataController`），借助`ASDeallocQueue`类在后台线程进行释放。由于`ASDataController`强引用了所有 cell node，所有 cell node 也会在后台线程释放。使用 Instruments 分析内存泄漏时，需要注意 cell node 延迟释放这一问题。

#### ASDeallocQueue

`ASDeallocQueue`强引用所持有对象，以便延迟释放，最后在后台线程进行释放。

## 未被 node container 管理的 node

将 node 添加到父视图后，直到从父视图移除或父视图释放才会释放 node。如果未从父视图移除，则 node 生命周期将于父视图一致。由于 node 一般长时间存在于视图层级中，整个 node 生命周期与 root node 一致。

#### ASM 管理下的 node 生命周期

ASM 通过布局规范决定视图层级，Texture 通过当前布局规范、过去布局规范计算 node 的插入、移除。为了支持插入、移除动画，新 node 在动画开始时添加，原来 node 在视图结束时移除。

> 上一篇：[Texture 基本控件 Node](https://github.com/pro648/tips/wiki/Texture%20%E5%9F%BA%E6%9C%AC%E6%8E%A7%E4%BB%B6%20Node)