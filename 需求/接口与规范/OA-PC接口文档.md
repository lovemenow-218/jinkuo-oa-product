# 当前项目接口文档

本文件记录当前 `jk-oa` 服务实际装配的 HTTP 接口，适用于前端联调、第三方系统接入和发布前路由核对。统计基准为 2026-10-07 的 Hono 路由装配结果：共 441 条路由注册记录；按“HTTP 方法 + 路径”去重后为 **438 个接口操作**、334 个路径。路由由 `src/app.js` 挂载各模块后生成，未包含未注册的服务函数或脚本入口。

## 服务地址和接口口径

- 本地默认地址：`http://localhost:8001`。实际监听地址由 `HOST`、`PORT` 配置决定，默认端口为 `8001`。
- 路径中的 `:id`、`:name`、`:token`、`:sessionId` 等是路径参数占位符；实际请求使用具体值，例如 `/oa_sys/business/app_versions/12`。
- 目录中的同一路径、多个方法代表不同操作；应以“方法 + 路径”定位接口，不能仅以路径判断。
- `GET /openapi.json` 是运行时 OpenAPI 3.1 文档。它只包含模块中显式登记的 OpenAPI 操作，当前主要覆盖健康检查、示例和 OA/仓库发布契约，**不是本文件 438 项接口的完整字段 schema**。
- 除正文标明的流式、下载、上传和回调接口外，写操作一般使用 `Content-Type: application/json`。由兼容层 `readJson` 读取的旧接口也接受 `application/x-www-form-urlencoded`；具体字段由对应路由和服务层决定。

## 模块总览

| 模块 | 操作数 | 主要前缀 | 路由实现 |
| --- | ---: | --- | --- |
| 基础及开放 | 32 | `/base*`、`/api/open/*` | `src/modules/base/` |
| 运行状态 | 3 | `/`、`/health`、`/ready`、`/openapi.json` | `src/modules/health/`、`src/core/openapi/` |
| 示例 | 2 | `/api/example` | `src/modules/example/` |
| 低代码后台 | 5 | `/api/admin/*` | `src/modules/lowcode/` |
| 财务账套 | 95 | `/finance_zt/business/*` | `src/modules/finance/` |
| 经营报表 | 3 | `/z_jinkuo_report/business/*` | `src/modules/jinkuo-report/` |
| 组织 | 23 | `/organize_sys/*` | `src/modules/organize/` |
| 个人空间 | 65 | `/api/biz/personal-space/*` | `src/modules/personal_space/` |
| OA | 196 | `/oa_sys/*` | `src/modules/oa/` |
| 云商店 | 13 | `/z_jinkuo_ysd/business/*` | `src/modules/ysd/` |
| 示例受保护接口 | 1 | `/api/protected/example` | `src/modules/example/` |

## 公共约定

### 身份认证和业务上下文

全局中间件依次读取下列任一令牌，并将可解析的用户信息放入请求上下文：

| 优先级 | 传递方式 | 说明 |
| ---: | --- | --- |
| 1 | `Authorization: Bearer <JWT>` | JWT 模式校验 HS256 签名；`AUTH_MODE=disabled` 时仅解析 payload。 |
| 2 | `syOpenAppToken: <token>` | 兼容开放应用的令牌传递方式。 |
| 3 | `Cookie: <AUTH_COOKIE_NAME>=<token>` | Cookie 名默认是 `jkoa_token`；登录和退出接口会写入或清除该 Cookie。 |

`/api/biz/personal-space/*` 的受保护资源以及 `/api/protected/example` 使用 `requireAuth()`，缺少有效登录态时返回 401。个人空间在登录态之后还会校验用户、角色、目录或分享资源权限；取得 JWT 不代表可操作任意文件。

大多数财务、组织、经营报表、云商店和 OA 兼容接口通过 `createBusinessContext` 取得并校验 `accountId`、`businessId`、`staffId`。生产环境这三个值必须来自令牌声明（兼容 `account_id`、`business_id`、`staff_id` 等名称）；开发和测试环境才允许使用 `x-account-id`、`x-business-id`、`x-staff-id`，或配置的本地 OA 开发上下文。不要把开发请求头作为生产接入方案。

