##一、公钥证书在项目中的结构
![image.png](https://upload-images.jianshu.io/upload_images/5328791-bcdf903cfb8dc2d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##二、读取配置
![image.png](https://upload-images.jianshu.io/upload_images/5328791-60c0611d8e446a89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##三、配置文件
```
#商户私钥
ALIPAY.APP_PRIVATE_KEY=MIIExxx

#支付宝APPID
ALIPAY.APPID=2019xxx

#应用公钥证书路径
ALIPAY.APP_CERT_PATH=/CRT/appCertPublicKey_20191203xxx.crt

#支付宝公钥证书文件路径
ALIPAY.ALIPAY_CERT_PATH=/CRT/alipayCertPublicKey_RSA2.crt

#支付宝CA根证书文件路径
ALIPAY.ALIPAY_ROOT_CERT_PATH=/CRT/alipayRootCert.crt

#请求网关
ALIPAY.SERVERURL=https://openapi.alipay.com/gateway.do

#支付成功的通知地址
ALIPAY.ALIPAY_NOTIFY_URL=http://www.example.com/front/payment/alipay/notify

#字符集
ALIPAY.CHARSET=utf-8

#签名类型
ALIPAY.SIGN_TYPE=RSA2

#格式
ALIPAY.FORMAT=json

#h5支付完成之后的回调地址
ALIPAY.RETURNURL = http://www.example.com/#/pages/money/paySuccess

#支付方式类型（h5或者wap）
ALIPAY.PAYTYPEWAP = QUICK_WAP_PAY

#支付方式类型（app）
ALIPAY.PAYTYPEAPP = QUICK_MSECURITY_PAY
```

##四、配置目录
![image.png](https://upload-images.jianshu.io/upload_images/5328791-07b6ef062a0faca5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码如下：
```
package com.leimingtech.config.alipay;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

/**
 * @param :
 * @Description 读取支付宝配置信息
 * @Author huangkeyuan
 * @Date 15:59 2019-12-09
 * @return
 */
@Data
@Component
@ConfigurationProperties(prefix = "alipay")
@PropertySource(value = "alipay/alipay-${spring.profiles.active}.properties")
public class AliPayConfig {
    /**
     * 商户私钥
     */
    public String APP_PRIVATE_KEY;
    /**
     * 支付宝APPID
     */
    public String APPID;

    /**
     * 应用公钥证书路径
     */
    public String APP_CERT_PATH;

    /**
     * 支付宝公钥证书文件路径
     */
    public String ALIPAY_CERT_PATH;

    /**
     * 支付宝CA根证书文件路径
     */
    public String ALIPAY_ROOT_CERT_PATH;

    /**
     * 请求网关
     */
    public String SERVERURL;

    /**
     * 支付成功的通知地址
     */
    public String ALIPAY_NOTIFY_URL;

    /**
     * 字符集
     */
    public String CHARSET;

    /**
     * 签名类型
     */
    public String SIGN_TYPE;

    /**
     * 格式
     */
    public String FORMAT;

    /**
     * h5支付完成之后的回调地址
     */
    public String RETURNURL;

    /**
     * 支付方式类型（h5或者wap）
     */
    public String PAYTYPEWAP;

    /**
     * 支付方式类型（app）
     */
    public String PAYTYPEAPP;

}

```
