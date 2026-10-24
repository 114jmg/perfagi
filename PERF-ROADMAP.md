# PerfAGI 改造路线图

> 本文档是整个改造工程的唯一规划来源,跨会话维护。每个里程碑完成后更新
> 「当前状态」一节。配套文档:`PERF-FORK.md`(fork 治理)、`PERF-FORK-CHANGES.md`(上游文件改动台账)。

## 一、北极星愿景

用户输入**一句话**,平台自动完成并呈现:

```
"帮我把订单接口压到 1000 并发,给我一份报告和优化建议"
   │
   ├─ ① 需求分析   → 对一句话的理解、SLA 假设、负载模型(并发/时长/梯度)
   ├─ ② 测试规划   → 场景设计、脚本结构、数据准备、执行阶梯
   ├─ ③ 脚本编写   → k6/JMeter 脚本(参数化、断言、思考时间)
   ├─ ④ 压测执行   → 冒烟验证 → 正式压测(实时进度可见)
   ├─ ⑤ 结果分析   → 指标解读、瓶颈归因(证据链:哪层、什么资源、为何)
   └─ ⑥ 结论报告   → 通过/未通过判定 + 分级修改意见 + 报告链接
```

**验收金标准**:perf 账号登录,输入上面那句话,离开键盘,回来看到一份完整报告。

## 二、总体策略:双轨制

| 轨道 | 载体 | 性质 | 节奏 |
|---|---|---|---|
| **A 轨:提示词先行** | DB overlay(per-user)+ target-ctl 增强 | 零 fork 风险,迭代以小时计 | 先行,快速逼近北极星 |
| **B 轨:代码转正** | fork 源码(Go/GraphQL/前端) | 永久性改动,受 merge 纪律约束 | 在 A 轨验证行为稳定后跟进 |

**为什么这个顺序**:提示词是行为,代码是身份。用 overlay 把行为打磨好(改坏了删一行 SQL 就回滚),
验证"多 agent 分工到底怎么切最好用"之后,再花 fork 的成本把它固化成 performer agent、原生工具和 UI。
反过来做(先写代码再调提示词)会把架构错误固化,返工代价高一个数量级。

## 三、设计原则(perf ≠ pentest)

1. **开放,少设限**:渗透提示词的安全栅栏(目标范围限定、破坏性操作确认、双通道语言的强约束)
   大部分不适用于性能测试。保留的只有三条:负载只打注册/自有目标(不打外部服务)、
   资源卫生(临时容器清理、实验产物入库)、诚实报告(不编造数字,证据链可追溯)。
2. **机制保留,领域替换**:提示词里的"机械结构"保留(工具调用纪律、结果契约、上下文纪律、
   双通道语言——这是 flow 引擎的依赖,不是限制),只替换领域内容(渗透方法论→性能工程方法论)。
3. **提示词 git 化**:所有提示词唯一真源在 `perfagi/prompts/*.tmpl`,`deploy.sh` 推到 DB,
   绝不直接在 UI/DB 手改。每次修订一个 commit,可 diff、可回滚。
4. **基准驱动迭代**:一套固定基准任务(见第六节)给提示词打分,修订必须跑分,防退化。

## 四、里程碑总览

依赖关系:M1→M3 是主线;M2 与 M1 可并行;M4 贯穿全程;M5+ 在 M3 验证后逐步展开。

```
M1 异步实验引擎 ──┐
                  ├→ M3 一句话管线(北极星首达) → M5 performer 转正 → M6 原生工具 → M7 UI 产品化 → M8 多目标开放
M2 多agent提示词 ─┘            ↑
        ↑                      │
        └── M4 基准套件(贯穿) ─┘
```

| 里程碑 | 内容 | 载体 | 预估 | 状态 |
|---|---|---|---|---|
| M1 | 异步实验引擎 + 实验索引 | target-ctl(Python) | 2-3 天 | ✅ 完成(2026-10-23) |
| M2 | 多 agent 提示词体系 v2(重点) | DB overlay | 1-2 周起,持续打磨 | 🔄 A 队完成(2026-10-24),B 队排队 |
| M3 | 一句话管线首达 | overlay 编排 | 3-5 天 | 未开始 |
| M4 | 基准任务套件 + 评分 | 脚本+文档 | 首套 2 天,之后每次修订 30 分钟 | 未开始 |
| M5 | performer agent 一等公民 | fork(Go/GraphQL/前端) | 3-5 天 | 未开始 |
| M6 | 原生结构化工具 | fork | 4-6 天 | 未开始 |
| M7 | 实验中心 UI | fork | 约 1 周 | 未开始 |
| M8 | 多目标注册表(开放) | target-ctl + 配置 | 3-5 天 | 未开始 |

---

## 五、里程碑详细任务

