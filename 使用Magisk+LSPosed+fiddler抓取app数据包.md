# Magisk+LSPosed+TrustMeAlready绕过SSL PING抓包

## 1.前言

关于如何使用fiddler、burpsuite或者Charles抓取app端数据包的方式大同小异，具体使用哪一款属于可根据自己喜好。具体使用方式这里不再赘述，见fiddler抓包一文。

说回正文，当我们测试APP进行抓包的时候的时候，有时候会碰到如下情况，APP内提示：网络异常。

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250228153956992.png" alt="image-20250228153956992" style="zoom:50%;" />

而在抓包软件（本人更习惯使用fiddler）中显示的情况如下：

![image-20250228154046179](https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250228154046179.png)

这时候很多人的第一反应是证书问题，可能是fidder证书没有导入进来，确认一遍又一遍之后还是不行。

**那究竟是不是证书问题呢**？

是证书问题！但是不是你想的那样fidder的内置证书问题，而是APP的内置证书--**SSL PING机制**

**什么是SSL PING机制？**

SSL PING将服务器提供的SSL/TLS证书内置到移动端开发的APP客户端中，当客户端发起请求时，通过比对内置的证书和服务器端证书的内容，以确定这个连接的合法性。

SSL/TLS Pinning提供了两种锁定方式： **Certificate Pinning（证书锁定）** 和**Public Key Pinning（公钥锁定）**
*1*.证书锁定
我们需要将APP代码内置仅接受指定域名的证书，而不接受操作系统或浏览器内置的CA根证书对应的任何证书，通过这种授权方式，保障了APP与服务端通信的唯一性和安全性，因此我们移动端APP与服务端（例如API网关）之间的通信是可以保证绝对安全。但是CA签发证书都存在有效期问题，所以缺点是在证书续期后需要将证书重新内置到APP中。
*2*.公钥锁定
公钥锁定则是提取证书中的公钥并内置到移动端APP中，通过与服务器对比公钥值来验证连接的合法性，我们在制作证书密钥时，公钥在证书的续期前后都可以保持不变（即密钥对不变），所以可以避免证书有效期问题。

开发者预先把证书相关信息预置到App中再打包，在https的建立连接过程中，当客户端向服务端发送了连接请求后，服务器会发送自己的CA证书给客户端，这样在通讯过程中App本地可以与服务器返回的CA证书可以做合法性校验，如果锁定过程失败，那么客户端APP将拒绝针对服务器的所有 SSL/TLS 请求。在新版本**(安卓7.0及以上版本)**的系统规则中，**应用只信任系统默认预置的CA证书，如果是第三方安装的证书（比如Fiddler安装的）将不会信任**。

如图所示，系统默认预置证书和用户证书

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250228165743711.png" alt="image-20250228165743711" style="zoom:50%;" />         <img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250228165841714.png" alt="image-20250228165841714" style="zoom:50%;" />

当我们使用抓包工具抓包时，抓包工具在拦截了服务端返回的内容并重新发给客户端的时候使用证书不是服务器端原来的证书，而是抓包工具自己的，抓包工具原来的证书并不是APP开发者设定的服务端原本的证书，于是就构成了中间人攻击，触发SSLPinning机制导致通信中断，所以我们无法直接抓到包。

知道原因之后，我们实际可以有一下几个方案：

方案一：绕过新版本，新规则只在安卓7.0及以上系统使用，使用旧版系统（不切实际，很多应用不适配7.0以前的系统）

