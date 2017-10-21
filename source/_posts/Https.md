---
title: HTTPS 问答
date: 2016-08-28 17:19:48
tags:
- NetWorking
---
本文从问题入手，简单分析了 HTTPS 中 TLS 握手过程以及简单介绍了数字签名、数字证书、对称加密、非对称加密等知识。
<!--more-->
©原创文章，转载请注明出处！

# Overview
________________________________
为了加强网络安全，2015年，Apple 在 iOS9、OS X v10.11 上提出 ATS (App Transport Security) 的概念，旨在通过 HTTPS 建立安全连接。当时，ATS 还是个可选项，但 Apple 在 WWDC 2016 上表示，从2017年1月1日起所有 app 都必须使用 ATS (不清楚不使用的后果是什么，可能是拒绝提交 app store)。

> Apple 对 ATS 有如下要求：
> 1. The server must support at least Transport Layer Security (TLS) protocol version 1.2.
> 2. Connection ciphers are limited to those that provide forward secrecy.
> 3. Certificates must be signed using a SHA256 or better signature hash algorithm, with either a 2048 bit or greater RSA key or a 256 bit or greater Elliptic-Curve (ECC) key.

由于系统网络 API，如：`NSURLConnection`、`NSURLSession` 已经支持 HTTPS，因此，客户端从 HTTP 切换到 HTTPS 几乎没有什么工作量 (前提是后台使用了可靠的证书)。

但作为开发人员，对于 HTTPS 的基本概念、流程还是需要了解，以便在出现问题时能快速定位问题、解决问题 (所谓知彼知己)。
下面我们将通过问答的形式讨论分析 HTTPS 的基本原理。
# HTTPS 的使命是什么？
__________________________________
**『建立安全的网络连接』**
没错，但反过来要问了， HTTP 为什么不安全？
『HTTP (Hypertext Transfer Protocol，超文本传输协议)』 通过明文传输通信内容，导致信息很容易被窃听、篡改，同时无法验证通信双方的身份。

因此，HTTPS 要解决这些问题：
+ 信息加密——防止信息被窃听；
+ 信息完整性校验——防止信息被篡改；
+ 身份认证——防止信息劫持。

# HTTPS 与 HTTP 什么关系？
____________________________________
我们知道，网络协议都是处在 OSI 七层网络模型之中，得益于良好的分层模型，HTTPS 就是在 HTTP 协议栈的基础之上添加了一个安全层 TLS/SSL (Transport Layer Security/Secure Sockets Layer)。![](/img/HTTPHTTPS.png)在 HTTPS 协议栈中，HTTP、TCP 协议无须任何改动，加解密、认证等工作由 TLS 层完成。

# HTTPS 工作原理是什么？
____________________________________
这是一个很大的问题，细分的话就是 HTTPS 如何完成它的3个使命：加密、完整性校验以及身份认证。
这一切都要从 TLS/SSL 协议说起，但在开始之前有必要先了解一些相关的基础知识：
## 对称加密与非对称加密
+ 对称加密 (Symmetric cryptography)，发送方与接收方使用相同的密钥加密与解密，这也是其最大的弱点，意味着双方在能加密通信前需要传送密钥；
+ 非对称加密 (也称为公钥加密，Asymmetric cryptography/Public-key cryptography)，旨在解决对称加密必须传送密钥的缺陷。1976年， Whitfield Diffie 与 Martin Hellman 提出在不交换密钥的情况下完成加解密的构想，即 Diffie–Hellman key exchange。1977年，数学家 Rivest、Shamir 以及 Adleman 受 Diffie–Hellman key exchange 的启发，设计出一种非对称加密算法，称之为 RSA 算法，其思想是通信中的一方生成一对密钥：公钥、私钥，公钥分开发布，任何人都可以获取，私钥则是密保的。公钥加密的信息只有私钥能解密，私钥加密的信息也只有其对应的公钥可以解密。

