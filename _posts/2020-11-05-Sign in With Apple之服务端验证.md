---
layout: post
title: 'Sign in With Apple之服务端验证'
date: 2020-11-05
author: pyihe
tags: [第三方登录]
---

### 介绍

2019年之后，对于Apple App来说，如果要支持第三方登录，则必须同时支持苹果的第三方登录，即[Sign in With Apple](https://developer.apple.com/sign-in-with-apple/), 本文主要介绍如何使用Go语言实现Sign in With Apple时服务端的验证, 即[Generate and Validate Tokens](https://developer.apple.com/documentation/sign_in_with_apple/generate_and_validate_tokens)。或者不支持第三方登录, 直接使用电话号码或者账号密码的方式进行注册以及登录。

### 登录流程

![](/assets/img/2020-11-05/apple_work_flow.png)

流程大概可以描述为: 

1. app请求通过Apple进行第三方登录，此时，客户端将会获得包括用户唯一凭证UserID(与微信的OpenId类似), 用户全名Full Name, 验证用的Code(IdentityCode)以及验证用的Token(IdentityToken)。

2. 客户端将获得的数据发送给服务器，由服务器通过IdentityCode或者IdentityToken来验证此次登录是否有效。

3. 如果验证通过, 服务端处理完自己内部的登录流程后, 将对应的登录结果(状态)返回给客户端。

在第二步服务器的验证过程中，服务器只需要选择Code或者Token中的任意一种进行验证即可: 

1. IdentityToken: 根据Apple官方文档, Token验证方式为[JSON WEB Token(JWT)](https://developer.apple.com/documentation/sign_in_with_apple/generate_and_validate_tokens), 按照对应的方式进行验证即可。
2. IdentityCode: 根据Apple官方文档, 通过Code验证需要Apple开发者对该App进行配置的额外`client_id`, `client_secret`以及`redirect_uri`三个参数。

#### IdentityToken验证

此种验证方法为传统的JWT验证, Token由Header, Payload以及Signature三部分组成, 通过JSON序列化每一部分，然后使用Base64URL编码后通过`.`拼接起来的字符串。 

1. Header: 包括的字段如下, 

    * kid: 表示用于验证签名的Apple公钥
    * alg: 表示用于签名的算法

2. Payload: 包括的字段有如下, 
    
    * iss(string): 表示Token签发机构, 值固定为: https://appleid.apple.com
    * aud(string): 表示Apple App的ID
    * exp(int64): 表示Token的过期时间, 时间戳
    * iat(int64): 表示client_secret生成时间，时间戳
    * sub(string): 表示用户唯一标识
    * c_hash(string): 文档中没看到这个字段, 作用未知
    * auth_time(int64): 表示签名生成时间
    * email(string): 表示用户邮箱, 可能是真实的也可能Apple处理过的密文邮件地址，取决于用户登录时是否选择了隐藏邮箱
    * email_verified(bool): 表示用户邮箱是否已验证, 由于Apple总是返回已验证了的邮箱, 所以这个字段的值总是为`true`, 但是需要注意的是, Apple返回的`true`, 可能是字符串也可能是bool类型, 需要自己处理一下。
    * nonce(string): 只有当发起登录请求的时候传递了此参数, 在验证的时候才会返回，目的是为了降低被攻击的可能性
    * nonce_supported(bool): 表示是否支持nonce, 如果为true, 则需要判断nonce字段值是否正确
    * is_private_email(bool): 表示用户提供的邮箱地址是否是Apple处理了的代理邮箱地址
    * real_user_status(int): 表示用户是否是真实用户: 0(Unsupported: 表示当前系统版本不支持该字段的值, 只有在IOS 14及以上版本, macOS 11及以上版本, watchOS 7及以上版本才支持), 1(Unknown: 系统无法识别是否是真实用户), 2(LikelyReal: 几乎可以确定为真实用户)

3. Signature: 表示签名字段，用Base64URL对Header和Payload分别编码，然后用`.`拼接, 最后使用RSA以及SHA256进行签名得到的结果

一个Header和Payload的例子为: 

```
{
    "alg": "RS256",
    "kid": "ABC123DEFG"
}
{
    "iss": "DEF123GHIJ",
    "iat": 1437179036,
    "exp": 1493298100,
    "aud": "https://appleid.apple.com",
    "sub": "com.mytest.app"
}
```

一个IdentityToken例子如下:

```
eyJraWQiOiJBSURPUEsxIiwiYWxnIjoiUlMyNTYifQ.eyJpc3MiOiJodHRwczovL2FwcGxlaWQuYXBwbGUuY29tIiwiYXVkIjoiY29tLmZ1bi5BcHBsZUxvZ2luIiwiZXhwIjoxNTY4NzIxNzY5LCJpYXQiOjE1Njg3MjExNjksInN1YiI6IjAwMDU4MC4wODdjNTU0ZGNlMzU0NjZmYTg1YzVhNWQ1OTRkNTI4YS4wODAxIiwiY19oYXNoIjoiel9KY0RscFczQjJwN3ExR0Nna1JaUSIsImF1dGhfdGltZSI6MTU2ODcyMTE2OX0.WmSa4LzOzYsdwTqAJ_8mub4Ls3eyFkxZoGLoy-U7DatsTd_JEwAs3_OtV4ucmj6ENT3153iCpYY6vBxSQromOMcXsN74IrUQew24y_zflN2g4yU8ZVvBCbTrR_6p9f2fbeWjZiyNcbPCha0dv45E3vBjyHhmffWnk3vyndBBiwwuqod4pyCZ3UECf6Vu-o7dygKFpMHPS1ma60fEswY5d-_TJAFk1HaiOfFo0XbL6kwqAGvx8HnraIxyd0n8SbBVxV_KDxf15hdotUizJDW7N2XMdOGQpNFJim9SrEeBhn9741LWqkWCgkobcvYBZsrvnUW6jZ87SLi15rvIpq8_fw
``` 

根据上面可以得出验证IdentityToken的步骤为: 

1. 以`.`为分隔点, 将IdentityToken分隔为三部分, 第三部分为签名, 留着用于验证

2. 使用Base64URL解码对应的Header和Payload, 并JSON反序列化为对应的结构体(或者键值对), 并且对Payload中相应对值进行验证，如exp, sub, iat, aud

3. 通过[接口](https://developer.apple.com/documentation/sign_in_with_apple/fetch_apple_s_public_key_for_verifying_token_signature)从Apple Server获取RSA公钥，接口地址[https://appleid.apple.com/auth/keys](https://appleid.apple.com/auth/keys), 这里需要注意, 获取到的结果通常为两个，需要用选择与Header中的`kid`值匹配的那个Key

4. 步骤3返回的Key中包含了RSA公钥中的`N`和`E`的值，同样是用Base64URL编码后的值, 需要解码, 然后再构造RSA公钥

5. 得到公钥后，将步骤1中得到的Base64URL编码的Header和Payload再次拼接起来，然后调用rsa.VerifyPKCS1v15()方法进行签名验证, 注意这里的Hash类型为`SHA256`

验证代码如下: 

```go
func (v *Validator) CheckIdentityToken(token string) (JWTToken, error) {
    if token == "" {
        return nil, ErrInvalidIdentityToken
    }
    appleToken, err := parseToken(token)
    if err != nil {
        return nil, err
    }
    key, err := fetchKeysFromApple(appleToken.header.Kid)
    if err != nil {
        return nil, err
    }
    if key == nil {
        return nil, ErrFetchKeysFail
    }
    
    pubKey, err := generatePubKey(key.N, key.E)
    if err != nil {
        return nil, err
    }
    
    //利用获取到的公钥解密token中的签名数据
    sig, err := decodeSegment(appleToken.sign)
    if err != nil {
        return nil, err
    }
    
    //苹果使用的是SHA256
    var h hash.Hash
    switch appleToken.header.Alg {
    case "RS256":
        h = crypto.SHA256.New()
    case "RS384":
        h = crypto.SHA384.New()
    case "RS512":
        h = crypto.SHA512.New()
    }
    if h == nil {
        return nil, ErrInvalidHashType
    }
    
    h.Write([]byte(appleToken.headerStr + "." + appleToken.claimsStr))
    
    return appleToken, rsa.VerifyPKCS1v15(pubKey, crypto.SHA256, h.Sum(nil), sig)
}
```

#### IdentityCode验证

按照官方文档, IdentityCode的验证相对来说限制要高一点，没有那么通用, 因为验证过程中需要用到`client_id`, `client_secret`, `redirect_uri`三个参数, 由于每个Apple App这三个参数都不相同, 所以没有IdentityToken那么通用。

根据官方文档, IdentityCode的验证需要调用[接口](https://developer.apple.com/documentation/sign_in_with_apple/generate_and_validate_tokens)向Apple Server验证, 接口地址为: [https://appleid.apple.com/auth/token](https://appleid.apple.com/auth/token)

文档中已经说得很明白, 具体代码如下: 

```go
func (v *Validator) CheckIdentityCode(code string) (*TokenResponse, error) {
    if code == "" {
        return nil, ErrInvalidIdentityCode
    }
    if v.clientID == "" {
        return nil, ErrInvalidClientID
    }
    if v.clientSecret == "" {
        return nil, ErrInvalidClientSecret
    }
    //验证IdentityCode时需要填写redirect_uri参数，且redirect_uri参数必须是https协议
    if uri := strings.ToLower(v.redirectUri); strings.HasPrefix(uri, "https://") {
        return nil, ErrInvalidRedirectURI
    }
    
    param := fmt.Sprintf("client_id=%s&client_secret=%s&code=%s&grant_type=authorization_code&redirect_uri=%s", v.clientID, v.clientSecret, code, v.redirectUri)
    rder := strings.NewReader(param)
    response, err := http.Post("https://appleid.apple.com/auth/token", "application/x-www-form-urlencoded", rder)
    if err != nil {
        return nil, err
    }
    defer response.Body.Close()
    
    if response.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("checking identityCode from apple server fail: %d", response.StatusCode)
    }
    
    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        return nil, err
    }
    
    var tkResult *TokenResponse
    if err = json.Unmarshal(data, &tkResult); err != nil {
        return nil, err
    }
    return tkResult, nil
}
```

详细代码请前往[Github](https://github.com/pyihe/apple_validator)

### 参考资料

[Sign in With Apple](https://developer.apple.com/sign-in-with-apple/)

[jwt-go](https://github.com/dgrijalva/jwt-go)


