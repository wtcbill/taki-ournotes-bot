# Taki 白名单与渐进式披露查询审阅

审阅仓库：wtcbill/taki-ournotes-bot。基线：main，提交 `5ea0f085c9863b98c0384e086460cedd98a5f662`，于 2026-09-29 获取。本文记录此前完成的只读审阅；本次文档变更仅提交这份报告，不修改业务代码。下文复现使用合成数据和模拟模型返回，不代表线上模型已经出现这些错误，也不估计发生概率。

结论：渐进式披露已经落地，能力 ID 和参数字段也有程序白名单。最值得改的是用户条件的完整性校验：模型返回合法字段、合法值，并不保证它忠实保留了用户条件。目前可复现擅加等级、替换难度、省略技能和旧协议绕过页码检查等问题。

## 项目结构与查询模块

```text
src/ournotes_bot/
  main.py / config.py                    启动与配置
  qq.py                                 QQ 接收、排队、图文准备、发送
  commands.py                           直接命令、选择结果、文字回复
  data.py / yatta.py / haneoka_*.py       游戏资料与来源适配
  song_meta.py / song_traits.py          独立歌曲分析及属性缓存
  visuals.py / *_visuals.py              图片绘制
  ai_query.py                           自然语言查询兼容入口
  local_query.py                        常见问法的确定性解析
  query_capabilities.py                 能力注册表、路由和专项提示词
  query_agent.py                        有界执行流程、缓存、修复、终止
  query_validation.py                   模型动作和用户条件校验
  entity_lexicon.py / song_conditions.py 实体与条件证据
  structured_query.py                   QuerySpec、QueryResult、执行与答案
  ai_client.py / ai_quota.py             模型请求、持久额度
  query_metrics.py / query_debug.py      匿名统计与调试计数
tests/                                  离线行为测试、模拟返回、临时数据
scripts/                                更新器、打包检查、图片预览
docs/                                   开发导航和发布说明
```

这里的白名单是查询能力、参数和实体约束；该模块不是 QQ 用户或群身份白名单。

## 实际执行链路

1. `/问` 进入 `AIQueryParser.answer_with_outcome`，交给 `QueryAgent.run`。
2. 先查缓存，再检查范围，尝试直接命令和 `parse_local_query`；本地命中不调用模型。
3. `local_route` 有明确能力证据时，直接选能力；否则给模型一个简短能力目录，只允许返回 `route` 动作。
4. 只向模型披露所选能力的参数结构、领域规则和少量实体锚点，不发送完整曲库或卡库。
5. `validate_capability_action` 检查动作、能力、字段和值，再核对页码、稀有度、实体及部分歌曲条件。
6. 校验失败时，向模型披露结构化 Observation，最多修复一次；修复后重新校验。
7. 校验通过后执行 `QuerySpec`，用可靠数据构建 `QueryResult` 和答案，模型自由文本不作为事实答案。

当前五个能力：`song.search`、`chart.get`、`member_card.search`、`support_card.search`、`song.meta`。

当前限制：最多 3 次模型调用、2 次工具执行、1 次修复、5 个步骤；流程在调用前检查 18 秒预算。这个预算检查不能中断已经运行的同步查询，不宜描述成端到端硬超时。真实模型请求前持久计入额度，成功计划缓存仍会重新检索数据。

已有优点应保留：仅执行白名单里的函数；不让模型生成任意命令、URL 或事实答案；实体回查本地资料；拒绝未知字段；页码和稀有度的新协议校验；同一查询结果供文字和图片使用；有限修复、匿名统计及离线测试。

## 已复现的问题

### P1：谱面难度可以被模型替换

