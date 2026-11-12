---
name: package-deploy-bundle
description: 用户侧 Agent 准备部署包的唯一说明（同时是部署契约）：Python 或 Node.js 后端、按外部应用接入指南接入 Lucimix（只支持认证、扣费、退款）、业务配置声明、数据与迁移、DEPLOYMENT.md 和 app.zip；不操作平台服务器、不发布应用。
---

# SkillA：准备应用部署包

本文是部署契约的唯一版本（第四版，2026-11-12 修订），部署方的 SkillB 和平台预检都按它检查。你负责应用代码、依赖、认证与扣费、`start.sh` 和适用的迁移，最终交付 `app.zip`。部署方按声明构建、预览和发布，**不改你的源码**；包有问题时给整改说明，由你修订后重新打包。已有可部署包时，部署请求交 SkillB；只是安装技能、读文档或提问时，不启动部署。

**本版只支持认证、扣费、退款三项对接**，按同目录下的[外部应用接入指南](references/external-app-integration-guide.md)实现，本文不复述协议，只写平台托管下的差异：

| 对接 | 实现内容（指南章节） |
|---|---|
| 认证 | 登记资料取得 `app_key` 与 `X-App-Token`（第 1 章）；iframe 嵌入与消息桥 `ready` / `ticket` / `interaction-request`（第 2 章、附录 A）；`POST /api/apps/session` 兑换身份（第 3 章） |
| 扣费 | `POST /api/apps/charge`（第 4.1、4.2、4.4、4.5 节）；消息桥按需发 `balance-changed` / `insufficient-balance` |
| 退款 | `POST /api/apps/refund`（第 4.3 节） |

**内容分享（指南第 5 章）和并入通知（第 6 章、`merge/verify`）本版不支持，不要实现**：不发 `share` 消息，不做分享快照和内容页，不在登记资料里填内容页 `path` 或 `claim_callback_path`，不调用 `merge/verify`。联调清单（第 7 章）和附录 D 只做与上面三项相关的条目。登记资料里内容分享一组字段“要做内容分享才填”、`claim_callback_path` 是可选项，不填不影响登记。

平台不理解接入协议：Lucimix 的地址和凭证是应用自己声明的业务配置（第 3 节），指南以后怎么变，都只需改应用，不需要等平台改动。

## 1. 第一版支持范围

- **后端是 Python 或 Node.js**。`start.sh` 的最后一条命令必须是 `exec` 启动应用进程：
  - Python：`python`/`python3`，或 `gunicorn`、`uvicorn`、`hypercorn`、`daphne`、`waitress-serve`、`granian`；
  - Node.js：`node`，或 `node_modules/.bin/` 下的启动命令（例如 `exec node_modules/.bin/next start -p 8080 -H 0.0.0.0`）。不要用 `npm start`、`npx` 启动，让应用进程自己收到 SIGTERM。
  - 其他后端（Java、PHP、二进制程序等）不支持，部署方会请你修包。
- **前端不限**。可以由后端直接提供构建出的静态文件，也可以用 Next.js 这类前后端一体的框架。一个容器只跑一个后端进程，不要 Python 和 Node 各起一个服务。
- `build_environment` 只能声明 `python` 和 `node`，运行时用的就是构建时的版本。可写版本或范围，例如 `{"python":"^3.11","node":"22"}`。
- 本地持久数据只支持 SQLite。读写远程数据库时，连接配置写进 `runtime_config`，平台不管理远程库（见 6.2）。

## 2. 运行形态

