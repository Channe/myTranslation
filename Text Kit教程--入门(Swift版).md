<!-- md文件头部 -->
title: "Text Kit教程:入门(Swift版)”
date: 2015-8-23
tags: [Swift]
categories: [Swift]
permalink: text-kit-tutorial-swift

<!-- Text Kit Tutorial: Getting Started -->
#Text Kit教程:入门(Swift版)

> raywenderlich原文链接: [Text Kit Tutorial: Getting Started](http://www.raywenderlich.com/77092/text-kit-tutorial-swift)
> 
> - 原文2014/12/12更新,适用于Xcode6.1.1
> - 译文2015/08/23翻译,适用于Xcode6.4 (原文代码稍有改动,以适配Swift 1.2)

<!-- The way that iOS renders text continues to grow more powerful over the years as Apple adds more features and capabilities. The release of iOS 7 brought with it some of the most significant text rendering changes yet. Now iOS 8 builds on that power, and makes it easier to use. A brief overview of iOS text editing might help you keep things in perspective. -->
`iOS`绘制文本的方式在持续变强,因为苹果多年来一直在增加特性和功能.`iOS7`的发布带来了文本绘制的一些重大变化.现在的`iOS8`在此基础上让它更加易用.一个简洁的概述能让你大致了解`iOS`的文本编辑功能.

<!--more-->

<!-- In the old days before iOS 6, web views were usually the easiest way to render text with mixed styling, such as bold, italics, or even colors.
In 2012, iOS 6 added attributed string support to a number of UIKit controls. This made it much easier to achieve this type of layout without resorting to rendered HTML.
In iOS 6, UIKit controls based their text capabilities on both WebKit and Core Graphics’ string drawing functions, as illustrated in the hierarchical diagram below: -->
在`iOS6`以前,web视图通常是绘制加粗/斜体/颜色等混合样式最简单的方式.
2012年,`iOS6`在一些`UIKit`控件中增加了属性字符串.这使得完成那些没有渲染为`HTML`的布局更加容易.
`iOS6`中,`UIKit`控件的文本功能建立在`WebKit`和`Core Graphics`的字符串绘图函数上,如下面的分层图所示:
![iOS6 Text](http://cdn4.raywenderlich.com/wp-content/uploads/2013/09/TextRenderingArchitecture-iOS6.png)

<!-- Note: Does anything strike you as odd in this diagram? That’s right — UITextView uses WebKit under the hood. iOS 6 renders attributed strings on a text views as HTML, a fact that’s not readily apparent to developers who haven’t dug deeply into the framework. -->
> **注意:**图中有没有一些古怪的东西冲击到你? 这就对了--**`UITextView`**的底层是`WebKit`. `iOS6`将属性字符串当做`HTML`绘制到文本视图上, 这个事实对于那些没有深挖框架的开发者来说并不显见.

<!-- However, since iOS 7 there’s an easier way. With the current minimalistic design focus that eschews ornamentation and focuses more on typography — such as the UIButton that strips away all borders and shadows, leaving only text — it’s no surprise that iOS 7 added a whole new framework for working with text and text attributes: Text Kit. -->
然而, 从iOS7开始, 有了更简单的方法. 使用当前简约的设计避免装饰,更多的聚焦于排版,比如UIButton去掉所有的边框和阴影,只留下文字. 不足为奇的是iOS7增加了一整个全新的框架用来处理文本和文本属性: **`Text Kit`**.???

<!-- The architecture is now much tidier; all of the text-based UIKit controls (apart from UIWebView) now use Text Kit, as shown in the following diagram: -->
现在架构更整洁了. 所有基于文本的UIKit控件(除了UIWebView)都使用Text Kit,如下图所示:
![iOS7 Text Kit](http://cdn3.raywenderlich.com/wp-content/uploads/2013/09/TextRenderingArchitecture-iOS7.png)

<!-- Text Kit is built on top of Core Text, inherits the full power of the Core Text framework, and to the delight of developers everywhere, wraps it in an improved object-oriented API.
In this Text Kit tutorial you’ll explore the various features of Text Kit as you create a simple yet feature-rich note-taking app for the iPhone that features reflowing text, dynamic text resizing, and on-the-fly text styling.
Ready to create something of note? :] Then read on to get started with Text Kit! -->
`Text Kit`建立在`Core Text`上面, 继承了`Core Text`框架的所有功能, 同时无处不让开发者高兴是, 将它包裹了一层改进的面向对象API.
在这个`Text Kit`教程里, 随着你创建一个简约但功能丰富的iPhone版记笔记app,你将探索`Text Kit`各方面的特性. 这些特性有reflowing text/动态文本缩放和on-the-fly文本风格.???
准备创建注意中的东西? 那么读完就开始学习`Text Kit`吧!

<!-- Note: At the time of writing this tutorial, our understanding is we cannot post screenshots of iOS 8 since it is still in beta. All the screenshots here are from iOS 7, which should look very close to what things will look like in iOS 8. -->
> **注意:**写本教程时,`iOS8`仍然是测试版,依照协议我们[不能发表iOS8的截屏](http://www.raywenderlich.com/?p=74138). 这里所有的截屏都来自`iOS7`, 它们看起来和`iOS8`中非常相似.

<!-- Getting Started -->
##入门
<!-- This tutorial includes a starter project with the user interface for the app pre-created so you can stay focused on Text Kit. Download the starter project archive to get started. Unzip the file, open the project in Xcode and build and run the app. It will look like the following: -->
本教程包含一个起始项目,它有创建好的用户界面,以便于专心学习`Text Kit`. [下载这个起始项目文件](http://cdn2.raywenderlich.com/wp-content/uploads/2014/09/SwiftTextKitNotepad-starter6.zip)后开始学习吧. 解压文件,在`Xcode`中打开项目,同时编译/运行这个App. 
>**译者注:**在`Xcode6.4`中,需要做如下修改方可运行:
>
>- 1,修改`AppDelegate.swift`文件第16行为:`func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool`; 
>- 2,修改`NotesListViewController.swift`文件第32行的`as`为`as!`;
>- 3,删除`NoteEditorViewController.swift`文件第22行的`!`号;

它看起来像下面这样:
![](http://cdn5.raywenderlich.com/wp-content/uploads/2014/08/newStarterImage.png)

<!-- The app creates an initial array of Note instances and renders them in a table view controller. Storyboards and segues detect cell selection in the table view and handle the transition to the view controller where users can edit the selected note. -->
这个App创建了一个`Note`实例的初始数组,并在表视图控制器中绘制它们. 故事板(Storyboard)和联线(segue)检测表视图cell的选中状态, 然后过渡到另一个能编辑和选中笔记的视图控制器.
<!-- Note: If you are new to Storyboards, check out Storyboards Tutorial in iOS 7. And if you’re subscribing to the video tutorials check out Video Tutorial Storyboards and Segues -->
> **注意:** 如果你没有用过故事板, 那学习[iOS7故事板教程](http://www.raywenderlich.com/?p=50308). 要是你订阅了视频教程,那学习[故事板和联线视频教程](http://www.raywenderlich.com/64419/video-tutorial-storyboards-segue).

<!-- Browse through the source code and play with the app a little to get a feel for the apps’ structure and how it functions. When you’re done with that, move on to the next section, which discusses the use of dynamic type in your app. -->
浏览和运行源代码能感受到这个App的结构和运行方式. 做完了这个,开始下一部分吧, 论述App中动态字体的使用.

<!-- Dynamic Type -->
##动态字体

<!-- Dynamic type is one of the most game-changing features of iOS 7; it places the onus on your app to conform to user-selected font sizes and weights.
In iOS 7, open the Settings app and navigate to General/Accessibility and General/Text Size to view the settings that affect how the app displays text: 
In iOS 8, open the Settings app and navigate to General/Accessibility/Larger Text to access Dynamic Type text sizes. -->
动态字体是iOS7中最改变游戏规则的特性之一; 它承担App符合用户选择的字体大小/宽度的责任.
iOS7中,打开设置App导航到通用/辅助功能和通用/文本大小,在这能看到影响App显示文本的设置(如下图),iOS8中,打开设置App导航到通用/辅助功能/更大字体能看到动态字体文本大小.
![iOS7中影响App显示文本的设置](http://cdn5.raywenderlich.com/wp-content/uploads/2013/09/UserTextPreferences.png)

<!-- iOS 7 offered the ability to enhance the legibility of text by increasing font weight, as well as an option to set the preferred font size for apps that support dynamic text. -->
iOS7可通过加粗字体来增强文本可读性,也有选项用来设置支持动态字体功能的App的首选字体大小.

<!-- Note: When Apple released Text Kit at WWDC 2013, they strongly encouraged developers to adopt Dynamic Type. At WWDC 2014 Apple went a bit further. They stressed that all built-in apps support Dynamic Type. Moreover, they made Dynamic Type much easier to use in iOS 8. While they didn’t threaten to break anyone’s legs if they didn’t get with the program, Apple did make their point forcefully. The WWDC 2014 session 226 What’s New in Tables and Collection Views, covers iOS 8 Dynamic Type support for table views and collections. You’d be well-advised to watch it! Users will expect apps written for iOS 7 and later to honor these settings, so ignore them at your own risk! -->
> **注意:** 苹果在WWDC 2013上发布Text Kit时,强烈鼓励开发者适配动态字体. WWDC 2014时,苹果又进了一步. 他们强调所有自带App都支持动态字体. 而且,他们让动态字体在iOS8中更加易用. 虽然苹果并没有威胁说不适配就打断谁的腿, 但是他们非常强调他们的看法. 在"WWDC 2014 session 226 表和集合视图的新内容"里,包含了iOS8动态字体支持表和集合视图的内容. 非常建议你观看这个视频! 用户会很期待为iOS7+开发的App能实现这些功能, 所以不要冒险忽视他们.

<!-- In order to make use of dynamic type you need to specify fonts using styles rather than explicitly stating the font name and size. iOS 7 added a new method to UIFont, preferredFontForTextStyle, that creates a font for the given style using the user’s font preferences.
The diagram below gives an example of each of the six different font styles: -->
为了使用动态字体,你需要指定的是字体样式而不是字体名称和大小. `iOS7`给`UIFont`增加了一个新方法,`preferredFontForTextStyle`,它为用户字体设置中给定的样式创建字体.
下图展示了6种不同的字体样式:
![6种不同的字体样式](http://cdn5.raywenderlich.com/wp-content/uploads/2013/09/TextStyles.png)
<!-- The text on the left uses the smallest user selectable text size, the text in the center uses the largest, and the text on the right shows the effect of enabling the accessibility bold text feature. -->
左边的文本是用户可选的最小字体大小,中间的文本是最大,右边的文本展示了辅助功能中加粗字体的效果.

<!-- Basic Support -->
###基本支持

<!-- Implementing basic support for dynamic text is relatively straightforward. Rather than using explicit fonts within your application, you instead request a font for a specific style. At runtime the app selects a suitable font based on the given style and the user’s text preferences. -->
实现动态字体的基本支持比较简单. 与其在App中使用明确的字体,不如请求特定样式的字体. App在运行时根据给定的样式和用户的文本设置来选择合适的字体.

<!-- With iOS 8, Apple made it far easier to implement Dynamic Type than was the case in iOS 7. In particular, default labels in table views support Dynamic Type automatically! Nonetheless, you may well want to support iOS 7, and/or you might want to use custom labels in your table views. So first you’ll learn how to handle Dynamic Type for iOS 7. Then you’ll discover how Apple makes your life even easier in iOS 8. -->
iOS8中,苹果让我们实现动态字体功能比iOS7中更加简单. 尤其是表视图中的默认label自动支持动态字体! 虽然如此,你可能想兼容iOS7,并且/或者你想在表视图中使用自定义label. 所以,你首先要学习如何在iOS7中处理动态字体,然后你会发现在iOS8中苹果如何让其变得更简单.

<!-- Why iOS 7 is Great, but iOS 8 is Even Greater -->
###为什么iOS7是伟大的,但iOS8更伟大

<!-- The starter project comes set with its deployment set to iOS 8. Before proceeding, build and run the app and try changing the default text size to various values. You will discover that both the text size and cell height in the table view list of notes changes accordingly. And you didn’t have to do a thing! But do observe also that the notes themselves do not reflect changes to the text size settings. -->
起始项目设置的部署平台是iOS8. 开始前,构建并运行这个App并试着多次改变默认字体大小. 你将发现表视图里笔记列表的文本大小和单元格高度会相应的改变. 而且,你不需要做什么! 但是, 确实也能看到笔记内容并没有反映文本大小设置的更改.

<!-- While not quite as wonderful, things are still pretty great in iOS 7. For most of this tutorial, it won’t matter if you’re using iOS 7 or iOS 8 (just be sure you’re using Xcode 6!), but for now, set the deployment level for the app to iOS 7 and follow along in the iOS simulator. And most of what follows is of use in iOS 8 too, so it’s worthwhile even if you do not plan to support iOS versions earlier than iOS 8. -->
尽管不十分完美,但在iOS7中表现的也不错. 本教程大部分情况下不需要考虑使用的是iOS7还是iOS8(只需确保使用Xcode6!),但是现在,把部署平台改为iOS7并使用相应的iOS模拟器. 大部分情况下也可用于iOS8, 因此甚至值得你不打算兼容iOS8之前的iOS版本.

<!-- Note: To set the deployment level to iOS 7 in Xcode 6, choose View/Navigators/Show Project Navigator. In the right hand panel, choose the project, click on info and select iOS 7 in the iOS Deployment Target popup menu. Also, select the Target in the right-hand panel and the set the Deployment Target to iOS 7.
It’s also important to make sure the simulator is acting as an iOS 7 device. So in the iOS Simulator choose Hardware/Device/iOS 7/iPhone 5s. -->
> **注意:** 在Xcode6中设置部署平台为iOS7,选择`View/Navigators/Show Project Navigator`. 在右手边的控制面板中,选择`project`,点击`info`,在`iOS Deployment Target`弹出菜单中选择`iOS7`. 确保模拟器是`iOS7`设备也是很重要的. 在`iOS Simulator`中选择`Hardware/Device/iOS7/iPhone 5s`.

<!-- Now that you’re all set to run as an iOS 7 app, go ahead and build and run. Play with the text size settings as before and you’ll see that sadly enough, the app is ignoring your settings. Now, you will do something to make it work in iOS 7. -->
现在你搞好了运行iOS7 App的所有设置,先构建和运行一下. 设置玩字体大小后你将非常失望, App忽略了那些设置. 现在,你要为它在iOS7中正确运行做点事.

<!-- Open NoteEditorViewController.swift and add the following to the end of viewDidLoad:
textView.font = UIFont.preferredFontForTextStyle(UIFontTextStyleBody)
Notice you’re not specifying an exact font such as Helvetica Neue. Instead, you’re asking for an appropriate font for body text with the UIFontTextStyleBody text style constant. -->
打开`NoteEditorViewController.swift`,在`viewDidLoad`最后面添加这些代码:
```swift
textView.font = UIFont.preferredFontForTextStyle(UIFontTextStyleBody)
```
这里并没有明确一种诸如Helvetica Neue的字体. 取而代之的是,用`UIFontTextStyleBody`文本样式常量请求了适合body文本的字体

<!-- Next, open NotesListViewController.swift and add the following to the tableView(_:cellForRowAtIndexPath:) method, just before the return call:
cell.textLabel?.font = UIFont.preferredFontForTextStyle(UIFontTextStyleHeadline)
Again, you’re specifying a text style and iOS will return an appropriate font. -->
下一步,打开`NotesListViewController.swift`,添加下面代码到`tableView(_:cellForRowAtIndexPath:)`方法的return语句前:
```swift
cell.textLabel?.font = UIFont.preferredFontForTextStyle(UIFontTextStyleHeadline)
```
再次,这里指定文本样式,iOS将返回合适的字体.

<!-- Using a semantic approach to font names, such as UIFontTextStyleSubHeadline, helps avoid hard-coded font names and styles throughout your code — and ensures that your app will respond properly to user-defined typography settings as expected.
Build and run the app again, and you’ll notice that the table view and the note screen now honor the current text size; the difference between the two is shown in the screenshots below: -->
用语义获取字体名称,如`UIFontTextStyleSubHeadline`,确保你的App像期望的一样正确地响应用户定义的排版设置,避免硬编码字体名和样式遍布你的代码. 再次构建和运行App,现在注意表视图和笔记屏幕显示正确的字体大小; 下图展示了2种不同显示的截图:
![2种不同显示的截图](http://cdn2.raywenderlich.com/wp-content/uploads/2014/07/NotepadWithDynamicType1.png)

<!-- That looks pretty good — but sharp readers will note that this is only half the solution. Head back to the Settings app under General/Text Size and modify the text size again. This time, switch back to SwiftTextKitNotepad — without re-launching the app — and you’ll notice that your app didn’t respond to the new text size.
Your users won’t take too kindly to that! Looks like that’s the first thing you need to correct in this app. -->
看上去很好,但是眼尖的读者会发现这只是解决方法的一半. 先回到设置App的通用/文字大小再次调整文字大小. 这次,不重新运行应用切换回SwiftTextKitNotepad,你会发现App并不会调整到新的字体大小. 你的用户们不会这样友好! 看起来这就是你需要改正此应用的第一个问题。

<!-- Responding to Updates -->
###响应更新

<!-- Open NoteEditorViewController.swift and add the following code to the end of viewDidLoad -->
打开`NoteEditorViewController.swift`,添加下面代码到`viewDidLoad`最后:
```swift
NSNotificationCenter.defaultCenter().addObserver(self, 
    selector: "preferredContentSizeChanged:", 
    name: UIContentSizeCategoryDidChangeNotification,
    object: nil)
```

<!-- The above code registers the class to receive notifications when the preferred content size changes and passes in the method to be called (preferredContentSizeChanged) when this event occurs.
Next, add the following method to the class: -->
上面的代码给类注册接受通知,当首选内容大小更改事件发生时会接到通知,并调用传递给它的方法(preferredContentSizeChanged).
下一步,给类添加下面的方法:
```
func preferredContentSizeChanged(notification: NSNotification) {
  textView.font = UIFont.preferredFontForTextStyle(UIFontTextStyleBody)
}
```
<!-- This simply sets the text view font to one based on the new preferred size. -->
这里简单的设置文本视图的字体为以新的首选大小为基础的字体.

<!-- Note: You might be wondering why it seems you’re setting the font to the same value it had before. When the user changes their preferred font size, you must request the preferred font again; it won’t update automatically. The font returned via preferredFontForTextStyle will be different when the font preferences are changed. -->
> **注意:** 你可能会奇怪为什么字体设置是和之前一样的值. 当用户改变首选字体大小时, 必须再次请求首选字体,它并不会自动更新. 当字体的选择更改时,通过preferredFontForTextStyle返回的字体会不同.

<!-- Open up NotesListViewController.swift and override the viewDidLoad function by adding the following code to the class: -->
打开NotesListViewController.swift,用下面的代码重写viewDidLoad方法:
```
override func viewDidLoad() {
  super.viewDidLoad()
  NSNotificationCenter.defaultCenter().addObserver(self,
      selector: "preferredContentSizeChanged:", 
      name: UIContentSizeCategoryDidChangeNotification, 
      object: nil)
}
```
<!-- Hey, isn’t that the same code you just added to NoteEditorViewController.swift? Yes, it is — but you’ll handle the preferred font change in a slightly different manner.
Add the following method to the class: -->
哈, 有没有发现这和添加到NoteEditorViewController.swift的代码一模一样? 是的,它们一样. 但是你处理首选字体的更改时有细微的不同.
添加下面的方法到类里:
```
func preferredContentSizeChanged(notification: NSNotification) {
  tableView.reloadData()
}
```
<!-- The above code simply instructs the table view to reload its visible cells, which updates the appearance of each cell. This will trigger the calls to preferredFontForTextStyle() and refresh the font choice.
Build and run your app; change the text size setting, and verify that your app responds correctly to the new user preferences. -->
上面的代码简单的教了刷新表视图的可视单元格, 刷新会更新每个单元格的外观. 这会触发preferredFontForTextStyle()的调用并刷新字体选择. 
构建并运行App; 改变文本大小设置,然后确认App能正确地响应新的用户设置.

<!-- Changing Layout -->
###改变布局

<!-- That part seems to work well, but when you select a really small font size, your table view ends up looking a little sparse, as shown in the right-hand screenshot below: -->
现在似乎不错了,可是在你选择相当小字体大小时,表视图最后看起来有点稀疏,就像下面右手边(原文出错,应该为左手边--译者注)的截图:
![截图](http://cdn4.raywenderlich.com/wp-content/uploads/2013/09/ChangingLayout.png)

<!-- This is one of the trickier aspects of dynamic type (in iOS 7). To ensure your application looks good across the range of font sizes, your layout needs to be responsive to the user’s text settings. Auto Layout solves a lot of problems for you, but this is one problem you’ll have to solve yourself. -->
(在iOS7中)这是动态字体的棘手部分之一. 为了确保应用的字体大小范围看起来不错,应用布局需要响应用户的文本设置. 自动布局(Auto Layout)为你搞定了很多问题,但是这个问题需要你亲自搞定.

<!-- Your table row height needs to change as the font size changes. Implementing the tableView(_:heightForRowAtIndexPath:) delegate method solves this quite nicely.
Add the following code to NotesListViewController.swift, in the section labelled Table view data source: -->
表行高需要同字体大小一起改变. 实现`tableView(_:heightForRowAtIndexPath:) `委托方法能很好的解决这个问题. 把下面的代码添加到NotesListViewController.swift的Table view data source分区标签下:(下面代码稍有改动以适应Swift1.2--译者注)
```
let label: UILabel = {
  let temporaryLabel = UILabel(frame: CGRect(x: 0, y: 0, width: Int.max, height: Int.max))
  temporaryLabel.text = "test"
  return temporaryLabel
}()
 
override func tableView(tableView: UITableView, heightForRowAtIndexPath indexPath: NSIndexPath) -> CGFloat {
  label.font = UIFont.preferredFontForTextStyle(UIFontTextStyleHeadline)
  label.sizeToFit()
  return label.frame.height * 1.7
}
```
<!-- The above code creates a single shared instance of UILabel which the table view uses to calculate the height of the cell. Then, in tableView(_:heightForRowAtIndexPath:) you set the label’s font to be the same font used by the table view cell. It then invokes sizeToFit on the label, which forces the label’s frame to fit tightly around the text, and results in a frame height proportional to the table row height. -->
上面的代码创建了一个UILabel的共用实例变量label,表视图用它来计算单元格的高度. 接着, 在`tableView(_:heightForRowAtIndexPath:)`里设置该label的字体为表视图单元用的字体. 然后调用label的`sizeToFit`方法,它能强制label的frame紧紧的贴合文本,最后frame的高度乘以一定比例返回给表视图高度.

<!-- Build and run your app; modify the text size setting once more and the table rows now size dynamically to fit the text size, as shown in the screenshot below: -->
构建并运行App; 再一次调整文本大小,现在表格的行能动态适应文本大小了,见下面的截图:
![截图](http://cdn3.raywenderlich.com/wp-content/uploads/2014/07/TableViewAdaptsHeights.png)

<!-- If you like, you may now reset the deployment to iOS 8 for the rest of the tutorial. -->
如果你喜欢,接下来的教程你可以重置部署平台为iOS8.

<!-- Letterpress Effect -->
###凸版印刷效果

<!-- The letterpress effect adds subtle shading and highlights to text that give it a sense of depth — much like the text has been slightly pressed into the screen. -->
凸版印刷效果是给文字添加细微的阴影和高亮以获得一种深度的感觉--仿佛文字是直接压进了屏幕上.

<!-- Note: The term “letterpress” is a nod to early printing presses, which inked a set of letters carved on blocks and pressed them into the page. The letters often left a small indentation on the page — an unintended but visually pleasing effect, which is frequently replicated in digital typography today. -->
> **注意:** "凸版印刷(letterpress)"一词???

<!-- Open NotesListViewController.swift and replace tableView(_:cellForRowAtIndexPath:) with the following implementation: -->
打开`NotesListViewController.swift`,用下面的实现替换`tableView(_:cellForRowAtIndexPath:)`:
```
override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
  let cell = tableView.dequeueReusableCellWithIdentifier("Cell", forIndexPath: indexPath) as! UITableViewCell
 
  let note = notes[indexPath.row]
  let font = UIFont.preferredFontForTextStyle(UIFontTextStyleHeadline)
  let textColor = UIColor(red: 0.175, green: 0.458, blue: 0.831, alpha: 1)
  let attributes = [
    NSForegroundColorAttributeName : textColor,
    NSFontAttributeName : font,
    NSTextEffectAttributeName : NSTextEffectLetterpressStyle
  ]
  let attributedString = NSAttributedString(string: note.title, attributes: attributes)
 
  cell.textLabel?.attributedText = attributedString
 
  return cell
}
```
<!-- The above code creates an attributed string for the title of a table cell using the letterpress style.
Build and run your app; your table view will now display the text with a nice letterpress effect, as shown below: -->
上面的代码创建了一个凸版印刷样式的属性字符串用作表单元格的标题.
构建并运行App; 现在表视图展示的文本有不错的凸版印刷效果,就像下面这样:
![截图](http://cdn5.raywenderlich.com/wp-content/uploads/2014/07/Letterpress.png)

<!-- Letterpress is a subtle effect — but that doesn’t mean you should overuse it! Visual effects may make your text more interesting, but they don’t necessarily make your text more legible. -->
凸版印刷是一种细微的效果--但不意味着你可以过渡使用! 可视效果也许能让你的文字更加有趣,但是它并不能让文字更易读.

<!-- Exclusion Paths -->
###排除路径(Exclusion Paths)

<!-- Flowing text around images or other objects is a standard feature of most word processors. Text Kit allows you to render text around complex paths and shapes with exclusion paths.
It would be handy to tell the user the note’s creation date; you’re going to add a small curved view to the top right-hand corner of the note that shows this information.
You’ll start by adding the view itself – then you’ll create an exclusion path to make the text wrap around it. -->
文字平滑地环绕在图片或其他对象周围是大部分文字处理工具的标准功能. Text Kit允许你围绕复杂的路径和排除路径表示的图形绘制文本. 
告诉用户笔记的创建日期,这会很方便; 你打算在笔记的中间偏右上角添加一个小的弯曲视图显示日期信息.
你要先添加视图本身--然后创建排除路径以使文本环绕在其周围。

<!-- Adding the View -->
####添加视图

<!-- Open up NoteEditorViewController.swift and add the following property declaration to the class: -->

```
var timeView: TimeIndicatorView!
```
<!-- As the name suggests, this houses the time indicator subview.
Next, add this code to the very end of viewDidLoad: -->

```
timeView = TimeIndicatorView(date: note.timestamp)
textView.addSubview(timeView)
```
<!-- This simply creates an instance of the new view and adds it as a subview.
TimeIndicatorView calculates its own size, but it won’t do this automatically. You need a mechanism to call updateSize when the view controller lays out the subviews.
Finally, add the following two methods to the class: -->

```
override func viewDidLayoutSubviews() {
  updateTimeIndicatorFrame()
}
 
func updateTimeIndicatorFrame() {
  timeView.updateSize()
  timeView.frame = CGRectOffset(timeView.frame, textView.frame.width - timeView.frame.width, 0)
}
```
<!-- viewDidLayoutSubviews calls updateTimeIndicatorFrame, which does two things: it calls updateSize to set the size of the subview, and positions the subview in the top right corner of the text view.
All that’s left is to call updateTimeIndicatorFrame when your view controller receives a notification that the size of the content has changed. Replace the implementation of preferredContentSizeChanged to the following: -->

```
func preferredContentSizeChanged(notification: NSNotification) {
  textView.font = UIFont.preferredFontForTextStyle(UIFontTextStyleBody)
  updateTimeIndicatorFrame()
}
```
<!-- Build and run your project; tap on a list item and the time indicator view will display in the top right hand corner of the item view, as shown below: -->

![截图](http://cdn5.raywenderlich.com/wp-content/uploads/2014/07/TimIndicator.png)
<!-- Modify the device Text Size preferences, and the view will automatically adjust to fit.
However, something doesn’t look quite right. The text of the note renders behind the time indicator view instead of flowing neatly around it. Fortunately, this is the exact problem that exclusion paths are designed to solve. -->

<!-- Exclusion Paths -->
####排除路径

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->

<!--  -->