位置：[query_validation.py:155](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_validation.py#L155)、[query_validation.py:218](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_validation.py#L218)。

用户问 `迷星叫EX物量`；模拟模型返回 `chart.get`、`difficulty="HARD"`。校验通过，最终返回 HARD 的数据，并标记 `success`。复现中的 HARD 等级和 Note 数来自合成测试资料，不是实际游戏事实。

原因：歌曲搜索会检查原文难度，但谱面能力只检查难度是否属于合法枚举，没有检查是否与原文一致。

建议：给 `chart.get` 补独立难度证据检查，区分未指定、明确指定、冲突和无法确认；拒绝替换、删除或擅加难度，不要直接复用包含歌曲等级逻辑的整段校验。

### P1：有角色锚点时，技能条件可以被省略

位置：[query_validation.py:210](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_validation.py#L210)、[query_validation.py:237](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_validation.py#L237)。

用户问 `高松灯相关的成员卡里，技能带有得分提升的有哪些`；模拟模型返回空的 `skill_query` 和 `skill_kind`。校验通过，返回角色卡列表；合成数据中的回复技能卡也进入结果。

原因：现有逻辑检查模型提出的技能片段是否出现在原文，但不完整检查原文中的技能条件是否被保留。禁止丢技能的保护只在没有实体锚点时触发。

建议：把角色锚点和技能谓词分别校验。凡原文明确要求技能筛选，空条件不能降级成全部角色卡；不能可靠解析时应澄清或拒绝，不能扩大范围。技能类型和完整效果片段也需避免被缩短成更宽的条件。

### P1：没有等级条件时，模型可以擅加等级

位置：[query_validation.py:62](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_validation.py#L62)。

用户问 `MyGO有哪些歌曲，麻烦列给我`；模拟模型添加 `level_operator="gt"`、`level=30`。校验通过，执行 `MyGO lv>30`，在合成曲库中返回空结果。

原因：有原文等级证据时会比对；没有证据时，没有对应分支拒绝模型新增等级。

建议：原文明确无此筛选时要求默认空值；原文有未识别的疑似条件时标记无法确认并澄清，避免“未识别”被视为“允许模型自行填值”。

### P1：旧 intent 协议绕过新协议的页码检查

位置：[query_validation.py:111](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_validation.py#L111)。

用户问 `高松灯相关的成员卡内容 页3`；模拟模型返回旧格式 `{"intent":"card","query":"高松灯"}`。系统接受并生成 `page=1` 的查询。同一问句在新协议下若返回 `page=1`，会正确判为 `INVALID_ARGUMENTS`。

原因：旧格式分支在新协议的页码、稀有度等检查前提前返回。

建议：所有模型输出先规范化成一个内部动作，再走同一套校验。若仍须兼容旧调用者，将兼容转换留在明确的适配入口，不让实时模型返回另一种更宽松协议。不能只用新提示词代替程序检查。

### P2：本地路由会把谱面问题锁到歌曲列表能力

位置：[query_capabilities.py:188](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_capabilities.py#L188)。

`迷星叫这首歌曲的EX物量是多少` 无本地完整解析结果，却因先匹配“歌曲”被路由为 `song.search`，后面的“物量”分支不会执行。此时渐进式披露只展示歌曲搜索能力；即使模型返回 `chart.get`，也会被能力绑定检查拒绝。

建议：针对查询目标做规则，而不是仅按关键词先后；区分“列出满足谱面等级条件的歌曲”和“获取某首歌曲物量”。存在冲突且规则不能可靠判断时，让 `local_route` 返回 `None`，交给受限能力路由。不要简单把所有含“谱面”的问句改路由为 `chart.get`，否则会破坏歌曲条件查询。

### P2：合法 JSON 的错误字段类型会让校验器抛异常

位置：[query_capabilities.py:73](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/query_capabilities.py#L73)。

模拟模型返回 `difficulty=["EXPERT"]`，`invalid_fields` 执行集合成员判断时抛 `TypeError`，没有生成可修复的校验结果。QQ 外层会退通用失败文字；CLI 调用该入口时异常会传播。`level_operator`、`skill_kind` 的同类判断也需检查。

建议：先检查字符串类型，再检查枚举；所有可由 JSON 表达的错误类型都应得到结构化失败。避免仅在最外层捕获异常，因为那样仍损失修复机会。

### P2：页码数字会被当作卡牌 ID

位置：[entity_lexicon.py:263](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/entity_lexicon.py#L263)、[entity_lexicon.py:305](https://github.com/wtcbill/taki-ournotes-bot/blob/5ea0f085c9863b98c0384e086460cedd98a5f662/src/ournotes_bot/entity_lexicon.py#L305)。

合成资料包含 ID 1、2 的高松灯卡。问 `高松灯相关的成员卡内容 页2`，模型即使正确返回角色和 `page=2`，仍被判 `unknown_entity`。锚点提取把页码 2 当成卡 ID；随后把角色锚点合并进这张卡，导致与模型提出的角色对象不一致。改成没有对应卡 ID 的 `页3` 则通过。

建议：实体扫描前排除已确认的页码、稀有度、等级等条件范围；自然语言中的卡 ID 尽量要求 `ID`、`号卡` 等明确语境，同时保留直接命令的纯数字 ID 用法。

## 渐进式披露自身的改进

- **修复信息更具体。** 当前 `Observation.allowed` 填的是参数字段名，方案文档的示例却是枚举值。可用 `field_errors` 表示字段、错误原因、允许值及程序已确认的原文条件，让一次修复更有把握；仍不传堆栈、完整记录和外部响应。
- **证据披露从实体扩展到条件。** 目前 `_evidence` 仅给一个实体字符串。可以加少量程序已经确认的难度、等级、页码、稀有度证据及文本位置；保持按能力披露，不能把未确认推断包装成事实。
- **减少双轨逻辑。** `validate_route` 目前从不设置 `prefetched`，但 Agent 保留了对应分支；`allow_skill`、`allow_difficulty` 字段没有使用。可在后续维护中局部移除无效分支和元数据，或纳入实际校验，避免注册表给人错误保证。
- **统一字段定义。** 参数白名单、类型、默认值、枚举、提示词和修复信息分散维护。用小型字段描述共用这些信息即可，暂不需要新增服务、自由工具系统或复杂 Agent 框架。用户条件的语义校验仍须独立保留。
- **扩展行为评测。** 现有测试覆盖作用域披露、未知对象、条件篡改、次数上限、修复和额度。优先补上述反例，以及数组/对象/null 等字段类型、关键词冲突、页码与 ID 混淆、重复调用和终止原因；模拟模型测试不能替代真实模型识别率评测。

## 建议实施顺序

1. 统一模型动作入口，补难度、等级、技能条件的双向约束和 JSON 类型检查；每个复现补一条行为回归测试。
2. 修复实体证据里的数字语境和本地路由冲突；保留已工作的本地零 AI 解析。
3. 改进 Observation，复用字段定义，删除无效兼容分支；保持现有文件职责，局部调整即可。
4. 经授权再做真实模型评测，比较本地命中率、路由准确率、条件保留率、修复成功率、请求数、耗时和费用。当前未调用真实模型。

## 本次验证

- 使用本地审阅副本及隔离 Python 3.12 环境，按仓库依赖安装。
- `test_query`、`test_ai_lexicon`、`test_query_refactor`、`test_natural_query_eval`、`test_stage5_limits`、`test_observability`、`test_query_debug` 共 **71 个离线测试通过**。
- 额外用合成资料、模拟模型返回复现了上述 7 类问题；测试和复现中阻止网络连接，清空 QQ/AI 凭据。
- 全套执行了 267 个测试，但**没有全套通过**：本机缺少代码指定的 CJK 字体，产生 40 个 error 和 1 个 failure（含字体失败引出的测试局部变量错误）。不以此宣称图片渲染正常。
- 未执行真实 AI 调用、QQ 收发、生产缓存同步、Windows 更新或部署。没有验证线上模型出错率。
- 只读审阅结束时仓库 `git status --short` 为空，提交仍为上述基线；复现资料保存在审阅工作区、未加入仓库。本次文档变更新增此报告，不代表报告中的问题已修复。