| 要求 | 说明 |
|---|---|
| 端口与健康检查 | 监听 `0.0.0.0:8080`；`/health` 匿名可访问，只返回就绪状态，不含业务数据或秘密。平台起容器后最多等 120 秒，`/health` 返回 200 即视为启动成功；期间容器退出就判定失败。平台只检查 `/health`，不检查其他接口 |
| 启动脚本 | 包根目录的 `start.sh`，LF 换行，可重复启动，最后 `exec` 前台应用进程；平台原样安装为 `/start.sh`，工作目录 `/app`。不在后台启动任何进程；定时任务写在应用进程里（见第 7 节） |
| 停止 | 应用进程正确处理 SIGTERM：停止接收写入、排空请求、关闭数据库；平台等待 15 秒 |
| 文件系统 | 根文件系统只读；`/tmp` 是 32 MiB、noexec 的临时目录；持久数据只能写 `APP_DATA_DIR`（`/data`），不能写源码目录或镜像层。`/data` 位于 NFS：SQLite 不能用 WAL（用默认的 DELETE 日志模式），应用只开 1 个 worker。日志只输出到 stdout/stderr（部署方用 `dlogs` 查看），不要写日志文件；启动时不要在 `/app` 下建目录或写文件，否则应用一启动就因只读而退出 |
| 时区 | 容器时区是北京时间（平台注入 `TZ=Asia/Shanghai`，应用不能声明 `TZ`），代码可以直接用本地时间；对时区有特殊要求的，在代码里显式指定 |
| 入口路径 | 正式 `https://app.aibedrock.cn/{app_name}/`，预览 `/{app_name}-preview/`。网关剥离前缀，应用按 `APP_BASE_PATH` 生成资源、API 和跳转路径；浏览器存储按应用和环境隔离 |
| 单副本 | 正式只有一个副本；预览是独立副本，数据和正式隔离 |

**同一个镜像先当预览、再当正式**：确认上线时发布的就是预览验过的那个镜像，不重新构建，所以路径前缀（`APP_BASE_PATH`）、业务配置都只能在运行时读取，不能在构建时写死。Node.js 应用尤其注意：

- Next.js 的 `basePath`、`assetPrefix` 和 `NEXT_PUBLIC_*` 变量在 `next build` 时就固定了，不能直接用来区分预览和正式；前缀要按 `APP_BASE_PATH` 在运行时处理。
- 运行时不能写 `/app`：Next.js 的图片优化缓存和 ISR 会写 `.next/cache`，关掉图片优化（`images: { unoptimized: true }`），不用 ISR。
- 交付前用只读根文件系统、两个不同的 `APP_BASE_PATH` 各启动一次自测，例如 `docker run --read-only --tmpfs /tmp -e APP_BASE_PATH=/demo-preview/ …`，页面、静态资源和接口都能正常加载。

## 3. 平台注入的变量与 Lucimix 配置

平台只注入入口与运行信息：

| 变量 | 用途 |
|---|---|
| `APP_PUBLIC_ORIGIN`、`APP_BASE_PATH`、`APP_PUBLIC_URL` | 本应用对外 origin、路径前缀和完整入口 |
| `APP_DATA_DIR` | 持久数据目录 `/data` |
| `APP_ENV` | `preview` 或 `prod` |
| `APP_RELEASE_PHASE` | 预览为 `acceptance`，正式为 `active` |

**Lucimix 接入所需的值由应用自己在 `runtime_config` 里声明**（第 5 节）。平台按声明生成“应用环境变量”表单，预览和正式分别保存，加密保管，只注入对应实例；秘密值不进模型提示、日志、ZIP 和镜像。变量名、个数都由应用定，平台不解析。指南要求哪些值，就声明哪些值，例如：

| 推荐名称 | 类型 | 内容 |
|---|---|---|
| `LUCIMIX_BASE_URL` | `url` | 指南里的接口基地址；可设默认 `https://test.lucimix.cn`，正式填 `https://lucimix.cn` |
| `LUCIMIX_APP_KEY` | `text` | 本应用的 `app_key` |
| `LUCIMIX_APP_TOKEN` | `secret` | 应用凭证 `X-App-Token`，只在后端使用 |
| `LUCIMIX_ORIGINS` | `text` | 指南要求列入 CSP 和消息白名单的**全部**平台 origin，空格分隔 |

