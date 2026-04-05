# ChatGPT 注册逻辑分析（执行器 + RT 模式）

> 日期：2026-04-05
> 目标：梳理 ChatGPT 注册链路，明确“无头/有头/协议”与“有 RT/无 RT”差异，并沉淀可续跑阶段任务。

## 一句话结论

当前 ChatGPT 注册链路的核心分叉是 **RT 模式分叉**，而不是执行器分叉：

- **有 RT (`refresh_token`)**：走“注册状态机 + OAuth 接续”双阶段链路，最终产出 `access_token + refresh_token + id_token (+session_token/workspace_id)`。
- **无 RT (`access_token_only`)**：走“注册状态机 + 会话复用取 token”单阶段链路，最终主要产出 `access_token + session_token`。
- **执行器 `protocol` vs `headless`**：在 ChatGPT 当前实现中几乎同路（都优先尝试浏览器获取 Sentinel token，失败再降级 HTTP PoW）。
- **执行器 `headed`**：与前两者的主要差异是显式“有头节奏与指纹特征”（例如额外暂停、`headed` 头部偏好、Sentinel 浏览器以非 headless 启动）。

## 链路总览

```text
前端(注册页/账号页)
  -> POST /tasks/register (platform, executor_type, extra)
    -> api/tasks.py::_run_register
      -> ChatGPTPlatform.register
        -> build_chatgpt_registration_mode_adapter(extra)
          -> RT: RefreshTokenRegistrationEngine.run
          -> 无RT: AccessTokenOnlyRegistrationEngine.run
            -> ChatGPTClient / OAuthClient 状态机推进
              -> Account(extra) 落库
```

## 1. 请求入口与参数注入

- 前端执行器来源：`executor_type`（`protocol/headless/headed`）。
- 前端 RT 开关注入：`chatgpt_registration_mode` + `chatgpt_has_refresh_token_solution`。
- 后端 `api/tasks.py` 将 `executor_type` 与 `extra` 组装进 `RegisterConfig`，传给插件。

关键证据：
- `frontend/src/lib/platformExecutorOptions.ts`
- `frontend/src/lib/chatgptRegistrationRequestAdapter.ts`
- `frontend/src/pages/RegisterTaskPage.tsx`
- `frontend/src/pages/Accounts.tsx`
- `api/tasks.py`

## 2. 模式分叉（有 RT / 无 RT）

### 2.1 分叉点

`platforms/chatgpt/chatgpt_registration_mode_adapter.py`：

- `chatgpt_registration_mode=refresh_token` -> `RefreshTokenRegistrationEngine`
- `chatgpt_registration_mode=access_token_only` -> `AccessTokenOnlyRegistrationEngine`
- 兼容别名：`with_rt/no_rt/without_rt/true/false/1/0` 等。

### 2.2 有 RT（新主链路）

`RefreshTokenRegistrationEngine.run` 的核心：

1. 创建邮箱与随机资料。
2. 调 `ChatGPTClient.register_complete_flow(..., stop_before_about_you_submission=True)`，把注册推进到 `about_you` 前停住。
3. 切 `OAuthClient.login_and_get_tokens(...)`（`prefer_passwordless_login=True`，`allow_phone_verification=False`，`complete_about_you_if_needed=True`）继续完成 OAuth、workspace、token。
4. 产出并填充 `access_token/refresh_token/id_token/session_token/workspace_id/account_id`。

### 2.3 无 RT（旧链路）

`AccessTokenOnlyRegistrationEngine.run` 的核心：

1. 创建邮箱与随机资料。
2. 调 `ChatGPTClient.register_complete_flow(...)` 走完整注册状态机。
3. 调 `chatgpt_client.reuse_session_and_get_tokens()` 复用注册会话，直接从 `/api/auth/session` 取 `access_token` 与 `session_token`。
4. 不走 OAuth token 交换主链，因此默认不产出 `refresh_token`。

## 3. 执行器分叉（协议 / 无头 / 有头）

### 3.1 在 ChatGPT 当前实现中的真实差异

