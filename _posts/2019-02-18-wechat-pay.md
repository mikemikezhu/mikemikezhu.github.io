---
layout: post
title:  "用Spring框架实现微信小程序支付功能"
categories: [Backend Dev]
tag: [wechat-pay, spring]
---

{:toc}

## 前言

近期在帮家里做微信小程序的时候，涉及到了微信支付功能，故尝试用主流的Java框架SpringMVC实现了微信支付这一功能。本文将主要向大家介绍实现微信支付功能的具体步骤。

<br>

## 开发前准备

在开发小程序的微信支付功能之前，我们需要完成以下几个准备工作。

### 1. 小程序微信认证

(1) 首先，在浏览器中打开[微信公众平台]([https://mp.weixin.qq.com/wxopen/waregister?action=step1)，从而注册微信小程序。若已有微信小程序，可以直接选择登录。

(2) 其次，在首页中找到微信认证这一步骤。

(3) 当完成微信认证后，我们可以在“开发->开发设置”这里找到小程序对应的AppId以及小程序密钥AppSecret。

![小程序对应的AppId](/assets/img/2019-02-18-wechat-pay/wechat-id.png)

<br>

### 2. 微信支付认证

(1) 在[微信公众平台]([https://mp.weixin.qq.com/wxopen/waregister?action=step1)，选择“微信支付->支付申请”，从而前往申请微信支付的认证。

(2) 申请成功以后，我们会收到来自微信支付认证的邮件。里面包含微信支付的商户号，也就是后面要用到的mch_id。

![微信支付商户号](/assets/img/2019-02-18-wechat-pay/mail.png)

<br>

### 3. 商户授权

(1) 登录[微信支付商户平台](https://pay.weixin.qq.com)。

(2) 点击“产品中心->APPID授权管理->新增授权”，进入授权申请页面。

![商户授权申请](/assets/img/2019-02-18-wechat-pay/authorization.png)

(3) 填写需要使用这个商户号的小程序的AppId并发起授权。

(4) 登录[微信公众平台]([https://mp.weixin.qq.com/wxopen/waregister?action=step1)，点击“微信支付->商户号管理”，确认授权申请。

<br>

### 4. 设置API密钥

在进行微信支付的过程中，我们需要对支付信息进行加密签名。为保证API安全，需要在[微信支付商户平台](https://pay.weixin.qq.com)中设置API密钥。

点击“账户中心->API安全->设置API密钥”。API密钥属于敏感信息，请妥善保管不要泄露，如果怀疑信息泄露，则需要重设密钥。

![API密钥](/assets/img/2019-02-18-wechat-pay/key.png)

<br>

## 开发步骤

### 1. 登录小程序

在微信支付的过程中，我们的后台需要唯一用户标识openid等信息。然而，出于安全考虑，微信后台不会直接把openid发送给小程序，而是需要小程序先请求微信后台，获取临时登录凭证code。
然后，小程序再将此code发送给我们自己的服务器后台，后台再调用微信的API，通过此登录凭证code，换取唯一用户标识openid等信息。

(1) 小程序端请求临时登录凭证code：

```javascript
wx.login({
    success: response => {
        const code = response.code
        if (code) {
            // Send "code" to our backend
            ...
        }
    }
})
```

(2) 换取唯一用户标识openid：

```java
public Map<String, Object> requestLoginSession(String code) throws RestClientException {

    UriComponentsBuilder builder = UriComponentsBuilder
            .fromUriString(loginSessionUrl)
            .queryParam(APP_ID_KEY, appId)
            .queryParam(APP_SECRET_KEY, appSecret)
            .queryParam(LOGIN_SESSION_CODE_KEY, code)
            .queryParam(LOGIN_SESSION_GRANT_TYPE_KEY, loginSessionGrantType);

    ResponseEntity<String> responseEntity = restTemplate.getForEntity(builder.toUriString(), String.class);
    String responseBody = responseEntity.getBody();
    JSONObject jsonObject = new JSONObject(responseBody);

    Map<String, Object> result = new HashMap<>();

    if (jsonObject.has(OPEN_ID_KEY)) {
        // Get open id from WeChat server
        String openId = jsonObject.getString(OPEN_ID_KEY);
        result.put(OPEN_ID_KEY, openId);
        return result;

    } else if (jsonObject.has(ERROR_CODE_KEY) && jsonObject.has(ERROR_MESSAGE_KEY)) {
        Integer errorCode = jsonObject.getInt(ERROR_CODE_KEY);
        String errorMessage = jsonObject.getString(ERROR_MESSAGE_KEY);
        result.put(ERROR_CODE_KEY, errorCode);
        result.put(ERROR_MESSAGE_KEY, errorMessage);
        return result;

    } else {
        throw new RestClientException("Fail to parse login session response");
    }
}
```

如果按照以上方法的话，每次微信支付都需要调用“wx.login”的API。那么有什么办法可以只调用一次“wx.login”的API呢？
为了改进，我们可以让小程序onLaunch启动的时候，调用一次“wx.login”的API。然后，我们的后台换取openid。拿到openid之后，我们可以生成一个session_id, 并将其存入Redis数据库当中。
为了安全起见，我们还可以将session_id设置一定的时效性。最后，我们再把这个session_id返回给小程序。
那么这样，每次微信支付的时候，小程序只需要通过session_id与我们后台沟通即可。

<br>

### 2. 统一下单

当我们拿到用户的openid的时候，就可以统一下单了。这一步骤的目的是为了获取prepay_id。
首先，我们要把相关的参数进行第一次签名，并以XML的形式发送给微信后台。
如果签名成功，微信后台就会发送含有prepay_id的回复。

```java
public Map<String, Object> sendWeChatPayRequest(String openId,
                                                String tradeNumber,
                                                String productName,
                                                String clientIpAddress,
                                                Double totalPrice) throws RestClientException {

    String nonceStr = RandomNonceUtil.generateRandomNonce();
    Integer totalPriceInt = ((Double) (totalPrice * 100)).intValue();

    Map<String, Object> requestParams = new HashMap<>();
    requestParams.put(APP_ID_KEY, appId);
    requestParams.put(MCH_ID_KEY, mchId);
    requestParams.put(NONCE_STR_KEY, nonceStr);
    requestParams.put(BODY_KEY, productName);
    requestParams.put(OUT_TRADE_NO_KEY, tradeNumber);
    requestParams.put(TOTAL_FEE_KEY, totalPriceInt.toString()); // Need to convert to string, otherwise pay sign will fail
    requestParams.put(SPBILL_CREATE_IP_KEY, clientIpAddress);
    requestParams.put(NOTIFY_URL_KEY, notifyUrl);
    requestParams.put(TRADE_TYPE_KEY, tradeType);
    requestParams.put(OPEN_ID_KEY, openId);

    // First sign
    String firstSign = PaySignUtil.createSign(requestParams, payKey);
    requestParams.put(PAY_SIGN_KEY, firstSign);

    Map<String, Object> orderedRequestParams = new TreeMap<>(requestParams);
    String xml = XmlUtil.convertToXml(orderedRequestParams);

    LOGGER.info("Request WeChat pay: " + xml);

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_XML);

    HttpEntity<String> entity = new HttpEntity<>(xml, headers);

    ResponseEntity<String> responseEntity = restTemplate.postForEntity(requestPayUrl, entity, String.class);
    String result = responseEntity.getBody();

    LOGGER.info("Receive WeChat pay response: " + result);

    Map<String, Object> response = XmlUtil.parseXml(result);

    // Parse XML server response
    ...
}
```

其中，需要注意的有以下几点：

(1) 支付金额的单位为“分”，而且需要转换成string，否则后面的签名会失败。

(2) 此步骤需要用XML的形式与微信后台沟通。

(3) 签名时，非空参数值的参数按照参数名ASCII码从小到大排序，使用key-value的格式（即key1=value1&key2=value2…）拼接成字符串“stringA”。在“stringA”最后拼接上“key”得到“stringSignTemp”字符串，并对“stringSignTemp”进行MD5运算，再将得到的字符串所有字符转换为大写，得到sign值signValue。

在这里，我用一个“PaySignUtil”的class实现以上签名的步骤：
```java
// Public methods

public static String createSign(Map<String, Object> params, String key) {
    // Create WeChat pay sign
    String linkString = createLinkString(params);
    linkString = linkString + "&" + KEY + "=" + key;
    return DigestUtils.md5DigestAsHex(getContentBytes(linkString, "utf-8")).toUpperCase();
}

// Private methods

private static String createLinkString(Map<String, Object> params) {
    // Create link string key-value pair
    List<String> keys = new ArrayList<>(params.keySet());
    Collections.sort(keys);

    String linkString = "";
    for (int i = 0; i < keys.size(); i++) {
        String key = keys.get(i);
        String value = params.get(key).toString();
        if (i == keys.size() - 1) {
            // Last value shall not contain "&"
            linkString = linkString + key + "=" + value;
        } else {
            linkString = linkString + key + "=" + value + "&";
        }
    }

    return linkString;
}

private static byte[] getContentBytes(String content, String charset) {
    if (charset == null || "".equals(charset)) {
        return content.getBytes();
    }
    try {
        return content.getBytes(charset);
    } catch (UnsupportedEncodingException e) {
        throw new RuntimeException("Incorrect charset: " + charset);
    }
}
```

<br>

### 3. 再次签名

当我们拿到微信后台发过来的，包含prepay_id的回复的时候，我们需要再次签名。并且将签名后的结果，发送给小程序。

```java
// Parse XML server response 
...

String prepayId = response.get("prepay_id").toString();
String responseNonceStr = response.get("nonce_str").toString();

long timeStamp = System.currentTimeMillis() / 1000;
String timeStampStr = Long.toString(timeStamp);

Map<String, Object> responseParams = new HashMap<>();
responseParams.put(APP_ID_KEY, appId);
responseParams.put(NONCE_STR_KEY, responseNonceStr);
responseParams.put(PACKAGE_KEY, PREPAY_ID_KEY + "=" + prepayId);
responseParams.put(SIGN_TYPE_KEY, SIGN_TYPE_MD5);
responseParams.put(TIME_STAMP_KEY, timeStampStr);

// Second sign
String secondSign = PaySignUtil.createSign(responseParams, payKey);

// Return the result to client
...

```

小程序收到签名结果后，再将相关信息发送给微信后台：

```javascript
wx.requestPayment({
    timeStamp: this.data.timeStampStr,
    nonceStr: this.data.nonceStr,
    package: this.data.package,
    signType: this.data.signType,
    paySign: this.data.sign,
    success: function(response) {
        // Request payment success
    },
    fail: function (response) {
        // Request payment fail
    }
})
```

<br>

### 4. 返回支付结果

如果我们成功完成以上几个步骤，在开发的过程中，就能看到这样一个二维码，让我们扫描并进行微信支付了。

![微信支付二维码](/assets/img/2019-02-18-wechat-pay/wechat-pay.png)

当用户支付成功以后，微信后台会给我们返回支付结果。当我们验证签名是否正确之后，可将结果发送回微信后台。

```java
public Map<String, Object> replyNotifyWeChatPay(Map<String, Object> response) {

    LOGGER.info("Receive notify WeChat pay response: " + response);

    Map<String, Object> result = new HashMap<>();
    if (response.get(RESULT_CODE_KEY) != null &&
            response.get(RESULT_CODE_KEY).toString().equals(RESULT_CODE_SUCCESS)) {

        // Verify pay sign
        boolean isValid = PaySignUtil.verify(response, response.get(PAY_SIGN_KEY).toString(), payKey);
        if (isValid) {
            result.put(RESULT_CODE_KEY, RESULT_CODE_SUCCESS);
            result.put(RETURN_MSG_KEY, RESULT_CODE_OK);
            return result;
        }
    }

    result.put(RESULT_CODE_KEY, RESULT_CODE_FAIL);
    result.put(RETURN_MSG_KEY, "Fail to verify pay sign");
    return result;
}
```

<br>

## 结论

本文将主要向大家介绍实现微信支付功能的具体步骤。演示项目请移步 [Github Project](https://github.com/mikemikezhu/wechat-pay-spring-demo)。

<br>

## 参考资料

(1) [微信支付开发文档](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=7_1)
<br>
(2) [微信支付安全规范](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=4_3)
<br>
(3) [微信支付时提示商户号mch_id与appid不匹配](https://www.kancloud.cn/itlian/help/604746)
<br>