开放接口和公开资源是按单条路由设计的，不能因为路径包含 `open`、`share` 或 `download` 就假定匿名可用。例如公开分享仍受总开关、分享令牌、访问令牌/密码约束；OnlyOffice 文件读取依赖短期编辑会话令牌。

### 请求追踪、跨域和错误

- 可传入 `X-Request-Id`；未传时服务生成 UUID，并在响应 `X-Request-Id` 返回。
- CORS 中间件接受 `OPTIONS` 预检并返回 204，允许 `GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD`；当请求带 `Origin` 时响应回显该 Origin。
- 未匹配路由使用标准错误响应并返回 404。
- 全局异常的标准错误体为：

```json
{
  "ok": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [{ "path": "name", "message": "..." }],
    "requestId": "<x-request-id>"
  }
}
```

### 两类成功/业务错误响应

新框架接口（健康检查、示例、个人空间的大部分 JSON 接口）使用标准包装：

```json
{ "ok": true, "data": {} }
```

使用 `legacyRoute` 的兼容业务接口通常使用旧包装：

```json
{
  "status": 200,
  "success": true,
  "msg": "",
  "data": {}
}
```

兼容列表可额外返回 `page`、`pagesize`、`pagemax`、`total`；这些字段只在对应列表服务提供分页信息时出现。兼容业务异常通常为 `{ "status": <业务状态码>, "success": false, "msg": "...", "data": null }`。个别历史兼容接口即使业务失败也会保持 HTTP 200，因此调用方必须同时判断 `success`、`status` 和 `msg`，不要只判断 HTTP 状态。

文件下载、附件预览、SSE、分片上传和 OnlyOffice 回调不保证上述 JSON 包装：应按接口的 `Content-Type`、`Content-Disposition`、`Range` 或回调协议处理。

### 可直接验证的基础接口

| 方法 | 路径 | 认证 | 说明 |
| --- | --- | --- | --- |
| GET | `/` | 否 | 服务自描述，返回健康、就绪和 OpenAPI 地址。 |
| GET | `/health` | 否 | 进程存活检查，不访问数据库。 |
| GET | `/ready` | 否 | 数据库就绪检查；依赖不可用时 HTTP 503。 |
| GET | `/openapi.json` | 否 | 已登记模块的 OpenAPI 3.1 文档。 |
| POST | `/api/example` | 否 | 唯一具有字段级 JSON schema 的示例：`{ "name": "1–100 字符" }`；成功 HTTP 201，校验失败 HTTP 422。 |
| GET | `/api/protected/example` | 是 | 返回当前解析到的用户上下文，用于认证链路联调。 |

## 特殊传输接口

| 场景 | 路径 | 接入要点 |
| --- | --- | --- |
| OA 实时通知 | `GET /oa_sys/business/oa_dingtalk_notifications` | SSE 流，不使用 legacy JSON 包装；仍需按服务实现满足账号、门店权限。 |
| OA 附件上传 | `POST /oa_sys/business/oa_flow_attachment`、`POST /oa_sys/business/oa_flow_inbound_document_attachment` | 由专用上传读取逻辑处理，不应按普通 JSON 字段假设文件载荷。 |
| OA 附件下载 | `GET /oa_sys/business/oa_flow_attachment_download` | 返回附件字节和 `Content-Disposition`；签名、对象路径和有效期由下载服务验证。 |
| 个人空间公开内容 | `/api/biz/personal-space/shares/:token/*` | 公开分享开关、分享令牌、密码换取的 `accessToken` 均可能是必需条件；`page` 返回 HTML，`archive` 返回 ZIP，`content` 支持 Range。 |
| OnlyOffice 回调和会话文件 | `/api/biz/personal-space/editor/callback/onlyoffice`、`/editor/sessions/:sessionId/file` | 使用编辑会话访问令牌而非普通浏览器 Cookie；回调按 OnlyOffice `{ "error": 0/1 }` 协议。 |
| 个人空间下载/解压 | `/entries/download`、`/entries/extract` | 下载返回 ZIP 流；解压可使用 `keepAlive: true`，调用方须按流式响应处理。 |
| 分片/直传上传 | `/uploads/*` | 先创建上传会话；分片 `PUT` 发送二进制并使用 `x-chunk-sha256`，直传 part 的 URL 与 ETag 提交按对应路径完成。 |

