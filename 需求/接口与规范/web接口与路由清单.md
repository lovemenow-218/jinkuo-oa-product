# Web 端接口与路由清单（从前端产物提取）

> 来源：`https://oacs.fzjinkuo.com` 主包 `index-CmUHnjLe.js`（构建号 test-20261012）

> 提取时间：2026-10-12（**第二次修订**：修正了初版正则漏扫 143 条的问题）

> 用途：web 测试用例与 `common/web_api.py` 封装的依据

> 统计：菜单路由 47 条、非菜单路由 4 条、**前端引用接口 256 条路径**


## 1. 菜单路由（47 项）

| 模块/菜单 | 路径 | 权限声明 |
| --- | --- | --- |
| 钉钉审批 | `/flow_all/dingtalk_approval` | - |
| 发起审批 | `/flow_all/dingtalk_start` | - |
| 待我处理 | `/flow_all/dingtalk_wait` | - |
| 我发起的 | `/flow_all/dingtalk_create` | - |
| 抄送我的 | `/flow_all/dingtalk_cc` | - |
| 我已处理 | `/flow_all/dingtalk_done` | - |
| 费控预警 | `/flow_all/dingtalk_budget` | - |
| 全部记录 | `/flow_all/dingtalk_manage` | - |
| 概览 | `/index` | GET@/ |
| 个人空间 | `/personal-space` | - |
| 共享网盘 | `/shared-drive` | - |
| 审批任务 | `/my_flow/index` | - |
| 发起审批 | `/my_flow/flow-start` | - |
| 待我处理 | `/my_flow/flow-appvove` | - |
| 我发起的 | `/my_flow/flow-my` | - |
| 抄送我的 | `/my_flow/flow-cc` | - |
| 我已处理 | `/my_flow/flow-record` | - |
| 费控预警 | `/my_flow/flow-budget` | - |
| 全部审批 | `/flow_all` | - |
| 全部记录 | `/flow_all/index` | - |
| 本地审批 | `/flow_all/scope_local` | - |
| 仓库审批 | `/flow_all/scope_procurement` | - |
| 钉钉审批 | `/flow_all/scope_dingtalk` | - |
| 审批类型 | `/flow_manage/index` | - |
| 类型配置 | `/flow_manage/type_index` | - |
| 钉钉审批配置 | `/flow_manage/dingtalk_config` | - |
| 类型分组 | `/flow_manage/group_index` | - |
| 预算预警表 | `/people_manage/budget` | - |
| 员工档案 | `/people_manage/emp-file` | - |
| 考勤记录 | `/people_manage/attend-record` | - |
| 工资表 | `/people_manage/payroll` | - |
| 组织架构 | `/organize/staff/list` | GET@/ |
| 角色权限 | `/organize/rbac/index` | - |
| 报表 | `/report` | - |
| 利润分析表 | `/report/profi-analysis` | - |
| 核单报表 | `/report/bb-hedan` | - |
| 业绩报表 | `/report/bb-yeji` | - |
| 凭证 | `/credentials` | - |
| 查凭证 | `/credentials/look` | - |
| 作废凭证 | `/credentials/invalid` | - |
| 账簿 | `/zhangbo` | - |
| 明细账 | `/zhangbo/mingxi_zhang` | - |
| 总账 | `/zhangbo/zong_zhang` | - |
| 科目余额表 | `/zhangbo/kemu_yueb` | - |
| 账套报表 | `/zt_report` | - |
| 资产负债表 | `/zt_report/balance_table` | - |
| 利润表 | `/zt_report/profit_table` | - |

## 2. 非菜单路由（详情/编辑页等）

| 名称 | 路径 |
| --- | --- |
| 个人空间表格编辑 | `/personal-space/editor/spreadsheet` |
| 个人空间文档编辑 | `/personal-space/editor/document` |
| 共享网盘表格编辑 | `/shared-drive/editor/spreadsheet` |
| 共享网盘文档编辑 | `/shared-drive/editor/document` |

## 3. 前端引用的接口路径（256 条，按归属分组）


### IM / 认证服务（**另一套后端**，本机 404）（105 条）

