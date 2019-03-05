CocoaPods是应用级依赖项管理器（dependency manager），用于管理Objective-C、Swift语言应用的依赖项。由Eloy Durán 和 Fabio Pelosin于2011年开发完成。

## 1. 安装CocoaPods

CocoaPods是使用Ruby语言编写的，macOS系统默认安装了Ruby。CocoaPods建议使用默认的Ruby安装CocoaPods，除非你对Ruby有所了解。

使用如下命令安装CocoaPods：

```
$ sudo gem install cocoapods
```

> 使用系统自带Ruby安装CocoaPods时，需要使用`sudo`权限。

输入上面命令后，命令行提示如下：

```
Fetching: concurrent-ruby-1.0.5.gem (100%)
Successfully installed concurrent-ruby-1.0.5
Fetching: i18n-0.9.1.gem (100%)
Successfully installed i18n-0.9.1
...
Installing ri documentation for cocoapods-1.3.1
Done installing documentation for concurrent-ruby, i18n, thread_safe, tzinfo, activesupport, nap, fuzzy_match, cocoapods-core, claide, cocoapods-deintegrate, cocoapods-downloader, cocoapods-plugins, cocoapods-search, cocoapods-stats, netrc, cocoapods-trunk, cocoapods-try, molinillo, colored2, nanaimo, xcodeproj, escape, fourflusher, gh_inspector, ruby-macho, cocoapods after 31 seconds
26 gems installed
```

使用如下命令查看CocoaPods版本：

```
$ gem list --local | grep cocoapods
cocoapods (1.3.1)
cocoapods-core (1.3.1)
cocoapods-deintegrate (1.0.1)
cocoapods-downloader (1.1.3)
cocoapods-plugins (1.0.0)
cocoapods-search (1.0.0)
cocoapods-stats (1.0.0)
cocoapods-trunk (1.3.0)
cocoapods-try (1.1.0)
```

可以看到`cocoapods`版本为`1.3.1`，同时也可以看到其它部分版本。

> 默认安装的CocoaPods为正式版，使用`gem install cocoapods --pre`命令安装测试版CocoaPods，使用`gem install cocoapods -v x.y.z` 命令安装指定版本CocoaPods。

## 2. 更新CocoaPods

只需要再次安装，就可以将CocoaPods升级到最新版：

```
$ [sudo] gem install cocoapods
```

如果安装CocoaPods时使用了`sudo`权限，更新时也需要`sudo`权限。

如果CocoaPods有了新版本，在你使用`pod`命令时会得到*CocoaPods X.X.X is now available, please update.*的提示信息。

## 3. 移除低版本CocoaPods

CocoaPods更新到新版本后，低版本的CocoaPods并不会被自动移除。如下所示：

```
$ gem list --local | grep cocoapods
cocoapods (1.3.1, 1.1.0)
cocoapods-core (1.3.1, 1.1.0)
cocoapods-deintegrate (1.0.1)
cocoapods-downloader (1.1.3)
cocoapods-plugins (1.0.0)
cocoapods-search (1.0.0)
cocoapods-stats (1.0.0)
cocoapods-trunk (1.3.0)
cocoapods-try (1.1.0)
```

使用如下命令移除指定版本CocoaPods：

```
$ gem uninstall cocoapods -v 1.1.0
Successfully uninstalled cocoapods-1.1.0
$ gem uninstall cocoapods-core -v 1.1.0
Successfully uninstalled cocoapods-core-1.1.0

// 查看cocoapods版本。
$ gem list --local | grep cocoapods
cocoapods (1.3.1)
cocoapods-core (1.3.1)
cocoapods-deintegrate (1.0.1)
cocoapods-downloader (1.1.3)
cocoapods-plugins (1.0.0)
cocoapods-search (1.0.0)
cocoapods-stats (1.0.0)
cocoapods-trunk (1.3.0)
cocoapods-try (1.1.0)
```

## 4. 管理第三方库

#### 4.1 创建`Podfile`文件

CocoaPods中`Podfile` 文件用于列出要在项目中使用的库，可以使用以下两种方式创建`Podfile`文件：

- 在项目目录中创建一个名为`Podfile`的空文本文件。
- 将Terminal定位到项目目录，输入`pod init`命令，CocoaPods会自动创建`Podfile`文件，并会自动初始化一些必要代码。推荐使用`pod init`命令创建`Podfile`文件。

使用`pod init`命令创建的`Podfile`文件内容如下：

```
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'

target 'MyApp' do
  # Uncomment the next line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!

  # Pods for MyApp

  target 'MyAppTests' do
    inherit! :search_paths
    # Pods for testing
  end

end                                                                          
```

#### 4.2 列出所需第三方库

