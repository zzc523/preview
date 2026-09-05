# XPIN eSIM 分销商 OpenAPI 接入说明

> **文档版本：beta**（2026-09-05）
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
   - [2.5 签名客户端](#25-签名客户端)
   - [2.6 自检向量](#26-自检向量)
3. [错误码](#3-错误码)
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
5. [业务流程与状态机](#5-业务流程与状态机)
6. [回调通知](#6-回调通知)

---

## 1. 基础约定

| 项 | 值 |
|---|---|
| Base URL（沙盒环境） | `https://tbetaopenapi.xpin.network` |
| Base URL（生产环境） | 请向平台方获取 |
| 协议 | 全部 `POST`，`Content-Type: application/json` |
| 字符集 | UTF-8 |
| **成功判定** | **看响应体的 `code` 字段，不看 HTTP 状态码** |
| 凭证 | 平台发放的 `app_key` / `app_secret`（需要从平台方获取） |

**所有接口都返回 `HTTP 200`**（少数传输层错误除外），响应体统一为：

```json
{ "data": {...}, "code": 200, "msg": "ok", "t": 1788431707137 }
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `data` | object | 业务数据。失败时为 `{}` |
| `code` | number | **业务码**。`200` 才是成功 |
| `msg` | string | 成功为 `ok`；失败为错误码或字段级提示 |
| `t` | number | 服务端毫秒时间戳 |

> ⚠️ **断言一定要写 `body.code === 200`**。写 `res.status === 200` 会把所有失败都当成功。

HTTP 80 端口的请求会 `301` 跳转到 HTTPS，且 **301 不保留 POST body**——直接用 `https://`。

---

## 2. 认证与签名

每个请求带四个头：

| 请求头 | 说明 |
|---|---|
| `x-api-key` | 你的 `app_key` |
| `x-timestamp` | **毫秒**时间戳（13 位），与服务器时钟偏差需 ≤ **5 分钟**（双向） |
| `x-nonce` | 随机串，匹配 `^[A-Za-z0-9._:-]{8,128}$`，**同一凭证同一 nonce 只能用一次** |
| `x-sign` | HMAC-SHA256 十六进制**小写**，见 2.1 |

### 2.1 签名算法 XPIN-KV1

签的是**解析后的数据**，不是报文字节 —— 所以你的 body 用任何 JSON 库、任何键序、任何缩进
发出都行，也**不要求**你能控制 HTTP 客户端实际发出的字节。被签的对象固定七个字段：

```jsonc
{
  "sigAlg":    "xpin.esim.openapi.kv1",   // 字面量
  "apiKey":    "<x-api-key 的值>",
  "method":    "POST",
  "path":      "/openapi/v1/order/create", // 归一后：重复斜杠折成一个、去尾斜杠、不解码 %XX
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

**以下值一律拒绝，不要尝试编码**：小数、超出 ±(2⁵³−1) 的整数、`NaN`、`Infinity`。
它们的字符串化在各语言里不一致（JS 的 `String(1.0)` 是 `"1"`，Java 的 `Double.toString(1.0)`
是 `"1.0"`），给它们编一个表示等于把跨语言分歧固化进契约。**金额请用最小单位整数或字符串。**

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
基串各段之间没有标签，而 nonce 的字符集是这两者的超集 —— 拿它们当 nonce 会让"基串
拼错顺序"这类实现缺陷在联调期**完全隐身**，等你哪天改成随机 nonce 才集体失败，那时
所有人都会去怀疑刚改的 nonce。

### 2.3 认证检查顺序

多个条件同时不满足时，你只会看到**最先命中**的那一个：

```
1. 四个签名头齐全 ......... 401 AUTH_HEADERS_MISSING
2. nonce 格式 ............. 401 NONCE_INVALID
3. 时间戳窗口 ............. 401 TIMESTAMP_INVALID
4. app_key 存在且启用 ..... 401 APP_KEY_INVALID
5. 凭证未过期 ............. 401 CREDENTIAL_EXPIRED
6. 源 IP 在白名单内 ....... 403 IP_NOT_ALLOWED
7. QPS 未超限 ............. 429 QPS_LIMITED
8. 载荷可规范化 ........... 400 CANONICAL_*
9. 签名正确 ............... 401 SIGNATURE_INVALID
10. scope 足够 ............ 403 SCOPE_FORBIDDEN
11. nonce 未被用过 ........ 401 NONCE_REPLAYED
```

第 8 步**刻意与 `SIGNATURE_INVALID` 分开**：它说的是"这份载荷里
有一个签不了的值"（小数、超范围整数、不合法的键名…），不是"你的密钥不对"。看到
`CANONICAL_*` 请查载荷，不要查密钥。

两条由此推出、对排障很有用的性质：

- **nonce 只在请求"本可成功"时才被消耗**（第 10 步在最后）。所以错签名重试永远回
  `SIGNATURE_INVALID`、越权重试永远回 `SCOPE_FORBIDDEN`，**不会被自己的重试污染成 `NONCE_REPLAYED`**。
- **参数校验在鉴权之前**。缺必填字段时你拿到的是 `400 xxx required`，不是 `401`。

### 2.4 scope

| scope | 覆盖接口 |
|---|---|
| `esim.read` | `products/list` `products/detail` `cardType` `order/query` `order/list` `iccid/profile` `order/usage` `iccid/renew` |
| `esim.write` | `order/create` `order/renew` |

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
BASE = 'https://tbetaopenapi.xpin.network'

def call(path, body):
    ts, nonce = str(int(time.time() * 1000)), 'n_' + secrets.token_hex(8)
    sign = sign_inbound(APP_SECRET, APP_KEY, 'POST', path, ts, nonce, body)
    # 可以直接用 json=body：签名不要求你控制发出去的字节，
    # requests 怎么序列化都不影响验签。
    return requests.post(BASE + path, json=body, headers={
        'x-api-key': APP_KEY, 'x-timestamp': ts, 'x-nonce': nonce, 'x-sign': sign,
    })
```

**Node.js**（34 行手写，无依赖）

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

调用时 `fetch(url, { body: JSON.stringify(body) })` 即可 —— **不需要**保证签名用的字符串
与发送用的字符串逐字节相同。

其它语言：Go / Java / PHP / .NET 都能在几十行内写完（只用到排序、UTF-8 字节长度、
HMAC-SHA256 三样）。**写完请先跑 [2.6](#26-自检向量) 那张自检向量表**，全过再联调。⚠️ 两个语言特有的坑：
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
| `403` | `SOURCE_PARTNER_NOT_ALLOWED` | `sourcePartnerCode` 不是你名下的子分销商 | 检查该字段 |
| `404` | `PRODUCT_NOT_FOUND` | 商品不存在或不属于你 | 从 `products/list` 动态取码 |
| `404` | `PRODUCT_NOT_AVAILABLE` | 下单时商品不可售/不存在 | 同上 |
| `404` | `ORDER_NOT_FOUND` | 订单不存在或不属于你 | 检查 `clientOrderNo` |
| `404` | `CARD_NOT_FOUND` | 卡不存在或不属于你 | 检查 `iccid` |
| `404` | — | 请求路径不存在 | 检查路径拼写与前缀 `/openapi/v1`。这一类的 `msg` 不是稳定错误码，不要对它做断言 |
| `409` | `IDEMPOTENCY_KEY_REUSED` | 同 `idempotencyKey` 但请求内容不同 | 换新的幂等键 |
| `409` | `CLIENT_ORDER_NO_REUSED` | `clientOrderNo` 已被别的订单占用 | 换新的业务单号 |
| `409` | `RENEW_NOT_ALLOWED` | 这张卡此刻不可续订 | 先调 `iccid/renew` 看 `reasons` |
| `413` | `REQUEST_ENTITY_TOO_LARGE` | 请求体超过 256KB | 拆小 |
| `415` | `CONTENT_TYPE_MUST_BE_JSON` | `Content-Type` 不是 `application/json` | 改头 |
| `429` | `QPS_LIMITED` | 超过凭证每秒请求上限 | 退避重试 |
| `503` | `OPENAPI_AUTH_UNAVAILABLE` / `PLATFORM_NOT_ENABLED` | 服务端依赖不可用 | 退避重试并联系平台 |

**跨租户隔离**：查不到与"不属于你"返回**同一个** `404`，平台不会告诉你"这个单存在但不是你的"。
所以 `404` 不能用来探测他人数据是否存在。

---

## 4. 接口详解

> 全部路径前缀 `/openapi/v1`。下文的响应样本均为接口的真实返回结构。

### 4.1 商品列表 `/products/list`

拉取**发布给你**的在售商品。这是获取 `productCode` 的**唯一正确来源**。

**请求**

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| `pageNo` | number | 否 | 下界 `1`，默认 `1` | 页码。深翻页有上界保护，越界返回 `400 pageNo Range error`；正常翻页碰不到 |
| `pageSize` | number | 否 | `[1, 100]`，默认 `20` | 每页条数 |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 商品数组，见下 |
| `hasMore` | boolean | 是否还有下一页（翻页终止条件） |
| `pageNo` / `pageSize` | number | 回显 |
| `language` | string | 商品文案的语言，当前恒为 `en` |

**商品对象**

| 字段 | 类型 | 说明 |
|---|---|---|
| `productCode` | string | 对外商品码，下单用它 |
| `productName` | string | 商品名 |
| `tagName` | string | 分组标签，如 `Europe` |
| `volume` | string | 流量额度（字符串数值），单位 MB；不限量为 `"0.000000"` |
| `dataLimited` | string | `Y` 限量 / `N` 不限量 |
| `validity` | number | 套餐天数 |
| `expireDay` | number | 卡有效期天数（从开卡起算） |
| `cardType` | string | 卡型，如 `ep1` `ep3` `eo1` `C4` |
| `productType` | string | 形态：`DATA_PACKAGE` 流量包 / `DAYPASS` 日包 |
| `mccList` | string[] | 覆盖国家的 MCC 列表 |
| `attributes` | object\|null | 扩展属性 |
| `retailPrice` | string | 零售价（字符串数值） |
| `currency` | string | 币种，如 `USD` |
| `configVersion` | number | 商品配置版本，变更时递增 |

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
        "retailPrice": "0.000000",
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
    "retailPrice": "0.000000",
    "currency": "USD",
    "configVersion": 1
  },
  "code": 200, "msg": "ok", "t": 1788431706435
}
```

**负例**：不存在的码 → `404 PRODUCT_NOT_FOUND`。

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

| 字段 | 类型 | 必填 |
|---|---|---|
| `productCode` | string | **是** |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `productCode` | string | 回显 |
| `cardType` | string | 卡型 |
| `productType` | string | 形态 |
| `capability.renewable` | boolean | **这个卡型**原理上是否支持续订 |
| `capability.supportUsageQuery` | boolean | 是否支持用量查询 |
| `capability.maxRenewCount` | number\|null | 卡上并发套餐数上限；`null` = 不限 |
| `capability.timeZone` | string | 卡型的计费时区，如 `UTC+0` / `UTC+8` |
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

> 🔴 **`capability.renewable` 不能用来点亮"续订"按钮。** 它说的是"这个卡型原理上能续"，
> 不是"这张卡现在能续"。判据只能是 §4.7 的 `allowed`。同一个 `renewable:true` 的卡型，
> 不同的卡 / 商品组合会给出不同答案。
>
> 同理 `supportUsageQuery:false` 的卡型，`order/usage` 恒返回 `usage: null` —— 那不是"还没数据"。

**负例**：不存在的码 → `404 PRODUCT_NOT_FOUND`。

---

### 4.4 开卡下单 `/order/create`

**需要 `esim.write` scope。**

**请求**

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| `productCode` | string | **是** | `[1, 160]` | 从 `products/list` 取 |
| `idempotencyKey` | string | **是** | `[8, 96]` | 幂等键，见下 |
| `clientOrderNo` | string | 否 | — | 你自己的业务单号，全局唯一 |
| `externalUserId` | string | 否 | — | 你系统里的终端用户标识 |
| `sourcePartnerCode` | string | 否 | — | 委托下单时指定你名下的子分销商 |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderNo` | string | 平台订单号 |
| `clientOrderNo` | string | 回显 |
| `productCode` | string | 回显 |
| `orderStatus` | string | 恒为 `ACCEPTED` |
| `accepted` | boolean | 恒为 `true` |
| `replayed` | boolean | `true` = 这是一次幂等重放，没有新建单 |

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

> 🔴 **`ACCEPTED` 只是受理，不是出卡。** 出卡是异步的，时长根据电信运营商的数据返回，可能在 1～15 分钟左右波动。
> 拿到 `orderNo` 后要么轮询 §4.5，要么等 §6 的 `ORDER_OPEN_RESULT` 回调。

#### 幂等语义

| 情况 | 结果 |
|---|---|
| 同 `idempotencyKey` + **相同请求内容** | `200`，返回**同一个** `orderNo`，`replayed: true` |
| 同 `idempotencyKey` + **不同请求内容** | `409 IDEMPOTENCY_KEY_REUSED` |
| 同 `clientOrderNo` + **不同** `idempotencyKey` | `409 CLIENT_ORDER_NO_REUSED` |

所以：**网络超时后原样重发是安全的**，不会重复建单。但换了内容就必须换幂等键。

**负例**：商品不存在或不可售 → `404 PRODUCT_NOT_AVAILABLE`；非法 `sourcePartnerCode` → `403 SOURCE_PARTNER_NOT_ALLOWED`。

**Node.js**

```js
async function createOrder(client, { productCode, clientOrderNo, externalUserId }) {
  // 幂等键与业务单号绑定：重试时必须复用同一个键，否则会建出第二张卡
  const idempotencyKey = `open-${clientOrderNo}`
  const data = await client.invoke('/openapi/v1/order/create', {
    productCode,
    idempotencyKey,
    clientOrderNo,
    externalUserId
  })
  if (data.replayed) console.log('幂等重放，复用已有订单', data.orderNo)
  return data.orderNo
}
```

**Java**

```java
public String createOrder(XpinEsimClient client, String productCode,
                          String clientOrderNo, String externalUserId) throws Exception {
    // 幂等键与业务单号绑定：重试时必须复用同一个键
    String idempotencyKey = "open-" + clientOrderNo;
    JsonNode data = client.invoke("/openapi/v1/order/create", Map.of(
            "productCode", productCode,
            "idempotencyKey", idempotencyKey,
            "clientOrderNo", clientOrderNo,
            "externalUserId", externalUserId));
    if (data.get("replayed").asBoolean()) {
        log.info("幂等重放，复用已有订单 {}", data.get("orderNo").asText());
    }
    return data.get("orderNo").asText();
}
```

---

### 4.5 订单查询 `/order/query`

**请求**

| 字段 | 类型 | 必填 |
|---|---|---|
| `clientOrderNo` | string | **是** |

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderNo` | string | 平台订单号 |
| `clientOrderNo` | string | 你的业务单号 |
| `externalUserId` | string\|null | 终端用户标识 |
| `productCode` | string | 商品码 |
| `orderStatus` | string | 见 §5 状态机 |
| `iccid` | string\|null | **只有履约后才有值** |
| `profileStatus` | string\|null | 卡的 profile 状态 |
| `product` | object | 下单时刻的**商品快照**（含 `settlementPrice` 结算价） |

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
    "product": {
      "productCode": "XP205032025ED92397E2AE461C",
      "productName": "Europe Unlimited 3 Days",
      "volume": "0", "dataLimited": "N", "validity": 3, "expireDay": 90,
      "cardType": "ep1", "productType": "DAYPASS",
      "mccList": ["276", "232"],
      "retailPrice": "0", "settlementPrice": null, "currency": "USD",
      "configVersion": 1
    }
  },
  "code": 200, "msg": "ok", "t": 1788431709839
}
```

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
| `pageSize` | number | 否 | 默认 20 |
| `pageToken` | string | 否 | 上一页返回的 `nextPageToken` |

**三条硬规则**：

| 构造 | 结果 |
|---|---|
| 零个选择器 | `400 ORDER_RANGE_SELECTOR_REQUIRED` |
| 两个及以上选择器 | `400 ORDER_RANGE_SELECTOR_CONFLICT` |
| **只传 `pageToken` 不带选择器** | `400 ORDER_RANGE_SELECTOR_REQUIRED`（不是游标错误！） |
| 选择器 + 伪造/过期游标 | `400 ORDER_PAGE_TOKEN_INVALID` |

> 🔴 **`pageToken` 不是独立的选择器**，翻页时必须把**原来的选择器一起带上**。游标带签名且**有效期约 10 分钟**。

**响应 `data`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 订单数组，元素结构同 §4.5 的 `data` |
| `hasMore` | boolean | 是否还有下一页 |
| `nextPageToken` | string\|null | 下一页游标；`null` 表示到底 |

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
| `productCode` | string | **是** | 想续的目标商品 |
| `externalUserId` | string | 否 | 传了才会校验用户归属 |

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
| `PRODUCT_NOT_FOUND` | 目标商品不存在或不属于你 | ❌ 终局 |
| `PRODUCT_OFF_SALE` | 目标商品已下架 | ⚠️ 可恢复（换商品或等上架） |
| `CARD_NOT_RENEWABLE` | 电信运营商明确这张卡不可续 | ❌ 终局 |
| `CARD_TYPE_MISMATCH` | 目标商品的卡型与这张卡不同 | ❌ 终局（续订永不换卡型） |
| `RENEW_WINDOW_EXPIRED` | 续订窗口已过 | ❌ 终局 |
| `ORDER_NOT_RENEWABLE` | 当前套餐没有续订截止时间 | ❌ 终局 |
| `EXTERNAL_USER_ID_CONFLICT` | 传入的 `externalUserId` 与卡上绑定的不一致 | ❌ 终局（改传对的） |
| `DAYPASS_REQUIRES_EMPTY_CARD` | 日包只能装在空卡上 | ❌ 终局（该组合永远不行） |
| **`RENEWABILITY_PENDING_SYNC`** | 平台还没从电信运营商同步到这张卡的可续订性 | 🔄 **稍后重试** |
| **`RENEW_WINDOW_PENDING_SYNC`** | 续订窗口尚未同步 | 🔄 **稍后重试** |
| **`RENEWABILITY_STALE`** | 同步过但证据已过期 | 🔄 **稍后重试** |
| **`DAYPASS_IN_FORCE`** | 日包在用期间整张卡不可续（用流量包续也不行） | 🔄 日包结束后解除 |
| `CONCURRENT_ORDER_LIMIT_REACHED` | 卡上并发套餐数已达上限 | 🔄 有套餐到期后解除 |
| `CARD_CAPABILITY_UNKNOWN` | 卡型能力未同步 | 🔄 稍后重试 |
| `CARD_TYPE_RULE_NOT_PUBLISHED` | 该卡型的续订能力尚未开放 | ⚙️ 联系平台 |

> 🔴 **不要把"稍后重试"类的 reason 当成永久拒绝**。尤其是 `RENEWABILITY_STALE`：
> 平台的续订证据有新鲜期，过期就会出现这一条，而它随时可能被刷新。
> 新鲜期与同步时机根据电信运营商的数据返回，可能在 1～15 分钟左右波动。
> 正确做法是**在用户点击时实时查一次**，而不是缓存资格结果。

#### 规则按卡型差异化（示例）

| 卡 | 用同形态商品续 | 用同卡型的另一形态商品续 |
|---|---|---|
| `C4` / 流量包卡 | ✅ 可续 | ✅ 可续 |
| `ep1` / 流量包卡 | ✅ 可续 | ❌ `DAYPASS_REQUIRES_EMPTY_CARD` |
| `ep1` / 日包卡 | ❌ `DAYPASS_IN_FORCE` | ❌ `DAYPASS_IN_FORCE` |

**Node.js**

```js
const RETRYABLE = new Set([
  'RENEWABILITY_PENDING_SYNC', 'RENEW_WINDOW_PENDING_SYNC', 'RENEWABILITY_STALE',
  'DAYPASS_IN_FORCE', 'CONCURRENT_ORDER_LIMIT_REACHED', 'CARD_CAPABILITY_UNKNOWN'
])

async function checkRenewable(client, iccid, productCode) {
  const d = await client.invoke('/openapi/v1/iccid/renew', { iccid, productCode })
  return {
    allowed: d.allowed,
    reasons: d.reasons,
    // 只有全部原因都是"可恢复"时，才提示用户稍后再试；否则是终局拒绝
    retryable: !d.allowed && d.reasons.length > 0 && d.reasons.every((r) => RETRYABLE.has(r))
  }
}
```

**Java**

```java
private static final Set<String> RETRYABLE = Set.of(
        "RENEWABILITY_PENDING_SYNC", "RENEW_WINDOW_PENDING_SYNC", "RENEWABILITY_STALE",
        "DAYPASS_IN_FORCE", "CONCURRENT_ORDER_LIMIT_REACHED", "CARD_CAPABILITY_UNKNOWN");

public record RenewCheck(boolean allowed, List<String> reasons, boolean retryable) {}

public RenewCheck checkRenewable(XpinEsimClient client, String iccid, String productCode) throws Exception {
    JsonNode d = client.invoke("/openapi/v1/iccid/renew", Map.of("iccid", iccid, "productCode", productCode));
    boolean allowed = d.get("allowed").asBoolean();
    List<String> reasons = new ArrayList<>();
    d.get("reasons").forEach(n -> reasons.add(n.asText()));
    // 只有全部原因都可恢复时才提示稍后再试
    boolean retryable = !allowed && !reasons.isEmpty() && RETRYABLE.containsAll(reasons);
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
| `productCode` | string | **是** | 目标商品 |
| `idempotencyKey` | string | **是** | `[8, 96]` |
| `clientOrderNo` | string | 否 | 你的业务单号 |
| `externalUserId` | string | 否 | 终端用户标识 |

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
- 不可续时 → `409 RENEW_NOT_ALLOWED`。

**Node.js —— 完整续订流程**

```js
async function renew(client, { iccid, productCode, clientOrderNo, externalUserId }) {
  // 1) 资格必须实时查，不能用缓存 —— 证据有新鲜期，且刷新时机随电信运营商的数据返回波动
  const check = await checkRenewable(client, iccid, productCode)
  if (!check.allowed) {
    const err = new Error(`不可续订: ${check.reasons.join(',')}`)
    err.reasons = check.reasons
    err.retryable = check.retryable
    throw err
  }

  // 2) 下单（受理）
  const data = await client.invoke('/openapi/v1/order/renew', {
    iccid,
    productCode,
    idempotencyKey: `renew-${clientOrderNo}`,
    clientOrderNo,
    externalUserId
  })

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
    // 1) 资格实时查，不缓存 —— 证据有新鲜期，且刷新时机随电信运营商的数据返回波动
    RenewCheck check = checkRenewable(client, iccid, productCode);
    if (!check.allowed()) {
        throw new IllegalStateException("不可续订: " + String.join(",", check.reasons()));
    }
    // 2) 受理
    JsonNode data = client.invoke("/openapi/v1/order/renew", Map.of(
            "iccid", iccid,
            "productCode", productCode,
            "idempotencyKey", "renew-" + clientOrderNo,
            "clientOrderNo", clientOrderNo,
            "externalUserId", externalUserId));
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
| `cardType` | string | 卡型 |
| `delivery.activationCode` | string | **激活码**，形如 `LPA:1$<smdp>$<matchingId>` |
| `delivery.latestActivationTime` | string\|null | 最晚激活时间 |
| `delivery.imsi` / `delivery.msisdn` | string\|null | IMSI / MSISDN |
| `profile.status` | string | `UNKNOWN` / `ENABLED` / `RELEASED` / `DISABLED` 等 |
| `profile.statusTime` | string | 状态时间（ISO-8601） |
| `profile.changeVersion` | number | 状态变更版本，**单调递增**，可用于丢序检测 |
| `renewal.renewable` | boolean\|null | 电信运营商给出的可续订性；`null` = 未同步 |
| `renewal.expirationTime` | string\|null | 续订截止时间。**开卡履约完成即可读到**，不必等后续刷新 |
| `renewal.activePackages` | number | 这张卡当前占用的套餐槽位数（**含尚未履约的在途订单**，见下方警告） |
| `renewal.maxConcurrentPackages` | number\|null | 并发上限 |
| `renewal.evidenceFresh` | boolean | 续订证据是否新鲜。新鲜期与刷新时机随电信运营商的数据返回波动 |
| `renewal.evidenceSyncedAt` | string\|null | 证据同步时刻。它是**卡型能力**的刷新时刻，同一卡型的多张卡上取值相同，与 `usage.observedAt` 不是一回事 |
| `usage` | object\|null | 用量；`null` 见 §4.10 |
| `packageStatus` | string | `CREATED` / `NOT_ACTIVATED` / `IN_USE` / `EXPIRED` 等 |
| `packageEndTime` | string\|null | 套餐结束时间 |

**示例响应**

```json
{
  "data": {
    "iccid": "89000000000000000442",
    "externalUserId": "your-user-0442",
    "cardType": "ep1",
    "delivery": {
      "activationCode": "LPA:1$smdp-a.example.com$7ADBC7BBD32B47438D0BB3F4EXAMPLE",
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
      "observedUnit": "MB", "observedAt": "2026-09-05T13:29:24.791Z"
    },
    "packageStatus": "NOT_ACTIVATED",
    "packageEndTime": "2026-09-08T12:53:54.000Z"
  },
  "code": 200, "msg": "ok", "t": 1788615660098
}
```

> 🔴 **`renewal.activePackages` 不是"已生效的套餐数"**：它把**尚未履约的在途订单**也计算在内，
> 所以一笔刚提交、还没出结果的续订单同样会把它 +1。**不能用它判断"续充成功了没"**。
> 判据永远是**续订单自己的 `orderStatus === 'FULFILLED'`**。

> **不同卡型走不同的 SM-DP+**（例如 `ep1` 与 `C4` 的 SM-DP+ 域名并不相同）。
> 不要对激活码里的域名做任何硬编码假设，整串原样交给终端用户。

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
| `dataTotalBytes` | number\|null | 套餐总量，**整数字节** |
| `dataUsedBytes` | number\|null | 已用量，**整数字节** |
| `dataRemainBytes` | number\|null | 剩余量，**整数字节** |
| `observedUnit` | string\|null | 这组数字换算前的单位，仅作证据。**它不是上面三个字段的单位** —— 那永远是字节 |
| `observedAt` | string | 这次读数的观测时刻（ISO-8601） |

> **三个数值字段是整数字节，字段名自带单位。** 不要按 `observedUnit` 去二次换算。

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

> **`usage: null` 有两种成因，靠 `cardType` 的 `capability.supportUsageQuery` 区分**：
> - `supportUsageQuery: false` → 这个卡型**本来就查不到用量**，永远是 `null`，不要重试。
> - `supportUsageQuery: true` 但 `usage` 为 `null` → 还没有样本，稍后再查。

> **读数是周期性刷新的**，不是每次调用都实时回源。`observedAt` 就是这次读数的时刻；
> 两次调用之间它可能不变，那表示期间没有新样本，不是接口失败。

**负例**：卡不存在 → `404 CARD_NOT_FOUND`。

`/iccid/profile` 的 `usage` 与本接口返回的是**同一个投影**，字段与取值完全一致，
按哪条取都行。

---

## 5. 业务流程与状态机

### 5.1 订单状态机

```
ACCEPTED ──► SUBMITTING ──► PROVIDER_ACCEPTED ──► FULFILLED
   │              │               │                   │
   │              │               └──► RECONCILING ───┘
   │              │                        │
   └──► RETRY_PENDING ──► SUBMITTING       │
                  │                        │
                  └────────► FAILED ◄──────┘
```

| 状态 | 含义 | 你该做什么 |
|---|---|---|
| `ACCEPTED` | 平台已受理，尚未提交电信运营商 | 继续等 |
| `SUBMITTING` | 正在提交电信运营商 | 继续等 |
| `PROVIDER_ACCEPTED` | 电信运营商已受理，等履约回调 | 继续等；**长时间不动是平台侧或电信运营商侧的问题，不是你的参数问题** |
| `RECONCILING` | 提交/对账阶段异常，正在核对 | 继续等，超时联系平台 |
| `RETRY_PENDING` | 等待重试 | 继续等 |
| **`FULFILLED`** | **已履约，`iccid` 可用** | 取激活码交付用户 |
| **`FAILED`** | **终态：这一单不会再出卡** | **停止轮询**。这一单不会自愈，需要出卡就重新下一单（用**新的** `idempotencyKey` 与 `clientOrderNo`）。原因需联系平台核查 |

> 🔴 **轮询必须同时判 `FAILED`。** 只等 `FULFILLED` 的循环会在失败单上一直转到超时，
> 而那一单永远不会变。`FULFILLED` 与 `FAILED` 都是**终态**，收到任一个就停。

**时延**：开卡与续订的 `ACCEPTED → FULFILLED` 时长根据电信运营商的数据返回，可能在 1～15 分钟左右波动。
明显超出该区间仍未 `FULFILLED` 也未 `FAILED` 时需与平台核查，**不要靠重复下单绕过**
（会建出多张卡）。

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

### 5.3 续订完整流程

```
1. iccid/renew             实时查资格（不要缓存！）
2. allowed === true 才继续，否则按 reasons 分类提示用户
3. order/renew             → ACCEPTED + 新的 orderNo（iccid 不变）
4. 轮询 order/query 到 FULFILLED，或等 ORDER_RENEW_RESULT 回调
5. 断言 iccid 未变
```

> ⚠️ **续订资格有时间窗**：新卡建成后，平台需要先同步到电信运营商侧的续订证据；该证据也有新鲜期。
> 同步时机与新鲜期根据电信运营商的数据返回，可能在 1～15 分钟左右波动。窗口外查资格会得到 `RENEWABILITY_STALE`。
> 因此**必须在用户点击的那一刻实时查**，并对 `RENEWABILITY_STALE` 做"稍后再试"的友好提示。

---

## 6. 回调通知

平台在事件发生时主动 `POST` 到你的回调地址。**这项能力需要先向平台方申请开通**，
申请时提供你的回调接收地址，并说明需要订阅哪些事件（不指定则默认订阅全部）。

> **事件只在入队那一刻按配置判定**：开通之前受理的订单**不会补发**，开通后要下新单才有事件。

### 6.1 事件类型

| `eventType` | 何时发 | `data` 字段 |
|---|---|---|
| `ORDER_OPEN_RESULT` | 开卡订单落 `FULFILLED` 时，每单一次 | `orderNo` `clientOrderNo` `idempotencyKey` `iccid` `imsi` `msisdn` `activationCode` `productCode` `orderStatus` |
| `ORDER_RENEW_RESULT` | 续订单落 `FULFILLED` 时 | **同上**（`activationCode` 是该卡**原有**的激活码，不变） |
| `STATUS_CHANGED` | 卡的 profile 状态真的变化时 | `orderNo` `iccid` `entityType`(恒 `PROFILE`) `previousStatus` `currentStatus` `changedAt` `changeVersion` |

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

**示例报文**（三类各一）

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
  if (typeof signHeader !== 'string' || signHeader.length !== expected.length) return false
  return crypto.timingSafeEqual(Buffer.from(signHeader), Buffer.from(expected))
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
>    处理并重投（间隔约 5 秒，见 6.4 的上限）。所以处理失败时**不要**回 `0000`。
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

> 🔴 **幂等键只能是 `eventId`（或头里的 `x-event-id`）**，不能用 `sign`、`nonce` 或 `timestamp`——它们每次投递都会变。
>
> 🔴 **`"0000"` 是字符串，同步接口的 `200` 是数字，两套码本。** 回 `{"code":200}` 或 `{"code":0}`
> 都会被判为失败，导致同一事件在约两小时内被反复重投。这是最容易写错、且不会立刻报错的一个坑。

> **密钥轮换注意**：平台用你凭证**当前**的 `app_secret` 对回调签名，且不会回落到旧密钥。
> 轮换期间已在投递中的事件可能验签失败，**请与平台方约定轮换时间窗**，避开有事件在途的时段。

---

