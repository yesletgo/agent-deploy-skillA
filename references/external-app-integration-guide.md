# 外部应用接入指南

> 合作方接入协议 · 最后更新 2026-11-02

本文面向应用的前端、后端及部署负责人（以及他们使用的 AI 编程助手），说明怎样把自己的网页应用以 iframe 形式接入 Lucimix：登记资料、嵌入页面、兑换身份、扣费退款、内容分享。

本文的接口和消息是平台契约。应用的业务页面、快照、隐私处理、数据库及下游执行由应用方实现。

## 0. 先读这里

### 0.1 你要做的全部事情

| 必须 / 可选 | 内容 | 章节 |
|---|---|---|
| **必须：前端** | 消息桥：发 `ready`，收 `ticket`，首次真实操作前发 `interaction-request`，按需发 `share` / `balance-changed` / `insufficient-balance` | 第 2 章、[附录 A](#附录-a-前端消息桥完整示例) |
| **必须：后端** | `POST /api/apps/session` 用票据换 `app_uid`；`POST /api/apps/charge` 扣费；`POST /api/apps/refund` 退款 | 第 3、4 章、[附录 B](#附录-b-后端最小示例) |
| **做内容分享时** | 快照先脱敏、不可变；提供内容页、有效期与失效页 | 第 5 章 |
| **可选** | 并入通知：用户登录后两个编号合成一个人时，平台通知应用合并数据 | 第 6 章 |

不论用户是否登录，应用看到的都是一个 `app_uid`，扣费、退款都只认它。

所有接口都在同一个地址下、用同一个凭证：

| 接口 | 请求体 | 成功响应 | 章节 |
|---|---|---|---|
| `POST https://lucimix.cn/api/apps/session` | `{ticket}` | `{app_uid, nickname}` | 3 |
| `POST https://lucimix.cn/api/apps/charge` | `{client_request_id, app_uid, credits, product_name}` | `{transaction_id, credits, charged}` | 4.2 |
| `POST https://lucimix.cn/api/apps/refund` | `{client_request_id, original_transaction_id, credits, reason?}` | `{transaction_id, original_transaction_id, credits, refunded}` | 4.3 |
| `POST https://lucimix.cn/api/apps/merge/verify`（可选） | `{ticket}` | `{from_app_uid, to_app_uid}` | 6 |

- 请求头一律 `X-App-Token: <应用凭证>` 和 `Content-Type: application/json`，只从应用后端调用。
- 错误一律是 `{"error":{"code":"…","message":"…"}}`，个别带 `"reason"`。唯一例外：请求体超过 16 KiB 时是 413 `{"detail":"guest_app_request_too_large"}`。

### 0.2 接入步骤

1. 向 Lucimix 对接人取得 `app_key` 和 `X-App-Token`，按 1.1 填好登记资料交给对接人（测试环境地址与凭证另向对接人索取）。
2. 配好 iframe 响应头（2.1），前端照[附录 A](#附录-a-前端消息桥完整示例) 接入消息桥。
3. 后端照[附录 B](#附录-b-后端最小示例) 接入 session / charge / refund。
4. 做内容分享的，按第 5 章做快照和内容页；想合并登录前后数据的，按第 6 章接并入通知。
5. 用第 7 章联调清单逐项验收；提交资料与上线前对照[附录 D](#附录-d-常见对接问题) 自查。

### 0.3 术语

| 术语 | 含义 |
|---|---|
| Lucimix | 面向用户的平台（网页 PC 版与 H5）。应用以 iframe 形式嵌在它的页面里 |
| 光点 | 平台计费单位，金额一律用十进制**字符串**表示，如 `"10"`、`"12.5"` |
| `app_uid` | 平台发给本应用的用户编号。本应用内稳定，不同应用之间无法关联；按不透明字符串存（列宽至少 128），不解析前缀 |
| 票据（ticket） | 平台经消息桥交给 iframe 的一次性短期凭证，60 秒有效；应用后端用它换 `app_uid` 后建立自己的会话 |
| 未登录访客 / 登录用户 | 未登录访客花平台送的试用额度，登录用户花自己的光点。两者在应用里都是 `app_uid`，按同一套接口处理，只有个别错误码不同 |
| ULID | 带时间戳的 26 位唯一 ID，扣费、退款的流水号用它 |
| outbox | 应用数据库里「待发给平台」的记录表，与业务结果同一事务写入，后台重试直到平台确认 |
| CAS | 比较并交换（compare-and-swap），保证同一订单只有一个请求能进入执行 |

**隐私边界**：应用只拿到本应用的 `app_uid` 和昵称，拿不到手机号、平台账号 ID 或平台会话；票据不进 URL、分析系统或普通日志。

## 1. 登记资料

### 1.1 应用方提交

以下以已上线的「运势助手」为例。把整张表照着自己的应用填一遍交给对接人即可，格式限制见 1.2。

| 要提交什么 | 字段 | 运势助手怎么填 | 平台拿它干什么 | 不提交会怎样 |
|---|---|---|---|---|
| **必填：所有应用** | | | | |
| 应用标识 | `app_key` | `bazi` | 应用的唯一编号，平台页面地址是 `/apps/bazi`，分享链接、票据、扣费都按它认应用 | 无法登记 |
| 应用名称 | `name` | `运势助手` | 侧栏、应用列表、介绍页标题显示 | 无法登记 |
| 图标 | `icon_url` | `https://baziapp.lucimix.cn/static/img/icons/bazi_log.png` | 侧栏和应用列表里的图标（HTTPS 地址） | 无法登记 |
| 应用首页地址 | `jump_url` | `https://baziapp.lucimix.cn/` | 平台用 iframe 加载这个地址；内容分享页也用它的域名拼地址（见 1.2「内容页固定地址」） | 无法登记 |
| 介绍资料 | `intro`（一句话 `tagline`、功能说明 `summary`、价格 `pricing`、注意事项 `notice`、截图 `screenshots`） | 一句话：「说出生辰就能排盘，用大白话看懂你的性格、事业、财运与流年运势」；价格：「每次 AI 解读或提问 10 光点；排盘、查看命盘与历史记录免费；解读失败自动退回」；截图 1 张 | 用户第一次打开应用时看到的介绍页，以及「应用说明」 | 没有介绍页，用户直接进应用；价格不明 |
| 支持的端 | `platforms` | `["pc", "mobile"]` | 只填 `["pc"]` 时，手机用户打开会被引导去电脑上用（2.2） | 默认按电脑和手机都支持处理 |
| **要做内容分享才填：每种可分享的内容填一组** | | | | |
| 内容类型 | `content_type` | `share` | 应用发分享消息时带的类型名，平台按它查下面的路径来生成链接 | 分享时平台找不到类型，回分享失败，应用提示「分享暂不可用」 |
| 内容名称 | `name` | `内容` | 平台提示文字，如只读提示「这是一个已分享的运势助手内容，只能查看。」 | 同上 |
| 内容页路径 | `path` | `s` | 访客打开分享链接时，平台在 iframe 里加载 `https://baziapp.lucimix.cn/s/{share_id}`，**应用必须在这个地址提供只读的分享内容页** | 同上；路径填错则访客打开是 404 |
| 隐私字段 | `private_fields` | `[{"field":"profile.name","handling":"未勾选「显示姓名」时显示为某先生/某女士"},{"field":"profile.birth_city","handling":"生成快照前删除"}]` | 登记备案，说明应用在快照里怎么隐藏个人信息 | 无法登记；没有敏感字段填 `[]` |
| 能否继续操作 | `allow_continue` | `false`（访客只能看） | `false` 时平台在内容上方显示只读提示，且不给访客发票据；`true` 时访客可在内容页上接着用 | 无法登记 |
| 默认有效期 | `ttl_default` | `604800`（7 天，与分享弹窗文案一致） | 登记备案；到期判断由应用自己做，过期后应用在内容页显示失效页 | 无法登记；长期有效填 `null` |
| **可选：要接并入通知才填** | | | | |
| 并入通知路径 | `claim_callback_path` | `/api/lucimix/guest-claim` | 用户登录后两个编号合成一个人时，平台向 `https://baziapp.lucimix.cn/api/lucimix/guest-claim` 发并入通知（第 6 章）。处理程序必须先调 `/api/apps/merge/verify` 验证票据、可靠保存后再返回 2xx；没验证就回 2xx，这条通知就丢了 | 不发并入通知 |
| **文字说明（不入库）** | | | | |
| 隐私与到期方案 | — | 快照保存哪些内容、隐藏哪些、7 天后怎么失效 | 对接人上线前审核 | 不予上线 |

**运势助手的完整登记资料**（介绍正文从略）：

```json
{
  "app_key": "bazi",
  "name": "运势助手",
  "icon_url": "https://baziapp.lucimix.cn/static/img/icons/bazi_log.png",
  "jump_url": "https://baziapp.lucimix.cn/",
  "intro": { "tagline": "…", "summary": "…", "pricing": "…", "notice": "…", "screenshots": [{ "url": "…", "alt": "…" }] },
  "platforms": ["pc", "mobile"],
  "sharing": {
    "claim_callback_path": "/api/lucimix/guest-claim",
    "content_types": [
      {
        "content_type": "share",
        "name": "内容",
        "path": "s",
        "private_fields": [
          { "field": "profile.name", "handling": "未勾选「显示姓名」时显示为某先生/某女士" },
          { "field": "profile.birth_city", "handling": "生成快照前删除" }
        ],
        "allow_continue": false,
        "ttl_default": 604800
      }
    ]
  }
}
```

`content_type`、`path`、`private_fields` 里的字段名都由应用自己定，上面只是示例；要求是**应用代码里发的分享类型名、真实的内容页路由，与登记的值完全一致**。

### 1.2 字段格式

资料交给对接人，由对接人在平台后台录入。请用实际功能、价格、截图和脱敏规则替换示例。

**基础字段**

| 字段 | 约束与含义 |
|---|---|
| `app_key` | 稳定唯一；匹配 `^[a-z0-9][a-z0-9-]{1,62}$` |
| `name` | 与侧栏显示一致 |
| `icon_url` | 有效非空的 HTTPS 图标地址 |
| `jump_url` | 公开业务入口，HTTPS；不含用户名密码、查询参数、fragment、编码路径或点路径 |
| `platforms` | 取值 `pc` / `mobile`；**必须含 `pc`**，不重复；不填默认 `["pc","mobile"]` |

**介绍资料 `intro`**

| 字段 | 约束与含义 |
|---|---|
| `intro` | 完整对象，拒绝未知字段，序列化最多 128 KiB；**提交即整体替换线上介绍**，不改介绍就不要带这个字段 |
| `tagline` | 去首尾空白后 1–120 个字符 |
| `summary` / `pricing` / `notice` | 各 1–10000 字符；受限 Markdown：文本段落、加粗、列表、普通 HTTPS 链接；不提交 HTML、脚本、CSS 或内嵌图片。`pricing` 要写清哪些免费、哪些收费、失败是否退、主动停止是否收费 |
| `screenshots` | 必填数组，0–10 项；每项 `url`（HTTPS，最多 2048 字符）、`alt`（1–200 字符）、可选 `caption`（最多 500 字符） |

**分享配置 `sharing`**（做内容分享或接并入通知时填写）

| 字段 | 约束与含义 |
|---|---|
| `content_types` | 必填数组，最多 20 类；只分享应用首页时为 `[]` |
| `claim_callback_path` | 可选，并入通知地址：应用同一 HTTPS origin 下的固定服务端路径；无点路径、查询、fragment、编码路径；必须直接返回响应，不能 3xx 跳转 |

**内容类型 `content_types[]`**

| 字段 | 约束与含义 |
|---|---|
| `content_type` | 匹配 `^[a-z][a-z0-9_-]{0,31}$`，本应用内唯一 |
| `name` | 内容名，1–64 字符，会拼进只读提示「这是一个已分享的{应用名}{内容名}，只能查看。」，所以用名词（「命盘」「报告」「内容」），不要用「分享」 |
| `path` | 1–3 段，以 `/` 分隔；每段匹配 `^[a-z0-9][a-z0-9-]{0,31}$`；无首尾斜线、编码、查询或通配符。不同类型的路径不能相同，也不能**按段**互为前缀：`reports` 与 `reports/2026` 冲突；`report` 与 `reports` 可以并存 |
| `private_fields` | 必填数组，无敏感字段填 `[]`；每类最多 50 项；`field` 1–128 字符、`handling` 1–500 字符；只登记处理规则，不上传原值 |
| `allow_continue` | 必须显式提交，布尔值：访客能否在快照基础上继续操作 |
| `ttl_default` | 必须显式提交，正整数秒或 `null`（长期） |

**内容页固定地址**：分享链接打开后，平台在 iframe 中加载的内容页地址由规则固定生成：

```text
{jump_url 的 scheme}://{jump_url 的 host[:port]}/{path}/{share_id}
```

- 只取 `jump_url` 的协议和主机（含非默认端口），**丢弃 `jump_url` 自己的路径**；`path` 有多段时原样拼接。
- 例：运势助手 `jump_url` 为 `https://baziapp.lucimix.cn/`，`path` 为 `s`，`share_id` 为 `abc`，内容页是 `https://baziapp.lucimix.cn/s/abc`。
- **应用部署在路径前缀下时**（如整个应用挂在 `/ref-app/` 下），前缀须写进每个内容类型的 `path`（如 `ref-app/reports`），并占用 3 段中的一段；只登记 `reports` 会得到前缀之外的地址。
- 应用应使用独立域名或子域名，不要挂在 Lucimix 自己的域名下。

**更新规则**：编辑时省略 `intro` / `sharing` 保留原值；`sharing: null` 关闭内容分享与并入通知。录入后约 1 分钟生效。

### 1.3 平台交付

| 交付项 | 正式环境取值 | 用途 |
|---|---|---|
| 平台 origin（PC） | `https://lucimix.cn`<br>`https://www.lucimix.cn` | 两个域名是同一个平台，用户从哪个进来，iframe 的父页面就是哪个。用于 iframe CSP、收消息的 `event.origin`、发消息的 `targetOrigin`，**全部都要列入** |
| 平台 origin（H5） | `https://m.lucimix.cn` | 同上。漏配 H5 会导致手机上加载不出页面或连不上身份 |
| Lucimix 接口基地址 | `https://lucimix.cn` | 应用后端调用本文全部接口；路径已含 `/api`，拼接时不要重复。这是唯一需要配置的平台地址 |
| `app_key`、`X-App-Token` | 由对接人单独交付 | 本应用标识和服务端凭证，经安全渠道交付；Token 只在创建时显示一次 |
| 联调联系人与核账渠道 | 由对接人提供 | 故障核账、测试光点调整 |

**`X-App-Token` 的保管**：它能代表应用发起扣费，只能存于应用后端的密钥管理；不放进浏览器、前端构建变量、登记素材、日志或版本库。轮换后旧 Token **立即**失效，先更新后端配置再请对接人轮换。

### 1.4 应用状态

| 应用状态 | 用户 |
|---|---|
| 草稿 | 任何人在 Lucimix 中都看不到 |
| 上线 | 登录与否都出现在应用列表中，未登录访客可直接使用 |
| 下线 | 看不到，不再签发票据，新扣费被拒；下线前的扣费仍可退款 |

联调前请对接人在测试环境把应用设为上线。

## 2. 嵌入与消息桥

### 2.1 HTTPS、iframe 与会话

**允许被嵌入**：应用页面不能返回 `X-Frame-Options: DENY` 或 `SAMEORIGIN`。用 CSP 响应头列出**全部**平台 origin，漏掉哪个，用户从那个域名进来就加载不出页面：

```http
Content-Security-Policy: frame-ancestors https://lucimix.cn https://www.lucimix.cn https://m.lucimix.cn;
```

**iframe 属性**（当前版本）：

| 属性 | 取值 |
|---|---|
| `sandbox` | `allow-scripts allow-same-origin allow-forms allow-popups allow-popups-to-escape-sandbox allow-downloads allow-modals` |
| `allow` | `fullscreen; clipboard-write`（可全屏、可写剪贴板，不能读剪贴板） |
| `referrerpolicy` | `strict-origin-when-cross-origin`（应用只能拿到平台 origin） |

- 不能跳转整个顶层页面；外部帮助或支付页面用新窗口打开。
- 应用内部导航不改变平台地址栏；iframe 重新加载时重新握手。

**应用会话不能依赖 Cookie**：浏览器可能拦截第三方 Cookie。应用会话由自己的后端签发令牌，前端只放在内存里、随 iframe 存活，通过 `Authorization` 请求自己的后端。不要存进 `sessionStorage` / `localStorage` 留到下次：同一设备上用户可能已经登录、退出或换了账号，iframe 每次加载都以新票据换到的身份为准，不恢复上一次的会话。独立打开业务入口拿不到票据时，提示「请从 Lucimix 进入」；公开快照可以照常显示公开内容。

### 2.2 页面在 PC 与 H5 中的显示

**应用页没有平台条**：不论登录与否、从侧栏还是分享链接进来，PC 都是「左侧导航栏 + 右侧 iframe」，H5 都是「Lucimix 顶栏 + 下方 iframe」。

| 场景 | iframe 区域 | 周围的平台界面 |
|---|---|---|
| PC 应用首页 | 浏览器宽度减去左侧导航栏，占满整个高度 | 只有左侧导航栏，宽 280px（收起后 64px）；登录入口或光点余额在侧栏 |
| H5 应用首页 | 占满屏幕宽度；高度为屏幕高度减去顶栏 | 顶栏 56px（另加刘海屏顶部安全区）：☰ · Lucimix · 应用名，未登录时右侧多一个「登录」；**没有返回键**；底部**没有**为安全区留白 |
| 内容分享页（PC 与 H5） | 上述区域再减去下面两条 | iframe 下方**一律**有一条「立即试用」操作条（约 65px，H5 另加底部安全区），点了打开应用首页、保留邀请码；`allow_continue:false` 时 iframe 上方还有一行只读提示。应用的失效页同样显示在这个外壳里 |
| 登录 | 不改变 iframe 尺寸 | PC 是居中弹窗、H5 是底部登录面板，都**覆盖**在 iframe 上方 |

**应用页面要做的适配**：

- 布局按 iframe 尺寸自适应；PC 可用宽度通常比屏幕窄 280px，H5 宽度一般 320–430px。
- 自己负责滚动，平台外层不滚动。
- H5 首页底部固定的按钮或输入框自行留出安全区（可试 `env(safe-area-inset-bottom)`，请在 iPhone 上实测）；软键盘弹出时的布局自行处理。
- 配色自定（可跟随 `prefers-color-scheme`）。
- iframe 挂载后平台显示「加载中…」遮罩，收到 `external-app:ready` 才移除；8 秒内没收到则显示「应用打不开，请稍后重试」。
- 只登记 `["pc"]` 且没有介绍资料的应用，在手机上仍会以手机宽度加载，页面至少不能白屏。

### 2.3 入口与打开方式

| 入口 | 平台页面 | iframe 打开方式 |
|---|---|---|
| 登录用户从侧栏 / 导航进入 | `/apps/{app_key}` | `business`：首次进入先看平台的介绍页，点「立即使用」后加载；应用发 `ready` 后平台**直接发票据**，不发 `context` |
| 未登录访客打开应用 | `/apps/{app_key}` | `home`：与应用首页分享相同 |
| 应用首页分享 | `/apps/{app_key}?from=share&ref={邀请码}` | `home` |
| 内容分享 | `/apps/{app_key}/{path}/{share_id}?ref={邀请码}` | `snapshot`：先只读；`allow_continue:true` 时访客可在自己的新数据上接着操作，不能修改原快照 |

**公开页（`home` / `snapshot`）的票据时机**：

- 应用发 `ready` 后，平台先发 `context`（只说明打开方式，不是登录证明）。
- **登录用户**：首页和可继续的内容页在 `context` 之后**直接**收到票据。
- **未登录访客**：纯阅读不建身份、不发票据；用户第一次真实保存、生成或修改时，应用发 `interaction-request`，平台才发票据。
- 只读内容页（`allow_continue:false`）不发票据。

统一处理即可：**任何时候收到 `ticket` 都交给后端换 `app_uid`；需要身份而还没有时发 `interaction-request` 再等票据**。附录 A 已按这个写好。

**登录前后**：未登录访客在应用里登录后，iframe 重新加载，应用照常握手、换 `app_uid`（何时会变见第 3 章）。

### 2.4 必须坚持的信任边界

- **消息来源校验**：每条平台消息都校验 `event.origin === LUCIMIX_ORIGIN` 和 `event.source === window.parent`。发消息用完整固定的 `targetOrigin`，不用 `"*"`。
- **平台有多个 origin**：页面加载时从**固定白名单**里选出当前嵌入方作为 `LUCIMIX_ORIGIN`（依据 `location.ancestorOrigins[0]`，不支持时用 `document.referrer` 的 origin），之后收发都只认这一个；选不出时不连接。写法见附录 A 开头。
- **`context.authenticated` 只作展示**，不是登录证明：不要凭它创建会话、启用权限或扣费。
- **后端不信任浏览器**：应用后端不接受浏览器指定的 `app_uid`、价格或商品，一律从自己的会话和价目表取。

### 2.5 消息参考

`request_id` 是应用生成的 1–64 位 ASCII 字母、数字、下划线或连字符（UUID 可用），只用于前端去重，**不是扣费流水号**。格式不合法的消息平台不回应。

| 方向 | 消息对象 | 用途 | 平台怎么处理 |
|---|---|---|---|
| 应用 → 平台 | `{type:"external-app:ready"}` | 监听器已装好；也用来重新要票据 | 停止 8 秒加载计时；`business` 发票据，公开页发 `context`（同一 iframe 1 秒内重复的 ready 忽略） |
| 平台 → 应用 | `{type:"external-app:context", mode:"home", authenticated:false, handshake_id:"…"}` | 公开页的打开方式，`mode` 为 `home` / `snapshot` | — |
| 应用 → 平台 | `{type:"external-app:interaction-request", request_id:"…"}` | `home` 页首次真实操作前要票据；不带 `share_id` | 发票据 |
| 应用 → 平台 | `{type:"external-app:interaction-request", request_id:"…", share_id:"…"}` | `snapshot` 页首次真实操作前要票据 | 只在登记 `allow_continue:true` 且 `share_id` 等于当前链接时发票据，否则不回应 |
| 平台 → 应用 | `{type:"external-app:ticket", ticket:"…"}` | 一次性票据，60 秒有效。只读 `ticket` 字段，其他字段忽略 | — |
| 应用 → 平台 | `{type:"external-app:share", request_id:"…"}` | 分享应用首页 | 已登录：生成链接、打开分享弹窗（H5 为分享面板）；未登录：回 `error` 并弹出登录 |
| 应用 → 平台 | `{type:"external-app:share", request_id:"…", content_type:"share", share_id:"…"}` | 分享内容快照（快照须先由应用后端生成） | 同上；`content_type` 未登记时回 `error` |
| 平台 → 应用 | `{type:"external-app:share-result", request_id:"…", status:"opened"}` | 分享弹窗已打开，结束 loading；失败为 `status:"error", code:"share_unavailable"` | — |
| 应用 → 平台 | `{type:"external-app:balance-changed"}` | 扣费或退款确认后提示平台刷新余额；1 秒内合并 | 刷新余额 |
| 应用 → 平台 | `{type:"external-app:insufficient-balance"}` | 扣费返回 402 `insufficient_credits` 或 401 `login_required`、或身份兑换返回 401 `login_required` 时发一次 | 未登录：弹出登录；已登录：弹「光点不足」并刷新余额 |

**`insufficient-balance` 的规则**：

- 只带 `type`，不带其他字段。
- 应用自己规定的试用次数用完、想让用户登录时，也发这一条。
- 只在以下情况发一次：扣费（4.2）明确返回 402 `insufficient_credits` 或 401 `login_required`；身份兑换（第 3 章）返回 401 `login_required`。不在轮询中反复发；503、`ticket_invalid` 等其他失败或暂时读不到余额都不发。

### 2.6 前端实现要点

完整可运行的消息桥见[附录 A](#附录-a-前端消息桥完整示例)。

- 示例中的 `/your-backend/*` 是**应用方自己的后端接口**，不是平台接口。
- `LUCIMIX_ORIGINS` 白名单要包含 1.3 的全部平台 origin；漏掉的那个域名上消息全部被丢弃，表现为一直连接不上。
- 先装消息监听，再立即发 `ready`；不要等用户点按钮才发。
- 收到票据立即交给自己的后端换 `app_uid`；票据一次性，不存、不重复兑换。兑换失败（票据过期、503、429 等；`login_required` 除外）就在用户下一次操作时重新要票据：`business` 页重发 `ready`，公开页再发一次 `interaction-request`（附录 A 的 `ensureSessionForUserAction` 就是这样做的），不要自动循环重试。
- 每次换到新的 `app_uid`，丢弃旧会话和旧的异步结果，按新编号重新加载用户数据（登录前后可能换人）。
- 业务输入先保留在页面，等身份就绪再提交，失败不清空输入。
- 示例用 `crypto.randomUUID()` 生成 `request_id`，浏览器只在 HTTPS（或 localhost）页面提供它；HTTP 测试页需改用 UUID 库。

## 3. 身份兑换

`POST https://lucimix.cn/api/apps/session`，请求头 `X-App-Token`、`Content-Type: application/json`。

```json
{ "ticket": "<从消息桥收到的票据>" }
```

成功 HTTP 200：

```json
{ "app_uid": "au_01JB7R6M2Y8ZQ4K9W3T5N1PXCV", "nickname": "小明" }
```

- `app_uid` 是这个用户在本应用里的编号，**按不透明字符串存储，列宽至少 128，不解析前缀**。扣费、退款都只认它。
- `nickname` 可能为 `null`（未登录访客一律为 `null`），不是主键。
- 拿到后由应用后端签发自己的会话令牌返回前端；之后每个业务请求都校验这个会话，从会话里取 `app_uid`。

**`app_uid` 什么时候会变**：

| 情况 | 应用看到的 |
|---|---|
| 登录用户反复进入、换设备 | 同一个 `app_uid` |
| 未登录访客用过应用，之后用**新账号**（或从没用过本应用的账号）登录 | **还是同一个 `app_uid`**，数据都在 |
| 未登录访客用过应用，之后用**以前用过本应用的账号**登录 | 该账号原来的 `app_uid`。访客期间的数据留在旧编号下；想合并，接第 6 章的并入通知 |
| 极少数情况下旧编号失效 | 再用它扣费得 403 `app_unavailable`。前端重新握手（附录 A 的 `lucimixBridge.reconnect()`）换新 `app_uid`；旧编号上的扣费仍可退款 |

**错误**（`error.code`）：

| HTTP / code | 含义与处理 |
|---|---|
| 401 `ticket_invalid` | 票据为空、无效、过期、已用过、属于其他应用，或 `X-App-Token` 不对（平台不细分）。前端重新要票据，不要拿旧票据重试 |
| 401 `login_required` | 这位未登录访客只能登录后再用（试用到期等）。前端发 `insufficient-balance`，平台会弹登录 |
| 403 `app_unavailable` | 只出现在未登录访客的票据上：应用未上架或已下线。提示暂不可用 |
| 422 `invalid_request` | 请求体不合法（见 4.5） |
| 429 | 限流（**按状态码 429 处理**：未登录访客的码是 `guest_app_rate_limited`，登录用户的是 `rate_limited`），退避后重新要票据 |
| 503 `unavailable` | 平台暂不可用。稍后重新要票据再兑换 |

## 4. 扣费与退款

**付费动作的规则只有一套**，对所有 `app_uid` 一样：先扣费，扣费成功才执行；执行失败由应用调退款。免费动作不调平台接口。

```text
用户确认报价
  → 应用后端 POST /api/apps/charge（新 ULID）
      ├─ 200            → 执行业务
      │                    ├─ 成功 → 结束
      │                    ├─ 失败 → POST /api/apps/refund（新 ULID），后台重试到 200
      │                    └─ 用户中途主动停止 → 按原价收费，不退款
      ├─ 402 / 401 login_required → 未扣费；前端发 insufficient-balance；这笔作废
      ├─ 403 app_unavailable → 编号已失效；前端重新握手换新 app_uid
      ├─ 其他 400 / 401 / 403 / 404 / 409 → 参数、凭证或原单问题；核对，不原样循环
      └─ 429 / 503 / 超时 / 500 → 结果未知或暂时故障；同一 ID 退避重试
  → 前端发 balance-changed
```

### 4.1 错误怎么处理（只需记这张表）

| 结果 | 应用怎么做 |
|---|---|
| 200 | 成功。`charged:false` / `refunded:false` 是同一流水的重放，代表原笔已成功，**不代表免费执行一次** |
| 402 `insufficient_credits` | 未扣费。前端发 `insufficient-balance`。用户之后再点，**用新 ID** 发新的扣费 |
| 401 `login_required` | 未扣费。只能登录后再用（试用到期、平台暂停试用、来源 IP 当日额度用完等，平台统一成这一个码）。处理同 402 |
| 403 `app_unavailable` | 这个 `app_uid` 已失效（极少见）。前端重新握手换新 `app_uid`（附录 A 的 `lucimixBridge.reconnect()`），用户再点 |
| 400 / 404 / 409 及其他 401 / 403 | 参数、编号、凭证或原单有问题。核对后修正，**不要原样循环重发** |
| 429 / 503 / 超时 / 500 | 结果未知或暂时故障。**同一 ID、同一参数**退避重试，拿到确定结果前不执行业务、也不换 ID |

### 4.2 扣费

`POST https://lucimix.cn/api/apps/charge`

```json
{
  "client_request_id": "01JB7S0Q3V5G2H8K4M6N9P1RTW",
  "app_uid": "au_01JB7R6M2Y8ZQ4K9W3T5N1PXCV",
  "credits": "10",
  "product_name": "AI 解读 × 1"
}
```

| 字段 | 规则 |
|---|---|
| `client_request_id` | 用 ULID 库现生成；匹配 `^([a-z0-9]{1,16}_)?[0-9A-HJKMNP-TV-Z]{26}$`（可带不超过 16 位的小写前缀加下划线）。UUID、订单号或示例里的值不能直接用 |
| 时间窗 | 新流水的 ULID 时间须在过去 24 小时至未来 5 分钟之间；已入账流水的重放不受限 |
| `app_uid` | 后端会话里的编号，不能扣别人的 |
| `credits` | 光点十进制**字符串**（JSON 数字被拒绝），正数，最多两位小数，单笔不超过 1000000 |
| `product_name` | 去首尾空白后 1–64 字符，显示在用户明细里；写清商品和数量 |

**响应**：

| HTTP / code | 含义与处理 |
|---|---|
| 200 `{transaction_id, credits, charged:true}` | 扣费成功。`transaction_id` 等于原 `client_request_id`，退款时用它；`credits` 是规范化字符串（`"10.00"` 返回 `"10"`），按数值比较 |
| 200 `charged:false` | 同一流水、同一编号、同一金额的重放，原笔成功 |
| 400 `invalid_input` | 字段格式、金额、ULID 或时间窗不合法；修正后用新请求 |
| 401 `login_required` | 见 4.1 |
| 401 `unauthenticated` | 应用已下线，未扣费（登录用户） |
| 402 `insufficient_credits` | 见 4.1 |
| 403 `app_unavailable` | 这个 `app_uid` 已失效（极少见，平台侧身份切换失败，见第 3 章），未扣费。不发 `insufficient-balance`；前端重新握手（附录 A 的 `lucimixBridge.reconnect()`）换新的 `app_uid`，旧编号不再重试 |
| 403 `guest_app_trial_unavailable` | 应用已下线，未扣费（未登录访客） |
| 404 `not_found` | `app_uid` 不认识、不属于本应用，或 `X-App-Token` 错误（三者平台不区分）；核对凭证配置，以及编号是否来自本应用的 session，不自造编号 |
| 409 `conflict` | 同一 `client_request_id` 换了金额或编号，或已被其他应用占用；核对原笔。带 `reason:"charge_voided"` 表示这笔曾被平台撤销，换新 ULID |
| 413 | 请求体超过 16 KiB，响应是 `{"detail":"guest_app_request_too_large"}` |
| 429 | 限流；同一 ID 退避重试。**按状态码 429 处理**，不要只认某个 code（未登录访客为 `guest_app_rate_limited`，登录用户为 `rate_limited`） |
| 503 `app_identity_converting` | 用户刚登录、平台正在切换编号，带 `Retry-After: 2`；等待后**同一 ID** 重试 |
| 503 `guest_app_finance_pending` | 同一流水另一个请求仍在处理，带 `Retry-After: 15`；等待后同一 ID 重试 |
| 503 `unavailable` 及其他 503 | 平台暂不可用；同一 ID 退避重试 |
| 超时、响应丢失、500 | **不能当作未扣费**；保留订单和 ID，同一 ID、同一参数重试，拿到 200 后再执行 |

- 平台判断「同一笔重放」比较编号和 `credits`，**不比较 `product_name`**。应用本地仍要把流水号与原商品和业务输入绑定，不能拿旧流水号换商品。
- 用户中途登录也照常用原 ID 重试，不会重复扣。
- 订单超过 24 小时仍无法确认是否入账时，停止重试，走核账渠道联系对接人。
- 已有登录用户的订阅续费，可由应用后端定时任务用该用户的 `app_uid` 发起，不要求用户在线。

### 4.3 退款

`POST https://lucimix.cn/api/apps/refund`

```json
{
  "client_request_id": "01JB7S4D8F1K3M5P7R9T2V4X6Z",
  "original_transaction_id": "01JB7S0Q3V5G2H8K4M6N9P1RTW",
  "credits": "10",
  "reason": "解读失败"
}
```

- `client_request_id` 是**本次退款**新生成的 ULID，与原扣费 ID 不同，遵守同样的格式和时间窗；它的 ULID 时间不能早于原扣费 ID 一小时以上（现生成即满足）。
- `original_transaction_id` 是原扣费返回的 `transaction_id`，不受时间窗限制，**扣费很久以后也能退**。
- 退款不需要 `app_uid`，也不需要用户在线；登录前扣的费，登录后照常退。
- `credits` 可部分退款，也可多次退款，累计不超过原扣费。`reason` 可选，去首尾空白后最多 64 字符。
- 幂等按退款自己的 `client_request_id`：同参数重放返回 `refunded:false`，参数不同返回 409。

| HTTP / code | 含义与处理 |
|---|---|
| 200 `{transaction_id, original_transaction_id, credits, refunded:true}` | 退款成功；重放为 `refunded:false` |
| 400 `invalid_input` | 字段、ULID、时间窗不合法，或退款 ID 与原单 ID 相同；修正后重发 |
| 401 `unauthenticated` | `X-App-Token` 无效或已轮换；未退款。核对凭证，不要原样循环重发 |
| 404 `not_found` | 原单不存在、不属本应用，或原扣费从未发生（如当时是 402）；原单是未登录访客的扣费时，`X-App-Token` 错误也返回它（不区分）。核对原单与凭证 |
| 409 `conflict`，`reason:"refund_exceeds_original"` | 累计退款将超过原扣费；核对已退金额 |
| 409 `conflict`，`reason:"refund_rejected"` | 这个退款 ID 曾被平台明确拒绝，不会再生效；换新 ULID 重新发起 |
| 409 `conflict` | 退款 ID 已被占用，或同一 ID 换了原单/金额；核对 |
| 429 / 503 / 超时 / 500 | 结果未知；用**原退款 ID**、原参数重试，不能再建一张新的退款单 |

应用保存原单、每笔退款 ID 和已确认金额；失败动作的退款单与业务结果同一事务写入（outbox），后台重试到 200。

### 4.4 执行结果

扣费 200 即已实扣：

- 业务成功：结束。
- 业务失败：调退款（4.3），退款单持久化、后台重试到 200。
- 用户执行中主动停止：按原价收费，不退款；须在 `pricing` 里写明。
- 批量操作会部分失败时，按可独立成功 / 退款的单位拆分扣费。
- 平台只保证财务幂等，应用必须自己保证业务只执行一次（本地唯一键 + 状态 CAS）。

### 4.5 请求格式、限流与超时

> 以下为当前取值，平台可能调整，重试逻辑不要写死这些数字。

| 项目 | 当前取值与处理 |
|---|---|
| 请求体大小 | 所有接口 16 KiB；超出返回 413 `{"detail":"guest_app_request_too_large"}`（唯一不是 `error` 信封的错误） |
| 扣费 / 退款的解析 | 须是 JSON 对象；未知字段忽略；字段缺失或为 `null` 按空字符串处理（随后因格式不合法返回 400）；字段不是字符串返回 400 `invalid_input`；不会返回 422 |
| session / merge/verify 的解析 | 严格：重复键、`NaN`、多余字段、类型不符返回 422 `invalid_request`（不回显请求内容） |
| 限流 | 每个应用每分钟约 300 次、同一编号每分钟约 120 次（未登录访客的扣费与兑换）；`X-App-Token` 或编号不对的请求单独计数，约 30 次/分钟后返回 429，不占真实应用的额度 |
| 429 | **按状态码处理**（未登录访客 `guest_app_rate_limited`，登录用户 `rate_limited`），**不带 `Retry-After`**；从 2 秒起指数退避，上限 60 秒。限流服务不可用时返回 503 `guest_app_rate_limit_unavailable`，同样退避 |
| 建议客户端超时 | 25 秒左右（一次扣费平台内部可能先后等两个下游，各最多约 10 秒）。超时一律按「结果未知」处理，用原 ID 重试 |
| 缓存 | 成功响应带 `Cache-Control: private, no-store`；应用侧也不要缓存这些结果 |

**未登录访客的试用额度**：默认 200 光点、自首次访问起 30 天有效（以环境配置为准）。不够时扣费返回 402，到期返回 401 `login_required`，处理同 4.1。访客资料何时删除，由应用按自己的隐私政策决定。

## 5. 内容分享

> 适用：做应用首页分享或内容分享的应用。

### 5.1 生成分享

**分享按钮对所有人显示**：未登录访客点了，平台回 `share-result` 的 `error` 并弹出登录，应用结束 loading 即可。

**两种分享**：

- **应用首页分享**：不生成快照，同一用户的链接稳定；只发 `external-app:share` 加 `request_id`。
- **内容分享**：先由应用后端在当前会话下鉴权、生成并保存脱敏快照，再发登记的 `content_type` 和 `share_id`。未登录访客点内容分享时，应用可能已生成快照、随后被平台拒绝；这份快照从未发出链接，属于访客本人，保留即可。

**链接由平台生成**：`share_id` 匹配 `^[A-Za-z0-9_-]{1,128}$`。访客打开链接后 iframe 加载 `{jump_url 的源}/{path}/{share_id}`（1.2）。

**快照与重试**：用户每次主动再分享生成新快照、新 ID；同一次创建的网络重试和消息重送复用原快照。等待 `share-result` 超时时保留已生成的快照，不自动再建一份。

### 5.2 快照由应用保管

- **先脱敏再保存**：姓名、生日、地址、手机号、金额等个人信息默认隐藏或脱敏，第三方个人信息必须隐藏。公开接口不能返回原值，不能只靠 CSS 遮盖。
- **快照不可变**：源内容后续更新不影响旧链接。访客在可继续的快照上操作，产生的是访客自己的新数据，不能写回快照或使用分享人的权限。
- **有效期与失效**：应用负责每条快照的真实到期和失效页；`ttl_default` 为 `null` 表示长期。
- 公开静态文件和 CDN 缓存也不能绕过到期校验。

### 5.3 展示、搜索引擎与异常

- 内容分享页底部一律有平台的「立即试用」条；只读提示只在 `allow_continue:false` 时显示。可继续的快照在应用自己的输入区承接交互（首次真实操作发 `interaction-request`）。
- 快照过期 / 不存在时，应用在 iframe 里显示失效页；网络故障提供重试，不假报到期。
- 快照页响应设置 `X-Robots-Tag: noindex, nofollow` 和对应 meta，不能只靠 SPA 脚本补；身份、票据和操作相关响应用 `Cache-Control: private, no-store`。
- 平台公开解析接口 `GET https://lucimix.cn/api/public/external-apps/{app_key}/resolve?path={path}&share_id={share_id}` 可用来自查登记：200 且 `frame_url` 是自己的内容页才算通；404 `detail.code` 为 `content_type_unavailable` 表示内容类型未登记，`app_unavailable` 表示应用不存在或已下线。

## 6. 可选：并入通知

**什么时候发生**：未登录访客在应用里留下了数据（编号 A），之后用一个**以前用过本应用**的账号登录（该账号的编号是 B）。之后这个人的编号是 B，A 名下的数据留在原处。想把 A 的数据合进 B，就登记 `claim_callback_path`（1.1）并实现下面两步。

**第一步：收通知**。平台向 `{jump_url 的源}{claim_callback_path}` 发 HTTPS POST：

```json
{ "type": "merge", "ticket": "mt_…" }
```

**第二步：用票据换真实内容**（通知本身可能被伪造，必须验证）：

`POST https://lucimix.cn/api/apps/merge/verify`，请求头 `X-App-Token`。

```json
{ "ticket": "mt_…" }
```

成功 HTTP 200：

```json
{ "from_app_uid": "ag_…", "to_app_uid": "au_…" }
```

意思是「`from_app_uid` 已并入 `to_app_uid`」。应用把 `from` 名下的数据合并到 `to`，**不覆盖 `to` 已有的资料**。

| 规则 | 说明 |
|---|---|
| 票据有效期 | 60 秒，期内可重复验证。来不及就返回非 2xx，平台会换新票据重发 |
| 验证失败 | 404 `not_found`：票据不认识、过期或属于其他应用（不区分）；503 `unavailable`：稍后重试。两种情况都对通知返回非 2xx，让平台重发 |
| 送达判定 | 应用对通知返回任意 2xx 即视为送达，不再重发。所以**先验证、可靠保存合并任务，再返回 2xx**；合并本身可以异步做 |
| 重发节奏 | 非 2xx、超时（10 秒）或跳转都算失败，约 2、4、8 … 256 秒后再发，之后最长约每 4 分钟一次；自产生起 24 小时仍未送达则放弃 |
| 重复 | 同一对 `from → to` 可能因网络原因收到多次，合并要幂等 |
| 地址要求 | 公网 HTTPS、证书有效、直接返回（不 3xx）；不能只靠来源 IP 或请求头鉴别，以 verify 结果为准 |
| 地址生效 | 平台在访客**兑换票据时**记下当时登记的地址。改路径前，旧地址须继续可用一段时间；登记前就兑换过票据的访客不会收到通知 |

示例代码见[附录 B](#附录-b-后端最小示例) 第 4 部分。

## 7. 联调清单

用应用实际的数据库和下游服务在测试环境逐项验收；示例代码跑通不代表应用的持久化已经适配。

**基础**

| 检查项 | 验收结果 |
|---|---|
| 登记与嵌入 | 三个平台 origin（测试环境为对接人提供的对应地址）都能加载应用并完成握手；介绍页、`platforms` 正确 |
| 握手 | `ready` 立即发送；严格校验 origin / source；任何模式下收到票据都能换到 `app_uid`；票据过期能重新要票据；Token 不进浏览器 |
| 登录用户付费 | 确认报价后先扣再做；重复请求不重复扣费或执行；失败自动退款并持久重试到 200；主动停止按原价收费 |
| 光点不足 | 测试账号光点低于单价时扣费得 402，平台弹「光点不足」；充值后用户再点，用新 ID 成功 |

**未登录访客**

| 检查项 | 验收结果 |
|---|---|
| 免费与付费 | 未登录访客可直接用免费与付费功能；纯阅读不要票据，首次真实操作发 `interaction-request` 后拿到票据 |
| 额度不足 / 到期 | 试用额度不足（402）或到期（401 `login_required`）时，应用发 `insufficient-balance`，平台弹登录 |
| 新账号登录 | 访客以**新账号**登录后，应用重新握手拿到的 `app_uid` 不变，数据仍在，后续扣费扣正式光点 |
| 老账号登录 | 访客以**用过本应用的老账号**登录后，应用看到该账号原来的 `app_uid`；登记了并入通知的，收到通知并能验证出 `from → to` |
| 登录前扣费、登录后退款 | 访客扣费后登录，再对那笔扣费调退款，得 200 |

**内容分享（做了才验）**

| 检查项 | 验收结果 |
|---|---|
| 两类分享 | 登录用户分享首页 / 内容得到平台链接；未登录访客点分享时平台弹登录、应用结束 loading；重送复用快照，主动重分享新建快照 |
| 内容页 | 内容页地址按 1.2 规则可访问；在「立即试用」条和只读提示下显示完整；`allow_continue:true` 时访客可接着用 |
| 隐私与失效 | 先脱敏再存，公开接口没有原值；到期显示失效页；CDN 不绕过；noindex 生效 |

**故障恢复**

| 检查项 | 验收结果 |
|---|---|
| 结果未知 | 在调用扣费后、保存结果前杀掉进程，重启后用原 ID 恢复，不重复执行 |
| 错误分支 | 402 / 401 `login_required`：发 `insufficient-balance`，用户再点用新 ID；扣费 403 `app_unavailable`：重新握手换新 `app_uid`；其余 400 / 401 / 403 / 404 / 409：不循环重发、记录后人工核对；429 / 503 / 超时：原 ID 退避重试 |
| 退款 | 部分退、多次退均得 200；累计超额得 409 `conflict`（`reason:"refund_exceeds_original"`）；同退款 ID 重放得 `refunded:false` |
| 应用下线与凭证轮换 | 请对接人操作：下线后新扣费被拒（登录用户 401 `unauthenticated`，未登录访客 403 `guest_app_trial_unavailable`）；下线前扣费的退款对访客和登录用户都仍得 200；轮换后旧 Token 立即失效（登录用户扣费的退款得 401 `unauthenticated`，未登录访客扣费的退款得 404 `not_found`，见 4.3） |

**在测试环境触发各种情况**：以上都通过真实操作触发。试用额度不足可在新浏览器配置下提交高于试用额度的金额（如默认 200 光点时提交 `"201"`）；登录用户光点不足、应用下线等，请对接人协助调整。

**排查材料**：用 `app_key`、应用自己的订单 ID、`client_request_id`、时间和 HTTP 状态定位；对外提供前去除 Token、票据、用户资料和业务正文。

## 附录 A. 前端消息桥完整示例

以下代码放在应用 iframe 页面，覆盖握手、首次交互要票据、分享和重连。实现要点见 2.6。

**错误体约定（与附录 B 配套）**：Lucimix 的错误是 `{"error":{"code","message"}}`，只到应用后端为止；应用后端给自己前端的失败响应统一为 HTTP 状态码加 `{"code":"…"}`（附录 B 的 `HttpError(status, code)` 由应用的 HTTP 框架序列化成这个形状），下面的 `ownBackend` 读的就是这个 `code`。换成别的形状时两边一起改。

- 整段包在立即执行函数里，只通过 `window.lucimixBridge` 对外暴露，避免与页面自己的全局变量冲突（也可改成 `<script type="module">`）。
- 放在 `<body>` 末尾（或 `defer`），因为它在加载时读取 `document.body.dataset`。内容页可用 `data-share-id`、`data-allow-continue="true"` 标出当前快照（由应用自己的内容页填写）。

```javascript
(() => {
// 全部平台 origin（1.3）。测试环境换成对接人给的测试地址。
const LUCIMIX_ORIGINS = [
  "https://lucimix.cn", "https://www.lucimix.cn",
  "https://m.lucimix.cn",
];
const LUCIMIX_ORIGIN = (() => {
  const candidates = [];
  if (location.ancestorOrigins && location.ancestorOrigins.length) candidates.push(location.ancestorOrigins[0]);
  try { if (document.referrer) candidates.push(new URL(document.referrer).origin); } catch {}
  return candidates.find((origin) => LUCIMIX_ORIGINS.includes(origin)) || null; // 选不出则不连接
})();
const pageShareId = document.body.dataset.shareId || null;
const allowContinue = document.body.dataset.allowContinue === "true";

let context = null;     // 公开页的打开方式；只作展示
let session = null;     // { token, appUid }：应用自己的会话
let generation = 0;     // 每张新票据 +1，丢弃旧的异步结果
let lastTicket = null;
let exchanging = false; // 正在拿票据换会话
let readyAttempts = 0;
let bridgeObserved = false;
const shares = new Map();
const waiters = new Set();
const identityListeners = new Set();

function send(data) {
  if (window.parent !== window && LUCIMIX_ORIGIN) window.parent.postMessage(data, LUCIMIX_ORIGIN);
}
function showStatus(message) {
  const el = document.querySelector("[data-lucimix-status]");
  if (el) el.textContent = message;
}
async function ownBackend(path, body, identity = session) {
  const response = await fetch(`/your-backend/${path}`, {
    method: "POST",
    credentials: "omit",
    headers: {
      "Content-Type": "application/json",
      ...(identity ? { Authorization: `Bearer ${identity.token}` } : {}),
    },
    body: JSON.stringify(body),
  });
  const data = await response.json().catch(() => ({}));
  if (!response.ok) {
    const error = new Error(`应用请求失败 (${response.status})`);
    error.status = response.status;
    error.code = data.code; // 应用后端回的 {code}（附录 B 从 Lucimix 的 error.code 转出），如 "login_required"
    throw error;
  }
  return data;
}
function ready() {
  if (window.parent === window || !LUCIMIX_ORIGIN) return;
  send({ type: "external-app:ready" });
  readyAttempts++;
  if (!bridgeObserved && readyAttempts < 4) {
    setTimeout(() => { if (!bridgeObserved) ready(); }, 2500);
  }
}
function waitForSession() {
  if (session) return Promise.resolve(session);
  return new Promise((resolve, reject) => {
    const waiter = { resolve, reject };
    waiters.add(waiter);
    setTimeout(() => {
      if (waiters.delete(waiter)) reject(new Error("连接未完成，请重试"));
    }, 30000); // 比后端调 Lucimix 的 25 秒超时略长
  });
}
// 用户第一次真实保存 / 生成 / 修改前调用；已有身份直接返回。
async function ensureSessionForUserAction() {
  if (session) return session;
  if (context?.mode === "home") {
    send({ type: "external-app:interaction-request", request_id: crypto.randomUUID() });
  } else if (context?.mode === "snapshot") {
    if (!pageShareId || !allowContinue) throw new Error("请打开应用首页后使用");
    send({ type: "external-app:interaction-request", request_id: crypto.randomUUID(),
           share_id: pageShareId });
  } else if (!exchanging) {
    // business 模式：票据随 ready 自动到达。之前兑换失败（503 / 429 / 票据过期等）就在这次操作时
    // 重发 ready 要新票据（2.6）；平台 1 秒内只回应一次，不会刷票据
    lastTicket = null;
    send({ type: "external-app:ready" });
  }
  return waitForSession();
}
// 扣费返回 402 / 401 login_required、或应用自有试用次数用完时调用。不带任何字段。
function notifyInsufficientBalance() {
  send({ type: "external-app:insufficient-balance" });
}
// 分享按钮对所有人显示；未登录访客由平台回 error 并弹登录。
function share(contentType, shareId) {
  const requestId = crypto.randomUUID();
  const message = { type: "external-app:share", request_id: requestId };
  if (contentType !== undefined || shareId !== undefined) {
    if (!/^[a-z][a-z0-9_-]{0,31}$/.test(contentType || "") ||
        !/^[A-Za-z0-9_-]{1,128}$/.test(shareId || "")) throw new Error("快照参数无效");
    Object.assign(message, { content_type: contentType, share_id: shareId });
  }
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      shares.delete(requestId);
      reject(new Error("分享暂不可用，请重试"));
    }, 10000);
    shares.set(requestId, { resolve, reject, timer });
    send(message); // 调用方用 finally 结束按钮 loading
  });
}
function exchangeTicket(ticket) {
  const revision = ++generation;
  const previousUid = session?.appUid ?? null;
  session = null;
  exchanging = true;
  ownBackend("session", { ticket }, null).then((verified) => {
    if (generation !== revision) return; // 期间又来了新票据
    exchanging = false;
    if (typeof verified.session_token !== "string" || typeof verified.app_uid !== "string") {
      throw new Error("应用身份响应无效");
    }
    session = { token: verified.session_token, appUid: verified.app_uid };
    if (session.appUid !== previousUid) {
      for (const fn of identityListeners) fn(session); // 换人了：页面按新编号重新加载数据
    }
    for (const waiter of waiters) waiter.resolve(session);
    waiters.clear();
    showStatus("");
  }).catch((error) => {
    if (generation !== revision) return;
    session = null;
    exchanging = false; // business 页：用户下次操作时 ensureSessionForUserAction 重发 ready
    if (error.code === "login_required") notifyInsufficientBalance(); // 平台会弹登录
    else showStatus("身份连接失败，请重试");
    for (const waiter of waiters) waiter.reject(error);
    waiters.clear();
  });
}

window.addEventListener("message", (event) => {
  if (!LUCIMIX_ORIGIN || event.origin !== LUCIMIX_ORIGIN || event.source !== window.parent) return;
  const data = event.data;
  if (!data || typeof data !== "object") return;
  if (data.type === "external-app:context") {
    if (!["home", "snapshot"].includes(data.mode)) return;
    bridgeObserved = true;
    context = data; // 只作展示；不因 authenticated=true 获得任何权限
    return;
  }
  if (data.type === "external-app:ticket") {
    if (typeof data.ticket !== "string" || !data.ticket) return; // 其他字段（kind 等）忽略
    if (lastTicket === data.ticket) return;
    lastTicket = data.ticket;
    bridgeObserved = true;
    exchangeTicket(data.ticket);
    return;
  }
  if (data.type === "external-app:share-result") {
    const pending = shares.get(data.request_id);
    if (!pending || !["opened", "error"].includes(data.status)) return;
    shares.delete(data.request_id);
    clearTimeout(pending.timer);
    if (data.status === "opened") pending.resolve();
    else pending.reject(new Error("分享暂不可用，请重试"));
  }
  // 其他消息一律忽略
});

if (window.parent === window || !LUCIMIX_ORIGIN) showStatus("请从 Lucimix 进入本应用");
else ready(); // 装好监听后立即发送

window.lucimixBridge = Object.freeze({
  ensureSessionForUserAction,
  currentSession: () => session,
  onIdentityChange: (fn) => { identityListeners.add(fn); return () => identityListeners.delete(fn); },
  shareApp: () => share(),
  shareSnapshot: (contentType, shareId) => share(contentType, shareId), // shareId 须先由后端保存快照后返回
  balanceChanged: () => send({ type: "external-app:balance-changed" }),
  insufficientBalance: notifyInsufficientBalance,
  reconnect: () => { // 票据兑换失败、或扣费返回 403 app_unavailable 时调用：丢掉旧身份重新握手
    session = null;
    readyAttempts = 0;
    bridgeObserved = false;
    lastTicket = null;
    ready(); // business 页与登录用户会直接收到新票据；未登录访客下次操作时由 ensureSessionForUserAction 再要
  },
});
})();
```

页面代码的典型用法：

```javascript
// actionKey 代表"这一次点击"：重试沿用同一个，后端据此回读原订单，不会重复扣费（附录 B 第 2 部分）。
async function onGenerateClick(input, actionKey = crypto.randomUUID()) {
  await lucimixBridge.ensureSessionForUserAction();     // 首次操作时才要身份
  const r = await callMyBackend("generate", { input, action_key: actionKey }); // 应用自己的接口，内部按附录 B 扣费
  if (r.state === "insufficient") lucimixBridge.insufficientBalance();
  else if (r.state === "reidentify") lucimixBridge.reconnect(); // 编号已失效：换新身份后用户再点
  else if (r.state === "pending") {
    // 扣费结果未知或平台正忙（附录 B 的 pending）：不算失败，也不能换新 actionKey
    showRetry("处理中，请稍候", () => onGenerateClick(input, actionKey)); // 应用自己的提示 + 重试按钮
  }
  if (r.balanceChanged) lucimixBridge.balanceChanged();
}
```

## 附录 B. 后端最小示例

以下为 Node.js 18+ 示例：兑换票据建立会话、确认报价后扣费、执行业务、失败退款（outbox 后台重试），以及可选的并入通知。`db.*`、`signAppSession`、`priceFor`、`runBusiness` 是应用自己的实现，示例只表达调用顺序和状态转换；HTTP 框架、数据库与会话签名方式按应用现有技术栈替换。

依赖：`npm i ulid`（生成扣费、退款流水号）。

```javascript
import { ulid } from "ulid";

const LUCIMIX_BASE = process.env.LUCIMIX_BASE_URL ?? "https://lucimix.cn"; // 测试环境向对接人索取
const APP_TOKEN = process.env.LUCIMIX_APP_TOKEN;                         // 只放后端密钥管理

// 给自己前端的失败响应：HTTP status + {"code": code}（附录 A 的 ownBackend 读 data.code）。
// Lucimix 的 {"error":{"code","message"}} 不原样透传给浏览器，只转出需要的 code。
class HttpError extends Error {
  constructor(status, code) { super(code); this.status = status; this.code = code; }
}

// 调用 Lucimix。网络错误、超时会抛异常：调用方必须按"结果未知"处理，不能当作失败。
async function callLucimix(path, body) {
  const res = await fetch(LUCIMIX_BASE + path, {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-App-Token": APP_TOKEN },
    body: JSON.stringify(body),
    redirect: "error",
    signal: AbortSignal.timeout(25_000), // 4.5 建议的客户端超时
  });
  const data = await res.json().catch(() => ({}));
  return {
    status: res.status,
    code: data?.error?.code,
    data,
    retryAfter: Number(res.headers.get("Retry-After")) || 0,
  };
}

// ---------- 1. 兑换票据，建立应用自己的会话 ----------
// 对应附录 A 的 POST /your-backend/session，请求体 {ticket}
export async function createSession({ ticket }) {
  if (typeof ticket !== "string" || !ticket) throw new HttpError(400, "ticket_required");
  let r;
  try {
    r = await callLucimix("/api/apps/session", { ticket });
  } catch {
    throw new HttpError(503, "session_unavailable");
  }
  if (r.status === 200) {
    const { app_uid, nickname } = r.data;
    await db.upsertUser({ app_uid, nickname }); // app_uid 列宽至少 128
    return { app_uid, session_token: signAppSession({ app_uid }) };
  }
  if (r.code === "login_required") throw new HttpError(401, "login_required"); // 前端发 insufficient-balance
  if (r.status === 401) throw new HttpError(401, "ticket_invalid");            // 前端重新要票据
  if (r.status === 403) throw new HttpError(403, "app_unavailable");
  throw new HttpError(503, "session_unavailable");
}
// 之后每个业务接口都校验 Authorization 里的应用会话，从会话取 app_uid，
// 不接受浏览器传来的编号、价格或商品。

// ---------- 2. 用户确认报价后发起付费动作 ----------
// actionKey 由前端为"这一次点击"生成，网络重试时保持不变；用户再点一次就是新的 actionKey。
export async function startPaidAction(appUid, actionKey, input) {
  const quote = priceFor(input); // 后端按价目表算，例如 { credits: "10", product_name: "AI 解读 × 1" }
  const order = await db.tx(async (t) => {
    const existing = await t.findOrderByActionKey(appUid, actionKey);
    if (existing) return existing; // 重试时回读原订单，不新建流水号
    return t.insertOrder({
      app_uid: appUid, action_key: actionKey, input,
      client_request_id: ulid(), // 只在建单时生成一次，之后所有重试都用它
      credits: quote.credits, product_name: quote.product_name,
      state: "charging",
    });
  });
  return driveOrder(order);
}

// 推进一张订单。用户重试、后台补偿任务都调用它，可以重复调用。
export async function driveOrder(order) {
  if (order.state !== "charging") return { state: order.state };
  let r;
  try {
    r = await callLucimix("/api/apps/charge", {
      client_request_id: order.client_request_id,
      app_uid: order.app_uid,
      credits: order.credits,
      product_name: order.product_name,
    });
  } catch {
    return { state: "pending" }; // 超时/网络错误：结果未知，保持 charging，后台用原 ID 重试
  }
  if (r.status === 200) {
    // 扣费已确认（charged:false 是重放，同样代表已扣）。CAS：只有抢到 charging→running 的请求才执行。
    if (!(await db.casOrderState(order.id, "charging", "running"))) return { state: "pending" };
    order.state = "running";
    await execute(order);
    return { state: "done", balanceChanged: true }; // 前端调 lucimixBridge.balanceChanged()
  }
  if (r.status === 402 || r.code === "login_required") {
    await db.setOrderState(order.id, "not_charged"); // 未扣费，这个 ID 作废；用户再点会建新订单、新 ID
    return { state: "insufficient" };                // 前端调 lucimixBridge.insufficientBalance()
  }
  if (r.status === 429 || r.status >= 500) {
    return { state: "pending", retryAfter: r.retryAfter }; // 含 503 app_identity_converting：同一 ID 稍后重试
  }
  if (r.status === 403 && r.code === "app_unavailable") {
    await db.setOrderState(order.id, "not_charged"); // 这个 app_uid 已失效，未扣费（第 3 章）
    return { state: "reidentify" };                  // 前端调 lucimixBridge.reconnect() 换新 app_uid 后再点
  }
  await db.setOrderState(order.id, "rejected");      // 其余 400/401/403/404/409：未扣费，记录并人工核对
  await db.flagForReview(order.id, r.status, r.code);
  return { state: "rejected" };
}

async function execute(order) {
  let failed;
  try {
    await runBusiness(order.input, { idempotencyKey: order.client_request_id }); // 下游也用稳定幂等键
    failed = false;
  } catch (e) {
    if (e.cancelledByUser) failed = false;      // 执行中主动停止：按原价收费，不退款
    else if (e.definitelyFailed) failed = true; // 确定失败：退款
    else return;                                // 结果未知：保持 running，查下游原结果后再定
  }
  // 业务结果与退款单（含固定的退款 ID）同一事务写入 —— 这就是 outbox
  await db.tx(async (t) => {
    await t.saveOrderResult(order.id, failed ? "failed" : "done");
    if (failed) {
      await t.insertRefund({
        order_id: order.id,
        client_request_id: ulid(),                         // 退款 ID 建单时生成一次，之后不变
        original_transaction_id: order.client_request_id,  // = 扣费返回的 transaction_id
        credits: order.credits,
      });
    }
  });
}

// ---------- 3. 后台定时任务：发送待退款 ----------
export async function flushRefunds() {
  for (const refund of await db.pendingRefunds()) {
    let r;
    try {
      r = await callLucimix("/api/apps/refund", {
        client_request_id: refund.client_request_id, // 始终用同一个退款 ID 重试
        original_transaction_id: refund.original_transaction_id,
        credits: refund.credits,
        reason: "生成失败",
      });
    } catch {
      continue; // 超时/网络错误：结果未知，下轮用相同参数重试
    }
    if (r.status === 200) await db.markRefundConfirmed(refund.id);
    else if ([400, 401, 403, 404, 409].includes(r.status)) await db.flagForReview(refund.id, r.status, r.code); // 不再自动重试，人工核对（401 多为 Token 无效或已轮换，见 4.3）
    // 429 / 5xx：保持，下轮重试
  }
}

// ---------- 4. 可选：并入通知（第 6 章） ----------
// 挂在登记的 claim_callback_path 上，例如 POST /api/lucimix/guest-claim。返回值是 HTTP 状态码。
export async function onMergeNotice(body) {
  if (body?.type !== "merge" || typeof body.ticket !== "string") return 400;
  let r;
  try {
    r = await callLucimix("/api/apps/merge/verify", { ticket: body.ticket });
  } catch {
    return 503; // 非 2xx：平台会换新票据重发
  }
  if (r.status !== 200) return 503;
  const { from_app_uid, to_app_uid } = r.data;
  await db.enqueueMerge({ from: from_app_uid, to: to_app_uid }); // 先可靠保存（幂等），合并异步做，不覆盖 to 已有资料
  return 200;
}
```

示例省略的部分：

- 处于 `charging` 和 `running` 的订单需要后台补偿任务定期调用 `driveOrder` 或查询下游原结果，覆盖进程重启；`retryAfter` 大于 0 时至少等这么久。
- 订单超过 24 小时仍是 `charging` 且无法确认是否入账时，停止重试并联系对接人核账。
- `rejected` 的订单中，409 带 `reason:"charge_voided"` 的可以换新 ULID 重新扣；退款 409 带 `reason:"refund_rejected"` 的可以换新退款 ID 重新发起。
- 失败动作的退款在 `flushRefunds` 后台完成，晚于前端收到业务结果。应用应让前端能查到订单的退款状态（例如轮询自己的订单接口），退款确认后再调一次 `lucimixBridge.balanceChanged()`。

## 附录 C. 需向对接人确认的事项

| 事项 | 说明 |
|---|---|
| 测试环境地址 | 测试环境的平台 origin（PC 与 H5）与 Lucimix 接口基地址 |
| `app_key`、`X-App-Token` | 经安全渠道交付，Token 只显示一次；测试与正式各一套 |
| 登记的内容类型 | `content_type`、`path` 等最终值；确认内容页 `{jump_url 的源}/{path}/{share_id}` 已上线 |
| 是否接并入通知 | 接则登记 `claim_callback_path` |
| 联调测试账号与测试光点 | 验证扣费、退款和不足分支；余额不足场景需请对接人调整光点 |
| 故障核账渠道与联系人 | 流水超出时间窗或财务事实无法确定时使用 |

## 附录 D. 常见对接问题

以下是实际对接中出现过的问题，以运势助手为例。提交登记资料、上线前请逐条自查。

### D.1 问题清单

| # | 现象 | 原因 | 怎么做 | 怎么自查 |
|---|---|---|---|---|
| 1 | 应用内点「生成链接」，应用提示「分享暂不可用，请重试」 | 应用发了内容分享（带 `content_type`），但平台没有登记这个内容类型，回 `share-result` 的 `error` | 每种可分享的内容都登记一组 `content_types`（1.1）；应用代码里发的类型名、内容页路由与登记值完全一致 | `curl "https://lucimix.cn/api/public/external-apps/bazi/resolve?path=s&share_id=abc"`：404 `content_type_unavailable` 表示未登记；200 且 `frame_url` 是 `https://baziapp.lucimix.cn/s/abc` 才算通 |
| 2 | 从 `www.lucimix.cn` 进入应用白屏，`lucimix.cn` 正常 | CSP `frame-ancestors` 和消息白名单只写了 `lucimix.cn`、`m.lucimix.cn` | 1.3 列出的**全部**平台 origin 都写进 CSP 和 `LUCIMIX_ORIGINS` | `curl -sI https://{应用域名}/ \| grep -i frame-ancestors`，三个平台 origin 都在 |
| 3 | 只读分享页提示「这是一个已分享的运势助手分享，只能查看。」读起来重复 | 内容名称 `name` 会拼进只读提示 | 用名词作内容名称，如「内容」「命盘」「报告」，不要用「分享」 | 打开一条只读分享链接看顶部提示 |
| 4 | 更新登记资料后，介绍页截图没了、文案变短 | 提交的 `intro` 会**整体替换**线上介绍；提交了 `"screenshots": []` | 不改介绍时登记资料里**不要带 `intro`**；要改就提交完整的新介绍（含截图） | 录入后在 Lucimix 打开介绍页核对 |
| 5 | 交给对接人的材料里出现 `<服务器上的实际值>` 之类的占位符 | 用 AI 生成材料时把模板占位符原样留下 | 提交前搜一遍 `<`、`待补`、`TODO`，确认都换成了真实值 | — |

### D.2 提交登记资料前的自查清单

- [ ] `app_key`、`name`、`icon_url`、`jump_url` 与线上一致；`icon_url` 能直接访问
- [ ] 不改介绍就不带 `intro`；要改就带完整介绍和截图
- [ ] 每种会分享的内容都有一组 `content_types`；内容页 `{jump_url 的源}/{path}/{share_id}` 已上线
- [ ] 内容名称是名词，不是「分享」
- [ ] 接并入通知时：`claim_callback_path` 是公网 HTTPS、不跳转，且已实现第 6 章两步
- [ ] CSP 与 `LUCIMIX_ORIGINS` 包含 1.3 的全部平台 origin
- [ ] 应用后端只配置了 `https://lucimix.cn` 一个平台地址和 `X-App-Token`
- [ ] 材料里没有占位符
