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

```text
Level 1：name + description
           ↓ 匹配到当前任务
Level 2：完整 SKILL.md
           ↓ 执行某个具体步骤时需要
Level 3：指定 script / reference / asset
```

关键点是“按需读指定资源”，不是匹配后把整个 Skill 目录都塞入上下文。

## `SKILL.md` 示例

```markdown
---
name: dependency-review
description: 分析 Maven 项目的依赖冲突与冗余；仅做诊断，不直接改 pom.xml。
---

# 依赖审查

1. 读取 `references/checklist.md`，确认检查项。
2. 收集 `pom.xml` 与依赖树。
3. 使用 `scripts/analyze.sh` 生成机器结果。
4. 按 `assets/report-template.md` 输出报告。

## 限制

- 默认只读。
- 修改依赖前必须获得用户明确授权。
```

description 同时写了“做什么、何时触发、不做什么”，比只写“一个 Maven Skill”更利于准确匹配。

## 匹配与加载过程

```python
def load_skill_for(task, catalog):
    # catalog 此时只有 name 和 description，成本较低
    candidates = semantic_match(task, catalog)

    for candidate in candidates[:3]:
        if candidate.description_clearly_applies(task):
            instructions = read(candidate.path / "SKILL.md")
            return instructions

    return None
```

匹配可以使用规则、语义相似度或模型判断，但结果仍要经过适用范围检查。若多个 Skill 都匹配，应明确执行顺序和冲突处理，而不是把所有指令混在一起。

## 生命周期与作用域

课程要求关注加载、作用域隔离和安全边界。链接资料给出的落地方向包括：

- 组织、团队、项目、个人 Skill 分层。
- 同名 Skill 有明确覆盖顺序。
- 主 Agent 和自定义子 Agent 显式控制继承范围。
- 共享 Skill 可只读，个人 Skill 可写。
- 修改共享 Skill 时可通过中断机制等待人工审批。

## Skill 组合与依赖

Skill 可以调用多个 Tool，也可引用脚本和模板，但依赖应显式写在说明中。若一个 Skill 隐式依赖未授权工具、网络或特定运行时，匹配成功后仍可能执行失败。

可以把依赖拆成四类：

- **工具依赖**：需要 Shell、浏览器、数据库还是某个 MCP Server。
- **资源依赖**：必须读取哪些 reference 或模板。
- **权限依赖**：只读、可写、联网或需要审批。
- **环境依赖**：运行时、系统命令、凭据和版本约束。

## 多来源覆盖规则

当系统 Skill、组织 Skill、项目 Skill、个人 Skill 同时存在时，需要固定优先级。例如：

```text
系统内置 < 组织共享 < 项目目录 < 用户个人
```

同名时由高优先级版本覆盖，目录扫描顺序不能成为隐含规则。覆盖适合定制说明，但不能用来突破 Runtime 的权限上限。

## Skill、Tool、Memory 的分工

| 组件 | 回答的问题 | 例子 |
| --- | --- | --- |
| Skill | 这类任务按什么 SOP 完成 | 代码审查步骤与报告模板 |
| Tool | 具体如何执行一个动作 | 读取文件、运行测试、创建工单 |
| Memory | 过去有哪些可复用事实 | 项目约定、用户偏好、历史决策 |

Skill 可以指导 Agent 读取 Memory 并调用 Tool，但 Skill 自身不应成为绕过权限的执行入口。

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

- [Deep Agents：Skills — 可复用的 Agent 能力包](https://datawhalechina.github.io/deepagents-in-action/chapters/ch07-skills/)
- [AI Agent Book：Skills 技能系统](https://waylandz.com/ai-agent-book/%E7%AC%AC05%E7%AB%A0-Skills%E6%8A%80%E8%83%BD%E7%B3%BB%E7%BB%9F/)

> “Skill 标准的支持产品数量”等生态数据没有写入正文；本文只整理目录、披露、作用域和权限等机制。
