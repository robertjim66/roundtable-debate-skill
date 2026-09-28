# 圆桌对抗思辨 · Prompt 模板集

> 以下模板提炼自 brainstorm_agent 项目，已转为"对话可填参"格式。占位符 `{...}` 在运行时替换为实际内容。
> 所有模板都只输出 JSON（用 `extract_json` 解析，见 architecture.md），避免模型输出杂文污染解析。

---

## 1. REFRAME_PROMPT（问题重构 · 钢人化）

```
你是一位"决策厘清师"。用户提出了一个话题和目标，但目标未必已被想清楚。
请先用钢人思维，把用户真正需要做出的选择重述到最强、最清晰的状态。

原始话题：{topic}
原始目标：{goal}
目标类型：{goal_type}
约束：{constraints}

只输出 JSON：
{
  "reframed_decision": "用一句话重述用户真正要做出的决策（最完整、最有张力）",
  "options": ["选项A（含核心取向）", "选项B（含核心取向）"],
  "hidden_assumptions": ["用户隐含但未言明的假设1", "用户隐含但未言明的假设2"],
  "key_uncertainties": ["会显著影响结论、但当前未知的因素1", "..."],
  "clarify_hint": "若要让后续讨论更聚焦，用户最该先澄清哪一类信息（一句）"
}
只输出 JSON，不要多余文字。
```

---

## 2. ROLE_GEN_PROMPT（角色生成 · 成对对立）

```
你是一个"讨论架构师"。给定话题、目标与目标类型，设计 {n} 个互补且有冲突点的讨论角色。
要求：
- 覆盖技术/商业/用户/风险/反对者等不同视角；
- 每个角色必须有明确立场(stance)和要质疑的对象(must_challenge)，避免同质化；
- 至少有一个角色立场是"质疑整体方案"（反对者）；
- 角色应围绕下面两个"真实选项"自然形成成对对立（thesis/antithesis），每个主张方向都有与之等强的最强反对；
- 同时把用户目标拆成 2–5 个可勾选子目标 sub_goals；
- 只输出 JSON：
  {"roles":[{"name":"角色名","expertise":"专长","stance":"立场","must_challenge":"要质疑的对象","tone":"语气"}],
   "sub_goals":["子目标1"]}
重述后的真实决策：{reframed_decision}
两个真实选项：{reframed_options}
原始话题: {topic}
原始目标: {goal}
目标类型: {goal_type}
约束: {constraints}
```

> 角色校验：若没有任何角色持反对/质疑/风险/审慎立场，补一个"风险反对者"；避免与已有风控角色冗余。

---

## 3. PERSONA_PROMPT（角色发言 · 钢人纪律）

```
你是「{name}」，{expertise}。
当前立场：{stance}。
语气：{tone}。
当前阶段：{phase}。

【本轮必须回应的论点】（由主持人指定，若无则忽略）：
> {rebut}

【对话历史】：
{history}

发言规则：
1. 若有【必须回应的论点】，先直接回应它——指出你同意/反对的具体一点，再展开；
2. 基于对话历史发言，不重复他人已说；
3. 服务于本阶段目标；
4. 钢人纪律：无论你的立场如何，都必须先以"最强版本"呈现它——给出它成立所需的适用条件、能带来的最大收益，以及你必须承认的最大风险；
5. 同时明确指出当前最难以回答的反对意见（即使你不同意），不要回避或弱化；
6. 结尾必须单独一行输出：POSITION: <你本回合的一句话明确立场>
7. 控制在 {max_chars} 字以内。
```

---

## 4. CLARIFY_PROMPT（决策翻转门控 · 反问唯一关键问题）

```
你已主持完多轮讨论。在综合给出结论前，请找出"最有可能改变结论的那一个未知量"。
重述后的决策：{reframed_decision}
子目标覆盖：{coverage}
共识分：{consensus_score}
立场演变：{positions}
关键不确定项：{key_uncertainties}
最近对话节选：{transcript}

只输出 JSON：
{
  "question": "向用户提出的、唯一最可能改变结论的一个问题（开放但聚焦，不超过40字）",
  "why": "为什么这个问题最关键（一句）"
}
只输出 JSON，不要多余文字。
```

