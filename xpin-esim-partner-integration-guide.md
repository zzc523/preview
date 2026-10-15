# XPIN eSIM 分销商 OpenAPI 接入说明

> **文档版本：beta**（2026-10-15）
>
> 面向下游分销商的接入文档，覆盖认证、目录、开卡、幂等、履约、ICCID、续订、退款、余额、回调十条链路。
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
   - [4.8 可续订商品 `/iccid/renewProducts`](#48-可续订商品-iccidrenewproducts)
   - [4.9 续订下单 `/order/renew`](#49-续订下单-orderrenew)
   - [4.10 卡档案 `/iccid/profile`](#410-卡档案-iccidprofile)
   - [4.11 用量查询 `/order/usage`](#411-用量查询-orderusage)
   - [4.12 退款申请 `/order/refund`](#412-退款申请-orderrefund)
   - [4.13 账户余额与充值记录 `/account/balance`](#413-账户余额与充值记录-accountbalance)
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
7. [沙盒专用：状态模拟 `/sandbox/card/status`](#7-沙盒专用状态模拟-sandboxcardstatus)
   - [7.1 可以设成哪些状态](#71-可以设成哪些状态)
   - [7.2 调用与返回](#72-调用与返回)
   - [7.3 拒绝](#73-拒绝)
   - [7.4 用它验证回调链路](#74-用它验证回调链路)

---

## 1. 基础约定

| 项 | 值 |
|---|---|
| Base URL（沙盒环境） | `https://sandboxapi.xpin.network` |
| Base URL（生产环境） | 请向平台方获取 |
| 协议 | 全部 `POST`，`Content-Type: application/json` |
| 字符集 | UTF-8 |
| 字符串参数 | 首尾空白先被去掉，再按约束校验长度。长度按宽度计：ASCII 字符计 1，其它字符（如中文）计 2 |
| 可选参数 | 不用时**不要放进请求体**。可选的字符串参数传 `null` 会被参数校验拒绝（`400`） |
| **成功判定** | **看响应体的 `code` 字段，不看 HTTP 状态码** |
| 凭证 | 平台发放的 `app_key` / `app_secret`（需要从平台方获取） |

**平台的业务应答一律是 `HTTP 200`**，响应体统一为：

```json
{ "data": {...}, "code": 200, "msg": "ok", "t": 1788431707137 }
```

业务成功还是失败，**看响应体的 `code`，不看 HTTP 状态码** —— 一个业务失败（余额不足、
幂等冲突、签名不对）同样是 `HTTP 200`，错误码在 `code` 里。

> 🔴 **但"请求没进到业务层"是另一回事，你的客户端必须能区分。** 用了 `http://`、路径**前缀**写错
> （不是 `/openapi/v1`）、请求在到达平台之前就被拦下、连接中途出错，这些都不会进到上面那套信封 ——
> 你拿到的是一个**真实的 HTTP 状态码**，响应体**可能不是 JSON**（网关的 HTML 错误页、
> 空响应、纯文本都可能）。前缀正确时，路径名拼错、方法不对、`Content-Type` 不对、请求体超过
> 256KB，仍然是 `HTTP 200` + 信封，见 [§3.1](#31-非-200-的响应怎么处理)。
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

每个响应都带 `X-Request-ID` 响应头：请求里带了合法的 `X-Request-ID`（`[A-Za-z0-9._:-]`，1–64 位）时原样回显，否则由平台生成。向平台报障时请附上它。

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

**另有两条结构上限**，都从**被签对象的根**算起，不是从 `body` 算起：嵌套深度 8 层 —— `body`
本身已是第 1 层，所以 `body.a.b.c.d.e.f.g` 是最深能签的路径；节点总数 512 —— 对象、数组本身也计入，
被签对象的固定部分已占 8 个，留给 `body` 内部的约 504 个。超出分别是 `CANONICAL_TOO_DEEP` 与
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

**去重范围与记忆期**：按「环境 + 商户 + nonce」去重，与 `app_key` 无关（轮换凭证之后，
10 分钟内用过的 nonce 仍不可复用），记忆期为时钟窗口的两倍（10 分钟）。
沙盒与生产互不干扰。随机 nonce 天然满足这条约束，无需自己维护池。

⚠️ **nonce 在鉴权通过的那一刻就被消耗，与业务结果无关。** 收到任何业务应答（包括 `402` /
`409` / `503`）之后再重试，都必须换新的 `x-nonce` 与 `x-timestamp` 重新签名 —— 把同一份已签名的
请求原样重发，只会得到 `401 NONCE_REPLAYED`。下文说的"原样重试"一律指**请求体不变**，签名头每次重新生成。

### 2.3 认证检查顺序

在这 11 步之前，请求先要通过几道与鉴权无关的检查：请求体大小（`413`）、`Content-Type` 与 JSON 格式
（`415` / `400 REQUEST_BODY_*`）、路径与方法（`404` / `405`）、参数校验（`400`）。没有参数要传时也请发 `{}`。

多个条件同时不满足时，你只会看到**最先命中**的那一个：

```
1. 四个签名头齐全 ......... 401 AUTH_HEADERS_MISSING
2. nonce 格式 ............. 401 NONCE_INVALID
3. 时间戳窗口 ............. 401 TIMESTAMP_INVALID
4. app_key 存在且启用 ..... 401 APP_KEY_INVALID
5. 凭证未过期 ............. 401 CREDENTIAL_EXPIRED
6. 源 IP 在白名单内 ....... 403 IP_NOT_ALLOWED
7. QPS 未超限 ............. 429 QPS_LIMITED   ← 免限接口跳过这一步
8. 载荷可规范化 ........... 400 CANONICAL_*
9. 签名正确 ............... 401 SIGNATURE_INVALID
10. scope 足够 ............ 403 SCOPE_FORBIDDEN
11. nonce 未被用过 ........ 401 NONCE_REPLAYED
```

第 8 步**刻意与 `SIGNATURE_INVALID` 分开**：它说的是"这份载荷里
有一个签不了的值"（小数、超范围整数、不合法的键名…），不是"你的密钥不对"。看到
`CANONICAL_*` 请查载荷，不要查密钥。

> **三个写接口不受 QPS 约束**：`/openapi/v1/order/create`（开卡）、`/openapi/v1/order/renew`（续订）、
> `/openapi/v1/order/refund`（退款申请）。
> 它们不占用配额，也不会因为你其他接口打满而被拒 —— 成交不该被查询流量挤掉。
> 第 7 步之外的其他十步，这三个接口一步都不少。
>
> **其余接口按下表分桶计数，每一份都是完整的 `qpsLimit`**：
>
> | 配额 | 接口 |
> |---|---|
> | 独立一份 | `/products/list`、`/products/detail`（两者**共用**这一份） |
> | 独立一份 | `/order/list` |
> | 独立一份 | `/order/query` |
> | 独立一份 | `/order/usage` |
> | 独立一份 | `/iccid/profile` |
> | 共用一份 | 其余全部，含 `/cardType`、`/iccid/renewProducts`、`/account/balance` |
>
> 也就是说：商品目录的全量同步、订单翻页、卡状态轮询彼此互不影响，也影响不到下单、续订与退款。
> 超限时返回的都是 `429 QPS_LIMITED`，**不区分是哪一份** —— 退避重试即可。
> 配额在验签之前计数，所以签名错误、scope 不足的请求同样占用配额。这里说的是平台的业务配额
> （`code: 429`）；传输层的 `HTTP 429`（[§3.1](#31-非-200-的响应怎么处理)）不区分接口。

两条由此推出、对排障很有用的性质：

- **nonce 在第 1–10 步全部通过之后才被消耗**（第 11 步在最后）。所以错签名重试永远回
  `SIGNATURE_INVALID`、越权重试永远回 `SCOPE_FORBIDDEN`，**不会被自己的重试污染成 `NONCE_REPLAYED`**。
  反过来，鉴权一旦通过，nonce 就已用掉 —— 之后的重试必须换新 nonce（见 [§2.2](#22-nonce)）。
- **参数校验在鉴权之前**。缺必填字段时你拿到的是 `400 xxx required`，不是 `401`。

### 2.4 scope

| scope | 覆盖接口 |
|---|---|
| `esim.read` | `products/list` `products/detail` `cardType` `order/query` `order/list` `iccid/profile` `order/usage` `iccid/renew` `iccid/renewProducts` `account/balance` |
| `esim.write` | `order/create` `order/renew` `order/refund` `sandbox/card/status`（仅沙盒，见 §7） |

缺 `esim.write` 调写接口 → `403 SCOPE_FORBIDDEN`（在签名校验**之后**判定，所以它确实是权限问题，不是签名问题）。
两个 scope 互相独立：`esim.write` 不包含 `esim.read`，只持写权限的凭证调查询接口同样是 `403 SCOPE_FORBIDDEN`。

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
            if not KEY_RE.fullmatch(k):  raise ValueError(f'键名不合法: {k}')   # 不用 match：$ 会放过末尾的换行
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

def call(path, body=None):
    body = {} if body is None else body   # 没有参数也要发 {}，否则会被 415 拦下
    ts, nonce = str(int(time.time() * 1000)), 'n_' + secrets.token_hex(8)
    sign = sign_inbound(APP_SECRET, APP_KEY, 'POST', path, ts, nonce, body)
    # 可以直接用 json=body：签名不要求你控制发出去的字节，
    # requests 怎么序列化都不影响验签。
    return requests.post(BASE + path, json=body, headers={
        'x-api-key': APP_KEY, 'x-timestamp': ts, 'x-nonce': nonce, 'x-sign': sign,
    })
```

**Node.js**（手写，无依赖；`fetch` 需要 Node 18+）

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

把签名、发送与 [§3.1](#31-非-200-的响应怎么处理) 的两步判定包成一个 `invoke`。§4 的 Node 示例都通过它调用；
Java 示例里的 `XpinEsimClient.invoke` 指同样职责的封装，请按同一套规则自行实现：

```js
// 成功返回 data；失败抛错：err.code 是业务码，err.httpStatus 是传输层状态码
function createClient({ baseUrl, appKey, appSecret }) {
  async function invoke(path, body = {}) {
    const timestamp = String(Date.now())
    const nonce = crypto.randomBytes(16).toString('hex')   // 每次请求一个新 nonce，重试也要换
    const sign = signInbound(appSecret, { apiKey: appKey, method: 'POST', path, timestamp, nonce, body })
    const res = await fetch(baseUrl + path, {
      method: 'POST',
      // 务必带 Content-Type，否则会在鉴权之前就被 415 CONTENT_TYPE_MUST_BE_JSON 拦下
      headers: {
        'content-type': 'application/json',
        'x-api-key': appKey, 'x-timestamp': timestamp, 'x-nonce': nonce, 'x-sign': sign
      },
      body: JSON.stringify(body)
    })
    // 第 1 步：HTTP 状态码不是 200 = 请求没进业务层，响应体可能不是 JSON，不要解析
    if (res.status !== 200) throw Object.assign(new Error(`HTTP ${res.status}`), { httpStatus: res.status })
    // 第 2 步：平台应答，看信封里的 code
    const envelope = await res.json()
    if (envelope.code !== 200) throw Object.assign(new Error(envelope.msg), { code: envelope.code, msg: envelope.msg })
    return envelope.data
  }
  return { invoke }
}

const client = createClient({ baseUrl: 'https://sandboxapi.xpin.network', appKey: 'ak_xxx', appSecret: 'sk_xxx' })
```

**不需要**保证签名用的字符串与发送用的字符串逐字节相同 —— 签的是数据，不是字节。
但签的必须是**与发出去的 JSON 等价的数据**：`body` 里只放字符串、整数、布尔、`null`、对象与数组；
`Date` 之类的对象请先转成字符串再签（否则签的是 `{}`、发出去的是时间串，验签必然失败）。

其它语言：Go / Java / PHP / .NET 都能在几十行内写完（只用到排序、UTF-8 字节长度、
HMAC-SHA256 三样）。**写完请先跑 [2.6](#26-自检向量) 那张自检向量表**，全过再联调。⚠️ 几个语言特有的注意点：
Python 的 `bool` 是 `int` 的子类型，必须先判布尔，否则会把 `true` 编码成 `i:1`；PHP 若用
`json_decode($s, true)`，`{}` 与 `[]` 都会变成空数组，`o:` 与 `a:` 就分不出来了 —— 请用默认的对象模式解析，
空对象用 `new stdClass()` 表达；Go 注意 `map` 遍历是随机序，必须显式排序。

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
| 12 | 数组下标按字节序排：`tags.10` 排在 `tags.2` 之前 | `/openapi/v1/products/list` | `{"tags":["a","b","c","d","e","f","g","h","i","j","k"]}` | `d05ecfa3deb79b2d0f1588560e0aedbc38fcff7c65211500fd112a984c33ef1b` |

> 第 3 与第 4 条、第 6 与第 7 条、第 10 与第 11 条各是一组**对照**：两条签出不同的值才算对。
> 第 3/4 组验的是类型标记（少了它字符串 `"1"` 与整数 `1` 会同签），第 6/7 组验的是空容器
> 留痕，第 10/11 组验的是路径归一（这一组反过来，两条必须**相同**）。
> 第 12 条验的是排序：按完整路径的字节序，`tags.10` 小于 `tags.2`，按数组下标顺序拼会签出别的值。
>
> ⚠️ 路径归一只作用于**被签的** `path`。实际请求的路径必须与 §4 逐字一致（大小写、斜杠都不能差），
> 否则在鉴权之前就会得到 `code: 404`。

---

## 3. 错误码

| `code` | `msg` | 含义 | 你该做什么 |
|---|---|---|---|
| `200` | `ok` | 成功 | — |
| `400` | `<字段> required` / `<字段> length error [a-b]` / `<字段> Range error` / `<字段> Type error, ...` | 参数校验失败（**自由文本，不是稳定错误码**）。必填字段传空串得到 `required`，传纯空白得到 `length error`（先去首尾空白再计长度） | 修参数。判定用 `code===400` + 字段名子串，不要精确匹配 `msg` |
| `400` | `ORDER_RANGE_SELECTOR_REQUIRED` / `ORDER_RANGE_SELECTOR_CONFLICT` / `ORDER_PAGE_TOKEN_INVALID` | 列表选择器/游标非法 | 见 §4.6 |
| `400` | `ACCOUNT_PAGE_NO_INVALID` / `ACCOUNT_PAGE_SIZE_INVALID` | 余额接口的分页参数过了参数校验、但不是合法整数（如数字 `0`、字符串 `"1.5"`） | 传整数，见 §4.13 |
| `400` | `REQUEST_BODY_NOT_JSON` / `REQUEST_BODY_MUST_BE_OBJECT` | 请求体不是 JSON，或是 JSON 但不是对象 | 检查序列化 |
| `401` | `AUTH_HEADERS_MISSING` | 四个签名头缺任意一个 | 补头 |
| `401` | `NONCE_INVALID` | nonce 格式不符 | 用 `^[A-Za-z0-9._:-]{8,128}$` |
| `401` | `TIMESTAMP_INVALID` | 时间戳非数字或偏差 > 5 分钟 | **对时**；确认用的是毫秒 |
| `401` | `APP_KEY_INVALID` | app_key 不存在/已停用/环境不匹配 | 找平台核对凭证 |
| `401` | `CREDENTIAL_EXPIRED` | 凭证已过期 | 找平台轮换 |
| `401` | `SIGNATURE_INVALID` | 签名不匹配 | 先跑 [2.6](#26-自检向量) 的自检向量，全过再查联调 |
| `400` | `CANONICAL_*` | 载荷里有签不了的值（小数、超范围整数、不合法键名、嵌套过深、节点过多） | **查载荷，不要查密钥**。`msg` 只有错误码、不指出字段，请用本地的规范化实现定位 |
| `401` | `NONCE_REPLAYED` | nonce 已被用过 | 每次请求换新 nonce |
| `403` | `SCOPE_FORBIDDEN` | 凭证 scope 不足 | 找平台加 scope |
| `403` | `IP_NOT_ALLOWED` | 源 IP 不在白名单 | 找平台加 IP |
| `400` | `ORDER_INPUT_INVALID` / `EXTERNAL_USER_ID_INVALID` | 受理类接口的兜底校验，**正常不会出现**：字符串参数先去首尾空白再校验长度，空白串、超长的 `externalUserId` 在通用参数校验那一步就会被拒（`<字段> required` / `<字段> length error [a-b]`） | 同通用参数校验 |
| `403` | `SOURCE_PARTNER_NOT_ALLOWED` | `sourcePartnerCode` 不是你名下的子分销商 | 检查该字段 |
| `404` | `PRODUCT_NOT_FOUND` | 商品不存在或不属于你 | 从 `products/list` 动态取码 |
| `404` | `PRODUCT_NOT_AVAILABLE` | 下单时商品不可售/不存在 | 同上 |
| `404` | `ORDER_NOT_FOUND` | 订单不存在或不属于你 | 检查 `clientOrderNo` |
| `404` | `CARD_NOT_FOUND` | 卡不存在或不属于你 | 检查 `iccid` |
| `404` / `405` | — | 路径名拼错（前缀正确），或 HTTP 方法不是 `POST`（`GET` 得到 `405`，其它方法得到 `404`）。`msg` 不是稳定错误码，不要对它做断言。⚠️ **前缀本身写错是另一回事**，见 [§3.1](#31-非-200-的响应怎么处理) | 核对路径名与方法 |
| `409` | `IDEMPOTENCY_KEY_REUSED` | 同 `idempotencyKey` 但请求内容不同 | 换新的幂等键 |
| `409` | `CLIENT_ORDER_NO_REUSED` | `clientOrderNo` 已被别的订单占用 | 换新的业务单号 |
| `409` | `RENEW_NOT_ALLOWED` | 这张卡此刻不可续订 | 先调 `iccid/renew` 看 `reasons` |
| `409` | `ORDER_NOT_REFUNDABLE` | 这一单现在不能退：未履约、已退款、或不具备可退款的计价信息 | 查 `/order/query` 看当前状态。**改参数改不出来**，见 §4.12 |
| `409` | `REFUND_ALREADY_IN_PROGRESS` | 这一单已有一张在途退款单 | 等 `ORDER_REFUND_RESULT` 出结论后再发起，见 §4.12 |
| `402` | `BALANCE_LIMIT_REACHED` | **预付费余额不足**：本单扣完会跌破你的透支额度，整单被拒、一分钱不扣 | 充值。**不要重试**，见下方 ⚠️ |
| `503` | `BALANCE_ACCOUNT_NOT_FOUND` | 余额账户尚未就绪 | 联系平台开通后重试 |
| `503` | `CURRENCY_MISMATCH` | 账户币种与商品币种不一致 | 联系平台核对，**改请求改不出来** |
| `503` | `ORDER_NO_COLLISION` | 平台生成订单号时连续冲突，本次请求什么都没写、没扣 | 用**同一个** `idempotencyKey` 重新签名后重试 |
| `503` | 其它未列出的 `msg` | 开卡 / 续订 / 退款受理时的平台侧临时故障。`msg` 是平台内部的错误标识，不是稳定错误码 | 用**同一个** `idempotencyKey` 重新签名后退避重试（已受理的会以 `replayed: true` 返回）；持续出现请联系平台 |
| `413` | `REQUEST_ENTITY_TOO_LARGE` | 请求体超过 256KB | 拆小 |
| `415` | `CONTENT_TYPE_MUST_BE_JSON` | `Content-Type` 不是 `application/json` | 改头 |
| `429` | `QPS_LIMITED` | 超过该商户每秒请求上限。**开卡、续订、退款三个写接口不受此限**，配额口径见 [§2.3](#23-认证检查顺序) | 退避重试 |
| `406` | `[controller error]` | 服务端异常。`msg` 为固定字符串 `[controller error]`，不携带原因 | 见下方 ⚠️ |
| `501` | `FEATURE_NOT_IMPLEMENTED` | 该路径是平台预留、尚未开放的接口（鉴权通过后才会给出这个码） | 只调用 §4 列出的接口 |
| `503` | `OPENAPI_AUTH_UNAVAILABLE` / `PLATFORM_NOT_ENABLED` / `PRODUCT_ADAPTER_UNKNOWN` | 平台侧依赖或配置问题，与你的请求无关 | 退避重试并联系平台 |

> ⚠️ **`402` 请不要重试。**
>
> 你的账户在平台上是**预付费**的：`/order/create` 与 `/order/renew` 在受理时就按结算价扣款，
> 余额扣完会跌破额度时整单被拒、一分钱不扣。充值之前重试是同一个结果。
>
> 请把 `402` 当成**需要人工介入**的信号（告警 + 暂停该
> 商户的下单队列），不要交给自动重试 —— 对它做无限重试除了浪费两边的资源，不会得到
> 任何不同的答案。
>
> - 收到 `402`：停止对这一单重试，转人工或转充值流程。充值到账后可用 `/account/balance`（§4.13）确认余额。
> - 收到 `503`：联系平台，处理完成后用同一个 `idempotencyKey` 重新签名重试即可。
>
> 三个资金相关的码里，只有 `402` 需要你侧采取动作。

> ⚠️ **`406` 的处置：退避重试，连续出现请联系平台。**
>
> `msg` 固定是 `[controller error]`。它不是参数问题（那是 `400`）、不是签名问题（`401`）、
> 也不是余额问题（`402`）。联系平台时请带上响应头里的 `X-Request-ID`、`clientOrderNo` 与大致时间。

**跨租户隔离**：查不到与"不属于你"返回**同一个** `404`，平台不会告诉你"这个单存在但不是你的"。
所以 `404` 不能用来探测他人数据是否存在。

沙盒专用接口 `/sandbox/card/status` 另有自己的拒绝码（含 `422`），见 [§7.3](#73-拒绝)。

### 3.1 非 200 的响应怎么处理

上面整张错误码表说的都是**平台的业务应答**，它们一律是 `HTTP 200` + JSON 信封，
错误码在 `code` 里 —— 包括 `429 QPS_LIMITED`。

但有一类响应**根本没有进到业务层**，它们带真实的 HTTP 状态码，响应体**可能不是 JSON**：

| 你会看到 | 通常是因为 |
|---|---|
| `301` + 非 JSON | 用了 `http://`。**301 不保留 POST body**，直接用 `https://` |
| `404` + 非 JSON | **路径前缀**写错（正确前缀是 `/openapi/v1`） |
| `429` | 请求频率异常 —— 退避重试，并检查客户端有没有重试风暴或死循环 |
| `5xx` / 连接中断 | 传输层或服务端异常 |

> ℹ️ **注意区分**：只有**前缀**写错才会落到上表的 `404`。前缀正确、只是路径名拼错时，
> 请求已经进到业务层，你拿到的是 `HTTP 200` + `code: 404` 的正常信封。
> 同样地，HTTP 方法用错（`code` 为 `405` 或 `404`）、`Content-Type` 不对（`415`）、请求体超过 256KB（`413`）
> **也都是 `HTTP 200` + 对应的 `code`**，不在上表之列。

**判定顺序必须是先状态码、后 JSON**：

```js
const res = await fetch(url, init)

// 第 1 步：先看 HTTP 状态码。不要在这之前 JSON.parse。
if (res.status !== 200) {
  // ⚠️ 这里的响应体可能不是 JSON，解析它会抛异常。
  // 只有 429 与 5xx 值得退避重试；其余（301/404/413 等）是配置或请求本身错了，
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
> 示例代码里的 `client.invoke(path, body)` 见 [§2.5](#25-签名客户端)：签名并发送，成功时返回 `data`，失败时抛错。

### 4.1 商品列表 `/products/list`

拉取**发布给你**的在售商品。这是获取 `productCode` 的**唯一正确来源**。

**请求**

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| `pageNo` | number | 否 | `[1, 100000]`，默认 `1` | 页码，从 `1` 开始。负数或大于 100000 返回 `400 pageNo Range error`；传 `0` 按 `1` 处理（不报错 —— 从 0 起翻会把第一页取两次） |
| `pageSize` | number | 否 | `[1, 100]`，默认 `20` | 每页条数。负数或大于 100 返回 `400 pageSize Range error`；传 `0` 按默认值 `20` 处理 |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 商品数组，见下 |
| `hasMore` | boolean | 是否还有下一页（翻页终止条件） |
| `pageNo` / `pageSize` | number | **实际生效的**页码 / 每页条数。传 `0` 时按默认值处理，以字符串传入的小数（如 `"2.5"`）向下取整 —— 它们可能与你发出的值不同。请始终传整数：JSON 数字形式的小数在验签环节即被拒（`400 CANONICAL_NUMBER_UNSUPPORTED`） |
| `language` | string | 商品文案的语言，当前恒为 `en`。平台不做语言协商，请不要在请求里传语言相关参数（传了 `lang` 也会被忽略，但仍按 2–8 位字符串校验，不合规返回 `400`） |

> ℹ️ `list` 按 `productCode` 升序排列。分页基于偏移量：翻页期间若恰有商品发布或下架，可能漏项或重复 ——
> 请按 `productCode` 去重，并定期全量重拉。

**商品对象**

同一个商品在 `/products/list`、`/products/detail`、`/order/query`、`/order/list` 上是
**同一份结构**，字段名、字段集、键序一致。

| 字段 | 类型 | 说明 |
|---|---|---|
| `productCode` | string | 对外商品码，下单用它 |
| `productName` | string\|null | 商品名 |
| `tagName` | string\|null | 分组标签，如 `Europe` |
| `volume` | string\|null | 流量额度（字符串形式的十进制数），**单位 MB**。不限量套餐（`dataLimited: "N"`）上它没有意义，当前为 `"0.000000"` —— 是否限量请以 `dataLimited` 为准 |
| `dataLimited` | string | `Y` 限量 / `N` 不限量 |
| `validity` | number\|null | 套餐天数：激活之后可以使用的天数 |
| `expireDay` | number\|null | 未激活状态下的有效期（天）：开卡之后须在这段时间内完成激活 |
| `cardType` | string | 卡型标识，如 `eo1` / `ep1` / `ep3` / `C4`。取值由平台目录决定，**请勿枚举或硬编码** |
| `productType` | string\|null | 形态：`DATA_PACKAGE` 流量包 / `DAYPASS` 日包；无法归类时为 `null` |
| `mccList` | string[]\|null | 覆盖网络的 MCC 列表 |
| `attributes` | object\|null | 展示用扩展属性，**见下方专节** |
| `settlementPrice` | string\|null | 结算单价。**默认结构不披露金额，恒为 `"0"`**；仅当平台方为贵方账号开通了结算价披露时才是真实单价（十进制字符串，6 位小数），此时早期受理订单的快照里可能为 `null` |
| `retailPrice` | string | 平台不披露零售价，恒为 `"0"` |
| `currency` | string\|null | 币种，如 `USD` |
| `configVersion` | number | 商品配置的**发布代次**：平台每次发布该商品都会递增，**即使内容未变**。可用于判断"需要重新拉取"，不能用于判断"内容确实变了" |

> ⚠️ **类型里带 `null` 的字段都可能是 `null`**，`attributes` 尤其常见（见下节）。
> 平台**不为未知值编造默认值** —— 请按上表的类型解析，不要假设可空字段一定有值。

> ℹ️ **`volume` 的单位是 MB**，与 `/order/usage` 的字节口径换算关系为
> `dataTotalBytes = volume × 1024 × 1024`（例："CIS 1GB 365D" 的 `volume` 是
> `"1024.000000"`，对应 `dataTotalBytes` 为 `1073741824`）。
>
> `dataLimited: "N"` 的不限量套餐没有总量概念，`volume` 当前为 `"0.000000"` ——
> 那是"没有额度上限"，不是"额度为零"。

> ℹ️ **需要不同的商品字段结构？** 上表是默认结构。若贵方的系统需要其它字段组织方式
> （不同的字段名、额外的派生字段、或把覆盖范围/价格按贵方口径重排），
> **可联系平台方定制** —— 平台以适配器的方式为渠道单独提供，不影响本文档描述的默认结构。

#### `attributes` 里有什么

`attributes` 是商品的**展示用**扩展属性块，由平台目录**原样返回**，**整块可能为 `null`**。

> 🔴 **除 `speedRule` 外，平台不解释这个块里的任何键。** 这些键不参与下单、续订、用量或任何业务判定，
> 平台也不校验它们的键名与取值。因此：
>
> - **键集不是契约**，不同商品可以不同，也可能随目录维护而增减，恕不另行通知；
> - **请按可选字段读取**，取不到就不展示；
> - **不要基于它做业务判定**，也不要对它做强类型解析。

以目前的沙盒目录为例，`attributes` 常见两种形态：**`null`**，或**只含一个 `speedRule` 键** —— 这不是承诺，见上方 🔴。
`speedRule` 出现在不限量（`dataLimited: "N"`）的日包上，描述"高速额度用完后限速"的规则：

```json
{
  "speedRule": {
    "period": "daily",
    "throttle": {
      "unit": "Kbps",
      "value": 512
    },
    "highSpeed": {
      "unit": "GB",
      "value": 2
    }
  }
}
```

`speedRule` 是唯一的例外：**它出现时，结构已在发布时按下表校验过**，不合下表的规则发布不上去。
它本身仍可能不出现，所以请照样按可选字段读取；出现时可以按下表解析。

| 路径 | 取值 |
|---|---|
| `speedRule.period` | `"daily"`：高速额度每天重置；`"total"`：整个有效期共用一份额度。**与 `productType` 无关**：日包上也可能是 `"total"`，总量包上也可能是 `"daily"`，额度周期以这个字段为准 |
| `speedRule.highSpeed.value` / `.unit` | 大于 0 的数值 + `"MB"` 或 `"GB"`，如 `2` + `"GB"` —— 每个周期的高速额度 |
| `speedRule.throttle` | `null`：高速额度用完即断网；或 `{ "value", "unit" }`：大于 0 的数值 + `"Kbps"` 或 `"Mbps"`，如 `512` + `"Kbps"` —— 超出后的限速值 |

> ⚠️ `speedRule` 里的 `unit` 是这个块自带的单位标注，**与商品对象顶层 `volume` 的 MB 口径无关**，
> 两者不要混用。单位区分大小写（`MB` 是兆字节，`Kbps` 是千比特每秒）。
> `speedRule` 同样**不参与**下单、续订与用量判定，它只描述套餐的限速规则。

**示例响应**（`pageSize: 2` 的第一页）

```json
{
  "data": {
    "list": [
      {
        "productCode": "XP01791078F286D359D6790456",
        "productName": "CIS 1GB 365D",
        "tagName": "CIS",
        "volume": "1024.000000",
        "dataLimited": "Y",
        "validity": 365,
        "expireDay": 365,
        "cardType": "eo1",
        "productType": "DATA_PACKAGE",
        "mccList": [
          "282",
          "401",
          "255",
          "283",
          "437",
          "434",
          "257",
          "436",
          "259",
          "250",
          "400"
        ],
        "attributes": null,
        "settlementPrice": "0",
        "retailPrice": "0",
        "currency": "USD",
        "configVersion": 9
      },
      {
        "productCode": "XP0909F3C1C8A9D7F97456F799",
        "productName": "CIS 7D Unlimited",
        "tagName": "CIS",
        "volume": "0.000000",
        "dataLimited": "N",
        "validity": 7,
        "expireDay": 365,
        "cardType": "eo1",
        "productType": "DAYPASS",
        "mccList": [
          "282",
          "401",
          "255",
          "283",
          "437",
          "434",
          "257",
          "436",
          "259",
          "250",
          "400"
        ],
        "attributes": {
          "speedRule": {
            "period": "daily",
            "throttle": {
              "unit": "Kbps",
              "value": 512
            },
            "highSpeed": {
              "unit": "GB",
              "value": 2
            }
          }
        },
        "settlementPrice": "0",
        "retailPrice": "0",
        "currency": "USD",
        "configVersion": 9
      }
    ],
    "hasMore": true,
    "pageNo": 1,
    "pageSize": 2,
    "language": "en"
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554466998
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

> ⚠️ **不要硬编码 `productCode`**，也不要使用任何不是从 `products/list` 取得的编码（例如其他渠道的商品 SKU）：
> 查询类接口会返回 `404 PRODUCT_NOT_FOUND`，开卡返回 `404 PRODUCT_NOT_AVAILABLE`，续订返回 `409 RENEW_NOT_ALLOWED`。
> 平台可能调整商品目录，对外码会随之变化。

#### 沙盒环境联调用的套餐码

沙盒同样可以调用 `products/list`，但沙盒里并非每个商品都能真正开卡 / 续充 —— 平台方为你配置好账号后，
请**向平台方索取本账号在沙盒环境可用于开卡 / 续充的套餐码**（它们都在 `products/list` 的结果里）。
生产环境直接调用 `products/list` 拉取即可 —— 它返回的就是发布给你的全部在售商品。

**联调选型建议**（与具体码无关，按 `products/list` 返回的属性挑）：

| 你要验证的 | 怎么挑 |
|---|---|
| 开卡 | 任选一个即可 |
| 续充正向流程 | 选 `productType = DATA_PACKAGE` 的。在"日包独占"规则的卡型上，卡上有日包（从下单到日包结束，含未激活期间）时整张卡都不可续订，会稳定返回 `DAYPASS_IN_FORCE`，跑不了正向流程 |
| 续充的"不可续"分支 | 反过来选 `DAYPASS`，正好用来验证你对 `DAYPASS_IN_FORCE` 的处理 |
| 跨形态续充规则 | 给 `ep3` 流量包卡续一个 `ep3` 日包，会得到 `DAYPASS_REQUIRES_EMPTY_CARD`；而 `C4` 卡型（"自由续订"规则）在同卡型内跨形态续充是放行的 |

> ⚠️ **沙盒环境与生产环境的 `productCode` 互不通用**（不保证不同，也不保证相同），切换环境时必须重新拉取目录，不能沿用联调期的码。

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
    "productCode": "XP0909F3C1C8A9D7F97456F799",
    "productName": "CIS 7D Unlimited",
    "tagName": "CIS",
    "volume": "0.000000",
    "dataLimited": "N",
    "validity": 7,
    "expireDay": 365,
    "cardType": "eo1",
    "productType": "DAYPASS",
    "mccList": [
      "282",
      "401",
      "255",
      "283",
      "437",
      "434",
      "257",
      "436",
      "259",
      "250",
      "400"
    ],
    "attributes": {
      "speedRule": {
        "period": "daily",
        "throttle": {
          "unit": "Kbps",
          "value": 512
        },
        "highSpeed": {
          "unit": "GB",
          "value": 2
        }
      }
    },
    "settlementPrice": "0",
    "retailPrice": "0",
    "currency": "USD",
    "configVersion": 9
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554467197
}
```

**负例**：不存在、已下架或不属于你的码 → `404 PRODUCT_NOT_FOUND`。

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
| `capability.supportUsageQuery` | boolean | 是否支持实时用量测量。为 `false` 时 `/order/usage` 仍可调用，含义见 §4.11 |
| `capability.maxRenewCount` | number\|null | 该卡型声明的同时有效套餐数上限，`null` = 未声明上限。仅供参考：是否按它限制取决于卡型的续订规则，能不能续以 §4.7 的 `allowed` 为准 |
| `capability.timeZone` | string\|null | 该卡型的时区标识。**取值格式不固定，不要解析它**，仅作展示；未知时为 `null` |
| `capability.observedAt` | string | 该能力的同步时刻（ISO-8601） |

**示例响应**

```json
{
  "data": {
    "productCode": "XP01791078F286D359D6790456",
    "cardType": "eo1",
    "productType": "DATA_PACKAGE",
    "capability": {
      "renewable": true,
      "supportUsageQuery": false,
      "maxRenewCount": 99,
      "timeZone": "UTC+0",
      "observedAt": "2026-09-16T10:18:45.929Z"
    }
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554467370
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

**负例**：不存在、已下架或不属于你的码 → `404 PRODUCT_NOT_FOUND`。

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
| `sourcePartnerCode` | string | 否 | `[1, 64]` | 委托下单时指定你名下**已启用的直属**子分销商。**扣款始终发生在你（发起方）的余额上**，订单与卡都归在你的账号下；子分销商只作归属记录，不在任何响应中返回，用子分销商自己的凭证也查不到这些订单 |

> 🔴 **`clientOrderNo` 虽然可选，但省略它这一单就查不了、也退不了。** `/order/query` 与 `/order/refund`
> **只能**按 `clientOrderNo` 定位订单 —— 平台返回的 `orderNo` **不是查询键**，没有任何接口按它查。
> 省略后只能在 `/order/list` 里翻找（`externalUserId` 选择器 —— 前提是下单时传了它；或 `productCode` 选择器），
> 开卡单在履约前连 `iccid` 选择器也捞不到。请始终传一个你自己能复现的值。

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

> ⚠️ **受理即扣款。** 平台在返回 `ACCEPTED` 的同一个事务里就按该商品的结算价从你的预付费余额扣了钱，不是等履约才扣。后续订单落 `FAILED`（不会出卡）或 `REFUNDED`（退款确认）时自动退回，中间态不动钱。极少数情况下订单状态会被事后更正（见 §5.1），余额随之重新扣回或退回。
>
> 余额不足以致本单会跌破额度时，**整单被拒、一分钱不扣**，返回 `402 BALANCE_LIMIT_REACHED`。
> **收到这个码请不要重试**，先完成充值，见 [§3](#3-错误码)。

**示例响应**

```json
{
  "data": {
    "orderNo": "XE17895552107334DAD5D0C9D",
    "clientOrderNo": "docverify-1789555210461",
    "productCode": "XP6EE2B3DABB39D0D1A121446C",
    "orderStatus": "ACCEPTED",
    "accepted": true,
    "replayed": false
  },
  "code": 200,
  "msg": "ok",
  "t": 1789555210748
}
```

> 🔴 **`ACCEPTED` 只是受理，不是出卡。** 出卡是异步的，绝大多数在数分钟内完成。
> **请不要按固定时长判失败** —— 在拿到终态（`FULFILLED` / `FAILED`）之前一律视为在途。
> 受理后用你的 `clientOrderNo` 轮询 §4.5，或等 §6 的 `ORDER_OPEN_RESULT` 回调。
> ⚠️ **回调只在 `FULFILLED` 时发出，`FAILED` 不发任何事件。** 只靠回调的接入方也必须用 §4.5 轮询兜底，
> 否则失败的单会被一直当成在途。

#### 幂等语义

| 情况 | 结果 |
|---|---|
| 同 `idempotencyKey` + **相同请求内容** | `200`，返回**同一个** `orderNo`，`replayed: true` |
| 同 `idempotencyKey` + **不同请求内容** | `409 IDEMPOTENCY_KEY_REUSED` |
| 同 `clientOrderNo` + **不同** `idempotencyKey` | `409 CLIENT_ORDER_NO_REUSED` |

所以：**网络超时后原样重发是安全的**，不会重复建单。但换了内容就必须换幂等键。

- **"请求内容相同"** 指 `productCode`、`clientOrderNo`、`externalUserId`、`sourcePartnerCode` 四个字段（去首尾空白后）完全相同；可选字段不传与传空串视为相同。
- **重放不会再扣一次款**，也不受之后商品下架、余额变化的影响。
- **被拒的请求不占用**幂等键与 `clientOrderNo`（`402` / `403` / `404` / `503` 都没有建单），处理完原因后可以用同一个键重发。
- `idempotencyKey` 与 `clientOrderNo` 都在"你的账号 + 环境"内唯一，**开卡与续订（§4.9）共用同一个命名空间**，永久有效、不会过期释放；比较时不区分大小写 —— 不要用只有大小写不同的值区分两单。
- "原样重发"指请求体不变；签名头每次都要重新生成（新的 `x-nonce`，见 [§2.2](#22-nonce)）。

**负例**：商品不存在或不可售 → `404 PRODUCT_NOT_AVAILABLE`。⚠️ **目录里存在的商品也可能返回这个码**（该商品当前不可下单）—— 收到后请跳过该商品并联系平台，**重新拉目录没有用**，码本来就是从那儿取的；非法 `sourcePartnerCode` → `403 SOURCE_PARTNER_NOT_ALLOWED`。

**Node.js**

```js
async function createOrder(client, { productCode, clientOrderNo, externalUserId }) {
  // clientOrderNo 必须有值：它既是幂等键的来源，也是这一单唯一的查询键。
  // ⚠️ 不要让它落到 undefined —— `open-undefined` 同样满足 [8,96] 的长度校验，
  // 于是之后同商品、同用户的下单都会被当成第一单的重放（200 + replayed:true，不建新单），
  // 换了商品则是 409 IDEMPOTENCY_KEY_REUSED。
  if (!clientOrderNo) throw new Error('clientOrderNo is required')
  // 幂等键与业务单号绑定：重试时必须复用同一个键（换了键而 clientOrderNo 不变会被拒为
  // 409 CLIENT_ORDER_NO_REUSED；键和单号都换了才会建出第二张卡）。
  // 这个键要和订单一起持久化，重试时读回同一个值，不要每次现算。
  // `open-` 占 5 个字符：clientOrderNo 取 3–91 个字符，拼出来的键才落在 [8, 96]。
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
    // `open-` 占 5 个字符：clientOrderNo 取 3–91 个字符，拼出来的键才落在 [8, 96]
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

查不到（不存在或不属于你）→ `404 ORDER_NOT_FOUND`。激活码不在本接口返回，请用 §4.10 的 `delivery.activationCode`。

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderNo` | string | 平台订单号 |
| `clientOrderNo` | string\|null | 你的业务单号。下单时未传的订单为 `null`（在 `/order/list` 上会出现） |
| `externalUserId` | string\|null | 终端用户标识 |
| `productCode` | string | 商品码 |
| `orderStatus` | string | 见 §5 状态机 |
| `iccid` | string\|null | **开卡单**履约后才有值；**续订单在受理时就已带上目标卡的 ICCID**。判断履约一律看 `orderStatus === 'FULFILLED'`，不要用它是否为空来推断 |
| `profileStatus` | string\|null | 这一单所在卡的 profile 状态：该单是卡上最近一次履约的订单时，随卡的状态变化同步更新；卡上有更晚履约的订单（如续订）之后就停在当时的值；履约瞬间可能为 `null`。要取卡的当前状态请一律用 `/iccid/profile` 的 `profile.status` |
| `fulfilledAt` | string\|null | **平台**判定该单履约成功的时刻，ISO-8601（`2026-03-01T08:15:30.123Z`） |
| `packageStartTime` | string\|null | 套餐可用窗口的**起点** |
| `packageEndTime` | string\|null | 套餐可用窗口的**终点** |
| `latestActivationTime` | string\|null | 套餐的**激活期限**：过了这个时刻仍未激活，套餐作废 |
| `product` | object | 下单时刻的**商品快照**，字段集与 §4.1 的商品对象相同。见下方说明 |

> **这四个时刻怎么用**
>
> - **判断「用户激活了没有」请读 `/iccid/profile` 的 `packageStatus`**（`orderStatus` 只反映开卡进度），不要用时间字段去推。`packageStartTime` 有值**不代表**用户已经开始使用 —— 部分套餐的可用窗口在履约时就已确定。
> - `packageStartTime` / `packageEndTime` 是这张套餐**什么时候可用到什么时候**，用于展示有效期与到期提醒。`latestActivationTime` 是另一回事：它是**必须在此之前激活**的期限，不是窗口的一部分。
> - `fulfilledAt` 是**平台**的动作时刻；其余三个来自**供应商**，两类不同源，跨源比较时刻差没有意义。
> - 四个都是**订单**维度。一张卡续订多次就有多个套餐窗口，各属各的订单 —— 要按卡看请用 `/order/list` 的 `iccid` 选择器取全部订单。`/iccid/profile` 上的同名字段取的是当前**主导订单**的值，多套餐时只代表其中一张。
> - 较早履约的订单，这四个字段可能为 `null`。**字段一定在**，值可能没有。
> - ⚠️ **刚落 `FULFILLED` 时，`packageStartTime` / `packageEndTime` / `latestActivationTime` 与 `profileStatus`
>   可能还是 `null`**（`fulfilledAt` 一定有值），待后续的状态通知到达后才补齐。自动激活类套餐的起止时刻
>   要等终端用户**真正激活**之后才出现 —— 用户一直不激活，它们就一直是 `null`。
>   所以**收到履约回调后立刻回查，拿到 `null` 是正常的** —— 不要据此判定失败。

**示例响应**（已履约）

```json
{
  "data": {
    "orderNo": "XE1788864310025C4C7895AE2",
    "clientOrderNo": "ev-t25-1788864309531",
    "externalUserId": null,
    "productCode": "XP01791078F286D359D6790456",
    "orderStatus": "FULFILLED",
    "iccid": "89000000000000000134",
    "profileStatus": "ENABLED",
    "fulfilledAt": "2026-09-08T10:46:30.537Z",
    "packageStartTime": "2026-09-08T10:45:16.000Z",
    "packageEndTime": "2027-09-08T10:45:15.000Z",
    "latestActivationTime": "2027-09-08T10:45:15.000Z",
    "product": {
      "productCode": "XP01791078F286D359D6790456",
      "productName": "CIS 1GB 365D",
      "tagName": "CIS",
      "volume": "1024.000000",
      "dataLimited": "Y",
      "validity": 365,
      "expireDay": 365,
      "cardType": "eo1",
      "productType": "DATA_PACKAGE",
      "mccList": [
        "282",
        "401",
        "255",
        "283",
        "437",
        "434",
        "257",
        "436",
        "259",
        "250",
        "400"
      ],
      "attributes": null,
      "settlementPrice": "0",
      "retailPrice": "0",
      "currency": "USD",
      "configVersion": 4
    }
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554467731
}
```

> ⚠️ **价格字段不用于对账。** 默认结构下 `product.retailPrice` 与 `product.settlementPrice`
> 恒为 `"0"`、不携带金额（`/products/list`、`/products/detail`、`/order/list` 同理）。
> 结算金额以平台与贵方的商务约定为准。

> ℹ️ 字段集与 §4.1 的**商品对象**相同。⚠️ 与 `/products/detail` 的差别只有一个：
> 那个接口的 `data` 额外带一个信封字段 `language`，订单里的 `product` 块**没有**它 ——
> 两边共用同一个反序列化类型时请注意这一项。
>
> ⚠️ **订单里的 `product` 是受理那一刻的冻结快照，可能与商品目录的现值不同。**
> 早期受理的订单，快照里没有冻结全部字段，因此 `attributes` 等字段
> 在这些订单上可能是 `null`，而同一个商品在 `/products/detail` 上可能是有值的；
> `configVersion` 也会停留在受理当时的代次。
>
> 这里的 `null` 含义是**"当时没有记录"**，不是"该商品没有这一项"。
> 快照按定义不可回填 —— 回填会把"当时的事实"改写成"现在的事实"。
>
> 不论渲染结果如何，**订单本身总是查得到**：商品块的字段可能为 `null`，但 `/order/query` 与 `/order/list` 不会因此失败。

**Node.js —— 轮询到出卡**

```js
// 仍在流转中的状态（见 §5.1 状态机）。落到这个集合之外且不是 FULFILLED，就是终态失败
const IN_FLIGHT = new Set(['ACCEPTED', 'SUBMITTING', 'PROVIDER_ACCEPTED', 'RECONCILING', 'RETRY_PENDING'])

// 超时只用来触发人工核查，不代表失败：到点仍在途就联系平台，不要重新下单（§5.1 不承诺固定时长）
async function waitForFulfilled(client, clientOrderNo, { timeoutMs = 20 * 60 * 1000, intervalMs = 5000 } = {}) {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    const d = await client.invoke('/openapi/v1/order/query', { clientOrderNo })
    if (d.orderStatus === 'FULFILLED') {
      // 极少数情况下履约时卡信息尚未下发：联系平台核查，不要重新下单
      if (!d.iccid) throw new Error('已履约但缺 ICCID，请联系平台核查')
      return d
    }
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

/** timeoutMs 只用来触发人工核查，不代表失败：到点仍在途就联系平台，不要重新下单 */
public JsonNode waitForFulfilled(XpinEsimClient client, String clientOrderNo,
                                 long timeoutMs, long intervalMs) throws Exception {
    long deadline = System.currentTimeMillis() + timeoutMs;
    while (System.currentTimeMillis() < deadline) {
        JsonNode d = client.invoke("/openapi/v1/order/query", Map.of("clientOrderNo", clientOrderNo));
        String status = d.get("orderStatus").asText();
        if ("FULFILLED".equals(status)) {
            // 极少数情况下履约时卡信息尚未下发：联系平台核查，不要重新下单
            if (d.get("iccid").isNull()) throw new IllegalStateException("已履约但缺 ICCID，请联系平台核查");
            return d;
        }
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
| `externalUserId` \| `iccid` \| `productCode` | string | **恰好一个** | 范围选择器。长度：`externalUserId` `[1, 128]`、`iccid` `[10, 32]`、`productCode` `[1, 160]` |
| `pageSize` | number | 否 | `[1, 100]`，默认 20 |
| `pageToken` | string | 否 | `[1, 2048]`，上一页返回的 `nextPageToken` |

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
        "orderNo": "XE1788864310025C4C7895AE2",
        "clientOrderNo": "ev-t25-1788864309531",
        "externalUserId": null,
        "productCode": "XP01791078F286D359D6790456",
        "orderStatus": "FULFILLED",
        "iccid": "89000000000000000134",
        "profileStatus": "ENABLED",
        "fulfilledAt": "2026-09-08T10:46:30.537Z",
        "packageStartTime": "2026-09-08T10:45:16.000Z",
        "packageEndTime": "2027-09-08T10:45:15.000Z",
        "latestActivationTime": "2027-09-08T10:45:15.000Z",
        "product": {
          "productCode": "XP01791078F286D359D6790456",
          "productName": "CIS 1GB 365D",
          "tagName": "CIS",
          "volume": "1024.000000",
          "dataLimited": "Y",
          "validity": 365,
          "expireDay": 365,
          "cardType": "eo1",
          "productType": "DATA_PACKAGE",
          "mccList": [
            "282",
            "401",
            "255",
            "283",
            "437",
            "434",
            "257",
            "436",
            "259",
            "250",
            "400"
          ],
          "attributes": null,
          "settlementPrice": "0",
          "retailPrice": "0",
          "currency": "USD",
          "configVersion": 4
        }
      },
      {
        "orderNo": "XE178876992006393150AC788",
        "clientOrderNo": "demo-open-1788769918662",
        "externalUserId": null,
        "productCode": "XP01791078F286D359D6790456",
        "orderStatus": "FULFILLED",
        "iccid": "89000000000000000131",
        "profileStatus": "ENABLED",
        "fulfilledAt": "2026-09-07T08:32:08.482Z",
        "packageStartTime": "2026-09-07T08:32:03.000Z",
        "packageEndTime": "2027-09-07T08:32:02.000Z",
        "latestActivationTime": "2027-09-07T08:32:02.000Z",
        "product": {
          "productCode": "XP01791078F286D359D6790456",
          "productName": "CIS 1GB 365D",
          "tagName": "CIS",
          "volume": "1024.000000",
          "dataLimited": "Y",
          "validity": 365,
          "expireDay": 365,
          "cardType": "eo1",
          "productType": "DATA_PACKAGE",
          "mccList": [
            "282",
            "401",
            "255",
            "283",
            "437",
            "434",
            "257",
            "436",
            "259",
            "250",
            "400"
          ],
          "attributes": null,
          "settlementPrice": "0",
          "retailPrice": "0",
          "currency": "USD",
          "configVersion": 2
        }
      }
    ],
    "hasMore": false,
    "nextPageToken": null
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554467544
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
要一次列出这张卡能续的全部商品，用 [§4.8](#48-可续订商品-iccidrenewproducts) —— 列表里的每个商品，本接口（不传 `externalUserId` 时）都判 `allowed: true`；反过来不一定：本接口判可续、但当前不可下单的商品不会出现在 §4.8 里。

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
| `reasons` | string[] | 不可续的原因；`allowed:true` 时为 `[]`。`CARD_NOT_FOUND` / `PRODUCT_NOT_FOUND` 出现时是唯一一条（不再评估其余条件），其余情况列出**全部**原因 |
| `iccid` / `productCode` | string | 回显 |

> 这个接口**不用错误码表达"不可续"**：卡不存在也是 `200`，判定在 `allowed:false` + `reasons:["CARD_NOT_FOUND"]`。

**示例响应**

```json
// 可续
{"data":{"allowed":true,"reasons":[],"iccid":"89000000000000000134","productCode":"XP01791078F286D359D6790456"},
 "code":200,"msg":"ok","t":1788495680855}

// 不可续
{"data":{"allowed":false,"reasons":["DAYPASS_IN_FORCE"],"iccid":"89000000000000000435","productCode":"XP0909F3C1C8A9D7F97456F799"},
 "code":200,"msg":"ok","t":1788505549000}
```

#### `reasons` 全表（按性质分类，这是接入方最需要的一张表）

| reason | 含义 | 性质 |
|---|---|---|
| `CARD_NOT_FOUND` | 卡不存在或不属于你 | ❌ 终局 |
| `PRODUCT_NOT_FOUND` | 目标商品不存在、未发布给你，**或与这张卡不兼容** | ❌ 终局 |
| `PRODUCT_OFF_SALE` | 目标商品已下架 | ⚠️ 可恢复（换商品或等上架） |
| `CARD_NOT_RENEWABLE` | 上游明确这张卡不可续 | ❌ 终局 |
| `CARD_TYPE_MISMATCH` | 目标商品的卡型与这张卡不同。极少数卡型信息缺失的卡也会报它，此时会与 `CARD_CAPABILITY_UNKNOWN`、`CARD_TYPE_RULE_NOT_PUBLISHED` 同时出现 —— 请联系平台 | ❌ 终局（续订永不换卡型） |
| `RENEW_WINDOW_EXPIRED` | 续订窗口已过 | ❌ 终局 |
| `ORDER_NOT_RENEWABLE` | 当前套餐没有续订截止时间 | ❌ 终局 |
| `EXTERNAL_USER_ID_CONFLICT` | 传入的 `externalUserId` 与卡上绑定的不一致 | ❌ 终局（改传对的） |
| **`RENEWABILITY_PENDING_SYNC`** | 平台还没同步到这张卡的可续订性 | 🔄 **稍后重试** |
| **`RENEW_WINDOW_PENDING_SYNC`** | 续订窗口尚未同步 | 🔄 **稍后重试** |
| **`DAYPASS_IN_FORCE`** | 卡上有尚未结束的日包（已下单、尚未激活的也算）时整张卡不可续（用流量包续也不行） | 🔄 日包结束后解除 |
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
> 🔴 **不要把"稍后重试"类的 reason 当成永久拒绝。** 上表标 🔄 的七条都会自行解除：
> `RENEWABILITY_PENDING_SYNC` / `RENEW_WINDOW_PENDING_SYNC` / `CARD_CAPABILITY_UNKNOWN` 随平台同步完成而解除，其余四条随卡上套餐的变动而解除。资格是**随时在变**的量（卡上每多一张单、每结束一张单都会变），
> 所以请**在用户点击的那一刻实时查**，不要缓存资格结果。
>
> ℹ️ 传 `externalUserId` 时预检才能预测归属冲突。**下单要传的 `externalUserId`，
> 预检也要传** —— 不传的话 `EXTERNAL_USER_ID_CONFLICT` 这一条预检看不出来。

#### 规则按卡型差异化（示例）

> ⚠️ 下表是**形态示例**，不是取值清单。具体哪个卡型适用哪条规则由平台目录决定，
> 请勿硬编码卡型码。

| 卡的当前状态 | 用同形态商品续 | 用同卡型的另一形态商品续 |
|---|---|---|
| 卡上有流量包在用（规则：自由续订） | ✅ 可续 | ✅ 可续 |
| 卡上有流量包在用（规则：日包独占） | ✅ 可续 | ❌ `DAYPASS_REQUIRES_EMPTY_CARD` |
| 卡上有日包在用（规则：日包独占） | ❌ `DAYPASS_IN_FORCE` | ❌ `DAYPASS_IN_FORCE` |

**Node.js**

```js
// 上表标 🔄 的那一档：稍后原样重查即可。
const RETRYABLE = new Set([
  'RENEWABILITY_PENDING_SYNC', 'RENEW_WINDOW_PENDING_SYNC',
  'DAYPASS_IN_FORCE', 'DAYPASS_REQUIRES_EMPTY_CARD', 'SERIAL_RULE_SLOT_OCCUPIED',
  'CONCURRENT_ORDER_LIMIT_REACHED', 'CARD_CAPABILITY_UNKNOWN'
])
const TERMINAL = new Set([
  'CARD_NOT_FOUND', 'PRODUCT_NOT_FOUND', 'CARD_NOT_RENEWABLE', 'CARD_TYPE_MISMATCH',
  'RENEW_WINDOW_EXPIRED', 'ORDER_NOT_RENEWABLE', 'EXTERNAL_USER_ID_CONFLICT'
])
// ⚠️ 与 ⚙️ 那两档：原样重试解决不了 —— 要换商品，或联系平台
const ACTION_NEEDED = new Set(['PRODUCT_OFF_SALE', 'CARD_TYPE_RULE_NOT_PUBLISHED', 'PACKAGE_TYPE_NOT_CLASSIFIABLE'])

// externalUserId 要透传：不传的话预检看不出归属冲突
async function checkRenewable(client, iccid, productCode, externalUserId) {
  const body = { iccid, productCode }
  if (externalUserId) body.externalUserId = externalUserId   // 可选字段为空就不要放进去
  const d = await client.invoke('/openapi/v1/iccid/renew', body)
  if (d.allowed) return { allowed: true, reasons: [], retryable: false }
  // ⚠️ 未知 reason **不要当成终局**。平台会新增 reason，把没见过的一律判死，
  // 会把一个本可恢复的拒绝在你的 UI 上变成永久禁用。按"稍后再试"处理并上报。
  const unknown = d.reasons.filter((r) => !RETRYABLE.has(r) && !TERMINAL.has(r) && !ACTION_NEEDED.has(r))
  if (unknown.length) report('unknown renew reason', unknown)
  return {
    allowed: false,
    reasons: d.reasons,
    // 只要有一条终局、或需要换商品 / 联系平台的 reason，原样重试就没有意义
    retryable: d.reasons.every((r) => !TERMINAL.has(r) && !ACTION_NEEDED.has(r))
  }
}
```

**Java**

```java
// 原样重试解决不了的那几条。判据用它而不是 RETRYABLE —— 未知 reason 必须落在"可重试"一侧，
// 把没见过的一律判死会把本可恢复的拒绝在 UI 上变成永久禁用。
private static final Set<String> NOT_RETRYABLE = Set.of(
        // ❌ 终局
        "CARD_NOT_FOUND", "PRODUCT_NOT_FOUND", "CARD_NOT_RENEWABLE", "CARD_TYPE_MISMATCH",
        "RENEW_WINDOW_EXPIRED", "ORDER_NOT_RENEWABLE", "EXTERNAL_USER_ID_CONFLICT",
        // ⚠️ / ⚙️：要换商品或联系平台
        "PRODUCT_OFF_SALE", "CARD_TYPE_RULE_NOT_PUBLISHED", "PACKAGE_TYPE_NOT_CLASSIFIABLE");

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
    boolean retryable = !allowed && reasons.stream().noneMatch(NOT_RETRYABLE::contains);
    return new RenewCheck(allowed, reasons, retryable);
}
```

---

### 4.8 可续订商品 `/iccid/renewProducts`

**只读接口，不产生订单。** 一次拿到这张卡**此刻能续的全部商品**，用来渲染"选择续订套餐"的页面。

**请求**

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `iccid` | string | **是** | 长度 `[10, 32]` |

**响应 `data`**：与 `/products/list`（[§4.1](#41-商品列表-productslist)）**同一份结构**，全部结果作为一页返回。

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 此刻能续的商品，商品对象与 §4.1 **逐字段相同**，按 `productCode` 排序。一个都不能续时为 `[]` |
| `hasMore` | boolean | 恒为 `false`：本接口不分页 |
| `pageNo` | number | 恒为 `1` |
| `pageSize` | number | 本次返回的商品数，即 `list` 的条数（可能为 `0`） |
| `language` | string | 同 §4.1，当前恒为 `en` |

| 情况 | 返回 |
|---|---|
| 卡在，有能续的商品 | `200`，`list` 为这些商品 |
| 卡在，一个都不能续 | `200`，`list: []` |
| 卡不存在或不属于你 | `404 CARD_NOT_FOUND`（与 `/iccid/profile` 相同） |

**列表里是哪些商品**

- **候选**：发布给你、当前可下单的在售商品里，与这张卡**卡型相同**的那些。续订永不换卡型，别的卡型的商品不会出现。
- **结论与 §4.7 同源**：一个候选商品出现在列表里，当且仅当 `/iccid/renew`（不传 `externalUserId`）对它返回 `allowed: true`。两个接口不会给出相反的答案。
- **不返回原因**：某个商品为什么不在列表里，用 §4.7 对它查 `reasons`。

**示例响应**

```json
// 卡上有流量包在用、卡型规则为"日包独占"：只能续流量包
{"data":{"list":[{"productCode":"XP2D6285459B2CB858B2E52390","productName":"Europe 365D 5GB","tagName":"Europe",
  "volume":"5120.000000","dataLimited":"Y","validity":365,"expireDay":365,"cardType":"ep3","productType":"DATA_PACKAGE",
  "mccList":["276","232","206","218","248","219","244","280","230","238","284","262","216","295","247","222","202","208","272","278","270","268","259","226","294","242","204","246","214","231","293","220","228","240","255","234","235","274"],
  "attributes":null,"settlementPrice":"0","retailPrice":"0","currency":"USD","configVersion":1}],
  "hasMore":false,"pageNo":1,"pageSize":1,"language":"en"},"code":200,"msg":"ok","t":1791543582196}

// 一个都不能续（例：卡上的日包还在用）
{"data":{"list":[],"hasMore":false,"pageNo":1,"pageSize":0,"language":"en"},"code":200,"msg":"ok","t":1791543582983}

// 卡不存在或不属于你
{"data":{},"code":404,"msg":"CARD_NOT_FOUND","t":1791543583688}
```

> 🔴 **列表里有，不等于一定下得了单。** 边界与 §4.7 的预检相同：受理时还会扣你的预付费余额
> （`402 BALANCE_LIMIT_REACHED`）；本接口**不收 `externalUserId`**，看不出你下单时要传的终端用户
> 与卡上绑定的是否一致（`EXTERNAL_USER_ID_CONFLICT`）；查询与下单之间，卡上的套餐也可能变化。
> 下单侧照样要处理 `409 RENEW_NOT_ALLOWED` 与 `402`。
>
> 🔴 **不要缓存。** 卡上每多一张单、每结束一张单，答案都会变。请在用户打开续订页的那一刻查。
>
> ℹ️ **与 §4.7 怎么配合**：列出"能续哪些"用本接口；用户想续某个商品却不能续、需要告诉他原因时，
> 用 §4.7 对那个商品查 `reasons`。新卡刚建成、续订信息还没同步的那段时间，本接口返回空列表，
> §4.7 给出的是"稍后重试"类的 reason。
>
> ℹ️ 本接口占用**共用配额**（[§2.3](#23-认证检查顺序)），不在免限接口之列。

**Node.js**

```js
// 渲染"选择续订套餐"页：列表里每一项的 productCode 都可以直接拿去 /order/renew。
// 卡不存在或不属于你时，invoke 按 404 CARD_NOT_FOUND 抛错。
async function renewOptions(client, iccid) {
  const d = await client.invoke('/openapi/v1/iccid/renewProducts', { iccid })
  // [] = 此刻一个都不能续。要告诉用户原因，对他想续的商品调 /iccid/renew 看 reasons。
  return d.list
}
```

**Java**

```java
// 卡不存在或不属于你时，invoke 按 404 CARD_NOT_FOUND 抛错。
public List<JsonNode> renewOptions(XpinEsimClient client, String iccid) throws Exception {
    JsonNode d = client.invoke("/openapi/v1/iccid/renewProducts", Map.of("iccid", iccid));
    List<JsonNode> products = new ArrayList<>();
    d.get("list").forEach(products::add);   // 空列表 = 此刻一个都不能续
    return products;
}
```

---

### 4.9 续订下单 `/order/renew`

**需要 `esim.write` scope。** 给**已有的卡**追加套餐，**不换卡**（ICCID 不变）。卡的激活码以 `/iccid/profile` 的 `delivery.activationCode` 为准。

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
    "orderNo": "XE17895552527309526B1A7E1",
    "clientOrderNo": "docrenew-1789555252454",
    "iccid": "89000000000000000712",
    "productCode": "XP6EE2B3DABB39D0D1A121446C",
    "orderStatus": "ACCEPTED",
    "accepted": true,
    "replayed": false
  },
  "code": 200,
  "msg": "ok",
  "t": 1789555252743
}
```

- 续订单有**独立于开卡单的 `orderNo`**，`iccid` 保持不变。
- 幂等语义与开卡完全相同（同键同内容 → `replayed:true`；同键异内容 → `409`）。
  `orderStatus` 同样**不要断言恒为 `ACCEPTED`**，理由见 [§4.4](#44-开卡下单-ordercreate)。
- 不可续时 → `409 RENEW_NOT_ALLOWED`。卡不存在 / 不属于你、商品不存在 / 已下架 / 当前不可下单也都是这个码（本接口不返回 `404`），具体原因用 §4.7 查。
- 🔴 **续订与开卡一样是受理即扣款。** 平台在受理的同一个事务里就按该商品的结算价从你的
  预付费余额扣钱；订单落 `FAILED` 或 `REFUNDED` 时自动退回，中间态不动钱。
- 🔴 **余额不足 → `402 BALANCE_LIMIT_REACHED`**，整单被拒、一分钱不扣。**不要重试** ——
  在充值之前每一次重试都是同一个结果。请为续订预留额度，并像开卡一样处理这个码。

**Node.js —— 完整续订流程**

```js
async function renew(client, { iccid, productCode, clientOrderNo, externalUserId }) {
  // clientOrderNo 必须有值，理由同 §4.4（它既是幂等键的来源，也是之后的查询键）
  if (!clientOrderNo) throw new Error('clientOrderNo is required')
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
  // `renew-` 占 6 个字符：clientOrderNo 取 2–90 个字符，拼出来的键才落在 [8, 96]
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
    Objects.requireNonNull(clientOrderNo, "clientOrderNo is required");   // 理由同 §4.4
    // 1) 资格实时查，不缓存 —— 卡上每多一张单、每结束一张单，答案都会变；externalUserId 要透传
    RenewCheck check = checkRenewable(client, iccid, productCode, externalUserId);
    if (!check.allowed()) {
        throw new IllegalStateException("不可续订: " + String.join(",", check.reasons()));
    }
    // 2) 受理。⚠️ 预检通过不等于受理一定成功：这里仍可能拿到 409 RENEW_NOT_ALLOWED
    //    或 402 BALANCE_LIMIT_REACHED（余额不足，**不要重试**）。
    Map<String, Object> req = new HashMap<>();      // 不用 Map.of：它对 null 值抛 NPE
    req.put("iccid", iccid);
    req.put("productCode", productCode);
    req.put("idempotencyKey", "renew-" + clientOrderNo);   // clientOrderNo 取 2–90 个字符，键才落在 [8, 96]
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

### 4.10 卡档案 `/iccid/profile`

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
| `delivery.latestActivationTime` | string\|null | 套餐的**激活期限**：过了这个时刻仍未激活，套餐作废 |
| `delivery.installTime` | string\|null | 这张 profile 被装到设备上的时刻。同一张卡可被多次安装，该值是**当前报告值**。⚠️ 见下方说明 |
| `delivery.imsi` / `delivery.msisdn` | string\|null | IMSI / MSISDN |
| `profile.status` | string | 生命周期：`RELEASED` → `DOWNLOADED` → `INSTALLED` → `ENABLED` ↔ `DISABLED` → `DELETED`；另有 `UNKNOWN`：平台尚未收到这张卡的 Profile 状态（**新卡在上游首次上报之前都是此值**，卡一直未安装时可能一直如此），或上游给了平台不认识的值。共 **7** 个取值 |
| `profile.statusTime` | string | 平台**观测到**该状态的时刻（平台时钟，**不是**上游报告的变更时刻；新卡上是履约投影时刻） |
| `profile.changeVersion` | number | 状态变更版本，**单调递增**，可用于丢序检测 |
| `renewal.renewable` | boolean\|null | 可续订性；`null` = 未同步 |
| `renewal.expirationTime` | string\|null | 续订截止时间。**开卡履约完成即可读到**，不必等后续刷新 |
| `renewal.activePackages` | number | 这张卡当前占用的套餐槽位数（**含尚未履约的在途订单**，见下方警告） |
| `renewal.maxConcurrentPackages` | number\|null | 并发上限。`null` 有两种含义 —— 该卡型未声明上限，或平台尚未同步到该卡型能力，**本接口无法区分**。请勿据此自行判断能否续订，那个问题请用 §4.7 |
| `renewal.evidenceFresh` | boolean | 续订证据是否仍在平台的新鲜度窗口内。**仅供展示，不参与续订判定**；能不能续以 §4.7 为准 |
| `renewal.evidenceSyncedAt` | string\|null | 证据同步时刻，取自**卡型能力**的刷新时刻（逐卡跟进：同一卡型的卡通常相同，刷新后的短时间内可能不一致），与 `usage.observedAt` 不是一回事 |
| `usage` | object\|null | 用量；`null` 见 §4.11 |
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
> ⚠️ **`usage` 与上面五个字段取自同一张主导订单，但更新节奏不同。** 五个字段随订单状态即时变化；
> `usage` 要等下一轮用量刷新才切到新的主导订单，若新主导订单处于不再刷新用量的状态，会一直保留上一单的最后读数。
> 所以多套餐卡上两者**仍可能不是同一张单** —— 请比对 `usage.orderNo`，不要默认它们同属一单
> （例如"拿 `packageStatus` 判激活、拿 `usage` 显示剩余"可能把 A 单的状态和 B 单的流量拼在一起）。
>
> ⚠️ **`delivery.installTime` 为 `null` 不代表卡没装上。** 这个时刻只来自上游的一次性推送，
> 没有轮询兜底 —— 卡已经是 `ENABLED` 而它仍为 `null` 是正常的稳定状态。
> **不要用它判断卡有没有装上**，那个问题请读 `profile.status`（`DOWNLOADED` / `INSTALLED` /
> `ENABLED` / `DISABLED` 都表示 Profile 已到达设备）。

**示例响应**

```json
{
  "data": {
    "iccid": "89000000000000000134",
    "externalUserId": null,
    "cardType": "eo1",
    "delivery": {
      "activationCode": "LPA:1$smdp.example.com$D0C23496B79C4179A7FA57E7EXAMPLE",
      "latestActivationTime": "2027-09-08T10:45:15.000Z",
      "installTime": null,
      "imsi": "450000000000134",
      "msisdn": "850000000000134"
    },
    "profile": {
      "status": "ENABLED",
      "statusTime": "2026-09-08T10:58:24.005Z",
      "changeVersion": 1
    },
    "renewal": {
      "renewable": true,
      "expirationTime": "2028-09-07T10:45:15.000Z",
      "activePackages": 1,
      "maxConcurrentPackages": 99,
      "evidenceFresh": true,
      "evidenceSyncedAt": "2026-09-16T10:18:45.929Z"
    },
    "usage": {
      "orderNo": "XE1788864310025C4C7895AE2",
      "dataTotalBytes": 1073741824,
      "dataUsedBytes": 0,
      "dataRemainBytes": 1073741824,
      "refuelingTotalBytes": null,
      "qtaConsumptionBytes": null,
      "observedUnit": "MB",
      "observedAt": "2026-09-16T10:25:39.378Z"
    },
    "packageStatus": "NOT_ACTIVATED",
    "packageStartTime": "2026-09-08T10:45:16.000Z",
    "packageEndTime": "2027-09-08T10:45:15.000Z"
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554467901
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
const qrPayload = p.delivery.activationCode   // 整串生成二维码给用户；可能为 null，先判空
```

```java
JsonNode p = client.invoke("/openapi/v1/iccid/profile", Map.of("iccid", iccid));
JsonNode ac = p.path("delivery").path("activationCode");
String qrPayload = ac.isTextual() ? ac.asText() : null;   // 为 null 时 asText() 会得到字符串 "null"
```

---

### 4.11 用量查询 `/order/usage`

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
    "iccid": "89000000000000000134",
    "usage": {
      "orderNo": "XE1788864310025C4C7895AE2",
      "dataTotalBytes": 1073741824,
      "dataUsedBytes": 0,
      "dataRemainBytes": 1073741824,
      "refuelingTotalBytes": null,
      "qtaConsumptionBytes": null,
      "observedUnit": "MB",
      "observedAt": "2026-09-16T10:25:39.378Z"
    }
  },
  "code": 200,
  "msg": "ok",
  "t": 1789554468075
}
```

> **不限量套餐没有总量。** `products/list` 里 `dataLimited: "N"` 的套餐，
> `dataTotalBytes` 与 `dataRemainBytes` 恒为 `null`，只有 `dataUsedBytes` 是数字。
> 这不是"取数失败"，是这类套餐本来就没有这个概念 —— 不要把它当异常重试。

> 🔴 **这是 `orderNo` 那一单的用量，不是这张卡的合计。** 一张卡上可以同时有多个套餐
> （上限见 `/cardType` 的 `capability.maxRenewCount`），这里返回的是**主导订单**（即 `/iccid/profile`
> 报 `packageStatus` 的那一单）的读数；主导订单刚切换时可能仍是上一单的读数。`orderNo` 就是说明这件事的字段。
> 只读 `dataRemainBytes` 的调用方在多套餐卡上会得到一个**偏小的数** —— 它是这一单还剩多少，不是这张卡还能用多少。

> **`usage: null` 的成因，参考 `cardType` 的 `capability.supportUsageQuery`**：
> - `supportUsageQuery: true` 但 `usage` 为 `null` → 还没有样本，稍后再查。
> - `supportUsageQuery: false` → 这个卡型**不做实时测量**。⚠️ 但这**不等于恒为 `null`**：
>   套餐未激活或已用尽时，仍会返回一组由套餐状态推导出的数字（未激活恒 `used=0`，
>   用尽恒 `remain=0`）；真正走量期间没有新读数 —— 此时 `usage` 仍是激活前推导出的那组数字
>   （如 `dataUsedBytes: 0`），`observedAt` 停在激活前，**不代表实际用量**。
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

### 4.12 退款申请 `/order/refund`

对一张**已履约**的订单发起退款申请。scope `esim.write`。

⚠️ **这个接口只受理，不动钱。** 它落一张退款单并返回 `REQUESTED`；金额要等平台确认这一单确实已退之后才退回你的预付费余额。确认由平台的定期核实流程做出，**不是同步的**，端到端时延取决于上游何时把这一单标记为已退 —— **不要把固定时限当作失败判据**（退款单不会因超时自动失败）；可以设一个宽松的人工跟进时限，超过后联系平台。

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
| `clientOrderNo` | string | 是 | 1–100 字符。你的业务单号（首尾空白会被去掉，比较不区分大小写）。**不引入第二种订单号** |
| `idempotencyKey` | string | 是 | 8–96 字符。与 `/order/create` 同族的重放保护（参与哈希的内容不同：这里是 `clientOrderNo` + `reason`） |
| `reason` | string | 否 | 1–512 字符。退款原因，落在平台退款单上供人工查阅。不需要时请省略（传 `null` 会被参数校验拒绝；省略与传 `""` 视为不同的请求内容） |

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

| `code` | `msg` | 含义与处置 |
|---|---|---|
| 404 | `ORDER_NOT_FOUND` | 你名下没有这个 `clientOrderNo` |
| 409 | `ORDER_NOT_REFUNDABLE` | 这一单现在不能退：尚未履约（处理中的订单款项已在受理时预扣，失败会自动退回，履约后才能申请退款）、已失败（款项已自动退回）、已退款、或受理于预付费上线之前（没有可退的计价信息）。查 `/order/query` 看当前状态 —— **改参数改不出来** |
| 409 | `REFUND_ALREADY_IN_PROGRESS` | 这一单已有一张在途退款单。等它出结论（`ORDER_REFUND_RESULT`）之后才能再发起 |
| 409 | `IDEMPOTENCY_KEY_REUSED` | 同一个 `idempotencyKey` 之前用过、但内容不同（`reason` 也算内容） |
| 400 | `<字段> required` / `<字段> length error [a-b]` | `clientOrderNo` / `idempotencyKey` 缺失或为空串（`required`）、纯空白（`length error`，先去首尾空白再计长度）。这是 §3 的通用参数校验 |
| 503 | 其它 `msg` | 平台侧临时故障。用**同一个** `idempotencyKey` 重新签名后重试是安全的（已建单的会以 `replayed: true` 返回）；持续出现请联系平台 |

**示例响应**

```json
{
  "data": {
    "refundNo": "XR17895552840796EAFB9CA6C",
    "orderNo": "XE17895552527309526B1A7E1",
    "clientOrderNo": "docrenew-1789555252454",
    "refundStatus": "REQUESTED",
    "amount": "0.100000",
    "currency": "USD",
    "accepted": true,
    "replayed": false
  },
  "code": 200,
  "msg": "ok",
  "t": 1789555284083
}
```

**幂等语义**：同一个 `idempotencyKey` + 同样的内容重发 → 返回同一张退款单 + `replayed: true`，**不会退两次钱**。一张订单可以有多张历史退款单（第一次被拒之后可以再申请），但**同时只能有一张在途**。

被拒（`REJECTED`）之后再次申请，必须使用**新的** `idempotencyKey`：原键原样重发只会重放出那张 `REJECTED` 单（`replayed: true`），原键改了 `reason` 则是 `409 IDEMPOTENCY_KEY_REUSED`。

⚠️ **没有"撤销退款申请"的接口。** 一张 `REQUESTED` 的单只有两个出口：平台确认（`CONFIRMED`）或平台拒绝（`REJECTED`）。改主意了请联系平台。

### 4.13 账户余额与充值记录 `/account/balance`

查询**你自己**（当前凭证所属分销商、当前环境）的预付费余额，以及平台为你入账的充值记录（分页，最新的在前）。scope `esim.read`。

商户身份只取自签名凭证，请求里**无法**指定别的分销商。

**入参**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `pageNo` | integer | 否 | 充值记录页码，默认 1，1–100000 |
| `pageSize` | integer | 否 | 每页条数，默认 25，1–100 |

**响应**

| 字段 | 类型 | 说明 |
|---|---|---|
| `hasAccount` | boolean | 你是否已开通预付费余额账户。`false` 时 `available` 与 `currency` 为 `null`，**不是余额为 0** —— 请联系平台开通 |
| `available` | string\|null | 当前余额，十进制字符串（6 位小数）。开通了透支额度时**可能为负数**，它不等于"还能下多少单"。下单受理时从这里扣；订单失败或退款确认后退回这里 |
| `currency` | string\|null | 余额币种 |
| `creditHistory.list[]` | array | 本页充值记录，按入账时间倒序 |
| `creditHistory.list[].rechargeNo` | string\|null | 平台充值单号，对账、沟通用它 |
| `creditHistory.list[].amount` | string\|null | 入账金额，正数十进制字符串 |
| `creditHistory.list[].currency` | string\|null | 入账币种 |
| `creditHistory.list[].creditedAt` | string | 入账时刻，ISO-8601 UTC |
| `creditHistory.hasMore` | boolean | 是否还有下一页 |
| `creditHistory.pageNo` / `pageSize` | integer | 回显本页分页参数 |
| `creditHistory.totalCount` | integer | 充值记录总条数，没有记录时为 0 |
| `creditHistory.totalPages` | integer | 按 `pageSize` 计的总页数，没有记录时为 0 |

> ℹ️ 只包含**充值入账**，不包含下单扣款与退款退回 —— 那些请看 `/order/list`。
>
> ℹ️ 充值记录从平台开始逐笔记录充值之日起才有；更早的入账只体现在余额里，不在 `list` 与 `totalCount` 中。
>
> ℹ️ 余额与充值记录是两次读取，不是同一时刻的快照：恰好在两次读取之间入账的一笔充值，
> 可能已出现在记录里而余额尚未体现（或相反），稍后重查即可一致。`pageNo` 超过总页数时
> 返回空 `list`，不是错误。

**拒绝**

| `code` | `msg` | 含义与处置 |
|---|---|---|
| 400 | `ACCOUNT_PAGE_NO_INVALID` | `pageNo` 过了 §3 的通用参数校验、但不是 1–100000 的整数：数字 `0`，或带小数的数字字符串（如 `"1.5"`）。负数、大于 100000、非数字走通用参数校验；JSON 数字形式的小数（如 `1.5`）在验签环节即被拒为 `400 CANONICAL_NUMBER_UNSUPPORTED` |
| 400 | `ACCOUNT_PAGE_SIZE_INVALID` | `pageSize` 同理：数字 `0`，或带小数的数字字符串（如 `"10.5"`） |

**示例响应**

```json
{
  "data": {
    "hasAccount": true,
    "available": "83.250000",
    "currency": "USD",
    "creditHistory": {
      "list": [
        { "rechargeNo": "RC-20261001-0001", "amount": "500.000000", "currency": "USD", "creditedAt": "2026-10-01T02:00:00.000Z" }
      ],
      "hasMore": true,
      "pageNo": 1,
      "pageSize": 1,
      "totalCount": 3,
      "totalPages": 3
    }
  },
  "code": 200,
  "msg": "ok",
  "t": 1791900000000
}
```

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

上图是主路径，不是全部跃迁：上游明确拒单或遇到不可重试的错误时，`SUBMITTING` 会直接落 `FAILED`；
履约通知也可能在 `SUBMITTING` / `RETRY_PENDING` / `RECONCILING` 期间到达，订单直接变成 `FULFILLED` ——
不要假设一定先看到 `PROVIDER_ACCEPTED`。

| 状态 | 含义 | 你该做什么 |
|---|---|---|
| `ACCEPTED` | 平台已受理，尚未提交电信运营商 | 继续等 |
| `SUBMITTING` | 正在提交电信运营商 | 继续等 |
| `PROVIDER_ACCEPTED` | 上游已受理，等履约回调 | 继续等；**长时间不动不是你的参数问题**，请联系平台核查 |
| `RECONCILING` | 提交/对账阶段异常，正在核对 | 继续等，超时联系平台 |
| `RETRY_PENDING` | 等待重试 | 继续等 |
| **`FULFILLED`** | **已履约，`iccid` 可用** | 取激活码交付用户 |
| **`FAILED`** | **这一单没有出卡** | **停止轮询**。需要出卡就重新下一单（用**新的** `idempotencyKey` 与 `clientOrderNo`）。原因需联系平台核查 |
| **`REFUNDED`** | **这一单已退款**（你提交的退款申请被确认，见 §4.12） | 停止轮询。这一单的套餐已作废（续订单退款只作废那一笔续订，卡上其它套餐不受影响） |

> 🔴 **轮询必须同时判 `FAILED`。** 只等 `FULFILLED` 的循环会在失败单上一直转到超时。
> `FULFILLED` / `FAILED` / `REFUNDED` 都按终态处理，收到任一个就停止轮询。
>
> ⚠️ **但请让你的状态写入保持可重入。** 极少数情况下，平台在补齐更晚到达的履约证据后
> 会把一张 `FAILED` 或 `REFUNDED` 的订单改回 `FULFILLED`（并按原结算价重新扣款）；反过来，上游事后报告
> 履约失败时，`FULFILLED` 也会被更正为 `FAILED`（款项退回余额）。这两种更正**通常不发事件** —— 只有这一单
> 此前从未发出过履约事件时才会补发一条（开卡单 `ORDER_OPEN_RESULT`，续订单 `ORDER_RENEW_RESULT`），
> 所以多半只能靠 `/order/query` 发现。
> **不要在收到 `FAILED` 时删除本地订单记录或让它变得不可更新** —— 否则你会收到一条
> 无处安放的事件，同时有一张已交付却无人认领的卡。
>
> ⚠️ **`REFUNDED` 只出现在你自己提交过退款申请的订单上**（§4.12），不使用退款接口的接入方
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
> `activationCode` 为空而 `iccid` 有值时改用 `/iccid/profile` 取；`iccid` 也为空、或仍取不到时请联系平台，**不要重新下单**。

### 5.3 续订完整流程

```
1. iccid/renewProducts     列出这张卡此刻能续的商品，供用户选择（实时查，不要缓存！）
   或 iccid/renew          对用户选中的商品查资格，不可续时拿到 reasons
2. 商品在列表里 / allowed === true 才继续，否则按 reasons 分类提示用户
3. order/renew             → ACCEPTED + 新的 orderNo（iccid 不变）
4. 轮询 order/query 到终态（FULFILLED 或 FAILED），或等 ORDER_RENEW_RESULT 回调（失败的单不发事件，回调路仍需超时兜底）
5. 断言 iccid 未变
```

> ⚠️ **续订资格有时间窗**：新卡建成后，平台需要先同步上游的续订信息。
> 同步完成前查资格会拿到 §4.7 中标 🔄 的同步类 reason（`RENEWABILITY_PENDING_SYNC` / `RENEW_WINDOW_PENDING_SYNC` /
> `CARD_CAPABILITY_UNKNOWN`），`/iccid/renewProducts` 则返回空列表，按"稍后再试"提示即可。资格还会随卡上套餐的变动实时改变，
> 所以**必须在用户点击的那一刻查**，不要缓存。

### 5.4 退款完整流程

```
1. order/query             确认这一单是 FULFILLED（只有已履约的单能退）
2. order/refund            → REQUESTED + refundNo（此时钱还没退）
3. 二选一：
   a. 等 ORDER_REFUND_RESULT 回调（推荐，见 §6；确认与拒绝都会发）
   b. 轮询 order/query 看 orderStatus 是否变 REFUNDED
      注意：拒绝不改变订单状态，所以这条路只能识别"已确认"，需配合超时
4. refundStatus === "CONFIRMED" → 金额已退回预付费余额，这一单的套餐已作废
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
| `ORDER_RENEW_RESULT` | 续订单落 `FULFILLED` 时 | **同上**（`activationCode` 取自本次续订的履约通知，可能为 `null`；为空时以 `/iccid/profile` 为准） |
| `STATUS_CHANGED` | 卡的 profile 状态真的变化时 | `orderNo` `iccid` `entityType`(恒 `PROFILE`) `previousStatus` `currentStatus` `changedAt` `changeVersion` |
| `ORDER_REFUND_RESULT` | 你提交的退款申请**有了结论**时（确认或拒绝），每张退款单一次 | `orderNo` `clientOrderNo` `refundNo` `idempotencyKey` `refundStatus` `orderStatus` `amount` `currency` `reason` |

> 🔴 **`data` 里的这几个字段可能是 `null`，请按可空解析**：`ORDER_OPEN_RESULT` /
> `ORDER_RENEW_RESULT` 的 `clientOrderNo`（下单时没传就是 `null`）、`iccid`、`imsi`、`msisdn`、`activationCode`。
> 上游的履约通知没带卡信息时，订单同样会落 `FULFILLED`，事件里对应字段为 `null`。
> 直接 `qrcode(event.data.activationCode)` 会在这条真实路径上抛异常 ——
> `activationCode` 为 `null` 而 `iccid` 有值时，请改用 §4.10 的 `/iccid/profile` 取；`iccid` 也为空、或仍取不到时请联系平台，**不要重新下单**。

> ⚠️ **`STATUS_CHANGED` 的 `orderNo` 是该 ICCID 当前最新的订单号，用来定位这张卡，
> 不代表引发本次变更的订单。** profile 变更是**卡级**事件，本来就不存在"引发它的那一单"；
> 续订之后这张卡的状态变更会挂在续订单的 `orderNo` 上。**请以 `iccid` 为关联主键。**
>
> `previousStatus` 是平台记录的上一个状态，未必等于你上次收到的 `currentStatus`（例如履约时随通知写入的状态不单独发事件）；`changedAt` 是平台处理这次变更的时刻。判断先后请用 `changeVersion`。

⚠️ **`ORDER_REFUND_RESULT` 对两种结论都会送达，请两种都处理。** `refundStatus: "REJECTED"` 是退款流程一个**成功完成的结论**（平台判定不退），不是投递失败：收到它即可停止等待。`reason` 里是原因，但**它可能为 `null`**（平台未附原因）—— 不要直接 `reason.trim()`，为空时提示用户联系客服即可。

| `refundStatus` | `orderStatus` | `reason` | 含义 |
|---|---|---|---|
| `"CONFIRMED"` | `"REFUNDED"` | `null` | 退款已确认，金额已退回你的预付费余额 |
| `"REJECTED"` | `"FULFILLED"` | 拒绝原因（`string\|null`） | 平台判定不退，订单状态**没有变化**，卡仍然有效 |

`ORDER_REFUND_RESULT` 的 `idempotencyKey` 是你调用 `/order/refund` 时传的幂等键（不是下单时的）；`amount` 是 6 位小数的字符串（如 `"3.200000"`）。

⚠️ **去重请按 `eventId`。** 一张退款单只会收到一次 `ORDER_REFUND_RESULT`（`CONFIRMED` 与 `REJECTED` 都是终态），但同一张**订单**可以收到多次——第一次被拒之后可以再次申请，那是另一张退款单、另一个 `refundNo`，`eventId` 也不同。按 `orderNo` 去重会把后续退款的结论一并滤掉。

### 6.2 请求形状

`POST` 到你的 `callback_url`，`Content-Type: application/json`，**10 秒超时**。

请求头：

| 头 | 值 |
|---|---|
| `x-event-id` | 与 body 的 `eventId` **完全相同**。头不在签名覆盖范围内，幂等以验签通过的 body `eventId` 为准 |
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
{"msg":"success","code":"0000","data":{"imsi":"460000000000433","iccid":"89000000000000000433","msisdn":"850000000000433","orderNo":"XE1788489809764F48C390102","orderStatus":"FULFILLED","productCode":"XP0909F3C1C8A9D7F97456F799","clientOrderNo":"m1whA-1788489808349","activationCode":"LPA:1$smdp-a.example.com$7ADBC7BBD32B47438D0BB3F4EXAMPLE","idempotencyKey":"m1whA-idem-1788489808349"},"eventId":"145b1ba79790575ae1fa22a5b172fb2fc6687a27de71d584","eventType":"ORDER_OPEN_RESULT","timestamp":"2026-09-04T02:43:37.180Z","nonce":"b2106d10-f164-4c89-84d9-4c7887dc7d03","businessType":"ESIM"}
```

```json
// ORDER_RENEW_RESULT
{"msg":"success","code":"0000","data":{"imsi":"460000000000574","iccid":"89000000000000000574","msisdn":"850000000000574","orderNo":"XE1788505564505CF4DE8D310","orderStatus":"FULFILLED","productCode":"XP01791078F286D359D6790456","clientOrderNo":"m1n-C4-DATA-rn-1788504527118","activationCode":"LPA:1$smdp-b.example.com$7ADBC7BBD32B47438D0BB3F4EXAMPLE","idempotencyKey":"m1n-C4-DATA-rnidem-1788504527118"},"eventId":"881c95a391b04045a10b5f91ebecb3512bbb8b99c70584f9","eventType":"ORDER_RENEW_RESULT","timestamp":"2026-09-04T07:06:22.502Z","nonce":"66677339-0820-4926-86aa-53fe1c5bc248","businessType":"ESIM"}
```

```json
// STATUS_CHANGED
{"msg":"success","code":"0000","data":{"iccid":"89000000000000000433","orderNo":"XE1788489809764F48C390102","changedAt":"2026-09-04T02:49:39.026Z","entityType":"PROFILE","changeVersion":1,"currentStatus":"ENABLED","previousStatus":"UNKNOWN"},"eventId":"56e045be3d6c736f968e19c690dd01fc6be016055fcc32bf","eventType":"STATUS_CHANGED","timestamp":"2026-09-04T02:49:43.548Z","nonce":"e154774a-636d-4ee3-b2a4-4b007db57a97","businessType":"ESIM"}
```

```json
// ORDER_REFUND_RESULT —— 确认（reason 为 null）
{"msg":"success","code":"0000","data":{"orderNo":"XE1788489809764F48C390102","clientOrderNo":"m1whA-1788489808349","refundNo":"XR1788692301447A31B7C0D28","idempotencyKey":"m1whA-refund-1788692300112","refundStatus":"CONFIRMED","orderStatus":"REFUNDED","amount":"3.200000","currency":"USD","reason":null},"eventId":"991536ed1ff784d8d2c026c24f0e1f2ebc20f1b5fd93ba74","eventType":"ORDER_REFUND_RESULT","timestamp":"2026-09-13T09:21:44.006Z","nonce":"5f2a1c93-70e8-4a1d-9b6e-2c81d4f0a7b3","businessType":"ESIM"}
```

```json
// ORDER_REFUND_RESULT —— 拒绝（orderStatus 不变，reason 是原因）
{"msg":"success","code":"0000","data":{"orderNo":"XE1788505564505CF4DE8D310","clientOrderNo":"m1n-C4-DATA-rn-1788504527118","refundNo":"XR17886931125503F9E82A146","idempotencyKey":"m1n-refund-1788693111204","refundStatus":"REJECTED","orderStatus":"FULFILLED","amount":"5.000000","currency":"USD","reason":"upstream declined: package already consumed"},"eventId":"a665fa8d571364599d7e4f168e36d3fa268d7da998cfb946","eventType":"ORDER_REFUND_RESULT","timestamp":"2026-09-13T09:38:02.771Z","nonce":"c0913ba7-1d55-4e0a-8f37-6ab2e59d4c10","businessType":"ESIM"}
```

> 🔴 **不要假设键序。** 当前键序为 `msg, code, data, eventId, eventType, timestamp, nonce, businessType`，
> 但它**不是契约的一部分**，平台给信封或 `data` 增加字段，都可能改变对应层级的键序。
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
    # 先转成 bytes 再比：两个 str 里任一含非 ASCII 字符时 compare_digest 会抛 TypeError
    return hmac.compare_digest(expected.encode('ascii'), (sign_header or '').encode('utf-8'))
```

`express.json()` / `@RequestBody Map` / `request.json` 都可以放心用 —— 拿解析后的对象就够了。

**接收端骨架（Express）**

```js
const express = require('express')
const app = express()

// 可以直接用 express.json()：验签不需要原始字节。
app.use('/xpin/webhook', express.json({ limit: '1mb' }))

app.post('/xpin/webhook', async (req, res) => {
  if (!verifyKv1(req.body, req.get('x-sign'), APP_SECRET)) {
    // 验不过就不要回 0000 —— 回了我们会认为已送达，这条事件不再重投。
    return res.json({ code: '9999', msg: 'bad signature' })
  }
  // 幂等键取验签覆盖的 body.eventId（x-event-id 头不在签名范围内）。重投是契约，不是异常。
  const eventId = req.body.eventId
  try {
    // 去重与处理放在同一个事务里（或处理成功之后才记"已处理"）：先记再处理的话，
    // 处理一旦失败，重投会被当成重复直接回 0000，这条事件就永久丢了。
    const isNew = await processOnce(eventId, () => handle(req.body))
    res.json({ code: '0000', msg: isNew ? 'success' : 'duplicate' })
  } catch (err) {
    // 处理失败：不要回 0000，让平台按 §6.4 重投
    res.json({ code: '9999', msg: 'retry later' })
  }
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
| 重投节奏 | **退避**，见下表 |
| 总尝试次数 | **7 次**（首投 + 6 次重投），之后不再重投 |
| 重投时 | `eventId` 与 `data` **不变**；`timestamp` / `nonce` / `sign` **每次重新生成** |
| ACK 之后 | **立即停止重投** |

**退避阶梯**

| 第几次失败 | 下一次重投在 | 距首投约 |
|---|---|---|
| 第 1 次 | 10 秒后 | 10 秒 |
| 第 2 次 | 30 秒后 | 40 秒 |
| 第 3 次 | 60 秒后 | 1 分 40 秒 |
| 第 4 次 | 600 秒后 | 11 分 40 秒 |
| 第 5 次 | 3600 秒后 | 1 小时 12 分 |
| 第 6 次 | 7200 秒后 | **3 小时 12 分** |
| 第 7 次 | — | 不再重投 |

> ⚠️ **上表是最短耗时，不是保证。** 实际重投在到点之后由后台按批处理，积压时会比表上更晚
> —— 只会更晚，不会更早。请按"最终会到、但可能迟"来设计，不要用固定时限判定平台失败。

> 🔴 **7 次全部失败之后事件不再重投，且没有补发入口。** 端点长时间不可用时，请按下方 💡 的方法对账补齐
> —— 尤其是 `ORDER_OPEN_RESULT`，它是激活码唯一的推送通道。

> 💡 **长时间没收到预期事件时**，按事件类型对账补齐：
> - 履约结果：用 `/order/query`（按 `clientOrderNo`）或 `/order/list`（按 `externalUserId` / `iccid` / `productCode`，
>   [§4.6](#46-订单列表-orderlist)）找到订单，再用 `/iccid/profile` 取激活码；
> - Profile 状态：看 `/iccid/profile` 的 `profile.status` 与 `profile.changeVersion`；
> - 退款确认：订单的 `orderStatus` 变为 `REFUNDED`。⚠️ 退款**被拒**（`REJECTED`）在任何查询接口上都看不到，
>   这条事件丢失时只能联系平台。

> 🔴 **幂等键只能是 `eventId`（或头里的 `x-event-id`）**，不能用 `sign`、`nonce` 或 `timestamp`——它们每次投递都会变。
>
> 🔴 **`"0000"` 是字符串，同步接口的 `200` 是数字，两套码本。** 回 `{"code":200}` 或 `{"code":0}`
> 都会被判为失败，导致同一事件在约 3 小时内被重投 6 次，之后不再重投。请务必按字符串 `"0000"` 应答。

> 🔴 **投递不保证顺序。** 一条重投过的事件会排在后面产生的新事件之后到达。
> 同一张卡的 `STATUS_CHANGED` 带**单调递增**的 `data.changeVersion` ——
> 请丢弃 `changeVersion` 不大于你已处理值的事件，否则乱序会让本地状态永久停在旧值。
> `ORDER_OPEN_RESULT` / `ORDER_RENEW_RESULT` 与 `STATUS_CHANGED` 之间同样不保证先后。

> 🔴 **密钥轮换会中断在途事件的投递。** 平台用你凭证**当前**的 `app_secret` 签名，
> 不回落旧密钥；轮换之后，**已在投递队列里的事件不会再发出**（你不会收到一条验不过的
> 通知，而是**什么都收不到**），并且**不会自行恢复**，未送达的事件在有限时间后被放弃。
>
> **请与平台方约定轮换时间窗**，选在没有事件在途的时段。轮换后请核对这段时间内是否有
> 订单只在 `/order/query` 里变成了 `FULFILLED` 却没有收到对应事件 —— 按上文 💡 的方法对账补齐。
>
> ⚠️ **回调地址在事件入队时就已确定。** 修改回调地址之后，已在队列里（含重投中）的事件仍发往旧地址；
> 换地址同样请与平台方约定时间窗。

---

## 7. 沙盒专用：状态模拟 `/sandbox/card/status`

把你自己名下的一张卡（或它的某一笔订单）设成你指定的状态。scope `esim.write`。

> 🔴 **这个接口只在沙盒环境存在。** 生产环境**没有这条路由** —— 调用它得到的是
> `HTTP 200` + `code: 404` 的信封，与一个拼错的路径名无法区分（见 [§3](#3-错误码) 的 `404` 行）。
> 请把它隔离在你的联调代码里，**不要**让它出现在发往生产的调用路径上。

**它解决什么**：沙盒上的卡开出来之后就停在未激活，不会自己走到使用中、用尽或过期。
于是「开卡之后的那一整段」你在沙盒上跑不出来 —— 收不到 `STATUS_CHANGED`，也没法验证
自己的状态机。这个接口让你直接把卡推到想验证的那个状态上。

**它不能做什么**：

- **不能凭空开出一张卡。** 它只能改已经存在的、你名下的卡；开卡仍然只能走
  `/order/create`（[§4.4](#44-开卡下单-ordercreate)）。
- **不能改别人的卡。** 目标按你的凭证归属校验，查不到与"不属于你"返回同一个
  `404 CARD_NOT_FOUND`。
- **不直接动钱。** 它本身不改你的预付费余额；但如果这一单有在途的退款申请，你设的套餐状态会像真实的上游状态一样
  参与退款确认 —— 命中平台的确认条件时，退款会被确认（金额退回余额、订单变 `REFUNDED`、发 `ORDER_REFUND_RESULT`）。

---

### 7.1 可以设成哪些状态

**不是** [§4.10](#410-卡档案-iccidprofile) 里状态取值的全集 —— `packageStatus` 只能设 12 个取值中的 7 个，`profileStatus` 可以设全部 7 个：

| 参数 | 可选值 |
|---|---|
| `packageStatus` | `NOT_ACTIVATED`、`ACTIVATED`、`IN_USE`、`EXHAUSTED`、`EXPIRED`、`ABANDONED`、`TERMINATED` |
| `profileStatus` | `UNKNOWN`、`RELEASED`、`DOWNLOADED`、`INSTALLED`、`ENABLED`、`DISABLED`、`DELETED` |

表以外的值返回 `422`（见 [§7.3](#73-拒绝)）。

#### 只给 `packageStatus` 时，Profile 状态会被一并模拟

设一个套餐状态，平台会推出**一个自洽的上游会推的那几条事件**，而不是只推你字面点到的
那一条。对应关系：

| 你设的 `packageStatus` | 连带把 `profileStatus` 设成 |
|---|---|
| `NOT_ACTIVATED` | `INSTALLED` |
| `ACTIVATED`、`IN_USE`、`EXHAUSTED` | `ENABLED` |
| `EXPIRED`、`ABANDONED`、`TERMINATED` | `DISABLED` |

> ⚠️ **这是平台的模拟策略，不是对上游行为的描述。** 我们并不声称真实上游在套餐进入
> `IN_USE` 时一定会同时推 Profile 变更 —— 这张表是"一个自洽的上游会怎么做"。
> 你的接收端不应把它当成生产环境的时序契约。

**显式传的 `profileStatus` 永远压过上表**，所以要一个不自洽的组合（例如套餐 `IN_USE`
但 Profile `DISABLED`）就两个都传。表里推不出 `DELETED` —— 要它就显式传。

---

### 7.2 调用与返回

**入参**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `iccid` | string | 是 | 10–32 字符。目标卡，必须在你名下 |
| `packageStatus` | string | 否 | 目标套餐状态，取值见 [§7.1](#71-可以设成哪些状态) |
| `profileStatus` | string | 否 | 目标 eSIM Profile 状态，取值见 [§7.1](#71-可以设成哪些状态) |
| `clientOrderNo` | string | 否 | 1–100 字符。指定改哪一笔订单的套餐状态。不传就取这张卡**最近一次履约**的那笔订单 |

`packageStatus` 与 `profileStatus` **至少要给一个**，都不给返回 `400`。

> ⚠️ **续订过的卡请传 `clientOrderNo`。** 一张卡上有多笔订单时，"这张卡的套餐状态"
> 本身是有歧义的；不传就落在最近一次履约的那一笔上，那未必是你想改的那一笔。
>
> 另外：**用量只会为状态生效后将成为这张卡主导订单（§4.10）的那一笔填充**（见下）；
> 不会成为主导订单时 `usage` 返回 `null`，`usageSkippedReason` 是 `ORDER_NOT_GOVERNING`。

**响应**

| 字段 | 类型 | 说明 |
|---|---|---|
| `iccid` | string | 回显目标卡 |
| `orderNo` | string \| null | 实际命中的平台订单号；只改 Profile 状态且这张卡尚无订单时为 `null` |
| `clientOrderNo` | string \| null | 命中订单的业务单号 |
| `simulationId` | string | 32 位十六进制。本次模拟的标识，排查时提供给平台 |
| `accepted` | array | 本次受理的状态变更，每项 `{ kind, eventType }`。`kind` 为 `ORDER_CHANGE`（套餐状态）或 `PROFILE_CHANGE`（Profile 状态）；`eventType` 是平台内部的编号，仅供排查，与 §6.1 的 `eventType` 无关，请勿依赖 |
| `usage` | object \| null | 本次**已经写入**的模拟用量，见下。不填时为 `null` |
| `usageSkippedReason` | string \| null | 没填用量的原因；填了则为 `null` |

**示例响应**

```json
{
  "data": {
    "iccid": "89000000000000000134",
    "orderNo": "XE17895552527309526B1A7E1",
    "clientOrderNo": "docrenew-1789555252454",
    "simulationId": "3f1a9c7e42b84d05a6c1e8f70b23d95c",
    "accepted": [
      { "kind": "ORDER_CHANGE", "eventType": 5 },
      { "kind": "PROFILE_CHANGE", "eventType": 4 }
    ],
    "usage": {
      "dataTotalBytes": 3145728,
      "dataUsedBytes": 1110118,
      "dataRemainBytes": 2035610
    },
    "usageSkippedReason": null
  },
  "code": 200,
  "msg": "ok",
  "t": 1789555284083
}
```

#### 用量

⚠️ **`accepted` 与 `usage` 的语义不同，不要混着读：**

- `accepted` 是**已受理**。状态变更走的是回调链路，落库是异步的（通常几秒）。
- `usage` 是**已生效**。流量数字在上游的回调协议里没有任何载体，所以平台直接写投影，
  返回时它已经能从 `/order/usage` 读到了。

**只有把套餐设成 `ACTIVATED` 或 `IN_USE` 时才填**，因为只有这两个状态下平台原本会去向
上游实查。其余状态一律不填，`usageSkippedReason` 会说明原因：

| 情形 | `usageSkippedReason` | 说明 |
|---|---|---|
| 设成 `NOT_ACTIVATED` / `EXHAUSTED` | `PLATFORM_DERIVE` | 平台按套餐总量自己算（用了 0 / 用满），不需要模拟。商品没有总量时 `EXHAUSTED` 为 `PLATFORM_KEEP` |
| 设成 `EXPIRED` 等终态 | `PLATFORM_KEEP` | 平台保留最后一次读数，覆盖它会毁掉那条语义 |
| 只设了 `profileStatus` | `NO_PACKAGE_STATUS_REQUESTED` | 用量跟着套餐状态走 |
| 目标订单在状态生效后不会成为主导订单 | `ORDER_NOT_GOVERNING` | 用量一卡一份，只记在主导订单名下（例如卡上另有 `IN_USE` 套餐时把另一笔设成 `ACTIVATED`） |
| 设成 `ACTIVATED` / `IN_USE`，但套餐不限量或商品没有配总量 | `PACKAGE_TOTAL_UNKNOWN` | 没有总量就没有合理区间，平台不编一个数字出来 |

填充规则：总量取自该订单**商品配置的套餐总量**（不是随机的）；已用量在区间内随机 ——
`ACTIVATED` 取 0–5%，`IN_USE` 取 5–85%；剩余量 = 总量 − 已用量。

> ⚠️ **模拟的用量是一次快照，不会随时间增长，也可能被覆盖。** 平台仍会按自己的节奏
> 就 `ACTIVATED` / `IN_USE` 的卡向上游查实时流量；查到了就按实测值覆盖你这次写进去的
> 数字。要一个稳定的"跑满了"场景，用 `EXHAUSTED`（平台会算出用满，且那个状态不实查）。

> 🔴 **状态变更的 `code: 200` 是"已受理"，不是"已生效"。**
>
> 平台收到你的请求之后走的是与真实上游回调完全相同的那条链路，状态落库是**异步**的
> —— 通常几秒内完成。**响应里刻意不带任何"新状态"字段**：那会是一个还没发生的事实。
>
> 要确认已经生效，调 `/iccid/profile`（[§4.10](#410-卡档案-iccidprofile)）复核：套餐状态看 `packageStatus`
> （只反映主导订单），Profile 状态看 `profile.status` / `profile.changeVersion`。**不要**假定调用返回即生效。
> （`usage` 是例外，它在返回时已经生效。）

> ⚠️ **每次调用都会重新推送一次，本接口没有 `idempotencyKey`。** 但只有状态**真的变化**时才会落库并发
> `STATUS_CHANGED`：同样的参数连续调两次，第二次通常不产生事件（见 §7.4 第 4 步）。可以重复推送是刻意的
> —— 状态被别的流程改回去之后，你需要能再推一次。

> ⚠️ **模拟出来的状态不是永久的。** 平台仍会按自己的节奏与上游核对这张卡；
> 一段时间之后，上游的真实状态可能把你设的值覆盖掉。请在设定之后**尽快**完成验证，
> 不要把一张模拟状态的卡当成长期夹具。

---

### 7.3 拒绝

| `code` | `msg` | 含义与处置 |
|---|---|---|
| 400 | `SIMULATION_TARGET_REQUIRED` | `packageStatus` 与 `profileStatus` 一个都没给 |
| 404 | `CARD_NOT_FOUND` | 你名下没有这个 `iccid` |
| 404 | `ORDER_NOT_FOUND` | 传了 `clientOrderNo`，但这张卡上没有这笔订单 |
| 409 | `ORDER_NOT_SIMULATABLE` | 命中的订单当前不是 `FULFILLED`（尚未履约、已失败或已退款），或没有可用的上游单号，套餐状态改不了。请用 `clientOrderNo` 指定一笔已履约的订单；只改 `profileStatus` 不受此限 |
| 422 | `PACKAGE_STATUS_NOT_SIMULATABLE` | `packageStatus` 不在 [§7.1](#71-可以设成哪些状态) 的表里 |
| 422 | `PROFILE_STATUS_NOT_SIMULATABLE` | `profileStatus` 不在 [§7.1](#71-可以设成哪些状态) 的表里 |
| 503 | `SIMULATION_NOT_AVAILABLE` | 这个环境没有开启状态模拟。**改请求改不出来**，联系平台 |
| 503 | `SIMULATION_INGRESS_UNCONFIGURED` | 平台侧配置缺失。同上 |
| 503 | `SIMULATION_SUPPLIER_UNAVAILABLE` | 平台侧配置问题。同上 |

以上都是 `HTTP 200` + 信封里的 `code`。所有 `503`（包括上表之外的 `msg`）都与你的请求无关，退避重试并联系平台；重试无害。

---

### 7.4 用它验证回调链路

**`STATUS_CHANGED`（[§6.1](#61-事件类型)）只由 Profile 状态变更产生。** 套餐状态自己
**不是**一个出站事件类型 —— 但由于 [§7.1](#71-可以设成哪些状态) 的连带规则，设一个套餐
状态通常也会带出一次 Profile 变更，因而**间接**产生 `STATUS_CHANGED`。

> ⚠️ 只在 Profile 状态**真的发生了变化**时才会发。卡本来就是 `ENABLED`，再设一次
> `IN_USE` 不会产生新事件（`changeVersion` 也不会前进）—— 那是正确行为，不是丢事件。

一条建议的联调顺序：

1. `/order/create` 开一张卡，等 `ORDER_OPEN_RESULT`。
2. `POST /sandbox/card/status`，`{"iccid": "...", "packageStatus": "IN_USE"}` ——
   一次调用同时验证：套餐状态推进、连带的 `STATUS_CHANGED`、`changeVersion` 去重逻辑，
   以及 `/order/usage` 上出现的模拟流量。
3. `{"iccid": "...", "profileStatus": "DISABLED"}` —— 验证你对 Profile 单独变更的处理。
4. 再调一次第 3 步同样的参数 —— 此时 Profile 没有变化，**不该**再收到事件；验证你没有
   把"没有事件"误判成链路故障。
5. `{"iccid": "...", "packageStatus": "EXHAUSTED"}` —— 验证你对"套餐用尽"的处理。连带规则会把 Profile 设回
   `ENABLED`，所以接在第 3 步之后会再收到一条 `STATUS_CHANGED`（`DISABLED` → `ENABLED`）；`/order/usage` 的剩余量
   会在平台下一轮用量刷新后变成 0。

#### 这个接口**不能**帮你验证的

| | 为什么 |
|---|---|
| **开卡 / 激活码** | 履约事件在上游协议里要求签名，本接口不构造签名。开卡只能走 `/order/create` |
| **续订资格** | 本接口改不了可续订截止时间。但套餐状态参与平台的续订规则（槽位占用、日包在用），设成 `EXHAUSTED` / `EXPIRED` / `ABANDONED` 等可能让 `/iccid/renew` 变为可续 —— 上游并不知道模拟状态，真实的 `/order/renew` 仍可能被拒。续订联调请用真实下单 |
| **用量随时间增长** | 模拟的用量是一次快照，见 [§7.2](#72-调用与返回) |

---

