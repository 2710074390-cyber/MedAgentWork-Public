# MedAgentWork — 把 LLM 关进质量管线

> 一个把临床医学教材 + 考研真题素材转成**结构化题库**的多 Agent 系统：5 个 Agent + HC-1~HC-18 出题硬约束 + D1-D21 质检维度 + 14+ 条机械化校验规则，覆盖 7 个学科，注册表累计 4,296 题（2026-08-21）。

**核心命题：如何用架构而非 Prompt，把 LLM 的不可靠性约束在可控范围内？**

医学题库场景同时踩中四个最难处理的约束——事实零容错、格式强结构、规模化需求、可追溯性。幻觉是 LLM 的内生属性，不是 Prompt bug，它必须靠**架构**解决。本仓库分享的正是这套架构的全部资产：Agent 提示词、质量门禁代码、契约 Schema 与工程方法论。

## 📌 v4.0 最新特性：HC-18 考研真题配额

每批题库 **约 1/5（目标 20%，合格带 15%–25%）为考研原题**——引用或轻度改编自 1994–2025 考研西综真题素材（GoldenSet 解析产物 9,616 条，上册题干/选项 + 下册贺银成精析答案/解析，按 gs_id 配对），其余 4/5 按教材原创。

- 标注体系：每题携带 `kaoyan_origin` 元数据（gs_id/年份/题号/模式）+ `[源:考研真题 GS-XXX]` 溯源 + 解析来源句
- 机械化：`tools/kaoyan_picker.py`（`pick` 按章节检索真题候选并配对答案 / `check` 终审校验占比）
- 红线：真题答案以官方公布为准——与教材冲突时升级告警，**禁止**静默改答案
- 无真题覆盖章节（如中医方剂、医患沟通）：配额清零、以原创补齐并如实标注

## 架构：5 Agent + 门禁管线

```
输入素材/教材笔记 + 考研真题素材
      │
      ▼
[MedMaster 编排器] ── 规划路径 / 生成下游指令 / 配额编排 / 汇总
      │
      ├─▶ [MedGen 出题]   ──▶ 结构化题目 JSON（契约 Schema 校验 + HC-18 真题配额）
      │         │
      │         ▼
      │   [质量门禁 gate]  ──▶ validate_options.py（Bloom 分层 / R2 / R7 / 字数 SLO）
      │         │
      ├─▶ [MedQC 质检]    ──▶ 质检报告（D1-D21 维度矩阵 + GoldenSet 交叉验证）
      │         │
      ├─▶ [MedFix 修复]   ──▶ 按质检报告定向修复（HC-4b 真题答案保护）
      │         │
      └─▶ [MedReview 主复习资料] ──▶ 最终复习资料产物
      │
      ▼
[用户终审签收] ──▶ healthcheck.py / gate_check.py / kaoyan_picker.py check 复检
```

设计要点（详见 `docs/`）：

- **文件系统当管线**，不上 LangGraph：每个 Agent 通过目录交接，失败点可追踪、可回滚、可人工介入
- **Pipeline as Code**：`pipeline/pipeline.yaml` 集中声明全部阶段、门禁、SLO 阈值
- **声明式规则会失效，强制式才有效**：Agent 契约 Schema（`pipeline/schemas/`）+ 机械化校验脚本兜底
- **对抗 Prompt 阻抗不匹配**：结构化指令 + 明确产出 Schema + 追溯元数据
- **代码、LLM、人三者分工**：LLM 做生成，代码做校验，人做签收

## 目录结构

