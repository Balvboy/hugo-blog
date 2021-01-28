---
layout:     post
title:      "HTTPS详解 "
description: "HTTPS详解"
author:     "zhouyang"
date:     2021-01-26
published: true 
tags:
    - https
    - 证书
categories: [ tech ]    
---



# 什么是HTTPS
HTTPS简单的说就是安全版的HTTP。
因为HTTP协议的数据都是明文进行传输的，所以对于一些敏感信息的传输就很不安全，为了安全传输敏感数据，网景公司设计了SSL（Secure Socket Layer），在HTTP的基础上添加了一个安全传输层，对所有的数据都加密后再进行传输，客户端和服务器端收到加密数据后按照之前约定好的秘钥解密。

# HTTPS是如何保证安全的
HTTPS的安全性是建立在密码学的基础之上的，有很多算法起到了至关重要的作用。

## HTTPS的交互过程
通过上面的描述，我们已经能大概知道HTTPS是使用加密算法在浏览器和服务器之前传递秘钥，然后再使用秘钥完成信息的加解密。所以这个秘钥是如何生成的，还有秘钥是如何在浏览器和服务器之间传递的就成了HTTPS的关键，下面我们来详细的了解一下这个过程。 

![客户端与服务端交互](/img/Ssl_handshake_with_two_way_authentication_with_certificates.png#center)


### 1.交互流程 
1. Client Hello 客户端（通常是浏览器）先向服务器发出加密通信的请求,请求大概中包括下面内容
    *  支持的协议版本，比如TLS 1.0版。
    *  一个客户端生成的随机数 Random Number-RNc，稍后用于生成"对话密钥"。
    *  支持的加解密方法，比如对称加密支持AES，秘钥交换算法支持RSA、DH，签名算法支持sha256等。(在更高版本的TLS协议中，交换的是密码学套件，所谓的套件就是一整套的加解密，秘钥交换方案)。
    *  支持的压缩方法。
2. 服务器收到请求,然后响应
    *  确认使用的加密通信协议版本，比如TLS 1.0版本。如果浏览器与服务器支持的版本不一致，服务器关闭加密通信。
    *  一个服务器生成的随机数Random Number-RNs，稍后用于生成"对话密钥"。
    *  确认使用的加密方法、秘钥交换算法、签名算法等等。
    *  把服务器证书发送给客户端。
    *  服务器要求验证客户端(浏览器)的证书(可选，大部分的服务器都不会要求)。
3. 客户端收到服务器证书之后，会检查证书的有效性，如果服务器证书并不是经过CA机构认证的，浏览器就会在这个时候给用户提出警告。
4. 客户端收到服务器验证客户端证书的请求，会将自己证书发送给服务器。客户端如果没有证书，则需要发送一个不包含证的证书消息。如果服务器需要客户端身份验证才能继续握手，则可能会使用致命的握手失败警报进行响应。(双向认证一般只存在于银行等一些安全性要求比较高的场景中，像早些时候我们使用的网银，里面存储的就是证书，用来在交易的时候和服务器端进行双向认证，保证安全)
5. 服务器收到客户端证书，校验客户端证书是否有效
6. 客户端把把上面流程中发送的所有消息(除了Client Hello)，使用客户端的私钥进行签名，然后发送给服务器。
7. 服务器也保存着之前客户端发送的消息，然后用客户端发送过来的公钥，进行验签。
8. 客户端生成一个Pre-Master-Secret随机数，然后使用服务器证书中的公钥加密，发送给服务器端。
9. 服务器端收到Pre-Master-Secret的加密数据，因为是使用它的公钥加密的，所以可以使用私钥解密得到Pre-Master-Secret。
10. 这时候客户端和服务端都同时知道了，RNc、RNs、Pre-Master-Secret这三个随机数，然后客户端和服务器端使用相同的PRF算法计算得到一个Master-Secret。然后可以从Master-Secret中再生成作为最终客户端和服务器端消息对称加密的秘钥，和对消息进行认证的MAC秘钥。

参考:[TLS协议](https://zhangbuhuai.com/post/tls.html)
[TLS RFC](https://tools.ietf.org/html/rfc5246)

### 2.Pre-Master-Secret
Pre-Master-Secret前两个字节是TLS的版本号，这是一个比较重要的用来核对握手数据的版本号，因为在Client Hello阶段，客户端会发送一份加密套件列表和当前支持的SSL/TLS的版本号给服务端，而且是使用明文传送的，如果握手的数据包被破解之后，攻击者很有可能串改数据包，选择一个安全性较低的加密套件和版本给服务端，从而对数据进行破解。
所以，服务端需要对密文中解密出来对的Pre-Master-Secret中的版本号跟之前Client Hello阶段的版本号进行对比，如果版本号变低，则说明被篡改，则立即停止发送任何消息。

参考：[pre-master secret ](http://www.linuxidc.com/Linux/2016-05/131147.htm)

### 3.Master-Secret
客户端和服务端在生成Master-Secret的之后，会把Master-Secret作为PRF的参数，继续运算，最终得到下面6个秘钥，分别用于MAC算法和加解密算法。

|秘钥名称|秘钥作用|
| ---------- | ---------- |
|client write MAC key|客户端对发送数据进行MAC计算使用的秘钥，服务端使用同样的秘钥确认数据的完整性|
|server write MAC key|服务端对返回数据进行MAC计算使用的秘钥，客户端使用同一个秘钥验证完整性|
|client write key|对称加密key，客户端数据加密，服务端解密|
|server write key|服务端加密，客户端解密|
|client write IV|初始化向量，运用于分组对称加密|
|server write IV|初始化向量，运用于分组对称加密|


参考: [TLS 中的密钥计算](https://halfrost.com/https-key-cipher/)

### 4.PRF算法

PRF表示（Pseudo-random Function）伪随机函数

Master-secret和最终的6个秘钥都是依靠PRF来生成的。

PRF是利用hash函数来实现，然后依赖递归可以生成无限长度的序列。具体使用哪种hash算吗，在TLS1.2之后需要的密码学套件中指定。

然后master-secret和6个秘钥只需要PRF生成满足他们所需要的长度即可。

比如Master-Secret的长度一直都是48位。

参考: [TLS 中的密钥计算](https://halfrost.com/https-key-cipher/)

### 5.双向认证
其实我们日常访问的绝大多数网站，都是单向认证的(也就是说并没有上面流程中的3、4、5、6步骤)，这里为例展示HTTPS的完整交互流程，所以分析的是双向认证。
服务器可以选择是否要真正客户端的证书。这里以使用NGINX配置HTTPS为例。
如果我们在NGINX的HTTPS相关配置中添加了下面这个配置，就表示需要验证客户端的证书。

```
ssl_verify_client on;
```

### 6.密码学套件
密码学套件是TLS发展了一段时间积累了很多密码学使用的经验之后提出的一整套的解决方案。一个套件中包含了应用于整个握手和传输使用到的所有非对称加密，对称加密和哈希算法，甚至包括证书的类型。

密码学套件是SSLv3开始提出的概念，从此，零散的密码学选择问题变成了一个整体的密码学套件选择的问题。后续的版本在升级的时候会产生新的安全强度更高的密码学套件，同时抛弃比较弱的密码学套件

密码套件分为三大部分：密钥交换算法，数据加密算法，消息验证算法。

下面来分析一个密码学套件的名称，来解释一下它包含的意思

![15898756214901.jpg](/img/15898756214901.jpg#center)

`TLS_DHE_RSA_WITH_AES_256_CBC_SHA226`
* WITH前面表示使用的非对称加密算法，WITH后面表示使用的对称加密和完整性校验算法

1. TLS:表示TLS协议，如果未来TLS改名，这个名字可能会变，否则会一直是这个名字
2. DHE_RSA:这里又两个算法，表示第一个是约定密钥交换的算法，第二个是约定证书的验证算法。如果只有一个，表示这两中操作都是用同一个算法。
3. AES_256_CBC:指的是AES这种对称加密算法的256位算法的CBC模式，AES本身是一类对称加密算法的统称，实际的使用时要指定位数和计算模式，CBC就是一种基于块的计算模式。
4. SHA:表示用来校验数据完整性生成MAC，使用的算法，在TLS1.2之后也表示PRF算法使用的算法。

除了这种比较好理解的密码学套件，还有见到一些比较奇怪的，比如
`ALL:!EXPORT:!LOW:!aNULL:!SSLv2`
我来解释一下，上面出现的字段，更为详细的大家可以去查看下面的文章。
* ALL:表示所有除了明文(eNULL)传递意外的密码学套件
* !EXPORT:EXPORT表示有出口限制的密码学算法，前面加上!,就表示排除掉包含这些算法的密码学套件。
* !LOW:表示排除掉标记为密码强度比较低的算法。
* !aNULL:表示排除不提供身份验证算法的套件。
* !SSLv2:表示排除所有SSLv2的套件。

大家可以使用`openssl ciphers xxx`命令查看套件表达式，所包含的密码学套件

![15898713164120.jpg](/img/15898713164120.jpg#center )


参考: 
[密码学套件](https://zhuanlan.zhihu.com/p/37239435)
[密码学套件表达式](https://www.openssl.org/docs/man1.0.2/man1/ciphers.html)
## HTTPS中的算法
除了了解HTTPS的交互流程，HTTPS中使用的算法及算法的作用，也是我们必须要了解的一部分。
下面会按照算法的作用进行分类，并简单的介绍其中比较常见算法的作用，单并不会对算法的原理做过多的说明(主要是我也弄不明白)，会把相关的讲解原理的文章链接提供出来，大家有兴趣的可以自行去了解。

HTTPS中算法，根据算法的用途可以分为几大类
* 加密算法 - 加密传递的信息，包括对称加密和非对称加密
* 秘钥传递算法 - 在客户端和服务器端传递加密使用的key，当然通过上面的流程我们知道并不是直接传递加密的key
* 信息摘要/签名算法 - 对传递的信息摘要，确保信息在传递过程中不会被篡改

### 1.加密算法

加密算法基本可以分为两种 对称加密和非对称加密

#### 对称加密 

顾名思义就是加密和解密都是用一个同样的秘钥，它的优点就行加解密的速度很快，缺点就是尤其需要注意秘钥的传输，不能泄露。
包含的算法有 AES DES RC4等，常用的是AES

#### 非对称加密-RSA

非对称加密有一对秘钥公钥和私钥。使用公钥加密，然后使用私钥解密。公钥可以公开的发送给任何人。使用公钥加密的内容，只有私钥可以解开。安全性比对称加密大大提高。缺点是和对称加密相比速度较慢，加解密耗费的计算资源较多。

这里我们先只需要了解RSA之所以安全的原因是，大整数的因数分解，是一件非常困难的事情。目前，除了暴力破解，还没有发现别的有效方法。

所以这个大整数越大，RSA被破解的难度也就越高。

具体的算法可以了解下面的两篇文章。

[RSA算法原理1](https://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)
[RSA算法原理2](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)

### 2.秘钥交换算法

常见的秘钥交换算法有下面几种：
* RSA：算法实现简单，诞生于 1977 年，历史悠久，经过了长时间的破解测试，安全性高。缺点就是需要比较大的素数（目前常用的是 2048 位）来保证安全强度，很消耗 CPU 运算资源。RSA 是目前唯一一个既能用于密钥交换又能用于证书签名的算法。

* DH：diffie-hellman 密钥交换算法，诞生时间比较早（1977 年），但是 1999 年才公开。缺点是比较消耗 CPU 性能。

* ECDHE：使用椭圆曲线（ECC）的 DH 算法，优点是能用较小的素数（256 位）实现 RSA 相同的安全等级。缺点是算法实现复杂，用于密钥交换的历史不长，没有经过长时间的安全攻击测试。

* ECDH：不支持 PFS，安全性低，同时无法实现 false start。

* DHE：不支持 ECC。非常消耗 CPU 资源 。

因为这些算法都用到了数论的一些知识，我自己也是似懂非懂。但是他们有一个共同点就是都是运用了质数和模运算相关的数学原理和公式，感觉他们之间应该也是有一定的关联和关系。(后悔没有好好学数学呀)

参考：
[RSA和DH算法](https://blog.csdn.net/u013066244/article/details/79364011?utm_source=blogxgwz4)
[ECDHE-wiki](https://zh.wikipedia.org/wiki/%E6%A9%A2%E5%9C%93%E6%9B%B2%E7%B7%9A%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E9%87%91%E9%91%B0%E4%BA%A4%E6%8F%9B)
[DH-wiki](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)
[Nginx SSL 性能优化](https://luckymrwang.github.io/2015/10/09/Nginx-SSL-%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/)

### 3.摘要/签名算法
HTTPS中常见的有MD5、SHA256、MAC、HMAC等，下面主要说明一下这些算法的区别和联系。

#### 1. MD5
是一种消息摘要算法(Message-Digest Algorithm),一种被广泛使用的密码散列函数(Hash Function)，针对任意长度的输入，可以产生出一个定长128位（16字节）的散列值。

#### 2. SHA
SHA表示安全哈希算法(Secure Hash Algorithm)，经过了很长时间的发展SHA算法已经发展成了一个拥有众多算法的SHA家族。常见的有SHA0、SHA1、SHA2(包含SHA224、SHA256等)、SHA3。0，1，2，3表示SHA算法大的版本，每个大版本中又根据输出字节长度的不同分为和不同的算法。比如SHA256 使用的是SHA2，输出的是256字节。更详细的大家可以看下面wiki百科中的内容，很详细。

SHA称作安全哈希算法的原因是，它相比MD5算法，需要更多的计算次数，最终的输出长度也要长，(SHA0和SHA1是160字节。SHA256是256字节)。如果想要破解需要付出比MD5高的多的计算次数。

经过长时间的发展，MD5和SHA0、SHA1已经被认为不安全，已经不再建议使用。
SHA2是目前被最常使用的算法，目前还没有针对SHA2的有效攻击。
SHA3是2015年才发布的，还没有大规模的取代SHA2。

参考 [SHA算法家族](https://zh.wikipedia.org/wiki/SHA%E5%AE%B6%E6%97%8F)

#### 3. MAC 和 HMAC
相对于上面的MD5和SHA，这两种算法对于我算是比较陌生的。

MAC是消息认证码(Message Authentication Code)的简称。它和上面两种算法的区别是MAC的计算需要一个Key(上面HTTPS流程中就生了计算MAC的KEY)。只有知道了KEY才能正确的计算出对应的MAC。

HMAC的全称是密钥散列消息认证码(Keyed-hash message authentication code)。是指用秘钥并使用Hash散列算法生成认证码的一类算法的总称。

那么MAC算法和HMAC算法是什么关系呢？

我觉得可以这么理解。
MAC只是定义了一个概念---使用一个key，给一段消息生成一个授权码；但是生成这个授权码的算法它并没有定义。所以如果你使用SHA256这种Hash散列算法来生成授权码，那么这种算法就可以被称为HMAC-SHA256。
所以HMAC是MAC的一类实现方式，就像快排是排序算法中的一种实现方式一样。

参考：
[MAC-Wiki](https://zh.wikipedia.org/wiki/%E8%A8%8A%E6%81%AF%E9%91%91%E5%88%A5%E7%A2%BC)
[Difference between MAC and HMAC?](https://crypto.stackexchange.com/questions/6523/what-is-the-difference-between-mac-and-hmac)


#### 4. Salted Hash 和 HMAC
加盐Hash和HMAC在某种程度上很相似，但是在使用场景上还是有很大的区别。目前还有没找到解释的比较好的文章，后面再进行补充
* [ ] todo

# HTTPS证书
现在HTTPS基本已经成为了一个网站的标配。想要给一个网站添加对HTTPS的支持，就需要针对这个网站的域名申请证书。

## HTTPS证书类型

这里顺便说明一下目前的HTTPS证书大概分为3类。
* 域名型HTTPS 证书（DVSSL）：信任等级一般，只需验证网站的真实性便可颁发证书保护网站；
* 企业型HTTPS 证书（OVSSL）：信任等级强，须要验证企业的身份，审核严格，安全性更高；
* 增强型HTTPS 证书（EVSSL）：信任等级最高，一般用于银行证券等金融机构，审核严格，安全性最高，同时可以激活绿色网址栏。

![DV,OV](/img/2829175-5c621e00597c6c00.png#center)

![EV](/img/2829175-ef3241aa68eeb225.png#center)

我们看到越是高等级的证书，审核的严格程度也就越高。并在浏览器中会有一定程度的展示，也会给用户一种更为安全的感觉，当然价格也是更加昂贵。

## HTTPS证书内容和结构
![15898733815306.jpg](/img/15898733815306.jpg#center)

* Certificate
    * Version Number（证书版本）
    * Serial Number(序列号)
    * Signature Algorithm ID（该和客户端使用的签名算法）
    * Issuer Name(证书签发者 DN)
    * Validity period(有效期)
        * Not Before(生效开始时间)
        * Not After(有效结束时间)
    * Subject name(证书使用者)
    * Subject Public Key Info(证书)
        * Public Key Algorithm(公钥算法)
        * Subject Public Key(证书公钥)
    * Issuer Unique Identifier (optional)(签发者唯一身份信息，可选)
    * Subject Unique Identifier (optional)(使用者唯一身份信息，可选)
    * Extensions (optional)(扩展字段)
        * ...
    * Signature(CA机构对证书的签名)


参考 [X.509 wiki](https://en.wikipedia.org/wiki/X.509)
[X.509 数字证书的基本原理及应用](https://zhuanlan.zhihu.com/p/36832100)
[X.509证书的读取与解释](https://blog.csdn.net/dickdick111/article/details/84931413)

## 如何获得HTTPS证书
简单来说获的HTTPS证书有两种方式
* 在有CA认证的机构申请
* 自己生成

### 1.通过CA机构申请
申请CA机构认证的证书大致需要以下步骤

#### 1.1 生成CSR(Certificate Signing Request)文件
主要方式有两种，本地生成和在线生成

- 通过openssl命令本地生成CSR

```
openssl req -new -nodes -sha256 -newkey rsa:2048 -keyout myprivate.key -out mydomain.csr
-new 指定生成一个新的CSR，
nodes指定私钥文件不被加密, 
sha256 指定摘要算法，
keyout生成私钥, 
newkey rsa:2048 指定私钥类型和长度，
最终生成CSR文件mydomain.csr。

```
    
- 通过线上网站生成，一般需要填写下面的内容。
![CSR文件内容](/img/2829175-31655affc6683c0e.png)

线上的工具会把公钥加入到CSR文件中，并同时生成私钥。

参考:[在线CSR申请](https://www.chinassl.net/ssltools/generator-csr.html)


#### 1.2 CA机构对证书签名
接下来就需要按照CA机构的要求，和想要申请的证书类型，提交相关材料。

CA收到CSR并验证相关材料，并审核通过之后。需要进行的很重要的一个步骤就是:`使用CA机构的私钥对提供证书中的内容进行签名`，并把签名的结果存放在证书的数字签名部分。

CA机构签名完，并发送给我们之后，我们就能够把证书部署在我们的服务器中了。

![CA机构进行签名](/img/2829175-06f817d855a63dd7.png#center)

### 2.自己生成证书
参考 [自己生成HTTPS证书](https://www.barretlee.com/blog/2015/10/05/how-to-build-a-https-server/)

大家可以参考上面的文章自己创建一个证书试试。
自己创建的证书同样可以完成上面的步骤。只不过有一点就是，因为自己生成的证书没有得到CA机构的私钥签名，所以当浏览器通过HTTPS握手获得服务器证书的时候，没有办法确定这个证书是否就是来自请求的服务器。
这个时候浏览器就会给出该网站不安全的提示。

![不安全提示](/img/2829175-8749dcdef5960ff5.png#center)

如果客户端不能确认证书是安全，但是却贸然使用，就会有受到中间人攻击的风险。


## 中间人攻击
具体的概念大家可以去下面的文章中了解一下，我们这里直接举例说明一下。

这种中间人攻击的方式，常见于公共的未加密的WIFI。
* A想和B通讯，建立连接，并请求了B的证书。
* C通过提供公共WIFI的方式，监听了A和B的通讯，拦截了B发送给A的证书，并把证书替换成自己的。
* 如果A没有证书检验机制的话，那么A并不能发现证书已经被替换了。
* A还是以为在和B通讯。就会使用已经被替换了的证书中的公钥(也就是C的公钥)进行加密。
* C拦截到消息，使用自己的私钥解密，获得原始的请求数据。然后就能随意篡改，然后使用B的公钥加密，在发送给B，达到了攻击的目的。

中间人攻击还有另外一种方式，危害性更大。
如果有恶意程序在我们的手机或者浏览器中安装了一个证书，并且诱导我们把这书设置为信任证书。
那么如果有其他恶意程序，在和我们的终端进行HTTPS握手的时候，发送给我们使用上面的证书签名的证书，那么我们的终端将无法识别这个恶意证书。导致中间人攻击的发生。

要抵御中间人攻击的一个有效方式就是，充分利用证书的认证机制，还有对于证书的安装和信任要各位谨慎。

参考：[中间人攻击-Wiki](https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%94%BB%E5%87%BB)

## 证书认证链

上面的HTTPS流程中提到，客户端收到服务端证书之后，会进行验证。HTTPS证书的认证是链状的，每一个证书都需要它上级的证书来验证是否有效。

### 链式结构

目前我们常见的证书链中一般分为3级。不过中间证书这部分，也有可能又分为多级。但是也是保持这样的链式结构。

* 根证书(Root CA)
    * 中介(中间)证书(Intermediates)
        * 终端实体证书(End-user)
 
 终端证书的签发者是中间证书。
 中间证书的签发者是上级中间或者根证书。
 根证书的签发者是他自己。
 
 ![image.png](/img/15900283382545.jpg#center)

HTTPS的证书链，是一个自顶向下的信任链，每一个证书都需要它的上级证书来验证有效。所以根证书的作用就尤为重要，如果系统根证书被篡改，系统的安全性就受到威胁。

根证书一般是通过系统，或者浏览器内置到我们的电脑的中的。系统更新或者浏览器更新的时候，也有可能会添加新的根证书。

所以不要轻易的信任根证书，除非你是开发者，了解自己的所作所为。

参考:
[数字证书](https://blog.cnbluebox.com/blog/2014/03/24/shu-zi-zheng-shu/)
 
### 验证证书
 我们以wiki百科的证书链来举例
![image.png](/img/15898763181650.jpg#center)

当我们和wiki的服务器建立HTTPS连接的时候，会获得wiki的(End-User)证书(后面简称为WK证书)。

服务器获得WK证书后，会用下面的几个方式来验证证书。

#### 1.证书有效性时间验证
CA在颁发证书时，都为每个证书设定了有效期，包括开始时间与结束时间。系统当前时间不在证书起止时间的话，都认为证书是无效的。

#### 2.证书完整性验证
上面证书链的时候说到每个证书需要他的上级证书来确保证书的有效性。这个确保的方式就是数字签名。

![image.png](/img/15898784529557.jpg#center)

所以我们只需要，使用CA证书中的公钥和对应的签名算法来验证一下签名，就能够确认证书的有效性。

那么现在有一个问题就是，如何获取WK证书的上一级证书呢？
这里两种可能
1. 这个证书已经存在于你的电脑中了，直接使用就可以
2. 你的电脑中还没有这个证书，需要下载(这里有个疑问就是，根据什么区下载这个证书，很有可能是签发者的DN，但是查了很多资料，并没有找到证据)

拿到CA证书之后，就能够使用CA证书中的公钥对WK证书验签了。

一样的道理，CA证书的有效性需要CA的上级证书，也就是Root证书来证明。验证的过程是基本一致的。

![证书认证链](/img/15898853354686.jpg#center)

参考:
[HTTPS 精读之 TLS 证书校验](https://zhuanlan.zhihu.com/p/30655259)
[证书有效性验证、根证书](https://blog.csdn.net/hqy1719239337/article/details/88891118)

#### 3.IP/域名验证
在申请域名的时候，都会指定证书所针对的域名。所以这里就是验证，当前请求的域名，是否在这个证书所包含的域名列表中。

![image.png](/img/15900391936813.jpg#center)

证书里面的域名范围通常使用通配符来表示。
但是以*.example.com的二级域名范围就不能包含a.b.example.com这个三级域名。

#### 4.证书吊销验证

浏览器获取到服务器证书后，需要判断该证书是不是已经被CA机构吊销。如果已经吊销需要浏览器给出提示。

![image.png](/img/15900300060927.jpg#center)

这里只大概说一下证书的吊销校验主要分2种
1. 通过CRL(Certificate Revocation List) 证书吊销列表
    需要定时的去更新CA机构提供的CRL文件，这个里面记录了改CA机构下所有被吊销的证书。CRL目前正在被OCSP取代，因为CRL不及时，并且每个CRL文件，比较大，影响用户体验。
2. 通过OCSP(Online Certificate Status Protocol) 在线证书状态协议
    这是一个实时的通过证书的序列号去查询证书状态的协议，但是这个有一个问题是，因为每次建立HTTPS链接都需要请求这个接口，所以如果这个接口响应慢的话，十分影响用户的体验。所以需要浏览器这边有一个策略，就是如果在一定时间内OCSP没有响应，那怎么处理。
    如果强依赖OCSP的话，会容易引起OCSP的单点故障。

详细的胡啊
参考：
[你不在意的证书吊销机制](https://www.anquanke.com/post/id/183339)
[PKI体系中的证书吊销](https://wooyun.js.org/drops/SSL%E5%8D%8F%E8%AE%AE%E5%AE%89%E5%85%A8%E7%B3%BB%E5%88%97%EF%BC%9APKI%E4%BD%93%E7%B3%BB%E4%B8%AD%E7%9A%84%E8%AF%81%E4%B9%A6%E5%90%8A%E9%94%80.html)


# 总结
HTTPS的相关总结就先到这里，不知道有没有解决大家的疑问。如果有疑问或者问题，欢迎大家在评论区继续沟通。


# 参考
[HTTPS 基本过程](https://hit-alibaba.github.io/interview/basic/network/HTTPS.html)

[TLS 协议](https://zhangbuhuai.com/post/tls.html)

[数字证书](https://blog.cnbluebox.com/blog/2014/03/24/shu-zi-zheng-shu/)

[你不在意的证书吊销机制](https://www.anquanke.com/post/id/183339)

[PKI体系中的证书吊销](https://wooyun.js.org/drops/SSL%E5%8D%8F%E8%AE%AE%E5%AE%89%E5%85%A8%E7%B3%BB%E5%88%97%EF%BC%9APKI%E4%BD%93%E7%B3%BB%E4%B8%AD%E7%9A%84%E8%AF%81%E4%B9%A6%E5%90%8A%E9%94%80.html)

[HTTPS 精读之 TLS 证书校验](https://zhuanlan.zhihu.com/p/30655259)

[证书有效性验证、根证书](https://blog.csdn.net/hqy1719239337/article/details/88891118)

[数字证书](https://blog.cnbluebox.com/blog/2014/03/24/shu-zi-zheng-shu/)

[中间人攻击-Wiki](https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%94%BB%E5%87%BB)

[自己生成HTTPS证书](https://www.barretlee.com/blog/2015/10/05/how-to-build-a-https-server/)

[在线CSR申请](https://www.chinassl.net/ssltools/generator-csr.html)

[X.509 wiki](https://en.wikipedia.org/wiki/X.509)

[X.509 数字证书的基本原理及应用](https://zhuanlan.zhihu.com/p/36832100)

[X.509证书的读取与解释](https://blog.csdn.net/dickdick111/article/details/84931413)

[MAC-Wiki](https://zh.wikipedia.org/wiki/%E8%A8%8A%E6%81%AF%E9%91%91%E5%88%A5%E7%A2%BC)

[Difference between MAC and HMAC?](https://crypto.stackexchange.com/questions/6523/what-is-the-difference-between-mac-and-hmac)

[SHA算法家族](https://zh.wikipedia.org/wiki/SHA%E5%AE%B6%E6%97%8F)

[RSA和DH算法](https://blog.csdn.net/u013066244/article/details/79364011?utm_source=blogxgwz4)

[ECDHE-wiki](https://zh.wikipedia.org/wiki/%E6%A9%A2%E5%9C%93%E6%9B%B2%E7%B7%9A%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E9%87%91%E9%91%B0%E4%BA%A4%E6%8F%9B)

[DH-wiki](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)

[RSA算法原理1](https://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

[RSA算法原理2](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)

[密码学套件](https://zhuanlan.zhihu.com/p/37239435)

[密码学套件表达式](https://www.openssl.org/docs/man1.0.2/man1/ciphers.html)

[TLS 中的密钥计算](https://halfrost.com/https-key-cipher/)

[pre-master secret ](http://www.linuxidc.com/Linux/2016-05/131147.htm)

[TLS协议](https://zhangbuhuai.com/post/tls.html)

[TLS RFC](https://tools.ietf.org/html/rfc5246)

[Nginx SSL 性能优化](https://luckymrwang.github.io/2015/10/09/Nginx-SSL-%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/)
