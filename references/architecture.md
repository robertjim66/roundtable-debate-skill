# 圆桌对抗思辨 · 应用架构（若要做成独立产品）

> 本 skill 的核心方法论可在"纯对话式"直接使用（见 SKILL.md）。若要把它做成一个可部署的
> 多智能体应用（如 brainstorm_agent 项目本身），下面是经过验证的架构。

## 技术栈

- **编排**：LangGraph（StateGraph + conditional_edges + checkpointer）
- **模型**：ChatOpenAI（OpenAI-compatible，可换 DeepSeek / 通义 / GLM / Ollama）
- **服务端**：FastAPI + SSE（流式推送每个角色发言）
- **前端**：React + Vite（按角色分色、流式渲染、可信度色点）
- **检索**：标准库 `urllib` 实现的 TavilyBackend（真联网）+ MockBackend（测试），工厂模式 `get_search_backend(settings)`

## LangGraph 节点与流转

```
START → reframe → role_gen → moderator ⇄ participant ⇄ goal_judge → fact_audit → synthesis → END
                                       ↑_____________________|
                                                    └→ (收敛) clarify →(interrupt) 等用户 → moderator（聚焦1轮）
```

- **reframe**：钢人化重构决策（见 prompts.md §1）
- **role_gen**：成对对立角色 + sub_goals
- **moderator**：决定下一个发言者、指定 `rebut_target`、推进阶段、判定退出
- **participant**：角色发言，结尾 `POSITION:`
- **goal_judge**：判定子目标覆盖；未覆盖 + 轮次有剩 → 回 moderator
- **clarify**：收敛前 `interrupt()` 暂停 → SSE 推问题 → 用户回答后 `Command(resume=)` 唤醒，注入"用户补充"发言并 `max_rounds +1`，再跑 1 轮
- **fact_audit**：抽取断言 → 逐条 `search.search()` → 5 级标签 + 推理链体检
- **synthesis**：11 段式报告 + 审计章节

## 状态隔离（防回声室）

- 静态角色 `roles` 锁定只读；动态状态（立场演变 / 被反驳次数 / 连续附和）与静态分离
- `transcript` / `round_summaries` 用 `Annotated[..., add]` reducer 隔离追加
- 共识分 = `0.5×关键词Jaccard + 0.5×立场极性对齐`（纯代码算，非 LLM）
- pydantic 对象（Role/Turn）存入 checkpointer 会打印 "unregistered type" 警告，生产环境需注册序列化器

## 关键工程坑（已踩过）

1. **interrupt 在 `astream` 下以特殊键 `__interrupt__` 暴露**（不是节点名）——server 流处理必须特判。
2. **SSE 是单向流，clarify 需有状态 run**：`/api/debate` 返回 `run_id` + MemorySaver；新增 `POST /api/debate/{run_id}/answer` 用 `Command(resume=)` 续跑。
3. **`extract_json` 必须用非贪心解析**（找首个 `{` 后做括号深度匹配），否则 LLM 在 JSON 后补 `}` 杂文会截断。
4. **默认 search 后端为 Tavily（通用检索）**，专业场景（投研/法律）必须换接垂直数据源（Wind/财汇/企查查/北大法宝/妙想），否则 5 级标签是"看起来严谨的空壳"。
5. **慢是致命体验问题**：N 角色 × M 轮 + 逐条检索会很长；建议默认"异步跑 + 报告推送"，而非同步坐等。

## 前端要点

- `DebateView` 按角色分色（暖色系，避免霓虹），每条发言左侧色条 + 圆点；可信度色点绿→红 5 档 + 图例
- `reframe` 卡片：展示重构后的真实决策 / 两个选项 / 隐含假设
- `clarify` 卡片：运行中弹出"决策前关键一问"，用户输入后回传续跑
- `ReportView`：`react-markdown` 渲染 11 段式 + 审计章节

## 测试约定

- `tests/test_graph_structure.py` 用 FakeModel（不调真实 LLM / 检索）做端到端：
  - reframe → role_gen → 多轮 → synthesis 出报告
  - clarify 中断 / `Command(resume=)` 续跑
  - fact_audit 注入 MockBackend + 专用 FakeModel 抽标
- 隔离 venv 安装 `requirements.txt` 后再跑（项目自身不一定带 venv）。