`Podfile`文件内容可以非常简单，如下所示：

```
target 'MyApp` do
  pod 'Mantle'
end
```

下面使用vim编辑`Podfile`文件，添加`Mantle`和`ReactiveObjC`第三方库：

```
# 这里Pod支持iOS系统，还可以支持osx。如果没有说明支持的系统，默认支持所有系统。
platform :ios, '9.0'

target 'MyApp' do
  pod 'Mantle'
  pod 'ReactiveObjC', '~> 3.1.0'
  
  target 'MyAppTests' do
    inherit! :search_paths
    # Pods for testing
  end

end
```

添加第三方库时，如果没有指定版本，则默认使用最新正式版。另外，使用如下命令指定版本：

```
// 使用2.1.0版本的Mantle
pod 'Mantle', '2.1.0'
```

除不使用版本号、固定版本号，还可以使用逻辑符指定版本号。

- `'> 0.2'`：比`0.2`高的任何版本。
- `>= 0.2`：大于等于`0.2`版本。
- `< 0.2`：小于`0.2`的版本。
- `~> 0.6.48'`：大于等于`0.6.48`版本，但小于`0.7`版本。
- `'~> 0.6'`：大于等于`0.6`版本，但小于`1.0`版本。
- `'~> 1'`：主版本号大于等于`1`。

#### 4.3 添加第三方库

使用`pod install`命令添加第三方库：

```
$ pod install
Analyzing dependencies
Downloading dependencies
Installing Mantle (2.1.0)
Installing ReactiveObjC (3.1.0)
Generating Pods project
Integrating client project

[!] Please close any current Xcode sessions and use `MyApp.xcworkspace` for this project from now on.
Sending stats
Pod installation complete! There are 2 dependencies from the Podfile and 2 total pods installed.
```

根据终端提示，如果想要使用刚添加的第三方库，必须打开`MyApp.xcworkspace`，而非之前的`MyApp.xcodeProj`。

#### 4.4 导入第三方库

CocoaPods创建`Mantle`pod工程，并添加到你的workspace。因此，当你导入`Mantle`时，是从workspace的其它工程导入，需要使用尖括号（< >）而非双引号（“ ”）。

```
#import <Mantle/Mantle.h>
```

## 5. `pod install` VS `pod update`

刚接触CocoaPods时，很多人会误以为`pod install`用在初次设置工程时，之后使用`pod update`，事实并非如此。

- 使用`pod install`命令在project中添加新的pods。即使之前已经使用过`pod install`命令，在添加、移除pods时，依然使用`pod install`命令。
- 只有需要把pods更新到新版本时，使用`pod update [PODNAME]`命令。

#### 5.1 pod install

第一次为project添加pods，编辑`Podfile`文件以添加、更新、移除pods，均使用`pod install`命令。

- 每次运行`pod install`命令时，CocoaPods会把添加的pod版本信息写入`Podfile.lock`文件，`Podfile.lock`文件跟踪已安装pod版本信息，并锁定到当前版本。
- 当运行`pod install`命令时，CocoaPods只处理未在`Podfile.lock`文件中列出的依赖项。
  - 对于`Podfile.lock`文件中列出的pods，CocoaPods直接下载`Podfile.lock`文件中列出的版本，不会尝试查找新版本。
  - 对于未在`Podfile.lock`文件中列出的pods，CocoaPods会根据`Podfile`文件添加pods。

#### 5.2 pod outdated

当运行`pod outdated`命令时，CocoaPods会查找`Podfile.lock`中所有pods，并列出更新版本的pods。也就是说，运行`pod update [PODNAME]`命令时，如果新版本符合`Podfile`的描述，则会更新pod到新版本。

```
$ pod outdated
Updating spec repo `master`
  $ /usr/local/bin/git -C /Users/ad/.cocoapods/repos/master fetch origin
  --progress
  remote: Counting objects: 45, done.        
  remote: Compressing objects: 100% (40/40), done.        
  remote: Total 45 (delta 27), reused 0 (delta 0), pack-reused 0        
  From https://github.com/CocoaPods/Specs
     f7d947da03c..a14b55af97b  master     -> origin/master
  $ /usr/local/bin/git -C /Users/ad/.cocoapods/repos/master rev-parse
  --abbrev-ref HEAD
  master
  $ /usr/local/bin/git -C /Users/ad/.cocoapods/repos/master reset --hard
  origin/master
  HEAD is now at a14b55af97b [Add] Waterpurifier 1.0.4

CocoaPods 1.4.0.rc.1 is available.
To update use: `gem install cocoapods --pre`
[!] This is a test version we'd love you to try.

For more information, see https://blog.cocoapods.org and the CHANGELOG for this version at https://github.com/CocoaPods/CocoaPods/releases/tag/1.4.0.rc.1

Analyzing dependencies
The following pod updates are available:
- Mantle 1.4.1 -> 1.4.1 (latest version 2.1.0)
```

