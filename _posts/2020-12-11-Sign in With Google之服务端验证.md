---
layout: post
title: 'Sign in With Google之服务端验证'
date: 2020-12-11
author: pyihe
tags: 第三方登录
---

### 前言

关于使用Google第三方登录服务端如何进行登录验证。Google登录流程中，服务器的主要工作为验证用户信息以确保此次登录为有效登录，然后让当前登录的用户进行应用服务器的相应流程。另一方面，用户在授权登录后，应用服务器可以通过用户的授权获取到access_token，以便后续访问Google的其他API。

### 登录流程

![](/assets/img/2020-12-11/sign-in-with-google-flow.jpg)

根据上面的图所示：

1. 在用户点击"Sign in With Google"的按钮后，客户端将会收到一个来自OAuth服务器并且只能使用一次的code，客户端将该code发送至应用服务器，这里由客户端处理。

2. 应用服务器用该code向Google API服务器请求access_token和refresh_token，这里需要注意:
    
    1. 每个code只能使用一次
    2. 只要code有效，那么每次请求Google都将返回access_token
    3. 在获取到refresh_token之后，需要将其保存起来用于后续刷新access_token使用(access_token存在过期时间)，因为在后续的refresh_token交换请求将返回空

    请求access_token和refresh_token的API为`https://oauth2.googleapis.com/token`

    请求参数有: 

    |参数名|解释|    
    |:-----|:-----|
    |code|用户授权后客户端获取到的code|
    |client_id|应用ID|
    |client_secret|应用密钥|
    |redirect_uri|在 [API控制台页面](https://console.developers.google.com/apis/credentials) 配置的授权重定向URI|
    |grant_type|固定值：authorization_code|
    
    Google返回的参数有: 
    
    |参数名|解释|
    |:----|:-----|
    |access_token|用于访问Google API的token|
    |expires_in|access_token有效期, 单位: 秒|
    |id_token|Google数字签名了的用户身份信息|
    |scope|access_token授予的访问范围，以空格分隔的、区分大小写的字符串列表表示。|
    |token_type|token类型，这里总是返回`Bearer`|
    |refresh_token|用于刷新access_token的token|
    
    下面以golang为例, 发送请求: 

    ```go
    var (
        clientId     = "your clientId"
        clientSecret = "your clientSecret"
        code         = "your code"
        redirectUri  = "your redirectUri"
        url          = "https://oauth2.googleapis.com/token"
    )
    
        reader := strings.NewReader(fmt.Sprintf("client_id=%s&client_secret=%s&redirect_uri=%s&grant_type=authorization_code&code=%s", clientId, clientSecret, redirectUri, code))
        response, err := http.Post(url, "application/x-www-form-urlencoded", reader)
        handleResponse(response)
        handleErr(err)
    ```


    刷新access_token的API为: `https://oauth2.googleapis.com/token`

    请求参数有:
    
    |参数名|解释|    
    |:-----|:-----|
    |client_id|应用ID|
    |client_secret|应用密钥|
    |refresh_token|access_token值|
    |grant_type|固定值：refresh_token|
    
    返回的参数有: 
    
    |参数名|解释|
    |:----|:-----|
    |access_token|用于访问Google API的token|
    |expires_in|access_token有效期|
    |token_type|token类型，这里总是返回`Bearer`|
    |scope|access_token授予的访问范围，以空格分隔的、区分大小写的字符串列表表示。|
 
3. 服务器获取到access_token和refresh_token之后便可以开始验证用户信息(idToken)是否有效。

### 如何验证用户身份信息

用户的身份信息是包含在`idToken`中的，`idToken`为 [JWT](https://tools.ietf.org/html/rfc7519) 格式。按照 [Google官网](https://developers.google.com/identity/sign-in/web/backend-auth) 介绍，`idToken`需要验证的内容有: 

1. 使用Google公钥确认`idToken`被正确的签名
2. `aud`的值应该与你应用的`client_id`相同
3. `iss`的值为`accounts.google.com`或者`https://accounts.google.com`
4. 有效期尚未过

官网建议的验证方式有两种:  

1. 使用Google的验证库，包括[Java](https://developers.google.com/api-client-library/java/google-api-java-client/setup), [Node.js](https://github.com/google/google-api-nodejs-client), [PHP](https://github.com/googleapis/google-api-php-client), [Python](https://google-auth.readthedocs.io/), 当然， Go语言的库为: [Golang](https://github.com/googleapis/google-api-go-client), 如果使用Go语言的库，需要注意如果你的项目中引入了etcd，那么你需要注意关于etcd GRPC版本所引起的一个老生常谈的问题了，我在使用过程中解决办法是降低了[Golang](https://github.com/googleapis/google-api-go-client)的版本，对应go.mod中为: 

```
replace (
    google.golang.org/grpc => google.golang.org/grpc v1.26.0
    google.golang.org/api => google.golang.org/api v0.14.0
)
```

关于验证`idToken`的代码为:

```go
import (
    "http"
    "github.com/dghubble/oauth1"
    "google.golang.org/api/option"
    "google.golang.org/api/oauth2/v2"
)
    
    oatuService, err := oauth2.NewService(context.Background(), option.WithHTTPClient(http.DefaultClient))
    if err != nil {
        log4go.Errorf("%v\n", err)
        return nil, err
    }
    tokenInfoCall := oatuService.Tokeninfo()
    tokenInfoCall.IdToken(googleToken)
    tokenInfo, err := tokenInfoCall.Do()
    //根据tokenInfo中的字段进行验证
``` 

**注意：Google建议生产环境使用此方法进行验证**

2. 调用Google的API进行验证, 验证API为: https://oauth2.googleapis.com/tokeninfo?id_token={idToken}, 如:

`https://oauth2.googleapis.com/tokeninfo?id_token=XYZ123`

请求方法既可以是`POST`也可以是`GET`, 需要注意的是, 在收到API回复后只有当StatusCode为`http.StatusOK`时才能进行下一步校验。

idToken校验API返回的结果为JSON键值对，如：
```json
{
 // 这6个字段是所有idToken都包含的
 "iss": "https://accounts.google.com",  //token签发者，值为https://accounts.google.com或者accounts.google.com
 "sub": "110169484474386276334", //用户在该Google应用中的唯一标识，类似于微信的OpenID
 "azp": "1008719970978-hb24n2dstb40o45d4feuo2ukqmcc6381.apps.googleusercontent.com", //具体我也不知道，猜测与aud相同，都是应用的client_id
 "aud": "1008719970978-hb24n2dstb40o45d4feuo2ukqmcc6381.apps.googleusercontent.com", //client_id
 "iat": "1433978353", //签发时间
 "exp": "1433981953", //过期时间

 // 下面的字段只有当用户授权了"profile"和"email"权限后才会出现
 "email": "testuser@gmail.com", //用户邮箱
 "email_verified": "true", //邮箱是否已验证
 "name" : "Test User", //用户名
 "picture": "https://lh4.googleusercontent.com/-kYgzyAWpZzJ/ABCDEFGHI/AAAJKLMNOP/tIXL9Ir44LE/s99-c/photo.jpg", //用户头像
 "given_name": "Test", //名
 "family_name": "User", //姓
 "locale": "en"  //所属地区
}
```

**注意: 因为在授权过程中和获取access_token的过程中都会获取到idToken，所以服务器验证的idToken既可以由客户端传给服务器，也可以是服务器自己获取。但是在登录过程中，客户端获取到的授权code必须传给服务器**


### 参考资料

[Google Sign-In for server-side apps](https://developers.google.com/identity/sign-in/web/server-side-flow)

[Authenticate with a backend server](https://developers.google.com/identity/sign-in/web/backend-auth)

[Using OAuth 2.0 to Access Google APIs](https://developers.google.com/identity/protocols/oauth2)
