# Findings

## 2026-04-05 ChatGPT 注册逻辑

- [REMEMBER] ChatGPT 注册模式由 `chatgpt_registration_mode` 决定，默认 `refresh_token`。
- [REMEMBER] `no_rt/without_rt/false/0` 会归一化为 `access_token_only`。
- [REMEMBER] 有 RT 路径是“注册状态机 -> OAuth 接续 -> token 交换”，无 RT 路径是“注册状态机 -> 会话复用取 access token”。
- [REMEMBER] 在 ChatGPT 当前实现中，`protocol` 和 `headless` 主要代码路径基本一致，`headed` 才有明显行为差异。
- [ARCHITECTURE] 入口链路：Frontend -> `/tasks/register` -> `ChatGPTPlatform.register` -> 模式适配器 -> 注册引擎 -> Account 落库。
- [ARCHITECTURE] ChatGPT 插件走自定义 `ChatGPTClient/OAuthClient` 状态机，不走 `BasePlatform._make_executor` 通用执行器实现。

## Evidence Files

- `platforms/chatgpt/plugin.py`
- `platforms/chatgpt/chatgpt_registration_mode_adapter.py`
- `platforms/chatgpt/refresh_token_registration_engine.py`
- `platforms/chatgpt/access_token_only_registration_engine.py`
- `platforms/chatgpt/chatgpt_client.py`
- `platforms/chatgpt/oauth_client.py`
- `platforms/chatgpt/utils.py`
- `frontend/src/lib/chatgptRegistrationRequestAdapter.ts`
- `frontend/src/components/ChatGPTRegistrationModeSwitch.tsx`
- `api/tasks.py`
- `tests/test_chatgpt_registration_mode_adapter.py`
- `tests/test_chatgpt_register.py`
