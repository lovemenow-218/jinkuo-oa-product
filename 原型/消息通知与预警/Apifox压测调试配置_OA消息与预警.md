# Apifox 压测功能调试配置（OA 系统 · 消息中心 + 预警管理）

> 目的：在 Apifox 上**熟悉压测功能**（建场景 → 配参数 → 跑 → 看报表），不是真压测。
> 生成日期：2026-11-18
> 安全前提：**纯读接口 + 个位数并发 + 不建任何数据 + 不碰仓库**。

## 0. 红线（动手前先读）

| 项 | 约束 |
|---|---|
| 压测目标 | OA 系统：消息中心 + 预警管理。**仓库接口（`warehouse_*` / `procurement_inventory_*`）一律不碰** |
| 数据 | **绝不在预警管理创建数据**；只做 GET 查询 |
| 并发量 | 线程数 **2–3**，绝不用 50（此前 50 并发把测试环境上游打崩过） |
| 持续时间 | 30 秒够看报表 |
| 环境 | test：`https://oacs.fzjinkuo.com`（H5 用 `https://h5cs.fzjinkuo.com`）；门店 `business_id=200242` |
| 账号 | 主账号 `15060038842` / 密码 `15060038842` |

⛔ **以下写接口绝对不拖进压测场景**（会改状态/清数据，不可逆）：
- 单条已读 `/read`、批量已读 `/read-all`（缺陷38：会把全站未读清零）
- 单笔确认 `/confirm`、批量确认 `/confirm-bulk`（改还款状态）
- 台账导出 `/export`（会生成文件、占资源）

## 1. 两套场景的设计

web 和 h5 连的是**同一套后端、同一个库**，但**产品功能不同**：**门店消息中心（`/stores` 系列，对应《APP首页改版-消息中心接口文档》）是 h5/APP 独有，web 端还没有这个功能**，web 只有旧版通知列表。这是 2026-11-18 用户（产品 oracle）确认的口径，**已纠正早前把 web/h5 新旧接口写反的错误**。

> 注意：后端接口是共用的——在 web 域名上直接调 `/stores` 也能返回 200，但 web 前端没有这个页面，压它不代表真实 web 流量，所以 web 场景不碰 `/stores`。

| | web 场景 | h5 场景 |
|---|---|---|
| 基础地址 | `https://oacs.fzjinkuo.com` | `https://h5cs.fzjinkuo.com` |
| 消息中心接口 | **旧版** `oa_app_notifications`（无门店消息中心） | **新版门店消息中心** `/stores` 系列 |
| `syOpenAppId` 头 | `jinkuo-pc-business` | 不传（h5 app 三件套为空） |
| `syOpenAppProject` / `Role` | `CaseJinkuoPro` / `org-business` | 不传 |
| `Sy-Client-Platform` | `pc-web` | `ios` |
| 登录 Body | 抓包版 `sdk_sms_code`/`verify_code`（见 §2.2） | 抓包版（**已实测通**） |

## 2. 公共配置

### 2.1 新建环境 `test`，加变量

| 变量名 | 值 |
|---|---|
| `baseUrlWeb` | `https://oacs.fzjinkuo.com` |
| `baseUrlH5` | `https://h5cs.fzjinkuo.com` |
| `tokenWeb` | 先留空，登录接口写入 |
| `tokenH5` | 先留空，登录接口写入 |
| `businessId` | `200242` |

### 2.2 登录接口（各建一个，拿 token）

> 2026-11-18 实测：前端当前登录 Body 是**抓包版**（`sdk_sms_code`/`verify_code`），不是从代码逆向的 `captcha_key` 版。web 与 h5 同走 `/base_passport/passport`，web 已实测抓包版能拿 token。

**web 登录**：`POST {{baseUrlWeb}}/base_passport/passport`
```json
{"account_password":"15060038842","account_mobile":"15060038842","sdk_sms_code":"","verify_code":"","handleType":"MobilePwdLogin"}
```
Header：`syOpenAppProject=CaseJinkuoPro`、`syOpenAppId=jinkuo-pc-business`、`syOpenAppRole=org-business`、`Sy-Client-Platform=pc-web`

**h5 登录**：`POST {{baseUrlH5}}/base_passport/passport`（**已实测通**）
```json
{"account_password":"15060038842","account_mobile":"15060038842","sdk_sms_code":"","verify_code":"","handleType":"MobilePwdLogin"}
```
Header：`Sy-Client-Platform=ios`（**不要**加 `syOpenApp*` 三个头）

