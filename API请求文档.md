# ITDOG & tool.lu API 请求文档

> 逆向分析日期：2026-09-12，所有请求均已实测验证。
> 涉及站点：`www.itdog.cn`（HTTP测速 / Ping / Ping IPv6）、`tool.lu`（IP归属地查询）

---

## 一、ITDOG 通用机制

### 1.1 整体流程

三个工具（HTTP测速、Ping、Ping IPv6）流程一致：

```
① (可选) GET /verify/clicaptcha.php?type=ajax   → 检查是否需要人机验证
② POST 表单创建任务（整页响应，非 AJAX）          → 从返回 HTML 中提取 task_id
③ 连接 WebSocket wss://www.itdog.cn/websockets/{task_id}/{hash}
④ 发送 {"task_id":"..."} 后，服务端逐条推送各监测节点结果
```

**没有 XHR 轮询**，实时进度全部通过 WebSocket 推送。

### 1.2 公共请求头

| 请求头 | 值 |
|---|---|
| `User-Agent` | 常规浏览器 UA，如 `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36` |
| `Content-Type` | `application/x-www-form-urlencoded` |
| `Referer` | 对应工具页面 URL |
| `Origin` | `https://www.itdog.cn` |

无需特殊 Cookie，无需登录。

### 1.3 人机验证检查

```
GET https://www.itdog.cn/verify/clicaptcha.php?type=ajax
```

响应：

```json
{"type":"success","tip":"无需人机验证 !"}
```

- `type == "success"`：无需验证，`clicaptcha-verify` 字段留空即可。
- 否则页面会弹出点击验证码（clicaptcha），需在浏览器中完成验证后，把验证值填入表单的 `clicaptcha-verify` 隐藏字段。

### 1.4 WebSocket 签名算法（核心）

```
URL:  wss://www.itdog.cn/websockets/{task_id}/{hash}
hash = md5(task_id + SALT) 的中间 16 位（即 32 位十六进制的第 8~24 位）
SALT = "What this is is no longer important."
```

- `task_id` 格式示例：`202609121206001euwc6p4uaoff3l82y`（日期时间 + 随机字符串）
- 该盐值从混淆的 `/frame/js/pages/http_speed.js` 中解码得到，并已用多组真实抓包数据验证。

Node.js 计算示例：

```javascript
const crypto = require('crypto');
const SALT = 'What this is is no longer important.';
const hash = crypto.createHash('md5').update(taskId + SALT).digest('hex').substr(8, 16);
```

Python 计算示例：

```python
import hashlib
SALT = "What this is is no longer important."
hash16 = hashlib.md5((task_id + SALT).encode()).hexdigest()[8:24]
```

### 1.5 WebSocket 通信协议

- 连接成功后，客户端发送：`{"task_id":"<task_id>"}`
- 服务端逐条推送节点结果（JSON，每行一条）
- 全部节点完成后连接关闭（或收到结束消息）。实测每个任务约推送 200~300 条节点消息。

---

## 二、HTTP 测速（/http/）

### 2.1 创建任务

```
POST https://www.itdog.cn/http/
Content-Type: application/x-www-form-urlencoded
```

**请求参数：**

| 参数 | 必填 | 说明 | 示例值 |
|---|---|---|---|
| `host` | 是 | 目标网址 | `www.baidu.com` |
| `host_s` | 是 | 目标网址（重复字段，值同 host） | `www.baidu.com` |
| `check_mode` | 是 | 测试模式：`fast`=快速测试，`slow`=缓慢测试 | `fast` |
| `http_version` | 是 | `auto`=默认http2(向下兼容) / `http_1_1` / `http_2` / `http_3` | `auto` |
| `method` | 是 | 请求方法：`get` / `post` | `get` |
| `line` | 否 | 线路：逗号分隔，`1`=电信 `2`=联通 `3`=移动 `5`=港澳台海外；空=全选 | `1,2,3,5` 或空 |
| `ipv4` | 否 | 指定解析 IP | 空 |
| `referer` | 否 | 自定义 Referer | 空 |
| `ua` | 否 | 自定义 User-Agent | 空 |
| `cookies` | 否 | 自定义 Cookie，格式 `key=123;page=123` | 空 |
| `redirect_num` | 是 | 允许重定向最大次数 | `5` |
| `dns_server_type` | 是 | `isp`=运营商DNS / `custom`=指定DNS | `isp` |
| `dns_server` | 否 | 自定义 DNS（仅 `dns_server_type=custom` 时有效） | 空 |
| `clicaptcha-verify` | 是 | 人机验证值，无需验证时留空 | 空 |

