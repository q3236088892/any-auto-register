# ChatGPT 注册逻辑阶段计划（2026-04-05）

## Goal
梳理并稳定 ChatGPT 注册链路，明确执行器差异与 RT/无RT 方案边界，为后续迭代提供可执行任务拆分。

## Phases

- [x] Phase 1: 代码证据收集与链路梳理
  - 输出：`docs/plans/2026-04-05-chatgpt-registration-logic-analysis.md`
- [ ] Phase 2: 输出执行器差异可观测性（日志/文案）
- [ ] Phase 3: 补齐 RT/无RT 组合回归测试矩阵
- [ ] Phase 4: 评估并落地 protocol/headless 语义收敛（可选）
- [ ] Phase 5: README 与设置页操作手册同步

## Decisions

- [DECISION] 本轮先做“证据化分析 + 阶段任务沉淀”，不直接改动注册逻辑。
- [DECISION] 阶段任务优先顺序：可观测性 -> 回归测试 -> 语义收敛 -> 文档。

## Risks

- `protocol` 与 `headless` 在 ChatGPT 上语义接近，可能导致用户认知偏差。
- RT 链路状态机复杂，缺少组合回归时容易引入回归。

## Next Checkpoint
完成 Phase 2 后，回填日志截图与关键字段样例到 `findings.md`。