- 缺必需的值时拒绝启动，不降级为匿名访问；格式（如 origin 必须是 HTTPS）由应用自己校验。
- 指南新增或改名参数时，改代码和 `runtime_config` 声明即可；用户在表单里补填新值。
- 不只是 Lucimix：应用连接的**所有**外部服务（大模型的地址和模型名、第三方 API、远程数据库）都要在 `runtime_config` 里声明，不要依赖代码里的默认地址。默认值指向的服务从执行服务器可能访问不到。

**登记资料**：用户本人按指南第 1 章向 Lucimix 对接人提交登记资料，取得 `app_key` 和只显示一次的 `X-App-Token`，再填进“应用环境变量”表单。登记时 `jump_url` 填平台正式入口 `https://app.aibedrock.cn/{app_name}/`。你可以在 DEPLOYMENT.md 正文里起草登记 JSON，交用户核对后提交，草稿里不能留占位符。用户侧 Agent 不需要也不应接触真实凭证，聊天、ZIP、源码、日志里只出现变量名。

## 4. 平台托管下的差异

按指南实现时，下面几处是平台托管特有的：

### 4.1 Lucimix iframe 首次握手

- **会话只放内存、不依赖 Cookie**：应用页面嵌在 Lucimix 的 iframe 里，第三方 Cookie 可能被拦截。
- **首次握手必须覆盖平台加载超时**：应用装好 `message` 监听后立即发送 `external-app:ready`；在收到 `external-app:ticket` 或 `external-app:context` 前，至少再重发 3 次，间隔应让最后一次发送落在 iframe 挂载后的 8 秒超时之前（参考实现为 2.5 秒，共在约 0、2.5、5、7.5 秒发送）。收到平台消息后停止首次握手重试。不能只在用户操作时发送 `ready`，也不能只发送一次，否则父页面尚未装好监听时会错过消息，显示“应用打不开”。遵循[外部应用接入指南附录 A](references/external-app-integration-guide.md)的消息来源校验和固定 `targetOrigin`。
- **预览验收必须在 Lucimix iframe 内完成**：从 Lucimix 的 `/apps/{app_key}` 打开预览，确认加载遮罩消失、应用页面出现，并完成 `ready` → `ticket` → 应用后端 `POST /api/apps/session` 换取身份。直接打开 `https://app.aibedrock.cn/{app_name}-preview/` 只会显示“请从 Lucimix 进入本应用”，不算握手或业务验收通过。

### 4.2 其他托管差异

- **路径前缀**：网关剥掉 `/{app_name}/`（预览是 `/{app_name}-preview/`）再转发，应用按 `APP_BASE_PATH` 生成资源、API 和跳转路径。
- **预览也会真实扣费**：预览和正式都连真实 Lucimix。如果两边填的是同一个应用凭证，预览里的付费操作扣真实光点，自测后用应用的退款功能退回；在 Lucimix 里把入口切到预览期间所有用户都会进预览，验完尽快切回。
- **扣费开关**：建议在 `runtime_config` 里声明一个扣费方式开关（例如 `BILLING_MODE`，`lucimix` 真实扣费、`ledger` 只记本地流水），出问题时改配置就能停掉真实扣费，不用改代码。

## 5. 业务配置声明

在 DEPLOYMENT.md 唯一的 deployment JSON 里填 `runtime_config`，逐项声明应用实际读取的业务环境变量。平台按声明生成“应用环境变量”表单，预览和正式分别保存，再注入对应实例。没有业务配置也要显式写 `{"version":1,"fields":[]}`。

| 字段 | 说明 |
|---|---|
| `name` | 必填，大写字母、数字、下划线，与代码读取的名称一致 |
| `type` | 必填，`text`、`secret`、`url`、`integer`、`boolean`、`enum` |
| `label` / `description` | 表单上显示的名称和说明，建议填 |
| `required` | 是否必填 |
| `default` | 默认值 |
| `choices` | `enum` 的选项 |
| `required_when` / `fallback_to` | 可选：`{"name":…, "equals":…}` 时才必填；缺值时改用另一个字段 |