**请求示例：**

```
POST /http/ HTTP/1.1
Host: www.itdog.cn
Content-Type: application/x-www-form-urlencoded

host=www.baidu.com&host_s=www.baidu.com&check_mode=fast&http_version=auto&method=get&line=&ipv4=&referer=&ua=&cookies=&redirect_num=5&dns_server_type=isp&dns_server=&clicaptcha-verify=
```

**响应：** HTTP 200，返回完整 HTML 页面（服务端渲染）。从中提取 `task_id`：

```javascript
const m = html.match(/var task_id='([^']+)'/);
const taskId = m[1];
```

页面内同时包含：`var wss_url='wss://www.itdog.cn/websockets/';`

### 2.2 WebSocket 结果消息

```
wss://www.itdog.cn/websockets/{task_id}/{hash}
```

单条消息示例：

```json
{
  "type": "success",
  "ip": "36.152.44.93",
  "http_code": 200,
  "all_time": "0.063",
  "dns_time": "0.010",
  "connect_time": "0.013",
  "download_time": "0.030",
  "redirect": 1,
  "redirect_time": "0.013",
  "head": "HTTP/1.1 302 Found<br>Connection: keep-alive<br>...",
  "node_id": "xxxx",
  "line": 1,
  "name": "上海11电信",
  "region": 1,
  "province": 2,
  "address": "中国/江苏/南京/电信"
}
```

| 字段 | 说明 |
|---|---|
| `type` | `success`=成功；失败节点另有失败类型 |
| `ip` | 解析出的响应 IP |
| `http_code` | HTTP 状态码 |
| `all_time` | 总耗时（秒） |
| `dns_time` | 解析耗时 |
| `connect_time` | 连接耗时 |
| `download_time` | 下载耗时 |
| `redirect` / `redirect_time` | 重定向次数 / 耗时 |
| `head` | 响应头（`<br>` 分隔） |
| `name` / `address` | 监测点名称 / IP 归属地 |
| `line` | 线路：1=电信 2=联通 3=移动 5=海外 |

---

## 三、Ping（/ping/）

### 3.1 创建任务

**注意：目标主机在 URL 路径中，不在表单体里。**

```
POST https://www.itdog.cn/ping/{host}
Content-Type: application/x-www-form-urlencoded
```

示例：`POST https://www.itdog.cn/ping/www.baidu.com`

**请求参数：**

| 参数 | 必填 | 说明 | 示例值 |
|---|---|---|---|
| `line` | 否 | 线路（同 HTTP 测速），空=全选 | 空 |
| `button_click` | 是 | 固定 `yes` | `yes` |
| `dns_server_type` | 是 | `isp` / `custom` | `isp` |
| `dns_server` | 否 | 自定义 DNS | 空 |
| `clicaptcha-verify` | 是 | 人机验证值，无需验证时留空 | 空 |

**响应：** HTTP 200 HTML 页面，同样提取 `var task_id='...'`。

### 3.2 WebSocket 结果消息

单条消息示例：

```json
{
  "ip": "110.242.69.21",
  "result": "7",
  "node_id": "406j3mzq1kxbvjw9",
  "line": 2,
  "name": "天津5联通",
  "region": 4,
  "province": 1,
  "address": "中国/河北/保定/联通"
}
```

| 字段 | 说明 |
|---|---|
| `ip` | 解析 IP |
| `result` | 延迟（ms）；`-1` 表示超时/丢包 |
| `name` / `address` | 监测点 / 归属地 |

---

## 四、Ping IPv6（/ping_ipv6/）

与 Ping 完全一致，仅路径不同：

```
POST https://www.itdog.cn/ping_ipv6/{host}
```

参数同 Ping（`line`、`button_click=yes`、`dns_server_type`、`dns_server`、`clicaptcha-verify`）。

WebSocket 消息格式同 Ping，`ip` 字段为 IPv6 地址，例如：

```json
{
  "ip": "2408:871a:2100:186c:0:ff:b07e:3fbc",
  "result": "8",
  "name": "天津5联通",
  "address": "中国/北京/北京/中国联通中国169骨干网"
}
```

---

## 五、tool.lu IP 归属地查询

