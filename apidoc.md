# API 文档规约

适用于 HTTP API 的文档编写与验收；Java / Spring 注解要求适用于采用 SpringDoc、swagger-annotations 的项目，Knife4j 排序要求仅在使用 Knife4j 时适用。

## 规则索引

按索引定位小节，只读命中的部分，不必通读全文。

| 触发场景 | 读哪一节 |
| --- | --- |
| 确认文档与实现一致、示例脱敏等通用约束 | 基本原则 |
| 写接口、参数、DTO 字段或错误码说明 | 接口与参数说明 |
| 新增或拆分 Controller，命名或合并 Tag | Tag 命名与合并 |
| 用 Knife4j 调整同 Tag 内接口顺序 | Knife4j 排序 |
| 写登录要求、Step-up 或认证错误语义 | 认证与错误文档 |
| 提交前核对 `/v3/api-docs` 等生成结果 | 提交前验收 |

## 基本原则

- API 文档与实际接口实现一致，不把文档描述当作运行行为已经正确的证据。
- 每个 HTTP 接口必须有完整文档；新增或修改接口时同步维护请求、响应、参数、错误和认证说明。
- 对外使用业务语义，不暴露外部集成协议细节、内部标识、密钥或签名信息。
- 示例不得包含真实密钥、访问令牌、生产连接串或个人环境配置。
- 接口采用项目统一的版本前缀，例如 `/api/v1/`；错误响应遵循 RFC 7807 `ProblemDetail`。

## 接口与参数说明

| 对象 | 文档要求 | Java / Spring 注解 |
| --- | --- | --- |
| Controller 分组 | 分组名称与说明 | `@Tag(name, description)` |
| HTTP 方法 | 简短标题与用途描述；summary 不超过 20 字 | `@Operation(summary, description)` |
| 请求 / 响应 DTO 字段 | 字段含义、示例、必填性 | `@Schema(description, example, requiredMode)` |
| 路径变量、查询参数、Header | 参数含义与示例 | `@Parameter(description, example)` |
| 非默认状态码与错误 | 状态码、含义及响应结构；错误 schema 引用 `ProblemDetail` | `@ApiResponses` / `@ApiResponse` |

- 请求 DTO 使用 `public record`，配合 `@Valid` 和 `@NotBlank`、`@NotNull` 等 `jakarta.validation` 约束。
- 文档中的必填性、字段类型和校验约束必须与实际请求校验一致，不能只修改 Schema 而不核对实现。
- 请求和响应字段均需说明，不得只补请求文档而忽略响应结构。

## Tag 命名与合并

- 使用简洁的中文领域名称，不附加无必要的外文前缀；仅在需要区分同名领域时保留限定词。
- 同一领域的接口即使分散在多个 Controller，也必须共用同一个 Tag。
- 共用 Tag 时，`name` 和 `description` 必须逐字完全一致；仅保持 name 相同不足以保证生成结果合并。
- 新增或拆分 Controller 后，检查同一领域全部 Controller 的 Tag 元数据，并核对实际生成的 OpenAPI 文档。
- 同一业务 Tag 在 OpenAPI 的 `tags` 中只能出现一次，不得产生同名分组或整组接口重复。

## Knife4j 排序

- 同一 Tag 内使用 `@ApiOperationSupport(order = N)` 控制接口顺序。
- 多个 Controller 共用 Tag 时，order 跨 Controller 统一递增，不在每个 Controller 内重新起算。
- 领域内部按接口使用先后排序；领域之间预留步长，如 100、200，便于后续插入。
- 必须核对实际页面顺序，不能只凭注解存在就认定排序正确。

## 认证与错误文档

- 需登录接口的认证要求应由项目统一 OpenAPI 配置维护，使用当前项目定义的 SecurityScheme，不复制其他项目的认证名称。
- 已由全局配置自动注入 SecurityRequirement 时，禁止在 Controller 方法上重复声明 `@SecurityRequirement(s)`。
- 已由全局配置注入的认证错误响应不重复维护，但必须检查实际生成结果完整。
- 认证要求、锁图标及错误状态必须与实际安全配置一致，避免公开接口误标为需登录，或受保护接口缺少认证说明。
- 敏感操作的 Step-up 文档应说明请求体中的 access token、一次性消费及用户归属校验要求，不暴露内部黑名单实现。
- 按统一安全契约说明失败语义：token 缺失、无效、过期、用途不匹配或重放返回 401；token 归属与当前用户不符返回 403。
- 错误响应使用统一 `ProblemDetail` schema，不为单个接口自行设计另一套错误结构。

## 提交前验收

新增、修改或拆分接口后，必须检查实际生成的 `/v3/api-docs` 及 Swagger UI / Knife4j 对应展示，至少确认：

- [ ] 标题与用途描述齐全，summary 不超过 20 字。
- [ ] 请求和响应字段说明、示例、必填性及校验约束一致。
- [ ] 路径变量、查询参数和 Header 的说明及示例齐全。
- [ ] 非默认状态码、错误描述和 `ProblemDetail` schema 正确。
- [ ] 登录要求、SecurityRequirement 和锁图标正确，没有重复声明。
- [ ] 同一领域 Tag 的 name / description 完全一致，生成结果中无重复 Tag 或接口分组。
- [ ] 使用 Knife4j 时，跨 Controller 的接口顺序符合统一编排。
- [ ] 对外字段和示例不泄漏内部集成细节或真实凭证。

不能只通过源码注解检查代替生成结果验收。未能启动应用或获取文档时，应明确说明尚未验证的项目，不宣称运行态验收通过。

Java 实现、校验、安全与测试要求另见 [Java 开发规约](./java-development.md)。