- API Key、口令、含凭据的连接串用 `secret`，真实值不进源码、`.env`、`start.sh` 或镜像。
- 不能用平台变量名：`APP_*`、`DEPLOY_*` 前缀，以及 `PATH`、`HOME`、`PORT`、`PYTHONPATH`、`DATABASE_URL` 等。

## 6. 数据与迁移（有持久数据时）

**数据从哪里来**：平台只给容器挂 `/data` 一个目录，不会额外挂载其他目录，也不提供外部的只读数据路径。应用用到的每一份数据，都要属于下面四种来源之一：

| 来源 | 适用 | 做法 |
|---|---|---|
| 打进包里 | 小的静态数据（字典、模板、提示词） | 放在源码目录，只读使用 |
| 应用首次启动时自己导入 `/data` | 历史数据、外部维护的参考库（如行情库） | 见 6.1 |
| 连远程服务 | 远程数据库、第三方 API | 见 6.2，地址与凭证在 `runtime_config` 声明 |
| 应用运行时自己生成 | 用户数据、业务状态 | 写 `APP_DATA_DIR` |

需要持续更新的参考库，导入后由应用按第 7 节在进程内定时同步，同步结果写回 `/data` 里的同一个文件；不要写“由部署环境提供路径”或“由外部脚本维护”，平台没有这种能力。

- 数据库和上传文件都在 `APP_DATA_DIR` 下，例如 `/data/db/app.sqlite3`、`/data/uploads/`。
- `/data` 是 NFS 上的目录，预览和正式各有独立的数据目录。预览数据长期保留并累积，不会在每次预览前重置；正式数据不会复制给预览。
- 平台不往 `/data` 写任何东西。应用首次启动面对空目录，由应用幂等初始化（要导入历史数据的见 6.1）；再次启动检测已有数据，禁止清库、覆盖已导入的数据或重建用户。
- 有 schema 变化时，由应用在启动时自己迁移，能重复执行。迁移不会先在正式数据上演练：预览用的是自己的数据目录，正式数据在新版本启动时就地迁移。平台不做发布前快照，正式数据由存储快照兜底，数据改坏需要恢复时找维护人。
- SQLite 不要用 WAL（`/data` 在 NFS 上），用默认的 DELETE 日志模式，并设置 `busy_timeout`。
- 通知、webhook 等外部副作用在 `runtime_config` 里按环境配置，预览留空。

### 6.1 历史数据

旧系统已有数据、或应用要用一份外部维护的参考库（如行情库）时，由应用在首次启动时自己下载并导入到 `/data`。平台不导入、不清空、不覆盖数据目录；数据怎么来、什么格式（tar 包、单个库文件、SQL 脚本、对象存储、第三方 API）都由应用决定。

用户侧准备：

1. 旧系统停写后，对每个 SQLite 库执行 `PRAGMA journal_mode=DELETE;` 和 `PRAGMA integrity_check;`。
2. 按 `/data` 的目录结构整理数据并打包：`tar -czf seed.tar.gz -C <数据目录> .`，包里的路径就是 `/data` 下的路径（如 `db/app.sqlite3`、`uploads/…`）。只有一个文件时也可以不打包。
3. 上传到 COS 等能用 http/https 下载的地方，生成带有效期的链接。预览和正式在各自首次启动时下载：正式的链接要撑到用户确认上线之后。
4. `runtime_config` 声明一个 `secret` 字段（建议名 `SEED_DATA_URL`），用户在“应用环境变量”里预览和正式分别填链接（预览可以填样本数据）。链接属于凭证，不写进 DEPLOYMENT.md。

应用侧要求：