### M1 异步实验引擎(target-ctl,不动 fork)

**动机**:现在 perf-run 同步阻塞,30 分钟 soak 顶到终端超时(1200s);实验元数据散在 artifacts
目录里没有索引,后续所有功能(工具化、UI、对比)都依赖"实验可查"。

任务:
- [ ] `POST /run` 改异步:立即返回 run_id,后台线程执行
- [ ] `GET /runs/<id>/status`:queued/running/done/failed + 已耗时 + 当前阶段
- [ ] `GET /runs/<id>/result`:结构化摘要(现有 k6 形状 summary JSON)
- [ ] 实验索引:SQLite(artifacts 目录内,零新依赖)。表 `runs(id, ts, engine, mode, target,
      params_json, status, rps, p95_ms, err_rate, verdict, report_path, notes)`
- [ ] `GET /runs` 升级:过滤(engine/status/日期)+ 分页
- [ ] 保留策略:可配置(默认保留最近 200 次或 30 天)
- [ ] `perf-run --async / --wait <id>` 两个模式
- [ ] (可选)JMeter JTL 尾部增量统计,补齐 JMeter 实时进度

**验收**:10 分钟 soak 通过异步轮询跑完不超时;`/runs` 返回全部历史;两次运行可对比。

### M2 多 agent 提示词体系 v2(本轮重点,纯 overlay)

**动机**:PentAGI 的专家团槽位(pentester/coder/installer/searcher/memorist/adviser + primary_agent)
与性能测试团队天然同构。利用 per-user overlay 把整支队伍"变身",不写一行 Go。

**角色重映射表**(perf 账号 cs88888 全套 overlay):

| 槽位 | 原角色 | → 性能团队角色 | 提示词文件 |
|---|---|---|---|
| primary_agent | 渗透总指挥 | **性能测试总指挥**:需求分析→设计→冒烟→压测→分析→报告的阶段门控编排 | primary-perf.tmpl |
| pentester | 渗透专家 | **性能主程**:测试战役执行者(现 28K 提示词 v1 → v2) | pentester-perf-system.tmpl(已有) |
| question_pentester | 渗透提问 | 性能主程的交互提问 | pentester-perf-human.tmpl(已有) |
| coder | 通用编码 | **脚本工匠**:k6 JS / JMeter JMX 专精(参数化、关联、断言、pacing、Content-Type 陷阱等实战知识) | coder-perf.tmpl |
| installer | 环境安装 | **环境管家**:目标健康检查、perf-reset、数据种子、压测前后状态管理 | installer-perf.tmpl |
| searcher | 漏洞情报搜索 | **技术调研**:k6/JMeter 文档、系统调优参考(2.2 的 web_search mode=research 正好适配) | searcher-perf.tmpl |
| memorist | 渗透记忆 | **实验记忆**:每次 run 存(目标/画像/头部指标/结论/瓶颈),支持"这个目标上次表现如何" | memorist-perf.tmpl |
| adviser | 渗透导师 | **方法论导师**:Little 定律、USE 方法、p95 vs 均值解读、容量规划思维 | adviser-perf.tmpl |
| assistant | 渗透助手 | 性能测试交互助手 | assistant-perf.tmpl |
| flow/task_descriptor | 任务标题 | 中文语义化标题("性能测试:订单接口1000并发") | descriptor-perf.tmpl |

**每个提示词的改造工序**(一个都不能省,这就是"深入理解并改造"):
1. **通读原文**:逐节理解 16 个 section 各自为什么存在(双通道语言是因为工具载荷必须英文;
   结果契约是因为 flow 引擎需要结构化收尾;Graphiti 分类法是因为记忆检索需要类型化)
2. **判定每节去留**:机械结构保留,领域内容替换,渗透安全栅栏删除(换成 perf 三原则)
3. **灌注领域知识**:不是翻译渗透提示词,而是写入真正的性能工程方法论
   (如脚本工匠要知道 per-VU 数据隔离、JMeter HeaderManager 陷阱、k6 executor 选型表)
4. **变量白名单核对**:每个 PromptType 的允许变量不同(templates.go `PromptVariables`),
   写完跑 validator,再经 2.2 加载时重验证(实测我们的三条 overlay 已通过)
5. **双语协议微调**:perf 场景对双通道要求可以放宽——工具载荷仍英文,但 agent 间协作
   报告可以中文(用户阅读为主)

**v1→v2 已知改进点**(性能主程提示词):
- 阶段门控不清晰 → 明确"冒烟未过不得进压测;单点结论不得跳过归因"
- 引擎选择无决策依据 → 写入 k6/JMeter 决策树(脚本复杂度/协议/已有资产)
- 负载模型太随意 → 负载模型速查表(并发/吞吐/RPS 换算,Little 定律校验)
- 结果归因提示词已不错 → 增加与 Grafana 截图/PromQL 证据的联动指引

