title: "Swift 编程思想，第一部分（补遗）：牺牲小马"
date: 2016-02-24
tags: []
categories: []
permalink:thinking-in-swift-1-addendum

---
原文链接=http://alisoftware.github.io/swift/2015/09/14/thinking-in-swift-1-addendum/
作者=GABRIEL THEODOROPOULOS
原文日期=2015/12/14
译者=Channe
校对=
定稿=
发布时间=

<!--此处开始正文-->
在发表[我的系列文章《Swift 编程思想》第一部分](http://swift.gg/2015/09/29/thinking-in-swift-1/)之后，我在Twitter上收到一些不错的反馈。现在，我在那些评论的基础上谈一谈什么时候可以使用*!*来牺牲小马。

#绝不杀死小马吗？
在我上篇文章里，我强烈建议不要使用`!`。有时候读起来好像是“永远不要使用它”。事实上我的意思是《每次添加`!`不能仅仅是为了方便编译器，这样是在杀死小马 🐴》。可是呢，如果你明确的知道你的代码在做什么，那小马🐴依旧能活下来。不要不加思索的这样做，仅仅是为了方便编译器。

#什么时候牺牲小马？
这里有几个在 Swift 中可以使用`!`地方的例子。它们可能不仅是使用`!`的案例，而且我觉得是绝大部分小马规则例外的地方。

##1. IBOutlets + 依赖注入
`IBOutlets` --- 和你做的那些依赖注入的属性一样 --- 是一个特殊的例子，因为它们将会一直是nil在init方法结束时（因为它们不能马上被初始化），但是它们在实例初始化后将迅速获得一个值（不管是通过依赖注入还是加载XIB）。

所以即便它们不得不是可选的（因为它们在init过程中不会被初始化，仅仅是在那一小会之后），在设计上，能十分肯定，它们在代码的其他部分绝不会是nil。这是一个隐含的例子--解包很方便，因为知道它们绝不会在使用它们的时候为nil。

##2. UIImage，UIStoryboard，UITableViewCell
下面的方法都是使用名称和标识符这些常量来初始化的：
```swift
UIImage(named:...)
UIStoryboard(name:,bundle:)
UIStoryboard.instantiateViewControllerWithIdentifier(_:)
UITableView.dequeueReusableCellWithIdentifier(_:, forIndexPath:)
// and some other similar stuff...
```
这里面的每个例子，如果姓名或标识符拼写错误都会导致代码崩溃，因为这是一个开发错误（由于bundle中缺失图片，或者target中错误包含storyboard导致的），也许希望在开发时能够尽可能早的发现这些错误。
注意这些例子，有些情况更愿意使用guard let else fatalError()模式，这能在开发时触发错误后提供更精确的异常消息：
```swift
guard let cell = tableView.dequeueReusableCellWithIdentifier("foo", forIndexPath:indexPath) as? MyCustomCell else {
  fatalError("cell with identifier 'foo' and class 'MyCustomCell' not found. "
    + "Check that your XIB/Storyboard is configured properly.")
}
```
这也是为什么会希望使用一些小技巧，比如我在文章《[enums as constants](http://alisoftware.github.io/swift/enum/constants/2015/07/19/enums-as-constants/)》中讨论过的，和将希望使用[SwiftGen](https://github.com/AliSoftware/SwiftGen)来避免可能的标识符拼写错误和崩溃。
例如[`SwiftGen`的实现里实际上使用了`UIImage(named: asset.rawValue)!`](https://github.com/AliSoftware/SwiftGen#generated-code)，因为枚举是通过SwiftGen工具生成的，因而它能保证图片的名字在Assets Catalog中存在 -- 所以在设计上不会是nil。

#一些特殊场景下该牺牲小马吗？
这个系列文章的第一部分是面向那些Swift新手，因为他们倾向于所有情况下都强制解包，仅仅是因为Xcode告诉他们这样做，或者是因为他们真的不知道为什么可选类型更适合他们。这是为什么我不说那些对于我的目标公众来说感觉更高级的例子。

所以，有的例子有使用`!`的意义。但是，我在第一部分中的建议依旧保留：仅仅在真的明白为什么使用`!`的情况下使用`!`，同时先考虑，同时不能只是方便编译器。特别地，如果你是一个Swift新手，同时你想添加一个`!`仅仅因为“那些该死的可选类型充满错误，希望编译器能够停止抱怨”，你就做错了，这是在开始杀死小马们。

在本系列文章的接下来的部分再见，探讨map，flatMap，??，可选链，以及避免数据不一致。

> 1. 在这里插入一个“强制解绑所有东西”文化 -- 这是我太赖了以至于不能创建和包含。