- **只在空目录导入，导入完写标记**：有标记就跳过；没有标记但目录里已有别的东西，也跳过，绝不覆盖。
- **先写临时目录再移入**：中途失败或进程被杀，下次启动能接着完成或重来，不会留下半份数据。
- **导入要在其他初始化之前**：先导入、再建表或迁移，否则目录先被应用自己写成非空，导入会被跳过。
- **不能卡住启动**：平台最多等 `/health` 120 秒。导入可能超过一分钟时，先启动 Web 服务，在后台线程导入；导入期间 `/health` 照常返回 200，业务接口提示“数据初始化中”。
- 导入失败时打印明确的错误（下载失败、链接过期、格式不对），不要建一个空库顶替；下次启动会重试。
- 预览要重新导入时，需要清空预览数据目录后重启，这一步找平台维护人。

参考实现（Python 3.11.4 及以上，只用标准库；链接是单个文件时，把解包那一行换成把 `resp` 写到 `tmp` 下对应路径；Node.js 应用按同样的步骤实现）：

```python
import os, shutil, tarfile, urllib.request
from pathlib import Path

DATA = Path(os.environ['APP_DATA_DIR'])
MARK, TMP = DATA / '.seeded', DATA / '.seed-tmp'

def seed_if_empty():
    """首次启动导入历史数据，可重复调用；有标记或目录已有数据时什么都不做。"""
    url = os.environ.get('SEED_DATA_URL')
    if MARK.exists() or not url:
        return
    if not (TMP / '.complete').exists():
        if any(p != TMP for p in DATA.iterdir()):
            return  # 已有数据：不导入、不覆盖
        shutil.rmtree(TMP, ignore_errors=True)
        TMP.mkdir()
        with urllib.request.urlopen(url, timeout=60) as resp, tarfile.open(fileobj=resp, mode='r|gz') as tar:
            tar.extractall(TMP, filter='data')  # 拒绝绝对路径、..、链接和设备文件
        (TMP / '.complete').touch()
    # 解包已完整：逐个移入；中途中断时下次启动从这里接着移
    for entry in TMP.iterdir():
        if entry.name != '.complete':
            entry.rename(DATA / entry.name)
    shutil.rmtree(TMP)
    MARK.touch()
```

数据大时在启动 Web 服务后用 `threading.Thread(target=seed_if_empty, daemon=True).start()` 放到后台；业务接口在 `MARK` 出现之前返回“数据初始化中”。

### 6.2 远程数据库

读写远程数据库（如 PostgreSQL、MySQL）时，平台不管理，也不需要专门声明。

- 连接地址和账号写进 `runtime_config`，账号密码用 `secret`，预览和正式分别填写。驱动写进依赖。
- 不能用保留名 `DATABASE_URL`，另取名称。
- 预览填测试库，正式填正式库。平台不检查，填同一个库时，预览里的写入和迁移会直接改正式数据。
- 远程库的白名单要放行执行服务器的出口 IP，部署前实测连通。
- 远程库的迁移和备份由应用和用户自己负责，平台没有快照。

## 7. 定时任务

先盘点原系统的定时任务（含原机器的 `crontab -l`，它通常不在仓库里），逐项问用户是否继续运行，不要自行排除。结果写进 DEPLOYMENT.md 的“定时任务”一节，没有就写“无”。

容器里没有 cron，也不能在 `start.sh` 里后台起进程。在应用进程里调度，原脚本用子进程运行。Python 用 APScheduler：

```python
def run_job(name, cmd):
    code = subprocess.run([sys.executable, *cmd.split()], cwd="/app").returncode
    print(f"JOB {name} ok" if code == 0 else f"JOB {name} failed: 退出码 {code}", flush=True)

scheduler = BackgroundScheduler(job_defaults={"coalesce": True, "max_instances": 1})
scheduler.add_job(run_job, CronTrigger.from_crontab("30 16 * * 1-5"), args=("daily_sync", "-m myapp.daily_sync"))
scheduler.add_job(run_job, args=("daily_sync", "-m myapp.daily_sync"), next_run_time=datetime.now())  # 启动后先跑一次
scheduler.start()
```

