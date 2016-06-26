# 第七章 iOS 网络通讯

几乎所有的应用程序或多或少都会用到 iOS 网络 API 接口。 URL 加载系统从抽象层面可划分为 `Foundation NSStream API` 接口和 `Core Foundation CFStream API` 接口。它用于获取和处理数据，如网络资源或文件，前提涉及 URL 链接。至于 `NSStream` 和 `CFStream` 这两个类提供了较为底层的方法来处理网络连接，但是未涉及套接字的范畴。它们既可用于那些基于非 HTTP 协议的通信，又适用需要对网络行为进行直接控制的情况。

本章中，我将从高级 API 接口入手，详细讨论有关 iOS 中的网络。在大多数情况下，应用程序只需与高级 API 接口打交道足以，但有时候这些 API 接口却无法满足我们的需求。而使用底层 API 接口，你往往需要考虑更多潜在的陷阱。

## 7.1 使用 iOS 自带的 URL 加载系统

URL 加载系统能够帮助应用处理大部分需要执行的网络任务。其主要方法是通过构建一个 `NSURLRequest` 对象与 URL API 进行交互，并通过它来实例化一个 `NSURLConnection` 对象，配合委托对象接收请求响应。一旦接收到完整的响应，委托对象就会收到 `connection:didReceiveResponse` 的消息，其中方法传入参数包含[`NSURLResponse` 对象](https://developer.apple.com/DOCUMENTATION/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.pdf)。								`

不过并非所有人使用 URL 加载系统都游刃有余，所以在本节中，我将首先向你演示如何绕过应用程序的安全传输层协议。紧接着，你会学习如何通过证书进行端点的身份验证，从而避免开放式重定向的危险，同时实现证书锁定以限制应用信任的证书总数。

### 7.1.1 正确使用安全传输层协议	

传输层安全协议（Transport Layer Security ，简称TLS）作为取代安全套接字（SSL）的现代规范，几乎对所有网络应用程序的安全性都至关重要。保证正确使用的前提下，TLS 在数据发送端与已验证的远程端点之间一直处于保密连接，同时还应确保远程端点反馈的证书是由受信任的证书颁发机构签署。默认情况下，iOS 设备都是这么实施的，它会拒绝所有不受信任或无效证书的端点连接。不过这对于各式各样的应用程序以及移动端交互过于频繁，因此开发者明确禁用了 TLS/SSL 端点验证，允许应用程序的流量被攻击者网络拦截。

 iOS 中， TLS 提供了多种禁用方式。早前，开发者习惯调用 `NSURLRequest` 提供的私有方法（文档并未给出）：`					setAllowsAnyHTTPSCertificate` 轻松禁用验证。不过随后，苹果公司雷厉风行地在应用审核中拒绝任何使用了该方法的应用程序，理由自然是调用私有 API 接口不符合规范。尽管如此，目前依旧有一些糊弄手段可以调用该 API 接口同时能够绕过苹果的审核过程，只要确保检查代码库不调用该方法即可。 									

此外还有一个更“丧心病狂”的方式绕过 TLS 验证，它将（可能）让你的应用永久性被拒绝掉，但同时它也说明了有关范畴（category）的重要性。曾经我做过的一个客户端项目，需要用到一个相当简单的第三方框架，因此把它添加到产品中。尽管一如既往地在项目中正确处理了 TLS，但要知道更新后的第三方代码版本并未验证任何 TLS 连接。显然，第三方代码提供者为 `NSURLRequest` 实现了一个范畴，凭借 `					allowsAnyHTTPSCertificateForHost` 方法绕过验证。该范畴仅包含那些返回 `YES` 的指令；导致所有 `NSURLRequests` 悄悄忽略了不受信任的证书。这里不得不提及原则：实践是检验真理的唯一标准，请不要武断臆测！此外，你必须审核第三方代码以及其余你自己的代码。可能并非是你导致的错误，但是没人会在意你的借口。

>注意：值得庆幸的是，iOS 9 中默认情况下故意禁用 TLS 连接变得相当困难，因为它不允许应用程序使用非 TLS 连接。另一方面，开发人员被要求在应用程序的 Info.plist 文件中加入明文 HTTP 访问 URL 时的特定异常。然而，这并不意味着能够杜绝那些故意禁用 TLS 保护的事件发生。

目前，实际上存在一个官方的 API 接口用来绕过 TLS 验证。你可以实例化一个 `NSURLConnection` 的委托对象，遵循 [`NSURLConnectionDelegate` 协议](https://developer.apple.com/library/mac/#documentation/Foundation/Reference/ NSURLConnectionDelegate_Protocol)。该委托对象必须实现 `willSendRequestForAuthenticationChallenge` 方法，它之后还会调用 `continueWithoutCredentialForAuthenticationChallenge` 方法 。这是当前用的最频繁，也是最新的方式；你可以在旧代码中看到调用了  `canAuthenticateAgainstProtectionSpace：` 或 `didReceiveAuthenticationChallenge:` 方法。代码 7-1 向你演示这究竟是怎么实现的。

```objective-c
- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *) challenge {
NSURLProtectionSpace *space = [challenge protectionSpace]; 
if([[space authenticationMethod] isEqualToString:NS
URLAuthenticationMethodServerTrust]) {
NSURLCredential *cred = [NSURLCredential credentialForTrust:
[space serverTrust]];
[[challenge sender] useCredential:cred forAuthenticationChallenge:
challenge]; }
}
```

> Listing 7-1: 发送给 challenge 一个虚假的 NSURLCredential 响应								

这段代码看起来用心良苦，尤其是它频繁使用了如下几个词语：保护（protection），证书（credential），授权（authentication）和 信任（trust）。而它实际所干的是绕过端点的 TLS 验证，留下网络连接可能被截取的风险。 									

当然，我并非真正鼓励你实施一些手段让你的应用程序绕过 TLS 验证。请不要一意孤行，误入歧途。这些例子只是说明你有可能在检查代码时遇到类似代码的书写形式。这些形式的代码可能很难被发现和理解，但如果你看到这些歧途绕过TLS 验证的代码块，一定要修改它。

### 7.1.2 NSURLConnection 的基本身份验证策略	

HTTP 基本身份验证机制并非特别强大。它不支持会话管理和密码管理，因此用户在不使用单个应用前提下无法注销或修改密码。但是对于一些任务，譬如验证 API 接口，这些问题显得不那么重要，你仍然可能会在代码库上以这个机制运行应用程序，或者通过自定义来实现。

你既可以使用 `NSURLSession` ，也可以使用 `NSURLConnection` 来实现 HTTP 的基本身份验证，但是不管你是在写一个应用程序或是检查别人的代码，这里都存在一些你需要知道的陷阱。

`NSURLConnection` 中的代理方法  `willSendRequestForAuthenticationChallenge` 提供了最简单的实现方式：

```objective-c
- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *) challenge {
  NSString *user = @"user"; 
    NSString *pass = @"pass";
  if ([[challenge protectionSpace] receivesCredentialSecurely] == YES && [[[challenge protectionSpace] host] isEqualToString:@"myhost.com"]) {
  NSURLCredential *credential = [NSURLCredential credentialWithUser:user password :pass persistence:NSURLCredentialPersistenceForSession];
  [[challenge sender] useCredential:credential forAuthenticationChallenge:challenge];
  } 
}
```

首先委托方法中传入了一个 `NSURLAuthenticationChallenge` 对象；紧接着它创建了一个用户名和密码变量，这可以由用户输入或者从钥匙串中提取获得；最后，challenge 的发送者接受 credential 和 challenge 两个传参。

以这种方式实现 HTTP 基本身份验证时，需要警惕两个潜在的问题。第一，无论是源代码还是共享偏好都不应该存储用户名和密码。不过你可以使用 `NSURLCredentialStorage API` 自动存储用户凭证到钥匙串中，如代码 7-2 中的 `sharedCredentialStorage` :

```objective-c
NSURLProtectionSpace *protectionSpace = [[NSURLProtectionSpace alloc] initWithHost: @"myhost.com" port:443 protocol:@"https" realm:nil authenticationMethod:nil];//1

NSURLCredential *credential = [NSURLCredential credentialWithUser:user password: pass persistence:NSURLCredentialPersistencePermanent];//2

[[NSURLCredentialStorage sharedCredentialStorage] setDefaultCredential:credential forProtectionSpace:protectionSpace];//3
```

> 代码7-2：设置保护空间的默认凭据									

代码 ❶ 简单创建了一个保护空间，其中包括主机，端口，协议，以及可选的 HTTP 跨域认证（如果使用 HTTP 基本身份验证）和身份验证方法（例如：使用 NTLM 或其他机制）；代码 ❷ ，示例中创建了一个最有可能来自用户输入的凭证：用户名和密码；然后，代码块 ❸ 为这个保护空间设置了默认凭证，同时自动存储该凭证到钥匙串中。在不久之后，应用程序中的代码能够使用 `defaultCredentialForProtectionSpace` 方法以相同的 API 接口读取凭证，如代码7-3。

```objective-c
credentialStorage = [[NSURLCredentialStorage sharedCredentialStorage] defaultCredentialForProtectionSpace:protectionSpace];
```

> 代码 7-3: 从保护空间中读取凭证										

但值得注意的是，存储在 `sharedCredentialStorage` 中的凭证都被标记了钥匙串属性：`kSecAttrAccessibleWhenUnlocked` 。如果你需要更严格的保护，则必须手动管理钥匙串的存储机制。我会在第 13 章涉及更多有关钥匙串的知识。

此外，创建凭证时指定 `persistence` 参数值一定要引起注意。如果你使用 `NSURLCredentialStorage`	存储到钥匙串中，那么在创建你的凭证时，既可以使用`NSURLCredentialPersistencePermanent` 也可以是`NSURLCredentialPersistenceSynchronizable` 类型。如果你正在使用一些短暂身份认证，使用`NSURLCredentialPersistenceNone` 或 `NSURLCredentialPersistenceForSession` 类型更为恰当。更多持久化类型说明请见表 7-1。

![](../Resource/Table-7-1.png)

> 表格 7-1: 证书持久性类型									

### 7.1.3 在 NSURLConnection 中实现 TLS 认证		

客户端进行身份验证的最佳方法之一是使用客户端证书和私有密钥；然而这在 iOS 上实现起来有些繁琐。其基本概念相对来说比较简单：实例化一个实现 `willSendRequestForAuthenticationChallenge` 方法（原`didReceiveAuthenticationChallenge` 方法）的委托对象，通过 `NSURLAuthenticationMethodClientCertificate` 方法进行身份验证，通过检索加载证书和私钥，并以此创建一个凭证传递给 `challenge` 对象。不幸的是，Cocoa 并未提供任何管理证书的内置 API 接口，所以你必须稍微混入一些 Core Foundation 的东西，就像下面这样：

```objective-c
- (void)connection:(NSURLConnection *) willSendRequestForAuthenticationChallenge:( NSURLAuthenticationChallenge *)challenge {
  if ([[[challenge protectionSpace] authenticationMethod] isEqualToString:NS URLAuthenticationMethodClientCertificate]) {
      SecIdentityRef identity;
      SecTrustRef trust;
      extractIdentityAndTrust(somep12Data, &identity, &trust);//1
      SecCertificateRef certificate;
      SecIdentityCopyCertificate(identity, &certificate);//2
      const void *certificates[] = { certificate };//3
      CFArrayRef arrayOfCerts = CFArrayCreate(kCFAllocatorDefault, certificates,
  1, NULL);//4

      NSURLCredential *cred = [NSURLCredential credentialWithIdentity:identity 		certificates:(__bridge NSArray*)arrayOfCerts
  	persistence:NSURLCredentialPersistenceNone]; //5
     [[challenge sender] useCredential:cred
  	forAuthenticationChallenge:challenge];//6
  } 
}
```

示例首先代码 ❶ 定义了 `SecIdentityRef` 和 `SecTrustRef` 两个变量，作为参数传递给 `extractIdentityAndTrust` 函数。该方法将从 PKCS #12 （后缀为 .p12 的文件）数据中提取得到身份和信任信息。这些固化文件存储了大量加密对象并放置在某个地方。

然后代码 ❷ 定义了一个 `SecCertificateRef` 变量保存从身份中提取的证书；紧接着代码 ❸ 将证书封装保存到一个的数组中；同时❹又将证书关联至 `CFArrayRef` 数组中；最后该代码片段创建了一个 `NSURLCredential` 对象，并传入身份信息和包含单个证书的数组两个参数❺；将此凭证作为 challenge 的应答❻。

你或许注意到了代码 ❶ 中并未对获取方式做详尽说明。这是因为我们能够以多种不同的方式获得实际的 p12 证书数据。你可以通过执行一次性引导从安全通道中获取新生成的证书，或选择在本地生成证书，接着从文件系统或钥匙串中读取。例子中 `somep12Data` 采用从文件系统中读取到证书信息，代码如下：

```objective-c
NSData *myP12Certificate = [NSData dataWithContentsOfFile:path]; CFDataRef somep12Data = (__bridge CFDataRef)myP12Certificate;
```

存储证书的最佳方式在使用 Keychain；我将进一步在13章详细阐述。

### 7.1.4 修改重定向行为

默认情况下，`NSURLConnection` 处理 HTTP 请求会默许已有的重定向行为。然而，当以下这些情况发生时，这些行为就非比寻常了。当遇到重定向时，`NSURLConnection` 将发送一个使用 `NSURLHttpRequest` 类封装的 HTTP 报头给目标地址。不幸的是，这意味着原本给原始域的 Cookie 信息被发送给了新的目标地址。结果显而易见，一旦攻击者重定向你的应用程序请求到欺骗地址，就能窃取到你的用户 Cookies 信息，同时还包括应用存储在 HTTP 包头的敏感数据。这种类型的漏洞称之为开放式重定向。

当然你可以修改这种行为，iOS 4.3 以及早前版本你需要实现 `NSURLConnectionDataDelegate` 协议中的 [`willSendRequest：redirectResponse` 方法](https://developer.apple.com/library/ios/#documentation/cocoa/conceptual/URLLoadingSystem/ Articles/ RequestChanges.html)，而在 iOS 5.0 和更新版本则是 [`NSURLConnectionDataDelegate` 协议](https://developer.apple.com/library/ios/#documentation/Foundation/Reference/ NSURLConnectionDataDelegate_protocol/ Reference/ Reference.html# // apple_ref/ occ/ intfm/ NSURLConnectionDataDelegate/ connection:willSendRequest:redirectResponse:)。

```objective-c
- (NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request
redirectResponse:(NSURLResponse *)redirectResponse
{
  NSURLRequest *newRequest = request;
  if (![[[redirectResponse URL] host] isEqual:@"myhost.com"]) {//1
  	return newRequest; 
  }
  else {
  	newRequest = nil; //2
    return newRequest;
  } 
}
```

代码 ❶ 检查重定向的域名是否与你的网站地址一致。如果相同，表明正常，否则将请求置为 `nil` ❷。

### 7.1.5 TLS 证书锁定																																					

在过去的几年里，已经涌现了一大批令开发者不安的证书颁发机构（CA），我们每天都会遇到需要 TLS 证书验证的情况。除了客户端应用程序信任的大量授权签署外，CA 证书颁发机构依旧存在几个突出的安全漏洞，可能造成签名密钥被泄露，或者发放证书权限过于宽松。这些漏洞允许任何人拥有签名密钥来冒充任何 TLS 服务器，意味着他们可以成功地，不费吹灰之力地读取或修改与服务器的请求以及答复。

为了帮助缓解这种攻击，许多类型的客户端应用程序已经实施了证书锁定策略。该术语涉及许多不同的技术，但核心思想是通过编程方式限制受应用程序信任的证书数量。你可以限制仅信任某个 CA（即你公司用于签署服务器证书有效的证书发布方），或是使用内部 Root CA 创建你自己的证书（信任链的顶部），要么干脆可以是叶证书（一种位于信任链底部的单一特定证书）。

作为 SSL 学院项目的负责人之一，我的同事 Alban Diquet 已经开发了一些便捷的工具包，允许你在应用程序中实现证书锁定。（更多详情请见： https://github.com/iSECPartners/ssl-conservatory	）。你可以选择自己写个工具包，或者使用已有的东西；无论哪种方式，一个出色的工具包能够使得证书锁定变得相当简单。例如，下面演示了使用 Alban 的工具包进行证书锁定是如何简单：

```objective-c
- (NSData*)loadCertificateFromFile:(NSString*)fileName {//1
  NSString *certPath = [[NSString alloc] initWithFormat:@"%@/%@", [[NSBundle
  mainBundle] bundlePath], fileName];
  NSData *certData = [[NSData alloc] initWithContentsOfFile:certPath]; return certData;
}
- (void)pinThings {
NSMutableDictionary *domainsToPin = [[NSMutableDictionary alloc] init];
 NSData *myCertData = [self loadCertificateFromFile:@"myCerts.der"]; //2
  if (myCertData == nil) {
NSLog(@"Failed to load the certificates"); return;
}
 [domainsToPin setObject:myCertData forKey:@"myhost.com"];//3
 if ([SSLCertificatePinning loadSSLPinsFromDERCertificates:domainsToPin] != 	YES)//4
 { 
   NSLog(@"Failed to pin the certificates");
	return;
	}
}
```

代码❶ 简单定义了一个方法，将  DER 文件格式的证书加载到一个 NSData 对象中，之后在代码 ❷ 处调用了该方法。如果成功，`myCertData` 对象会被关联到 `NSMutableDictionary` 字典对象中 ❸，紧接着将该字典传入 `SSLCertificatePinning `  类的 `loadSSLPinsFromDERCertificates` 方法 ❹。这些锁定参数加载完毕的同时，应用程序还需要实现一个  `NSURLConnection` 的委托对象，如示例7-4。

```objective-c
- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *) challenge {
if([challenge.protectionSpace.authenticationMethod isEqualToString:NS URLAuthenticationMethodServerTrust]) {
SecTrustRef serverTrust = [[challenge protectionSpace] serverTrust]; NSString *domain = [[challenge protectionSpace] host]; SecTrustResultType trustResult;
SecTrustEvaluate(serverTrust, &trustResult);
if (trustResult == kSecTrustResultUnspecified) {
// 在服务器的证书链中寻找锁定公钥
if ([SSLCertificatePinning verifyPinnedCertificateForTrust:serverTrust andDomain:domain]) {
// 找到对应证书; 保持连接
[challenge.sender useCredential:[NSURLCredential credentialForTrust :challenge.protectionSpace.serverTrust] forAuthenticationChallenge:challenge];
}
else {
// 证书未找到; 中断连接
[[challenge sender] cancelAuthenticationChallenge: challenge]; }
}
else {
// 证书链验证失败; 中断连接
[[challenge sender] cancelAuthenticationChallenge: challenge]; }
} 
}
```

> 代码 7-4: `NSURLConnection` 委托对象实现证书锁定逻辑									

这里只是简单验证了远程服务器提供的证书链，并将其与应用程序附带的固定证书进行比较。如果找到一个固定的证书，则保持连接；如果未找到，则认证过程将被取消。

委托对象实现如上所示，所有你使用的  `NSURLConnection` 都应该逐一检查，以确保它们被固定在你的预定义列表中的域和证书对中。如果你对此好奇，你可以找到其他代码来实现自己的证书锁定，请见https://github.com/iSECPartners/ssl-conservatory/tree/master/ios。由于关联了不少其他逻辑，所以这里我就不再一一介绍了。

> 如果你是个急性子，不妨通过继承示例代码中的 SSL 委托对象来实现证书锁定。										

到目前为止，我围绕 `NSURLConnection` 讲述了有关网络的安全问题和解决方案。但 iOS 7 之后，`NSURLSession` 类较传统的 `NSURLConnection` 具有更多优点。接下来让我们来仔细看看这个 API 。

## 7.2 使用 NSURLSession

`NSURLSession` 类普遍受到开发者的青睐，因为它专注于使用网络会话，而不像 `NSURLConnection` 只注重单个请求。在对 `NSURLConnection` 类范围有所扩展的前提下， `NSURLSession` 还允许配置单个会话，而非为整个应用程序，这样做的好处是提供了额外的灵活性。一旦会话实例化完毕，它们会被递交给各个任务去执行，任务类型可以是 `NSURLSessionDataTask`，`NSURLSessionUploadTask` 和 `NSURLSessionDownloadTask` 这三个类中的其中一个。

在本节中，我们将探索一些使用 `NSURLSession` 的方式，以及一些潜在的安全隐患，还包括一些`NSURLConnection`  类没有提供的安全机制。

### 7.2.1 NSURLSession 配置	

通过 `NSURLSessionConfiguration` 类封装配置选项并传递给 `NSURLSession ` 对象，这样你就可以单独配置不同类型的请求。例如，你可以应用不同的缓存和 cookie 策略来要求获取不同敏感等级的数据，而不是为整个应用设置一套请求策略。你可以使用 `[NSURLSessionConfigurationdefaultConfiguration]` 获得系统默认的 `NSURLSession` 配置，甚至你可以使用`[NSURLSession sharedSession]` 简单实例化一个请求对象，而无须指定配置策略。

对于那些安全敏感的请求，请勿在本地保留任何信息，此时应该选择 `ephemeralSessionConfiguration` 配置方法最佳。第三种方法，`backgroundSessionConfiguration` 专门用于长时间运行的上传或下载任务。这种类型的会话将交由一个系统服务来管理完成，这意味着你的应用程序被杀掉或崩溃都不会对请求产生任何影响。

除此之外，首次你使用时可以指定一个只使用 TLS 1.2版本协议的连接，这有助于抵御如 [BEAST](https://bug665814.bugzilla.mozilla.org/attachment.cgi?id=540839) 和 [CRIME](https://docs.google.com/presentation/d/11eBmGiHbYcHR9gL5nDyZChu_-lCa2GizeuOfaLU2HOU/edit?pli=1#slide=id.g1d134dff_1_222) 攻击，防止网络攻击者读取或篡改你的 TLS 连接。

> 注意：`NSURLSession` 实例化对象的会话配置是一个只读属性；会话期间无法修改策略和配置，也不能将当前配置替换成别的。

### 7.2.2 执行 NSURLSession 任务

接下来我们会创建一个 `NSURLSessionConfiguration` 配置，然后指派一个简单的任务给它用来演示典型操作流程，如代码 7-5。

```objective-c
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];//1

[configuration setTLSMinimumSupportedProtocol = kTLSProtocol12]; //2

NSURL *url = [NSURL URLWithString:@"https://www.mycorp.com"];//3

NSURLRequest *request = [NSURLRequest requestWithURL:url];

NSURLSession *session = [NSURLSession sessionWithConfiguration:configuration delegate:self
delegateQueue:nil];//4

//5
NSURLSessionDataTask *task = [session dataTaskWithRequest:request completionHandler:
^(NSData *data, NSURLResponse *response, NSError *error) { // 你的完成处理闭包写在这里
}];//6

[task resume];//7
```

> 代码 7-5: 创建一个支持 TLSv1.2 协议的短连接配置										

❶ 处首先实例化了一个短连接的 `NSURLSessionConfiguration` 配置对象。这能够防止缓存数据在本地存储。然后 ❷ 中配置指定 TLS 1.2 版本协议，允许开发者能够控制端点以及是否支持该版本。接着，就像 `NSURLConnection` 所做的那样，我们实例化了一个 `NSURL` 对象，并凭借它创建了一个 `NSURLRequest` 请求❸。随着配置和请求创建完成，应用程序接下来将实例化一个会话❹，并指派任务给它❺。

`NSURLSessionDataTask` 以及其他任务类都需要接受一个完成时处理程序块作为 ` block` 参数❻。该 `block` 异步处理来自服务器的响应和数据。另一种方式（或者两种方式混合使用），你可以指定一个实现  `NSURLSessionTaskDelegate`  协议的委托对象。而混用 `completionHandler` 和委托的好处在于：`completionHandler` 负责处理请求结果，而委托对象负责身份验证和缓存处理，更多的是从会话的角度去考虑，而非任务基础（我将在下一节中讨论这个问题）。

最后，代码 ❼ 调用 `resume` 方法开始执行任务，因为所有任务刚创建时都处于挂起状态。

### 7.2.3 如何绕过 NSURLSession 的 TLS 验证

`NSURLSession` 同样有途径可以绕过 TLS 验证。应用程序只需要实现 `didReceiveChallenge` 委托方法，并在方法中发送消息给 `challenge` 对象，最后将返回结果作为会话凭证即可。请见代码7-6。

```objective-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NS URLAuthenticationChallenge *)challenge completionHandler:(void (^)(NS URLSessionAuthChallengeDisposition disposition, NSURLCredential * credential)) completionHandler {
  
completionHandler(NSURLSessionAuthChallengeUseCredential, [challenge proposedCredential]);//1
}
```

> 代码 7-6 : 在 NSURLSession 中绕过服务器验证

这是另一种绕过 TLS 检查的方式，略带技巧性。请仔细阅读代码 ❶ ，这里使用了一个 `completionHandler` 程序块处理传入的 `proposedCredential` 凭证参数。 

### 7.2.4 NSURLSession 中的基本身份验证	

`NSURLSession` 中的 HTTP 认证是由会话管理，整个认证过程在委托对象的 `didReceiveChallenge` 方法中实现，如代码 7-7 所示。

```objective-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NS URLAuthenticationChallenge *)challenge completionHandler:(void (^)(NS URLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {//1
NSString *user = @"user"; NSString *pass = @"pass";
NSURLProtectionSpace *space = [challenge protectionSpace];
  if ([space receivesCredentialSecurely] == YES && [[space host] isEqualToString:@"myhost.com"] && [[space authenticationMethod] isEqualToString:NSURLAuthenticationMethodHTTPBasic]) {
    NSURLCredential *credential = [NSURLCredential credentialWithUser:user
    password:pass persistence:NSURLCredentialPersistenceForSession];//2
    
    completionHandler(NSURLSessionAuthChallengeUseCredential, credential);//3 
  }
}
```

> 代码7-7：`didReceiveChallenge` 委托方法实现示例

这种实现方式定义了一个委托对象和一个完成处理程序块❶，并实例化了一个`NSURLCredential` 对象❷，接着将凭证传递给完成处理程序块❸。注意，无论是 `NSURLConnection`  还是  `NSURLSession` 的做法，一些开发者容易遗忘对主机进行身份验证以及安全地发送凭证。这可能导致你的凭证被发送给应用程序加载的所有连接，而不仅仅只是你期望的；代码 7-8 列出的例子演示了这个错误是如何发生。

```objective-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NS URLAuthenticationChallenge *)challenge completionHandler:(void (^)(NS URLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
NSURLCredential *credential = [NSURLCredential credentialWithUser:user
password:pass persistence:NSURLCredentialPersistenceForSession];
completionHandler(NSURLSessionAuthChallengeUseCredential, credential); 
}
```

> Listing 7-8: HTTP 授权的错误用法										

如果你想为一个专门的端点使用持久化凭证，可以像 `NSURLConnection` 所做的那样将凭证存储到 `sharedCredentialStorage` 当中。当会话构建完成，你只需事先提供这些凭证，而不必担心委托方法，如代码7-9。

```objective-c
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
[config setURLCredentialStorage: [NSURLCredentialStorage sharedCredentialStorage]];
NSURLSession *session = [NSURLSession sessionWithConfiguration:config delegate:nil
delegateQueue:nil];
```

> 代码 7-9: 将已存储的凭证关联给 `NSURLSessionConfiguration` 对象										

上面例子仅仅创建了一个 `NSURLSessionConfiguration` 对象，同时指定它使用共享凭证存储。当你访问那些将凭证保存在 Keychain 中的资源时，会话将直接使用这些凭证验证身份。

### 7.2.5 管理已存储的 URL 凭证																																																

你已经了解如何使用 `sharedCredentialStorage` 存储和读取凭证，但 `NSURLCredentialStorag` 还提供了 API 允许你调用 `			removeCredential:forProtectionSpace` 方法删除凭证。例如，当用户明确表示退出应用程序或删除账户时，你可能需要借助这个接口来做一些事情。代码7-10展示了一个典型的用例。

```objective-c
NSURLProtectionSpace *space = [[NSURLProtectionSpace alloc] initWithHost:@"myhost.com"
port:443 protocol:@"https"
realm:nil authenticationMethod:nil];
NSURLCredential *credential = [credentialStorage defaultCredentialForProtectionSpace:space];
[[NSURLCredentialStorage sharedCredentialStorage] removeCredential:credential forProtectionSpace:space];
```

> 代码 7-10: 移除默认凭证										

这将从本地 Keychain 中删除相应凭证。然而，如果一个凭证设置了 `NSURLCredentialPersistenceSynchronizable` 持久属性，那么它可能已经通过 iCloud 同步到其他设备上了。 要删除所有设备的凭证，使用 `NSURLCredentialStorageRemove SynchronizableCredentials` 选项，如代码7-11。

```objective-c
NSDictionary *options = [NSDictionary dictionaryWithObjects forKeys:NS URLCredentialStorageRemoveSynchronizableCredentials, YES];
[[NSURLCredentialStorage sharedCredentialStorage] removeCredential:credential forProtectionSpace:space
options:options];
```

> 代码 7-11: 移除本地钥匙串和 iCloud 云服务中的所有凭证								

到目前为止，你应该对 `NSURLConnection` 和 `NSURLSession`  的 API 接口以及基本用法有一定了解了。你可能会遇到其他的网络第三方库，它们都具有各自的特点，同时请求安全配置也略有不同。现在我将介绍其中的几个。

## 7.3 使用第三方网络 API 接口的风险

目前 iOS 应用中存在较多流行的第三方网络框架，主要用于简化各种网络任务，例如多部分上传和证书锁定。其中最常用的当属 [AFNetworking](https://github.com/AFNetworking/AFNetworking)，其次是已过时的 [ASIHTTPRequest](https://github.com/pokeb/asi-http-request)，以上两者我都会在章节中向你介绍。

### 7.3.1 如何正确使用 AFNetworking

AFNetworking 是基于 NSOperation 和 NSHTTPRequest 封装的网络库，受欢迎程序有目共睹。它提供了多种便捷的方法来与不同类型的网络 API 接口进行交互，同时也支持自定义 HTTP 网络任务。

和其他网络框架一样，确保 TLS 安全机制不被禁用是一个至关重要的任务。 AFNetworking 中可以通过几种方式禁用TLS 证书验证。第一种是通过设置 `_AFNETWORKING_ALLOW_INVALID_SSL_CERTIFICATES` 标志位，通常是在 `Prefix.pch` 文件中进行设置；另外一种方法是设置 `AFHTTPClient` 属性，如 7-12 中所示。

```objective-c
NSURL *baseURL = [NSURL URLWithString:@"https://myhost.com"]; 
AFHTTPClient* client = [AFHTTPClient clientWithBaseURL:baseURL]; 
[client setAllowsInvalidSSLCertificate:YES];
```

> Listing 7-12: 使用 `setAllowsInvalidSSLCertificate										 `方法禁用 TLS 验证									

最后一种方式是改变 `AFHTTPRequestOperationManager` 的安全策略，设置为 `AllowsInvalidSSLCertificate` 来禁用 TLS 验证，如代码 7-13

```objective-c
AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager]; [manager [securityPolicy setAllowInvalidCertificates:YES]];
```

> Listing 7-13: 使用安全策略来禁用 TLS 验证									

同样你还需要验证发布版的代码中不混入你的测试代码，比如使用 `AFHTTPRequestOperationLogger` 类打印日志。该类通过封装了 `NSLog` 将请求 URL 链接地址记录到苹果系统日志当中，允许在某些 iOS 版本中被其他应用程序查看。

AFNetworking 提供了一个特别有用的功能用于证书锁定。你只需要在项目的 `.pch` 文件中设置 `_AFNETWORKING_PIN_SSL_CERTIFICATES_#define` ，同时设置 `AFHTTPClient` 实例的锁定属性为`defaultSSLPinningMode` 即可，更多可用的锁定模式描述请见表 7-2。然后你将你要锁定的证书放到程序包根下，文件扩展名为 `.cer` 。

![](../Resource/Table-7-2.png)

下面附上 `AFNetworking` 提供的示例代码，你可以通过检查 URL 以确定它们是否应该被锁定。仅仅需要评估方案和域名就能知道这些领域是否属于你。请见代码 7-14  。

```objective-c
if ([[url scheme] isEqualToString:@"https"] &&
[[url host] isEqualToString:@"yourpinneddomain.com"]) {
[self setDefaultSSLPinningMode:AFSSLPinningModePublicKey]; 
}
else {
[self setDefaultSSLPinningMode:AFSSLPinningModeNone];
}
return self; 
}
```

> 代码 7-14: 确定是否要锁定某个URL链接

上面示例中的 `else` 语句处理字段并非绝对必要的，因为默认是不锁定证书，但这样做逻辑显然更为清晰。

请牢记 `AFNetworking` 是锁定程序包中的所有证书，但它检查证书通用名称和网络端点匹配的主机名。如果你的应用程序锁定了不同安全标准的多个站点，这会引起一个问题。换句话说，如果你的应用程序同时锁定了 https://funnyimages.com 和 https://www.bank.com 两个站点，此时一位拥有 funnyimages.com 私钥的攻击者就可以截获你的应用程序与 bank.com 的通信内容。

现在你已经看到如何使用和滥用 AFNetworking 库，让我们继续探讨 ASIHTTPRequest。

### 7.3.2 ASIHTTPRequest 不安全使用方式			

ASIHTTPRequest 是一个类似于 AFNetworking 的过时库，不过它的功能并不完善，而且它是基于 `CFNetwork` 的 API 。 它不应该被用于新的项目，而且你可以发现迁移现有的代码库代码过于昂贵。想要绕过标准的 SSL 验证，你需要研究这些代码库寻找 `setvalidatessecurecertificate:NO` 字段。

此外，你还需要检查项目中的 `ASIHTTPRequestConfig.h` 文件确保未启用详细日志记录选项（请见代码7-15）。

```objective-c
// If defined, will use the specified function for debug logging 
// Otherwise use NSLog
#ifndef ASI_DEBUG_LOG
#define ASI_DEBUG_LOG NSLog 
#endif

// When set to 1, ASIHTTPRequests will print information about what a request is doing
#ifndef DEBUG_REQUEST_STATUS 
#define DEBUG_REQUEST_STATUS 0
#endif

// When set to 1, ASIFormDataRequests will print information about the request body to the console
#ifndef DEBUG_FORM_DATA_REQUEST 
#define DEBUG_FORM_DATA_REQUEST 0
#endif

// When set to 1, ASIHTTPRequests will print information about bandwidth throttling to the console
#ifndef DEBUG_THROTTLING 
#define DEBUG_THROTTLING 0
#endif

// When set to 1, ASIHTTPRequests will print information about persistent connections to the console
#ifndef DEBUG_PERSISTENT_CONNECTIONS 
#define DEBUG_PERSISTENT_CONNECTIONS 0
#endif

// When set to 1, ASIHTTPRequests will print information about HTTP authentication (Basic, Digest or NTLM) to the console
#ifndef DEBUG_HTTP_AUTHENTICATION #define DEBUG_HTTP_AUTHENTICATION 0
#endif
```

> 代码 7-15:ASIHTTPRequestConfig.h 中定义的日志打印函数

如果你想使用这些日志记录功能，你可能需要在 `#ifdef DEBUG` 条件上再封装一层，就像这样：

```objective-c
#ifndef DEBUG_HTTP_AUTHENTICATION 
	#ifdef DEBUG
		#define DEBUG_HTTP_AUTHENTICATION 1 
	#else
		#define DEBUG_HTTP_AUTHENTICATION 0 
	#endif
#endif
```

ASIHTTPRequestConfig.h 文件在条件语句中封装了日志记录功能，防止在发布版中泄露打印这些信息。

## 7.4 多点连接

iOS 7 引入了[多点连接](https://developer.apple.com/library/prerelease/ios/documentation/MultipeerConnectivity/Reference/MultipeerConnectivityFramework/ index.html)，允许附近的设备能够以最低的网络配置彼此间进行通讯。多点连接通信方式可以是 Wi-Fi(点对点和多点网络)或蓝牙等个人区域网络（PANs）。Bonjour 是浏览和广告可用服务的默认机制。

开发者可以利用多点连接技术进行设备间文件或流媒体内容传输。正如任何类型的对等通信，验证来自不明身份节点的数据至关重要；然而，传输安全机制也是必须的，确保数据安全不被窃听。

多点连接会话既可以通过 `initWithPeer` 构造方法来创建，也可以是 `MCSession`类的类方法 `	initWithPeer:securityIdentity:encryptionPreference:` 。后者允许你进行加密操作，以及包括使用证书链验证设备。									

在为 `encryptionPreference` 赋值时，你有三个选项：`MCEncryptionNone`、`MCEncryptionRequired` 和 `MCEncryptionOptional` 。请注意这些值分别代表了 `0`，`1` 和 `2`。因此你可以将 `0 ` 和 `1` 值当做布尔值来使用，分别代表不加密和加密选项，而 `2` 值从功能上来说等同于不加密选项。

这对于无条件要求加密无疑是一个不错的注意，要知道 `MCEncryptionOptional`
会受到降级攻击（更多详情可见 Alban Diquet 关于多点连接协议的[逆向黑帽演讲](https://nabla-c0d3.github.io/blog/2014/08/20/multipeer-connectivity-follow-up/)。）代码 7-16 展示了一个典型的调用方式，创建一个会话同时进行加密操作。


```objective-c
MCPeerID *peerID = [[MCPeerID alloc] initWithDisplayName:@"my device"];
MCSession *session = [[MCSession alloc] initWithPeer:peerID securityIdentity:nil
encryptionPreference:MCEncryptionRequired];
```

> 代码 7-16: 创建一个 MCSession										

当成功连接到远程设备后，调用代理方法 `session:didReceiveCertificate:fromPeer:certificateHandler:` ，通过传入对等方的证书，允许你指定一个处理程序方法用于证书验证成功后的具体操作。

> 注意：如果你无法创建 `didReceiveCertificate` 代理方法，或者未在代理方法中实现  `certificateHandler` ，则不会验证远程端点，存在被第三方截取信息的潜在危险。

当使用多点连接 API 检查代码库时，请确保 `MCSession` 的所有实例都提供了一种身份以及要求传输加密。涉及任何类型的敏感信息，会话都不应该简单使用 `initWithPeer`  构造方法实例化一个对象。另外你还要确保正确实现了  `didReceiveCertificate` 代理方法，同时 `certificateHandler` 处理程序能够应对验证节点证书失败的情况。我猜你特别不希望看到这样的代码：

```objective-c
- (void) session:(MCSession *)session didReceiveCertificate:(NSArray *)certificate fromPeer:(MCPeerID *)peerID
certificateHandler:(void (^)(BOOL accept))certificateHandler
{
certificateHandler(YES);
}
```

此代码盲目地将一个 YES 布尔类型值传递到处理程序中，你绝对不应该这么做。

这是由你来决定你想以何种方式实现验证的。系统验证往往能够自定义，但会给定你几个基本选项。你可以在客户端生成证书，然后在第一次使用时（TOFU:trust on first use）选择信任，即首次与设备进行配对时会弹出证书进行验证。另外你可以搭建一个服务器，集中式管理用户的共用证书，当用户查询时再取回分发。选择一个适合你业务模式和威胁模式的解决方案吧。

## 7.5 使用 NSStream 较底层网络

`NSStream` 适用于创建非 HTTP 的网络连接，但只要折腾一小些它同样也能用于 HTTP 通信。对于 OSX Cocoa 和 iOS Cocoa Touch 中一些难以理解的原因，Apple 公司移除`getStreamsToHost`方法，使得无法通过 NSStream 来建立网络连接到远程主机。所以喽，如果你想要自己来捣鼓流数据（Stream），牛逼。否则，请见技术问答 [QA1652](https://developer.apple.com/library/ios/#qa/qa2009/qa1652.html)，苹果公司描述了一个范畴，你可以用它定义一个大体上相当于 `NSStream` 中 `getStreamsToHostNamed` 方法。

另外一种方法是使用更底层 Core Foundation 中的 `CFStreamCreatePairWithSocketToHost` 函数，将`CFStream` 类型的输入和输出数据转换成 `NSStream` 类型，如代码7-17。 

```objective-c
NSInputStream *inStream; NSOutputStream *outStream;
CFReadStreamRef readStream;
CFWriteStreamRef writeStream;
CFStreamCreatePairWithSocketToHost(NULL, (CFStringRef)@"myhost.com", 80, &
readStream, &writeStream);
inStream = (__bridge NSInputStream *)readStream; outStream = (__bridge NSOutputStream *)writeStream;
```

> 代码 7-17:将 CFStreams 类型转成 NSStreams 类型

`NSStream` 仅允许用户略微控制连接的特性，如 TCP 端口和 TLS 设置（见代码7-18）。

```objective-c
NSHost *myhost = [NSHost hostWithName:[@"www.conglomco.com"]];
[NSStream getStreamsToHostNamed:myhost port:443
inputStream:&MyInputStream outputStream:&MyOutputStream];
[MyInputStream setProperty:NSStreamSocketSecurityLevelTLSv1 forKey:NSStreamSocketSecurityLevelKey];//1
```

> 代码 7-18: 使用 NSStream 打开一个基本的 TLS 连接									

以上是 NSStream 的典型使用方式：配置一台主机，端口和输入输出流。尽管你没有大量控制 TLS 的配置， 唯一可能被搞砸的配置是代码1处的 `NSStreamSocketSecurityLevel` 选项。你必须把他设置为 `NSStreamSocketSecurityLevelTLSv1` 确保最终不会使用旧的，破的 SSL/TLS 协议。

## 7.6 使用 CFStream 底层网络

对于 `CFStream`，开发者对 TLS 会话协商控制内容也少得可怜。具体请参阅表 7-3 中的 `CFStream` 属性，你可能会感兴趣。这些控制项允许开发者重写或禁用设备间规范名称（CN）的验证，忽略到期日期，允许不受信任的根证书，并完全忽略验证所有证书链。

![](../Resource/Table-7-3.png)

你可能不应该使用这些安全常量，但如果必须使用 TLS `CFStream` 的话，一定要以正确方式操作。这很简单！前提是你在应用程序本身创建一个网络服务器（这是  `CFStream` 在 iOS 应用程序中比较罕见的一种使用方式 ），你应该遵循以下两个步骤：

1. 设置 `kCFStreamSSLLevel` 等于 `kCFStreamSocketSecurityLevelTLSv1`
2. 不要掺杂任何其他的东西。

## 7.7 章节回顾

你已经了解应用程序与外界通信的不少方式，以及可能实现一些错误的方式。现在我们将注意力转到与其他应用程序的通信上，例如通过 IPC 来查看数据移动时可能发生的陷阱。

