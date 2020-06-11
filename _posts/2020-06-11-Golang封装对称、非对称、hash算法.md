---
layout: post
title: 'Golang封装常规加密算法'
date: 2020-06-11
author: pyihe
tags: [Golang, 加密算法]
---

## 前言

为了使平时工作更加高效，自己使用golang对常规加密算法进行了封装，方便在以后的工作中直接使用。

[**项目地址**](https://github.com/pyihe/secret)

## 实现功能

实现的功能如下：

#### 对称加密

|Type|Mode|Padding|
|:----|:----|:----|
|DES|ECB/CBC|PKCS5/PKCS7/Zero/None|
|3DES|ECB/CBC|PKCS5/PKCS7/Zero/None|
|AES|ECB/CBC|PKCS5/PKCS7/Zero/None|
|DES|CFB/OFB/CTR/GCM||
|3DES|CFB/OFB/CTR/GCM||
|AES|CFB/OFB/CTR/GCM||

```go
package main

import (
    "fmt"
    "github.com/pyihe/secret"
)

func main() {
    c := secret.NewCipher()
    var request = &secret.SymRequest{
        PlainData:   "this field is for data to be encrypt",
        Key:         []byte("1234567812345678"),
        Type:        secret.SymTypeAES,
        ModeType:    secret.BlockModeECB,
        PaddingType: secret.PaddingTypeZeros,
    }
    cipherString, err := c.SymEncryptToString(request)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("encrypt result = %s\n", cipherString)
    request.CipherData = cipherString
    plainText, err := c.SymDecrypt(request)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("decrypt result = %s\n", plainText)
}
```

#### 非对称加密

支持RSA加密、RSA签名以及签名验证

```go
package main

import (
    "fmt"
    "github.com/pyihe/secret"
)

func main() {
    c := secret.NewCipher()
    _, _, err := c.GenerateRSAKey(1024, "conf", secret.PKCSLevel1)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    cipherString, err := c.RSAEncryptToString("this field is for data to be encrypt", secret.RSAEncryptTypeOAEP, nil)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("rsa encrypt result: %s\n", cipherString)
    plainText, err := c.RSADecrypt(cipherString, secret.RSAEncryptTypeOAEP, nil)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("%s\n", plainText)
}
```

#### Hash函数

支持大部分Hash函数

```go
package main

import (
	"crypto"
	"fmt"
	"github.com/pyihe/secret"
)

func main() {
    var data = "this is data for hash"
    h := secret.NewHasher()
    hashStr, err := h.HashToString(data, crypto.SHA256)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("hash result: %s\n", hashStr)
}
```

#### 签名与验证签名

支持DSA、ECDSA、Ed25519签名和签名验证。

```go
package main

import (
    "crypto"
    "crypto/dsa"
    "fmt"
    "github.com/pyihe/secret"
)

func main() {
    var data = "this is data for hash"
    s := secret.NewSigner()
    if err := s.SetDSAKey(dsa.L2048N256); err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    signString, err := s.DSASignToString(data, crypto.SHA256)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("sign result: %s\n", signString)
    ok, err := s.DSAVerify(data, signString, crypto.SHA256)
    if err != nil {
        fmt.Printf("%v\n", err)
        return
    }
    fmt.Printf("verify result: %v\n", ok)
}
```