方案二：绕过信任区，将用户证书安装到系统默认预置CA证书区域（需要root，有兴趣可以参考[安卓高版本安装系统证书 HTTPS 抓包](https://www.jianshu.com/p/5f72ab33a342)）

方案三（推荐）：本文提及的Magisk+LSPosed+TrustMeAlready方案（此方法针对高版本安卓，低版本安卓使用Xposed+JustTrustMe）

方案四（针对自研app）：添加授权，对于自研app，打包时可额外添加信任的用户CA证书

1、在network_security_config.xml中添加指定信任用户证书

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
          <!-- 信任系统预置CA 证书 -->
          <certificates src="system" />
          <!-- 信任用户添加的 CA 证书，Charles 和 Fiddler 抓包工具安装的证书属于此类 -->
          <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>

```

2、在Manifest中添加配置

```
<?xml version="1.0" encoding="utf-8"?>
<manifest ... >
    <application android:networkSecurityConfig="@xml/network_security_config"
                    ... >
        ...
    </application>
</manifest>

```



## 2.准备工作

所需软件清单，附上下载地址：

![image-20250228172755956](https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250228172755956.png)

Magisk下载地址：[Github_Kitsune Magisk](https://huskydg.github.io/magisk-files/)

LSPosed-zygisk-release.zip下载地址：[Github_LSPosed](https://github.com/LSPosed/LSPosed/releases)

TrustMeAlready下载地址：[Github_TrustMeAlready](https://github.com/ViRb3/TrustMeAlready/releases)

[百度网盘]( https://pan.baidu.com/s/14WEtIYfySziNertJunVFaw )提取码: vupr （全部合集）

## 3.使用教程

### 1、安装模拟器

这里不再赘述，**注意：需要开启模拟器root权限**

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250228174440705.png" alt="image-20250228174440705" style="zoom:50%;" />

### 2、安装/设置Magisk Delta

安装Magisk Delta，安装完成后打开，点击“允许”超级用户权限

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304203257456.png" alt="image-20250304203257456" style="zoom:50%;" />

点击Magisk右边的安装

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304203408987.png" alt="image-20250304203408987" style="zoom:50%;" />

模拟器提醒请求访问权限，点击“允许”

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304203529148.png" alt="image-20250304203529148" style="zoom:50%;" />

<font color=red>**取消勾选**</font>，并点击下一步

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304203718457.png" alt="image-20250304203718457" style="zoom:50%;" />

选择“**直接安装(直接修改/system)**”，然后点击“开始”， 开始安装。<font color=red>**如果没有看到该选项，可以在机型设置中更换品牌型号，重启模拟器。**</font>

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304204044924.png" alt="image-20250304204044924" style="zoom:50%;" />

安装完成后重启手机（建议手动重启）

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304204632089.png" alt="image-20250304204632089" style="zoom:50%;" />

重启后再次进入Magisk，进入后可能会提醒“检测到不属于Maigsk的su文件，请删除其他超级用户程序”。此提醒可以忽略不计，因为模拟器的系统已经内置了su文件，这个文件用于获取超级权限，而安装magisk后也会有个su文件，所以程序提示冲突，但是不影响使用。

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304204853201.png" alt="image-20250304204853201" style="zoom:50%;" />

这时，我们可以看到我们已经成功安装了，下面的超级用户和模块都已经是可使用状态

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304205152533.png" alt="image-20250304205152533" style="zoom:50%;" />

点击设置，打开Zygisk并重启

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304205657404.png" alt="image-20250304205657404" style="zoom:33%;" />             <img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304210103343.png" alt="image-20250304210103343" style="zoom:25%;" />

先将LSPosed-zygisk-release.zip 移动到模拟器中。点击模块，选择从本地安装，选择LSPosed-zygisk-release.zip进行安装

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304210331322.png" alt="image-20250304210331322" style="zoom:33%;" />    <img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304210705997.png" alt="image-20250304210705997" style="zoom: 50%;" /><img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304210859747.png" alt="image-20250304210859747" style="zoom:33%;" />

安装完成后，继续重启

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304210949204.png" alt="image-20250304210949204" style="zoom:33%;" />

### 3、LSPoed+TrustMeAlready

安装LSPosed.APK

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304211433958.png" alt="image-20250304211433958" style="zoom: 33%;" />

安装TrustMeAlready.apk

打开LSPosed，系统显示已激活，点击下方模块

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304211855094.png" alt="image-20250304211855094" style="zoom:50%;" />

选择TrustMeAlready，点击“启用模块”，勾选需要抓包的app

<img src="https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304212122887.png" alt="image-20250304212122887" style="zoom:50%;" />

## 效果演示

现在使用fiddler抓包，成功get！ （fiddler使用方法fiddler抓包一文）

![image-20250304221955287](https://raw.githubusercontent.com/y4ngwei/testNx/main/img/image-20250304221955287.png)