## 字段定义的查阅边界

当前项目绝大多数旧接口只在路由层调用服务函数，未提供可覆盖全部 438 项操作的统一 JSON Schema/OpenAPI components。为避免写入与实际实现不一致的字段，本文件的目录保证方法和路径完整；参数、校验规则、权限条件和返回字段应以以下实际实现为准：

| 范围 | 优先查阅位置 |
| --- | --- |
| 基础与开放接口 | `src/modules/base/base.routes.js`、`src/modules/base/base-compat.routes.js`、对应 `*.service.js` |
| 财务账套 | `src/modules/finance/finance.routes.js`、`finance-report.routes.js`、`voucher-all-mutation.routes.js` |
| 组织与经营报表 | `src/modules/organize/organize.routes.js`、`src/modules/jinkuo-report/jinkuo-report.routes.js` |
| 个人空间 | `src/modules/personal_space/routes.js`、`routes_v2.js`、`permission_routes.js` 与被调用服务 |
| OA 业务 | `src/modules/oa/*/*.routes.js`、`src/modules/oa/oa.routes.js` |
| 云商店 | `src/modules/ysd/ysd.routes.js` |
| 通用认证、响应和错误 | `src/core/auth/`、`src/core/http/`、`src/core/errors/`、`src/modules/oa/shared/` |

## 完整接口目录

以下清单按运行时模块分组。目录保留 `staff_action` 的两个尾随空格兼容路径：`/organize_sys/base/staff_action ` 与 URL 编码形式 `/organize_sys/base/staff_action%20`；新调用方应使用无尾随空格的标准路径。

### 基础及开放（32 项操作 / 29 个路径）

| 方法 | 路径 |
| --- | --- |
| GET | `/` |
| POST | `/api/open/auth` |
| GET | `/api/open/authcodes` |
| GET | `/api/open/columns/:name` |
| POST | `/api/open/entry` |
| POST | `/api/open/join` |
| GET | `/api/open/last/:name` |
| POST | `/api/open/open` |
| DELETE / POST | `/api/open/token` |
| DELETE / POST | `/api/open/user` |
| GET | `/api/open/version/:name` |
| POST | `/auth` |
| GET | `/authcodes` |
| POST | `/base_passport/logout` |
| POST | `/base_passport/passport` |
| POST | `/base_passport/sms` |
| GET | `/base_sys/common/plat_info` |
| PUT | `/base.account_security/common/change_pwd` |
| GET | `/base.account_sys/common/key_client` |
| GET | `/base.base_load/common/api_auth` |
| GET | `/base.base_load/common/api_code` |
| GET | `/base.base_load/common/api_flag` |
| GET | `/base.base_passport/login_code_data` |
| GET | `/base.base_passport/login_code_need` |
| GET | `/base.base_sys/common/plat_setting` |
| POST | `/oa/open/entry` |
| POST | `/open` |
| DELETE / POST | `/user` |
| GET | `/version/:name` |

### 运行状态（3 项操作 / 3 个路径）

| 方法 | 路径 |
| --- | --- |
| GET | `/health` |
| GET | `/openapi.json` |
| GET | `/ready` |

### 示例（2 项操作 / 1 个路径）

| 方法 | 路径 |
| --- | --- |
| GET / POST | `/api/example` |

### 低代码后台（5 项操作 / 5 个路径）

| 方法 | 路径 |
| --- | --- |
| GET | `/api/admin/menus` |
| GET | `/api/admin/profile` |
| GET | `/api/admin/system_dict` |
| GET | `/api/admin/system_dict/:name` |
| GET | `/api/admin/tables` |

### 财务账套（95 项操作 / 65 个路径）

