# COINCHAT 网页登录

如果用户在COINCHAT客户端中访问第三方网页，公众号可以通过COINCHAT网页授权机制，来获取用户基本信息，进而实现业务逻辑。

### 关于网页授权回调域名的说明

1、在COINCHAT开发者请求用户网页授权之前，开发者需要先到COINCHAT官网中的“开发者中心 - 管理API - 编辑 - 安全域名”的配置选项中，修改授权回调域名。请注意，这里填写的是域名（是一个字符串），而不是URL，因此请勿加 http:// 等协议头；

2、授权回调域名配置规范为全域名，比如需要网页授权的域名为：www.coinchat.im，配置以后此域名下面的页面http://www.coinchat.im/music.html 、 http://www.coinchat.im/login.html 都可以进行OAuth2.0鉴权。但http://pay.coinchat.im 、 http://music.coinchat.im 、 http:/coinchat.im无法进行OAuth2.0鉴权


### 关于特殊场景下的静默授权

1、当用户已经授权过某个开发API后，24小时内再次进入这个授权页面，就静默授权的，用户无感知；


具体而言，网页授权流程分为四步：

1、引导用户进入授权页面同意授权，获取code

2、通过code换取网页授权access_token，在返回access_token的时候会一并返回用户的基础信息。

3、如果需要，开发者可以刷新网页授权access_token，避免过期

4、通过网页授权access_token和user_id获取用户基本信息（可以用这个接口来更新用户资料）。

### 目录

1 第一步：用户同意授权，获取code

2 第二步：通过code换取网页授权access_token

3 第三步：刷新access_token（如果需要）

4 第四步：拉取用户信息

## 第一步：用户同意授权，获取code

在你的网页上，引导用户打开如下页面：

```
https://api.coinchat.im/oauth/authorize.html?partner_no=1532505616045468&redirect_uri=http%3A%2F%2Fweb.coinchat.com%2Fmisc%2Foauth%2Fcallback.html&response_type=code&scope=user_info&state=test#coinchat_redirect
```
尤其注意：跳转回调redirect_uri，应当使用https链接来确保授权code的安全性。

参数说明

参数 | 是否必须 | 说明
------------ | ------------- | ------------
partner_no | 是  |  API的唯一标识，在开发者中心中，申请API后可以获得
redirect_uri | 是  | 授权后重定向的回调链接地址， 请使用 urlEncode 对链接进行处理
response_type	 | 是  | 返回类型，请填写code
scope	| 是	| 应用授权作用域，请填写user_info
state	| 否	| 重定向后会带上state参数，开发者可以填写a-zA-Z0-9的参数值，最多128字节

\#coinchat_redirect	是	无论直接打开还是做页面302重定向时候，必须带此参数

下图为授权页面示例：