- `/api/v1/auth/forget`
- `/api/v1/auth/login`
- `/api/v1/auth/logout`
- `/api/v1/auth/refresh-token`
- `/api/v1/auth/register`
- `/api/v1/common/email-code`
- `/api/v1/common/sms-code`
- `/api/v1/contact/apply/accept`
- `/api/v1/contact/apply/create`
- `/api/v1/contact/apply/decline`
- `/api/v1/contact/apply/records`
- `/api/v1/contact/apply/unread-num`
- `/api/v1/contact/delete`
- `/api/v1/contact/detail`
- `/api/v1/contact/edit-remark`
- `/api/v1/contact/group/list`
- `/api/v1/contact/group/save`
- `/api/v1/contact/list`
- `/api/v1/contact/move-group`
- `/api/v1/contact/search`
- `/api/v1/emoticon/customize/create`
- `/api/v1/emoticon/customize/delete`
- `/api/v1/emoticon/del-collect-emoticon`
- `/api/v1/emoticon/list`
- `/api/v1/emoticon/system/install`
- `/api/v1/emoticon/system/list`
- `/api/v1/group/apply/agree`
- `/api/v1/group/apply/all`
- `/api/v1/group/apply/create`
- `/api/v1/group/apply/decline`
- `/api/v1/group/apply/list`
- `/api/v1/group/apply/unread`
- `/api/v1/group/assign-admin`
- `/api/v1/group/create`
- `/api/v1/group/detail`
- `/api/v1/group/dismiss`
- `/api/v1/group/handover`
- `/api/v1/group/invite`
- `/api/v1/group/list`
- `/api/v1/group/member/invites`
- `/api/v1/group/member/list`
- `/api/v1/group/member/remark`
- `/api/v1/group/member/remove`
- `/api/v1/group/mute`
- `/api/v1/group/no-speak`
- `/api/v1/group/notice/edit`
- `/api/v1/group/notice/list`
- `/api/v1/group/overt`
- `/api/v1/group/overt/list`
- `/api/v1/group/secede`
- `/api/v1/group/setting`
- `/api/v1/note/annex/delete`
- `/api/v1/note/annex/forever/delete`
- `/api/v1/note/annex/recover`
- `/api/v1/note/annex/recover/list`
- `/api/v1/note/annex/upload`
- `/api/v1/note/article/asterisk`
- `/api/v1/note/article/delete`
- `/api/v1/note/article/detail`
- `/api/v1/note/article/editor`
- `/api/v1/note/article/forever/delete`
- `/api/v1/note/article/list`
- `/api/v1/note/article/merge`
- `/api/v1/note/article/move`
- `/api/v1/note/article/recover`
- `/api/v1/note/article/tag`
- `/api/v1/note/article/upload/image`
- `/api/v1/note/class/delete`
- `/api/v1/note/class/editor`
- `/api/v1/note/class/list`
- `/api/v1/note/class/sort`
- `/api/v1/note/tag/delete`
- `/api/v1/note/tag/editor`
- `/api/v1/note/tag/list`
- `/api/v1/organize/department/all`
- `/api/v1/organize/member/check`
- `/api/v1/organize/personnel/all`
- `/api/v1/talk/create`
- `/api/v1/talk/delete`
- `/api/v1/talk/disturb`
- `/api/v1/talk/get-records-context`
- `/api/v1/talk/list`
- `/api/v1/talk/message/collect`
- `/api/v1/talk/message/delete`
- `/api/v1/talk/message/file`
- `/api/v1/talk/message/publish`
- `/api/v1/talk/message/revoke`
- `/api/v1/talk/message/vote`
- `/api/v1/talk/message/vote/handle`
- `/api/v1/talk/records`
- `/api/v1/talk/records/forward`
- `/api/v1/talk/records/history`
- `/api/v1/talk/search-chat-records`
- `/api/v1/talk/topping`
- `/api/v1/talk/unread/clear`
- `/api/v1/upload/avatar`
- `/api/v1/upload/image`
- `/api/v1/upload/multipart`
- `/api/v1/upload/multipart/initiate`
- `/api/v1/users/change/detail`
- `/api/v1/users/change/email`
- `/api/v1/users/change/mobile`
- `/api/v1/users/change/password`
- `/api/v1/users/detail`
- `/api/v1/users/setting`