```
├── docs/                  # 工程方法论与实现教程
│   ├── MedAgentWork_项目介绍_v4.0.md          # 📖 最新版项目介绍（建议从这里读起）
│   ├── MedAgentWork_使用指南_v2.0.md          # 唯一用户手册（DSH 主流程）
│   ├── MedAgentWork_架构演进与技术分享_v3.0.md # 系统全貌与事件复盘
│   ├── MedAgentWork_工程思路与技术分享_v1.0.md # 方法论：问题→失败→最终思路→可迁移结论
│   ├── 基于CherryStudio的实现教程.md             # 如何在 Cherry Studio 中复刻
│   ├── 工作区目录结构与资源路径说明.md
│   ├── 操作流程.txt                              # 每日操作手册
│   └── TODO.md
├── prompts/               # 5 个 Agent 的系统提示词（核心资产，含 HC-18 配额规则）
│   ├── MedMaster_prompt.md       # 编排器：规划路径/生成指令/配额编排/汇总
│   ├── MedGen_prompt.md          # 出题 Agent：教材+真题→结构化题目（HC-18 ≈1/5 原题）
│   ├── MedQC_prompt.md           # 质检 Agent：D1-D21 维度矩阵（D21 考研原题一致性）
│   ├── MedFix_prompt.md          # 修复 Agent：定向修复（HC-4b 真题答案保护）
│   └── Agent5_MedReview_Prompt.md # 主复习资料生成
├── pipeline/              # 管线声明式配置（单一事实来源）
│   ├── pipeline.yaml             # 阶段/门禁/SLO 阈值
│   ├── schemas/                  # 各 Agent 输出契约 Schema（agent2/3/4）
│   └── skills/                   # DSH 技能（medmaster/medgen/medqc/medfix/medreview/medbatch）
├── tests/                 # 回归测试套件（64 用例，零外部依赖可跑）
└── tools/                 # 质量门禁与维护工具
    ├── gate_check.py            # 质量门禁检查（流水线门控 + HALT）
    ├── qbank.py                 # 统一题库注册表 + 交付 MD 导出
    ├── fact_check.py            # 事实校验（页码反查 + GoldenSet 交叉验证）
    ├── kaoyan_picker.py         # 考研真题配额（HC-18）：真题候选检索 + 占比校验
    ├── workflow_state.py        # 工作流状态管理（原子写盘/按批次 HALT）
    ├── validate_options.py      # 选项质量验证（Bloom 分类/R2/R7/字数 SLO）
    ├── healthcheck.py           # 管线健康检查
    ├── run_tests.py             # 测试套件运行器
    ├── bloom_sampler.py / r2_balancer.py / frequency_analyzer.py
    ├── blueprint.py / anchor_bank.py / paper_builder.py   # 考频蓝图/锚点难度/组卷公式
    ├── contract_check.py / metrics.py / runbook.py / maintenance.py / sync_tools.py
    ├── kb/                      # RAG 知识库工具（search_kb/embed_*/索引）
    ├── goldenset/               # 考研真题解析器（parse_goldenset v2.0）
    ├── fixes/                   # 历史产物修复脚本（路径已相对化）
    └── render/                  # 复习资料 HTML 渲染脚本 + 统一押题卷模板
```

## 快速开始

```bash
# 环境要求
# - Python 3.12+（python-pptx/lxml/Pillow 视工具而定）
# - 运行在 DeepSeek Harness / Cherry Studio / 任意支持 MCP 文件系统的 Agent 运行时

# 1. 先读项目介绍（2 分钟）
# docs/MedAgentWork_项目介绍_v4.0.md

# 2. 配置 Agent
# prompts/ 下的 5 个提示词分别配置为 5 个 Agent（DSH 版技能见 pipeline/skills/）

# 3. 运行质量门禁（工具都是独立的，直接可跑）
python tools/gate_check.py --help
python tools/validate_options.py --help
python tools/kaoyan_picker.py pick --subject 内科学 --keywords "心衰,心力衰竭" --target 10 --out candidates.json
```

## 注意

- 本仓库**不含**任何教材素材、题库产物与真题数据（版权与隐私考虑）
- `tools/goldenset/` 与 `tools/kaoyan_picker.py` 默认数据目录为 `GoldenSet/`（需自行准备真题解析产物：`python tools/goldenset/parse_goldenset.py`），路径已相对化到项目根
- 跨工作区同步类工具（`sync_tools.py`、`maintenance.py`）的路径基于 `Path.home()/Desktop`，可按需修改

## 贡献者

系统 5 个 Agent 的运行依赖以下大模型服务，感谢三家模型提供方：

<a href="https://github.com/deepseek-ai" title="深度求索 DeepSeek"><img src="https://avatars.githubusercontent.com/deepseek-ai?s=72&v=4" width="64" height="64" alt="深度求索 DeepSeek"/></a>
<a href="https://github.com/zhipuai" title="智谱 AI · GLM"><img src="https://avatars.githubusercontent.com/zhipuai?s=72&v=4" width="64" height="64" alt="智谱 AI · GLM"/></a>
<a href="https://github.com/QwenLM" title="通义千问 Qwen"><img src="https://avatars.githubusercontent.com/QwenLM?s=72&v=4" width="64" height="64" alt="通义千问 Qwen"/></a>

- **深度求索（DeepSeek）** — [deepseek.com](https://www.deepseek.com)
- **智谱 AI（GLM）** — [zhipuai.cn](https://www.zhipuai.cn)
- **通义千问（Qwen · 阿里云）** — [tongyi.aliyun.com](https://tongyi.aliyun.com)

## 许可

MIT License，详见 [LICENSE](LICENSE)。
