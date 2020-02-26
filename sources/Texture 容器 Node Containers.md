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

## ASViewController

`ASViewController`是`UIViewController`的子类，添加了诸多实用功能以维护`ASDisplayNode`层级结构。

`ASViewController`用以替代`UIViewController`，可以用在`UINavigationController`、`UITabBarController`和`UISplitViewController`中，也可以单独使用。

使用`ASViewController`的好处：

1. 节省内存。离开屏幕的`ASViewController`将自动减少获取数据的大小，减少子级的显示范围。这是大型应用程序中内存管理的关键。
2. `ASVisibility`功能。当在`ASNavigationController`和`ASTabBarController`中使用时，`ASViewController`将获知用户点击几次后会显示该控制器。

#### ASViewController的使用

`UIViewController`提供了自己的视图，`ASViewController`在其初始化方法`initWithNode:`中指定了负责管理的 node。

下面`ASViewController`的子类`PhotoFeedNodeController`想要使用 table node 作为管理者。table node 被赋值给`initWithNode:`：

```
- (instancetype)init
{
  _tableNode = [[ASTableNode alloc] initWithStyle:UITableViewStylePlain];
  self = [super initWithNode:_tableNode];
  
  if (self) {
    _tableNode.dataSource = self;
    _tableNode.delegate = self;
  }
  
  return self;
}
```

> 示例来自[ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)

> 如果app已经具有复杂视图层级，也可以将其全部更改为`ASViewController`的子类。也就是说，即使不使用`ASViewController`的指定初始化程序`initWithNode:`，也可以仅以传统`UIViewController`方式使用`ASViewController`。这样可以选择在部分位置支持`ASViewController`。

## ASTableNode

`ASTableNode`用以代替`UIKit`中的`UITableView`。

`UITableView`中的

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
```

可以使用`ASTableNode`中的下面任一方法替代：

```
- (ASCellNode *)tableNode:(ASTableNode *)tableNode nodeForRowAtIndexPath:(NSIndexPath *)indexPath