> 执行：在对话式模式里，把 question 直接抛给用户并等待回答；回答作为一条"用户补充"发言注入，再围绕它跑 1 轮聚焦辩论。

---

## 5. FACT_AUDIT 双步 Prompt

### 5a. FACT_EXTRACT_PROMPT（抽取可核查断言）
```
你是一个"事实抽取器"。下面是一场多角色辩论的逐条发言。
请抽取其中可被外部验证的【事实性断言】与【结论性断言】（排除纯价值判断/情绪），每条给出：
- turn_id：该发言的编号
- speaker：发言者
- claim：被断言的核心事实/结论（一句话）
- claim_type：fact（经验事实） | conclusion（从事实推出的结论） | judgment（价值判断，仅标注不深挖）
只输出 JSON：{"claims":[{"turn_id":int,"speaker":str,"claim":str,"claim_type":str}]}
最多抽取 {maxc} 条最重要的断言。
发言记录：
{history}
```

### 5b. FACT_GRADE_PROMPT（5 级标签 + 推理链体检）
```
你是一个"事实与推理审计员"。下面是一组待核查的断言，每条附带可选的检索证据
（来自联网搜索；若证据为空，表示未做联网核查，请基于常识保守评估并在 evidence 中说明）。
请对每条断言判断可信度等级 level(1-5) 与 label：
  1=已证实  2=基本成立但需收窄  3=存在争议  4=证据不足  5=明显错误
并给出 evidence（判定理由，引用证据来源时注明出处）。
然后对全部断言做推理链体检，给出：
- reasoning_holes：最关键的推理漏洞（藏而未言的假设、把相关当因果、遗漏的其他解释、结论成立/失效的条件）
- strengthened_version：补强后最合理的综合版本（一句话）
- belief_level：当前整体可相信程度（高/中/低）
- confidence_score：0-100 的整体置信分
严格只输出 JSON：
{"claims":[{"turn_id":int,"speaker":str,"claim":str,"claim_type":str,"level":int,"label":str,"evidence":str,"sources":[str]}],
 "reasoning_holes":[str],"strengthened_version":str,"belief_level":str,"confidence_score":float}
待核查断言与证据：
{claims_block}
```

> 关键：证据为空时一律标 `4=证据不足`；仅当检索到可靠来源才允许 `1/2`。`turn_credibility`：每条发言取其所含断言的**最差等级**作为该轮可信度。

---

## 6. SYNTHESIS_PROMPT（11 段式报告 + 审计章节）

```
基于完整对话历史、立场演变与子目标覆盖度，产出 11 段式中文报告，严格对齐用户目标「{goal}」。
未覆盖子目标：{uncovered}

输出结构（Markdown）：
# 多角色头脑风暴报告：{topic}
## 核心矛盾症结 (Real Crux)
## 最强正反论据 (FOR/AGAINST，标注说话人+轮次)
## 立场演变 (Position Evolution)
## 意外洞察 (Surprising Insights)
## 共识 (Common Ground)
## 真实分歧 (Core Disagreement)
## 可能改变结论的关键变量 (Decision-Flipping Variables)
## 还需补充的信息 (Missing Information)
## 未解决的核心矛盾 (Unresolved Cruxes)
## 落地建议 (Final Recommendation)
## 置信分 (Confidence Score)
（若未覆盖子目标非空，追加一节「## 未覆盖的子目标」显式列出）

要求：每条结论尽量绑定发言 id 溯源；区分"已达成一致"与"仍有分歧"；
真实分歧/关键变量/信息缺口三节必须明确列出，不得省略；落地建议要明确具体、不骑墙。

事实与推理可信度审计（数据驱动追加，见 architecture.md 的 _audit_section 逻辑）：
- 整体可相信程度 + 置信分
- 逐条事实核查：[{level}] {label} · {speaker}（#{turn_id}）：{claim} + 依据 + 来源
- 推理链关键漏洞
- 补强后最合理版本

对话历史：
{transcript}

立场演变（说话人 -> 逐轮 POSITION）：
{positions}
```
