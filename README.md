# check-standards

> Java 后端代码生成 / 修改完成后的**关键规范兜底自检**（收敛闸门）——用 grep/ast-grep 实际扫描产出，逐项核对关键规范，把没执行到位的项**提示用户确认是否补齐**。

规范条目多、分散在多个规范 skill（java-code-standards / comment-standards / database-standards / build-standards），AI 编码时容易漏执行。本 skill 是**最后一道兜底**：代码写完或改完后，逐项核对 12 项 HIGH + 代码规范组 C1-C6 + 注释组 N1-N4 + INFO + 场景化，每项附「文件:行号」证据，未到位项统一提示用户确认。

## 它解决什么问题

- **注释没加全** → 注释组 N1-N4 核对类/方法 Javadoc 覆盖率、步骤注释、禁翻译式
- **日志没加全** → HIGH #2 核对 @Slf4j / System.out / 入口入参耗时 / 关键节点 INFO / 异常 ERROR 带堆栈
- **check-standards 兜底没触发** → 本 skill 独立成可单独触发的 skill（不依赖 ai-dev-workflow 全流程），description 覆盖"写完代码/改完代码/提交前检查"等触发时机

## 安装

将本目录复制到你的 skills 目录：

```bash
# OpenCode / Codex
cp -r check-standards ~/.agents/skills/

# Claude Code
cp -r check-standards ~/.claude/skills/
```

> 建议与配套规范 skill 一起安装：`ai-dev-workflow`、`java-code-standards`、`comment-standards`、`database-standards`、`build-standards`、`test-standards`、`legacy-onboarding`。

## 触发方式

**方式 A：自然语言（推荐）**——代码写完/改完后对 Agent 说：

| 你想做什么 | 对 Agent 说 |
|---|---|
| 写完代码后兜底自检 | "代码写完了，跑一下规范自检" / "帮我核对一下代码规范" |
| 检查注释/日志是否齐全 | "检查一下注释和日志有没有加全" |
| 提交前检查 | "提交前过一遍关键规范核对" |
| 验收复核 | "对照规范核对报告复核" |

**方式 B：配合 ai-dev-workflow 流程**——`/check-standards <项目路径>`（ai-dev-workflow 5.1 收尾 / 5.3 验收时自动触发本 skill）。

## 核对范围

| 组 | 项 |
|---|---|
| HIGH（12 项） | 接口文档 / 日志框架 / SQL 在 XML / SQL 注释 / DDL 注释 / JSON 产物 / 事务 rollbackFor / SQL 注入 / UPDATE-DELETE 带 WHERE / 统一返回体 / 密码加密 / 分页上限 |
| 代码规范组 C1-C6 | 构造器注入 / 分层边界 / Entity 不暴露 / 异常处理 / 命名 / 公共组件复用 |
| 注释组 N1-N4 | 类注释 / 方法 Javadoc / 步骤注释+WHY / 禁翻译式 |
| INFO | 敏感信息进日志 / 集合命名 / 魔法值 |
| 场景化 | Job 防重入 / Listener 幂等 / 文件上传安全 / 写接口幂等 |

## 核心原则

1. **实际执行 grep/ast-grep，禁止凭记忆答 ✅**——每项附「文件:行号」证据
2. **未到位项（含 INFO，不分重要与否）统一提示用户确认是否补齐**——人确认后才动手改代码，不自动默默补
3. **先判模式**（标准 / 存量适配），选型敏感项按项目约束判定，非规范默认值一刀切

## 配套 skill 生态

| skill | 职责 |
|---|---|
| [ai-dev-workflow](https://github.com/soft6096/ai-dev-workflow) | 流程编排（需求→方案→任务→验收），5.1 收尾调用本 skill |
| [java-code-standards](https://github.com/soft6096/java-code-standards) | Java 代码规范（本 skill 核对项的规范出处） |
| [comment-standards](https://github.com/soft6096/comment-standards) | 注释规范（注释组 N1-N4 出处） |
| [database-standards](https://github.com/soft6096/database-standards) | SQL/表/索引规范 |
| [build-standards](https://github.com/soft6096/build-standards) | 构建/依赖规范 |
| [test-standards](https://github.com/soft6096/test-standards) | 测试规范 |
| [legacy-onboarding](https://github.com/soft6096/legacy-onboarding) | 存量项目接入 |

## License

MIT
