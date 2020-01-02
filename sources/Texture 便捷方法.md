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

## Hit Test Slop

`ASDisplayNode`具有类型为`UIEdgeInsets`的`hitTestSlop`属性。当将其设置为正值时，缩小点击范围；设置为负值时，扩大点击范围。

所有 node 均继承自`ASDisplayNode`，因此所有 node 均可以使用`hitTestSlop`属性。

node 获取触摸事件的能力受父 node 尺寸、`hitTestSlop`限制，如果子 node 想要超出父 node 尺寸，则需要扩大父 node `hitTestSlop`以包含 child node 需要响应触摸事件区域。

#### 用途

如果控件高度不足44 point（推荐的最小点击范围），则可以计算差值，使用`hitTestSlop`属性设置负值扩大点击区域。

```
ASTextNode *textNode = [[ASTextNode alloc] init];

CGFloat padding = (44.0 - button.bounds.size.height)/2.0;
textNode.hitTestSlop = UIEdgeInsetsMake(-padding, 0, -padding, 0);
```

## 批量拉取

Texture 的批量拉取（batch fetching api）功能可以很方便的拉取数据。`UIKit`通常在`scrollViewDidScroll:`方法中实现批量拉取功能，Texture 提供了一种更易用的拉取机制。

默认情况下，当用户滑动到距离 table、collection 内容末尾两屏幕时，将尝试拉取更多数据。

如果需要配置触发拉取的距离，只需设置`ASTableNode`、`ASCollectionNode`的`leadingScreensForBatching`属性。

```
tableNode.leadingScreensForBatching = 3.0;  // overriding default of 2.0
```

#### 批量拉取 delegate

在以下方法中决定是否执行批量拉取：

```
// ASTableNode
- (BOOL)shouldBatchFetchForTableNode:(ASTableNode *)tableNode
{
  if (_weNeedMoreContent) {
    return YES;
  }

  return NO;
}

// ASCollectionNode
- (BOOL)shouldBatchFetchForCollectionNode:(ASCollectionNode *)collectionNode
{
  if (_weNeedMoreContent) {
    return YES;
  }

  return NO;
}
```

当进入到需要拉取的区域时会触发上述方法。如果有更多数据则返回YES，进行拉取；反之，不拉取。

> 如果未实现上述方法，在进入拉取区域时会通知其`asyncDelegate`。

如果上述方法返回了YES，则会调用下面的方法：

```
// ASTableNode
-tableNode:willBeginBatchFetchWithContext:

// ASCollectionNode
-collectionNode:willBeginBatchFetchWithContext:
```

在上述方法中执行拉取工作。比如网络 API、本地数据库。

```
- (void)tableNode:(ASTableNode *)tableNode willBeginBatchFetchWithContext:(ASBatchContext *)context 
{
  // Fetch data most of the time asynchronously from an API or local database
  NSArray *newPhotos = [SomeSource getNewPhotos];

  // Insert data into table or collection node
  [self insertNewRowsInTableNode:newPhotos];

  // Decide if it's still necessary to trigger more batch fetches in the future
  _stillDataToFetch = ...;

  // Properly finish the batch fetch
  [context completeBatchFetching:YES];
}
```

> 上述API会在后台线程调用。如需在主线程执行任务，则应将其调度到主线程。

拉取完成后，必须告知 Texture 该过程已经完成。为此，需要使用参数`context`调用`completeBatchFetching:`方法，且为`completeBatchFetching:`方法传入YES。只有传入YES，再次需要拉取时才会尝试拉取。

可以查看以下部分demo了解具体使用：

