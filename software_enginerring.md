当前文档为软件工程规约文档

## 最高原则
- 以源代码(source code)为唯一真相源,其他任何文档、脚本、注视都不应该认为是事实真相

## 技术架构决策原则
- 技术决策应遵循技术最优和架构最优，你不是传统人类开发者，不应考虑开发成本，开发周期从而对技术决策做出妥协
- 不要过度设计导致软件维护困难

## 工程纪律
- 技术决策或技术架构决策要落实ADR存档，请参考skill `adr-writing`
- 创建git提交时，请参考skill `gen-commit-msg-zh`
- 编写的代码应当寻求 fast-fail，在出错位置就地崩溃，而不是捕获错误，也不是 fallback。

## 专项开发规约（按需必读）
开始设计、编码、排查问题或代码评审前，按任务涉及的技术加载对应规约；涉及多项时必须组合阅读，不以本入口代替详细规约。

| 涉及的工作 | 必读规约 |
| --- | --- |
| Java / Spring 开发，架构分层、DTO、API、事务、安全、测试与格式化 | [Java 开发规约](./java-development.md) |
| HTTP API 文档、OpenAPI / SpringDoc、Swagger UI / Knife4j、Tag、认证与文档验收 | [API 文档规约](./apidoc.md) |
| Redis、Redisson 分布式锁、一次性令牌状态与安全命名空间 | [Redis 规约](./redis.md) |
| PostgreSQL、JPA / JDBC、SQL、分页、索引、类型映射、Flyway 迁移 | [PostgreSQL 规约](./pgsql.md) |

上述文件位于 `~/.pi/steerings/`，仅保留通用技术约束，不携带具体业务域、专用接口或本机部署路径。当前项目的 `AGENTS.md` / `CLAUDE.md` 仍须先读；技术基线或约束冲突时先确认，不机械套用其他项目的版本、目录或运行环境。
