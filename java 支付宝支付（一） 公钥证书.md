  最近接到任务是负责写支付宝的支付接口；以前一直写过iOS的支付宝支付，这次我要负责写java后台的支付宝接口还有前端h5的支付宝调用。这个任务还是挺有挑战的。一周过去了，中间也遇到很多问题，最多出现的问题就是签名失败，在「外部商户创建订单并支付」和「alipay.trade.refund交易退款接口」都是签名失败的错误。最后也是顺利的完成了支付宝支付这个功能。觉得很有必要记录下来。
#一、准备工作
这个部分主要参考支付宝的接入文档
[支付宝手机网站（wap或者h5）支付]([https://docs.open.alipay.com/203/107084/](https://docs.open.alipay.com/203/107084/)
)
![image.png](https://upload-images.jianshu.io/upload_images/5328791-cd7932158f944837.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接入完成后看到的应用信息大概是这样的：
![image.png](https://upload-images.jianshu.io/upload_images/5328791-47fd9dd75068e47a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#二、签名
签名方式有两种(普通公钥方式、公钥证书方式),一般最常用的就是普通公钥方式，也相对比较简单，我做的时候用的是公钥证书方式；这个证书方式是现在支付宝支付官方文档上面推荐的签名方式，但是在网上可以查到的资料比较少。这是我写这篇博客的目的可以让后面还有做支付宝支付并用的签名方式是公钥证书签名的方式的开发同学可以少踩一些坑。
申请步骤可以参照官方文档：
参考链接：https://docs.open.alipay.com/291/105971/
按照文档一步步就可以完成生成并上传公钥，在这里使用开放平台的SDK方式进行签名及验证，也是最简单最高效的方式。
![image.png](https://upload-images.jianshu.io/upload_images/5328791-4b5efae03c18f876.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按照文档可以一步步的生成公钥证书及私钥相关的文件。一共有三个：
![image.png](https://upload-images.jianshu.io/upload_images/5328791-b6586dd7a8654199.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里需要提醒一句的是：后续在使用这三个证书的时候，支付宝里面的sdk的接口要求给的是证书所在的绝对路径；读取绝对路径在我们项目spring-cloud中会会出现在本地电脑开发可以顺利读取到，但是部署到线上服务器环境上会存在读取不到的情况，这个问题也是我在开发中遇到的比较困难的。

好在最后在同事的帮助下解决了，同时读取证书也调整为读取相对路径；因为在我们的开发中有三套不同的环境分别是开发环境、测试环境、生产环境。我相信大多数公司也是基本是这三个环境，到项目后期的话可能还会有sit和uat环境。

所以证书的读取很有必要使用相对路径，否则每个环境你需要用绝对路径的话，在服务器部署方面会存在麻烦。这部分的处理在后续的博客也会提到。当然如果你们的环境很少只有一个，你也不想麻烦只是想快速的把支付功能开发完也可以用绝对路径。

我还是建议使用相对路径。

接下来主要实现服务端代码：

#三、导入依赖（项目中使用的是4.5.0.ALL版本，后续支付宝更新的sdk版本可能会和博客中写的有出入；本文档以及后续文档都是基于4.5.0.ALL版本的编写）
```
<!-- 支付宝支付 -->
        <dependency>
            <groupId>com.alipay.sdk</groupId>
            <artifactId>alipay-sdk-java</artifactId>
            <version>4.5.0.ALL</version>
        </dependency>
```
#四、梳理需要调用的支付宝的接口
我们这个项目是电商项目，所以涉及到和支付宝交互调用的接口有以下几个：
```
1.手机网站支付alipay.trade.wap.pay：
2.手机网站支付结果异步通知 
3.交易退款接口alipay.trade.refund：
```


