# 补货监控 · n8n 工作流

一个可以自己部署的 [n8n](https://n8n.io) 工作流：**自由选择要监控的网站和商品**，从缺货变成有货时立刻发邮件提醒你。

> A self-hosted n8n workflow that watches the products you choose, on the sites you choose, and emails you the moment one comes back in stock. See [English quick start](#english-quick-start).

## 特点

- **自选网站**：一个工作流可以同时监控多个商品，支持三种判断方式
- **只在补货那一刻提醒**：一直有货不会重复发；再次缺货后又补货会再提醒；同一轮补货的商品合并成一封邮件
- **宁可不报，不误报**：识别不了状态时算读取失败，绝不当作有货
- **失效会告诉你**：连续读取失败 8 次，发一封「监控可能失效」，并说明原因
- **频率可控**：默认 15 分钟一次，最小 5 分钟；接口提供预计补货时间时，临近时自动加快
- **零额外依赖**：只需要 n8n 和一个能用 SMTP 发信的邮箱，不需要 AI、不需要数据库

## 工作原理

```
每 5 分钟（或手动）
  → 读取上次状态 → 按各自间隔挑出这一轮要查的商品
  → 抓取页面或接口 → 判断是否有货 → 保存状态
  → 有商品刚补货（或监控失效）→ 发邮件
```

## 安装

需要 n8n 支持 **Data tables** 功能（在 n8n 2.43 上测试过）。

1. 下载 [`workflow/restock-monitor.json`](workflow/restock-monitor.json)，在 n8n 里新建工作流 → 右上角 `⋯` → **Import from File**。
2. 打开「**发送邮件**」节点：新建 **SMTP** 凭据，并把 From Email、To Email 里的 `you@example.com` 改成你自己的邮箱。
3. 打开「**监控列表**」节点，按下面的说明配置 `TARGETS`。
4. 在工作流 Settings 里设置 **Timezone**（影响邮件里显示的时间）。
5. 把鼠标移到「**发送测试邮件**」节点上，点它的 ▶，确认能收到邮件。
6. 把鼠标移到「**手动测试**」节点上，点它的 ▶，然后到 **Overview → Data tables → 补货监控状态** 查看结果：`status` 应该是 `in_stock` 或 `out_of_stock`，`failCount` 应该是 0。
7. 点右上角 **Publish**，之后就会自动运行。

## 配置要监控的商品

所有配置都在「**监控列表**」节点顶部的 `TARGETS` 里，每个商品一项。自带的 3 个示例默认没有启用，把 `enabled` 改成 `true` 才会生效。

**先判断网站属于哪一种：**

### 方式一：Shopify 店铺

很多独立站基于 Shopify。在商品地址后面加 `.js`，浏览器里能看到一段带 `"available"` 的内容，就可以用这种方式：

```js
{
  enabled: true,
  key: 'my-shopify-item',
  name: '我想买的东西',
  type: 'json',
  url: 'https://shop.example.com/products/your-product.js',
  buyUrl: 'https://shop.example.com/products/your-product',
  path: 'available',
  inStockWhen: true,
}
```

这里的 `available` 表示商品的任一款式有货。如果只想等某个尺码或颜色，可以写成 `path: 'variants.0.available'`（`0` 是第几个款式，从 0 开始数）。

### 方式二：网站有返回库存的 JSON 接口

```js
{
  enabled: true,
  key: 'my-api-item',
  name: '我想买的东西',
  type: 'json',
  url: 'https://api.example.com/product/123',
  buyUrl: 'https://www.example.com/product/123',
  path: 'data.status',
  outValues: ['out_of_stock', 'sold_out'],
  restockTimePath: 'data.restock_time',   // 可选
}
```

`path` 指向的字段可以是单个值或数组，只要出现 `outValues` 里的任意一个就算缺货，否则算有货。字段不存在时算读取失败。

### 方式三：普通网页

```js
{
  enabled: true,
  key: 'my-page-item',
  name: '我想买的东西',
  type: 'html',
  url: 'https://www.example.com/item/123',
  outOfStockText: ['Sold Out', '缺货', '售罄'],
  inStockText: ['Add to Cart', '加入购物车'],
  intervalMinutes: 30,                    // 可选
}
```

先找 `outOfStockText`，有就算缺货；否则找 `inStockText`，有就算有货；都没有算读取失败。脚本和样式里的文字不参与判断。文字要写成页面**实际显示**的内容。

### 字段说明

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `enabled` | 是 | `true` 才会监控 |
| `key` | 是 | 唯一标识，只用英文、数字、`-`、`_`，配好后不要改，否则会被当成新商品 |
| `name` | 是 | 邮件里显示的名称 |
| `type` | 是 | `json` 或 `html` |
| `url` | 是 | 要读取的地址 |
| `buyUrl` | 否 | 邮件按钮的链接，默认用 `url` |
| `intervalMinutes` | 否 | 检查间隔（分钟），默认 15，最小 5 |
| `path` | json | 库存字段的路径，用 `.` 分隔 |
| `outValues` / `inStockWhen` | json | 二选一：缺货时出现的值 / 有货时字段等于的值 |
| `restockTimePath` | 否 | 预计补货时间（Unix 秒）的路径；前 10 分钟到后 2 小时内按最小间隔检查 |
| `outOfStockText` / `inStockText` | html | 表示缺货 / 有货的文字 |

## 哪些网站不适用

状态读取不到时，你会收到「可能失效」的邮件，里面写着原因。常见情况：

| 情况 | 说明 |
| --- | --- |
| 内容由脚本加载，页面 HTML 是空壳 | `html` 方式读不到。可以在浏览器开发者工具的 Network 面板里看网页自己请求的 JSON 数据，能公开访问的话改用 `json` 方式 |
| 要求登录才能看到库存 | 不支持 |
| 有人机验证或反爬保护（淘宝、天猫、京东等大平台通常如此） | 不支持，工作流检测到验证页面时会停止判断 |

遇到这些情况，优先使用网站自带的「到货通知」功能。

## 合理使用

这个工具用来帮**你自己**留意想买的商品，请合理使用：

- 使用前查看目标网站的使用条款和 `robots.txt`。很多网站限制自动化访问，后果由使用者自行承担。
- 不要把检查间隔调到 5 分钟以下，不要监控大量商品。
- **不要用它绕过**登录、人机验证、IP 封禁等访问限制，也不要往工作流里放你的账号 Cookie。
- 工作流只读取页面，**不会自动下单**，也不建议把它改成抢购工具。

## 隐私与安全

- 工作流不含任何密钥，SMTP 密码保存在你自己的 n8n 凭据里，由 n8n 加密存储；分享工作流前，记得删掉你填写的邮箱和监控列表。
- 所有状态都存在你自己 n8n 的数据表里，不会发送给任何第三方；对外的请求只有访问你配置的地址，以及向你的邮箱服务商发信。
- 请求不带任何账号信息，User-Agent 是 `Mozilla/5.0 (compatible; n8n-restock-monitor)`，向网站如实表明这是自动化工具。
- 成功的运行不保存执行数据，避免每天产生大量记录。

## 常见问题

**点了 Execute workflow 没反应，或者报「还没有启用任何监控项」**：要先在「监控列表」里配置商品，并把 `enabled` 改成 `true`。

**想发到 Telegram 或企业微信**：把「发送邮件」节点换成对应的节点即可，提醒内容在上一个节点输出的 `subject` 和 `html` 里。

**想让某个商品重新触发提醒**：在数据表里删掉它那一行。

## English quick start

1. Import [`workflow/restock-monitor.json`](workflow/restock-monitor.json) into n8n (`⋯` → **Import from File**). Requires n8n with **Data tables** (tested on 2.43).
2. In the **发送邮件** (send email) node, create an **SMTP** credential and replace `you@example.com` with your address.
3. In the **监控列表** (watch list) node, edit `TARGETS` and set `enabled: true`. Supported modes: JSON field (`path` with `inStockWhen` or `outValues`, which covers Shopify's `/products/<handle>.js`) and HTML text (`outOfStockText`, `inStockText`).
4. Hover the **手动测试** (manual test) node and click ▶, then check the **补货监控状态** Data Table. Click **Publish** when it looks right.

Only monitor sites whose terms allow it. The workflow does not bypass logins, CAPTCHAs or anti-bot systems, and it never places orders. Minimum interval is 5 minutes.

## License

[MIT](LICENSE)