两个登录接口都加「后置操作 → 提取变量」：JSONPath `$.data.result_data.token_access`，分别存到 `tokenWeb` / `tokenH5`。
（若为 null，改提 `$.data.result_token`。1506 是 `alone` 单门店，一段登录就出 token。）

## 3. web 场景接口（4 个纯读）

> web **没有门店消息中心**，消息走旧版 `oa_app_notifications`（2026-11-18 用户确认）。这 4 个就是 web 场景的全部读接口，压测够用。

路径前缀 `{{baseUrlWeb}}/oa_sys/business/`：

| # | 接口名 | 方法与路径 | 参数 |
|---|---|---|---|
| 1 | 消息通知主页（列表+未读数） | GET `/oa_app_notifications` | `filter=all`, `limit=20` |
| 2 | 通知详情 | GET `/oa_app_notifications/{{noticeId}}` | 路径参数 |
| 3 | 预警管理列表 | GET `/oa_repayment_warnings` | `page=1`, `limit=20` |
| 4 | 预警管理详情 | GET `/oa_repayment_warnings/{{recordId}}` | 路径参数 |

> `noticeId`、`recordId` 需先手工从接口 1 / 3 的响应里取一个真实值填进环境变量（路径参数是写死的，不会自动变）。详情接口每次读相同一条即可，调试功能够用。

## 4. h5 场景接口（门店消息中心，3 个纯读）

> **门店消息中心是 h5/APP 独有**（对应《APP首页改版-消息中心接口文档》），web 没有。这 3 个 `/stores` 接口在 h5 域名已实测可用（3 店 2 品牌、1506 未读 1393）。

路径前缀 `{{baseUrlH5}}/oa_sys/business/`：

| # | 接口名 | 方法与路径 | 参数 |
|---|---|---|---|
| 1 | 消息首页汇总 | GET `/oa_app_notifications/stores` | 无 |
| 2 | 门店通知列表 | GET `/oa_app_notifications/stores/{{businessId}}/notifications` | `filter=all`, `limit=20` |
| 3 | 单条通知详情 | GET `/oa_app_notifications/stores/{{businessId}}/notifications/{{noticeIdH5}}` | 路径参数 |

> 预警管理（`/oa_repayment_warnings`）是 web 后台功能，h5 域名能否访问**需先探针确认**；本次 h5 场景聚焦门店消息中心 3 个接口即可。

## 5. 每个接口的通用配置

- **Header**（两套场景各自带自己的 token）：
  - web：`Authorization: Bearer {{tokenWeb}}` + `syOpenAppToken: {{tokenWeb}}` + §2.2 的 web 头
  - h5：`Authorization: Bearer {{tokenH5}}` + `syOpenAppToken: {{tokenH5}}` + `Sy-Client-Platform: ios`
- **断言**（关键，别漏）：`JSONPath $.status 等于 200`。这些是 legacy 包装，存在「HTTP 200 但业务失败」，只看 HTTP 码会漏判。
- **思考时间**：场景里加定时器，间隔 1 秒，进一步降低后端压力。

## 6. 压测参数（调试用，务必保守）

> **入口（Apifox 桌面版实测 2026-11-18）**：左侧栏**没有独立的「性能测试」顶级入口**。正确路径：左侧「自动化测试」→ 新建「测试场景」→ 把接口拖进场景 → 右上角「运行」旁的下拉箭头 → 选「性能测试」切到压测模式。备选：接口详情页顶部「测试用例」标签 →「性能测试」（只能压单条）。**性能测试必须基于测试场景**，不能选中接口直接压。

| 参数 | 值 |
|---|---|
| 并发用户数（线程数） | **2** |
| Ramp-Up | **5 秒** |
| 持续时间 | **30 秒** |
| 循环次数 | 按持续时间，不额外限制 |
| 场景内容 | web 场景跑 4 个读接口；h5 场景跑门店消息中心 3 个，单独建、单独跑 |

跑之前**先跟开发打招呼**（共享环境），全程有人盯，随时能停。看到 TPS / 平均响应时间 / p95 / 错误率报表，这套功能就算走通了。

## 7. 三处 Apifox 复刻不全（提前知道）

1. 真实压测脚本的 `@task` 是随机权重 90/10，Apifox 场景是顺序执行，做不到精确配比——把高频接口多排几遍即可。
2. 真实脚本每次随机翻列表，Apifox 只能固定顺序。
3. 中断压测若留下数据要核销——本次全是读接口，不产生数据，无需核销。