Node.js 用 croner 等进程内调度库，做法相同：同一任务不重叠执行，启动后先跑一次，打印同样的 `JOB` 结果行。

- 启动后每个任务先跑一次，预览里也跑，问题在预览日志里暴露；每次打印 `JOB <任务名> ok` 或 `JOB <任务名> failed: <原因>`。
- 路径都从 `APP_DATA_DIR` 拼，不要回退到项目目录；日志只输出到 stdout/stderr。
- webhook 等外部副作用在 `runtime_config` 里按环境配置，预览留空。
- 只开 1 个 worker；任务可安全重复执行；SQLite 写入分批提交，不长时间锁库。

## 8. 安装、构建与测试

- 依赖用锁文件固定版本：Python 用带版本（最好带哈希）的 `requirements.txt` 或等价锁文件；前端用 `package-lock.json`。
- **安装命令里不要写死软件源**，例如 `pip install -i …`、`--index-url`、`npm --registry …`。平台会测速选出最快的国内源并注入，写死会盖掉它。
- `test` 写你本地能跑通的自测命令，没有可以留空；测试不连真实平台。
- test 阶段没有任何业务环境变量（`runtime_config` 的值只在预览和正式实例里注入），请在不设这些变量的环境里确认测试能跑通：导入模块时不要初始化需要密钥的客户端（例如 `ts.pro_api(token=os.environ.get(...))`），改为用到时再初始化；需要外部服务的测试用模拟接口，或缺少配置时跳过；依赖某变量取值的测试在测试里自己设定。因缺配置而失败的测试，部署时会被排除并在回报里列出，相关功能留到预览验证。

## 9. DEPLOYMENT.md

包根目录的 DEPLOYMENT.md 必须含**唯一一个** deployment 代码块（UTF-8，整份不超过 1 MiB；每阶段最多 30 条命令）。平台按 install → build → initialize → test → cleanup 的顺序在构建容器里执行。命令必须是你测试过的真实命令；不需要的阶段写空数组。不要加 approved、身份、凭据或执行器配置。

````markdown
```deployment
{
  "build_environment": {"python": "^3.11", "node": "22"},
  "steps": {
    "install": ["pip install -r requirements.txt", "npm --prefix frontend ci"],
    "build": ["npm --prefix frontend run build"],
    "initialize": [],
    "test": ["python -m pytest -q"],
    "cleanup": []
  },
  "schema": {"database": "db/app.sqlite3"},
  "resources": {"memory": "1g", "cpus": 1},
  "runtime_config": {"version": 1, "fields": [
    {"name": "LUCIMIX_BASE_URL", "label": "Lucimix 接口地址", "type": "url", "required": true,
     "default": "https://test.lucimix.cn", "description": "外部应用接入指南里的接口基地址；正式环境填 https://lucimix.cn"},
    {"name": "LUCIMIX_APP_KEY", "label": "Lucimix app_key", "type": "text", "required": true,
     "description": "Lucimix 对接人分配的应用标识"},
    {"name": "LUCIMIX_APP_TOKEN", "label": "Lucimix 应用凭证", "type": "secret", "required": true,
     "description": "X-App-Token，只在后端使用"},
    {"name": "LUCIMIX_ORIGINS", "label": "Lucimix 平台 origin", "type": "text", "required": true,
     "default": "https://test.lucimix.cn", "description": "全部平台 origin，空格分隔；正式填 https://lucimix.cn https://www.lucimix.cn https://m.lucimix.cn"},
    {"name": "BILLING_MODE", "label": "扣费方式", "type": "enum", "required": false, "default": "lucimix",
     "choices": ["lucimix", "ledger"], "description": "lucimix 调平台真实扣费；ledger 只记本地流水，出问题时用来停掉扣费"}
  ]}
}
```
````

