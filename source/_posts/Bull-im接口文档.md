---
title: Bull.im接口文档
date: 2020-02-11 14:14:56
tags:
---



### 接口地址
> https://api.bull.im

### 接口统一header返回说明

header				|说明    | 备注
:----:			    |:---   | :---:
x-session		    | session值  | 
x-csrf				| csft token | 

### 接口统一header请求说明

**主要公共参数:**

header				|说明    | 备注
:----:			    |:---   | :---:
x-session		    | 从header同名参数取 | （可选）默认使用cookie
x-csrf				| 从header同名参数取 | post请求必填，防止xss攻击
uid | 用户id | (可选)
Locale | 区域（86） | (可选)
TimeZone| 时区（+8） | (可选)
Lang | 语言（zh-CN） | (可选)
IP | ip(127.0.0.1) | (可选)
DevID | uuid | (可选)
Platform | "Android","iOS","Web" | 必选
DevOS | windows,ios | 必选
DevHW | {"cpu":4,"gpu":"","mem":"2GB"} |必选

---

### 统一消息说明

**错误消息说明：**
success: http.statuscode == 200
failed: http.statuscode != 200

```
{
    "code": xxxxx, //错误码
    "message":xxxx //错误消息
}

{
	1001: "verify captcha code fail",     // 验证码错误
	1002: "need captcha code",            // 需要验证码
	1003: "email/sms verifycode fail",    // 邮件/sms验证码错误
	1004: "email/sms verifycode timeout", // 邮件/sms验证码错误
	1005: "update error",                 // 更新失败
	1006: "update success",               // 更新成功

	1101: "user exist",            // 该账号已经存在
	1102: "user not found",        // 账号不存在
	1103: "user auth error",       // 用户验证失败
	1104: "login failed",          // 登录失败
	1105: "register failed",       // 注册失败
	1106: "verify salt failed",    //验证salt
	1107: "Username exists",       //用户名存在
	1108: "Email exists",          //邮箱存在
	1109: "Phone exists",          //电话存在
	1110: "Send sms/email failed", //发送验证码失败

	1201: "2FA Has Enabled",    // 2fa验证已经开启
	1202: "2FA Verify Fail",    // 2fa验证失败
	1203: "2FA Bind Fail",      // 2fa绑定失败
	1204: "2FA Has Disabled",   // 2fa未开启
	1205: "2FA need",           // 2fa需要验证
	1206: "Verify user need",   // 验证用户需要
	1207: "Verify user failed", // 严重用户失败
}
```

所有post/put等接口都后续都会做频率验证码验证，一开始没有验证，但当触发后会返回1002错误码请求验证验证码。目前有几个接口默认有图片验证码

**验证码参数:**
参数名称 | 类型 | 是否必须 | 说明
:---: | :---: | :---: | :---
generate_id|string| true| 20位长度
generate_code|string| true| 4位长度

### 1.请求图片验证码id

- **接口地址** ：

> GET /v1/captcha/generate

- **请求参数**: 无

- **返回参数**：

```
{
    "code": 200,
    "data": {
        "generate_id": "rbrQk24qJbykUDIw4kCr",
        "expire": 3600
    }
}
```

### 2.获取验证码图片

- **接口地址** ：

> GET /v1/captcha/generate/${captcha_id}.png

- **请求参数**: 无

### 3.获取qrcode图片

- **接口地址** ：

> GET /v1/qrcode?text=${otpauth}

- **请求参数**: 无

### 4.获取国家码

- **接口地址** ：

> GET /v1/tools/areas

- **请求参数**: 无

### 5.发送注册验证码

- **接口地址** ：

> POST /v1/register/start

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im

### 6.重新发送注册验证码

- **接口地址** ：

> POST /v1/register/start/vcode

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im

### 7.注册账号

- **接口地址** ：

> POST /v1/register

- **请求参数**: 无

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im
password|string|true|md5(inputpassword+salt)
salt|string|true|md5(code)
code|string|true|len(6)

### 8.登陆前获取账户信息（salt）

- **接口地址** ：

> GET /v1/login/start

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im

- **返回数据:**

```
{
    "salt": "xxxxx"
}
```

### 9.账号密码登录/退出

- **接口地址** ：

> POST /v1/login 登录
> GET /v1/logout 退出

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im
password|string|true|md5(inputpassword+salt)
salt|string|true|
totp_code|string|falase|这是二次验证码需要以后开启后才有，登录时会提示输入验证码，那个时候再弹出二次验证输入框 加上这个字段后重新提交 可以参考big.one的二次验证弹窗

### 10.验证码登陆发送验证码

- **接口地址** ：

> POST /v1/login/start/vcode

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im

