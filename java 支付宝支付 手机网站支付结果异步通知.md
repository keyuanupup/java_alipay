对于手机网站支付产生的交易，支付宝会根据原始支付API中传入的异步通知地址notify_url，通过POST请求的形式将支付结果作为参数通知到商户系统。
支付宝开发文档：[https://docs.open.alipay.com/203/105286/](https://docs.open.alipay.com/203/105286/)

这边博客讲的1.8后台通知支付结果 notify_url

![image.png](https://upload-images.jianshu.io/upload_images/5328791-429a56e1c48c5d75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
这个环节的业务逻辑处理为：收到支付宝的回调结果通知，先进行签名校验，如果签名校验结果是对的，然后进行自己的业务逻辑处理（比如辑比如更新订单支付信息成功，保存交易流水，修改订单状态）；

当然支付宝的官方文档还推荐在验签通过之后还需要查询这个支付单号在支付宝服务器的支付结果是否为成功，以这个结果为准；然后再进行支付成功后的业务逻辑处理。主要是为了防止数据篡改和数据的幂等性处理。
```

项目中的具体代码实现：
###controller层的处理：
```
    /**
     * @param request:
     * @param response:
     * @return java.lang.String
     * @Description 支付宝回调
     * @Author huangkeyuan
     * @Date 16:24 2019-12-11
     */
    @PostMapping("notify")
    @ApiOperation("支付宝支付回调，修改订单支付状态")
    public String aliPayCertNotify(HttpServletRequest request, HttpServletResponse response) throws IOException {
        Map<String, String> returnMap = new HashMap<>();
        //获取支付宝POST过来反馈信息转换为Entry
        Set<Map.Entry<String, String[]>> entries = request.getParameterMap().entrySet();
        // 遍历
        for (Map.Entry<String, String[]> entry : entries) {
            String key = entry.getKey();
            StringBuffer value = new StringBuffer("");
            String[] val = entry.getValue();
            if (null != val && val.length > 0) {
                for (String v : val) {
                    value.append(v);
                }
            }
            returnMap.put(key, value.toString());
        }

        log.info("支付宝支付回调,{}", returnMap.toString());

        return paymentService.aliPayCertNotify(returnMap);
    }
```
***
###service层中的处理：
```
 /**
     * @param requestParams : 回调参数
     * @return java.lang.String
     * @Description 支付宝回调(公钥证书方式)
     * @Author huangkeyuan
     * @Date 16:34 2019-12-09
     */
    @PostMapping("alipay/notify")
    @Override
    public String aliPayCertNotify(@RequestParam Map<String, String> requestParams) {

        log.info(">>>支付宝回调参数：" + requestParams.toString());

        Map<String, String> logMap = new HashMap<>();

        try {
            // 获取支付宝的支付key
            boolean flag = aliPayService.checkSignature(requestParams);

            if (flag) {
                log.info(">>>支付宝回调签名认证成功");
                //商户订单号
                String out_trade_no = requestParams.get("out_trade_no");
                //交易状态
                String trade_status = requestParams.get("trade_status");
                //交易金额
                String amount = requestParams.get("total_amount");

                //支付宝交易号
                String trade_no = requestParams.get("trade_no");

                if ("TRADE_SUCCESS".equals(trade_status) || "TRADE_FINISHED".equals(trade_status)) {
                    log.info("支付宝回调支付成功，trade_status:{}", trade_status);
                    /**
                     * 自己的业务处理
                     */
                    
                    // 执行支付成功后的商城业务逻辑比如更新订单支付信息成功，保存交易流水，修改订单状态

                    mlogger.info(PaymentStatusCode.ALI_PAY_NOTIFY_SUCCESS_CODE, PaymentStatusCode.ALI_PAY_NOTIFY_SUCCESS_MESSAGE, logMap);
                    // 修改订单状态成功
                    log.info("支付宝支付回调修改订单、支付状态成功，结果码：" + trade_status + ",支付单号：" + out_trade_no + "支付宝支付单号：" + trade_no);

                } else {
                    log.error("没有处理支付宝回调业务，支付宝交易状态：{},params:{}", trade_status, requestParams);
                    mlogger.info(PaymentStatusCode.ALI_PAY_NOTIFY_SERVER_RESPONSE_VALIDATION_ERRORB_CODE,
                            PaymentStatusCode.ALI_PAY_NOTIFY_SERVER_RESPONSE_VALIDATION_ERRORB_MESSAGE, logMap);
                }
            } else {
                log.info("支付宝回调签名认证失败，signVerified=false, params:{}", requestParams);
                mlogger.info(PaymentStatusCode.ALI_PAY_NOTIFY_SERVER_INTERNAL_ERROR_CODE,
                        PaymentStatusCode.ALI_PAY_NOTIFY_SERVER_INTERNAL_ERROR_MESSAGE, logMap);

                return "failure";
            }
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            log.info("支付宝回调签名认证失败，signVerified=false, params:{}", requestParams);
            mlogger.info(PaymentStatusCode.ALI_PAY_NOTIFY_SERVER_INTERNAL_ERROR_CODE,
                    PaymentStatusCode.ALI_PAY_NOTIFY_SERVER_INTERNAL_ERROR_MESSAGE, logMap, e);
            return "failure";
        }
        return "success";//请不要修改或删除
    }
```
third-party-service中的处理
```
/**
     * @param requestParams :
     * @return boolean
     * @Description 收到支付宝的成功回调之后验证签名是否正确
     * @Author huangkeyuan
     * @Date 17:01 2019-12-12
     */
    @PostMapping("check/signature")
    @Override
    public boolean checkSignature(Map<String, String> requestParams) {
        try {
            String alipaypublickey = this.getAlipayPublicKey(aliPayConfig.getALIPAY_CERT_PATH());
            log.info("读取服务器的支付宝公钥证书,{}", alipaypublickey);
            //切记alipaypublickey是支付宝的公钥，请去open.alipay.com对应应用下查看。
            boolean flag = AlipaySignature.rsaCheckV1(requestParams, alipaypublickey, aliPayConfig.getCHARSET(),
                    aliPayConfig.getSIGN_TYPE());

            return flag;
        } catch (Exception e) {
            return false;
        }
    }
```

##遇到的问题
直接打印支付宝返回的参数是这样的：
```
支付宝回调参数：{gmt_create=[Ljava.lang.String;@362e073d, charset=[Ljava.lang.String;@1445a60e, seller_email=[Ljava.lang.String;@20f2aa67, subject=[Ljava.lang.String;@30c863fd, sign=[Ljava.lang.String;@76198d36, body=[Ljava.lang.String;@4fd5202d, buyer_id=[Ljava.lang.String;@7a410a22, invoice_amount=[Ljava.lang.String;@965491c, notify_id=[Ljava.lang.String;@33af053b, fund_bill_list=[Ljava.lang.String;@551d8f2b, notify_type=[Ljava.lang.String;@27ff9fb0, trade_status=[Ljava.lang.String;@4f11ffa, receipt_amount=[Ljava.lang.String;@4fbc4482, buyer_pay_amount=[Ljava.lang.String;@76e1ee89, app_id=[Ljava.lang.String;@2d38edfa, sign_type=[Ljava.lang.String;@21ba2968, seller_id=[Ljava.lang.String;@c0ff189, gmt_payment=[Ljava.lang.String;@7563d327, notify_time=[Ljava.lang.String;@1fdeb74c, version=[Ljava.lang.String;@6f5f3cb6, out_trade_no=[Ljava.lang.String;@1c2f1b6d, total_amount=[Ljava.lang.String;@77be0924, trade_no=[Ljava.lang.String;@1db0b448, auth_app_id=[Ljava.lang.String;@b1c81c4, buyer_logon_id=[Ljava.lang.String;@6f8c07b9, point_amount=[Ljava.lang.String;@68fcd445}
```
需要通过这样数据处理才能转换成我们常用的map结构，方便项目中数据分析和使用
```
Map<String, String> returnMap = new HashMap<>();
        //获取支付宝POST过来反馈信息转换为Entry
        Set<Map.Entry<String, String[]>> entries = request.getParameterMap().entrySet();
        // 遍历
        for (Map.Entry<String, String[]> entry : entries) {
            String key = entry.getKey();
            StringBuffer value = new StringBuffer("");
            String[] val = entry.getValue();
            if (null != val && val.length > 0) {
                for (String v : val) {
                    value.append(v);
                }
            }
            returnMap.put(key, value.toString());
        }
```