### `/oa_sys_sys/business/`（103 条）

- `/oa_sys/business/app_push/test`
- `/oa_sys/business/app_version_android`
- `/oa_sys/business/app_version_ios`
- `/oa_sys/business/app_versions`
- `/oa_sys/business/app_versions/package`
- `/oa_sys/business/charts`
- `/oa_sys/business/charts_day`
- `/oa_sys/business/charts_month`
- `/oa_sys/business/check`
- `/oa_sys/business/check_confirm`
- `/oa_sys/business/check_pay`
- `/oa_sys/business/dict_bank`
- `/oa_sys/business/employee_attend`
- `/oa_sys/business/employee_attend_import`
- `/oa_sys/business/employee_attend_month`
- `/oa_sys/business/employee_column_sett`
- `/oa_sys/business/employee_enum`
- `/oa_sys/business/employee_image`
- `/oa_sys/business/employee_join`
- `/oa_sys/business/employee_profile`
- `/oa_sys/business/employee_restore`
- `/oa_sys/business/finance_budget_sett`
- `/oa_sys/business/finance_budget_stat`
- `/oa_sys/business/km_code`
- `/oa_sys/business/km_com`
- `/oa_sys/business/km_map`
- `/oa_sys/business/kpi`
- `/oa_sys/business/kpi_confirm`
- `/oa_sys/business/kpi_employee_order`
- `/oa_sys/business/kpi_employee_vip`
- `/oa_sys/business/kpi_pay`
- `/oa_sys/business/kpi_table`
- `/oa_sys/business/kpi_user`
- `/oa_sys/business/oa_dingtalk_payment_subject_config`
- `/oa_sys/business/oa_dingtalk_pending`
- `/oa_sys/business/oa_dingtalk_process_sync`
- `/oa_sys/business/oa_dingtalk_setting`
- `/oa_sys/business/oa_dingtalk_start`
- `/oa_sys/business/oa_dingtalk_template`
- `/oa_sys/business/oa_dingtalk_template_map`
- `/oa_sys/business/oa_dingtalk_template_sync`
- `/oa_sys/business/oa_dingtalk_user_lookup`
- `/oa_sys/business/oa_dingtalk_user_map`
- `/oa_sys/business/oa_dt`
- `/oa_sys/business/oa_flow_attachment`
- `/oa_sys/business/oa_flow_category`
- `/oa_sys/business/oa_flow_contract`
- `/oa_sys/business/oa_flow_form`
- `/oa_sys/business/oa_flow_form_copy`
- `/oa_sys/business/oa_flow_form_record`
- `/oa_sys/business/oa_flow_form_rel`
- `/oa_sys/business/oa_flow_inbound_document_attachment`
- `/oa_sys/business/oa_flow_log`
- `/oa_sys/business/oa_flow_manage`
- `/oa_sys/business/oa_flow_msg`
- `/oa_sys/business/oa_flow_pay`
- `/oa_sys/business/oa_flow_record`
- `/oa_sys/business/oa_flow_record_open`
- `/oa_sys/business/oa_flow_recordcount`
- `/oa_sys/business/procurement_inventory_business_type`
- `/oa_sys/business/procurement_inventory_category`
- `/oa_sys/business/procurement_inventory_config`
- `/oa_sys/business/procurement_inventory_goods`
- `/oa_sys/business/procurement_inventory_operation_log`
- `/oa_sys/business/procurement_inventory_record`
- `/oa_sys/business/procurement_inventory_record_import`
- `/oa_sys/business/procurement_inventory_supplier`
- `/oa_sys/business/procurement_inventory_warehouse`
- `/oa_sys/business/profit`
- `/oa_sys/business/profit_month`
- `/oa_sys/business/user_account`
- `/oa_sys/business/warehouse_aftersales`
- `/oa_sys/business/warehouse_businesses`
- `/oa_sys/business/warehouse_carrier_claims`
- `/oa_sys/business/warehouse_counterpart_names`
- `/oa_sys/business/warehouse_credit`
- `/oa_sys/business/warehouse_document_action`
- `/oa_sys/business/warehouse_documents`
- `/oa_sys/business/warehouse_export`
- `/oa_sys/business/warehouse_fulfillment`
- `/oa_sys/business/warehouse_fulfillment_roles`
- `/oa_sys/business/warehouse_home`
- `/oa_sys/business/warehouse_payment_decide`
- `/oa_sys/business/warehouse_payment_revoke`
- `/oa_sys/business/warehouse_payment_withdraw`
- `/oa_sys/business/warehouse_payments`
- `/oa_sys/business/warehouse_profile_options`
- `/oa_sys/business/warehouse_profiles`
- `/oa_sys/business/warehouse_quarantine_policy`
- `/oa_sys/business/warehouse_refunds`
- `/oa_sys/business/warehouse_reports`
- `/oa_sys/business/warehouse_review_action`
- `/oa_sys/business/warehouse_review_context`
- `/oa_sys/business/warehouse_review_policy`
- `/oa_sys/business/warehouse_shipment`
- `/oa_sys/business/warehouse_shipment_resolve`
- `/oa_sys/business/warehouse_shipment_reverse`
- `/oa_sys/business/warehouse_shipment_sign`
- `/oa_sys/business/warehouse_stock_inspection`
- `/oa_sys/business/warehouse_supply_orders`
- `/oa_sys/business/warehouse_target_warehouses`
- `/oa_sys/business/warehouse_transport_returns`
- `/oa_sys/business/warehouse_units`