**验收**:perf 账号一个 flow 里能看到总指挥分工→各角色产出;脚本质量明显提升
(冒烟一次通过率);报告结论有证据链。

### M3 一句话管线(overlay 编排,北极星首达)

- [ ] 总指挥提示词写阶段门控:每阶段入口/出口条件、失败回路(冒烟失败→回脚本工匠)
- [ ] 模糊输入策略:SLA 未给定时,先复述理解+声明假设再执行(不阻塞等待确认)
- [ ] 全部产物归一:报告 HTML 承载全部六段(现有 11 节报告已覆盖大半,补"需求理解"节)
- [ ] 实操调优:拿基准任务 T1-T6 逐个跑通,把总指挥的分工-回收循环调顺

**验收**:金标准那句话,零人工干预,产出完整报告。

### M4 基准任务套件(贯穿全程的回归网)

固定 6 个标准任务,每次提示词修订后全量跑分:

| 编号 | 任务 | 考察点 |
|---|---|---|
| T1 | "给目标做个冒烟测试" | 最小闭环:脚本→运行→报告 |
| T2 | "订单接口 50 并发 2 分钟,p95<200ms" | SLA 判定、通过/失败结论 |
| T3 | "找出系统最大吞吐" | 阶梯加压、饱和识别、停止条件 |
| T4 | "缓存开和关性能差多少" | A/B 对比实验设计 |
| T5 | "为什么高并发时出现 5xx" | 瓶颈归因、证据链、跨层推理 |
| T6 | "跑 10 分钟稳定性测试" | 异步执行、soak 纪律、资源监控 |

评分维度(各 0-2 分,满分 60):脚本可运行、指标解读正确、结论有证据、修改意见可执行、
报告完整、过程自主(无人工纠偏)。记录进 `prompt-bench.md`,防退化。

### M5 performer agent 转正(fork 第一个代码里程碑)

把 overlay 验证过的行为固化为代码身份:
- [ ] DB 迁移:prompt_type / msgchain 相关 enum 加 `performer`(当年 ALTER TYPE 的死值复活)
- [ ] `templates.go`:PromptType 常量 + 变量白名单(按台账)
- [ ] `prompts/performer.tmpl` + `question_performer.tmpl`(内容从 overlay 平移,成为内置默认)
- [ ] `registry.go`:`perf_result` 工具常量 + primary_agent 委托工具注册
- [ ] `handlers.go`:GetPerformerHandler(镜像 GetPentesterHandler)
- [ ] GraphQL AgentsPrompts 枚举 + gqlgen/swag/frontend 三连代码生成
- [ ] 镜像路由:performer 任务 → workbench
- [ ] UI 标签:渗透专家旁出现"性能工程师"

**验收**:新 flow 在 UI 显示性能工程师 agent;渗透 flow 完全不受影响(admin 路径回归测试)。

### M6 原生结构化工具(fork)

终端+jq → 类型化工具:
- [ ] `run_experiment`(async,立即返回 run_id)/`get_run_status`/`get_run_result`
- [ ] `list_experiments`/`compare_experiments`
- [ ] `query_metrics`(Prometheus 直查,PromQL 白名单)
- [ ] 每个工具返回紧凑结构化 JSON(上下文体积纪律:摘要而非全量 JTL)
- [ ] 配置项:target-ctl 地址进 config.go(走 env)

**验收**:agent 全程零 shell 调用完成一次 T3(阶梯加压)任务。

### M7 实验中心 UI(fork)

- [ ] GraphQL 适配层(pentagi 后端代理 target-ctl 实验索引)
- [ ] 实验列表页:过滤/状态/一键打开报告
- [ ] flow 页内嵌实验卡片(进行中实时进度)
- [ ] 基线管理:打标 baseline,自动对比偏差

### M8 多目标注册表(开放性的落点)

- [ ] `targets.yaml`:name/base_url/类型/鉴权/SLA 画像
- [ ] /run 按 target 路由;目标健康预检
- [ ] 临时目标(用户给 URL,白名单网段校验,不打公网)
- [ ] 每目标独立 SLA 与基线

---

## 六、提示词工程方法论(单独成节,因为最重要)

1. **真源在 git**:`perfagi/prompts/*.tmpl`,deploy.sh 推 DB,UI 里只看不错
2. **一提示词一文档**:每个角色的打磨笔记记在 `perfagi/prompts/NOTES-<role>.md`
   (原文结构分析、替换决策、领域知识清单、迭代记录)
3. **小步验证**:改一节跑一次相关基准任务,不攒大版本
4. **失败模式驱动**:agent 犯的每类错(选错工具/跳阶段/报告空洞)都回写成提示词的
   显式反例("不要…;应当…")