#### 5.3 pod update

运行`pod update [PODNAME]`命令时，Cocoapods会查找`[PODNAME]`的新版本，如果新版本同时满足`Podfile` 文件对版本描述，则会更新pod。

如果运行`pod update`命令时不添加pod名称，则会查找所有pods的新版本，并尝试更新。

## 6. 查找第三方库 pod search

`Podfile`文件中第三方库名称区分大小写，写错后install第三方库时会提示语法错误（syntax error）。为避免出错，可以在添加之前先查找对应库。

在Terminal中输入`pod search [PODNAME]`即可进行查找，查找时不区分大小写。可以配合其它可选项进行检索，如`—simple`只查找pod名称，`—stats`显示GitHub中watch fork数据。

```
$ pod search --simple --stats AFNetworking
-> AFNetworking (3.1.0)
   A delightful iOS and OS X networking framework.
   pod 'AFNetworking', '~> 3.1.0'
   - Homepage: https://github.com/AFNetworking/AFNetworking
   - Source:   https://github.com/AFNetworking/AFNetworking.git
   - Versions: 3.1.0, 3.0.4, 3.0.3, 3.0.2, 3.0.1, 3.0.0, 3.0.0-beta.3,
   3.0.0-beta.2, 3.0.0-beta.1, 2.6.3, 2.6.2, 2.6.1, 2.6.0, 2.5.4, 2.5.3, 2.5.2,
   2.5.1, 2.5.0, 2.4.1, 2.4.0, 2.3.1, 2.3.0, 2.2.4, 2.2.3, 2.2.2, 2.2.1, 2.2.0,
   2.1.0, 2.0.3, 2.0.2, 2.0.1, 2.0.0, 2.0.0-RC3, 2.0.0-RC2, 2.0.0-RC1, 1.3.4,
   1.3.3, 1.3.2, 1.3.1, 1.3.0, 1.2.1, 1.2.0, 1.1.0, 1.0.1, 1.0, 1.0RC3, 1.0RC2,
   1.0RC1, 0.10.1, 0.10.0, 0.9.2, 0.9.1, 0.9.0, 0.7.0, 0.5.1 [master repo]
   - Author:   Mattt Thompson
   - License:  MIT
   - Platform: iOS 7.0 - macOS 10.9 - tvOS 9.0 - watchOS 2.0
   - Stars:    30058
   - Forks:    9386
   - Subspecs:
     - AFNetworking/Serialization (3.1.0)
     - AFNetworking/Security (3.1.0)
     - AFNetworking/Reachability (3.1.0)
     - AFNetworking/NSURLSession (3.1.0)
     - AFNetworking/UIKit (3.1.0)
```

另外，也可以在[CocoaPods](https://cocoapods.org/)官网进行搜索。

## 总结

由于Swift不支持静态库（static library），在Swift中使用时必须使用framework。

```
target 'MyApp` do
  use_frameworks!
  pod 'MJRefresh`, '~> 3.1.15`
end
```

有时添加的第三库会有一些警告信息，通过在`Podfile`添加以下命令可以隐藏第三方库中的警告信息。

```
platform :ios, '9.0'
# 隐藏所有警告
inhibit_all_warnings!

target 'MyApp' do
  pod 'Mantle', '1.4.1'
  # 不要隐藏ReactiveObjC的警告，也可以用此方法只隐藏指定pod警告。
  pod 'ReactiveObjC', '~> 3.1.0', :inhibit_warnings => false

end
```

另外，关于是否将`Pods`文件加入版本控制，可以根据下面的利弊自行决定：

- 将`Pods`文件加入版本控制：
  - 可以在clone仓库后立即开始工作，不需要`pod install`。
  - 可以保证每人都使用相同版本。
  - 即使原始pod已经移除也可以继续使用。
- 将`Pods`文件排除在版本控制系统外：
  - 使得仓库更小。
  - 合并分支时不会出现不同Pod版本间冲突。

但需要注意的是，一定要把`Podfile`文件和`Podfile.lock`文件加入版本控制。

最后，CocoaPods除了将开源库添加到你的项目外，还可以创建公开、私有pod，具体方法可以查看[使用CocoaPods创建公开、私有pod](https://github.com/pro648/tips/wiki/%E4%BD%BF%E7%94%A8CocoaPods%E5%88%9B%E5%BB%BA%E5%85%AC%E5%BC%80%E3%80%81%E7%A7%81%E6%9C%89pod)这篇文章。

