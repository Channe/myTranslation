<!-- md文件头部 -->
title: "Swift小贴士: 优雅地设置IBOutlets”
date: 2015-8-28
tags: [Swift]
categories: [Natasha The Robot]
permalink: ios-a-beautiful-way-of-styling-iboutlets-in-swift

<!-- iOS: A Beautiful Way of Styling IBOutlets in Swift -->
#Swift小贴士: 优雅地设置IBOutlet

> 原文链接: [iOS: A Beautiful Way of Styling IBOutlets in Swift](http://natashatherobot.com/ios-a-beautiful-way-of-styling-iboutlets-in-swift/)
> 
> - 原文日期: 2015/07/29
> - 译文日期: 2015/08/28,译者:[Channe](http://www.jianshu.com/users/7a07113a6597/latest_articles)

<!-- This morning I saw a beautiful tweet from @jesse_squires: -->
早上我看到[@jesse_squires](https://twitter.com/jesse_squires/)发了个好推:

<!-- #Swift tip: Use didSet on your IBOutlets to configure views instead of cramming code into viewDidLoad. Much cleaner. Still called only once. 
-- Jesse Squires (@jesse_squires) July 29, 2015 -->
> [#Swift](https://twitter.com/hashtag/Swift?src=hash)小贴士: 在`IBOutlets`的`didSet`中设置视图,而不是将代码塞满`viewDidLoad`. 更清晰,同样只被调用一次.
>
> -- Jesse Squires (@jesse_squires) 2015年7月29日.

<!-- Settings colors, fonts, and accessibility for UI elements in apps is always in pain. Ideally this would happen in the storyboard, but color management in the storyboard is pretty horrible (one way to mitigate this is through an Xcode Color Palette), and more advanced accessibility stuff can’t even be done in the storyboard. -->
设置App界面元素的颜色,字体和辅助功能总是很痛苦. 理想情况下,storyboard能搞定,但是storyboard中的颜色管理相当糟糕(可以用[Xcode调色板](http://natashatherobot.com/xcode-color-palette/)减轻这种痛苦),更糟糕的是,比之更高级的辅助功能选项并不能在storyboard中设置.

<!-- So I personally prefer to do this in code – much easier to see where all the colors / font / accessibility / etc changes need to be made when the app is re-designed. I often see this translated into a super long viewDidLoad as Jesse mentions, which I try to extract into one or more functions in private extension in Swift like this: -->
因此我个人更喜欢在代码中设置,这样在App重新设计时更容易看到所有颜色/字体/辅助功能等的变化. 我经常看到Jesse说的那种超长viewDidLoad方法, 我试图把它们提取到一个或多个私有扩展中:

```swift
import UIKit

class ViewController: UIViewController {

    @IBOutlet weak var myLabel: UILabel!
    @IBOutlet weak var myOtherLabel: UILabel!
    @IBOutlet weak var myButton: UIButton!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 提取到私有方法中,让viewDidLoad更短
        configureStyling()
    }
}

// MARK: 界面样式
private extension ViewController {
    
    func configureStyling() {
        myLabel.textColor = UIColor.purpleColor()
        myOtherLabel.textColor = UIColor.yellowColor()
        myButton.tintColor = UIColor.magentaColor()
    }
}
```
<!-- But I really love the readability and simplicity of Jesse’s solution: -->
然而,我真的很喜欢Jesse方案的可读性和简洁性:

```swift
import UIKit

class ViewController: UIViewController {

    @IBOutlet weak var myLabel: UILabel! {
        didSet {
            myLabel.textColor = UIColor.purpleColor()
        }
    }
    
    @IBOutlet weak var myOtherLabel: UILabel! {
        didSet {
            myOtherLabel.textColor = UIColor.yellowColor()
        }
    }
    
    @IBOutlet weak var myButton: UIButton! {
        didSet {
            myButton.tintColor = UIColor.magentaColor()
        }
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```
<!-- Time to refactor! -->
是时候去重构代码了!

