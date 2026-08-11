# MedAgentWork — 把 LLM 关进质量管线

> 一个把临床医学教材转成**结构化题库**的多 Agent 系统：5 个 Agent + 6 层防御 + 32 条硬约束 + 13 条机械化校验规则，13 个批次累计产出 2,923 道题。

**核心命题：如何用架构而非 Prompt，把 LLM 的不可靠性约束在可控范围内？**

医学题库场景同时踩中四个最难处理的约束——事实零容错、格式强结构、规模化需求、可追溯性。幻觉是 LLM 的内生属性，不是 Prompt bug，它必须靠**架构**解决。本仓库分享的正是这套架构的全部资产：Agent 提示词、质量门禁代码、契约 Schema 与工程方法论。

## 架构：5 Agent + 门禁管线

```
输入素材/教材笔记
      │
      ▼
[MedMaster 编排器] ── 规划路径 / 生成下游指令 / 汇总
      │
      ├─▶ [MedGen 出题]   ──▶ 结构化题目 JSON（契约 Schema 校验）
      │         │
      │         ▼
      │   [质量门禁 gate]  ──▶ validate_options.py（Bloom 分层 / R2 / R7 / 字数 SLO）
      │         │
      ├─▶ [MedQC 质检]    ──▶ 质检报告（13 条机械化校验规则）
      │         │
      ├─▶ [MedFix 修复]   ──▶ 按质检报告定向修复
      │         │
      └─▶ [MedReview 主复习资料] ──▶ 最终复习资料产物
      │
      ▼
[用户终审签收] ──▶ healthcheck.py / gate_check.py 复检
```

设计要点（详见 `docs/`）：

- **文件系统当管线**，不上 LangGraph：每个 Agent 通过目录交接，失败点可追踪、可回滚、可人工介入
- **Pipeline as Code**：`pipeline/pipeline.yaml` 集中声明全部阶段、门禁、SLO 阈值
- **声明式规则会失效，强制式才有效**：Agent 契约 Schema（`pipeline/agent_contracts.json`）+ 机械化校验脚本兜底
- **对抗 Prompt 阻抗不匹配**：结构化指令 + 明确产出 Schema + 追溯元数据
- **代码、LLM、人三者分工**：LLM 做生成，代码做校验，人做签收

## 目录结构

```
├── docs/                  # 工程方法论与实现教程（建议从 v1.0 读起）
│   ├── MedAgentWork_工程思路与技术分享_v1.0.md   # 方法论：问题→失败→最终思路→可迁移结论
│   ├── MedAgentWork_架构演进与技术分享_v3.0.md   # 系统全貌与事件复盘
│   ├── 基于CherryStudio的实现教程.md             # 如何在 Cherry Studio 中复刻
│   ├── 工作区目录结构与资源路径说明.md
│   ├── 操作流程.txt                              # 每日操作手册
│   └── TODO.md
├── prompts/               # 5 个 Agent 的系统提示词（核心资产）
│   ├── MedMaster_prompt.md       # 编排器：规划路径/生成指令/汇总
│   ├── MedGen_prompt.md          # 出题 Agent：教材→结构化题目
│   ├── MedQC_prompt.md           # 质检 Agent：13 条校验规则
│   ├── MedFix_prompt.md          # 修复 Agent：定向修复
│   └── Agent5_MedReview_Prompt.md # 主复习资料生成
├── pipeline/              # 管线声明式配置（单一事实来源）
│   ├── pipeline.yaml             # 阶段/门禁/SLO 阈值
│   └── agent_contracts.json      # Agent 输出契约 Schema
└── tools/                 # 质量门禁与维护工具
    ├── gate_check.py            # 质量门禁检查
    ├── validate_options.py      # 选项质量验证（Bloom 分类/R2/R7/字数 SLO）
    ├── healthcheck.py           # 管线健康检查
    ├── bloom_sampler.py         # Bloom 分层采样监控
    ├── r2_balancer.py           # 选项长度比均衡
    ├── frequency_analyzer.py    # 考点频次分析
    ├── contract_check.py        # Agent 契约合规校验
    ├── metrics.py / runbook.py / maintenance.py / sync_tools.py
    ├── fixes/                   # 历史产物修复脚本（路径已相对化）
    ├── goldenset/               # GoldenSet 交叉验证工具（数据不分享）
    └── render/                  # 复习资料 HTML 渲染脚本
```

## 快速开始

```bash
# 环境要求
# - Python 3.12+（python-pptx/lxml/Pillow 视工具而定）
# - 运行在 Cherry Studio / 任意支持 MCP 文件系统的 Agent 运行时

# 1. 先读方法论（30 分钟）
# docs/MedAgentWork_工程思路与技术分享_v1.0.md

# 2. 配置 Agent
# prompts/ 下的 5 个提示词分别配置为 5 个 Agent
# 参考 docs/基于CherryStudio的实现教程.md

# 3. 运行质量门禁（工具都是独立的，直接可跑）
python tools/gate_check.py --help
python tools/validate_options.py --help
```

## 注意

- 本仓库**不含**任何教材素材、题库产物与真题数据（版权与隐私考虑）
- `tools/goldenset/` 脚本默认数据目录为 `GoldenSet/`（需自行准备），路径已相对化到项目根
- 跨工作区同步类工具（`sync_tools.py`、`maintenance.py`）的路径基于 `Path.home()/Desktop`，可按需修改

## 许可

MIT License，详见 [LICENSE](LICENSE)。
