承接上文，我们已经罗列了需要调用的三个接口；接下来我们先用第一个接口说起。

支付宝官方文档：[https://docs.open.alipay.com/203/105285/](https://docs.open.alipay.com/203/105285/)
![image.png](https://upload-images.jianshu.io/upload_images/5328791-6b2984e094fe128c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
文档的api是提供大致的参考，可以理解为伪代码。比较有用的是你可以看一下他大致的传参和这个接口的接口名称「alipayClient.pageExecute」

手机网站支付的具体代码：
```
    @PostMapping("certunified/order")
    @Override
    public H5AlipayResponseDTO aliPayCertUnifiedOrder(@RequestBody H5AliPayRequestDTO h5AliPayRequestDTO) {
        H5AlipayResponseDTO h5AlipayResponseDTO = new H5AlipayResponseDTO();

        /**
         * 以下部分可以共用，复制即可
         * *********************************************************
         */
        //实例化具体API对应的request类,类名称和接口名称对应,当前调用接口名称：alipay.trade.app.pay
        AlipayTradeWapPayRequest request = new AlipayTradeWapPayRequest();
        //SDK已经封装掉了公共参数，这里只需要传入业务参数。以下方法为sdk的model入参方式(model和biz_content同时存在的情况下取biz_content)。
        AlipayTradeWapPayModel model = new AlipayTradeWapPayModel();
        model.setBody(h5AliPayRequestDTO.getBody());
        model.setSubject(h5AliPayRequestDTO.getSubject());
        model.setOutTradeNo(h5AliPayRequestDTO.getOutTradeNo());
        model.setTotalAmount(h5AliPayRequestDTO.getTotalAmount() + "");
        model.setProductCode(h5AliPayRequestDTO.getProductCode());
        request.setBizModel(model);
        request.setNotifyUrl(aliPayConfig.getALIPAY_NOTIFY_URL());
        String returnFinishUrl = aliPayConfig.getRETURNURL() + h5AliPayRequestDTO.getReturnUrl();
        request.setReturnUrl(returnFinishUrl);
        log.info(">>>>支付宝统一下单接口请求参数：" + model.getBody() + "," + model.getOutTradeNo() + "," + model.getTotalAmount());

        /**实例化客户端*/
        CertAlipayRequest certAlipayRequest = new CertAlipayRequest();
        certAlipayRequest.setServerUrl(aliPayConfig.getSERVERURL());
        certAlipayRequest.setAppId(aliPayConfig.getAPPID());
        certAlipayRequest.setPrivateKey(aliPayConfig.getAPP_PRIVATE_KEY());
        certAlipayRequest.setFormat(aliPayConfig.getFORMAT());
        certAlipayRequest.setCharset(aliPayConfig.getCHARSET());
        certAlipayRequest.setSignType(aliPayConfig.getSIGN_TYPE());
        certAlipayRequest.setCertPath(aliPayConfig.getAPP_CERT_PATH());
        certAlipayRequest.setAlipayPublicCertPath(aliPayConfig.getALIPAY_CERT_PATH());
        certAlipayRequest.setRootCertPath(aliPayConfig.getALIPAY_ROOT_CERT_PATH());
        try {
            //构造client
            AlipayClient alipayClient = new LMAlipayClient(certAlipayRequest);
            String form  = alipayClient.pageExecute(request).getBody();

            //就是orderString 可以直接给客户端请求，无需再做处理。
            log.info(">>>生成调用支付宝参数" + form);
            h5AlipayResponseDTO.setOrderString(form);
            h5AlipayResponseDTO.setReturn_code("200");
            h5AlipayResponseDTO.setReturn_msg("支付完成");
        } catch (AlipayApiException e) {
            log.error(e.getMessage(), e);
            h5AlipayResponseDTO.setReturn_code("500");
            h5AlipayResponseDTO.setReturn_msg("支付异常");
        }

        return h5AlipayResponseDTO;
    }
```

#注意点 
看到我这篇文档的同学肯定有做APP支付和h5（wap）支付的，这两个支付调用的参数和接口名称有区别。我当时找到文档就犯了一个错误。我要做的是h5（wap）支付但是调用的方法用的是APP的。所以在这里我把两个不同的api都写出来，避免调错。主要的不同点在于AlipayTradeWapPayModel、AlipayTradeWapPayRequest、productCode和alipayClient.pageExecute

## h5（wap）支付
```
AlipayTradeWapPayModel、AlipayTradeWapPayRequest、productCode=QUICK_WAP_PAY和alipayClient.pageExecute
```
![image.png](https://upload-images.jianshu.io/upload_images/5328791-7e6012282b11c1b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##APP支付
```
AlipayTradeAppPayModel、AlipayTradeAppPayRequest、productCode=QUICK_MSECURITY_PAY和alipayClient.execute
```

##开发的时候要注意这几个参数是否调用正确，根据自己的业务选择正确的参数。支付宝申请退款接口也需要注意这三个参数是否正确。

***

###注意项1、notifyUrl
notifyUrl也就是支付成功的通知地址，这个接口写在服务端，如果网站有鉴权校验的处理要去掉可以直接让支付宝调到。

###注意项2、returnUrl
returnUrl是完成支付后，跳转到前端h5对应的地址，这个地址由前端提供；也就是说支付完成后，需要跳转到前端哪一个页面就填上相对应的链接，可以带上自己业务需要的参数。我们项目这里放的就是支付成功的页面，可以选择跳转到订单详情或者首页。这个页面的作用是让用户知道自己的支付是成功的。
  还有一点，经过测试之后发现，这个returnUrl只有在支付宝客户端中完成支付后才会跳转。如果你在支付宝客户端中取消付款或者返回是不会触发跳转到这个页面的。
  这个和微信支付的h5支付（非微信浏览器）支付处理的逻辑不一样；微信支付的redirect_url是支付是否完成都会跳到这个页面。所以做微信支付的时候还需要对接微信的支付订单状态查询接口，跳转回去h5之后需要查询支付在微信是否完成。微信支付的redirect_url会在调起微信客户端超过5秒后就返回到这个地址；
微信h5支付文档链接：[https://pay.weixin.qq.com/wiki/doc/api/H5.php?chapter=15_4](https://pay.weixin.qq.com/wiki/doc/api/H5.php?chapter=15_4)

具体参看下图：
![image.png](https://upload-images.jianshu.io/upload_images/5328791-c2eae751bfe3affc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###注意项3、公钥证书的读取
公钥证书在我们项目中是优化成相对路径读取