### 5.1 请求

```
POST https://tool.lu/ip/ajax.html
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Referer: https://tool.lu/ip/
Origin: https://tool.lu
```

**请求参数：**

| 参数 | 必填 | 说明 | 示例值 |
|---|---|---|---|
| `ip` | 是 | 要查询的 IP 或域名 | `8.8.8.8` |

### 5.2 响应

```json
{
  "status": true,
  "message": "",
  "text": {
    "ip": "8.8.8.8",
    "l": 134744072,
    "chunzhen": "美国 加利福尼亚州 圣克拉拉 山景城 谷歌公司DNS服务器",
    "taobao": "美国   ",
    "ipip": "GOOGLE.COM GOOGLE.COM  -",
    "ip2region": "United States California  Google LLC",
    "geolite": "United States   -",
    "dbip": "United States   -",
    "ipDataCloud": "美国 California Mountain View ",
    "amap": null
  }
}
```

| 字段 | 说明 |
|---|---|
| `status` | `true`=查询成功 |
| `text.ip` | 查询的 IP |
| `text.l` | IP 的长整型数值 |
| `text.chunzhen` | 纯真数据归属地 |
| `text.ipip` | ipip.net 数据 |
| `text.taobao` | 淘宝数据 |
| `text.ip2region` | IP2REGION 数据 |
| `text.geolite` | GeoLite2 数据 |
| `text.dbip` | DB-IP 数据 |
| `text.ipDataCloud` | IP数据云（高精定位） |

### 5.3 ⚠️ WAF 限制

该站 nginx WAF 会拦截**非真实浏览器**的 POST 请求（返回 `403 Forbidden`）：

- `requests` / `curl` / PowerShell `Invoke-WebRequest` 直接 POST → 403
- `curl_cffi` 模拟 Chrome/Safari/Edge/Firefox TLS 指纹 → 仍 403（疑似同时限制数据中心出口 IP）
- GET 请求不受影响（`GET /ip/ajax.html` 返回 200，但查询需要 POST）

**可行的调用方式：**

1. 真实浏览器环境（住宅网络 + 浏览器发起）
2. Playwright / Puppeteer 等浏览器自动化：

```javascript
// Playwright 示例
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('https://tool.lu/ip/');
  await page.fill('input[name="ip"]', '8.8.8.8');
  const [resp] = await Promise.all([
    page.waitForResponse(r => r.url().includes('/ip/ajax.html')),
    page.click('button[type="submit"]'),
  ]);
  console.log(await resp.json());
  await browser.close();
})();
```

官方提示：`请勿将本页面作为 api 使用，当请求量过高时，系统会限制访问！`

---

## 六、完整调用示例（Node.js ≥ 22）

项目内已提供可直接运行的实现：

| 文件 | 说明 |
|---|---|
| `itdog_tool.js` | ITDOG 三合一：`node itdog_tool.js <http\|ping\|ping6> <域名>` |
| `toollu_ip.js` | tool.lu 查询：`node toollu_ip.js <IP>` |

ITDOG 最小调用流程：

```javascript
const crypto = require('crypto');
const SALT = 'What this is is no longer important.';

// 1. 创建任务
const resp = await fetch('https://www.itdog.cn/ping/www.baidu.com', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded', 'User-Agent': UA },
  body: 'line=&button_click=yes&dns_server_type=isp&dns_server=&clicaptcha-verify=',
});
const taskId = (await resp.text()).match(/var task_id='([^']+)'/)[1];

// 2. 连接 WebSocket
const hash = crypto.createHash('md5').update(taskId + SALT).digest('hex').substr(8, 16);
const ws = new WebSocket(`wss://www.itdog.cn/websockets/${taskId}/${hash}`);
ws.onopen = () => ws.send(JSON.stringify({ task_id: taskId }));
ws.onmessage = (ev) => console.log(JSON.parse(ev.data));
```

---

## 七、注意事项

1. **频率控制**：频繁请求会触发 ITDOG 的点击验证码（clicaptcha）或 tool.lu 的 403 限流，请控制请求频率。
2. **盐值时效**：ITDOG 的 WebSocket 盐值来自其混淆 JS，站点改版后可能变化，若 403/连接失败需重新逆向 `/frame/js/pages/http_speed.js`。
3. **合规**：以上接口仅供个人学习研究，请勿用于商业用途或高频调用。