| 方法 | 路径 |
| --- | --- |
| POST | `/finance_zt/business/bb_lrb_export` |
| GET | `/finance_zt/business/bb_lrb_show` |
| POST | `/finance_zt/business/bb_rlrb_export` |
| GET | `/finance_zt/business/bb_rlrb_show` |
| POST | `/finance_zt/business/bb_sett_init` |
| GET / POST | `/finance_zt/business/bb_sett_kemu` |
| POST | `/finance_zt/business/bb_zcfzb_export` |
| GET | `/finance_zt/business/bb_zcfzb_show` |
| GET / POST | `/finance_zt/business/daybook_all` |
| DELETE / GET / PUT | `/finance_zt/business/daybook_all/:id` |
| GET | `/finance_zt/business/daybook_detail` |
| GET | `/finance_zt/business/daybook_detail/:id` |
| POST | `/finance_zt/business/daybook_to_voucher` |
| GET | `/finance_zt/business/dict_bank` |
| POST | `/finance_zt/business/dingtalk` |
| GET | `/finance_zt/business/dingtalk_exec` |
| GET / POST | `/finance_zt/business/dingtalk_setting` |
| GET | `/finance_zt/business/kmlb_balance` |
| GET | `/finance_zt/business/kmlb_details` |
| POST | `/finance_zt/business/kmlb_init` |
| GET / POST | `/finance_zt/business/kmlb_qichu` |
| GET / POST | `/finance_zt/business/kmlb_sett` |
| GET | `/finance_zt/business/kmlb_sett_account` |
| DELETE / GET / PUT | `/finance_zt/business/kmlb_sett/:id` |
| GET / POST | `/finance_zt/business/sett_check` |
| GET / POST | `/finance_zt/business/sett_com` |
| DELETE / GET / PUT | `/finance_zt/business/sett_com/:id` |
| GET / POST | `/finance_zt/business/sett_currency` |
| DELETE / GET / PUT | `/finance_zt/business/sett_currency/:id` |
| GET / POST | `/finance_zt/business/sett_pzzi` |
| DELETE / GET / PUT | `/finance_zt/business/sett_pzzi/:id` |
| GET | `/finance_zt/business/sett_zhidanren` |
| GET / POST | `/finance_zt/business/sett_zhlx` |
| DELETE / PUT | `/finance_zt/business/sett_zhlx_km/:id` |
| DELETE / GET / PUT | `/finance_zt/business/sett_zhlx/:id` |
| GET | `/finance_zt/business/stat_cash` |
| GET | `/finance_zt/business/stat_cost_diff` |
| GET | `/finance_zt/business/stat_fee_diff` |
| GET | `/finance_zt/business/stat_income_diff` |
| GET | `/finance_zt/business/stat_performance` |
| GET | `/finance_zt/business/stat_profit` |
| GET | `/finance_zt/business/stat_profit_month` |
| GET | `/finance_zt/business/stat_profit_year` |
| GET / POST | `/finance_zt/business/voucher_all` |
| DELETE / GET / PUT | `/finance_zt/business/voucher_all/:id` |
| GET | `/finance_zt/business/voucher_detail` |
| GET | `/finance_zt/business/voucher_detail/:id` |
| POST | `/finance_zt/business/voucher_export` |
| GET | `/finance_zt/business/voucher_km_use` |
| GET / POST | `/finance_zt/business/voucher_look` |
| GET | `/finance_zt/business/voucher_neighbor` |
| GET | `/finance_zt/business/voucher_num` |
| POST | `/finance_zt/business/voucher_print` |
| POST | `/finance_zt/business/voucher_replace` |
| PUT | `/finance_zt/business/voucher_shenhe/:id` |
| PUT | `/finance_zt/business/voucher_shenhe2/:id` |
| GET / POST | `/finance_zt/business/voucher_template` |
| DELETE / GET / PUT | `/finance_zt/business/voucher_template/:id` |
| GET | `/finance_zt/business/yybb` |
| GET | `/finance_zt/business/zbo_kmyeb` |
| POST | `/finance_zt/business/zbo_kmyeb_export` |
| GET | `/finance_zt/business/zbo_mxz` |
| POST | `/finance_zt/business/zbo_mxz_export` |
| GET | `/finance_zt/business/zbo_zongz` |
| POST | `/finance_zt/business/zbo_zongz_export` |

