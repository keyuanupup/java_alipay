alipay.trade.refund(统一收单交易退款接口) 

 > 当交易发生之后一段时间内，由于买家或者卖家的原因需要退款时，卖家可以通过退款接口将支付款退还给买家，支付宝将在收到退款请求并且验证成功之后，按照退款规则将支付款按原路退到买家帐号上。 交易超过约定时间（签约时设置的可退款时间）的订单无法进行退款 支付宝退款支持单笔交易分多次退款，多次退款需要提交原支付订单的商户订单号和设置不同的退款单号。一笔退款失败后重新提交，要采用原来的退款单号。总退款金额不能超过用户实际支付金额

支付宝开发文档参考 [https://docs.open.alipay.com/api_1/alipay.trade.refund](https://docs.open.alipay.com/api_1/alipay.trade.refund)

 项目中具体代码实现：
```
/**
     * @param alipayRefundRequestDTO :
     * @return java.lang.String
     * @Description 支付宝退款接口
     * @Author huangkeyuan
     * @Date 20:21 2019-12-12
     */
    @PostMapping("refund")
    @Override
    public String alipayRefund(@RequestBody AlipayRefundRequestDTO alipayRefundRequestDTO) {

        //实例化具体API对应的request类,类名称和接口名称对应,当前调用接口名称：
        AlipayTradeRefundRequest request = new AlipayTradeRefundRequest();
        //SDK已经封装掉了公共参数，这里只需要传入业务参数。以下方法为sdk的model入参方式(model和biz_content同时存在的情况下取biz_content)。
        AlipayTradeRefundModel model = new AlipayTradeRefundModel();

        model.setOutTradeNo(alipayRefundRequestDTO.getOuttradeno());
        model.setOutRequestNo(alipayRefundRequestDTO.getOutrefundno());
        model.setRefundAmount(alipayRefundRequestDTO.getRefundfee());
        request.setBizModel(model);

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
            AlipayTradeRefundResponse response = alipayClient.execute(request);
            log.info("支付宝退款返回信息,{}",response.getBody());
            return response.getBody();
        } catch (AlipayApiException e) {
            log.error(e.getMessage(), e);
            return null;
        }
    }
```

##注意点
###1.参数
```
1.outTradeNo 支付的时候传入的商家的支付订单编号
2.outRequestNo 本次退款的退款单号，标识一次退款请求，同一笔交易多次退款需要保证唯一，如需部分退款，则此参数必传。
3.refundAmount 退款金额
```
###2.签名错误
在签名调用支付下单接口时，签名顺利通过了，本以为这里退款接口的签名应该也没问题；但是确实出现签名错误的问题；
  
后面找到原因是下单和退款的接口调用的api不同所以后面走的签名方法也不同。问题出现在：退款接口的签名在读取appCertSN为空。 做了修改之后签名就对了；
附上修改的方法：
```
private <T extends AlipayResponse> T _execute(AlipayRequest<T> request, AlipayParser<T> parser, String authToken, String appAuthToken) throws AlipayApiException {
        long beginTime = System.currentTimeMillis();
        Map<String, Object> rt = this.doPost(request, authToken, appAuthToken, this.appCertSN);
        if (rt == null) {
            return null;
        } else {
            Map<String, Long> costTimeMap = new HashMap();
            if (rt.containsKey("prepareTime")) {
                costTimeMap.put("prepareCostTime", (Long)((Long)rt.get("prepareTime")) - beginTime);
                if (rt.containsKey("requestTime")) {
                    costTimeMap.put("requestCostTime", (Long)((Long)rt.get("requestTime")) - (Long)((Long)rt.get("prepareTime")));
                }
            }

            AlipayResponse tRsp = null;

            try {
                ResponseEncryptItem responseItem = this.decryptResponse(request, rt, parser);
                tRsp = parser.parse(responseItem.getRealContent());
                tRsp.setBody(responseItem.getRealContent());
                this.checkResponseSign(request, parser, responseItem.getRespContent(), tRsp.isSuccess());
                if (costTimeMap.containsKey("requestCostTime")) {
                    costTimeMap.put("postCostTime", System.currentTimeMillis() - (Long)((Long)rt.get("requestTime")));
                }
            } catch (RuntimeException var11) {
                AlipayLogger.logBizError((String)rt.get("rsp"), costTimeMap);
                throw var11;
            } catch (AlipayApiException var12) {
                AlipayLogger.logBizError((String)rt.get("rsp"), costTimeMap);
                throw new AlipayApiException(var12);
            }

            tRsp.setParams((AlipayHashMap)rt.get("textParams"));
            if (!tRsp.isSuccess()) {
                AlipayLogger.logErrorScene(rt, tRsp, "", costTimeMap);
            } else {
                AlipayLogger.logBizSummary(rt, tRsp, costTimeMap);
            }

            return (T)tRsp;
        }
    }

```

