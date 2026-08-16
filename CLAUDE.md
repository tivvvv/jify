# Jify

## 项目概述

Jify 是一个面向团队内部的轻量级 AI Agent 开发平台, 核心思路是从 Dify 中提取最刚需的功能模块, 做成一个人可独立交付的简化版.

### 产品定位

让 20-50 人的团队在本地环境中用上 AI Agent -- 能接模型, 能接工具, 能挂知识库, 一句话聊天就能干活.

### 做什么

- 多模型接入 (OpenAI 兼容, Claude, Ollama 等)
- Agent 智能体 (ReAct 模式, 支持工具调用和多步推理)
- MCP 协议接入, 复用社区 MCP Server 工具生态
- 知识库 & RAG (上传文档 → 分段 → 嵌入 → 检索增强生成)
- 可视化工作流 (后期用 YAML/JSON 配置代替拖拽画布, 先做线性流程)
- 权限 (Admin / User 两级)
- 请求日志 (简单的对话和工具调用记录)
- RESTful API (核心端点, 供业务系统集成)

### 不做什么 (一期)

- 可视化拖拽工作流画布 (一期用配置文件代替)
- 插件市场和多租户 SaaS
- 标注系统, 数据集管理, 模型微调
- 多渠道发布, 模型并排对比
- 复杂的 RBAC 权限体系
- 50+ 内置工具 (靠 MCP 生态覆盖, 自带仅 5 个核心工具)

### 技术栈

- 后端: Spring Boot 3.x + MyBatis-Plus + MySQL 8.x + Redis 7.x
- 前端: Vue 3 + TypeScript + Element Plus
- JDK: 17+
- 容器化: Docker + Docker Compose

### 部署与运维预期

- 部署方式: Docker Compose 一键启动
- JVM 内存上限: -Xmx512m, -XX:MaxMetaspaceSize=128m
- 用户量: 20-50 人同时在线, 单机部署
- QPS: 峰值 3-5, 瓶颈在 LLM 长连接而非并发
- 监控: 起步 Spring Boot Actuator + 日志, 后期接 Prometheus + Grafana
- 扩容: 一期单机, 架构上不堵死后续水平扩容的路

### 代码架构: Maven 多模块单体

```
com.jify
├── jify-app                # 启动模块 (Spring Boot 主类, 装配各模块)
├── jify-model              # 模型提供商管理
├── jify-agent              # Agent 配置与 ReAct 推理引擎 (配置 + 运行核心循环)
├── jify-conversation       # 会话管理与 SSE 流式 (执行分发入口)
├── jify-knowledge          # 知识库 & RAG (文档摄入, 嵌入, 检索)
├── jify-workflow           # 简版工作流 (YAML/JSON 配置驱动, LLM/工具步骤)
├── jify-mcp                # MCP 协议 Client, 工具接入
├── jify-auth               # 认证与权限 (Admin / User 两级)
├── jify-common             # 跨模块共享 (异常, 配置, 工具类)
└── jify-web                # 前端 (Vue 3 + TypeScript + Element Plus)
```

每个后端模块是一个 Maven 子模块 (artifactId 即模块名), 内部包含自己的 Controller, Service, Mapper, Entity, 包名对应 `com.jify.<module>`. jify-app 是唯一包含 Spring Boot main 方法的启动模块, 作为组合根装配所有业务模块; jify-web 是独立前端工程, 通过 RESTful API 与后端交互.

**模块间交互铁律:** 模块间只能通过目标模块的 Service 接口交互, 严禁跨模块直接 import Mapper 或 Entity. 这条规则现在靠代码纪律执行, 模块数超过 10 个后引入 Spring Modulith 做编译期隔离 -- 目录结构天然兼容, 只需在模块下新增 `internal` 包将内部类挪入即可.

**依赖方向 (A→B 表示 A 依赖 B, 每个模块 pom 只声明直接依赖, 其余靠 Maven 传递):** `app → conversation → {agent, workflow} → {model, mcp} → auth → common`, agent 额外依赖 knowledge, knowledge 依赖 model. conversation 只做会话管理与 SSE 分发; workflow 严禁反向依赖 conversation 或 agent. app 以 `@ComponentScan("com.jify")` 扫描全部模块组件.

## 代码组织规范

本规范定义每个后端模块的内部分层结构, 各层职责边界, 以及跨模块调用规则. 生成代码时直接照此执行, 无歧义.

### 模块内部分层结构

除 jify-app / jify-common / jify-web 外, 每个后端模块内部按以下包结构组织, 以 jify-agent 为例:

```
com.jify.agent
├── controller/          # REST 入口
├── service/             # Service 接口 (对外契约)
│   └── impl/            # Service 实现类
├── dto/                 # 数据传输对象
│   ├── request/         # 入参 DTO (带校验注解)
│   └── response/        # 出参 DTO
├── mapper/              # MyBatis-Plus Mapper 接口
├── entity/              # 数据库表实体 (模块私有)
├── config/              # 本模块配置类
├── constant/            # 本模块常量与枚举
└── internal/            # 预留, 未来 Modulith 隔离时把私有类移入
```

