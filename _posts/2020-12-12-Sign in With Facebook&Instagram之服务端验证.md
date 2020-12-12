---
layout: post
title: 'Sign in With Facebook&Instagram之服务端验证'
date: 2020-12-12
author: pyihe
tags: 第三方登录
---

### 前言

继关于服务端如何接入[Apple](https://www.jianshu.com/p/8646f599c627), [Google](https://www.jianshu.com/p/995ca7739fb2) 的第三方登录之后，这里介绍在整合[Facebook](https://www.facebook.com/) 和[Instagram](https://www.instagram.com/) 的第三方登录时服务端的验证工作。

### Facebook

Facebook的登录验证与Google相似，同样是验证客户端通过登录请求获取到的登录token是否有效。

根据[官方文档](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow) 显示，Facebook的身份验证有两种方式: 

1. 通过code交换`access_token`，然后用获取的`access_token`访问API验证客户端传来的登录token是否有效。服务端收到客户端传来的code后，通过Facebook的API( `https://graph.facebook.com/v9.0/oauth/access_token` )交换访问口令，即access_token。最后调用API( `https://graph.facebook.com/debug_token?input_token={token-to-inspect}&access_token={app-token-or-admin-token}` )验证登录token是否有效。

    ```go
        var (
            clientId     = "your clientId"
            clientSecret = "your clientSecret"
            code         = "your code"
            redirectUri  = "your redirectUri"
            url          = fmt.Sprintf("https://graph.facebook.com/v9.0/oauth/access_token?client_id=%s&redirect_uri=%s&client_secret=%s&code=%s", clientId, redirectUri, clientSecret, code)
        )
        
            response, err := http.Get(url)
            handleResponse(response)
            handleErr(err)
    ```
    
    API返回的结果: 
    ```json
    {
      "access_token": {访问口令}, 
      "token_type": {type},
      "expires_in":  {还有多久过期, 单位: 秒}
    }
    ```
    
    ```go
        var (
            accessToken = "get from above"
            token = "your token"
        )
        
            response, err := http.Get(fmt.Sprintf("https://graph.facebook.com/debug_token?input_token=%s&access_token=%s", token, accessToken))
            handleResponse(response)
            handleErr(err)
    ```

2. token API

    客户端授权并取得Facebook返回的登录token后，将token传给服务端，服务端通过API( `https://graph.facebook.com/debug_token?input_token={token-to-inspect}&access_token={app-token-or-admin-token}` )
    
    其中`input_token`为客户端传过来的登录token，`access_token`为API访问口令，注意这里的`access_token`不是通过API获取的，而是`client_id|client_secret`
    
    ```go
    var (
            clientId     = "your clientId"
            clientSecret = "your clientSecret"
            token        = "your token"
            url          = fmt.Sprintf("https://graph.facebook.com/debug_token?input_token=%s&access_token=%s|%s", token, clientId, clientSecret)
    )
    
        response, err := http.Get(url)
        handleResponse(response)
        handleErr(err)
    ```
    
    具体是code还是token API，取决于客户端在用户点击登录按钮时，客户端发起请求中`response_type`的值。根据官网显示，`response_type`的值包括: 
    
    |值|解释|
    |:---|:---|
    |code|响应数据作为网址参数纳入，且包含 code 参数（每个登录请求独有的加密字符串）。如果未指定此参数，这便是默认行为。当服务器处理口令时，这尤其有用|
    |token|响应数据作为网址片段纳入，且包含访问口令。桌面应用必须为 response_type 选用此设置。当客户端处理口令时，这尤其有用|
    |code%20token|响应数据作为网址片段纳入，且包含访问口令和 code 参数|
    |granted_scopes|返回用户在登录时授予应用的所有权限的逗号分隔列表。可与其他 response_type 值合并。与 token 合并时，响应数据作为网址片段纳入；与其他值合并时，响应数据则作为网址参数纳入。|

token验证API返回的结构为: 

```json
{
    "data": {
        "app_id": 138483919580948,  //应用ID，等于client_id
        "type": "USER", 
        "application": "Social Cafe", 
        "expires_at": 1352419328, //token过期时间
        "is_valid": true,  //是否有效
        "issued_at": 1347235328, //什么时候签发的
        "metadata": {
            "sso": "iphone-safari"
        }, 
        "scopes": [ //权限范围
            "email", 
            "publish_actions"
        ], 
        "user_id": "1207059" //用户在该应用下的唯一ID，类似于微信的OpenID
    }
}
```

最后如果需要服务器自己获取用户的头像、昵称、性别等信息，需要服务端通过前面获取到的`access_token`自己去调用Facebook对应的API：

API: `https://graph.facebook.com/{facebook_user_id}?fields=name,picture&access_token={access_token}`

其中`fields`字段根据自己的需求填写，用`,`分隔，具体参考[官方文档](https://developers.facebook.com/docs/graph-api/reference/user)

### Instagram

根据[Instagram官方文档](https://developers.facebook.com/docs/instagram) 显示，Instagram不推荐使用Instagram作为身份验证解决方案，其推荐使用[Facebook登录](https://developers.facebook.com/docs/facebook-login), 但是为了调用Instagram的[图谱API](https://developers.facebook.com/docs/instagram-api/),需要开发者完成如下准备工作: 

1. [Instagram Business 帐户](https://help.instagram.com/502981923235522) 或 [Instagram 创作者帐户](https://help.instagram.com/1158274571010880)
2. [与该帐户相关联的 Facebook 公共主页](https://developers.facebook.com/docs/instagram-api/overview#pages)
3. 一个 Facebook 开发者帐户，可在[公共主页上执行任务](https://developers.facebook.com/docs/instagram-api/overview#tasks)
4. [已注册的 Facebook 应用](https://developers.facebook.com/docs/apps#register) ，且已配置基本设置

这里介绍如果通过[Instagram 图谱 API](https://developers.facebook.com/docs/instagram-api?locale=zh_CN) 获取Instagram的访问口令以及用户信息

客户端在App内集成Instagram授权窗口后，在用户授权后客户端将获取授权码，并将其传给服务端，然后客户端使用该授权码来换取访问口令(access_token)。

**这里需要注意的是，Instagram返回的授权码包含了后缀`#_`，所以使用时需要将`#_`去掉才是真正的授权码**

交换访问口令的API: `https://api.instagram.com/oauth/access_token`, 需要的参数有: 

|参数名|参数值|
|:----|:---|
|client_id|Instagram应用ID|
|client_secret|应用密钥|
|redirect_uri|Instagram后台配置的用户授权后的重定向URI|
|grant_type|固定值: `authorization_code`|
|code|授权码(注意去掉`#_`后缀)|

```go
    tokenParam := fmt.Sprintf("client_id=%s&client_secret=%s&grant_type=authorization_code&redirect_uri=%s&code=%s", clientId, clientSecret, redirectUri, code)
	response, err := http.Post(tokenUrl, "application/x-www-form-urlencoded", strings.NewReader(tokenParam))
	if err != nil {
		log4go.Errorf("%v\n", err)
		panic(err)
	}
    if response.StatusCode != http.StatusOK {
    	panic(err)
    }
```

返回的结果格式为: 

```json
{ 
  "access_token": "IGQVJ...", //访问口令 
  "user_id": 17841405793187218  //Instagram 应用中用户的唯一ID，类似于微信的OpenID
}
```

最后使用API: `https://graph.instagram.com/{instagram_user_id}?fields=username&access_token={access_token}` 查询用户节点，其中`fields`字段参考[用户节点文档](https://developers.facebook.com/docs/instagram-basic-display-api/reference/user#fields)

### 参考资料

[Facebook登录](https://developers.facebook.com/docs/facebook-login)

[手动构建登录流程](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow)

[Facebook User](https://developers.facebook.com/docs/graph-api/reference/user)

[Instagram官方文档](https://developers.facebook.com/docs/instagram)

[Instagram 图谱 API](https://developers.facebook.com/docs/instagram-api?locale=zh_CN)

[Instagram 用户节点文档](https://developers.facebook.com/docs/instagram-basic-display-api/reference/user#fields)