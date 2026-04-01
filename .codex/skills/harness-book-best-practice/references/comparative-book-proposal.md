# 《Claude Code 和 Codex 的 Harness 设计哲学：殊途同归还是各表一枝》

## 一、任务定义

这套内容不是单独介绍 Claude Code，也不是单独介绍 Codex。它要做的，是把两套 harness 放在同一张工程桌面上比较，看它们分别如何处理下面这些现实：

- 模型并不可靠
- 工具调用会扩大错误半径
- 会话需要连续状态
- 团队需要局部规则、审批和可追责性
- 多代理需要分工，而不是热闹

问题不在于谁更“聪明”，而在于谁把不可靠的模型接进现实世界以后，长出了怎样的约束结构。

## 二、核心论点

这套内容围绕一个主判断展开：

> Claude Code 与 Codex 都承认模型不是可信执行体，但前者更像从运行时纪律出发塑造 harness，后者更像从类型化控制面与策略系统出发塑造 harness。

换句话说，它们并不是简单的“你有我也有”的功能镜像。它们确实走向了同一个目的地：让模型在受约束条件下完成工程任务。但它们沿路长出的骨架不同。

## 三、建议篇幅

目标篇幅：约 50 页

建议分配：

- 前言：3 页
- 第 1 章：6 页
- 第 2 章：8 页
- 第 3 章：8 页
- 第 4 章：10 页
- 第 5 章：7 页
- 第 6 章：6 页
- 第 7 章：5 页
- 附录：3 页

这比前一套 80 页的 Claude Code 单体内容更紧，因为比较写法天然要求收敛。话说得太散，比较就会失效。

## 四、比较轴

全篇重点围绕七个轴：

1. 为什么比较这两套 harness
2. Prompt 组装与 instruction fragment
3. Query loop 与 thread/rollout/state
4. Tool schema、sandbox、approval、execpolicy
5. Skills、AGENTS.md、hooks 与本地治理
6. 多代理、验证、持久状态与恢复
7. 最终判断：它们究竟是同一路数，还是不同谱系

## 五、写法边界

这套内容必须遵守两个边界。

第一，不提供源代码副本。我们讨论架构，不转录实现。

第二，不把比较写成功能营销。技术比较最容易陷入一种偷懒：把几十个功能点排成对照表，仿佛谁多一个开关谁就赢了。那不是架构比较，那只是采购手册。

## 六、文风要求

延续上一套内容的基本气质：

- 语言专业，不装神秘
- 有判断，不假客观
- 偶尔有一点反讽，但不拿反讽代替论证
- 以工程现实为中心，不以品牌叙事为中心

所谓“王小波的口吻”，不应理解为模仿句式，而应理解为三种克制：

- 对技术神话保持警惕
- 对制度和约束保持尊重
- 对空话和假大词保持一点必要的不耐烦

## 七、依据文件

Claude Code 侧主要依据：

- `src/constants/prompts.ts`
- `src/utils/systemPrompt.ts`
- `src/query.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/compact/*`
- `src/utils/forkedAgent.ts`
- `src/tools/SkillTool/*`

Codex 侧主要依据：

- `AGENTS.md`
- `core/src/lib.rs`
- `instructions/src/lib.rs`
- `instructions/src/fragment.rs`
- `instructions/src/user_instructions.rs`
- `tools/src/lib.rs`
- `tools/src/local_tool.rs`
- `execpolicy/src/lib.rs`
- `hooks/src/lib.rs`
- `hooks/src/engine/mod.rs`
- `sdk/typescript/src/thread.ts`

## 八、结论预告

这套内容大概会落到这样一个结论上：

- 它们是“殊途同归”，因为都把 harness 当成模型之上的真正控制层
- 它们又“各表一枝”，因为 Claude Code 更强调运行时连续编排，Codex 更强调显式模块化、可序列化 instruction、策略语言与线程化状态

要是把两者混成一种通用产品，就看不见真正值得学的地方了。