其中，[Diffie-Hellman Key Exchange在原理大致如下](https://commons.wikimedia.org/wiki/File:Diffie-Hellman_Key_Exchange_)：![](/img/Diffie-Hellman_Key_Exchange.png)更多信息可以参考[Diffie–Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)。

## 数字签名 (Digital Signature)
顾名思义，签名主要就是防伪，数字签名主要用于认证消息的发送者 (发送者无法抵赖)、消息完整性校验。
> A digital signature is a mathematical scheme for demonstrating the authenticity of a digital message or documents. A valid digital signature gives a recipient reason to believe that the message was created by a known sender, that the sender cannot deny having sent the message (authentication and non-repudiation), and that the message was not altered in transit (integrity).

![](/img/DigitalSignature.png)
从[上图](https://upload.wikimedia.org/wikipedia/commons/2/2b/Digital_Signature_diagram.svg)可以看出，数字签名就是将要发送的数据通过 hash 函数计算消息摘要，再通过发送者私钥对摘要加密，最后附加在数据后面一起发送。
接收者，在收到消息后通过发送者公钥解密其中的签名，并同样通过 hash 函数计算消息摘要，若两者一致说明消息没有被篡改(完整性)，同时也能证明消息的确来自发送者。其中的关键有两点：
+ 计算消息摘要的 hash 函数：对输入敏感且不可逆；
+ 非对称加密：一个公钥解一个私钥，通过发送者发布的公钥能解开的加密信息一定是通过发送者配对的私钥加密的。

## 数字证书 (Digital Certificate)
仅仅通过非对称加密能实现身份认证吗？
我们知道，在非对称加密中，接收方仅仅掌握了所谓来自发送方的公钥，而公钥是不包含任何身份信息的。
![](/img/MiddlemanAttack.jpg)
如上图所示中间人攻击，攻击者可以劫持发送者公钥，将自己的公钥发送给接收者，此时其完全可以冒充发送者与接收者通信，而接收者毫无察觉。
此时，数字证书应运而生：![](/img/digitalcertificatestructure.jpg)
上图是 X.509 定义的 v3 版数字证书结构，证书本身包含的重要信息有：证书持有者以及公钥、证书签发者、过期时间等。当然，最后还要附上签发者的数字签名，其中，数字签名用于判断证书的真伪。
从上面我们知道，数字签名需要使用签发者公钥解密，那么如何获得证书签发者的公钥以便验证证书的真伪？如何判断获得的签发者公钥没被篡改？
额，似乎要进入死循环了...

此时，CA (Certificate Authority，证书授权中心) 应运而生，其作为第三方权威可信机构，承担 PKI 中公钥合法性校验的职责，并签发认证证书以及对证书的维护管理等。
> A public key infrastructure (PKI) is a set of roles, policies, and procedures needed to create, manage, distribute, use, store, and revoke digital certificates[1] and manage public-key encryption.

![](/img/CA.jpg)
上图为证书从 CA 到服务器，再到客户端的大致流程：
+ 服务器向 CA 提出证书申请 (需要提供公钥、域名、申请人信息等)；
+ CA 对提出的申请进行核审；
+ 核审通过后向申请者签发证书 (证书除了包含申请人公钥、域名等信息，还有证书过期时间，最重要的是 CA 的签名)；
+ 客户端连接服务器时，服务器下发证书；
+ 客户端验证证书，若验证通过则进行后续连接操作 (如：协商通信密钥等)。

再回到前面的问题『如何获得证书签发者的公钥以便验证证书的真伪？如何判断获得的签发者公钥没被篡改？』
由于 CA 是公认的权威机构，系统或浏览器会内置其自签名的证书 (根证书)。

## 证书链 (Certificate chains)
由于证书的需求量巨大，为了减少一级 CA 的工作量，出现了二级、三级、... CA 机构，其签发出来的证书也就构成了一个证书链。
![](/img/CertificateChain.jpg)
如[上图](https://docs.oracle.com/cd/E19693-01/819-0997/gdzen/index.html)所示，所有 CA 机构组成一个树状结构。
![](/img/VerifyingACertificateChain.jpg)
证书的校验逆证书链而上，直到找到系统信任的证书。
![](/img/googleCertificateChain.jpg)
上图是 google.com.hk 的证书链，可以看到 *google.com.hk 的证书是由二级 CA: Google Internet Authority G2签发，而该二级 CA 的证书则由根 CA: GeoTrust Global CA 签发，从而构成一个可信任的证书链。

好了，下面该回到正题了。

## TLS/SSL 协议
____________________________________
我们已经知道，在 HTTPS 协议栈中，TLS/SSL 处于 HTTP 与 TCP 之间。进一步细分的话，TLS 分为2层5条协议：![](/img/TLSprotocolstack.jpg)

其中，TLS Record Protocol 用于处理传输的数据，其主要职责是将上层协议下发的数据分块 (一块最大2^14 = 16384 bytes)、压缩、添加 MAC、加密等 (对于从底层协议上行的数据，其操作正好相反：解密、校验、解压、重组)。
![](/img/RecordProtocoloperation.jpg)

### TLS/SSL Handshake
_____________________________________
TCP 为了建立可靠连接需要三次握手，TLS 为了实现加密、校验、身份认证同样需要握手 (握手，简单讲就是通信双方协商在后续通信过程中需要使用的信息)。

整个握手过程双方大概要确认以下几点：
+ 双方通信使用的 TLS 版本；
+ 交换双方用于生成对称加密密钥的参数；
+ 确认双方使用的加密算法、压缩算法；
+ 完成身份认证。

我们知道，身份认证时的数字签名采用的是非对称加密。出于效率考虑，在 TLS 链接建立后，双方通信时采用的是对称加密。因此，TLS 握手需要确定对称加密密钥 (Master Secret)。目前有两种方法计算 Master Secret：RSA 以及 Diffie-Hellman，因此，TLS Handshake 分为 RSA Handshake、Diffie-Hellman 握手。

### RSA Handshake
![](/img/RSAHandshake.jpg)
上图是 RSA Handshake 的基本流程：
+ Client Hello：客户端首先发起 TLS 握手，告诉服务端其支持的最高 TLS 版本、加密套件列表 (cipher suites)、压缩方法并产生随机数 Random_C；
+ Server Hello：服务端根据客户端支持的最大 TLS 版本决定此次通信使用的 TLS 版本、从客户端加密套件列表中选择一种加密套件以及压缩方法、生成随机数 Random_S 并将 Session Id 一起传给客户端；
+ Server Certificate：服务端将证书传给客户端以便认证其身份；
+ Server Hello Done：告知客户端服务端已完成握手相关的所有数据的传送；
+ 客户端校验 Server Certificate；
+ Client Key Exchange：客户端生成随机密钥 PreMaster Secret，并用服务端公钥加密后发送给服务端；
+ Client change_cipher_spec：通知服务端后续消息将加密；
+ Client Finished：第一条加密的消息，表示握手完成；
+ 服务端接收到加密的 PreMaster Secret后，用私钥解密，此时双方都掌握了计算 Master Secret 的所有信息 (Random_C、Random_S、PreMaster Secret)，双方分别计算 Master Secret；
+ Server change_cipher_spec：通知客户端后续通信会加密；
+ Server Finished：握手完成。

### Diffie-Hellman Handshake
新技术的出现往往是原有技术存在不足或缺陷，那么 RSA Handshake 存在什么问题？
在 RSA 中，身份认证以及对 PreMaster Secret 加密使用的是同一对 public-private key，如果攻击者截获了服务器的私钥并窃听了整个握手过程，那么其可以顺利的破解整个通信消息。更严重的是，整个通信过程对称加密密钥是固定不变的，所以即使当前攻击者没有掌握服务端私钥，其可以将整个通信过程记录下来，等其获取到服务端私钥后，再进行破解。

而在 Diffie-Hellman Handshake 过程中，双方不需要在握手过程中传递类似 PreMaster Secret 的信息，更可喜的是，通过 Diffie-Hellman key exchange 可以生成新的对称密钥，废弃老的密钥，也就是在通信过程中对称密钥不固定，可以随时更换。

在握手流程上与 RSA Handshake 相比，Diffie-Hellman Handshake 双方都要交换 Diffie–Hellman 参数：
![](/img/Diffie-HellmanHandshake.jpg)

下面我们使用 Safari 访问 『https://www.google.com.hk』，并通过 Wireshark 抓包来了解一下其中 TSL 握手的过程：
1、Client Hello![](/img/ClientHello.jpg)
可以看到在 Client Hello 中包含了：TLS Version (客户端支持的最高版本 TLS 1.2)、Random、Cipher Suites (37套)、 Compression Methods (null，表示不压缩)。

2、Server Hello![](/img/ServerHello.jpg)
可以看到在 Server Hello 中包含了：TLS Version (确定双方通信使用 TLS 1.2)、Random、Session ID、Cipher Suite 选择了 TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA、不加密 (Compression Method 为 null)。
其中，TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA 的含义如下：
+ TLS 自然表示是 TLS 协议；
+ ECDHE 表示使用 Elliptic Curve Diffie-Hellman Ephemeral 方式生成对称加密密钥 (Master Secret)；
+ RSA 表示数字签名使用非对称加密 RSA；
+ AES_128_CBC 表示对称加密使用 128 bit 的 AES 方式；
+ SHA 表示使用安全哈希算法 (Secure Hash Algorithm) 计算摘要。

3、Server Certificate![](/img/ServerCertificate.jpg)
4、Server Key Exchange![](/img/ServerKeyChange.jpg)
5、Server Hello Done![](/img/ServerHelloDone.jpg)
Server Hello Done 表示服务端已发送完所有握手相关的数据。
6、Client Key Exchange![](/img/ClientKeyExchange.jpg)
7、Client Change_Cipher_Spec![](/img/ClientChangeCipherSpec.jpg)
Change_Cipher_Spec 表示通知对方后续通信将加密 (此时 Master Secret 已计算出来)
8、Client Finished (Encrypted Handshke Message)![](/img/TLSHandshakeFinished.jpg)
紧接着 Change_Cipher_Spec，会发送一条 Finished 消息，用于验证之前的 key exchange 以及身份认证是否成功。Finished 消息是 TLS 链接建立后的第一条使用握手过程协商的加密算法、密钥加密的消息。Finished 消息的内容是整个握手过程中接收到的数据，对方在接收到 Finished 消息后需要对其解密并验证消息内容是否正确。
9、Server Change_Cipher_Spec
10、Server Finished
服务端同样需要发送 Change_Cipher_Spec、Finished 消息。
至此，若所有步骤都正确无误，则 TLS 握手过程顺利完成！后续所有消息都将通过对称加密的方式发送。

# 小结
____________________________
再来看 HTTPS 的三个主要任务：
+ 加密：通过握手过程协商的加密算法、密钥对消息加密；
+ 完整性校验：通过 MAC 实现；
+ 身份认证：通过数字证书完成。

以上就是 HTTPS 相关的一些知识。

# 未尽事宜
+ 证书管理：证书可能会被 CA 吊销、失效等，终端可以通过 CRL (Certificate Revocation List) 或 OCSP (Online Certificate Status Protocol) 等机制查询证书的状态；
在证书中会包含 CRL、OCSP 相关的信息：![](/img/CRLOCSP.jpg)
+ session 复用：从上述介绍我们可以知道，建立 TLS 链接是十分耗时的，可以通过 session id 以及 session ticket 机制实现链接复用。

# 参考资料
[Transport Layer Security (TLS)](https://hpbn.co/transport-layer-security-tls/)
[The Transport Layer Security (TLS) Protocol Version 1.2](https://tools.ietf.org/html/rfc5246)
[Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile](https://tools.ietf.org/html/rfc5280)
[Transport Layer Security (secure Socket Layer)](http://www.uniroma2.it/didattica/iss/deposito/4_tls.pdf)
[Keyless SSL: The Nitty Gritty Technical Details](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/)
[Certificates and Certificate Authorities (CA)](https://docs.oracle.com/cd/E19693-01/819-0997/gdzen/index.html)