### `/finance_zt/business/`（23 条）

- `/finance_zt/business/bb_lrb_export`
- `/finance_zt/business/bb_lrb_show`
- `/finance_zt/business/bb_sett_kemu`
- `/finance_zt/business/bb_zcfzb_export`
- `/finance_zt/business/bb_zcfzb_show`
- `/finance_zt/business/kmlb_details`
- `/finance_zt/business/kmlb_sett`
- `/finance_zt/business/sett_com`
- `/finance_zt/business/stat_profit`
- `/finance_zt/business/voucher_all`
- `/finance_zt/business/voucher_detail`
- `/finance_zt/business/voucher_export`
- `/finance_zt/business/voucher_km_use`
- `/finance_zt/business/voucher_look`
- `/finance_zt/business/voucher_neighbor`
- `/finance_zt/business/voucher_print`
- `/finance_zt/business/voucher_shenhe`
- `/finance_zt/business/zbo_kmyeb`
- `/finance_zt/business/zbo_kmyeb_export`
- `/finance_zt/business/zbo_mxz`
- `/finance_zt/business/zbo_mxz_export`
- `/finance_zt/business/zbo_zongz`
- `/finance_zt/business/zbo_zongz_export`

### 基础 / 其他（12 条）

- `/base.account_security/common/change_pwd`
- `/base.account_sys/common/key_client`
- `/base.base_load/common/api_auth`
- `/base.base_load/common/api_code`
- `/base.base_load/common/api_flag`
- `/base.base_passport/login_code_data`
- `/base.base_passport/login_code_need`
- `/base.base_sys/common/plat_setting`
- `/base_passport/passport`
- `/base_passport/scan_login/scanQrcode`
- `/opentest/getOrgTree`
- `/opentest/switchUser`

### `/organize_sys_sys/base/`（7 条）

- `/organize_sys/base/change`
- `/organize_sys/base/dept_all`
- `/organize_sys/base/menu_all`
- `/organize_sys/base/rbac_all`
- `/organize_sys/base/rbac_my`
- `/organize_sys/base/role_all`
- `/organize_sys/base/staff_all`

### `/im_sys_sys/common/`（3 条）

- `/im_sys/common/talk_act_topping`
- `/im_sys/common/talk_act_unread_clear`
- `/im_sys/common/talk_all`

### `/z_jinkuo_ysd/business/`（3 条）

- `/z_jinkuo_ysd/business/excel_import`
- `/z_jinkuo_ysd/business/excel_map`
- `/z_jinkuo_ysd/business/excel_salary`