### 经营报表（3 项操作 / 3 个路径）

| 方法 | 路径 |
| --- | --- |
| GET | `/z_jinkuo_report/business/hedan_diff` |
| GET | `/z_jinkuo_report/business/maoli_cf_fenxi` |
| GET | `/z_jinkuo_report/business/shouru_fenlei_fenxi` |

### 组织（23 项操作 / 13 个路径）

| 方法 | 路径 |
| --- | --- |
| GET / POST | `/organize_sys/base/change` |
| GET | `/organize_sys/base/dept_all` |
| GET | `/organize_sys/base/menu_all` |
| DELETE / GET / POST | `/organize_sys/base/rbac_all` |
| GET | `/organize_sys/base/rbac_my` |
| GET | `/organize_sys/base/role_all` |
| PATCH / POST | `/organize_sys/base/staff_action` |
| PATCH / POST | `/organize_sys/base/staff_action ` |
| PATCH / POST | `/organize_sys/base/staff_action%20` |
| DELETE / GET / POST | `/organize_sys/base/staff_all` |
| DELETE / GET / PUT | `/organize_sys/base/staff_all/:id` |
| GET | `/organize_sys/saas/business_group` |
| GET | `/organize_sys/saas/business_group/:id` |

### 个人空间（65 项操作 / 63 个路径）

| 方法 | 路径 |
| --- | --- |
| GET | `/api/biz/personal-space/audit` |
| POST | `/api/biz/personal-space/editor/callback/onlyoffice` |
| GET | `/api/biz/personal-space/editor/open` |
| POST | `/api/biz/personal-space/editor/save` |
| POST | `/api/biz/personal-space/editor/session` |
| DELETE | `/api/biz/personal-space/editor/sessions/:sessionId` |
| GET | `/api/biz/personal-space/editor/sessions/:sessionId/file` |
| POST | `/api/biz/personal-space/editor/sessions/:sessionId/heartbeat` |
| POST | `/api/biz/personal-space/editor/spreadsheet-jobs` |
| GET | `/api/biz/personal-space/editor/spreadsheet-jobs/:jobId` |
| POST | `/api/biz/personal-space/editor/spreadsheet-sessions` |
| GET | `/api/biz/personal-space/editor/spreadsheet-sessions/:sessionId` |
| POST | `/api/biz/personal-space/editor/spreadsheet-sessions/:sessionId/changes` |
| POST | `/api/biz/personal-space/editor/spreadsheet-sessions/:sessionId/commit` |
| GET | `/api/biz/personal-space/entries` |
| POST | `/api/biz/personal-space/entries/copy` |
| POST | `/api/biz/personal-space/entries/delete` |
| POST | `/api/biz/personal-space/entries/download` |
| POST | `/api/biz/personal-space/entries/extract` |
| POST | `/api/biz/personal-space/entries/move` |
| POST | `/api/biz/personal-space/entries/order` |
| GET / POST | `/api/biz/personal-space/entries/permissions` |
| POST | `/api/biz/personal-space/entries/permissions/batch` |
| POST | `/api/biz/personal-space/entries/permissions/batch-preview` |
| GET | `/api/biz/personal-space/entries/permissions/effective` |
| GET | `/api/biz/personal-space/entries/permissions/operations/:operationId` |
| POST | `/api/biz/personal-space/entries/rename` |
| POST | `/api/biz/personal-space/entries/share` |
| POST | `/api/biz/personal-space/entries/share/revoke` |
| POST | `/api/biz/personal-space/entries/tags` |
| GET | `/api/biz/personal-space/files/content` |
| GET | `/api/biz/personal-space/files/download-url` |
| POST | `/api/biz/personal-space/files/upload` |
| POST | `/api/biz/personal-space/folders` |
| POST | `/api/biz/personal-space/folders/batch` |
| POST | `/api/biz/personal-space/navigation-token` |
| GET | `/api/biz/personal-space/quota` |
| GET | `/api/biz/personal-space/search` |
| GET | `/api/biz/personal-space/shared-entries` |
| GET | `/api/biz/personal-space/shared-entries/directory` |
| GET | `/api/biz/personal-space/shared-files/content` |
| GET | `/api/biz/personal-space/shares` |
| GET | `/api/biz/personal-space/shares/:token` |
| POST | `/api/biz/personal-space/shares/:token/access` |
| GET | `/api/biz/personal-space/shares/:token/archive` |
| GET | `/api/biz/personal-space/shares/:token/content` |
| GET | `/api/biz/personal-space/shares/:token/entries` |
| GET | `/api/biz/personal-space/shares/:token/page` |
| GET | `/api/biz/personal-space/sync/changes` |
| GET | `/api/biz/personal-space/sync/manifest` |
| GET | `/api/biz/personal-space/trash` |
| DELETE | `/api/biz/personal-space/trash/:id` |
| POST | `/api/biz/personal-space/trash/:id/restore` |
| POST | `/api/biz/personal-space/trash/purge` |
| POST | `/api/biz/personal-space/uploads` |
| DELETE / GET | `/api/biz/personal-space/uploads/:id` |
| PUT | `/api/biz/personal-space/uploads/:id/chunks/:index` |
| POST | `/api/biz/personal-space/uploads/:id/complete` |
| PUT | `/api/biz/personal-space/uploads/:id/parts/:partNumber` |
| GET | `/api/biz/personal-space/uploads/:id/parts/:partNumber/url` |
| GET | `/api/biz/personal-space/users` |
| GET | `/api/biz/personal-space/versions` |
| POST | `/api/biz/personal-space/versions/:id/restore` |

