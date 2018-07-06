# 商户相关 API

## 签名方法
所有接口的请求都使用相同的签名方法。

请求参数可以是 HTTP GET 的内容，也可以是 HTTP POST 的内容，也可以是调用 JSSDK 的方法时传入的 object 对象。

例如，我们要请求用户编号为 3 的用户的信息，那么请求内容可以是

```
curl https://{$api_domain}/user/info?user_id=3
```

```
curl https://{$api_domain}/user/info -d "user_id=3"
```

```
CoinchatJSBridge.invoke('getUserInfo', { user_id: 3})
```

签名时，需要将请求内容当成一个键值对进行处理。签名的具体计算过程是：
1. 根据键名对键值对做升序排序；
2. 将每一个键值中的键和值用等号连接，键名和值不需要做任何编码处理；
3. 把在上一步中拼接好的字符串，使用 & 符号相连，得到一个待签名字符串；
4. 对上一步得到的待签名字符串计算 HMAC-SHA256 散列值，结果以小写的十六进制可读字符串表示，HMAC 秘钥为商户的 API_SECRET

## 请求接口
请求接口前，需要在请求参数中添加 timestamp 字段和 nonce 字段。timestamp 是当前的 UNIX 时间戳，单位为秒，nonce 是一个随机字符串，由客户端生成，并保证每个请求的 nonce 都是唯一的。

为了区分商户身份，请求参数中还应该添加 partner_no 或 api_key 参数。

在请求内容中添加了 timestamp、nonce 以及 partner_no（或 api_key） 参数后，将请求内容进行签名，把签名结果作为 sign 参数添加到请求内容中，然后就可以提交请求了。

注意，大部分情况下，都应该使用 partner_no 参数，不要使用 api_key 参数。api_key 参数只在极少的情况下会使用。

## 示例
假设有一个商户，API_KEY 是 foo, API_SECRET 是 bar，商户要发送一个这样的请求：

```
{user_id: 1, group_id: 2}
```

首先在请求参数中添加 timestamp、nonce 以及 api_key 信息，得到：

```
{user_id: 1, group_id: 2, timestamp: 1234567890, nonce: "some_random_character", api_key: "foo"}
```

对请求内容的参数进行排序，得到：

```
{api_key: "foo", group_id: 2, nonce: "some_random_character", timestamp: 1234567890, user_id:1}
```

将请求内容拼接成待签名字符串，得到：

```
api_key=foo&group_id=2&nonce=some_random_character&timestamp=1234567890&user_id=1
```

计算 HMAC-SHA256('api_key=foo&group_id=2&nonce=some_random_character&timestamp=1234567890&user_id=1', 'bar')，得到：

```
b27eb8f5bc570ca9782dd572aae7d7cc51f4ba4c56aca786c40a780c8ed81631
```

将签名结果作为 sign 参数添加回请求内容中，最后的请求内容是：

```

{
    user_id: 1,
    group_id: 2,
    timestamp: 1234567890,
    nonce: "some_random_character",
    api_key: "foo",
    sign: "b27eb8f5bc570ca9782dd572aae7d7cc51f4ba4c56aca786c40a780c8ed81631"
}
```