5. **上下文预算意识**:提示词不是越长越好,总指挥控制在 8K 内,专家控制在 12K 内,
   给对话历史留空间

## 七、风险与治理

| 风险 | 对策 |
|---|---|
| 上游 merge 冲突 | 台账纪律(只加注册行);每 release 一次 merge;AI 辅助解冲突+人工复核 |
| 提示词改坏 | git 版本化+DB overlay 可秒回滚;基准套件防退化 |
| VM IP 漂移 | 已选手动追模式;.env 四处引用一处脚本化(update-ip.sh,待办) |
| 磁盘 | 已扩 100G;kali 镜像教训:按解压后体积评估 |
| LLM 输出不稳定 | 基准套件多跑两轮取多数;关键判定(SLA 通过)由代码算,不靠模型口算 |
| 密钥泄露面 | PERFAGI_TOKEN/密码永不入库不进提示词;GitHub 密码已暴露过,建议改密+开 2FA |

## 八、当前状态(每次会话结束时更新)

- 2026-10-23:fork 基建完成(perf-fork@v2.2.0,基线镜像已冒烟,GitHub 备份就位,
  VM 磁盘 100G/12 容器健康,双账号分流运行中)。路线图定稿。
- 2026-10-23 晚:**M1 异步实验引擎完成并验收**——POST /run 异步模式(唯一
  排队槽,409 语义),GET /runs/<id>/status(phase+elapsed,免令牌)、
  /runs/<id>/result(完整载荷)、/runs?format=json(过滤+分页);SQLite 索引
  (artifacts/index.db,重启恢复+历史回填+保留策略 PERFAGI_KEEP_RUNS=200);
  perf-run 新增 async/status/wait/list;10 分钟 soak 异步全程不超时,
  排队接棒与全部 409 路径实测通过。HTML 索引页状态列+自动刷新。
  **下一动作:M2 多 agent 提示词体系 v2**(从 primary-perf 总指挥开始)。
- 2026-10-24 凌晨:**M2 A 队完成并实测验收**——新基建 `perfagi/prompts/`:
  文件名=DB prompt_type、`deploy-prompts.py`(变量白名单+{{if}}平衡校验,
  校验不过绝不入库,只推 perf 账号)、每角色 NOTES。A 队三份:
  `primary_agent`(总指挥:名册重映射+实验委派质量条+证据标准)、
  `generator`(六阶段性能管线替代渗透特例;修掉上游重复段 bug;
  一句话/精确请求自适应)、`pentester` v2(异步实验姿势:提交→短轮询→取
  结果,禁止一次调用吞长跑;409 并发语义;k6/JMeter 决策树;Little 定律
  负载速查;冒烟门与归因门标为硬门)。
  **实测**(perf 账号 flow #24,一句话"订单接口冒烟压测"):五 overlay
  零校验警告;generator 拆出 契约→脚本→冒烟→判定+报告 4 子任务(全中文);
  refiner 中途重规划但管线结构保持;agent 自主完成 /info 九项核验、脚本
  编写、perf-run smoke(run 20261023-181438-a174, 1555 rps, p95 0.76ms)、
  报告链接输出。**顺手修 M1 契约漏洞**:/info USAGE 未写死 summary 键名,
  导致 agent 自造 p95/avg 键→分析缺 SLA 行、p95 列为空;已补 canonical
  键名契约 + parse_summary 别名归一(p95→p95_ms 等,实测生效)。
  **B 队进度(2026-10-24)**:`coder`(脚本工匠)完成——k6/JMX 契约
  (URL 必须 __ENV.TARGET、禁 scenarios、group 子度量+groups 对象发射、canonical
  summary 键、think time、种子区间参数化、静态校验 7 步清单、脚本头注释);
  上游破损的 DELEGATION 段(未闭合标签+重复名册)未复制;共享容器事实已核实
  (tools/terminal.go 单一 primary 终端,/work 全角色可见)。
  **实测**:flow 25(参数化 80/20 读写脚本)由总指挥亲自落地,质量命中规范
  ——/info USAGE 在起教育作用;冒烟 rps=1.6 验证 think-time 闭模型数学。
  coder overlay 本身待首个"脚本编写委派"实测。顺手又修两处平台-脚本契约:
  USAGE 补 groups 对象发射契约(报告分端点表读 summary.groups,只教 group()+
  threshold 不够——agent 会自造 group_p95_ms 扁平键)与报告端 `_num`→`_agg_mid`
  (多轮聚合叶子是 {min,p50,max,n},原实现渲染 0)。实测 run 20261024-012044-2020
  分端点表正常。
  **下一动作**:installer(环境管家)→ searcher → memorist → adviser →
  descriptors → refiner → assistant。