特殊模块:

- jify-app: 只含 main 主类 + @SpringBootApplication + @ComponentScan("com.jify") + @MapperScan("com.jify.**.mapper"), 不写任何业务 Controller/Service/Mapper.
- jify-common: 无 controller/entity/mapper (业务表), 只放 result/exception/util/config/constant 等跨模块通用件.
- jify-web: 独立前端工程, 不遵循此 Java 分层.

### 各层职责边界

**Controller 层**

- 只做 4 件事: 1) 声明路由, 统一前缀 /api/v1/<模块名>; 2) 用 @Valid 触发入参校验; 3) 调用一个 Service 方法; 4) 把结果包进 Result<T> 返回.
- 禁止: 写业务逻辑; 直接调用 Mapper; 直接返回 Entity; 用 try-catch 吞异常; 读 HttpServletRequest/Response 做业务.
- 每个方法只调用一次 Service, 多步编排一律下沉到 Service.

**Service 层**

- 承载全部业务逻辑与事务边界. @Transactional 只写在 Service 方法上, 禁止写在 Controller/Mapper.
- 对外暴露的 Service 一律接口 + 实现: 接口放 service 包, 实现放 service/impl 包, 命名 XxxServiceImpl.
- 模块内部可调用本模块 Service; 跨模块只可调用目标模块 Service 接口 (见跨模块调用规则).
- 禁止: 拼 SQL; 接收 HTTP 对象; 用 null 表示失败 (失败一律抛 BizException).
- Entity 与 DTO 的转换在 Service 层完成, 不散落到 Controller.

**Mapper 层**

- 每个 Mapper 继承 BaseMapper<XxxEntity>, 只对应一张表, 只做单表数据访问.
- 复杂查询写 XML (resources/mapper/), SQL 只做数据存取, 不掺业务判断.
- 禁止: 写业务逻辑; 一个 Mapper 跨多张表.

**Entity 层**

- 用 MyBatis-Plus 注解映射: @TableName (显式写表名, snake_case), @TableId, @TableField.
- 纯 POJO 字段映射, 无业务方法.
- 模块私有, 禁止被其他模块 import, 禁止直接返回给 Controller/前端.

**DTO 层 (request/response)**

- request: Controller 入参, 字段加 JSR-303 校验注解 (@NotBlank/@NotNull/@Min 等).
- response: Service 出参, 由 Entity 转换而来, 用 DTO 内静态方法 from(entity) 完成, 禁止字段拷贝散落在各处.
- 跨模块传递数据一律用 DTO, 不用 Entity.

### 统一返回与异常 (由 jify-common 提供)

- 所有接口返回 Result<T> { code, message, data }, 定义于 com.jify.common.result.
- 业务失败抛 BizException(code, message), 由 jify-common 的全局 @RestControllerAdvice 统一捕获转 Result, 各模块不重复写异常处理.
- 分页查询: 出参 PageResult<T> { total, list, pageNum, pageSize }, 入参继承 PageQuery.

### 跨模块调用规则

- 铁律: 模块 A 依赖模块 B 时, A 只可 import B 的 service 包接口 + dto 包对象. 禁止 import B 的 controller/mapper/entity/config/constant/internal.
- 注入方式: 只允许构造器注入 (Lombok @RequiredArgsConstructor), 禁止 @Autowired/@Resource 字段注入.
- 依赖方向严格按上文依赖链, 严禁反向依赖, 严禁 Service 层循环依赖.
- 跨模块 DTO 由被调用模块定义并拥有, 调用方 import 之. 只有跨模块通用件 (Result/PageResult/PageQuery/BizException/通用枚举) 放 jify-common, 禁止把业务 DTO 堆进 common.
- 跨模块调用默认在同一事务内 (单数据源, 事务传播 REQUIRED), 一期不做分布式事务与最终一致性.

### 命名规范

- Controller: XxxController
- Service: XxxService / XxxServiceImpl
- Mapper: XxxMapper
- Entity: XxxEntity
- DTO: XxxRequest / XxxResponse
- 表名: snake_case, 每个 Entity 显式 @TableName, 不依赖 MyBatis-Plus 自动猜表名

## 文案与提交规范

### 标点规范

所有文案一律使用英文半角标点, 禁止中文全角标点. 适用范围: 代码注释, commit message, 日志输出, 配置文件说明等.

禁止: `，` `。` `：` `（）` `、` `「」` 等全角标点
使用: `,` `.` `:` `()` `/` `"` 等半角标点

### Commit Message 规范

格式: `<type>: <summary>`

type 前缀: feat (新功能), fix (修复), refactor (重构), docs (文档), style (格式), test (测试), perf (性能), build (构建), chore (杂务), ci (持续集成)

冒号后跟一个空格, summary 用高度概括的语言总结本次提交, 简洁易懂明了.

示例:

```
feat: 添加模型提供商管理模块
fix: 修复 SSE 流式响应客户端断开后连接未释放的问题
refactor: 将 ReAct 循环从 conversation 迁移到 agent
docs: 补充模块依赖方向说明
```