| 维度 | protocol | headless | headed |
| --- | --- | --- | --- |
| 传入 `browser_mode` | `protocol` | `headless` | `headed` |
| Sentinel 浏览器调用参数 | `headless=True` | `headless=True` | `headless=False` |
| `_browser_pause` 行为 | 不触发 | 不触发 | 触发随机停顿 |
| `build_browser_headers(..., headed=...)` | `headed=False` | `headed=False` | `headed=True`（附加 `Priority/DNT/Sec-GPC`） |
| 主要 token 兜底 | HTTP PoW 兜底 | HTTP PoW 兜底 | HTTP PoW 兜底 |

结论：当前 ChatGPT 注册实现里，`protocol` 与 `headless` 代码路径几乎一致；真正明显差异是是否 `headed`。

### 3.2 与通用执行器的关系

`BasePlatform._make_executor` 提供了 `ProtocolExecutor/PlaywrightExecutor`，但 ChatGPT 插件当前主链路是自定义 `ChatGPTClient/OAuthClient`，并未走 `_make_executor` 这套通用执行器抽象。

## 4. 账号落库字段差异（模式相关）

统一由适配器构建 `Account.extra`，包含：

- `chatgpt_registration_mode`
- `chatgpt_has_refresh_token_solution`
- `chatgpt_token_source`
- token 相关字段 (`access_token/refresh_token/id_token/session_token/workspace_id`)

其中 **有 RT 与无 RT 的主要差异** 在 `refresh_token` 可用性及后续依赖 RT 的能力可用性。

## 5. 关键风险与注意事项

1. `protocol` 与 `headless` 目前语义接近，UI 上给了三种选项，但实际差异主要在 `headed`，容易让使用者误解。
2. RT 链路依赖 OAuth 状态机与 workspace 解析，分支复杂度显著高于无 RT。
3. 环境变量 `PLAYWRIGHT_HEADLESS/REGISTER_HEADLESS` 会覆盖部分 headless 判定（Sentinel 浏览器侧）。

## 6. 阶段任务（供后续继续）

### 阶段 A：语义对齐与可观测性（建议先做）

- 目标：把 `protocol/headless/headed` 在 ChatGPT 上的“实际行为差异”输出到 UI 文案和日志。
- 验收：
  - 注册日志明确打印 Sentinel 走的是 browser 还是 HTTP PoW。
  - UI 提示 `protocol` 与 `headless` 当前差异边界。

### 阶段 B：RT/无RT 回归测试矩阵

- 目标：补齐以下组合最小回归：
  - RT x (`protocol/headless/headed`)
  - 无RT x (`protocol/headless/headed`)
- 验收：
  - 每组至少覆盖“成功路径 + 常见失败分支（OTP 超时 / add_phone / workspace 提取失败）”。

### 阶段 C：执行器语义收敛（可选改造）

- 目标：决定是否让 `protocol` 真正“完全不触发 Playwright”，并把 Sentinel 浏览器逻辑仅放在 `headless/headed`。
- 验收：
  - 有明确设计决策文档。
  - 行为改造后测试通过，且失败兜底策略不退化。

### 阶段 D：文档更新与操作手册

- 目标：同步 README / 设置页说明，明确 RT 与执行器选择建议。
- 验收：
  - README 新增“执行器差异矩阵 + RT 选择建议 + 常见故障定位”。

## 证据索引（核心文件）

- `platforms/chatgpt/plugin.py`
- `platforms/chatgpt/chatgpt_registration_mode_adapter.py`
- `platforms/chatgpt/refresh_token_registration_engine.py`
- `platforms/chatgpt/access_token_only_registration_engine.py`
- `platforms/chatgpt/chatgpt_client.py`
- `platforms/chatgpt/oauth_client.py`
- `platforms/chatgpt/sentinel_browser.py`
- `platforms/chatgpt/utils.py`
- `core/browser_runtime.py`
- `frontend/src/lib/chatgptRegistrationRequestAdapter.ts`
- `frontend/src/components/ChatGPTRegistrationModeSwitch.tsx`
- `frontend/src/lib/platformExecutorOptions.ts`
- `api/tasks.py`
- `tests/test_chatgpt_registration_mode_adapter.py`
- `tests/test_chatgpt_register.py`