- `schema.database` 是应用主库在 `APP_DATA_DIR` 下的相对路径，可省略。
- `resources` 是预览和正式容器的资源，可省略，默认内存 512m、CPU 1 核；`memory` 写 `768m`、`2g` 这样的形式，最多 `4g`，`cpus` 写数字，最多 2。按本地实测的峰值留些余量填写，例如定时任务会把大表读进 pandas 时；超出内存时容器会被杀掉，预览报“内存超限被杀”。
- 纯后端、无前端构建时 `build_environment` 只写 `{"python": "^3.11"}`；这个字段可以省略。
- 要导入历史数据时，下载链接在 `runtime_config` 里声明为 `secret`（见 6.1），不写进 DEPLOYMENT.md。
- 代码块之外可以写些说明，帮助部署方排查问题；真实值一律不写。
- 代码块之外必须有一节“定时任务”：列出任务名、时间表、命令，以及用户确认不运行的脚本；没有就写“无”。

## 10. 打包与交付

- ZIP 根目录直接放 DEPLOYMENT.md、源码、依赖声明和锁文件、`start.sh`、迁移文件，外面不要再套一层目录。
- 排除：`.git/`、依赖缓存、虚拟环境、`node_modules/`、日志、数据库文件、上传文件、`.env`、私钥和任何真实凭证。
- 限制：压缩后不超过 200 MiB，展开不超过 256 MiB，最多 10000 个成员；Stored 或 Deflate 压缩，不加密、不分卷；没有软链接、特殊文件、重复路径或路径穿越。
- 交付 `app.zip`。

### 交付前自检清单

下面几项不满足时，应用在平台上一定跑不起来，交付前确认一下：

- [ ] `start.sh` 是 LF 换行，最后一行 `exec` 启动 Python 或 Node.js 进程（不经 `npm start`、`npx`），只开 1 个 worker，前面没有 `… &` 后台进程。
- [ ] 已盘点原系统定时任务并问过用户；DEPLOYMENT.md 有“定时任务”一节；任务在应用进程内调度，打印 `JOB` 结果行（第 7 节）。
- [ ] 应用只写 `APP_DATA_DIR`；日志只输出到 stdout/stderr；启动时不在 `/app` 下建目录或写文件。
- [ ] 路径前缀和业务配置在运行时读取，没有在构建时写死；用两个不同的 `APP_BASE_PATH` 自测过（第 2 节）。
- [ ] SQLite 没有用 WAL，设置了 `busy_timeout`。
- [ ] 应用用到的每份数据都属于第 6 节的四种来源之一；需要导入历史数据的，应用首次启动时按 6.1 自己导入（空目录才导入、写标记、不卡住 `/health`）。
- [ ] 导入后需要持续更新的数据，应用里有对应的定时同步任务写回 `/data`；不更新的，已和用户确认数据停在导入那天可以接受。
- [ ] 应用连接的所有外部服务（含大模型地址和模型名）都在 `runtime_config` 里声明，没有依赖代码里的默认地址。
- [ ] `/health` 匿名可访问，启动后 120 秒内返回 200。
- [ ] Lucimix 消息监听已先安装；`external-app:ready` 在收到 `ticket` 或 `context` 前按参考实现重发，最后一次发送覆盖到 8 秒加载超时之前；在 Lucimix iframe 内验证 `ready` → `ticket` → `POST /api/apps/session` 完整成功。
- [ ] 直接打开预览入口时提示从 Lucimix 进入；此独立页面不作为 iframe 握手验收。
- [ ] ZIP 里没有 `.env`、数据库文件、日志、`node_modules/`、真实凭证。

## 11. 上传后平台做什么

用户上传并发送“部署这个程序”后，部署 Agent 按 DEPLOYMENT.md 完成预检、构建、测试和预览：

- 哪一步失败，部署 Agent 会按报错和日志告诉你原因；
- 预览就绪后由用户本人查验，确认一次即自动发布正式版；
- Lucimix 里登记的入口由用户手动切换。

收到整改说明时，修好应用问题后重新打包；平台配置或能力问题交平台处理。包变了就是新候选，旧的确认不能沿用。不要声称已经部署，也不要编造预览地址。
