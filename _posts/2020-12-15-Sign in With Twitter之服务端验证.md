---
layout: post
title: 'Sign in With Twitter之服务端验证'
date: 2020-12-15
author: pyihe
tags: 第三方登录
---

### 前言

本文是关于如何集成Twitter第三方登录。与Google等不同的是，Twitter的第三方登录采用的是OAuth 1.0开放授权标准，所以在向授权服务器发送请求的时候需要注意了。

### OAuth 1.0

关于OAuth的详细介绍，可以参考[维基百科](https://zh.wikipedia.org/zh-hans/%E5%BC%80%E6%94%BE%E6%8E%88%E6%9D%83) ，下面介绍在Twitter的第三方登录中，如何发起一个授权请求。

根据[Twitter 官方文档](https://developer.twitter.com/en/docs/authentication/oauth-1-0a/authorizing-a-request) 显示，需要在http请求的Header中包含足够识别请求者身份信息的内容，这些内容放在`Authorization`字段中。

`Authorization`中需要包含的参数有: 

|参数名|值|
|:---|:---|
|oauth_consumer_key|Twitter client_id|
|oauth_nonce|每次请求的唯一凭证，Twitter将根据此值来判断是否是已经被提交了多次，需要Base64编码，长度为32字节，不能有特殊字符|
|oauth_signature|签名，除了`oauth_signature`字段本身的其他字段的签名|
|oauth_signature_method|签名方法，固定值: HMAC-SHA1|
|oauth_timestamp|请求时间戳，精确到秒|
|oauth_token|access_token|
|oauth_version|固定值: 1.0|

构建`Authorization`的值，假设代表最终值的变量名为`DST`字符串(初始值为空)

1. 追加字符串"OAuth "(包括一个空格)到`DST`
2. [百分号编码](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81) 参数名，并将其追加到`DST`
3. 将字符`=`追加到`DST`
4. 追加`"`到`DST`
5. [百分号编码](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81) 参数值，并将其追加到`DST`
6. 追加`"`到`DST`
7. 如果还有剩余到参数键值对，则追加`, `(包含一个空格)到`DST`

最后得到的`DST`应该是下面这种格式: 

```
OAuth oauth_consumer_key="xvz1evFS4wEEPTGEFPHBog", oauth_nonce="kYjzVBB8Y0ZFabxSWbWovY3uYSQ2pTgmZeNu2VS4cg", oauth_signature="tnnArxj06cWHq44gCs1OSKk%2FjLY%3D", oauth_signature_method="HMAC-SHA1", oauth_timestamp="1318622958", oauth_token="370773112-GmHxMAgYyLbNEtIKZeRNFsMKPR9EyMZeS9weJAEb", oauth_version="1.0"
```

### 关于签名

上面提到`Authorization`中需要包含签名，Twitter采用的签名类型是: `HMAC-SHA1`， 与微信支付类似，除了签名字段本身外，其他字段全部需要参与到签名中去，不同的是Twitter的签名还需要包含HTTP请求方法，以及请求的URL。

所以，最终Twitter待签名的字符串由三部分组成: HTTP请求方法(POST/GET等，大写)，请求的API URL(如：https://api.twitter.com/1.1/statuses/update.json，URL需要URL编码)，以及由参数组成的键值对。

参数签名步骤如下: 

1. [百分号编码](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81) 每个参与到签名的键值对，key与value都需要编码
2. 将编码后的键值对根据key的字母顺序排序
3. 将编码后的key追加到目标签名字符串中
4. 每个key后面追加一个`=`字符
5. 将每个key对应的编码后的value追加到`=`后面
6. 如果还有剩余的key/value，则使用字符`&`连接每个键值对

最后得到的参数键值对字符串的格式如下: 

```
include_entities=true&oauth_consumer_key=xvz1evFS4wEEPTGEFPHBog&oauth_nonce=kYjzVBB8Y0ZFabxSWbWovY3uYSQ2pTgmZeNu2VS4cg&oauth_signature_method=HMAC-SHA1&oauth_timestamp=1318622958&oauth_token=370773112-GmHxMAgYyLbNEtIKZeRNFsMKPR9EyMZeS9weJAEb&oauth_version=1.0&status=Hello%20Ladies%20%2B%20Gentlemen%2C%20a%20signed%20OAuth%20request%21
```

所以最后的待签名字符串格式应该为: 

```
POST&https%3A%2F%2Fapi.twitter.com%2F1.1%2Fstatuses%2Fupdate.json&include_entities%3Dtrue%26oauth_consumer_key%3Dxvz1evFS4wEEPTGEFPHBog%26oauth_nonce%3DkYjzVBB8Y0ZFabxSWbWovY3uYSQ2pTgmZeNu2VS4cg%26oauth_signature_method%3DHMAC-SHA1%26oauth_timestamp%3D1318622958%26oauth_token%3D370773112-GmHxMAgYyLbNEtIKZeRNFsMKPR9EyMZeS9weJAEb%26oauth_version%3D1.0%26status%3DHello%2520Ladies%2520%252B%2520Gentlemen%252C%2520a%2520signed%2520OAuth%2520request%2521
```

接下来是获取签名用的key，key由两部分组成: `consumer secret`和`OAuth token secret`，其中`consumer secret`在Twitter后台获取，`OAuth token secret`的获取由几种方式，将在后面介绍。

假设`consumer secret`的值为: `kAcSOqF21Fu85e7zjz7ZN2U4ZRhfV3WpwPAoE3Z7kBw`, `OAuth token secret`的值为: `LswwdoUaIvS8ltyTt5jkRh4J50vUPVVHtR2YPi5kE`

分别将二者进行[百分号编码](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81) ，然后使用`&`将二者连接在一起，即组成签名密钥: `kAcSOqF21Fu85e7zjz7ZN2U4ZRhfV3WpwPAoE3Z7kBw&LswwdoUaIvS8ltyTt5jkRh4J50vUPVVHtR2YPi5kE`

这里需要注意的一点是, 在某些尚未获取到`OAuth token secret`的API中，签名密钥的格式为: `kAcSOqF21Fu85e7zjz7ZN2U4ZRhfV3WpwPAoE3Z7kBw&`,即没有`OAuth token secret`

最后就是用签名密钥进行签名:

```go
key := "kAcSOqF21Fu85e7zjz7ZN2U4ZRhfV3WpwPAoE3Z7kBw&LswwdoUaIvS8ltyTt5jkRh4J50vUPVVHtR2YPi5kE"
toBeSignStr := "POST&https%3A%2F%2Fapi.twitter.com%2F1.1%2Fstatuses%2Fupdate.json&include_entities%3Dtrue%26oauth_consumer_key%3Dxvz1evFS4wEEPTGEFPHBog%26oauth_nonce%3DkYjzVBB8Y0ZFabxSWbWovY3uYSQ2pTgmZeNu2VS4cg%26oauth_signature_method%3DHMAC-SHA1%26oauth_timestamp%3D1318622958%26oauth_token%3D370773112-GmHxMAgYyLbNEtIKZeRNFsMKPR9EyMZeS9weJAEb%26oauth_version%3D1.0%26status%3DHello%2520Ladies%2520%252B%2520Gentlemen%252C%2520a%2520signed%2520OAuth%2520request%2521"

m := hmac.New(sha1.New, []byte(key))
m.Write([]byte(toBeSignStr))
m.Sum(nil)
sign := base64.StdEncoding.EncodeToString(m.Sum(nil))
```

### 获取OAuth token secret

Twitter的每个API请求都需要进行签名，在开始之前你可以在[Twitter APP详情页面](https://developer.twitter.com/en/docs/basics/apps/overview) 对下面这些值进行配置: 

`oauth_consumer_key`, `oauth_consumer_secret`: 当发起API请求时，可以将这两个值视为Twitter开发者应用的用户名和密码

`oauth_token`, `oauth_token_secret`: 在OAuth 1.0中，这两个值是用于发起API请求的特定用户的身份凭证，它们标识了API请求是由哪个Twitter账号发起的。如果你愿意使用相同的`oauth_token`, `oauth_token_secret`，则可以在[Twitter APP详情页面](https://developer.twitter.com/en/docs/basics/apps/overview) 进行配置，那么每个用户请求API时都将使用同一对`oauth_token`和`oauth_token_secret`。当然你可以让不同的用户使用不同的`oauth_token`和`oauth_token_secret`。如下: 

```
Request URL: POST https://api.twitter.com/oauth/request_token

Request POST Body: N/A

Authorization Header: OAuth oauth_nonce="K7ny27JTpKVsTgdyLdDfmQQWVLERj2zAK5BslRsqyw", oauth_callback="http%3A%2F%2Fmyapp.com%3A3005%2Ftwitter%2Fprocess_callback", oauth_signature_method="HMAC-SHA1", oauth_timestamp="1300228849", oauth_consumer_key="OqEqJeafRSF11jBMStrZz", oauth_signature="Pc%2BMLdv028fxCErFyi8KXFM%2BddU%3D", oauth_version="1.0"

Response: oauth_token=Z6eEdO8MOmk394WozF5oKyuAv855l4Mlqo7hhlSLik&oauth_token_secret=Kd75W4OQfb2oJTV0vzGzeXftVAwgMnEK9MumzYcM&oauth_callback_confirmed=true
```

### Sign in With Twitter接入步骤

1. 获取`oauth_token`, `oauth_token_secret`（客户端）

由客户端向Twitter服务器发起获取token的Request请求, 根据[OAuth 1.0a](https://developer.twitter.com/en/docs/authentication/oauth-1-0a/authorizing-a-request), Http Header中的`Authorization`需要添加的字段有: 

|参数名|值|
|:---|:---|
|oauth_callback|用户点击登录按钮后，客户端需要用户重定向到的URI，URI需要URL编码，该值需要在[Twitter 后台](https://developer.twitter.com/en/docs/developer-portal/overview) 注册|
|oauth_consumer_key|Twitter client_id|
|oauth_nonce|每次请求的唯一凭证，Twitter将根据此值来判断是否是已经被提交了多次，需要Base64编码，长度为32字节，不能有特殊字符|
|oauth_signature|签名，除了`oauth_signature`字段本身的其他字段的签名|
|oauth_signature_method|签名方法，固定值: HMAC-SHA1|
|oauth_timestamp|请求时间戳，精确到秒|
|oauth_version|固定值: 1.0|
 
2. 用户重定向（客户端）

利用步骤1获取到的`oauth_token`调用API: `https://api.twitter.com/oauth/authenticate`: `https://api.twitter.com/oauth/authenticate?oauth_token={oauth_token}` ,用户将会重定向到指定页面并且做出相应选择:

|登录并授权|如果用户登录twitter.com并且已经批准了调用应用程序，则将立即对它们进行身份验证，并使用有效的OAuth请求令牌返回回调URL。重定向到twitter.com对用户来说不明显|
|登录但不授权|如果用户登录到twitter.com但尚未批准调用应用程序，将显示与调用应用程序共享访问权限的请求。接受授权请求后，用户将被重定向到带有有效OAuth请求令牌的回调URL|
|不登录|如果用户未登录twitter.com将提示他们在屏幕上输入相同的访问权限。登录后，用户将使用有效的OAuth请求令牌返回回叫URL|

一旦成功但授权，你的`callback_url`将会收到包含`oauth_token`和`oauth_verifier`参数，你的应用应该校验该`oauth_token`是否和步骤1中获取到的`oauth_token`匹配

3. 获取用于身份验证的`oauth_token`, `oauth_token_secret`（客户端）

为了转换token为可用的`access_token`口令，你需要调用`POST-oauth/access_token`API，包含`oauth_verifier`和`oauth_token`参数，并且需要签名。一个成功的调用，API将会返回`oauth_token`和`oauth_token_secret`,客户端将这两个值传给服务端，服务端用以验证用户信息。

同时，获取`access_token`的API为: `POST-https://api.twitter.com/oauth/access_token`, 比如: `POST https://api.twitter.com/oauth/access_token?oauth_token=qLBVyoAAAAAAx72QAAATZxQWU6P&oauth_verifier=ghLM8lYmAxDbaqL912RZSRjCCEXKDIzx`, `access_token`的获取最好放在服务端。

4. 身份验证（服务端）

服务端调用API: `GET-account/verify_credentials`来验证此次登录的用户信息是否有效。

### 参考资料

[Twitter 官方文档](https://developer.twitter.com/en/docs/authentication/oauth-1-0a/authorizing-a-request)

[Twitter APP详情页面](https://developer.twitter.com/en/docs/basics/apps/overview)

[Twitter 后台](https://developer.twitter.com/en/docs/developer-portal/overview)

[百分号编码](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81)

[OAuth 1.0a](https://developer.twitter.com/en/docs/authentication/oauth-1-0a/authorizing-a-request)