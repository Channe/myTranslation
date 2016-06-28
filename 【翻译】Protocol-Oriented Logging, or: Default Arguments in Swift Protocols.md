# Protocol-Oriented Logging, or: Default Arguments in Swift Protocols
**Swift 2.2 不允许在协议声明时提供默认参数。如果你想用协议抽象出 App 中的日志代码，这就是个问题。因为默认参数常常用来传递源代码位置给日志函数。然而，你可以在协议扩展中使用默认参数，这是一个变通方案。**

一个典型的[日志](https://en.wikipedia.org/wiki/Logfile)消息应该包括日志事件的源代码位置（文件名、行号和可能的函数名）。Swift 为此提供了 `#file`，`#line`，`#column` 和 `#function` [调试标识](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Expressions.html#//apple_ref/doc/uid/TP40014097-CH32-ID390)。在编译时，解析器会展开这些占位符为字符串或整数文本用来描述当前源代码位置。如果我们在每个调用日志函数时都包含这些参数，那重复的次数就太多了，所以它们通常都是作为默认参数传递。这里之所以可行是因为编译器足够聪明，能够在计算默认参数列表时将调试标识展开为[函数调用处](https://en.wikipedia.org/wiki/Call_site)。标准库中的[assert](http://swiftdoc.org/v2.2/func/assert/#func-assert_-bool_-string-file_-staticstring-line_-uint)函数就是[一个例子](https://developer.apple.com/swift/blog/?id=15)，它被这样声明：

```swift
func assert(
    @autoclosure condition: () -> Bool,
    @autoclosure _ message: () -> String = default,
    file: StaticString = #file,
    line: UInt = #line)
```

第三个和第四个参数默认展开为调用者源代码的位置。（如果你对 `@autoclosure` 属性有疑问，它把一个表达式包裹为一个闭包，没有要求调用者使用明确的闭包表达式，有效地将表达式的计算从调用处延迟到函数体。`assert` 只在调试构建时使用它来执行 condition  参数的计算（可能代价高昂或者有副作用），同时只在断言失败时才计算 message 参数。）

### 一个简单、全局的日志函数

你可以使用同样的方法来写一个日志函数来产生日志信息和日志级别。它的接口和实现可能会像这样：

```swift
enum LogLevel: Int {
    case verbose = 1
    case debug = 2
    case info = 3
    case warning = 4
    case error = 5
}

func log(
    logLevel: LogLevel,
    @autoclosure _ message: () -> String,
    file: StaticString = #file,
    line: Int = #line,
    function: StaticString = #function)
{
    // 使用 `print` 打印日志
    // 此时不用考虑 `logLevel`
    print("\(logLevel) – \(file):\(line) – \(function) – \(message())")
}
```

你可能主张使用另一种方法，而不是像这里一样将 message 参数声明为 `@autoclosure`。这个属性在这个简单的例子里并没有益处，因为 message 参数无论什么情况都会计算。但是，我们将会在下一步改变这一点。

### 具体类型

为了代替全局的日志函数，我们创建一种叫做 `PrintLogger` 的类型，它用最小日志级别初始化。它只会记录日志级别至少为最小的事件。`LogLevel` 为此需要 `Comparable` 协议，这是为什么我之前把它声明为 `Int` 型来存储原始数据的原因：

```swift
extension LogLevel: Comparable {}

func <(lhs: LogLevel, rhs: LogLevel) -> Bool {
    return lhs.rawValue < rhs.rawValue
}

struct PrintLogger {
    let minimumLogLevel: LogLevel

    func log(
        logLevel: LogLevel,
        @autoclosure _ message: () -> String,
        file: StaticString = #file,
        line: Int = #line,
        function: StaticString = #function)
    {
        if logLevel >= minimumLogLevel {
            print("\(logLevel) – \(file):\(line) – \(function) – \(message())")
        }
    }
}
```

你将会这样使用 `PrintLogger `：
```swift
let logger = PrintLogger(
    minimumLogLevel: .warning)
logger.log(.error, "This is an error log")
    // 获取日志
logger.log(.debug, "This is a debug log")
    // 啥也没做
```

### 带默认参数的协议

下一步，我将会创建一个 `Logger ` 协议作为 `PrintLogger ` 的抽象。这将允许我以后使用高级实现替换这个使用 print 语句的简单日志，比如记录日志到文件或者发送日志给服务器。然而，我在这碰到了障碍，因为 Swift 不允许在协议声明时提供默认参数。下面的代码无法编译通过：

```swift
protocol Logger {
    func log(
        logLevel: LogLevel,
        @autoclosure _ message: () -> String,
        file: StaticString = #file,
        line: Int = #line,
        function: StaticString = #function)
    // 错误: 协议方法中不允许默认参数
}
```