### OA（196 项操作 / 143 个路径）

| 方法 | 路径 |
| --- | --- |
| POST | `/oa_sys/business/app_bind` |
| POST | `/oa_sys/business/app_unbind` |
| GET | `/oa_sys/business/app_version_android` |
| GET | `/oa_sys/business/app_version_ios` |
| GET / POST | `/oa_sys/business/app_versions` |
| DELETE / GET / PUT | `/oa_sys/business/app_versions/:id` |
| POST | `/oa_sys/business/app_versions/:id/publish` |
| POST | `/oa_sys/business/app_versions/:id/unpublish` |
| POST | `/oa_sys/business/app_versions/package` |
| GET / POST | `/oa_sys/business/business_config` |
| GET / POST | `/oa_sys/business/cf_confirm` |
| DELETE | `/oa_sys/business/cf_confirm/:id` |
| POST | `/oa_sys/business/cf_date` |
| GET | `/oa_sys/business/charts` |
| GET | `/oa_sys/business/charts_day` |
| GET | `/oa_sys/business/charts_month` |
| GET | `/oa_sys/business/check` |
| GET / POST | `/oa_sys/business/check_confirm` |
| GET / POST | `/oa_sys/business/check_pay` |
| GET | `/oa_sys/business/check_qsfx/:id` |
| GET | `/oa_sys/business/check/:id` |
| GET | `/oa_sys/business/dict_bank` |
| GET / POST | `/oa_sys/business/employee_attend` |
| POST | `/oa_sys/business/employee_attend_import` |
| GET / POST | `/oa_sys/business/employee_attend_month` |
| POST | `/oa_sys/business/employee_attendcheck` |
| GET / POST | `/oa_sys/business/employee_column` |
| GET / POST | `/oa_sys/business/employee_column_sett` |
| GET | `/oa_sys/business/employee_enum` |
| POST | `/oa_sys/business/employee_image` |
| POST | `/oa_sys/business/employee_join` |
| GET | `/oa_sys/business/employee_kpi` |
| GET / POST | `/oa_sys/business/employee_profile` |
| GET / POST | `/oa_sys/business/employee_profile_my` |
| DELETE / GET / PUT | `/oa_sys/business/employee_profile/:id` |
| POST | `/oa_sys/business/employee_profile/check` |
| GET | `/oa_sys/business/employee_profile/columns` |
| GET | `/oa_sys/business/employee_profile/columns/:name` |
| POST | `/oa_sys/business/employee_profile/delete` |
| POST | `/oa_sys/business/employee_profile/entry` |
| GET / PUT | `/oa_sys/business/employee_profile/profile` |
| POST | `/oa_sys/business/employee_restore` |
| GET / POST | `/oa_sys/business/finance_budget_sett` |
| GET | `/oa_sys/business/finance_budget_stat` |
| GET / POST | `/oa_sys/business/km_code` |
| GET | `/oa_sys/business/km_com` |
| GET | `/oa_sys/business/km_map` |
| GET | `/oa_sys/business/kpi` |
| GET / POST | `/oa_sys/business/kpi_confirm` |
| GET | `/oa_sys/business/kpi_dept` |
| GET | `/oa_sys/business/kpi_dept/old` |
| GET | `/oa_sys/business/kpi_employee_order` |
| GET | `/oa_sys/business/kpi_employee_vip` |
| GET | `/oa_sys/business/kpi_month` |
| GET | `/oa_sys/business/kpi_pay` |
| GET | `/oa_sys/business/kpi_table` |
| GET | `/oa_sys/business/kpi_table/employeeorder` |
| GET | `/oa_sys/business/kpi_table/employeevip` |
| GET | `/oa_sys/business/kpi_table/pays` |
| GET | `/oa_sys/business/kpi_user` |
| GET | `/oa_sys/business/kpi/:id` |
| GET | `/oa_sys/business/oa_dingtalk_notifications` |
| POST | `/oa_sys/business/oa_dingtalk_payment_subject_config` |
| GET / POST | `/oa_sys/business/oa_dingtalk_pending` |
| POST | `/oa_sys/business/oa_dingtalk_process_sync` |
| GET / POST | `/oa_sys/business/oa_dingtalk_setting` |
| GET / POST | `/oa_sys/business/oa_dingtalk_start` |
| GET | `/oa_sys/business/oa_dingtalk_template` |
| GET / POST | `/oa_sys/business/oa_dingtalk_template_map` |
| POST | `/oa_sys/business/oa_dingtalk_template_sync` |
| GET | `/oa_sys/business/oa_dingtalk_user_lookup` |
| GET / POST | `/oa_sys/business/oa_dingtalk_user_map` |
| GET | `/oa_sys/business/oa_dt` |
| GET / POST | `/oa_sys/business/oa_flow_attachment` |
| GET | `/oa_sys/business/oa_flow_attachment_download` |
| GET / POST | `/oa_sys/business/oa_flow_category` |
| DELETE / GET / PUT | `/oa_sys/business/oa_flow_category/:id` |
| GET | `/oa_sys/business/oa_flow_contract` |
| GET / POST | `/oa_sys/business/oa_flow_form` |
| POST | `/oa_sys/business/oa_flow_form_copy` |
| POST | `/oa_sys/business/oa_flow_form_record` |
| GET | `/oa_sys/business/oa_flow_form_rel/:id` |
| DELETE / GET / PUT | `/oa_sys/business/oa_flow_form/:id` |
| POST | `/oa_sys/business/oa_flow_inbound_document_attachment` |
| POST | `/oa_sys/business/oa_flow_log` |
| GET | `/oa_sys/business/oa_flow_manage` |
| GET | `/oa_sys/business/oa_flow_msg` |
| PUT | `/oa_sys/business/oa_flow_pay/:id` |
| GET / POST | `/oa_sys/business/oa_flow_record` |
| GET | `/oa_sys/business/oa_flow_record_open` |
| GET / PUT | `/oa_sys/business/oa_flow_record/:id` |
| GET | `/oa_sys/business/oa_flow_recordcount` |
| GET | `/oa_sys/business/org` |
| GET | `/oa_sys/business/org_tree` |
| GET | `/oa_sys/business/org/:id` |
| GET | `/oa_sys/business/org/tree` |
| GET / POST | `/oa_sys/business/procurement_inventory_business_type` |
| GET / POST | `/oa_sys/business/procurement_inventory_category` |
| GET / POST | `/oa_sys/business/procurement_inventory_config` |
| DELETE / GET / POST | `/oa_sys/business/procurement_inventory_goods` |
| GET | `/oa_sys/business/procurement_inventory_operation_log` |
| GET | `/oa_sys/business/procurement_inventory_record` |
| POST | `/oa_sys/business/procurement_inventory_record_import` |
| GET / POST | `/oa_sys/business/procurement_inventory_supplier` |
| GET / POST | `/oa_sys/business/procurement_inventory_warehouse` |
| GET | `/oa_sys/business/profit` |
| GET | `/oa_sys/business/profit_month` |
| GET | `/oa_sys/business/system_dict` |
| GET / POST | `/oa_sys/business/user_account` |
| DELETE / GET / PUT | `/oa_sys/business/user_account/:id` |
| GET / POST | `/oa_sys/business/warehouse_aftersales` |
| GET | `/oa_sys/business/warehouse_businesses` |
| GET / POST | `/oa_sys/business/warehouse_carrier_claims` |
| GET | `/oa_sys/business/warehouse_counterpart_names` |
| GET / POST | `/oa_sys/business/warehouse_credit` |
| POST | `/oa_sys/business/warehouse_document_action` |
| GET / POST | `/oa_sys/business/warehouse_documents` |
| POST | `/oa_sys/business/warehouse_export` |
| GET | `/oa_sys/business/warehouse_fulfillment` |
| GET / POST | `/oa_sys/business/warehouse_fulfillment_roles` |
| POST | `/oa_sys/business/warehouse_payment_decide` |
| POST | `/oa_sys/business/warehouse_payment_revoke` |
| POST | `/oa_sys/business/warehouse_payment_withdraw` |
| GET / POST | `/oa_sys/business/warehouse_payments` |
| GET / POST | `/oa_sys/business/warehouse_quarantine_policy` |
| GET / POST | `/oa_sys/business/warehouse_refunds` |
| GET | `/oa_sys/business/warehouse_reports` |
| POST | `/oa_sys/business/warehouse_review_action` |
| GET | `/oa_sys/business/warehouse_review_context` |
| POST | `/oa_sys/business/warehouse_review_policy` |
| POST | `/oa_sys/business/warehouse_shipment` |
| POST | `/oa_sys/business/warehouse_shipment_resolve` |
| POST | `/oa_sys/business/warehouse_shipment_reverse` |
| POST | `/oa_sys/business/warehouse_shipment_sign` |
| GET | `/oa_sys/business/warehouse_stock_inspection` |
| GET | `/oa_sys/business/warehouse_supply_orders` |
| GET | `/oa_sys/business/warehouse_target_warehouses` |
| GET / POST | `/oa_sys/business/warehouse_transport_returns` |
| GET / POST | `/oa_sys/business/warehouse_units` |
| GET | `/oa_sys/operator/system_config` |
| GET | `/oa_sys/operator/system_config/:id` |
| GET | `/oa_sys/operator/system_dict` |
| GET | `/oa_sys/operator/system_dict/:name` |

