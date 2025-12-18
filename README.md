# todo-actions

> 基于 Microsoft Graph 的 Microsoft To Do OpenAPI 3.1 规范，可直接导入 GPT Actions / 其他自动化平台使用。

[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.1-6BA539)](openapi.yaml)
[![Microsoft Graph](https://img.shields.io/badge/Microsoft-Graph-00A4EF)](https://learn.microsoft.com/graph/overview)
[![License: MIT](https://img.shields.io/badge/License-MIT-black)](LICENSE)

## 这是什么
本仓库**不包含后端服务代码**，仅提供一份可复用的 `OpenAPI 3.1` 接口定义文件：`openapi.yaml`。
它将 Microsoft To Do 的常用能力（列出清单、列出任务、创建任务、更新任务）以标准化方式描述，便于：

- 在 ChatGPT 的 **Actions** 中一键导入并通过 OAuth 登录后调用
- 在 Postman / Swagger UI / OpenAPI Generator 等工具中生成或调试请求

## 功能清单（当前版本）
OpenAPI 版本：`1.0.2`（见 `openapi.yaml`）

- 列出 To Do 清单：`GET /me/todo/lists`（`operationId: listLists`）
- 列出某清单下的任务：`GET /me/todo/lists/{listId}/tasks`（`operationId: listTasks`）
- 创建任务：`POST /me/todo/lists/{listId}/tasks`（`operationId: createTask`）
- 更新任务（如改标题/状态/到期/提醒/备注）：`PATCH /me/todo/lists/{listId}/tasks/{taskId}`（`operationId: updateTask`）

`listTasks` 额外支持常用 OData 查询参数（可选）：`$filter` / `$orderby` / `$top` / `$select`（见 `openapi.yaml`）。

| operationId | Method | Path | 说明 |
| --- | --- | --- | --- |
| `listLists` | GET | `/me/todo/lists` | 获取当前用户的 To Do 清单 |
| `listTasks` | GET | `/me/todo/lists/{listId}/tasks` | 获取指定清单下的任务列表 |
| `createTask` | POST | `/me/todo/lists/{listId}/tasks` | 在指定清单中创建任务 |
| `updateTask` | PATCH | `/me/todo/lists/{listId}/tasks/{taskId}` | 更新任务字段（含完成状态） |

## 文件说明
- `openapi.yaml`：OpenAPI 3.1 定义（server：`https://graph.microsoft.com/v1.0`）
- `prompt.md`：推荐的 Actions 提示词（偏“待办/提醒助手”行为）
- `PRIVACY_POLICY.md`：隐私政策（面向 Actions/集成场景）
- `LICENSE`：MIT License

## 推荐提示词（Actions Instructions）
如果你在 ChatGPT Actions 里导入了本项目的 OpenAPI，建议把 `prompt.md` 的内容粘贴到该 Action 的 `Instructions`，让模型在创建/更新任务时更稳定（例如：先选清单、尽量避免重复、只在必要时设置提醒）。

## 快速开始：用于 ChatGPT Actions
下面以“在 ChatGPT Actions 中导入并使用”为例（其他平台流程类似：导入 OpenAPI → 配置 OAuth → 测试接口）。

### 1) 创建 Azure 应用（App Registration）
在 Azure Portal 中完成：**Microsoft Entra ID → App registrations → New registration**

建议配置：
- **Supported account types**：根据你的用户范围选择（常见选择为同时支持组织账号与个人微软账号）
- **Redirect URI**：按 Actions 平台给出的回调地址填写（在配置 OAuth 时会提示/生成）
- **Client secret**：创建并保存（仅显示一次）

### 2) 配置 API 权限（Delegated）
为应用添加 Microsoft Graph 的委派权限（Delegated permissions）：
- `User.Read`
- `Tasks.ReadWrite`
- （可选但推荐）`offline_access`：允许刷新 token，减少频繁登录
- `openid`：OpenID Connect 登录

必要时为租户执行 Admin consent（取决于组织策略）。

### 3) 在 Actions 中导入并配置 OAuth
在 Actions 平台中：
1. 导入 `openapi.yaml`
2. 认证方式选择 **OAuth 2.0 Authorization Code**
3. 填写 `Client ID` / `Client Secret`
4. Scope 填写（空格分隔）：`openid offline_access User.Read Tasks.ReadWrite`

本规范内置：
- Authorization URL：`https://login.microsoftonline.com/common/oauth2/v2.0/authorize`
- Token URL：`https://login.microsoftonline.com/common/oauth2/v2.0/token`

### 4) 试运行
优先用 `listLists` 获取 `listId`，再调用 `listTasks` / `createTask` / `updateTask`。

## 请求示例（Graph API）
以下示例适用于“你已拿到 `access_token`，直接调用 Graph API”的场景（也可用于排查权限/数据问题）。

### 列出清单
```bash
curl -sS "https://graph.microsoft.com/v1.0/me/todo/lists" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### 创建任务
```bash
curl -sS -X POST "https://graph.microsoft.com/v1.0/me/todo/lists/$LIST_ID/tasks" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "回邮件给 Alex",
    "importance": "high",
    "dueDateTime": { "dateTime": "2025-12-18T09:00:00", "timeZone": "China Standard Time" },
    "isReminderOn": true,
    "reminderDateTime": { "dateTime": "2025-12-18T08:50:00", "timeZone": "China Standard Time" },
    "body": { "contentType": "text", "content": "附上合同与报价单。" }
  }'
```

### 将任务标记为完成
```bash
curl -sS -X PATCH "https://graph.microsoft.com/v1.0/me/todo/lists/$LIST_ID/tasks/$TASK_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "completed" }'
```

### 列出未完成任务（带筛选/排序/字段裁剪）
```bash
curl -sS "https://graph.microsoft.com/v1.0/me/todo/lists/$LIST_ID/tasks?\
%24filter=status%20ne%20'completed'&%24orderby=dueDateTime%2FdateTime%20asc&%24top=20&%24select=id,title,status,dueDateTime" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

## 数据模型要点
- `DateTimeTimeZone`
  - `dateTime`：**无时区偏移**的本地时间字符串（示例：`2025-12-16T22:00:00`）
  - `timeZone`：通常使用 Windows 时区 ID（示例：`China Standard Time`）
- `ItemBody`
  - `contentType`：`text` 或 `html`
  - `content`：备注内容
- `TaskStatus`：`notStarted` / `inProgress` / `completed` / `waitingOnOthers` / `deferred`
- `Importance`：`low` / `normal` / `high`

## 调试与预览 OpenAPI
- 在线预览：用 Swagger Editor / Stoplight 等工具直接导入 `openapi.yaml`
- 本地调试：拿到 `access_token` 后可直接按上面的 `curl` 示例请求 Graph（便于定位 401/403/字段问题）

## 扩展指南（可选）
如果你想把更多 To Do 能力也纳入 Actions：
- 在 `openapi.yaml` 的 `paths` 下新增你需要的 Graph 端点，并给出清晰的 `operationId`/`summary`
- 复用现有 `securitySchemes.oauth2`（本项目定位为 **Delegated/OAuth 用户授权**，不包含 application permissions 流程）
- 更新 `info.version`（建议遵循语义化版本：新增接口/字段递增次版本，破坏性变更递增主版本）

## 常见问题（FAQ）
### 为什么只有 4 个接口？
这是一个“够用、好维护”的最小集合，覆盖了自动化最常见的场景：找清单、看任务、加任务、改状态/时间/备注。
你可以在 `openapi.yaml` 上继续扩展（例如删除任务、获取单个任务详情、清单管理等）。

### 报 401/403 怎么办？
通常是 OAuth 配置或权限范围问题：
- 确认授权 Scope 包含：`User.Read Tasks.ReadWrite`（以及需要时的 `offline_access openid`）
- 确认 Azure 应用已添加对应的 **Delegated permissions**
- 组织账号场景下确认是否需要 Admin consent

## 隐私
见 `PRIVACY_POLICY.md`。

## 许可证
MIT License，见 `LICENSE`。