## 8. 用 cURL 导入建接口（推荐）

Apifox 支持「导入 cURL」：新建接口 → 选「导入 cURL」→ 粘贴下面的命令 → 自动解析出方法/URL/Header/Body。

**顺序很重要**：① 先导入登录接口并「发送」拿到 token → ② 把 token 存进环境变量 → ③ 再导入业务接口，并把 Header 里的 token 占位符 `{{tokenWeb}}` / `{{tokenH5}}` 改成引用环境变量。

> 下面的域名先写死真实地址便于一次导入成功；导入后可在接口里把域名替换成 `{{baseUrlWeb}}` / `{{baseUrlH5}}`。路径里 `{{businessId}}` 等占位符若导入后不识别，手工填 `200242` 即可。

### 8.1 登录接口（先建、先跑）

**web 登录**：
```bash
curl -X POST 'https://oacs.fzjinkuo.com/base_passport/passport' \
  -H 'Content-Type: application/json' \
  -H 'syOpenAppProject: CaseJinkuoPro' \
  -H 'syOpenAppId: jinkuo-pc-business' \
  -H 'syOpenAppRole: org-business' \
  -H 'Sy-Client-Platform: pc-web' \
  -d '{"account_password":"15060038842","account_mobile":"15060038842","sdk_sms_code":"","verify_code":"","handleType":"MobilePwdLogin"}'
```

**h5 登录（已实测通）**：
```bash
curl -X POST 'https://h5cs.fzjinkuo.com/base_passport/passport' \
  -H 'Content-Type: application/json' \
  -H 'Sy-Client-Platform: ios' \
  -d '{"account_password":"15060038842","account_mobile":"15060038842","sdk_sms_code":"","verify_code":"","handleType":"MobilePwdLogin"}'
```

跑通后取响应里的 `data.result_data.token_access`（或 `data.result_token`）存入对应环境变量。

### 8.2 web 业务接口（4 个纯读，Header 带 `Authorization: Bearer {{tokenWeb}}` + `syOpenAppToken: {{tokenWeb}}`）

**建接口顺序：先导入 1（消息主页）和 3（预警列表）→ 跑一次从响应里拿真实 id → 再导入 2（通知详情）、4（预警详情）并把假 id 换掉**（详情接口按假 id 直接建会 404）。

```bash
# 1. 消息通知主页（列表+未读数）
curl 'https://oacs.fzjinkuo.com/oa_sys/business/oa_app_notifications?filter=all&limit=20' \
  -H 'Authorization: Bearer {{tokenWeb}}' -H 'syOpenAppToken: {{tokenWeb}}' \
  -H 'syOpenAppProject: CaseJinkuoPro' -H 'syOpenAppId: jinkuo-pc-business' \
  -H 'syOpenAppRole: org-business' -H 'Sy-Client-Platform: pc-web'

# 2. 通知详情（把 12388 换成真实通知 id，从接口 1 响应里取）
curl 'https://oacs.fzjinkuo.com/oa_sys/business/oa_app_notifications/12388' \
  -H 'Authorization: Bearer {{tokenWeb}}' -H 'syOpenAppToken: {{tokenWeb}}' \
  -H 'syOpenAppProject: CaseJinkuoPro' -H 'syOpenAppId: jinkuo-pc-business' \
  -H 'syOpenAppRole: org-business' -H 'Sy-Client-Platform: pc-web'

# 3. 预警管理列表
curl 'https://oacs.fzjinkuo.com/oa_sys/business/oa_repayment_warnings?page=1&limit=20' \
  -H 'Authorization: Bearer {{tokenWeb}}' -H 'syOpenAppToken: {{tokenWeb}}' \
  -H 'syOpenAppProject: CaseJinkuoPro' -H 'syOpenAppId: jinkuo-pc-business' \
  -H 'syOpenAppRole: org-business' -H 'Sy-Client-Platform: pc-web'

# 4. 预警管理详情（把 32058 换成真实 record_id，从接口 3 响应里取）
curl 'https://oacs.fzjinkuo.com/oa_sys/business/oa_repayment_warnings/32058' \
  -H 'Authorization: Bearer {{tokenWeb}}' -H 'syOpenAppToken: {{tokenWeb}}' \
  -H 'syOpenAppProject: CaseJinkuoPro' -H 'syOpenAppId: jinkuo-pc-business' \
  -H 'syOpenAppRole: org-business' -H 'Sy-Client-Platform: pc-web'
```

