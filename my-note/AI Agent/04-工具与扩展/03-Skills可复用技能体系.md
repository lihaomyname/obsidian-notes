# Skills 可复用技能体系

> Skill 是面向 Agent 的可复用能力包，把多步骤流程、领域知识、脚本、参考资料和模板组织在一起；Tool 则更接近单次原子操作。

## Tool 与 Skill

| Tool | Skill |
| --- | --- |
| 一次搜索、一次 API、一次文件读取 | 完整审查流程、报告生成规范、领域 SOP |
| 主要描述输入、输出与执行接口 | 主要描述何时触发、按什么步骤工作、读取哪些资源 |
| Runtime 直接调用 | Agent 匹配后加载指令，再组合多个 Tool 执行 |

## 典型目录

```text
skill-name/
├── SKILL.md       # 元数据与核心指令
├── scripts/       # 可执行脚本
├── references/    # 详细参考资料
└── assets/        # 模板、Schema、静态资源
```

`SKILL.md` 的 description 决定 Skill 是否能被正确匹配，因此需要写清能力、触发条件和不适用场景。

## 渐进式披露

Deep Agents 链接把加载分为三层：

1. 启动时只加载 name 和 description。
2. 匹配成功后读取完整 `SKILL.md`。
3. 执行过程中按需读取 scripts、references 和 assets。

这种方式减少无关内容进入上下文，也避免大量 Skill 指令同时影响模型。

## 生命周期与作用域

课表要求关注加载、作用域隔离和安全边界。链接资料给出的落地方向包括：

- 组织、团队、项目、个人 Skill 分层。
- 同名 Skill 有明确覆盖顺序。
- 主 Agent 和自定义子 Agent 显式控制继承范围。
- 共享 Skill 可只读，个人 Skill 可写。
- 修改共享 Skill 时可通过中断机制等待人工审批。

## Skill 组合与依赖

Skill 可以调用多个 Tool，也可引用脚本和模板，但依赖应显式写在说明中。若一个 Skill 隐式依赖未授权工具、网络或特定运行时，匹配成功后仍可能执行失败。

## 安全原则

- 外部 Skill 在使用前需要审查内容和脚本。
- 共享规范默认只读，修改走审批。
- 子 Agent 仅加载任务需要的 Skill。
- Skill 指令不能绕过工具权限和沙箱。
- 资源按需加载，不把整个知识库一次性塞进上下文。

## 关联笔记

- [[01-Tool-Calling与工具系统]]
- [[02-MCP协议]]
- [[04-沙箱与执行安全]]
- [[../05-上下文记忆/01-上下文工程|上下文工程]]

## 来源与核对范围

- [POPO：Agent开发学习课表](https://docs.popo.netease.com/team/pc/EHRZHAOPIN/pageDetail/5b5253077e2a43f7ac89deedfc6db29a)
- [Deep Agents：Skills — 可复用的 Agent 能力包](https://datawhalechina.github.io/deepagents-in-action/chapters/ch07-skills/)
- [AI Agent Book：Skills 技能系统](https://waylandz.com/ai-agent-book/%E7%AC%AC05%E7%AB%A0-Skills%E6%8A%80%E8%83%BD%E7%B3%BB%E7%BB%9F/)

> “Skill 标准的支持产品数量”等生态数据没有写入正文；本文只整理目录、披露、作用域和权限等机制。