### 云商店（13 项操作 / 8 个路径）

| 方法 | 路径 |
| --- | --- |
| GET / POST | `/z_jinkuo_ysd/business/baobiao_all` |
| DELETE / GET / PUT | `/z_jinkuo_ysd/business/baobiao_all/:id` |
| GET | `/z_jinkuo_ysd/business/baobiao_export` |
| GET | `/z_jinkuo_ysd/business/cf_change` |
| GET | `/z_jinkuo_ysd/business/hrp_employee` |
| POST | `/z_jinkuo_ysd/business/hrp_employee_export` |
| DELETE / GET / PUT | `/z_jinkuo_ysd/business/hrp_employee/:id` |
| GET | `/z_jinkuo_ysd/business/kpi_change` |

### 示例受保护接口（1 项操作 / 1 个路径）

| 方法 | 路径 |
| --- | --- |
| GET | `/api/protected/example` |

## 维护规则

新增、移除或变更 HTTP 方法/路径时，应同步更新本文件的模块计数和接口目录，并以运行时 Hono 路由装配结果复核。若接口补充了稳定字段 schema，应优先同步登记到模块 OpenAPI 定义，使 `/openapi.json` 能提供机器可读的请求与响应契约；在此之前，不应根据前端页面或接口命名猜测字段。