- (ASCellNodeBlock)tableNode:(ASTableNode *)tableNode nodeBlockForRowAtIndexPath:(NSIndexPath *)indexPath
```

> 推荐使用 node block，这样 cell 可以并发绘制。也就是所有 cell node 初始化方法会在后台调用，因此需要注意线程安全。

上述方法返回`ASCellNode`或`ASCellNodeBlock`，`ASCellNodeBlock`block可以在后台线程创建`ASCellNode`。`ASCellNode`被用在了`ASTableNode`、`ASCollectionNode`和`ASPagerNode`。

`UITableViewCell`会采用复用机制提高性能，但 Texture 中的上述方法不会复用 cell。

#### 使用ASViewController替换UITableViewController

Texture 未提供与`UITableViewController`对应的控件，但可以使用`ASViewController`初始化`ASTableNode`获得。

如[ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)中PhotoFeedNodeController所示，在`ASViewController`的初始化方法中指定由其管理`ASTableNode`。

```
- (instancetype)init
{
    _tableNode = [[ASTableNode alloc] initWithStyle:UITableViewStylePlain];
    self = [super initWithNode:_tableNode];
    
    if (self) {
      _tableNode.dataSource = self;
      _tableNode.delegate = self;
    }
    
    return self;
}
```

#### Node Block 线程安全

Node block 必须线程安全，需要确保在 node block 外访问数据模型。因此，不要在 node block 中使用 indexPath。

查看[ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)中的`PhotoFeedNodeController`的`tableNode:nodeBlockForRowAtIndexPath:`方法：

```
- (ASCellNodeBlock)tableNode:(ASTableNode *)tableNode nodeBlockForRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoModel *photoModel = [_photoFeed objectAtIndex:indexPath.row];
    
    // this may be executed on a background thread - it is important to make sure it is thread safe
    ASCellNode *(^cellNodeBlock)() = ^ASCellNode *() {
        PhotoCellNode *cellNode = [[PhotoCellNode alloc] initWithPhoto:photoModel];
        cellNode.delegate = self;
        return cellNode;
    };
    
    return cellNodeBlock;
}
```

从上述代码可以看到如何在创建 node block 前使用 indexPath 获取数据模型。

#### 获取 ASTableView

为了推荐使用`ASTableNode`，目前已经移除了`ASTableView`。

> `ASTableView`是`UITableView`的子类，目前仍被`ASTableNode`在内部使用，但不应直接创建`ASTableView`，可以通过`ASTableNode`的`view`属性获取。但应在`viewDidLoad`或`didLoad`之后调用。

例如，想要设置表视图的分割线，可以在`viewDidLoad`方法中设置 table node 的 view 属性实现：

```
- (void)viewDidLoad
{
  [super viewDidLoad];
  
  _tableNode.view.allowsSelection = NO;
  _tableNode.view.separatorStyle = UITableViewCellSeparatorStyleNone;
  _tableNode.view.leadingScreensForBatching = 3.0;  // default is 2.0
}
```

#### Table Row Height

`ASTableNode`并未提供与`UITableView`的`tableView:heightForRowAtIndexPath:`对应的方法。

这是因为 node 会根据约束决定自身高度，因此无需在 table node 层设置 cell 高度。

node 通过`layoutSpecThatFits:`方法返回的 layout spec 确定自身尺寸。所有 node 均能够根据布局规范计算自身尺寸。

> 默认情况下，`ASTableNode`为 cell 提供 size range，最小宽度为 tableNode 的宽，最小高度为0；最大宽度仍然为 tableNode 的宽，最大高度为`FLT_MAX`。
>
> 也就是说，tableNode cell 的宽度始终为 tableNode 的宽，但高度是灵活的，以便 cell 自适应调整大小。

在`ASCellNode`上调用`setNeedsLayout`，会自动重新布局。如果 cell 大小发生变化，会自动通知 table node 进行更新。在`UIKit`中必须调用`reloadRowsAtIndexPaths:withRowAnimation:`。查看`ASDKgram` demo 对比`UITableView`和`ASTableNode`的区别。

以下是使用了`ASTableNode`的demo：

- [ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)
- [Kittens](https://github.com/texturegroup/texture/tree/master/examples/Kittens)
- [HorizontalWithinVerticalScrolling](https://github.com/texturegroup/texture/tree/master/examples/HorizontalWithinVerticalScrolling)
- [VerticalWithinHorizontalScrolling](https://github.com/texturegroup/texture/tree/master/examples/VerticalWithinHorizontalScrolling)
- [SocialAppLayout](https://github.com/texturegroup/texture/tree/master/examples/SocialAppLayout)

## ASCollectionNode

`ASCollectionNode`与`UIKit`中的`UICollectionView`等效，用以替代`UICollectionView`。

`UICollectionView`中的

```
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath;
```

使用下面任一方法替换：

```
- (ASCellNode *)collectionNode:(ASCollectionNode *)collectionNode nodeForItemAtIndexPath:(NSIndexPath *)indexPath

