# XPIN eSIM 分销商 OpenAPI 接入说明

> **文档版本：beta**（2026-09-16）
>
> 面向下游分销商的接入文档，覆盖认证、目录、开卡、幂等、履约、ICCID、续订、回调八条链路。
> 文中的响应样本取自接口的真实返回，其中 ICCID、IMSI、MSISDN、激活码已做脱敏。
>
> 当文档与运行中的服务不一致时，**以服务为准**。

---

## 目录

1. [基础约定](#1-基础约定)
2. [认证与签名](#2-认证与签名)
   - [2.1 签名算法 XPIN-KV1](#21-签名算法-xpin-kv1)
   - [2.2 nonce](#22-nonce)
   - [2.3 认证检查顺序](#23-认证检查顺序)
   - [2.4 scope](#24-scope)
   - [2.5 签名客户端](#25-签名客户端)
   - [2.6 自检向量](#26-自检向量)
3. [错误码](#3-错误码)
   - [3.1 非 200 的响应怎么处理](#31-非-200-的响应怎么处理)
4. [接口详解](#4-接口详解)
   - [4.1 商品列表 `/products/list`](#41-商品列表-productslist)
   - [4.2 商品详情 `/products/detail`](#42-商品详情-productsdetail)
   - [4.3 卡型能力 `/cardType`](#43-卡型能力-cardtype)
   - [4.4 开卡下单 `/order/create`](#44-开卡下单-ordercreate)
   - [4.5 订单查询 `/order/query`](#45-订单查询-orderquery)
   - [4.6 订单列表 `/order/list`](#46-订单列表-orderlist)
   - [4.7 续订资格判定 `/iccid/renew`](#47-续订资格判定-iccidrenew)
   - [4.8 续订下单 `/order/renew`](#48-续订下单-orderrenew)
   - [4.9 卡档案 `/iccid/profile`](#49-卡档案-iccidprofile)
   - [4.10 用量查询 `/order/usage`](#410-用量查询-orderusage)
   - [4.11 退款申请 `/order/refund`](#411-退款申请-orderrefund)
5. [业务流程与状态机](#5-业务流程与状态机)
   - [5.1 订单状态机](#51-订单状态机)
   - [5.2 开卡完整流程](#52-开卡完整流程)
   - [5.3 续订完整流程](#53-续订完整流程)
   - [5.4 退款完整流程](#54-退款完整流程)
6. [回调通知](#6-回调通知)
   - [6.1 事件类型](#61-事件类型)
   - [6.2 请求形状](#62-请求形状)
   - [6.3 验签](#63-验签)
   - [6.4 ACK 与重投](#64-ack-与重投)

---

## 1. 基础约定

| 项 | 值 |
|---|---|
| Base URL（沙盒环境） | `https://sandboxapi.xpin.network` |
| Base URL（生产环境） | 请向平台方获取 |
| 协议 | 全部 `POST`，`Content-Type: application/json` |
| 字符集 | UTF-8 |
| **成功判定** | **看响应体的 `code` 字段，不看 HTTP 状态码** |
| 凭证 | 平台发放的 `app_key` / `app_secret`（需要从平台方获取） |

**平台的业务应答一律是 `HTTP 200`**，响应体统一为：

```json
{ "data": {...}, "code": 200, "msg": "ok", "t": 1788431707137 }
```

业务成功还是失败，**看响应体的 `code`，不看 HTTP 状态码** —— 一个业务失败（余额不足、
幂等冲突、签名不对）同样是 `HTTP 200`，错误码在 `code` 里。

> 🔴 **但"请求没进到业务层"是另一回事，你的客户端必须能区分。** 路径或方法写错、
> 用了 `http://`、请求体过大、连接中途出错，这些都不会进到上面那套信封 ——
> 你拿到的是一个**真实的 HTTP 状态码**，响应体**可能不是 JSON**（网关的 HTML 错误页、
> 空响应、纯文本都可能）。
>
> `JSON.parse(响应体)` 在这种情况下会抛异常，而抛出来的是 `Unexpected token <` 这类
> 与真实原因毫无关系的错误 —— 排查方向会整个跑偏。
>
> **所以判定是两步，不是一步：**
>
> ```
> 第 1 步  HTTP 状态码 != 200  ->  请求没进业务层。响应体可能不是 JSON，不要解析。
>                                  按状态码处理，详见 §3.1。
> 第 2 步  HTTP 状态码 == 200  ->  平台应答。解析 JSON，看响应体的 `code`。
> ```
>
> 上面那句"看 `code` 不看 HTTP 状态码"说的是**第 2 步之内**的事，不是"完全不用看状态码"。

| 字段 | 类型 | 说明 |
|---|---|---|
| `data` | object | 业务数据。失败时为 `{}` |
| `code` | number | **业务码**。`200` 才是成功 |
| `msg` | string | 成功为 `ok`；失败为错误码或字段级提示 |
| `t` | number | 服务端毫秒时间戳。⚠️ 个别服务端异常响应（`406`）不带这个字段，不要当作必有字段解析 |

> ⚠️ **断言一定要写 `body.code === 200`**。写 `res.status === 200` 会把所有业务失败都当成功。

HTTP 80 端口的请求会 `301` 跳转到 HTTPS，且 **301 不保留 POST body**——直接用 `https://`。

---

## 2. 认证与签名

每个请求带四个头：

| 请求头 | 说明 |
|---|---|
| `x-api-key` | 你的 `app_key` |
| `x-timestamp` | **毫秒**时间戳（13 位），与服务器时钟偏差需 ≤ **5 分钟**（双向） |
| `x-nonce` | 随机串，匹配 `^[A-Za-z0-9._:-]{8,128}$`，**同一商户同一 nonce 在 10 分钟内只能用一次**（见 [2.2](#22-nonce)） |
| `x-sign` | HMAC-SHA256 十六进制**小写**，见 2.1 |

### 2.1 签名算法 XPIN-KV1

签的是**解析后的数据**，不是报文字节 —— 所以你的 body 用任何 JSON 库、任何键序、任何缩进
发出都行，也**不要求**你能控制 HTTP 客户端实际发出的字节。被签的对象固定七个字段：

```jsonc
{
  "sigAlg":    "xpin.esim.openapi.kv1",   // 字面量
  "apiKey":    "<x-api-key 的值>",
  "method":    "POST",
  "path":      "/openapi/v1/order/create", // 只取路径部分，**不含 query string**；
                                           // 归一后：重复斜杠折成一个、去尾斜杠、不解码 %XX
  "timestamp": "<x-timestamp 的值>",       // 字符串
  "nonce":     "<x-nonce 的值>",
  "body":      { ... }                     // 请求体本身
}
```

把它按下面四条规则拼成基串，再 `hex(HMAC_SHA256(app_secret, base))`。

**R1 拍平** —— 递归展开成「路径 → 标量」的列表：

- 嵌套对象：路径段用 `.` 连接。`{"data":{"orderNo":"X"}}` → 路径 `data.orderNo`
- 数组：下标就是一段。`{"tags":["a","b"]}` → `tags.0`、`tags.1`
- **空对象**落一条 `路径 → o:`，**空数组**落一条 `路径 → a:`。不留痕的话 `{"a":{}}` 与 `{}` 会拼出同一个基串
- 键名必须匹配 `^[A-Za-z][A-Za-z0-9_]*$`。不匹配就**拒绝**，不要尝试转义 —— 这条正是拍平
  能保持单射的原因（否则键名里的 `.` 会与路径分隔符混淆）

**R2 值编码**（带类型标记）：

| 值 | 编码 |
|---|---|
| 字符串 | `s:` + 原文（UTF-8，**不转义任何字符**） |
| 整数 | `i:` + 十进制，无前导零，`-0` 写作 `0` |
| 布尔 | `b:true` / `b:false` |
| `null` | `n:` |

类型标记不是装饰：没有它，字符串 `"1"` 与整数 `1` 会编码成同一个东西。

**数值必须是绝对值 ≤ 2⁵³−1 的整数**，判据是**值**不是字面量的写法（`2` 与 `2.0` 都按整数
处理）。小数、超范围整数、`NaN`、`Infinity` **一律拒绝，不要尝试编码** —— 它们的字符串化
在各语言里不一致（JS 的 `String(1.0)` 是 `"1"`，Java 的 `Double.toString(1.0)` 是 `"1.0"`），
给它们编一个表示等于把跨语言分歧固化进契约。**金额请用最小单位整数或字符串。**

> 💡 在 Python / Java 这类区分 int 与 float 的语言里，请在构造 body 时就把整数用整数类型
> 表达（Python 的 `2` 而不是 `2.0`），否则你的客户端可能在本地就拒签一个服务端接受的值。

**另有两条结构上限**：嵌套深度 8 层、叶子总数 512。超出分别是 `CANONICAL_TOO_DEEP` 与
`CANONICAL_TOO_LARGE`（都是 `400`）。正常业务载荷离这两条线很远。

**R3 每一项的字面量**（长度前缀）：

```
路径 + "=" + <编码值的 UTF-8 字节数> + ":" + 编码值
```

⚠️ 长度算的是**编码值整体**的字节数，**包含 `s:` / `i:` 这两个字符的类型标记**。
`"ak_demo"` 编码成 `s:ak_demo`，长度是 **9** 不是 7 —— 这是最常见的一处实现偏差。

长度前缀是**承重的**：没有它，`{"a":"1&b=s:2"}` 与 `{"a":"1","b":"2"}` 会拼出同一个基串 ——
而 `clientOrderNo` 这类字段的内容由你自由填写。

**R4 排序与连接** —— 按**完整路径的 UTF-8 字节序**升序排序，用 `&` 连接。键名限 ASCII
之后，字节序 = 码元序 = 字典序，三者不会分叉。

**一个完整例子**，把四条规则串起来：

```jsonc
// 请求：POST /openapi/v1/order/create
// x-api-key: ak_demo   x-timestamp: 1788400000000   x-nonce: n_demo0001
// body:
{ "productCode": "XP_DEMO", "idempotencyKey": "idem_0001", "qty": 2 }
```

被签对象拍平、编码、加长度前缀、按路径排序后：

```
apiKey=9:s:ak_demo&body.idempotencyKey=11:s:idem_0001&body.productCode=9:s:XP_DEMO&body.qty=3:i:2&method=6:s:POST&nonce=12:s:n_demo0001&path=26:s:/openapi/v1/order/create&sigAlg=23:s:xpin.esim.openapi.kv1&timestamp=15:s:1788400000000
```

再对这串做 `HMAC_SHA256(app_secret, base)` 取十六进制小写，就是 `x-sign`。
用 `app_secret = demo_secret` 时结果是：

```
32f3534d7daf9a53458ba2f403924b64dcbbd831dc7eb90f529f6a963d01242c
```

更多自检向量见 [2.6](#26-自检向量)。

### 2.2 nonce

⚠️ **`x-nonce` 请用真随机值**，不要用时间戳、也不要用 body 的摘要。
`nonce` 的字符集是时间戳的超集 —— 一旦你把 nonce 取成与 `timestamp` 相同的值，
客户端把这两个实参**传反**将无法被签名发现（两边算出的基串完全一样）。
真随机值让这类实参错位在第一次联调就暴露。

**去重范围与记忆期**：按「环境 + 商户 + nonce」去重（同一商户名下的多把 `app_key` 共用
同一个 nonce 命名空间），记忆期为时钟窗口的两倍（10 分钟）。
沙盒与生产互不干扰。随机 nonce 天然满足这条约束，无需自己维护池。

### 2.3 认证检查顺序

多个条件同时不满足时，你只会看到**最先命中**的那一个：

```
1. 四个签名头齐全 ......... 401 AUTH_HEADERS_MISSING
2. nonce 格式 ............. 401 NONCE_INVALID
3. 时间戳窗口 ............. 401 TIMESTAMP_INVALID
4. app_key 存在且启用 ..... 401 APP_KEY_INVALID
5. 凭证未过期 ............. 401 CREDENTIAL_EXPIRED
6. 源 IP 在白名单内 ....... 403 IP_NOT_ALLOWED
7. QPS 未超限 ............. 429 QPS_LIMITED   ← 下单接口跳过这一步
8. 载荷可规范化 ........... 400 CANONICAL_*
9. 签名正确 ............... 401 SIGNATURE_INVALID
10. scope 足够 ............ 403 SCOPE_FORBIDDEN
11. nonce 未被用过 ........ 401 NONCE_REPLAYED
```

第 8 步**刻意与 `SIGNATURE_INVALID` 分开**：它说的是"这份载荷里
有一个签不了的值"（小数、超范围整数、不合法的键名…），不是"你的密钥不对"。看到
`CANONICAL_*` 请查载荷，不要查密钥。

> **两个下单接口不受 QPS 约束**：`/openapi/v1/order/create`（开卡）与
> `/openapi/v1/order/renew`（续订）。它们不占用配额，也不会因为你其他接口打满而被拒 ——
> 成交不该被查询流量挤掉。第 7 步之外的其他十步，下单接口一步都不少。
>
> **其余接口分三份配额，每份都是 `qpsLimit`**：
>
> | 配额 | 接口 |
> |---|---|
> | 独立一份 | `/products/list`、`/products/detail`（两者**共用**这一份） |
> | 独立一份 | `/order/list` |
> | 共用一份 | 其余全部，含 `/order/query`、`/iccid/profile`、`/order/usage`、`/iccid/renew` |
>
> 也就是说：商品目录的全量同步不会影响你的订单查询，反之亦然；两者都影响不到下单。
> 超限时返回的都是 `429 QPS_LIMITED`，**不区分是哪一份** —— 退避重试即可。

两条由此推出、对排障很有用的性质：

- **nonce 只在请求"本可成功"时才被消耗**（第 11 步在最后）。所以错签名重试永远回
  `SIGNATURE_INVALID`、越权重试永远回 `SCOPE_FORBIDDEN`，**不会被自己的重试污染成 `NONCE_REPLAYED`**。
- **参数校验在鉴权之前**。缺必填字段时你拿到的是 `400 xxx required`，不是 `401`。

### 2.4 scope

| scope | 覆盖接口 |
|---|---|
| `esim.read` | `products/list` `products/detail` `cardType` `order/query` `order/list` `iccid/profile` `order/usage` `iccid/renew` |
| `esim.write` | `order/create` `order/renew` `order/refund` |

缺 `esim.write` 调写接口 → `403 SCOPE_FORBIDDEN`（在签名校验**之后**判定，所以它确实是权限问题，不是签名问题）。

### 2.5 签名客户端

两份实现都是完整的，直接抄走即可，除 HTTP 库外不依赖任何第三方包。规则见 2.1。

**Python**

```python
import hashlib, hmac, re, secrets, time, requests

KEY_RE = re.compile(r'^[A-Za-z][A-Za-z0-9_]*$')

def _encode(v, path):
    if v is None:                        return 'n:'
    if isinstance(v, bool):              return 'b:true' if v else 'b:false'   # 必须先于 int 判断
    if isinstance(v, str):               return 's:' + v
    if isinstance(v, int):
        if abs(v) > 2**53 - 1:           raise ValueError(f'{path}: 只能签安全整数')
        return 'i:' + str(v)
    raise ValueError(f'{path}: 不支持的类型 {type(v).__name__}')

def _flatten(v, path, out):
    if isinstance(v, list):
        if not v:                        out.append((path, 'a:')); return
        for i, item in enumerate(v):     _flatten(item, f'{path}.{i}', out)
    elif isinstance(v, dict):
        if not v:                        out.append((path, 'o:')); return
        for k, item in v.items():
            if not KEY_RE.match(k):      raise ValueError(f'键名不合法: {k}')
            _flatten(item, f'{path}.{k}' if path else k, out)
    else:
        out.append((path, _encode(v, path)))

def canonicalize(root):
    leaves = []
    _flatten(root, '', leaves)
    leaves.sort(key=lambda kv: kv[0].encode('utf-8'))
    return '&'.join(f'{p}={len(e.encode("utf-8"))}:{e}' for p, e in leaves)

def _norm_path(p):
    c = re.sub(r'/{2,}', '/', str(p))
    return c[:-1] if len(c) > 1 and c.endswith('/') else c

def sign_inbound(secret, api_key, method, path, timestamp, nonce, body):
    base = canonicalize({
        'sigAlg': 'xpin.esim.openapi.kv1', 'apiKey': api_key,
        'method': str(method).upper(), 'path': _norm_path(path),
        'timestamp': str(timestamp), 'nonce': str(nonce), 'body': body or {},
    })
    return hmac.new(secret.encode(), base.encode('utf-8'), hashlib.sha256).hexdigest()

APP_KEY, APP_SECRET = 'ak_xxx', 'sk_xxx'
BASE = 'https://sandboxapi.xpin.network'

def call(path, body):
    ts, nonce = str(int(time.time() * 1000)), 'n_' + secrets.token_hex(8)
    sign = sign_inbound(APP_SECRET, APP_KEY, 'POST', path, ts, nonce, body)
    # 可以直接用 json=body：签名不要求你控制发出去的字节，
    # requests 怎么序列化都不影响验签。
    return requests.post(BASE + path, json=body, headers={
        'x-api-key': APP_KEY, 'x-timestamp': ts, 'x-nonce': nonce, 'x-sign': sign,
    })
```

**Node.js**（约 50 行手写，无依赖）

```js
const crypto = require('crypto')
const KEY = /^[A-Za-z][A-Za-z0-9_]*$/

function encode(v, path) {
  if (v === null) return 'n:'
  if (typeof v === 'string') return `s:${v}`
  if (typeof v === 'boolean') return `b:${v}`
  if (typeof v === 'number') {
    if (!Number.isSafeInteger(v)) throw new Error(`${path}: 只能签安全整数`)
    return `i:${Object.is(v, -0) ? '0' : String(v)}`
  }
  throw new Error(`${path}: 不支持的类型 ${typeof v}`)
}

function flatten(v, path, out) {
  if (Array.isArray(v)) {
    if (!v.length) return out.push([path, 'a:'])
    return v.forEach((item, i) => flatten(item, `${path}.${i}`, out))
  }
  if (v !== null && typeof v === 'object') {
    const keys = Object.keys(v)
    if (!keys.length) return out.push([path, 'o:'])
    return keys.forEach((k) => {
      if (!KEY.test(k)) throw new Error(`键名不合法: ${k}`)
      flatten(v[k], path ? `${path}.${k}` : k, out)
    })
  }
  out.push([path, encode(v, path)])
}

function canonicalize(root) {
  const leaves = []
  flatten(root, '', leaves)
  leaves.sort((a, b) => Buffer.compare(Buffer.from(a[0]), Buffer.from(b[0])))
  return leaves.map(([p, e]) => `${p}=${Buffer.byteLength(e)}:${e}`).join('&')
}

const normalizePath = (p) => {
  const c = String(p).replace(/\/{2,}/g, '/')
  return c.length > 1 && c.endsWith('/') ? c.slice(0, -1) : c
}

function signInbound(secret, { apiKey, method, path, timestamp, nonce, body }) {
  const base = canonicalize({
    sigAlg: 'xpin.esim.openapi.kv1',
    apiKey,
    method: method.toUpperCase(),
    path: normalizePath(path),
    timestamp: String(timestamp),
    nonce: String(nonce),
    body: body || {}
  })
  return crypto.createHmac('sha256', secret).update(base, 'utf8').digest('hex')
}
```

调用时务必带上 `Content-Type`，否则会在鉴权之前就被 `415 CONTENT_TYPE_MUST_BE_JSON` 拦下：

```js
await fetch(url, {
  method: 'POST',
  headers: { 'content-type': 'application/json', ...signHeaders },
  body: JSON.stringify(body)
})
```

**不需要**保证签名用的字符串与发送用的字符串逐字节相同 —— 签的是数据，不是字节。

其它语言：Go / Java / PHP / .NET 都能在几十行内写完（只用到排序、UTF-8 字节长度、
HMAC-SHA256 三样）。**写完请先跑 [2.6](#26-自检向量) 那张自检向量表**，全过再联调。⚠️ 两个语言特有的注意点：
PHP 要注意 `json_decode` 默认给关联数组、布尔要先于整数判断（PHP/Python 的 `bool` 是
`int` 的子类型，不先判会把 `true` 编码成 `i:1`）；Go 注意 `map` 遍历是随机序，必须显式排序。

### 2.6 自检向量

以下向量用 `app_secret = demo_secret`、`x-api-key = ak_demo`、
`x-timestamp = 1788400000000`、`x-nonce = n_demo0001`、`method = POST`。
**先让这张表全过，再去联调**——签名对不上时，能省掉绝大部分来回。

| # | 用意 | `path` | `body` | 期望 `x-sign` |
|---|---|---|---|---|
| 1 | 最小：空 body | `/openapi/v1/products/list` | `{}` | `3eb1cdb9b139d2de36df73bcae880b72c4b6e408d6c22b8ce9727735b9ee6c75` |
| 2 | 标量三型 + null | `/openapi/v1/order/list` | `{"a":"x","b":1,"c":true,"d":null}` | `78240ca8b1a938974f2ce3ebf9b95b0ea31103c045fa8940a0989b9ad6d18343` |
| 3 | 字符串 1 与整数 1 不同签 | `/openapi/v1/products/list` | `{"v":"1"}` | `31f9ec6d8e2098fbdf3d4c749d8c9bc5296479a0db3907196ba5e5d48e6bab89` |
| 4 | （对照）整数 1 | `/openapi/v1/products/list` | `{"v":1}` | `15a7330e47ebc988b3772f69ca452800125f7b56592372722d5b87d5425a9688` |
| 5 | 嵌套与数组 | `/openapi/v1/order/create` | `{"data":{"orderNo":"X"},"tags":["a","b"]}` | `a0f6a4db62b1e51330fd67cf61c9b3e3cb7d1fce1c10b022c0fd6be35d95e84d` |
| 6 | 空对象与空数组要留痕 | `/openapi/v1/products/list` | `{"o":{},"a":[]}` | `2628471b9872c77f1f84391657cf02e8770e06a686024ac20e94ef6cbb40b629` |
| 7 | （对照）两者都缺省 | `/openapi/v1/products/list` | `{}` | `3eb1cdb9b139d2de36df73bcae880b72c4b6e408d6c22b8ce9727735b9ee6c75` |
| 8 | 值里含 & = : 不与拆字段碰撞 | `/openapi/v1/order/query` | `{"clientOrderNo":"1&b=s:2"}` | `38b88ad966366fe0b0ab78b9040d188a22ad6dc5f9c62210109b7a488d449053` |
| 9 | 非 ASCII 原文参与，不转义 | `/openapi/v1/products/detail` | `{"productCode":"套餐-中文"}` | `a7d86f5a2ea0a0e83d36f6112766360afabb08df407de15543600ef8cf2542d5` |
| 10 | 路径归一：重复斜杠与尾斜杠 | `/openapi/v1//products//detail/` | `{"productCode":"XP_DEMO"}` | `63655a08ff52c78dd0942219cdfab9df4072fcc25d9784f1fcef6e488603f13e` |
| 11 | （对照）已归一的路径 | `/openapi/v1/products/detail` | `{"productCode":"XP_DEMO"}` | `63655a08ff52c78dd0942219cdfab9df4072fcc25d9784f1fcef6e488603f13e` |

> 第 3 与第 4 条、第 6 与第 7 条、第 10 与第 11 条各是一组**对照**：两条签出不同的值才算对。
> 第 3/4 组验的是类型标记（少了它字符串 `"1"` 与整数 `1` 会同签），第 6/7 组验的是空容器
> 留痕，第 10/11 组验的是路径归一（这一组反过来，两条必须**相同**）。

---

## 3. 错误码

| `code` | `msg` | 含义 | 你该做什么 |
|---|---|---|---|
| `200` | `ok` | 成功 | — |
| `400` | `<字段> required` / `<字段> length error [a-b]` / `<字段> Range error` / `<字段> Type error, ...` | 参数校验失败（**自由文本，不是稳定错误码**） | 修参数。判定用 `code===400` + 字段名子串，不要精确匹配 `msg` |
| `400` | `ORDER_RANGE_SELECTOR_REQUIRED` / `ORDER_RANGE_SELECTOR_CONFLICT` / `ORDER_PAGE_TOKEN_INVALID` | 列表选择器/游标非法 | 见 §4.6 |
| `400` | `REQUEST_BODY_NOT_JSON` / `REQUEST_BODY_MUST_BE_OBJECT` | 请求体不是 JSON，或是 JSON 但不是对象 | 检查序列化 |
| `401` | `AUTH_HEADERS_MISSING` | 四个签名头缺任意一个 | 补头 |
| `401` | `NONCE_INVALID` | nonce 格式不符 | 用 `^[A-Za-z0-9._:-]{8,128}$` |
| `401` | `TIMESTAMP_INVALID` | 时间戳非数字或偏差 > 5 分钟 | **对时**；确认用的是毫秒 |
| `401` | `APP_KEY_INVALID` | app_key 不存在/已停用/环境不匹配 | 找平台核对凭证 |
| `401` | `CREDENTIAL_EXPIRED` | 凭证已过期 | 找平台轮换 |
| `401` | `SIGNATURE_INVALID` | 签名不匹配 | 先跑 [2.6](#26-自检向量) 的自检向量，全过再查联调 |
| `400` | `CANONICAL_*` | 载荷里有签不了的值（小数、超范围整数、不合法键名、嵌套过深） | **查载荷，不要查密钥**。错误信息会点出是哪个字段 |
| `401` | `NONCE_REPLAYED` | nonce 已被用过 | 每次请求换新 nonce |
| `403` | `SCOPE_FORBIDDEN` | 凭证 scope 不足 | 找平台加 scope |
| `403` | `IP_NOT_ALLOWED` | 源 IP 不在白名单 | 找平台加 IP |
| `400` | `ORDER_INPUT_INVALID` | 受理类接口的必填字段**传了但内容为空白**（例如 `"   "`）。**缺字段**走上面那条通用参数校验，不是这个码 | 去掉空白后重传 |
| `400` | `EXTERNAL_USER_ID_INVALID` | `externalUserId` 长度不在 1–128 | 改该字段 |
| `403` | `SOURCE_PARTNER_NOT_ALLOWED` | `sourcePartnerCode` 不是你名下的子分销商 | 检查该字段 |
| `404` | `PRODUCT_NOT_FOUND` | 商品不存在或不属于你 | 从 `products/list` 动态取码 |
| `404` | `PRODUCT_NOT_AVAILABLE` | 下单时商品不可售/不存在 | 同上 |
| `404` | `ORDER_NOT_FOUND` | 订单不存在或不属于你 | 检查 `clientOrderNo` |
| `404` | `CARD_NOT_FOUND` | 卡不存在或不属于你 | 检查 `iccid` |
| `404` | — | 路径拼写错误（前缀对、路径名错）。`msg` 不是稳定错误码，不要对它做断言 | 核对路径名 |

> ⚠️ **前缀本身写错时看到的不是上面这一行。** 那种情况请求根本不会进到业务层，你拿到的是
> 一个真实的 `HTTP 404` + 非 JSON 响应体 —— 按 [§3.1](#31-非-200-的响应怎么处理) 的第 1 步处理。
| `409` | `IDEMPOTENCY_KEY_REUSED` | 同 `idempotencyKey` 但请求内容不同 | 换新的幂等键 |
| `409` | `CLIENT_ORDER_NO_REUSED` | `clientOrderNo` 已被别的订单占用 | 换新的业务单号 |
| `409` | `RENEW_NOT_ALLOWED` | 这张卡此刻不可续订 | 先调 `iccid/renew` 看 `reasons` |
| `409` | `ORDER_NOT_REFUNDABLE` | 这一单现在不能退：未履约、已退款、或不具备可退款的计价信息 | 查 `/order/query` 看当前状态。**改参数改不出来**，见 §4.11 |
| `409` | `REFUND_ALREADY_IN_PROGRESS` | 这一单已有一张在途退款单 | 等 `ORDER_REFUND_RESULT` 出结论后再发起，见 §4.11 |
| `402` | `BALANCE_LIMIT_REACHED` | **预付费余额不足**：本单扣完会跌破你的透支额度，整单被拒、一分钱不扣 | 充值。**不要重试**，见下方 ⚠️ |
| `503` | `BALANCE_ACCOUNT_NOT_FOUND` | 余额账户尚未就绪 | 联系平台开通后重试 |
| `503` | `CURRENCY_MISMATCH` | 账户币种与商品币种不一致 | 联系平台核对，**改请求改不出来** |
| `413` | `REQUEST_ENTITY_TOO_LARGE` | 请求体超过 256KB | 拆小 |
| `415` | `CONTENT_TYPE_MUST_BE_JSON` | `Content-Type` 不是 `application/json` | 改头 |
| `429` | `QPS_LIMITED` | 超过该商户每秒请求上限。**下单接口不受此限**，配额口径见 [§2.3](#23-认证检查顺序) | 退避重试 |
| `406` | `[controller error]` | 服务端异常。`msg` 为固定字符串 `[controller error]`，不携带原因 | 见下方 ⚠️ |
| `422` | `PRODUCT_NOT_RENDERABLE` | 该商品对你确实存在且在售，但缺字段、渲染不成你的商品结构 | **联系平台**。改请求改不出来，发布状态也是正常的 |
| `501` | `FEATURE_NOT_IMPLEMENTED` | 你调用了一个本文档未列出的路径 | 只调用 §4 列出的接口 |
| `503` | `OPENAPI_AUTH_UNAVAILABLE` / `PLATFORM_NOT_ENABLED` / `PRODUCT_ADAPTER_UNKNOWN` | 平台侧依赖或配置问题，与你的请求无关 | 退避重试并联系平台 |

> ⚠️ **`402` 请不要重试。**
>
> 你的账户在平台上是**预付费**的：`/order/create` 与 `/order/renew` 在受理时就按结算价扣款，
> 余额扣完会跌破额度时整单被拒、一分钱不扣。充值之前重试是同一个结果。
>
> ⚠️ 充值之前重试是同一个结果。请把 `402` 当成**需要人工介入**的信号（告警 + 暂停该
> 商户的下单队列），不要交给自动重试 —— 对它做无限重试除了浪费两边的资源，不会得到
> 任何不同的答案。
>
> - 收到 `402`：停止对这一单重试，转人工或转充值流程。
> - 收到 `503`：联系平台，处理完成后原样重试即可。
>
> 三个资金相关的码里，只有 `402` 需要你侧采取动作。

> ⚠️ **`406` 的处置：退避重试，连续出现请联系平台。**
>
> `msg` 固定是 `[controller error]`。它不是参数问题（那是 `400`）、不是签名问题（`401`）、
> 也不是余额问题（`402`）。联系平台时请带上 `clientOrderNo` 与大致时间。

**跨租户隔离**：查不到与"不属于你"返回**同一个** `404`，平台不会告诉你"这个单存在但不是你的"。
所以 `404` 不能用来探测他人数据是否存在。

### 3.1 非 200 的响应怎么处理

上面整张错误码表说的都是**平台的业务应答**，它们一律是 `HTTP 200` + JSON 信封，
错误码在 `code` 里 —— 包括 `429 QPS_LIMITED`。

但有一类响应**根本没有进到业务层**，它们带真实的 HTTP 状态码，响应体**可能不是 JSON**：

| 你会看到 | 通常是因为 |
|---|---|
| `404` + 非 JSON | 路径前缀写错（正确前缀是 `/openapi/v1`） |
| `301` | 用了 `http://`。**301 不保留 POST body**，直接用 `https://` |
| `405` | 路径对但 HTTP 方法不对（本 API 全部是 `POST`） |
| `413` / `415` | 请求体过大、或 `Content-Type` 不是 `application/json` |
| `429` | 请求频率异常 —— 退避重试，并检查客户端有没有重试风暴或死循环 |
| `5xx` / 连接中断 | 传输层或服务端异常 |

**判定顺序必须是先状态码、后 JSON**：

```js
const res = await fetch(url, init)

// 第 1 步：先看 HTTP 状态码。不要在这之前 JSON.parse。
if (res.status !== 200) {
  // ⚠️ 这里的响应体可能不是 JSON，解析它会抛异常。
  // 只有 429 与 5xx 值得退避重试；其余（301/404/405/413/415）是配置或请求本身错了，
  // 重试一万次也是同一个结果 —— 重试它们只会把问题变成一场重试风暴。
  if (res.status === 429 || res.status >= 500) throw new RetryableError(`transport ${res.status}`)
  throw new TransportError(`transport ${res.status}`)   // 不可重试：去查 URL、方法、请求头
}

// 第 2 步：HTTP 200 才是平台应答，这时才解析。
const body = await res.json()
if (body.code === 429) throw new RetryableError('QPS_LIMITED')     // 平台层配额，退避重试
if (body.code !== 200) throw new ApiError(body.code, body.msg)
return body.data
```

> ⚠️ **只检查 `body.code` 的客户端，第一次碰到这类响应就会崩在 JSON 解析上**，
> 而报出来的错误（`Unexpected token <`）与真实原因毫无关系。
>
> 反过来也要注意：**不要把所有非 200 都当成可重试**。把 `404`（路径写错）当成
> 可重试错误无限重发，是我们见过最常见的一种自伤。

---

## 4. 接口详解

> 全部路径前缀 `/openapi/v1`。下文的响应样本均为接口的真实返回结构。

### 4.1 商品列表 `/products/list`

拉取**发布给你**的在售商品。这是获取 `productCode` 的**唯一正确来源**。

**请求**

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| `pageNo` | number | 否 | `[1, 100000]`，默认 `1` | 页码。**两端**越界都返回 `400 pageNo Range error`；正常翻页碰不到 |
| `pageSize` | number | 否 | `[1, 100]`，默认 `20` | 每页条数 |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 商品数组，见下 |
| `hasMore` | boolean | 是否还有下一页（翻页终止条件） |
| `pageNo` / `pageSize` | number | **实际生效的**页码 / 每页条数。平台会对越界或小数入参做归一，这两个字段告诉你归一后的结果 —— 它们可能与你发出的值不同 |
| `language` | string | 商品文案的语言，当前恒为 `en`。平台不接受语言协商，请不要在请求里传语言相关参数 |

**商品对象**

> 🔴 **商品的字段结构按贵方的账号配置决定，本节给出的是默认结构。**
> 平台支持多种商品结构，不同结构的**字段名与字段集都不同**（例如流量额度在另一种结构里
> 不叫 `volume`、覆盖范围不叫 `mccList`）。贵方拿到的是哪一种，以**实际调用的返回为准** ——
> 请在联调时先打一次 `/products/list` 核对，不要按本表硬编码。
>
> 同一个账号在 `/products/list`、`/products/detail`、`/order/query`、`/order/list` 上
> 看到的是**同一种结构**，所以核对一次即可。若返回的字段与本表对不上，请联系平台确认
> 贵方配置的是哪种结构并索取对应字段表。

| 字段 | 类型 | 说明 |
|---|---|---|
| `productCode` | string | 对外商品码，下单用它 |
| `productName` | string\|null | 商品名 |
| `tagName` | string\|null | 分组标签，如 `Europe` |
| `volume` | string\|null | 流量额度（字符串数值），单位 MB；不限量为 `"0.000000"` |
| `dataLimited` | string | `Y` 限量 / `N` 不限量 |
| `validity` | number\|null | 套餐天数 |
| `expireDay` | number\|null | 卡有效期天数（从开卡起算） |
| `cardType` | string | 卡型标识。取值由平台目录决定，**请勿枚举或硬编码** |
| `productType` | string\|null | 形态：`DATA_PACKAGE` 流量包 / `DAYPASS` 日包；无法归类时为 `null` |
| `mccList` | string[]\|null | 覆盖国家的 MCC 列表 |
| `attributes` | object\|null | 扩展属性 |
| `settlementPrice` | string | 本结构下为 `"0"`，不携带金额 |
| `retailPrice` | string | 本结构下为 `"0"`，不携带金额 |
| `currency` | string\|null | 币种，如 `USD` |
| `configVersion` | number | 商品配置的**发布代次**：平台每次发布该商品都会递增，**即使内容未变**。可用于判断"需要重新拉取"，不能用于判断"内容确实变了" |

> ⚠️ **除 `productCode` / `dataLimited` / `cardType` / `configVersion` 外，其余字段均可能为 `null`** ——
> 平台不为未知值编造默认值。请按可空解析。

**示例响应**（截取一条）

```json
{
  "data": {
    "list": [
      {
        "productCode": "XP205032025ED92397E2AE461C",
        "productName": "Europe Unlimited 3 Days",
        "tagName": "Europe",
        "volume": "0.000000",
        "dataLimited": "N",
        "validity": 3,
        "expireDay": 90,
        "cardType": "ep1",
        "productType": "DAYPASS",
        "mccList": ["276", "232", "206", "218", "248"],
        "attributes": null,
        "settlementPrice": "0",
        "retailPrice": "0",
        "currency": "USD",
        "configVersion": 1
      }
    ],
    "hasMore": false,
    "pageNo": 1,
    "pageSize": 20,
    "language": "en"
  },
  "code": 200,
  "msg": "ok",
  "t": 1788431706435
}
```

**Node.js**

```js
async function listAllProducts(client) {
  const all = []
  for (let pageNo = 1; ; pageNo++) {
    const page = await client.invoke('/openapi/v1/products/list', { pageNo, pageSize: 100 })
    all.push(...page.list)
    if (!page.hasMore) break
  }
  return all
}
```

**Java**

```java
public List<JsonNode> listAllProducts(XpinEsimClient client) throws Exception {
    List<JsonNode> all = new ArrayList<>();
    for (int pageNo = 1; ; pageNo++) {
        JsonNode page = client.invoke("/openapi/v1/products/list",
                Map.of("pageNo", pageNo, "pageSize", 100));
        page.get("list").forEach(all::add);
        if (!page.get("hasMore").asBoolean()) break;
    }
    return all;
}
```

> ⚠️ **不要硬编码 `productCode`**，也不要拿电信运营商的 SKU 当 `productCode`（电信运营商的 SKU 一律返回 `404 PRODUCT_NOT_FOUND`）。
> 平台可能调整商品目录，对外码会随之变化。

#### 沙盒环境联调用的套餐码

平台方为你配置好账号后，请**向平台方索取本账号在沙盒环境可用于开卡 / 续充的套餐码**；
生产环境可以直接调用 `products/list` 拉取——它返回的就是发布给你的全部在售商品。

**联调选型建议**（与具体码无关，按 `products/list` 返回的属性挑）：

| 你要验证的 | 怎么挑 |
|---|---|
| 开卡 | 任选一个即可 |
| 续充正向流程 | 选 `productType = DATA_PACKAGE` 的。日包（`DAYPASS`）在套餐生效期间整张卡都不可续订，会稳定返回 `DAYPASS_IN_FORCE`，跑不了正向流程 |
| 续充的"不可续"分支 | 反过来选 `DAYPASS`，正好用来验证你对 `DAYPASS_IN_FORCE` 的处理 |
| 跨形态续充规则 | 给 `ep1` 流量包卡续一个 `ep1` 日包，会得到 `DAYPASS_REQUIRES_EMPTY_CARD`；而 `C4` 卡型在同卡型内跨形态续充是放行的 |

> ⚠️ **沙盒环境与生产环境的 `productCode` 完全不同**，切换环境时必须重新拉取目录，不能沿用联调期的码。

---

### 4.2 商品详情 `/products/detail`

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `productCode` | string | **是** | 长度 `[1, 160]` |

**响应 `data`**：字段与 §4.1 的商品对象相同，另加 `language`（商品文案语言，当前恒为 `en`）。

**示例响应**

```json
{
  "data": {
    "language": "en",
    "productCode": "XP205032025ED92397E2AE461C",
    "productName": "Europe Unlimited 3 Days",
    "tagName": "Europe",
    "volume": "0.000000",
    "dataLimited": "N",
    "validity": 3,
    "expireDay": 90,
    "cardType": "ep1",
    "productType": "DAYPASS",
    "mccList": ["276", "232", "206"],
    "attributes": null,
    "settlementPrice": "0",
    "retailPrice": "0",
    "currency": "USD",
    "configVersion": 1
  },
  "code": 200, "msg": "ok", "t": 1788431706435
}
```

**负例**：不存在的码 → `404 PRODUCT_NOT_FOUND`；商品存在且在售但渲染不成贵方的商品结构 → `422 PRODUCT_NOT_RENDERABLE`（联系平台，改请求改不出来）。

**Node.js / Java**

```js
const detail = await client.invoke('/openapi/v1/products/detail', { productCode })
```

```java
JsonNode detail = client.invoke("/openapi/v1/products/detail",
        Map.of("productCode", productCode));
```

---

### 4.3 卡型能力 `/cardType`

查这个商品对应卡型的**静态能力**。

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `productCode` | string | **是** | `[1, 160]` |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `productCode` | string | 回显 |
| `cardType` | string | 卡型 |
| `productType` | string\|null | 形态。无法归类时为 `null` |
| **`capability`** | **object\|null** | **该卡型的能力尚未同步时整块为 `null`**，见下方 🔴 |
| `capability.renewable` | boolean | **这个卡型**原理上是否支持续订 |
| `capability.supportUsageQuery` | boolean | 是否支持用量查询 |
| `capability.maxRenewCount` | number\|null | 卡上并发套餐数上限；`null` = 不限 |
| `capability.timeZone` | string\|null | 该卡型的时区标识。**取值格式不固定，不要解析它**，仅作展示；未知时为 `null` |
| `capability.observedAt` | string | 该能力的同步时刻（ISO-8601） |

**示例响应**

```json
{
  "data": {
    "productCode": "XP205032025ED92397E2AE461C",
    "cardType": "ep1",
    "productType": "DAYPASS",
    "capability": {
      "renewable": true,
      "supportUsageQuery": false,
      "maxRenewCount": 99,
      "timeZone": "UTC+0",
      "observedAt": "2026-09-03T10:22:40.620Z"
    }
  },
  "code": 200, "msg": "ok", "t": 1788431706780
}
```

> 🔴 **先判空再取字段。** 该卡型的能力尚未同步时，`capability` **整块是 `null`**，
> 此时五个子字段一个都没有。这不是异常分支 —— 新卡型上线到首轮能力同步之间就是这个状态。
> 直接写 `data.capability.renewable` 会抛异常。`capability: null` 的含义是"尚不知道"，
> **不要据此推断任何默认值**，按"稍后再查"处理。
>
> 🔴 **`capability.renewable` 不能用来点亮"续订"按钮。** 它说的是"这个卡型原理上能续"，
> 不是"这张卡现在能续"。判据只能是 §4.7 的 `allowed`。同一个 `renewable:true` 的卡型，
> 不同的卡 / 商品组合会给出不同答案。

**负例**：不存在的码 → `404 PRODUCT_NOT_FOUND`。

---

### 4.4 开卡下单 `/order/create`

**需要 `esim.write` scope。**

**请求**

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| `productCode` | string | **是** | `[1, 160]` | 从 `products/list` 取 |
| `idempotencyKey` | string | **是** | `[8, 96]` | 幂等键，见下 |
| `clientOrderNo` | string | 否 | `[1, 100]` | 你自己的业务单号，**在你的账号内必须唯一**（沙盒与生产各自独立）。🔴 **强烈建议必传**，见下 |
| `externalUserId` | string | 否 | `[1, 128]` | 你系统里的终端用户标识 |
| `sourcePartnerCode` | string | 否 | `[1, 64]` | 委托下单时指定你名下的子分销商。**扣款始终发生在你（发起方）的余额上**，子分销商只作归属记录 |

> 🔴 **`clientOrderNo` 虽然可选，但省略它这一单就查不了。** `/order/query` **只能**按
> `clientOrderNo` 查单 —— 平台返回的 `orderNo` **不是查询键**，没有任何接口按它查。
> 省略后只能通过 `/order/list` 的 `externalUserId` 选择器间接定位，而开卡单在履约前
> 连 `iccid` 选择器也捞不到。请始终传一个你自己能复现的值。

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderNo` | string | 平台订单号 |
| `clientOrderNo` | string\|null | 回显。你没传时为 `null` |
| `productCode` | string | 回显 |
| `orderStatus` | string | 新受理时是 `ACCEPTED`。**幂等重放时是这一单此刻的状态**，见下方 🔴 |
| `accepted` | boolean | 恒为 `true` |
| `replayed` | boolean | `true` = 这是一次幂等重放，没有新建单 |

> 🔴 **不要断言 `orderStatus === 'ACCEPTED'`。** 重放时返回的是该订单**当前**的状态 ——
> 而重放正是这个接口最常被走到的路径（超时重发）。那一刻订单很可能已经推进到
> `SUBMITTING` / `PROVIDER_ACCEPTED` / `FULFILLED`，甚至 `FAILED`。
>
> 判断本次调用是否成功，用 `code === 200` 与 `accepted`；判断这一单进行到哪一步，用
> `orderStatus` 本身的取值（§5.1）。把 `ACCEPTED` 当成"受理成功"的判据，会让你在每一次
> 超时重试上误判为失败并再次重试。

> ⚠️ **受理即扣款。** 平台在返回 `ACCEPTED` 的同一个事务里就按该商品的结算价从你的预付费余额扣了钱，不是等履约才扣。后续订单落 `FAILED`（不会出卡）或 `REFUNDED`（退款确认）时自动退回，中间态不动钱。
>
> 余额不足以致本单会跌破额度时，**整单被拒、一分钱不扣**，返回 `402 BALANCE_LIMIT_REACHED`。
> **收到这个码请不要重试**，先完成充值，见 [§3](#3-错误码)。

**示例响应**

```json
{
  "data": {
    "orderNo": "XE1788431707121E04989EE4D",
    "clientOrderNo": "m1lc-1788431700325-1",
    "productCode": "XP205032025ED92397E2AE461C",
    "orderStatus": "ACCEPTED",
    "accepted": true,
    "replayed": false
  },
  "code": 200, "msg": "ok", "t": 1788431707137
}
```

> 🔴 **`ACCEPTED` 只是受理，不是出卡。** 出卡是异步的，绝大多数在数分钟内完成。
> **请不要按固定时长判超时** —— 以 §6 的 `ORDER_OPEN_RESULT` 回调或 §4.5 的轮询结果为准，
> 在拿到终态（`FULFILLED` / `FAILED`）之前一律视为在途。
> 拿到 `orderNo` 后要么轮询 §4.5，要么等 §6 的 `ORDER_OPEN_RESULT` 回调。

#### 幂等语义

| 情况 | 结果 |
|---|---|
| 同 `idempotencyKey` + **相同请求内容** | `200`，返回**同一个** `orderNo`，`replayed: true` |
| 同 `idempotencyKey` + **不同请求内容** | `409 IDEMPOTENCY_KEY_REUSED` |
| 同 `clientOrderNo` + **不同** `idempotencyKey` | `409 CLIENT_ORDER_NO_REUSED` |

所以：**网络超时后原样重发是安全的**，不会重复建单。但换了内容就必须换幂等键。

**负例**：商品不存在或不可售 → `404 PRODUCT_NOT_AVAILABLE`。⚠️ **目录里存在的商品也可能返回这个码**（该商品当前不可下单）—— 收到后请跳过该商品并联系平台，**重新拉目录没有用**，码本来就是从那儿取的；非法 `sourcePartnerCode` → `403 SOURCE_PARTNER_NOT_ALLOWED`。

**Node.js**

```js
async function createOrder(client, { productCode, clientOrderNo, externalUserId }) {
  // clientOrderNo 必须有值：它既是幂等键的来源，也是这一单唯一的查询键。
  // ⚠️ 不要让它落到 undefined —— `open-undefined` 同样满足 [8,96] 的长度校验，
  // 于是第一单之后**每一单都会返回第一单**，而且是 200 + accepted:true，没有任何报错。
  if (!clientOrderNo) throw new Error('clientOrderNo is required')
  // 幂等键与业务单号绑定：重试时必须复用同一个键，否则会建出第二张卡。
  // 这个键要和订单一起持久化，重试时读回同一个值，不要每次现算。
  const idempotencyKey = `open-${clientOrderNo}`
  const body = { productCode, idempotencyKey, clientOrderNo }
  if (externalUserId) body.externalUserId = externalUserId   // 可选字段为空就不要放进去
  const data = await client.invoke('/openapi/v1/order/create', body)
  if (data.replayed) console.log('幂等重放，复用已有订单', data.orderNo)
  // ⚠️ 不要在这里断言 data.orderStatus === 'ACCEPTED'：重放时它是这一单此刻的状态
  return data.orderNo
}
```

**Java**

```java
public String createOrder(XpinEsimClient client, String productCode,
                          String clientOrderNo, String externalUserId) throws Exception {
    // clientOrderNo 必须有值：它既是幂等键的来源，也是这一单唯一的查询键
    Objects.requireNonNull(clientOrderNo, "clientOrderNo is required");
    // 幂等键与业务单号绑定：重试时必须复用同一个键（要与订单一起持久化，不要每次现算）
    String idempotencyKey = "open-" + clientOrderNo;
    // ⚠️ 不要用 Map.of：它对 null 值抛 NPE，而 externalUserId 是可选的
    Map<String, Object> req = new HashMap<>();
    req.put("productCode", productCode);
    req.put("idempotencyKey", idempotencyKey);
    req.put("clientOrderNo", clientOrderNo);
    if (externalUserId != null) req.put("externalUserId", externalUserId);
    JsonNode data = client.invoke("/openapi/v1/order/create", req);
    if (data.get("replayed").asBoolean()) {
        log.info("幂等重放，复用已有订单 {}", data.get("orderNo").asText());
    }
    return data.get("orderNo").asText();
}
```

---

### 4.5 订单查询 `/order/query`

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `clientOrderNo` | string | **是** | `[1, 100]` |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderNo` | string | 平台订单号 |
| `clientOrderNo` | string\|null | 你的业务单号。下单时未传的订单为 `null`（在 `/order/list` 上会出现） |
| `externalUserId` | string\|null | 终端用户标识 |
| `productCode` | string | 商品码 |
| `orderStatus` | string | 见 §5 状态机 |
| `iccid` | string\|null | **开卡单**履约后才有值；**续订单在受理时就已带上目标卡的 ICCID**。判断履约一律看 `orderStatus === 'FULFILLED'`，不要用它是否为空来推断 |
| `profileStatus` | string\|null | **该订单履约那一刻**记录的 profile 状态，**不随卡的后续变化更新**。要取卡的当前状态请用 `/iccid/profile` 的 `profile.status` |
| `fulfilledAt` | string\|null | **平台**判定该单履约成功的时刻，ISO-8601（`2026-03-01T08:15:30.123Z`） |
| `packageStartTime` | string\|null | 套餐可用窗口的**起点** |
| `packageEndTime` | string\|null | 套餐可用窗口的**终点** |
| `latestActivationTime` | string\|null | 套餐的**激活期限**：过了这个时刻仍未激活，套餐作废 |
| `product` | object | 下单时刻的**商品快照**，按贵方配置的商品结构适配器渲染，字段集与 §4.1 的商品对象相同。见下方说明 |

> **这四个时刻怎么用**
>
> - **判断「用户激活了没有」请读 `orderStatus` 与 `/iccid/profile` 的 `packageStatus`**，不要用时间字段去推。`packageStartTime` 有值**不代表**用户已经开始使用 —— 部分套餐的可用窗口在履约时就已确定。
> - `packageStartTime` / `packageEndTime` 是这张套餐**什么时候可用到什么时候**，用于展示有效期与到期提醒。`latestActivationTime` 是另一回事：它是**必须在此之前激活**的期限，不是窗口的一部分。
> - `fulfilledAt` 是**平台**的动作时刻；其余三个来自**供应商**，两类不同源，跨源比较时刻差没有意义。
> - 四个都是**订单**维度。一张卡续订多次就有多个套餐窗口，各属各的订单 —— 要按卡看请用 `/order/list` 的 `iccid` 选择器取全部订单。`/iccid/profile` 上的同名字段取的是当前**主导订单**的值，多套餐时只代表其中一张。
> - 老订单（本次改造之前履约的）这四个字段可能为 `null`。**字段一定在**，值可能没有。

**示例响应**（履约中）

```json
{
  "data": {
    "orderNo": "XE1788431707121E04989EE4D",
    "clientOrderNo": "m1lc-1788431700325-1",
    "externalUserId": "m1lc-user-1788431700325",
    "productCode": "XP205032025ED92397E2AE461C",
    "orderStatus": "PROVIDER_ACCEPTED",
    "iccid": null,
    "profileStatus": null,
    "fulfilledAt": null,
    "packageStartTime": null,
    "packageEndTime": null,
    "latestActivationTime": null,
    "product": {
      "productCode": "XP205032025ED92397E2AE461C",
      "productName": "Europe Unlimited 3 Days",
      "tagName": "Europe",
      "volume": "0.000000", "dataLimited": "N", "validity": 3, "expireDay": 90,
      "cardType": "ep1", "productType": "DAYPASS",
      "mccList": ["276", "232"],
      "attributes": null,
      "settlementPrice": "0", "retailPrice": "0", "currency": "USD",
      "configVersion": 1
    }
  },
  "code": 200, "msg": "ok", "t": 1788431709839
}
```

> ⚠️ **价格字段不用于对账。** 在贵方当前的商品结构下，`product.retailPrice` 与
> `product.settlementPrice` 不携带金额（`/products/list`、`/products/detail`、
> `/order/list` 同理）。结算金额以平台与贵方的商务约定为准。

> ℹ️ **`product` 块的结构随贵方的商品结构适配器变化**，与 §4.1 / §4.2 是同一套规则 ——
> 贵方在商品目录上看到的是哪种结构，订单里就是哪种。
>
> 字段集与 §4.1 的**商品对象**相同。⚠️ 与 `/products/detail` 的差别只有一个：
> 那个接口的 `data` 额外带一个信封字段 `language`，订单里的 `product` 块**没有**它 ——
> 两边共用同一个反序列化类型时请注意这一项。
>
> ⚠️ **老订单可能有字段为 `null`。** `product` 块取自受理那一刻冻结的快照，而快照在 2026-09-15 之前不含 `attributes` / `countryCodeList` / `sourceRawData` 三项——那些订单上由它们派生的字段恒为 `null`（例如非默认结构里的 `coverages`）。这是旧数据的兼容显示，**不表示该商品没有覆盖国家**；`null` 是"未知"，与空数组含义不同。此前受理的订单无法补齐：快照按定义是冻结的，回填会把"当时的事实"改写成"现在的事实"。
>
> 不论渲染结果如何，**订单本身总是查得到**：商品块的字段可能为 `null`，但 `/order/query` 与 `/order/list` 不会因此失败。

**Node.js —— 轮询到出卡**

```js
// 仍在流转中的状态（见 §5.1 状态机）。落到这个集合之外且不是 FULFILLED，就是终态失败
const IN_FLIGHT = new Set(['ACCEPTED', 'SUBMITTING', 'PROVIDER_ACCEPTED', 'RECONCILING', 'RETRY_PENDING'])

// 超时要覆盖履约时延的上沿（见 §5.1），设太短会把正常的慢单误判成失败
async function waitForFulfilled(client, clientOrderNo, { timeoutMs = 20 * 60 * 1000, intervalMs = 5000 } = {}) {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    const d = await client.invoke('/openapi/v1/order/query', { clientOrderNo })
    if (d.orderStatus === 'FULFILLED' && d.iccid) return d
    if (!IN_FLIGHT.has(d.orderStatus)) throw new Error(`订单落入终态但未履约: ${d.orderStatus}`)
    await new Promise((r) => setTimeout(r, intervalMs))
  }
  throw new Error(`${timeoutMs}ms 内未出卡，请联系平台核查`)
}
```

**Java**

```java
/** 仍在流转中的状态（见 §5.1 状态机） */
private static final Set<String> IN_FLIGHT = Set.of(
        "ACCEPTED", "SUBMITTING", "PROVIDER_ACCEPTED", "RECONCILING", "RETRY_PENDING");

/** timeoutMs 要覆盖履约时延的上沿（见 §5.1），设太短会把正常的慢单误判成失败 */
public JsonNode waitForFulfilled(XpinEsimClient client, String clientOrderNo,
                                 long timeoutMs, long intervalMs) throws Exception {
    long deadline = System.currentTimeMillis() + timeoutMs;
    while (System.currentTimeMillis() < deadline) {
        JsonNode d = client.invoke("/openapi/v1/order/query", Map.of("clientOrderNo", clientOrderNo));
        String status = d.get("orderStatus").asText();
        if ("FULFILLED".equals(status) && !d.get("iccid").isNull()) return d;
        if (!IN_FLIGHT.contains(status)) {
            throw new IllegalStateException("订单落入终态但未履约: " + status);
        }
        Thread.sleep(intervalMs);
    }
    throw new IllegalStateException(timeoutMs + "ms 内未出卡，请联系平台核查");
}
```

---

### 4.6 订单列表 `/order/list`

**请求**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `externalUserId` \| `iccid` \| `productCode` | string | **恰好一个** | 范围选择器 |
| `pageSize` | number | 否 | `[1, 100]`，默认 20 |
| `pageToken` | string | 否 | 上一页返回的 `nextPageToken` |

**硬规则**：

| 构造 | 结果 |
|---|---|
| 零个选择器 | `400 ORDER_RANGE_SELECTOR_REQUIRED` |
| 两个及以上选择器 | `400 ORDER_RANGE_SELECTOR_CONFLICT` |
| **只传 `pageToken` 不带选择器** | `400 ORDER_RANGE_SELECTOR_REQUIRED`（不是游标错误！） |
| 选择器 + 伪造/过期游标 | `400 ORDER_PAGE_TOKEN_INVALID` |
| **选择器与签发该游标时不一致**（换了值或换了字段） | `400 ORDER_PAGE_TOKEN_INVALID` —— 游标与选择器绑定，换选择器必须从第一页重开 |

> 🔴 **`pageToken` 不是独立的选择器**，翻页时必须把**原来的选择器一起带上**。游标带签名且**有效期 15 分钟**。

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 订单数组，元素结构同 §4.5 的 `data`。⚠️ 下单时未传 `clientOrderNo` 的订单，该字段为 `null` |
| `hasMore` | boolean | 是否还有下一页 |
| `nextPageToken` | string\|null | 下一页游标；`null` 表示到底 |

> **排序**：按受理时间**倒序**（最新的在前）。游标只能向后翻，不支持反向。

**示例响应**

```json
{
  "data": {
    "list": [
      {
        "orderNo": "XE1788431707121E04989EE4D",
        "clientOrderNo": "m1lc-1788431700325-1",
        "externalUserId": "m1lc-user-1788431700325",
        "productCode": "XP205032025ED92397E2AE461C",
        "orderStatus": "FULFILLED",
        "iccid": "89000000000000000430",
        "profileStatus": null,
        "fulfilledAt": "2026-03-01T08:15:30.123Z",
        "packageStartTime": "2026-03-02T09:20:40.000Z",
        "packageEndTime": "2026-03-09T09:20:40.000Z",
        "latestActivationTime": "2026-06-01T00:00:00.000Z",
        "product": { "...": "同 order/query 的商品快照" }
      }
    ],
    "hasMore": false,
    "nextPageToken": null
  },
  "code": 200, "msg": "ok", "t": 1788431720198
}
```

**Node.js —— 正确的游标翻页**

```js
async function listOrdersByUser(client, externalUserId) {
  const all = []
  let pageToken = null
  do {
    // 选择器必须每页都带上，不能只带 pageToken
    const body = { externalUserId, pageSize: 50 }
    if (pageToken) body.pageToken = pageToken
    const page = await client.invoke('/openapi/v1/order/list', body)
    all.push(...page.list)
    pageToken = page.nextPageToken
  } while (pageToken)
  return all
}
```

**Java**

```java
public List<JsonNode> listOrdersByUser(XpinEsimClient client, String externalUserId) throws Exception {
    List<JsonNode> all = new ArrayList<>();
    String pageToken = null;
    do {
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("externalUserId", externalUserId);   // 选择器必须每页都带
        body.put("pageSize", 50);
        if (pageToken != null) body.put("pageToken", pageToken);
        JsonNode page = client.invoke("/openapi/v1/order/list", body);
        page.get("list").forEach(all::add);
        JsonNode next = page.get("nextPageToken");
        pageToken = next.isNull() ? null : next.asText();
    } while (pageToken != null);
    return all;
}
```

> **按 `iccid` 查是对账的主要抓手**：一张卡的开卡单与全部续订单都挂在同一个 ICCID 下，一次查全。

---

### 4.7 续订资格判定 `/iccid/renew`

**只读接口，不产生订单。** 续订按钮的可用性**只能**由它决定。

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `iccid` | string | **是** | 长度 `[10, 32]` |
| `productCode` | string | **是** | `[1, 160]`，想续的目标商品 |
| `externalUserId` | string | 否 | `[1, 128]`。**传了才会校验用户归属** —— 不传则预检看不出 `EXTERNAL_USER_ID_CONFLICT` |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `allowed` | boolean | **唯一判据** |
| `reasons` | string[] | 不可续的**全部**原因；`allowed:true` 时为 `[]` |
| `iccid` / `productCode` | string | 回显 |

> 这个接口**不用错误码表达"不可续"**：卡不存在也是 `200`，判定在 `allowed:false` + `reasons:["CARD_NOT_FOUND"]`。

**示例响应**

```json
// 可续
{"data":{"allowed":true,"reasons":[],"iccid":"89000000000000000571","productCode":"XP563E64913D5D5E0B8DBC925F"},
 "code":200,"msg":"ok","t":1788495680855}

// 不可续
{"data":{"allowed":false,"reasons":["DAYPASS_IN_FORCE"],"iccid":"89000000000000000435","productCode":"XP205032025ED92397E2AE461C"},
 "code":200,"msg":"ok","t":1788505549000}
```

#### `reasons` 全表（按性质分类，这是接入方最需要的一张表）

| reason | 含义 | 性质 |
|---|---|---|
| `CARD_NOT_FOUND` | 卡不存在或不属于你 | ❌ 终局 |
| `PRODUCT_NOT_FOUND` | 目标商品不存在、未发布给你，**或与这张卡不兼容** | ❌ 终局 |
| `PRODUCT_OFF_SALE` | 目标商品已下架 | ⚠️ 可恢复（换商品或等上架） |
| `CARD_NOT_RENEWABLE` | 上游明确这张卡不可续 | ❌ 终局 |
| `CARD_TYPE_MISMATCH` | 目标商品的卡型与这张卡不同，**或这张卡的卡型尚未同步** | ❌ 终局（续订永不换卡型）；后者稍后重试即可 |
| `RENEW_WINDOW_EXPIRED` | 续订窗口已过 | ❌ 终局 |
| `ORDER_NOT_RENEWABLE` | 当前套餐没有续订截止时间 | ❌ 终局 |
| `EXTERNAL_USER_ID_CONFLICT` | 传入的 `externalUserId` 与卡上绑定的不一致 | ❌ 终局（改传对的） |
| **`RENEWABILITY_PENDING_SYNC`** | 平台还没同步到这张卡的可续订性 | 🔄 **稍后重试** |
| **`RENEW_WINDOW_PENDING_SYNC`** | 续订窗口尚未同步 | 🔄 **稍后重试** |
| **`RENEWABILITY_STALE`** | 同步过但证据已过期 | 🔄 **稍后重试** |
| **`DAYPASS_IN_FORCE`** | 日包在用期间整张卡不可续（用流量包续也不行） | 🔄 日包结束后解除 |
| **`DAYPASS_REQUIRES_EMPTY_CARD`** | 日包要求卡上没有占用中的套餐 | 🔄 卡上现有套餐全部结束后解除 |
| **`SERIAL_RULE_SLOT_OCCUPIED`** | 该卡型要求串行续订，卡上已有占用中的套餐 | 🔄 该套餐结束后解除 |
| `CONCURRENT_ORDER_LIMIT_REACHED` | 卡上并发套餐数已达上限 | 🔄 有套餐到期后解除 |
| `CARD_CAPABILITY_UNKNOWN` | 卡型能力未同步 | 🔄 稍后重试 |
| `CARD_TYPE_RULE_NOT_PUBLISHED` | 该卡型的续订能力尚未开放 | ⚙️ 联系平台 |
| `PACKAGE_TYPE_NOT_CLASSIFIABLE` | 目标商品的套餐形态无法判定 | ⚙️ 联系平台 |

> 🔴 **`allowed: true` 不保证下得了单。** 这是一次**预检**：`allowed: false` 一定下不了单，
> 反过来不成立。受理时还有两道闸门不在预检范围内 —— 该商品当前是否可下单
> （`409 RENEW_NOT_ALLOWED`）与你的预付费余额（`402 BALANCE_LIMIT_REACHED`）。
> **下单侧必须照样处理这两个码**，不要因为预检通过就省掉分支。
>
> 🔴 **不要把"稍后重试"类的 reason 当成永久拒绝。** 上表标 🔄 的七条都会随卡上套餐的
> 变动而解除。资格是**随时在变**的量（卡上每多一张单、每结束一张单都会变），
> 所以请**在用户点击的那一刻实时查**，不要缓存资格结果。
>
> ℹ️ 传 `externalUserId` 时预检才能预测归属冲突。**下单要传的 `externalUserId`，
> 预检也要传** —— 不传的话 `EXTERNAL_USER_ID_CONFLICT` 这一条预检看不出来。

#### 规则按卡型差异化（示例）

> ⚠️ 下表是**形态示例**，不是取值清单。具体哪个卡型适用哪条规则由平台目录决定，
> 请勿硬编码卡型码。

| 卡 | 用同形态商品续 | 用同卡型的另一形态商品续 |
|---|---|---|
| 流量包卡（规则：自由续订） | ✅ 可续 | ✅ 可续 |
| 流量包卡（规则：日包独占） | ✅ 可续 | ❌ `DAYPASS_REQUIRES_EMPTY_CARD` |
| 日包卡（规则：日包独占） | ❌ `DAYPASS_IN_FORCE` | ❌ `DAYPASS_IN_FORCE` |

**Node.js**

```js
// 上表标 🔄 的那一档。其余分档不要塞进来：⚠️ 那一档要换商品，不是原样重试。
const RETRYABLE = new Set([
  'RENEWABILITY_PENDING_SYNC', 'RENEW_WINDOW_PENDING_SYNC', 'RENEWABILITY_STALE',
  'DAYPASS_IN_FORCE', 'DAYPASS_REQUIRES_EMPTY_CARD', 'SERIAL_RULE_SLOT_OCCUPIED',
  'CONCURRENT_ORDER_LIMIT_REACHED', 'CARD_CAPABILITY_UNKNOWN'
])
const TERMINAL = new Set([
  'CARD_NOT_FOUND', 'PRODUCT_NOT_FOUND', 'CARD_NOT_RENEWABLE', 'CARD_TYPE_MISMATCH',
  'RENEW_WINDOW_EXPIRED', 'ORDER_NOT_RENEWABLE', 'EXTERNAL_USER_ID_CONFLICT'
])

// externalUserId 要透传：不传的话预检看不出归属冲突
async function checkRenewable(client, iccid, productCode, externalUserId) {
  const d = await client.invoke('/openapi/v1/iccid/renew', { iccid, productCode, externalUserId })
  if (d.allowed) return { allowed: true, reasons: [], retryable: false }
  // ⚠️ 未知 reason **不要当成终局**。平台会新增 reason，把没见过的一律判死，
  // 会把一个本可恢复的拒绝在你的 UI 上变成永久禁用。按"稍后再试"处理并上报。
  const unknown = d.reasons.filter((r) => !RETRYABLE.has(r) && !TERMINAL.has(r))
  if (unknown.length) report('unknown renew reason', unknown)
  return {
    allowed: false,
    reasons: d.reasons,
    retryable: d.reasons.every((r) => !TERMINAL.has(r))
  }
}
```

**Java**

```java
// 终局的那几条。判据用它而不是 RETRYABLE —— 未知 reason 必须落在"可重试"一侧，
// 把没见过的一律判死会把本可恢复的拒绝在 UI 上变成永久禁用。
private static final Set<String> TERMINAL = Set.of(
        "CARD_NOT_FOUND", "PRODUCT_NOT_FOUND", "CARD_NOT_RENEWABLE", "CARD_TYPE_MISMATCH",
        "RENEW_WINDOW_EXPIRED", "ORDER_NOT_RENEWABLE", "EXTERNAL_USER_ID_CONFLICT");

public record RenewCheck(boolean allowed, List<String> reasons, boolean retryable) {}

// externalUserId 要透传：不传的话预检看不出归属冲突
public RenewCheck checkRenewable(XpinEsimClient client, String iccid, String productCode,
                                 String externalUserId) throws Exception {
    Map<String, Object> req = new HashMap<>();
    req.put("iccid", iccid);
    req.put("productCode", productCode);
    if (externalUserId != null) req.put("externalUserId", externalUserId);   // Map.of 不接受 null
    JsonNode d = client.invoke("/openapi/v1/iccid/renew", req);
    boolean allowed = d.get("allowed").asBoolean();
    List<String> reasons = new ArrayList<>();
    d.get("reasons").forEach(n -> reasons.add(n.asText()));
    boolean retryable = !allowed && reasons.stream().noneMatch(TERMINAL::contains);
    return new RenewCheck(allowed, reasons, retryable);
}
```

---

### 4.8 续订下单 `/order/renew`

**需要 `esim.write` scope。** 给**已有的卡**追加套餐，**不换卡、不换激活码**。

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `iccid` | string | **是** | 长度 `[10, 32]` |
| `productCode` | string | **是** | `[1, 160]`，目标商品 |
| `idempotencyKey` | string | **是** | `[8, 96]` |
| `clientOrderNo` | string | 否 | `[1, 100]`，你的业务单号。**强烈建议必传**，理由同 [§4.4](#44-开卡下单-ordercreate) |
| `externalUserId` | string | 否 | `[1, 128]`，终端用户标识 |

**响应 `data`**：与 §4.4 相同，另含 `iccid`。

**示例响应**

```json
{
  "data": {
    "orderNo": "XE17884956816778866A4A70E",
    "clientOrderNo": "m1n-C4-DATA-rn-1788494789993",
    "iccid": "89000000000000000571",
    "productCode": "XP563E64913D5D5E0B8DBC925F",
    "orderStatus": "ACCEPTED",
    "accepted": true,
    "replayed": false
  },
  "code": 200, "msg": "ok", "t": 1788495681690
}
```

- 续订单有**独立于开卡单的 `orderNo`**，`iccid` 保持不变。
- 幂等语义与开卡完全相同（同键同内容 → `replayed:true`；同键异内容 → `409`）。
  `orderStatus` 同样**不要断言恒为 `ACCEPTED`**，理由见 [§4.4](#44-开卡下单-ordercreate)。
- 不可续时 → `409 RENEW_NOT_ALLOWED`。
- 🔴 **续订与开卡一样是受理即扣款。** 平台在受理的同一个事务里就按该商品的结算价从你的
  预付费余额扣钱；订单落 `FAILED` 或 `REFUNDED` 时自动退回，中间态不动钱。
- 🔴 **余额不足 → `402 BALANCE_LIMIT_REACHED`**，整单被拒、一分钱不扣。**不要重试** ——
  在充值之前每一次重试都是同一个结果。请为续订预留额度，并像开卡一样处理这个码。

**Node.js —— 完整续订流程**

```js
async function renew(client, { iccid, productCode, clientOrderNo, externalUserId }) {
  // 1) 资格必须实时查，不能用缓存 —— 卡上每多一张单、每结束一张单，答案都会变。
  //    externalUserId 要透传，否则预检看不出归属冲突。
  const check = await checkRenewable(client, iccid, productCode, externalUserId)
  if (!check.allowed) {
    const err = new Error(`不可续订: ${check.reasons.join(',')}`)
    err.reasons = check.reasons
    err.retryable = check.retryable
    throw err
  }

  // 2) 下单（受理）。
  //    ⚠️ 预检通过**不等于**受理一定成功：这里仍可能拿到 409 RENEW_NOT_ALLOWED
  //    （商品当前不可下单）或 402 BALANCE_LIMIT_REACHED（余额不足）。两个都要接住，
  //    402 尤其**不要重试** —— 充值之前每次都是同一个结果。
  const body = { iccid, productCode, idempotencyKey: `renew-${clientOrderNo}`, clientOrderNo }
  if (externalUserId) body.externalUserId = externalUserId
  const data = await client.invoke('/openapi/v1/order/renew', body)

  // 3) 等履约。ICCID 必须不变 —— 变了说明发的是新卡，属异常
  const done = await waitForFulfilled(client, clientOrderNo)
  if (done.iccid !== iccid) throw new Error(`续订后 ICCID 变了: ${iccid} -> ${done.iccid}`)
  return { orderNo: data.orderNo, iccid: done.iccid }
}
```

**Java**

```java
public String renew(XpinEsimClient client, String iccid, String productCode,
                    String clientOrderNo, String externalUserId) throws Exception {
    // 1) 资格实时查，不缓存 —— 卡上每多一张单、每结束一张单，答案都会变
    RenewCheck check = checkRenewable(client, iccid, productCode);
    if (!check.allowed()) {
        throw new IllegalStateException("不可续订: " + String.join(",", check.reasons()));
    }
    // 2) 受理。⚠️ 预检通过不等于受理一定成功：这里仍可能拿到 409 RENEW_NOT_ALLOWED
    //    或 402 BALANCE_LIMIT_REACHED（余额不足，**不要重试**）。
    Map<String, Object> req = new HashMap<>();      // 不用 Map.of：它对 null 值抛 NPE
    req.put("iccid", iccid);
    req.put("productCode", productCode);
    req.put("idempotencyKey", "renew-" + clientOrderNo);
    req.put("clientOrderNo", clientOrderNo);
    if (externalUserId != null) req.put("externalUserId", externalUserId);
    JsonNode data = client.invoke("/openapi/v1/order/renew", req);
    // 3) 等履约，ICCID 必须不变
    JsonNode done = waitForFulfilled(client, clientOrderNo, 20 * 60 * 1000L, 5_000);
    if (!iccid.equals(done.get("iccid").asText())) {
        throw new IllegalStateException("续订后 ICCID 变了");
    }
    return data.get("orderNo").asText();
}
```

---

### 4.9 卡档案 `/iccid/profile`

一次拿到这张卡的全部投影：交付物、profile 状态、续订证据、用量。

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `iccid` | string | **是** | 长度 `[10, 32]` |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `iccid` | string | 卡号 |
| `externalUserId` | string\|null | 绑定的终端用户 |
| `cardType` | string\|null | 卡型 |
| `delivery.activationCode` | string\|null | **激活码**，形如 `LPA:1$<smdp>$<matchingId>` |
| `delivery.installTime` | string\|null | 这张 profile 被装到设备上的时刻。同一张卡可被多次安装，该值是**当前报告值**。⚠️ 见下方说明 |
| `delivery.latestActivationTime` | string\|null | 套餐的**激活期限**：过了这个时刻仍未激活，套餐作废 |
| `delivery.imsi` / `delivery.msisdn` | string\|null | IMSI / MSISDN |
| `profile.status` | string | 生命周期：`RELEASED` → `DOWNLOADED` → `INSTALLED` → `ENABLED` ↔ `DISABLED` → `DELETED`；另有 `UNKNOWN`（上游给了平台不认识的值）。共 **7** 个取值 |
| `profile.statusTime` | string | 平台**观测到**该状态的时刻（平台时钟，**不是**上游报告的变更时刻；新卡上是履约投影时刻） |
| `profile.changeVersion` | number | 状态变更版本，**单调递增**，可用于丢序检测 |
| `renewal.renewable` | boolean\|null | 可续订性；`null` = 未同步 |
| `renewal.expirationTime` | string\|null | 续订截止时间。**开卡履约完成即可读到**，不必等后续刷新 |
| `renewal.activePackages` | number | 这张卡当前占用的套餐槽位数（**含尚未履约的在途订单**，见下方警告） |
| `renewal.maxConcurrentPackages` | number\|null | 并发上限。`null` 有两种含义 —— 该卡型未声明上限，或平台尚未同步到该卡型能力，**本接口无法区分**。请勿据此自行判断能否续订，那个问题请用 §4.7 |
| `renewal.evidenceFresh` | boolean | 平台对续订证据新鲜度的判定结果。`false` 表示平台目前不会凭这份证据受理续订 —— 具体原因请用 §4.7 拿 |
| `renewal.evidenceSyncedAt` | string\|null | 证据同步时刻。它是**卡型能力**的刷新时刻，同一卡型的多张卡上取值相同，与 `usage.observedAt` 不是一回事 |
| `usage` | object\|null | 用量；`null` 见 §4.10 |
| `packageStatus` | string\|null | 套餐当前状态，共 12 个取值：`CREATED` / `NOT_ACTIVATED` / `ACTIVATED` / `IN_USE` / `SUSPENDED` / `EXHAUSTED` / `EXPIRED` / `CANCELLED` / `ABANDONED` / `TERMINATED` / `REFUNDED` / `UNKNOWN`；`null` 等同 `CREATED`。**这是判断「用户激活了没有」的字段 —— 已激活的判据是 `ACTIVATED` 或 `IN_USE`** |
| `packageStartTime` | string\|null | 套餐可用窗口的**起点** |
| `packageEndTime` | string\|null | 套餐可用窗口的**终点** |

> **三个时间字段的分工**（名字很像，含义互不相同）
>
> | 字段 | 回答的问题 | 维度 |
> |---|---|---|
> | `delivery.installTime` | 卡什么时候被装到设备上 | 卡 |
> | `packageStartTime` / `packageEndTime` | 这张套餐什么时候可用到什么时候 | 订单 |
> | `delivery.latestActivationTime` | 最晚必须在什么时候之前激活 | 订单 |
>
> ⚠️ **判断「用户激活了没有」请读 `packageStatus`，不要用时间字段去推。** `packageStartTime`
> 有值不代表用户已经开始使用 —— 部分套餐的可用窗口在履约时就已确定。
>
> ⚠️ **下面这 **五** 个字段是订单维度的事实，摆在这个卡维度的响应里**：
> `packageStatus`、`packageStartTime`、`packageEndTime`、`delivery.latestActivationTime`、
> `renewal.expirationTime`。它们全部取自这张卡当前的**主导订单** —— 一张续订过多次、
> 同时挂着多个套餐的卡，这里只代表其中一张。要按订单逐笔看，请用 `/order/query` 与 `/order/list`。
>
> ⚠️ **`usage` 的选单规则与上面五个不同。** 一个按主导订单，一个按最近一次履约。
> 多套餐卡上两者**可以不是同一张单** —— 请比对 `usage.orderNo`，不要默认它们同属一单
> （例如"拿 `packageStatus` 判激活、拿 `usage` 显示剩余"会把 A 单的状态和 B 单的流量拼在一起）。
>
> ⚠️ **`delivery.installTime` 为 `null` 不代表卡没装上。** 这个时刻只来自上游的一次性推送，
> 没有轮询兜底 —— 卡已经是 `ENABLED` 而它仍为 `null` 是正常的稳定状态。
> **不要用它判断卡有没有装上**，那个问题请读 `profile.status`（`INSTALLED` / `ENABLED` /
> `DISABLED` 都表示已在设备上）。

**示例响应**

```json
{
  "data": {
    "iccid": "89000000000000000442",
    "externalUserId": "your-user-0442",
    "cardType": "ep1",
    "delivery": {
      "activationCode": "LPA:1$smdp-a.example.com$7ADBC7BBD32B47438D0BB3F4EXAMPLE",
      "installTime": "2026-09-05T12:58:03.000Z",
      "latestActivationTime": "2026-12-04T12:53:54.000Z",
      "imsi": "460000000000442",
      "msisdn": "850000000000442"
    },
    "profile": { "status": "ENABLED", "statusTime": "2026-09-05T13:00:21.079Z", "changeVersion": 1 },
    "renewal": {
      "renewable": true, "expirationTime": "2026-12-07T12:53:53.000Z",
      "activePackages": 1, "maxConcurrentPackages": 99,
      "evidenceFresh": true, "evidenceSyncedAt": "2026-09-05T13:04:45.813Z"
    },
    "usage": {
      "orderNo": "XE1788612831173B352E275F4",
      "dataTotalBytes": null, "dataUsedBytes": 0, "dataRemainBytes": null,
      "refuelingTotalBytes": null, "qtaConsumptionBytes": null,
      "observedUnit": "MB", "observedAt": "2026-09-05T13:29:24.791Z"
    },
    "packageStatus": "NOT_ACTIVATED",
    "packageStartTime": "2026-09-05T12:53:54.000Z",
    "packageEndTime": "2026-09-08T12:53:54.000Z"
  },
  "code": 200, "msg": "ok", "t": 1788615660098
}
```

> 🔴 **`renewal.activePackages` 不是"已生效的套餐数"**：它把**尚未履约的在途订单**也计算在内，
> 所以一笔刚提交、还没出结果的续订单同样会把它 +1。**不能用它判断"续充成功了没"**。
> 判据永远是**续订单自己的 `orderStatus === 'FULFILLED'`**。

> **不同卡型走不同的 SM-DP+**，激活码里的域名不是固定值。
> 不要对它做任何硬编码假设，整串原样交给终端用户。

**负例**：卡不存在 → `404 CARD_NOT_FOUND`。与 `/order/usage` 是同一个码 ——
ICCID 不存在与不属于你，两者不作区分。

**Node.js / Java**

```js
const p = await client.invoke('/openapi/v1/iccid/profile', { iccid })
const qrPayload = p.delivery.activationCode   // 整串生成二维码给用户
```

```java
JsonNode p = client.invoke("/openapi/v1/iccid/profile", Map.of("iccid", iccid));
String qrPayload = p.get("delivery").get("activationCode").asText();
```

---

### 4.10 用量查询 `/order/usage`

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `iccid` | string | **是** | 长度 `[10, 32]` |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `iccid` | string | 回显 |
| `usage` | object\|null | 用量投影，字段见下 |

**`usage` 对象**

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderNo` | string | 这组数字属于**哪一张订单**。见下面的量程说明 |
| `dataTotalBytes` | number\|null | **当前周期**总量，**整数字节** |
| `dataUsedBytes` | number\|null | **当前周期**已用量，**整数字节** |
| `dataRemainBytes` | number\|null | **当前周期**剩余量，**整数字节** |
| `refuelingTotalBytes` | number\|null | **当日**加油包总量，**整数字节** |
| `qtaConsumptionBytes` | number\|null | **当日**高速已用量，**整数字节** |
| `observedUnit` | string\|null | 这组数字换算前的单位，仅作证据。**它不是上面五个字段的单位** —— 那永远是字节 |
| `observedAt` | string | 这组读数的观测时刻（ISO-8601）。终止态套餐上它不再前进，见下方 ⚠️ |

> **五个数值字段是整数字节，字段名自带单位。** 不要按 `observedUnit` 去二次换算。
>
> ⚠️ **但它们分属两个窗口。** 前三个是「当前周期」，后两个是「当日」：
> `refuelingTotalBytes` **不是** `dataTotalBytes` 的一部分，`qtaConsumptionBytes` 也
> **不是** `dataUsedBytes` 的子集。把它们混在一起算剩余，得到的是一个说不清是哪个窗口的
> 数字。
>
> 日用包不返回周期总量与剩余，所以它最常见的形态是前三个里两个为 `null`、后两个有值。

**示例响应**

```json
{
  "data": {
    "iccid": "89000000000000000430",
    "usage": {
      "orderNo": "XE1788614425833E413E81C10",
      "dataTotalBytes": null,
      "dataUsedBytes": 0,
      "dataRemainBytes": null,
      "refuelingTotalBytes": null,
      "qtaConsumptionBytes": null,
      "observedUnit": "MB",
      "observedAt": "2026-09-05T13:29:24.791Z"
    }
  },
  "code": 200, "msg": "ok", "t": 1788431716876
}
```

> **不限量套餐没有总量。** `products/list` 里 `dataLimited: "N"` 的套餐，
> `dataTotalBytes` 与 `dataRemainBytes` 恒为 `null`，只有 `dataUsedBytes` 是数字。
> 这不是"取数失败"，是这类套餐本来就没有这个概念 —— 不要把它当异常重试。

> 🔴 **这是 `orderNo` 那一单的用量，不是这张卡的合计。** 一张卡上可以同时有多个套餐
> （见 `cardType` 的 `maxConcurrentPackages`），而这里返回的是**最新那一单**的读数。
> `orderNo` 就是说明这件事的字段。只读 `dataRemainBytes` 的调用方在多套餐卡上会得到一个
> **偏小的数** —— 它是最新那一单还剩多少，不是这张卡还能用多少。

> **`usage: null` 的成因，参考 `cardType` 的 `capability.supportUsageQuery`**：
> - `supportUsageQuery: true` 但 `usage` 为 `null` → 还没有样本，稍后再查。
> - `supportUsageQuery: false` → 这个卡型**不做实时测量**。⚠️ 但这**不等于恒为 `null`**：
>   套餐未激活或已用尽时，仍会返回一组由套餐状态推导出的数字（未激活恒 `used=0`，
>   用尽恒 `remain=0`）；真正走量期间才没有新读数。
> - `capability` 整块为 `null`（该卡型尚未同步，见 §4.3）→ 无从判断，按"稍后再查"处理。

> **读数是周期性刷新的**，不是每次调用都实时回源。两次调用之间 `observedAt` 不变，
> 表示期间没有新样本，不是接口失败。
>
> ⚠️ **套餐进入终止状态（过期 / 取消 / 退款等）后，这里保留的是最后一次读数，
> `observedAt` 不再前进。** 那就是最终值 —— 不要继续轮询等一个不会到来的更新。

**负例**：卡不存在 → `404 CARD_NOT_FOUND`。

`/iccid/profile` 的 `usage` 与本接口返回的是**同一个投影**，字段与取值完全一致，
按哪条取都行。

---

### 4.11 退款申请 `/order/refund`

对一张**已履约**的订单发起退款申请。scope `esim.write`。

⚠️ **这个接口只受理，不动钱。** 它落一张退款单并返回 `REQUESTED`；金额要等平台确认这一单确实已退之后才退回你的预付费余额。确认由平台的定期核实流程做出，**不是同步的**，端到端时延取决于上游何时把这一单标记为已退 —— **请不要按固定时限设置超时告警**。

结论通过 `ORDER_REFUND_RESULT` 回调推给你（§6.1）。

> ⚠️ **轮询 `/order/query` 只能看到"确认"这一种结论。** 确认会把 `orderStatus` 改成
> `REFUNDED`；**拒绝不改变订单状态**（订单留在 `FULFILLED`），轮询看不出来。
> 要同时拿到两种结论，只能订阅 `ORDER_REFUND_RESULT`。
>
> 回调路同样需要超时兜底：一张 `REQUESTED` 的退款单在拿到结论前不会有任何事件，
> 超过你设定的时限仍无结论，请联系平台 —— 本接口**没有撤销退款申请的入口**。

**入参**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `clientOrderNo` | string | 是 | 1–100 字符。你的业务单号，精确匹配。**不引入第二种订单号** |
| `idempotencyKey` | string | 是 | 8–96 字符。与 `/order/create` 同族的重放保护（参与哈希的内容不同：这里是 `clientOrderNo` + `reason`） |
| `reason` | string | 否 | 1–512 字符。退款原因，落在平台退款单上供人工查阅 |

> ⚠️ **你传的 `reason` 不会回到 `ORDER_REFUND_RESULT` 的 `reason` 字段。** 那个字段在
> `CONFIRMED` 时恒为 `null`，在 `REJECTED` 时是**平台的拒绝原因**（也可能为 `null`）。
> 不要用它传递你自己的关联信息 —— 关联请用 `refundNo` 或 `idempotencyKey`。

**响应**

| 字段 | 类型 | 说明 |
|---|---|---|
| `refundNo` | string | 平台退款单号，后续沟通用它 |
| `orderNo` | string | 被退的平台订单号 |
| `clientOrderNo` | string | 回显你传的业务单号 |
| `refundStatus` | string | 新受理时是 `"REQUESTED"`。**重放时是那张退款单此刻的状态**，可能已是 `CONFIRMED` / `REJECTED` —— 那不是错误，正是你要的幂等结果 |
| `amount` | string | 退款金额，**恒等于该单的结算金额**（本期只支持全额退） |
| `currency` | string | 取自受理那一刻冻结的商品快照，本期恒为 `USD` |
| `accepted` | boolean | 恒为 `true` |
| `replayed` | boolean | `true` = 这是一次重放，没有新建退款单 |

**拒绝**

| HTTP | `msg` | 含义与处置 |
|---|---|---|
| 404 | `ORDER_NOT_FOUND` | 你名下没有这个 `clientOrderNo` |
| 409 | `ORDER_NOT_REFUNDABLE` | 这一单现在不能退：未履约（钱本来就没扣或已经退了）、已退款、或不具备可退款的计价信息。查 `/order/query` 看当前状态 —— **改参数改不出来** |
| 409 | `REFUND_ALREADY_IN_PROGRESS` | 这一单已有一张在途退款单。等它出结论（`ORDER_REFUND_RESULT`）之后才能再发起 |
| 409 | `IDEMPOTENCY_KEY_REUSED` | 同一个 `idempotencyKey` 之前用过、但内容不同（`reason` 也算内容） |
| 400 | `ORDER_INPUT_INVALID` | `clientOrderNo` 或 `idempotencyKey` 传了但是空白串。**缺字段**走 §3 的通用参数校验（`400 <字段> required`），不是这个码 |

**幂等语义**：同一个 `idempotencyKey` + 同样的内容重发 → 返回同一张退款单 + `replayed: true`，**不会退两次钱**。一张订单可以有多张历史退款单（第一次被拒之后可以再申请），但**同时只能有一张在途**。

⚠️ **没有"撤销退款申请"的接口。** 一张 `REQUESTED` 的单只有两个出口：平台确认（`CONFIRMED`）或平台拒绝（`REJECTED`）。改主意了请联系平台。

## 5. 业务流程与状态机

### 5.1 订单状态机

```
ACCEPTED ──► SUBMITTING ──► PROVIDER_ACCEPTED ──► FULFILLED ──► REFUNDED
                 │  ▲              │
                 │  │              └──────────────► FAILED
                 │  │                                 ▲
                 ├──┴─► RETRY_PENDING ────────────────┤
                 └────► RECONCILING ──────────────────┘
                        （这两个到期后回到 SUBMITTING 再试；次数耗尽则 FAILED）
```

| 状态 | 含义 | 你该做什么 |
|---|---|---|
| `ACCEPTED` | 平台已受理，尚未提交电信运营商 | 继续等 |
| `SUBMITTING` | 正在提交电信运营商 | 继续等 |
| `PROVIDER_ACCEPTED` | 上游已受理，等履约回调 | 继续等；**长时间不动不是你的参数问题**，请联系平台核查 |
| `RECONCILING` | 提交/对账阶段异常，正在核对 | 继续等，超时联系平台 |
| `RETRY_PENDING` | 等待重试 | 继续等 |
| **`FULFILLED`** | **已履约，`iccid` 可用** | 取激活码交付用户 |
| **`FAILED`** | **这一单没有出卡** | **停止轮询**。需要出卡就重新下一单（用**新的** `idempotencyKey` 与 `clientOrderNo`）。原因需联系平台核查 |
| **`REFUNDED`** | **这一单已退款**（你提交的退款申请被确认，见 §4.11） | 停止轮询。卡已废弃，激活码不再可用 |

> 🔴 **轮询必须同时判 `FAILED`。** 只等 `FULFILLED` 的循环会在失败单上一直转到超时。
> `FULFILLED` / `FAILED` / `REFUNDED` 都按终态处理，收到任一个就停止轮询。
>
> ⚠️ **但请让你的状态写入保持可重入。** 极少数情况下，平台在补齐更晚到达的履约证据后
> 会把一张 `FAILED` 或 `REFUNDED` 的订单改回 `FULFILLED`，并补发一条 `ORDER_OPEN_RESULT`。
> **不要在收到 `FAILED` 时删除本地订单记录或让它变得不可更新** —— 否则你会收到一条
> 无处安放的事件，同时有一张已交付却无人认领的卡。
>
> ⚠️ **`REFUNDED` 只出现在你自己提交过退款申请的订单上**（§4.11），不使用退款接口的接入方
> 不会看到它。按契约你的状态解析**不应因遇到未知取值而报错** —— 请把它当作与 `FAILED`
> 同族的终态处理。

**时延**：开卡与续订的 `ACCEPTED → FULFILLED` 绝大多数在数分钟内完成，但**不承诺固定时长**，
请以终态为准、不要按时长判超时。长时间既未 `FULFILLED` 也未 `FAILED` 时请与平台核查，
**不要靠重复下单绕过**（会建出多张卡）。

### 5.2 开卡完整流程

```
1. products/list           取 productCode
2. order/create            → ACCEPTED + orderNo
3. 二选一：
   a. 轮询 order/query 到终态（FULFILLED 或 FAILED）
   b. 等 ORDER_OPEN_RESULT 回调（推荐，见 §6；失败的单不发事件，所以回调路仍需超时兜底）
4. iccid/profile           取 delivery.activationCode
5. 把整串激活码生成二维码交付终端用户
```

> 💡 **走回调路可以省掉第 4 步** —— `ORDER_OPEN_RESULT` 事件里已经带了 `activationCode`，
> 与 `/iccid/profile` 同名同值。
>
> ⚠️ 但**两条路都要判空**：`activationCode` 与 `iccid` 都可能是 `null`（见 §6.1）。
> 事件里为空时改用 `/iccid/profile` 取；仍为空请联系平台，**不要重新下单**。

### 5.3 续订完整流程

```
1. iccid/renew             实时查资格（不要缓存！）
2. allowed === true 才继续，否则按 reasons 分类提示用户
3. order/renew             → ACCEPTED + 新的 orderNo（iccid 不变）
4. 轮询 order/query 到 FULFILLED，或等 ORDER_RENEW_RESULT 回调
5. 断言 iccid 未变
```

> ⚠️ **续订资格有时间窗**：新卡建成后，平台需要先同步上游的续订证据；该证据也有新鲜期。
> 期间查资格会拿到 §4.7 标 🔄 的 reason（`RENEWABILITY_PENDING_SYNC` / `RENEW_WINDOW_PENDING_SYNC` /
> `CARD_CAPABILITY_UNKNOWN`），按"稍后再试"提示即可。资格还会随卡上套餐的变动实时改变，
> 所以**必须在用户点击的那一刻查**，不要缓存。

### 5.4 退款完整流程

```
1. order/query             确认这一单是 FULFILLED（只有已履约的单能退）
2. order/refund            → REQUESTED + refundNo（此时钱还没退）
3. 二选一：
   a. 等 ORDER_REFUND_RESULT 回调（推荐，见 §6；确认与拒绝都会发）
   b. 轮询 order/query 看 orderStatus 是否变 REFUNDED
      注意：拒绝不改变订单状态，所以这条路只能识别"已确认"，需配合超时
4. refundStatus === "CONFIRMED" → 金额已退回预付费余额，卡已废弃
   refundStatus === "REJECTED"  → 不退，卡仍有效，reason 是原因
```

> ⚠️ **确认是异步的，不承诺时限。** 平台要先确认这一单确实已退，才把钱退回你的余额；
> 这个确认由平台的定期核实流程做出，不是你调接口的那一刻，端到端时延取决于上游何时
> 把这一单标记为已退。所以 `/order/refund` 返回 `REQUESTED` 之后，
> **不要立刻去查余额并断言已经涨回来**，也不要按固定时限设置超时告警。
>
> ⚠️ **回调路同样需要超时兜底**：一张 `REQUESTED` 的退款单在拿到结论前不会有任何事件，
> 而这两种结论都可能迟迟不发生。超过你设定的时限仍无结论请联系平台 ——
> **没有撤销退款申请的接口**。
>
> ⚠️ **两种结论的可见性不同，选轮询时请注意。** 确认（`CONFIRMED`）会把 `orderStatus` 变为
> `REFUNDED`；拒绝（`REJECTED`）**不改变订单状态**。因此轮询 `/order/query` 只能识别"已确认"
> 这一种结论 —— **推荐走 `ORDER_REFUND_RESULT` 回调**，它对两种结论都会送达；确需轮询的请设置
> 超时，超时后联系平台确认结论。

---

## 6. 回调通知

平台在事件发生时主动 `POST` 到你的回调地址。**这项能力需要先向平台方申请开通**，
申请时提供你的回调接收地址，并说明需要订阅哪些事件（不指定则默认订阅全部）。

> **判定发生在事件产生的那一刻**：开通之前**已经产生**的事件不会补发；开通之后产生的
> 事件都会发 —— 包括开通前下的单在开通后才履约的那些，以及开通前就存在的老卡在开通后
> 发生的状态变更。

### 6.1 事件类型

| `eventType` | 何时发 | `data` 字段 |
|---|---|---|
| `ORDER_OPEN_RESULT` | 开卡订单落 `FULFILLED` 时，每单一次 | `orderNo` `clientOrderNo` `idempotencyKey` `iccid` `imsi` `msisdn` `activationCode` `productCode` `orderStatus` |
| `ORDER_RENEW_RESULT` | 续订单落 `FULFILLED` 时 | **同上**（`activationCode` 是该卡**原有**的激活码，不变） |
| `STATUS_CHANGED` | 卡的 profile 状态真的变化时 | `orderNo` `iccid` `entityType`(恒 `PROFILE`) `previousStatus` `currentStatus` `changedAt` `changeVersion` |
| `ORDER_REFUND_RESULT` | 你提交的退款申请**有了结论**时（确认或拒绝），每张退款单一次 | `orderNo` `clientOrderNo` `refundNo` `idempotencyKey` `refundStatus` `orderStatus` `amount` `currency` `reason` |

> 🔴 **`data` 里的这几个字段可能是 `null`，请按可空解析**：`ORDER_OPEN_RESULT` /
> `ORDER_RENEW_RESULT` 的 `iccid`、`imsi`、`msisdn`、`activationCode`。
> 推送模式开卡的订单可以在没有 `iccid` 的情况下落 `FULFILLED`。
> 直接 `qrcode(event.data.activationCode)` 会在这条真实路径上抛异常 ——
> 为 `null` 时请改用 §4.9 的 `/iccid/profile` 取；仍为空请联系平台，**不要重新下单**。

> ⚠️ **`STATUS_CHANGED` 的 `orderNo` 是该 ICCID 当前最新的订单号，用来定位这张卡，
> 不代表引发本次变更的订单。** profile 变更是**卡级**事件，本来就不存在"引发它的那一单"；
> 续订之后这张卡的状态变更会挂在续订单的 `orderNo` 上。**请以 `iccid` 为关联主键。**

⚠️ **`ORDER_REFUND_RESULT` 对两种结论都会送达，请两种都处理。** `refundStatus: "REJECTED"` 是退款流程一个**成功完成的结论**（平台判定不退），不是投递失败：收到它即可停止等待。`reason` 里是原因，但**它可能为 `null`**（平台未附原因）—— 不要直接 `reason.trim()`，为空时提示用户联系客服即可。

| `refundStatus` | `orderStatus` | `reason` | 含义 |
|---|---|---|---|
| `"CONFIRMED"` | `"REFUNDED"` | `null` | 退款已确认，金额已退回你的预付费余额 |
| `"REJECTED"` | `"FULFILLED"` | 拒绝原因（`string\|null`） | 平台判定不退，订单状态**没有变化**，卡仍然有效 |

⚠️ **去重请按 `eventId`。** 一张退款单只会收到一次 `ORDER_REFUND_RESULT`（`CONFIRMED` 与 `REJECTED` 都是终态），但同一张**订单**可以收到多次——第一次被拒之后可以再次申请，那是另一张退款单、另一个 `refundNo`，`eventId` 也不同。按 `orderNo` 去重会把后续退款的结论一并滤掉。

### 6.2 请求形状

`POST` 到你的 `callback_url`，`Content-Type: application/json`，**10 秒超时**。

请求头：

| 头 | 值 |
|---|---|
| `x-event-id` | 与 body 的 `eventId` **完全相同**（可只读头做幂等） |
| `x-sign` | 签名，HMAC-SHA256 十六进制小写。**签名只在这个头里，body 里没有 `sign` 字段** |

信封字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | **string** | 恒为 `"0000"`（**字符串**，不是数字） |
| `msg` | string | `success` |
| `eventId` | string | `sha256("<orderNo>:<eventType>:<version>")` 的前 48 位十六进制。**确定性、重投不变** |
| `eventType` | string | 见 §6.1 |
| `data` | object | 事件数据 |
| `timestamp` | string | 本次投递尝试的 ISO-8601 时刻，**重投会变** |
| `nonce` | string | 每次投递重新生成的 UUID，**重投会变** |
| `businessType` | string | 恒为 `ESIM` |

**示例报文**（四类各一）

```json
// ORDER_OPEN_RESULT
{"msg":"success","code":"0000","data":{"imsi":"460000000000433","iccid":"89000000000000000433","msisdn":"850000000000433","orderNo":"XE1788489809764F48C390102","orderStatus":"FULFILLED","productCode":"XP205032025ED92397E2AE461C","clientOrderNo":"m1whA-1788489808349","activationCode":"LPA:1$smdp-a.example.com$7ADBC7BBD32B47438D0BB3F4EXAMPLE","idempotencyKey":"m1whA-idem-1788489808349"},"eventId":"145b1ba79790575ae1fa22a5b172fb2fc6687a27de71d584","eventType":"ORDER_OPEN_RESULT","timestamp":"2026-09-04T02:43:37.180Z","nonce":"b2106d10-f164-4c89-84d9-4c7887dc7d03","businessType":"ESIM"}
```

```json
// ORDER_RENEW_RESULT
{"msg":"success","code":"0000","data":{"imsi":"460000000000574","iccid":"89000000000000000574","msisdn":"850000000000574","orderNo":"XE1788505564505CF4DE8D310","orderStatus":"FULFILLED","productCode":"XP563E64913D5D5E0B8DBC925F","clientOrderNo":"m1n-C4-DATA-rn-1788504527118","activationCode":"LPA:1$smdp-b.example.com$7ADBC7BBD32B47438D0BB3F4EXAMPLE","idempotencyKey":"m1n-C4-DATA-rnidem-1788504527118"},"eventId":"881c95a391b04045a10b5f91ebecb3512bbb8b99c70584f9","eventType":"ORDER_RENEW_RESULT","timestamp":"2026-09-04T07:06:22.502Z","nonce":"66677339-0820-4926-86aa-53fe1c5bc248","businessType":"ESIM"}
```

```json
// STATUS_CHANGED
{"msg":"success","code":"0000","data":{"iccid":"89000000000000000433","orderNo":"XE1788489809764F48C390102","changedAt":"2026-09-04T02:49:39.026Z","entityType":"PROFILE","changeVersion":1,"currentStatus":"ENABLED","previousStatus":"UNKNOWN"},"eventId":"56e045be3d6c736f968e19c690dd01fc6be016055fcc32bf","eventType":"STATUS_CHANGED","timestamp":"2026-09-04T02:49:43.548Z","nonce":"e154774a-636d-4ee3-b2a4-4b007db57a97","businessType":"ESIM"}
```

```json
// ORDER_REFUND_RESULT —— 确认（reason 为 null）
{"msg":"success","code":"0000","data":{"orderNo":"XE1788489809764F48C390102","clientOrderNo":"m1whA-1788489808349","refundNo":"XR1788692301447A31B7C0D28","idempotencyKey":"m1whA-refund-1788692300112","refundStatus":"CONFIRMED","orderStatus":"REFUNDED","amount":"3.20","currency":"USD","reason":null},"eventId":"991536ed1ff784d8d2c026c24f0e1f2ebc20f1b5fd93ba74","eventType":"ORDER_REFUND_RESULT","timestamp":"2026-09-13T09:21:44.006Z","nonce":"5f2a1c93-70e8-4a1d-9b6e-2c81d4f0a7b3","businessType":"ESIM"}
```

```json
// ORDER_REFUND_RESULT —— 拒绝（orderStatus 不变，reason 是原因）
{"msg":"success","code":"0000","data":{"orderNo":"XE1788505564505CF4DE8D310","clientOrderNo":"m1n-C4-DATA-rn-1788504527118","refundNo":"XR17886931125503F9E82A146","idempotencyKey":"m1n-refund-1788693111204","refundStatus":"REJECTED","orderStatus":"FULFILLED","amount":"5.00","currency":"USD","reason":"upstream declined: package already consumed"},"eventId":"a665fa8d571364599d7e4f168e36d3fa268d7da998cfb946","eventType":"ORDER_REFUND_RESULT","timestamp":"2026-09-13T09:38:02.771Z","nonce":"c0913ba7-1d55-4e0a-8f37-6ab2e59d4c10","businessType":"ESIM"}
```

> 🔴 **不要假设键序。** 当前键序为 `msg, code, data, eventId, eventType, timestamp, nonce, businessType`，
> 但它**不是契约的一部分**，给 `data` 加一个短字段就可能整体重排。
>
> 键序与验签**无关**，随便它怎么排。

### 6.3 验签

与入站**同一套规则**（2.1 的四条规则），所以你只需实现一次。
两点要注意：

1. **`sign` 不在 body 里**，只在 `x-sign` 头；
2. 被签的不是报文字节，而是从**解析后的数据**拼出的规范串。键序、空白、转义、用哪个
   JSON 库，全都不影响结果。你**不需要**拿到原始字节。

验签 = 解析 body → 加上 `"sigAlg": "xpin.esim.webhook.kv1"` → 按四条规则算基串 →
HMAC-SHA256 → 与 `x-sign` 常数时间比对。

```js
// canonicalize() 与 2.5 里那份逐字相同 —— 入站出站共用，写一次
function verifyKv1(parsedBody, signHeader, appSecret) {
  const base = canonicalize({ ...parsedBody, sigAlg: 'xpin.esim.webhook.kv1' })
  const expected = crypto.createHmac('sha256', appSecret).update(base, 'utf8').digest('hex')
  // ⚠️ 必须先转成 Buffer 再比长度：字符串的 .length 是 UTF-16 码元数，而 Buffer 按
  // UTF-8 编码，含非 ASCII 字符时两者不等价，timingSafeEqual 会**抛异常**而不是返回
  // false。x-sign 是公网上完全由调用方控制的头，抛出去会让你的端点挂住或 500。
  const a = Buffer.from(String(signHeader ?? ''), 'utf8')
  const b = Buffer.from(expected, 'utf8')
  if (a.length !== b.length) return false
  return crypto.timingSafeEqual(a, b)
}
```

```python
import hashlib, hmac
# canonicalize() 与 2.5 那份逐字相同 —— 入站出站共用，写一次
def verify_kv1(parsed_body, sign_header, app_secret):
    base = canonicalize({**parsed_body, 'sigAlg': 'xpin.esim.webhook.kv1'})
    expected = hmac.new(app_secret.encode(), base.encode('utf-8'), hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, sign_header or '')
```

`express.json()` / `@RequestBody Map` / `request.json` 都可以放心用 —— 拿解析后的对象就够了。

**接收端骨架（Express）**

```js
const express = require('express')
const app = express()

// 可以直接用 express.json()：验签不需要原始字节。
app.use('/xpin/webhook', express.json({ limit: '1mb' }))

app.post('/xpin/webhook', async (req, res) => {
  const eventId = req.get('x-event-id')
  if (!verifyKv1(req.body, req.get('x-sign'), APP_SECRET)) {
    // 验不过就不要回 0000 —— 回了我们会认为已送达，这条事件不再重投。
    return res.json({ code: '9999', msg: 'bad signature' })
  }
  // 幂等：同一 eventId 只处理一次。重投是契约，不是异常。
  if (!(await markProcessedIfNew(eventId))) return res.json({ code: '0000', msg: 'duplicate' })
  await handle(req.body)
  res.json({ code: '0000', msg: 'success' })
})
```

> **两个提示**
> 1. **ACK 判据是响应体 `code === "0000"` 加 HTTP 2xx，两者都要**。任何其它组合都按失败
>    处理并重投（间隔与上限见 [6.4](#64-ack-与重投)）。所以处理失败时**不要**回 `0000`。
> 2. **幂等存储必须跨进程生效**（唯一索引表或 Redis `SETNX`）。我们的重投可能落到你集群
>    里的另一个实例上，进程内的 `Set` 挡不住。`x-event-id` 头与 body 的 `eventId` 恒等，
>    用哪个都行 —— 但要注意**头不在签名覆盖范围内**，所以以 body 里的 `eventId` 为准更稳。

---

### 6.4 ACK 与重投

| 项 | 值 |
|---|---|
| 送达判定 | **HTTP 2xx 且响应体 `code === "0000"`（字符串）** |
| 失败情形 | 非 2xx、响应超时、`code` 不是 `"0000"`、响应体过大 |
| 重投间隔 | **约 5 秒** |
| 最多重投 | 计划内多次（约 2 小时内），超出后不再重投 |
| 重投时 | `eventId` 与 `data` **不变**；`timestamp` / `nonce` / `sign` **每次重新生成** |
| ACK 之后 | **立即停止重投** |

> 💡 **长时间没收到预期事件时**，用 `/order/list`（§4.6）按 `externalUserId` 或 `iccid`
> 对账补齐 —— 那是事件之外唯一的兜底入口。

> 🔴 **幂等键只能是 `eventId`（或头里的 `x-event-id`）**，不能用 `sign`、`nonce` 或 `timestamp`——它们每次投递都会变。
>
> 🔴 **`"0000"` 是字符串，同步接口的 `200` 是数字，两套码本。** 回 `{"code":200}` 或 `{"code":0}`
> 都会被判为失败，导致同一事件在约两小时内被反复重投。请务必按字符串 `"0000"` 应答。

> 🔴 **投递不保证顺序。** 一条重投过的事件会排在后面产生的新事件之后到达。
> 同一张卡的 `STATUS_CHANGED` 带**单调递增**的 `data.changeVersion` ——
> 请丢弃 `changeVersion` 不大于你已处理值的事件，否则乱序会让本地状态永久停在旧值。
> `ORDER_OPEN_RESULT` / `ORDER_RENEW_RESULT` 与 `STATUS_CHANGED` 之间同样不保证先后。

> 🔴 **密钥轮换会中断在途事件的投递。** 平台用你凭证**当前**的 `app_secret` 签名，
> 不回落旧密钥；轮换之后，**已在投递队列里的事件不会再发出**（你不会收到一条验不过的
> 通知，而是**什么都收不到**），并且**不会自行恢复**，未送达的事件在有限时间后被放弃。
>
> **请与平台方约定轮换时间窗**，选在没有事件在途的时段。轮换后请核对这段时间内是否有
> 订单只在 `/order/query` 里变成了 `FULFILLED` 却没有收到对应事件 ——
> 用 `/order/list` 按 `externalUserId` 或 `iccid` 对账补齐。

---

