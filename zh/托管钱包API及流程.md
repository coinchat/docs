# 托管钱包 API 及流程

## 整体介绍
托管钱包是 Coinchat 提供给第三方商户的一个服务。商户可以在 Coinchat 上申请创建一个托管钱包，用户可以向这个托管钱包内存入一定的数量的数字货币，随后商户可以通过 Coinchat API 任意调用托管钱包账户内的钱，包括转账、调用智能合约等操作。


## 接口签名
部分接口需要进行签名，签名方法请参考《商户相关 API》中的说明。

# 具体流程

1. 商户调用“创建托管支付请求”(v1/entrust_wallet/deposit/add)接口，传入要托管的金额、对应的 Coinchat 用户的编号以及预存的 ETH 手续费，接口会返回一个 wallet_id 和一个 deposit_no。其中 wallet_id 是托管钱包的编号，在用户存入资金后，商户可以通过这个钱包编号任意使用这个钱包。deposit_no 是充值订单编号，商户需要将这个编号提交给 Coinchat 用户要求用户进行支付。

2. 商户使用 JSSDK 调用 entrustPay() 方法，传入 deposit_no。此时，Coinchat 会弹出支付页面，等待用户确认并支付。

3. 用户支付完成后，商户会收到回调通知。当 deposit.status 为 success 时，表示用户已支付成功，款项已进入托管钱包。这时候商户就可以开始自由使用托管钱包了。

商户也可以调用“查询托管支付请求”(v1/entrust_wallet/deposit/load)接口，主动查询用户是否已经付款成功。

4. 商户自行组装用于以太坊的 transaction 数据，然后调用“发布一笔链上交易”(v1/entrust_wallet/transaction/add)接口，传入 transaction 和 wallet_id，此接口会同步返回发布后的 transaction hash

例如，要向 0x077c984542199a7a6f1d2163038a4a1d2b6c8ef0 转账 1wei，那么 transaction 数据就是这样子的：
```
{ "value": "0x1", "to": "0x077c984542199a7a6f1d2163038a4a1d2b6c8ef0", "data": "0x"}
```


## API 列表
### 创建托管支付请求 POST v1/entrust_wallet/deposit/add

*此接口需要签名*

* user_id       （必填）创建的托管钱包对应的用户的用户编号
* eth_fee       （必填）托管钱包中预先存入用作手续费的 ETH 金额
* wallet_id     （可选）如果不填写，则自动创建一个新的托管钱包。如果填写了此参数，则复用之前已经创建过的托管钱包。
* coin          （必填）要存入的货币的简称
* coin_amount   （必填）要存入的货币的数量
* remark        （必填）显示给用户的备注信息
* callback_url  （可选）充值成功之后，将通过此 URL 将 deposit 信息通过 POST 方式发送到此 URL，发送的内容与通过“查询托管支付请求”接口返回的数据相同。回调详情见本文的“回调说明”章节

### 查询托管支付请求 GET v1/entrust_wallet/deposit/load

*此接口需要签名*

* deposit_no    （必填）托管钱包支付请求的编号


### 发布一笔链上交易 POST v1/entrust_wallet/transaction/add

*此接口需要签名*

* wallet_id     （必填）要使用的托管钱包的编号
* transaction   （必填）要发布到以太坊的交易数据，以 JSON 字符串表示
* partner_nonce （可选）商户生成的一个 nonce 值（不是以太坊交易的 nonce 值）。如果传入了曾经出现过的 nonce 值，Coinchat 就不会再发布这个请求。这是为了避免因为网络故障等原因导致商户重复发布相同的 transaction


### 将托管钱包中的资金退还给用户 POST v1/entrust_wallet/refund

*此接口需要签名*

* wallet_id       （必填）要退还的托管钱包的编号

调用此接口后，托管钱包中的剩余资金将全部回到用户的账户中，且再也不能通过此钱包发起链上交易


## 回调说明
所有的回调均通过 HTTP POST 方式，将一段 JSON 数据发送到你指定的 URL。

收到回调后，应该返回 HTTP 200 并输出 ```ok``` 两个字，表示回调已收到并处理完成。如果输出其他内容，Coinchat 会尝试继续回调。

Coinchat 最多尝试 10 次回调，两次回调的间隔按指数增长。一般情况下，在首次回调失败之后，Coinchat 大概会在 1～2 日内继续回调，直到达到回调失败次数上线。

由于网络状况不可预知，即使在你输出了 ```ok``` 之后，Coinchat 仍可能再次发送请求，你的处理代码中需要处理好重复收到回调请求，或同时收到两个相同的回调请求的情况。

### 回调签名
Coinchat 发送的回调中，会附带一个名为 ```X-COINCHAT-CALLBACK-SIGNTURE``` 的 HTTP 头，内容是签名值。签名是请求体中 JSON 字符串数据的 HMAC-SHA256 哈希值，以十六进制表示，字符均为小写。HMAC 密钥是商户的 API_SECRET.

### 回调签名实例
例如，Coinchat 回调的内容是 
```
{"foo": "bar", "var":1}
```
商户的 API_SECRET 是 ```api_secret_123456```

签名 signture = HMAC-SHA256('{"foo": "bar", "var":1}', 'api_secret_123456')

签名结果是 `fb80d873cadd2d0c739c0709d3c600dd5ac987ac7230c4cc7da5ca9b448c8d4d`

需要注意的是，你需要将 Coinchat 传来的 JSON 字符串原样传入 HMAC 函数，不能对 JSON 字符串预先做任何处理，否则将得不到正确的签名结果。

将你计算得到的签名结果和 X-COINCHAT-CALLBACK-SIGNTURE 的值进行对比，如果两者相同，说明请求确实是 Coinchat 发出的，可以放心使用。
