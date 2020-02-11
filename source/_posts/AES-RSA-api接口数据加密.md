---
title: AES RSA api接口数据加密
date: 2020-02-11 14:13:32
tags:
---
解决方案
需要联合使用对称加密AES与非对称加密RSA

每次调用客户端随机产生一个aes密码，并把调用明文加密成调用密文，然后将aes密码用rsa公玥加密成密码密文后连同调用调用密文一起传给后端，后端使用rsa私玥对密码密文进行解密aes密码，然后aes密码解密调用密文获得调用明文。然后进行业务处理。后端在返回响应数据时，使用刚才的aes密码，将响应明文加密成响应密文，传递给客户端。客户端收到响应密文后，使用aes密码解密得到响应明文。

![20180227173359994_biliyun](https://cdn.oss.biliyun.net/blog/1.0/production/20180227173359994_biliyun.png)

```
明文数据使用 AES 对称加密算法进行加密, AES 密钥使用 RSA 进行加密传输并对数据进行数据签名
加密结果密文数据结构: [数字签名(AES密钥密文+明文数据密文)][AES密钥密文][明文密文]
```

**参考**
[使用AES ECB PKCS5Padding+RSA对接口进行签名及加密的go代码实现](http://www.lampnick.com/php/728)
[使用 AES 和 RSA 对数据进行加密和解密](https://github.com/takmongwai/aes-rsa)
[RSA总结](https://blog.csdn.net/xz_studying/article/details/80314111)

[aes cfb参考nodejs](https://ekyu.moe/article/aes-cfb-in-golang-and-nodejs/)

[aes gcm cbc cfb等5种模式简介](https://www.cnblogs.com/starwolf/p/3365834.htmlhttps://www.cnblogs.com/starwolf/p/3365834.html)