- (ASCellNodeBlock)collectionNode:(ASCollectionNode *)collectionNode nodeBlockForItemAtIndexPath:(NSIndexPath *)indexPath
```

与`ASTableNode`一样，仍然推荐使用`ASCellNodeBlock`方法，这样 cell 可以并发创建。

和`ASTableNode`一样，`ASCollectionNode`满足以下几点：

- `ASCollectionNode`不会对 cell 进行复用。
- 推荐使用 nodeBlock 方法创建 node。
- 返回的 node block 需线程安全。
- `ASCellNode`同时用在`ASTableNode`、`ASCollectionNode`和`ASPagerNode`。

> 在线程安全、替换`UICollectionViewController`、获取`ASCollectionView`方面，`ASCollectionNode`与`ASTableNode`类似，具体可以查看[文档](https://github.com/TextureGroup/Texture/blob/master/docs/_docs/containers-ascollectionnode.md#node-block-thread-safety-warning)。

#### Cell 大小和布局

Cell 将根据 constrained size 调整大小，并根据提供的`UICollectionViewLayout`进行布局。

也可以通过`ASCollectionNode`的`constrainedSizeForItemAtIndexPath:`限制 cell 大小。

#### 示例

[CustomCollectionView](https://github.com/texturegroup/texture/tree/master/examples/CustomCollectionView)详细演示了如何使用`ASCollectionNode`，其包含一个 Pinterest 风格`ASCollectionNode`，并使用自定义`UICollectionViewLayout`进行布局。

以下是采用了`ASCollectionNode`的demo：

- [ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)
- [CatDealsCollectionView](https://github.com/texturegroup/texture/tree/master/examples/CatDealsCollectionView)
- [ASCollectionView](https://github.com/texturegroup/texture/tree/master/examples/ASCollectionView)
- [CustomCollectionView](https://github.com/texturegroup/texture/tree/master/examples/CustomCollectionView)

#### 与 UICollectionViewCell 组合使用

`ASCollectionNode`支持`UICollectionViewCell`和`ASCellNode`一起使用。但`UIKit`的 cell 将无法获得`ASCellNode`的性能提升（例如，预加载、异步布局、异步绘制），即使在`ASCollectionNode`中使用。

> 组合使用可以方便开发人员测试框架，而无需转换所有 cell。[CustomCollectionView](https://github.com/texturegroup/texture/tree/master/examples/CustomCollectionView)组合使用了`UICollectionViewCell`和`ASCellNode`。

## ASPagerNode

`ASPagerNode`是`ASCollectionNode`的子类，其使用了特定的`UICollectionViewLayout`。

使用`ASPagerNode`可以创建类似于`UIPageViewController`样式的 UI。旋转设备时，`ASPagerNode`能停留在当前页码，但不支持循环滚动。

dataSource 方法如下：

```
- (NSInteger)numberOfPagesInPagerNode:(ASPagerNode *)pagerNode

// 下面两个方法选择其一即可，推荐使用ASCellNodeBlock
- (ASCellNode *)pagerNode:(ASPagerNode *)pagerNode nodeAtIndex:(NSInteger)index
- (ASCellNodeBlock)pagerNode:(ASPagerNode *)pagerNode nodeBlockAtIndex:(NSInteger)index`
```

与`ASTableNode`、`ASCollectionNode`类似，推荐使用`ASCellNodeBlock`。这样可以在后台线程创建`ASCellNode`。

`ASCellNode`不会进行复用，请勿依赖复用机制。与`UIKit`不同，即将显示时不会调用这些方法。

> 使用`ASCellNodeBlock`创建 cell 时，需要注意线程安全。

#### 使用ASViewController优化性能

将已有的`UIViewController`或`ASViewController`返回给`ASCellNode`的初始化方法，可以提高性能，使用`ASViewController`能进一步优化性能。

```
- (ASCellNode *)pagerNode:(ASPagerNode *)pagerNode nodeAtIndex:(NSInteger)index
{
    NSArray *animals = self.animals[index];
    
    ASCellNode *node = [[ASCellNode alloc] initWithViewControllerBlock:^{
        return [[AnimalTableNodeController alloc] initWithAnimals:animals];;
    } didLoadBlock:nil];
    
    node.style.preferredSize = pagerNode.bounds.size;
    
    return node;
}
```

上面的示例使用`initWithViewControllerBlock:`方法构建 node。通常会为 cell 提供大小（style.preferredSize）以便正确布局。

以下是采用了`ASPagerNode`的demo：

- [PagerNode](https://github.com/texturegroup/texture/tree/master/examples/PagerNode)
- [VerticalWithinHorizontalScrolling](https://github.com/texturegroup/texture/tree/master/examples/VerticalWithinHorizontalScrolling)

> 上一篇：[Texture 性能优化](https://github.com/pro648/tips/wiki/Texture%20%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
>
> 下一篇：[Texture 基本控件 Node](https://github.com/pro648/tips/wiki/Texture%20%E5%9F%BA%E6%9C%AC%E6%8E%A7%E4%BB%B6%20Node)