### 11.验证码登陆

- **接口地址** ：

> POST /v1/login/code

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im
code|string|true|len(6)
totp_code|string|falase|这是二次验证码需要以后开启后才有，登录时会提示输入验证码，那个时候再弹出二次验证输入框 加上这个字段后重新提交 可以参考big.one的二次验证弹窗

### 12.找回密码前校验账号

- **接口地址** ：

> POST /v1/forget/password/start

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im

### 13.找回密码重新发送验证码

- **接口地址** ：

> POST /v1/forget/password/start/vcode

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im

### 14.找回密码重置密码

- **接口地址** ：

> POST /v1/forget/password

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im
code|string|true|len(6)
salt|string|true|md5(code)
password|string|true|md5(inputpassword+salt)

### 15.获取当前用户信息
- **登录用户:** true

- **接口地址** ：

> GET /v1/account/

- **请求参数**: 无


### 16.登录用户发送验证码
- **登录用户:** true

- **接口地址** ：

> POST /v1/account/vcode

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
type|string|true|目前只有2个值email/phone
这个主要是登录用户修改一些东西需要验证码时发送验证码 发送验证码前tab让他选择验证方式 返回的用户信息里有phone_status,email_status表示对应可用状态。一般用什么方式注册默认这个状态就是true

### 17.通过密码修改密码
- **登录用户:** true

- **接口地址** ：

> POST /v1/account/reset/password

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
salt|string|true|md5(code)这是新的salt
new_salt|string|true|md5(code)这是新的salt
password|string|true|md5(inputmassword+salt)
new_password|string|true|md5(inputmassword+newsalt)
generate_id|string| false| 20位长度(后期根据请求频率出现，默认不需要)
generate_code|string| false| 6位长度(后期根据请求频率出现，默认不需要)

### 18.通过发送验证码修改密码
- **登录用户:** true

- **接口地址** ：

> POST /v1/account/reset/password/code

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
type|string|true|目前只有2个值email/phone，前面登录用户用什么方式发送的验证码这里就填什么
code|string|true|len(6)
salt|string|true|md5(code)
password|string|true|md5(inputpassword+salt)

### 19.二次验证totp 获取qrcodeUrl&&secret
- **登录用户:** true

- **接口地址** ：

> GET /v1/acccount/totp/start

- **请求参数**: 无
- 
- **返回数据:**

```
{
    "code": 200,
    "data": {
        "otpauth": "otpauth%3A%2F%2Ftotp%2FBULL.im%3A8615102366689%3Fissuer%3DBULL.im%26secret%3DHLBUICZJTZBPSBIK",
        "secret": "HLBUICZJTZBPSBIK"
    }
}
```

> optauth这个放到前面生成二维码的参数text 显示绑定二次验证的二维码
> secret是密钥 用户自己保存好

### 20.二次验证开启
- **登录用户:** true

- **接口地址** ：

> POST /v1/account/totp/enable

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
code|string|true|len(6)

### 21.二次验证关闭

- **登录用户:** true

- **接口地址:** 

> POST /v1/accoount/totp/disable

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
code|string|true|len(6)

### 22.更新用户通用信息

- **登录用户:** true

- **接口地址:** 

> PUT /v1/accoount

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
name|string|false|用户昵称
nation|string|false|国家地区
timezone|string|false|时区
avatar|string|false|头像图片地址
lang|string|false|语言

> 需要更新什么对应传对应的字段

### 23.登录用户修改邮箱和电话发送验证码后的验证码验证

- **登录用户:** true

- **接口地址:** 

> POST /v1/accoount/verify/code

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
type|string|true|目前只有2个值email/phone，前面登录用户用什么方式发送的验证码这里就填什么
code|string|true|len(6)

### 24.登录用户修改邮箱

- **登录用户:** true

- **接口地址:** 

> PUT /v1/accoount/email

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im
code|string|true|len(6)

> 注意这里的账号验证码是新的    修改前的验证用前面的/account/vcode发送验证码和/account/verify/code验证验证码后才能在这里提交新账号修改

### 24.登录用户修改电话号码

- **登录用户:** true

- **接口地址:** 

> PUT /v1/accoount/phone

- **请求参数**: 

参数名称 | 类型 | 要求 | 说明
:---: | :---: | :---: | :---
account|string|true|phone: +86\|15102366688 email: admin@bull.im
code|string|true|len(6)

> 注意这里的账号验证码是新的    修改前的验证用前面的/account/vcode发送验证码和/account/verify/code验证验证码后才能在这里提交新账号修改

### 25.登录用户获取上传头像token

- **登录用户:** true

- **接口地址:** 

> GET /v1/accoount/upload/token

- **请求参数**: 无

