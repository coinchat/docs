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
* callback_url  （可选）充值成功之后，将通过此 URL 将 deposit 信息通过 POST 方式发送到此 URL，发送的内容与通过“查询托管支付请求”接口返回的数据相同

### 查询托管支付请求 GET v1/entrust_wallet/deposit/load

*此接口需要签名*

* deposit_no    （必填）托管钱包支付请求的编号


### 发布一笔链上交易 POST v1/entrust_wallet/transaction/add

*此接口需要签名*

* wallet_id     （必填）要使用的托管钱包的编号
* transaction   （必填）要发布到以太坊的交易数据，以 JSON 字符串表示
* partner_nonce （可选）商户生成的一个 nonce 值（不是以太坊交易的 nonce 值）。如果传入了曾经出现过的 nonce 值，Coinchat 就不会再发布这个请求。这是为了避免因为网络故障等原因导致商户重复发布相同的 transaction


