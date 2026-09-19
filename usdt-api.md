# 币达商户 API 对接文档

> 版本：v1.0  
> 更新时间：2026-09-13

---

## 目录

- [1. 接入说明](#1-接入说明)
- [2. 签名规则](#2-签名规则)
- [3. 代收接口](#3-代收接口)
- [4. 代付接口](#4-代付接口)
- [5. 异步通知](#5-异步通知)
- [6. 附录](#6-附录)
- [7. 代付反查](#7-代付反查)

---

## 1. 接入说明

### 1.1 基本信息

| 项目   | 说明                                |
| ---- | --------------------------------- |
| 请求协议 | HTTPS                             |
| 请求方式 | POST                              |
| 数据格式 | application/x-www-form-urlencoded |
| 字符编码 | UTF-8                             |

### 1.2 商户凭证

接入前需获取以下凭证：

| 参数          | 说明               |
| ----------- | ---------------- |
| `appid`     | 商户唯一标识           |
| `appsecret` | 商户密钥（用于签名，请妥善保管） |

### 1.3 链类型说明

| chain_type | 链类型         |
| ---------- | ----------- |
| 1          | TRC20（波场链）  |
| 2          | ERC20（以太坊链） |

---

## 2. 签名规则

### 2.1 签名步骤

1. **过滤参数**：去除 `signature` 参数和空值参数
2. **参数排序**：将剩余参数按照 key 的 ASCII 码升序排序
3. **拼接字符串**：使用 `key=value&` 格式拼接，末尾追加 `&appsecret=您的密钥`
4. **MD5加密**：对拼接后的字符串进行 MD5 加密，结果转为大写

### 2.2 签名示例

**请求参数：**

```json
{
    "appid": "your_appid",
    "amount": "100.00",
    "order_sn": "M202601010001",
    "timestamp": "1735689600"
}
```

**签名过程：**

```
1. 按 key 排序: appid, amount, order_sn, timestamp
2. 拼接字符串: amount=100.00&appid=your_appid&order_sn=M202601010001&timestamp=1735689600&appsecret=您的密钥
3. MD5加密并转大写: 9A8B7C6D5E4F3A2B1C0D...
```

### 2.3 Java 签名代码示例

```java
import java.security.MessageDigest;
import java.util.Map;
import java.util.TreeMap;

public class SignUtil {
    public static String getSignature(Map<String, String> data, String secret) {
        if (data == null || data.isEmpty()) {
            return md5("&appsecret=" + secret).toUpperCase();
        }

        // 过滤空值和签名字段(使用 TreeMap 自动按照 key 的 ASCII 码升序排序)
        Map<String, String> filteredData = new TreeMap<>();
        for (Map.Entry<String, String> entry : data.entrySet()) {
            String key = entry.getKey();
            Object val = entry.getValue();
            // 过滤 signature 字段、null 值以及 toString() 后的空字符串
            if (!"signature".equals(key) && val != null) {
                String valueStr = String.valueOf(val).trim();
                if (!valueStr.isEmpty()) {
                    filteredData.put(key, valueStr);
                }
            }
        }

        // 拼接字符串: key1=value1&key2=value2&...
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<String, String> entry : filteredData.entrySet()) {
            sb.append(entry.getKey()).append("=").append(entry.getValue()).append("&");
        }
        sb.append("appsecret=").append(secret);

        // MD5加密并转大写
        return md5(sb.toString()).toUpperCase();
    }

    private static String md5(String str) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] bytes = md.digest(str.getBytes("UTF-8"));
            StringBuilder sb = new StringBuilder();
            for (byte b : bytes) {
                sb.append(String.format("%02x", b));
            }
            return sb.toString();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 2.4 timestamp 说明

- `timestamp` 为当前 Unix 时间戳（秒级）
- 服务器验证签名有效期为 **5分钟**，请确保服务器时间准确

### 2.5 Headers 说明

- `x-api-key` 为商户ID
- `x-timestamp` 为时间戳
- `x-signature` 为签名

---

## 3. 代收接口

### 3.1 创建收款订单

**接口地址：** `/usdt-api/collect_create`

**请求参数：**

| 参数           | 必填  | 类型     | 说明                  |
| ------------ | --- | ------ | ------------------- |
| appid        | 是   | string | 商户ID                |
| order_sn     | 是   | string | 商户订单号（唯一）           |
| amount       | 是   | double | 支付金额（USDT）          |
| chain_type   | 是   | int    | 链类型：1=TRC20，2=ERC20 |
| notify_url   | 是   | string | 异步通知地址              |
| username     | 是   | string | 付款用户标识              |
| callback_url | 否   | string | 同步回调地址              |
| product_name | 否   | string | 商品名称                |
| product_desc | 否   | string | 商品描述                |
| product_num  | 否   | int    | 商品数量                |
| attach       | 否   | string | 附加数据（原样返回）          |
| timestamp    | 是   | long   | 时间戳                 |
| signature    | 是   | string | 签名                  |

**响应参数：**

```json
{
    "code": 0,
    "msg": "操作成功",
    "data": {
        "appid": "your_appid",
        "chain": "TRON",
        "token": "TRC20",
        "amount": 100,
        "orderSn": "M202601010003",
        "address": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "signature": "55B55B966E0006AA8F5F5182E79FE559",
        "cashier-en": "https://域名/apps/cashier/en/M202601010003",
        "cashier-zh": "https://域名/apps/cashier/zh/M202601010003",
        "base64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAASwAAAEsAQAAAABRBrPYAAAByklEQVR42u2aQY7CMAxFXbHoskfoUThaORpHmSN0yQLhcWwnNQOVaMxmpJ8FqprXjW19/zgQf7JWAgbsP2M/ZOssD6cH8eXMfNMH3wCWwiaN9EWxGPvCMz+AJbFbCbVg8jAwLVei8W5vLC/AvoLJ7iDBp9mCD+yrmJW2iIn8Kg/sK1jVkInvrtWWjh2pAXYIax1wssbXHnYaJbAjWLBpY5UOleg9mwfsCCYvZbdJx1bkfJ3XmCxgPVgJPpuvMEHmYo9dVYrlAJbDiiv2LJhqTK0DWrUDy2CjKUWzx17b0gFXYFmsGAzrd9r4aGEXk4VjkQPrxVQxRJDdHjvvHzKwFGbS4QMf5T0dVEdAwBJYtRPuNFQ6zFeoegPLYfWkTKR8Cf5aP7RWCCyBBQ2xkv4r2sAy2FMW1L7N6uhs8hPPzsD6MB3vlA54r/4tOg1gGSyendl2V0+H5OXtaALY51hbJialyM1ybPICrB9rc8t4yjvVCdvrWBjYMSxM3a3xzX5kfjkDAuvBnq7k3L81Ddm5uQN2FKtnDfbLo9IB32QBWCfW5payBrYshMEasD4s3Hg2/1Zb4eZDgPVhzzeerbY9CwOwFIb/NAIDJtgvO1FZBIFSL7AAAAAASUVORK5CYII=",
        "status": 0,
        "time_out": 1753358299
    }
}
```

**响应字段说明：**

| 字段         | 说明        |
| ---------- | --------- |
| address    | 收款地址      |
| cashier-en | 收银台地址（英文） |
| cashier-zh | 收银台地址（中文） |
| time_out   | 订单超时时间戳   |

**订单状态说明：**

| status | 说明   |
| ------ | ---- |
| 0      | 未支付  |
| 1      | 支付成功 |
| 2      | 已超时  |

---

### 3.2 收款订单查询

**接口地址：** `/usdt-api/collect_search`

**请求参数：**

| 参数        | 必填  | 类型     | 说明    |
| --------- | --- | ------ | ----- |
| appid     | 是   | string | 商户ID  |
| order_sn  | 是   | string | 商户订单号 |
| timestamp | 是   | long   | 时间戳   |
| signature | 是   | string | 签名    |

**响应参数：**

```json
{
    "code": 0,
    "msg": "操作成功",
    "data": {
        "appid": "your_appid",
        "chain": "TRON",
        "token": "TRC20",
        "amount": 100,
        "orderSn": "M202601010003",
        "address": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "signature": "55B55B966E0006AA8F5F5182E79FE559",
        "cashier-en": "https://域名/apps/cashier/en/M202601010003",
        "cashier-zh": "https://域名/apps/cashier/zh/M202601010003",
        "base64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAASwAAAEsAQAAAABRBrPYAAAByklEQVR42u2aQY7CMAxFXbHoskfoUThaORpHmSN0yQLhcWwnNQOVaMxmpJ8FqprXjW19/zgQf7JWAgbsP2M/ZOssD6cH8eXMfNMH3wCWwiaN9EWxGPvCMz+AJbFbCbVg8jAwLVei8W5vLC/AvoLJ7iDBp9mCD+yrmJW2iIn8Kg/sK1jVkInvrtWWjh2pAXYIax1wssbXHnYaJbAjWLBpY5UOleg9mwfsCCYvZbdJx1bkfJ3XmCxgPVgJPpuvMEHmYo9dVYrlAJbDiiv2LJhqTK0DWrUDy2CjKUWzx17b0gFXYFmsGAzrd9r4aGEXk4VjkQPrxVQxRJDdHjvvHzKwFGbS4QMf5T0dVEdAwBJYtRPuNFQ6zFeoegPLYfWkTKR8Cf5aP7RWCCyBBQ2xkv4r2sAy2FMW1L7N6uhs8hPPzsD6MB3vlA54r/4tOg1gGSyendl2V0+H5OXtaALY51hbJialyM1ybPICrB9rc8t4yjvVCdvrWBjYMSxM3a3xzX5kfjkDAuvBnq7k3L81Ddm5uQN2FKtnDfbLo9IB32QBWCfW5payBrYshMEasD4s3Hg2/1Zb4eZDgPVhzzeerbY9CwOwFIb/NAIDJtgvO1FZBIFSL7AAAAAASUVORK5CYII=",
        "status": 0,
        "time_out": 1753358299
    }
}
```

---

## 4. 代付接口

### 4.1 创建下发订单

**接口地址：** `/usdt-api/payment_create`

**请求参数：**

| 参数         | 必填  | 类型     | 说明                  |
| ---------- | --- | ------ | ------------------- |
| appid      | 是   | string | 商户ID                |
| order_sn   | 是   | string | 商户订单号（唯一）           |
| amount     | 是   | string | 代付金额（USDT）          |
| address    | 是   | string | 收款地址                |
| chain_type | 是   | int    | 链类型：1=TRC20，2=ERC20 |
| notify_url | 是   | string | 异步通知地址              |
| attach     | 否   | string | 附加数据                |
| timestamp  | 是   | long   | 时间戳                 |
| signature  | 是   | string | 签名                  |

**响应参数：**

```json
{
    "code": 0,
    "msg": "操作成功",
    "data": {
        "appid": "your_appid",
        "chain": "TRON",
        "token": "TRC20",
        "amount": 100,
        "address": "TSeu7b3XHuPn6SZcUDZr7HafKdgw7h1Ghk",
        "orderSn": "M202601010002",
        "signature": "8A76383F7D567BBF098BFE7835346901",
        "status": 0
    }
}
```

**注意事项：**

- 代付金额将从商户余额中扣除
- 代付手续费另计，从余额中额外扣除
- 请确保商户余额充足（代付金额 + 手续费）
- 收款地址必须与链类型匹配（TRC20 地址以 T 开头，ERC20 地址以 0x 开头）

---

### 4.2 下发订单查询

**接口地址：** `/usdt-api/payment_search`

**请求参数：**

| 参数        | 必填  | 类型     | 说明    |
| --------- | --- | ------ | ----- |
| appid     | 是   | string | 商户ID  |
| order_sn  | 是   | string | 商户订单号 |
| timestamp | 是   | long   | 时间戳   |
| signature | 是   | string | 签名    |

**代付状态说明：**

| status | 说明      |
| ------ | ------- |
| 0      | 待处理     |
| 1      | 处理中     |
| 2      | 支付中     |
| 3      | 待确认     |
| 4      | 已确认     |
| 9      | 已拒绝/已取消 |

**响应参数：**

```json
{
    "code": 0,
    "msg": "操作成功",
    "data": {
        "appid": "your_appid",
        "chain": "TRON",
        "token": "TRC20",
        "amount": 100,
        "address": "TSeu7b3XHuPn6SZcUDZr7HafKdgw7h1Ghk",
        "orderSn": "M202601010002",
        "signature": "8A76383F7D567BBF098BFE7835346901",
        "success_time": 1735689900,
        "status": 4
    }
}
```

---

## 5 基础信息查询

### 5.1 商户余额查询

**接口地址：** `/usdt-api/balance`

**请求参数：**

| 参数        | 必填  | 类型     | 说明   |
| --------- | --- | ------ | ---- |
| appid     | 是   | string | 商户ID |
| timestamp | 是   | long   | 时间戳  |
| signature | 是   | string | 签名   |

**响应参数：**

```json
{
    "code": 0,
    "msg": "操作成功",
    "data": {
        "trc10Amt": 1000520.72,
        "trc20Amt": 1000139.50,
        "erc10Amt": 200.00,
        "erc20Amt": 203.00
    }
}
```

**响应字段说明：**

| 字段       | 说明            |
| -------- | ------------- |
| trc10Amt | TRC10-TRX 余额  |
| trc20Amt | TRC20-USDT 余额 |
| erc10Amt | ERC10-ETH 余额  |
| erc20Amt | ERC20-USDT 余额 |

---

## 6. 异步通知

### 6.1 代收回调通知

当订单支付成功后，系统会向商户的 `notify_url` 发送 POST 请求。

**通知参数：**

| 参数           | 说明         |
| ------------ | ---------- |
| appid        | 商户ID       |
| order_sn     | 商户订单号      |
| amount       | 支付金额       |
| status       | 订单状态（1=成功） |
| success_time | 支付成功时间戳    |
| attach       | 附加数据       |
| signature    | 签名         |

**响应要求：**

- 收到通知后，请返回纯文本 `OK`（大写）
- 返回 `OK` 后，系统将停止通知
- 未返回 `OK`，系统将重试通知（最多 5 次）

**重试策略：**

| 次数  | 间隔   |
| --- | ---- |
| 第1次 | 立即   |
| 第2次 | 20秒后 |
| 第3次 | 30秒后 |
| 第4次 | 1分钟后 |
| 第5次 | 5分钟后 |

---

### 6.2 代付回调通知

当代付完成后，系统会向商户的 `notify_url` 发送 POST 请求。

**通知参数：**

| 参数           | 说明         |
| ------------ | ---------- |
| appid        | 商户ID       |
| order_sn     | 商户订单号      |
| amount       | 代付金额       |
| status       | 订单状态（4=成功） |
| success_time | 完成时间戳      |
| signature    | 签名         |

**响应要求：** 同代收回调

---

## 7. 附录

### 7.1 错误码说明

| 错误码 | 说明   |
| --- | ---- |
| 0   | 成功   |
| 500 | 通用错误 |

### 7.2 常见错误信息

| 错误信息                 | 说明                 |
| -------------------- | ------------------ |
| Method not supported | 方法不支持：方法名错误或调用方式错误 |
| 商户不存在                | appid 无效或商户被禁用     |
| 签名错误                 | 签名验证失败             |
| 请求已过期                | 时间戳超过有效期（5分钟）      |
| 请求过于频繁               | 触发接口限频             |
| 该链路目前不支持             | chain_type 参数错误    |
| 商户订单号重复              | 订单号已存在             |
| 收款地址不是TRC地址          | TRC20 地址格式错误       |
| 收款地址不是ETH地址          | ERC20 地址格式错误       |
| 商户TRC的USDT余额不足       | 代付时余额不足            |
| 商户ERC的USDT余额不足       | 代付时余额不足            |

### 7.3 接口限频

| 接口   | 限制        |
| ---- | --------- |
| 创建订单 | 10次/分钟/IP |
| 订单查询 | 20次/分钟/IP |

### 7.4 注意事项

1. **签名验证**：所有请求必须携带有效签名，签名错误将被拒绝
2. **时间戳**：请确保服务器时间准确，时间戳有效期为 5 分钟
3. **订单号**：商户订单号必须唯一，重复订单号将被拒绝
4. **回调处理**：请做好幂等处理，同一笔订单可能收到多次回调通知
5. **HTTPS**：生产环境请使用 HTTPS 协议
6. **地址格式**：
   - TRC20 地址以 `T` 开头，长度 34 位
   - ERC20 地址以 `0x` 开头，长度 42 位

### 7.5 测试环境

| 项目   | 说明                    |
| ---- | --------------------- |
| 商户后台 | `https://域名` |
| 测试账号 | 请联系客服获取               |

### 7.6 联系方式

如有技术问题，请联系：

- Telegram: @xxxxx
- Email: support@xxxxx.com

---

---

## 8 代付反查

### 8.1 功能说明

为保障代付（出款）安全，平台在收到您的代付申请后、正式受理前，会先回调**您提供的反查接口**，向您确认这笔出款是真实有效的。

- 校验**通过**：平台继续受理该笔代付订单。
- 校验**不通过**（或接口超时 / 报错）：平台拦截该笔代付，不予受理。

请您开发并提供一个反查接口地址（建议 HTTPS），并在商户后台配置「开启反查」+「反查地址」。

### 8.2 调用方向

平台 → 商户（由平台主动请求您的接口）

### 8.3 请求说明

| 项目   | 说明                 |
| ---- | ------------------ |
| 请求方式 | `POST`             |
| 数据格式 | `application/json` |
| 连接超时 | 5 秒                |
| 总超时  | 10 秒               |

### 请求参数（JSON Body）

| 参数名                  | 类型     | 说明                       |
| -------------------- | ------ | ------------------------ |
| `order_sn` | string | 商户订单号（即您提交代付时的 `order_sn`）  |
| `address` | string | 提现收款地址                   |
| `amount`    | string | 提现金额（保留两位小数，例如 `100.00`） |

### 请求示例

```json
{
    "order_sn": "D20260613100001",
    "address": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "amount": "100.00"
}
```

### 8.4 返回说明（重要）

您的接口需返回 JSON，平台依据返回的 `code` 字段判断：

| 返回 code | 含义            | 平台处理     |
| ------- | ------------- | -------- |
| `200`   | 校验通过，该笔出款真实有效 | 继续受理代付   |
| 其他任意值   | 校验不通过         | 拦截，不受理代付 |

### 通过示例（必须返回此结构才放行）

```json
{
    "code": 200,
    "message": "success"
}
```

### 不通过示例

```json
{
    "code": 400,
    "message": "订单不存在或金额不匹配"
}
```

### 8.5 对接注意事项

1. 接口必须**稳定可用**。若接口超时、网络错误、返回非 JSON 或 `code` 非 `200`，平台都会判定为校验不通过并拦截该笔代付。
2. 建议在接口内核对以下信息后再返回 `200`：
   - 订单号 `user_withdrawal_id` 是您系统中真实存在且待出款的订单；
   - 收款地址 `withdrawal_address` 与该订单一致；
   - 金额 `currency_amount` 与该订单一致。
3. 接口响应应尽量快（建议 1 秒内），避免因超时被拦截。
4. 反查地址需在商户后台正确配置并开启「反查」开关后才会生效。

## 更新日志

| 版本   | 日期         | 内容   |
| ---- | ---------- | ---- |
| v1.0 | 2026-09-13 | 初始版本 |