- [ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)
- [Kittens](https://github.com/texturegroup/texture/tree/master/examples/Kittens)
- [CatDealsCollectionView](https://github.com/texturegroup/texture/tree/master/examples/CatDealsCollectionView)
- [ASCollectionView](https://github.com/texturegroup/texture/tree/master/examples/ASCollectionView)

## 自动节点管理

想要使用 Texture 布局动画，必须开启自动节点管理（Automic Subnode Management，简称ASM）。即使没有使用 Texture 布局动画，开启 ASM 也可以减少代码量。

开启 ASM 后，无需调用`addSubnode:`、`removeFromSuperNode`方法。添加、移除完全由`layoutSpecThatFits:`方法决定。

#### 示例

示例代码来自[ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)，其中`ASCellNode`创建了一个简单的照片流。

下面的代码1使用了熟悉的`addSubNode:`模式，代码2使用了 automatic subside management。如下所示：

```
// 代码1
- (instancetype)initWithPhotoObject:(PhotoModel *)photo {
    self = [super init];
    
    if (self) {
        _photoModel = photo;
        
        _userAvatarImageNode = [[ASNetworkImageNode alloc] init];
        _userAvatarImageNode.URL = photo.ownerUserProfile.userPicURL;
        [self addSubnode:_userAvatarImageNode];
        
        _photoImageNode = [[ASNetworkImageNode alloc] init];
        _photoImageNode.URL = photo.URL;
        [self addSubnode:_photoImageNode];
        
        _userNameTextNode = [[ASTextNode alloc] init];
        _userNameTextNode.attributedString = [photo.ownerUserProfile usernameAttributedStringWithFontSize:FONT_SIZE];
        [self addSubnode:_userNameTextNode];
        
        _photoLocationTextNode = [[ASTextNode alloc] init];
        [photo.location reverseGeocodedLocationWithCompletionBlock:^(LocationModel *locationModel) {
            if (locationModel == _photoModel.location) {
                _photoLocationTextNode.attributedString = [photo locationAttributedStringWithFontSize:FONT_SIZE];
                [self setNeedsLayout];
            }
        }];
        [self addSubnode:_photoLocationTextNode];
    }
    
    return self;
}

// 代码2
- (instancetype)initWithPhotoObject:(PhotoModel *)photo {
    self = [super init];
    
    if (self) {
        self.automaticallyManagesSubnodes = YES;
        
        _photoModel = photo;
        
        _userAvatarImageNode = [[ASNetworkImageNode alloc] init];
        _userAvatarImageNode.URL = photo.ownerUserProfile.userPicURL;
        
        _photoImageNode = [[ASNetworkImageNode alloc] init];
        _photoImageNode.URL = photo.URL;
        
        _userNameTextNode = [[ASTextNode alloc] init];
        _userNameTextNode.attributedString = [photo.ownerUserProfile usernameAttributedStringWithFontSize:FONT_SIZE];
        
        _photoLocationTextNode = [[ASTextNode alloc] init];
        [photo.location reverseGeocodedLocationWithCompletionBlock:^(LocationModel *locationModel) {
            if (locationModel == _photoModel.location) {
                _photoLocationTextNode.attributedString = [photo locationAttributedStringWithFontSize:FONT_SIZE];
                [self setNeedsLayout];
            }
        }];
    }
    
    return self;
}
```

`_userAvatarImageNode`、`_photoImageNode`和`_photoLocationLabel`根据网络数据决定是否添加到视图中，应当何时添加呢？

ASM 会根据 cell 的`ASLayoutSpec`决定是否将其添加到 UI 中。

> `ASLayoutSpeck`决定 UI 的视图层级，其由`layoutSpecThatFits:`返回。

你需要使用`layoutSpecThatFits:`构建视图，查看`ASCellNode`的部分布局代码：

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize {
    ASStackLayoutSpec *headerSubStack = [ASStackLayoutSpec verticalStackLayoutSpec];
    headerSubStack.flexShrink         = YES;
    if (_photoLocationLabel.attributedString) {
        [headerSubStack setChildren:@[_userNameLabel, _photoLocationLabel]];
    } else {
        [headerSubStack setChildren:@[_userNameLabel]];
    }
    
    _userAvatarImageNode.preferredFrameSize = CGSizeMake(USER_IMAGE_HEIGHT, USER_IMAGE_HEIGHT);     // constrain avatar image frame size
    
    ASLayoutSpec *spacer           = [[ASLayoutSpec alloc] init];
    spacer.flexGrow                = YES;
    
    UIEdgeInsets avatarInsets      = UIEdgeInsetsMake(HORIZONTAL_BUFFER, 0, HORIZONTAL_BUFFER, HORIZONTAL_BUFFER);
    ASInsetLayoutSpec *avatarInset = [ASInsetLayoutSpec insetLayoutSpecWithInsets:avatarInsets child:_userAvatarImageNode];
    
    ASStackLayoutSpec *headerStack = [ASStackLayoutSpec horizontalStackLayoutSpec];
    headerStack.alignItems         = ASStackLayoutAlignItemsCenter;                     // center items vertically in horizontal stack
    headerStack.justifyContent     = ASStackLayoutJustifyContentStart;                  // justify content to the left side of the header stack
    [headerStack setChildren:@[avatarInset, headerSubStack, spacer]];
    
    // header inset stack
    UIEdgeInsets insets                = UIEdgeInsetsMake(0, HORIZONTAL_BUFFER, 0, HORIZONTAL_BUFFER);
    ASInsetLayoutSpec *headerWithInset = [ASInsetLayoutSpec insetLayoutSpecWithInsets:insets child:headerStack];
    
    // footer inset stack
    UIEdgeInsets footerInsets          = UIEdgeInsetsMake(VERTICAL_BUFFER, HORIZONTAL_BUFFER, VERTICAL_BUFFER, HORIZONTAL_BUFFER);
    ASInsetLayoutSpec *footerWithInset = [ASInsetLayoutSpec insetLayoutSpecWithInsets:footerInsets child:_photoCommentsNode];
    
    // vertical stack
    CGFloat cellWidth                  = constrainedSize.max.width;
    _photoImageNode.preferredFrameSize = CGSizeMake(cellWidth, cellWidth);              // constrain photo frame size
    
    ASStackLayoutSpec *verticalStack   = [ASStackLayoutSpec verticalStackLayoutSpec];
    verticalStack.alignItems           = ASStackLayoutAlignItemsStretch;                // stretch headerStack to fill horizontal space
    [verticalStack setChildren:@[headerWithInset, _photoImageNode, footerWithInset]];
    
    return verticalStack;
}
```

可以看到`headerSubStack`的 children 根据`_photoLocationLabel`字符串是否存在来决定。

#### 更新`ASLayoutSpec`

如果某些变化改变了`ASLayoutSpec`，需要调用`setNeedsLayout`。其与动画中的`transitionLayout:duration:0`方法类似。可以在[PhotoCellNode](https://github.com/facebookarchive/AsyncDisplayKit/blob/master/examples/ASDKgram/Sample/PhotoCellNode.m#L85-L94)中查看如下：

```
    [photo.location reverseGeocodedLocationWithCompletionBlock:^(LocationModel *locationModel) {
      
      // check and make sure this is still relevant for this cell (and not an old cell)
      // make sure to use _photoModel instance variable as photo may change when cell is reused,
      // where as local variable will never change
      if (locationModel == _photoModel.location) {
        _photoLocationLabel.attributedText = [photo locationAttributedStringWithFontSize:FONT_SIZE];
        [self setNeedsLayout];
      }
    }];
```

构建好的`ASLayoutSpec`将自动添加、移除或设置动画。

可以查看[ASDKgram](https://github.com/texturegroup/texture/tree/master/examples/ASDKgram)demo了解具体布局过程，你会发现编写`ASCellNode`非常简单，该布局会根据大量单独数据自行调整。

该示例非常简单，但此功能有许多更强大的用途。

> 当 node 开启了 ASM 后，将不能调用`addSubnode:`和`removeFromSuperNode`方法，否则会抛出 A flattened layout must consist exclusively of node sublayouts 的异常。

## 反转 inversion

`ASTableNode`和`ASCollectionNode`有一个`BOOL`类型属性`inverted`，当设置为`YES`时会自动反转内容，以便从下到上进行布局，即第一个（indexPath 为（0, 0）) node 位于底部，而非像往常一样在顶部。这对于聊天应用非常方便，只需设置`inverted`属性。

开启 inversion 后，开发者只需调整`ASTableNode`和`ASCollectionNode`的`contentInset`属性。如下所示：

```
  CGFloat inset = [self topBarsHeight];
  self.tableNode.view.contentInset = UIEdgeInsetsMake(0, 0, inset, 0);
  self.tableNode.view.scrollIndicatorInsets = UIEdgeInsetsMake(0, 0, inset, 0);
  
  let inset = self.topBarsHeight
  self.tableNode.view.contentInset = UIEdgeInsets(top: 0.0, left: 0.0, bottom: inset, right: 0.0)
  self.tableNode.view.scrollIndicatorInsets = UIEdgeInsets(top: 0.0, left: 0.0, bottom: inset, right: 0.0)
```

> 查看[SocialAppLayout-Inverted](https://github.com/texturegroup/texture/tree/master/examples/SocialAppLayout-Inverted)demo了解详细实现。

## 修改图像块 Image Modification Blocks

通常，修改图像外观的操作对于主线程来说是昂贵操作，将其移到后台线程更为高效。

通过将`imageModificationBlock`分配给 imageNode，可以定义一组转换。转换会异步修改图像。

```
_backgroundImageNode.imageModificationBlock = ^(UIImage *image) {
	UIImage *newImage = [image applyBlurWithRadius:30
		tintColor:[UIColor colorWithWhite:0.5 alpha:0.3]
		saturationDeltaFactor:1.8
		maskImage:nil];
	return newImage ?: image;
};

//some time later...

_backgroundImageNode.image = someImage;
```

someImage 先异步修改，后分配给 imageNode 进行显示。

#### 添加图像处理

利用`imageModificationBlock`添加图像处理是最高效的方式。如果提供了 block，则可以在显示阶段执行绘制工作。由于显示是在后台线程执行的，因此不会堵塞主线程。

在下面的代码中，在父视图`init`方法中初始化 avatar node，avatar node 头像需要为圆形。通过提供`imageModificationBlock`将头像转换为圆形。

```
- (instancetype)init
{
// ...
  _userAvatarImageNode.imageModificationBlock = ^UIImage *(UIImage *image) {
    CGSize profileImageSize = CGSizeMake(USER_IMAGE_HEIGHT, USER_IMAGE_HEIGHT);
    return [image makeCircularImageWithSize:profileImageSize];
  };
  // ...
}
```

实际的绘制代码添加到了`UIImage`的分类中，如下所示：

```
@implementation UIImage (Additions)
- (UIImage *)makeCircularImageWithSize:(CGSize)size
{
  // make a CGRect with the image's size
  CGRect circleRect = (CGRect) {CGPointZero, size};

  // begin the image context since we're not in a drawRect:
  UIGraphicsBeginImageContextWithOptions(circleRect.size, NO, 0);

  // create a UIBezierPath circle
  UIBezierPath *circle = [UIBezierPath bezierPathWithRoundedRect:circleRect cornerRadius:circleRect.size.width/2];

  // clip to the circle
  [circle addClip];

  // draw the image in the circleRect *AFTER* the context is clipped
  [self drawInRect:circleRect];

  // get an image from the image context
  UIImage *roundedImage = UIGraphicsGetImageFromCurrentImageContext();

  // end the image context since we're not in a drawRect:
  UIGraphicsEndImageContext();

  return roundedImage;
}
@end
```

`imageModificationBlock`方法非常方便，可以用于添加各种图像效果而无需额外的调用显示。例如：圆角、边框或其他覆盖。

## 占位符 Placeholders

#### ASDisplayNode 实现占位符

`ASDisplayNode`的子类可以实现`placeholderImage`方法，以提供覆盖内容的占位符，直到节点内容完成显示。要使用占位符，请设置`placeholderEnabled = YES;`，另外还可以选择实现`placeholderFadeDuration`。

对于渲染图片，使用 node 的`calculatedSize`属性。

> `placeholderImage`函数会在后台线程调用，因此需要确保线程安全。`[UIImage imageNamed:]`在 iOS 9 及以后线程安全，如果需要支持更低版本系统，可以使用`[UIImage imageWithContentsOfFile:]`方法。
>
> `imageNamed:`方法会先检查缓存中是否有所需图片，如果缓存中不存在所需图片，则从 asset catalog 或磁盘加载。系统可能清空缓存以释放内存，清空时只会移除在缓存中且未正在使用的图片。
>
> 如果图片只显示一次，不想将图片加入到缓存中，则可以使用`imageWithContentsOfFile:`方法加载图片。

`UIImage+ASConvenience`分类提供了创建占位图图片的方法，包括圆形、矩形等。

查看[Placeholders](https://github.com/texturegroup/texture/tree/master/examples_extra/Placeholders)demo了解具体使用。

#### ASNetworkImageNode 默认图片

除占位符，`ASNetworkImageNode`还有`defaultImage`属性。占位符一般是临时性的，默认图片可能永久存在。例如图片 URL 为`nil`，或加载失败。

推荐为头像设置默认图片，为图片设置占位符。

## 与UICollectionViewCell组合使用

`ASCollectionNode`提供了`ASCellNode`和`UICollectionViewCell`同时使用的机制，但`UICollectionViewCell`不会获得预加载、异步布局、异步渲染的性能提升，即使与其他`ASCellNode`组合使用在同一个`ASCollectionNode`中。

组合使用方便开发者尝试 Texture，而不必重写所有 cell。

想用组合使用`UICollectionViewCell`，需满足：

1. 遵守`ASCollecitonDataSourceInterop`协议，可选实现`ASCollectionDelegateInterop`协议。
2. 在`viewDidLoad`中使用`collectionNode.view`调用`registerCellClass:`，或注册一个`onDidLoad:`块。
3. 在`collectionNode:nodeBlockForItemAtIndexPath:`方法或`collectionNode:nodeForItemAtIndexPath:`方法中返回nil。需要注意的是，不能在`nodeBlock`中返回nil。
4. 最后，必须实现提供 item 大小的方法。有以下两种方式：
   1. `UICollectionViewFlowLayout`，实现`collectionNode:constrainedSizeForItemAtIndexPath:`方法。
   2. 自定义布局。设置`view.layoutInspector`，并实现`collectionView:constrainedSizeForNodeAtIndexPath:`。

默认情况下，`UICollectionViewDataSource`只在未提供`ASCellNode`时使用。然而，如果开启了`dequeuesCellsForNodeBackedItems`，则会调用`UICollectionViewDataSource`协议内方法重用cell，并期望返回`ASCollectionViewCells`类型对象。

[CustomCollectionView](https://github.com/texturegroup/texture/tree/master/examples/CustomCollectionView)demo 演示了如何组合使用`UIKit`cell和`ASCellNodes`。

打开demo后确保`kShowUICollectionViewCells`为`YES`。在这个示例中，`collectionNode:nodeBlockForItemAtIndexPath:`会为3倍数 cell 返回 nil。当返回 nil 时，`ASCollectionNode`会自动调用`collectionView:cellForItemAtIndexPath:`方法。

```
- (ASCellNodeBlock)collectionNode:(ASCollectionNode *)collectionNode nodeBlockForItemAtIndexPath:(NSIndexPath *)indexPath {
  if (kShowUICollectionViewCells && indexPath.item % 3 == 1) {
    // When enabled, return nil for every third cell and then cellForItemAtIndexPath: will be called.
    return nil;
  }
  
  UIImage *image = _sections[indexPath.section][indexPath.item];
  return ^{
    return [[ImageCellNode alloc] initWithImage:image];
  };
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath {
  return [_collectionNode.view dequeueReusableCellWithReuseIdentifier:kReuseIdentifier forIndexPath:indexPath];
}
```

> 上一篇：[Texture 布局 Layout](https://github.com/pro648/tips/wiki/Texture%20%E5%B8%83%E5%B1%80%20Layout)
>
> 下一篇：[Texture 性能优化](https://github.com/pro648/tips/wiki/Texture%20%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)