### 8.3 h5 业务接口（门店消息中心 3 个纯读，Header 带 `Authorization: Bearer {{tokenH5}}` + `Sy-Client-Platform: ios`，**不要** `syOpenApp*`）

```bash
# 1. 消息首页汇总
curl 'https://h5cs.fzjinkuo.com/oa_sys/business/oa_app_notifications/stores' \
  -H 'Authorization: Bearer {{tokenH5}}' -H 'syOpenAppToken: {{tokenH5}}' \
  -H 'Sy-Client-Platform: ios'

# 2. 门店通知列表
curl 'https://h5cs.fzjinkuo.com/oa_sys/business/oa_app_notifications/stores/200242/notifications?filter=all&limit=20' \
  -H 'Authorization: Bearer {{tokenH5}}' -H 'syOpenAppToken: {{tokenH5}}' \
  -H 'Sy-Client-Platform: ios'

# 3. 单条通知详情（把 12388 换成真实通知 id，从接口 2 响应里取）
curl 'https://h5cs.fzjinkuo.com/oa_sys/business/oa_app_notifications/stores/200242/notifications/12388' \
  -H 'Authorization: Bearer {{tokenH5}}' -H 'syOpenAppToken: {{tokenH5}}' \
  -H 'Sy-Client-Platform: ios'
```

### 8.4 导入后的三个检查点

1. **占位符**：`{{tokenWeb}}` / `{{tokenH5}}` 若没被识别成变量，手工把 Header 值改成引用环境变量。
2. **追加断言**：导入只带 Header，不会带断言——每个接口手动补 `JSONPath $.status = 200`。
3. **路径参数**：详情接口的 `12388` / `32058` 是示例值，先跑列表接口从响应里取一个真实的填进去。

## 9. 在 Apifox 里怎么区分 H5 和 Web（三处）

同一个项目里两套场景，靠下面三处区分。调试阶段**以左侧目录为主**，最直观。

| # | 区分位置 | H5 | Web |
|---|---|---|---|
| 1 | **左侧接口目录** | 建 `H5` 文件夹，放 H5 登录 + 3 个门店消息中心接口 | 建 `Web` 文件夹，放 Web 登录 + 4 个旧版消息/预警接口 |
| 2 | **右上角环境** | `H5测试`，变量 `tokenH5` | `Web测试`，变量 `tokenWeb` |
| 3 | **接口 URL + Header** | 域名 `h5cs.fzjinkuo.com`，消息走 `/stores`，Header 只带 `Sy-Client-Platform: ios` | 域名 `oacs.fzjinkuo.com`，消息走旧版 `oa_app_notifications`，Header 带 `Sy-Client-Platform: pc-web` + `syOpenAppProject/Id/Role` 三件套 |

操作要点：
- **目录**：左侧「默认模块」下建 `H5` / `Web` 两个子文件夹，接口分开放，避免把 web 接口拖进 h5 压测场景。
- **环境**：右上角环境选择器 →「管理环境」，建两套环境各存自己的 token 变量；登录成功后写入对应变量，切换环境即切换 token。
- **压测场景**：也建两个（如 `H5压测场景` / `Web压测场景`），分别只拖各自目录里的读接口，别混。

### 登录 Body 的差异（实测提醒）

Web 与 H5 登录都走 `/base_passport/passport`。2026-11-18 用户在 Apifox 里实测 H5 抓包版 Body 通了：
```json
{"account_password":"15060038842","account_mobile":"15060038842","sdk_sms_code":"","verify_code":"","handleType":"MobilePwdLogin"}
```
（多 `sdk_sms_code` / `verify_code`，少 `captcha_key` / `business_id`。）

**web 用同一份抓包版即可**（已实测能拿 token）。web 若不通，再试从 `common/web_api.py` 逆向的备选：
```json
{"handleType":"MobilePwdLogin","account_mobile":"15060038842","account_password":"15060038842","business_id":0,"captcha_code":"","captcha_key":""}
```

## 10. 参考来源（均已实测核对）

- `docs/OA-PC接口文档.md`（OA 接口目录）
- `docs/备用金还款预警通知与预警管理接口文档.md`（旧版通知 + 预警管理，§2 接口总览）
- `APP首页改版-消息中心接口文档.md`（新版 `/stores` 系列，h5/APP 专属）
- `common/web_api.py`（web 登录与 app 头）、`common/h5_api.py`（h5 登录）、`config/config.yaml`（账号与 app 配置）