![](https://cdn.static.coinchat.im/upload/original_upload_image/9f/e8/9fe8eaecfd38cc3b017effa367674ca7224f1ee0fbf521f168d8d54ebe20cbaf.png)

### 用户同意授权后

如果用户同意授权，页面将跳转至 redirect_uri/?code=CODE&state=STATE。

code说明 ： code作为换取access_token的票据，每次用户授权带上的code将不一样，code只能使用一次，5分钟未被使用自动过期。

## 第二步：通过code换取网页授权access_token和用户信息

由于COINCHAT的API的api_secret和获取到的access_token安全级别都非常高，必须只保存在服务器，不允许传给客户端。后续刷新access_token、通过access_token获取用户信息等步骤，也必须从服务器发起。

请求方法

获取code后，请求以下链接获取access_token:

```
https://api.coinchat.im/v1/oauth/get_token?partner_no=PARTNER_NO&api_secret=API_SECRET&code=CODE&grant_type=authorization_code
```

参数说明

参数 | 是否必须 | 说明
------------ | ------------- | ------------
partner_no | 是  |  API的唯一标识，在开发者中心中，申请API后可以获得
api_secret | 是  |  API_SECRET，在开发者中心中，申请API后可以获得
code	 | 是  | 填写第一步获取的code参数
grant_type	| 是	| 填写为authorization_code

返回说明

正确时返回的JSON数据包如下：
```
{
    "status": "success",
    "code": 0,
    "data": {
        "create_time": 1537429169,
        "refresh_token": "fCYbfMlB5RoWA4r0iSEyDLGqvlecw6jG",
        "access_token": "r0D5N2PTvH9UEsjX9Gi4KtuzEDQoLmX8",
        "access_token_expire_time": 1537520379,
        "refresh_token_expire_time": 1540025979,
        "create_time_usec": 1537429169204535,
        "update_time": 1537433979,
        "update_time_usec": 0,
        "delete_time": 0,
        "delete_time_usec": 0,
        "user": {
            "user_id": "uFao5N2EHKlg-mevHZpsUoqyFmN5YnDF2",
            "name": "dreamcog",
            "avatar_url": "http://static.coinchat.com/upload/avatar/e9/93/e99330230270ab95284736d6ea8a17077900ff7d36d3baa384460d238cdb7421.jpg"
        },
        "partner": {
            "partner_no": "1536829343954693",
            "name": "未来存钱罐"
        }
    }
}
```

至此，基本授权流程结束，你可以通过以下第三第四步来更新用户信息。

## 第三步：刷新access_token（如果需要）

由于access_token拥有较短的有效期（24小时），当access_token超时后，可以使用refresh_token进行刷新，refresh_token有效期为30天，当refresh_token失效之后，需要用户重新授权。

请求方法

```
获取第二步的refresh_token后，请求以下链接获取access_token：
https://api.coinchat.im/v1/oauth/refresh_token.html?partner_no=PARTNER_NO&refresh_token=REFRESH_TOKEN
```

参数说明

参数 | 是否必须 | 说明
------------ | ------------- | ------------
partner_no | 是  |  API的唯一标识，在开发者中心中，申请API后可以获得
refresh_token | 是  |  填写通过第二步获取到的refresh_token参数

返回说明

正确时返回的JSON数据包如下：
```
{
    "status": "success",
    "code": 0,
    "data": {
        "create_time": 1537429169,
        "refresh_token": "GNeUuvn97e57pNkDfJWogdsb9xv9cn9A",
        "access_token": "7lQtWYQWAGuvuk7pEVlJxiHDrdYYdyHV",
        "access_token_expire_time": 1537516965,
        "refresh_token_expire_time": 1540022565,
        "create_time_usec": 1537429169204535,
        "update_time": 1537430565,
        "update_time_usec": 0,
        "delete_time": 0,
        "delete_time_usec": 0,
        "user": {
            "user_id": "uFao5N2EHKlg-mevHZpsUoqyFmN5YnDF2",
            "name": "username",
            "avatar_url": "http://static.coinchat.com/upload/avatar/e9/93/e99330230270ab95284736d6ea8a17077900ff7d36d3baa384460d238cdb7421.jpg"
        },
        "partner": {
            "partner_no": "1536829343954693",
            "name": "未来存钱罐"
        }
}
```

## 第四步：拉取用户信息

开发者可以通过access_token和user_id拉取用户信息了。

请求方法

```
获取第二步的refresh_token后，请求以下链接获取access_token：
https://api.coinchat.im/v1/oauth/user_info.html?partner_no=PARTNER_NO&refresh_token=REFRESH_TOKEN
```

参数说明

参数 | 是否必须 | 说明
------------ | ------------- | ------------
partner_no | 是  |  API的唯一标识，在开发者中心中，申请API后可以获得
access_token | 是  |  填写通过第二步获取到的access_token参数
openid	| 是  | 第二步获取到用户id
language | 是 | 返回国家地区语言版本，zh 简体中文, ja 日语, ko 韩语, en 英文


