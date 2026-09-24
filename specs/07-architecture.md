# 07 软件架构设计说明书（Software Architecture Document）

- **输入**：`specs/02-requirements.md`（v1.1）、`specs/05-domain-model.md`（v1.1）、`specs/03-use-cases.md`（v1.0）、`specs/06-domain-class-diagram.puml`、`specs/constitution.md`
- **版本**：v1.0
- **日期**：2026-09-23
- **状态**：**草稿，待人工审查**（依 `constitution.md` 第四条：未经审查的产出不得作为下游输入）
- **编号约定**：`LAY-xx` 层职责 / `MOD-xx` 业务模块 / `SEC-xx` 权限 / `EXC-xx` 异常 / `TX-xx` 事务 / `RULE-xx` 规则归属 / `DP-xx` 设计模式 / `ASM-xx` 假设
- **约束**：本文件受 `constitution.md` 约束；术语与枚举严格遵循 `02` 附录 A 与 `01` 基线，不得改写；「不做」清单（`02` §2.3）中的概念**禁止**出现在架构设计中
- **留痕**：本次由 Agent 生成，依 NFR-007 / 第四条第 5 款须在 `19-ai-usage-log.md` 登记一条记录；本轮人类指令限定**仅修改本文件**，故登记条目**尚未写入**，须由人类补齐或另行授权后再补（见 §14 `ASM-07`）

---

## 1. 文档信息

| 项 | 内容 |
|---|---|
| 文档编号 | `07-architecture.md` |
| 上游基线 | `02-requirements.md`（v1.1，`FR-001`~`FR-023` / `BR-001`~`BR-032` / `NFR-001`~`NFR-009`）、`05-domain-model.md`（v1.1）、`03-use-cases.md`（v1.0，`UC-01`~`UC-24`）、`constitution.md`（v1.0） |
| 下游影响 | `08-package-diagram.puml`、`09-design-model.md`、`13-database-design.md`、`14-api-spec.md`、`15-test-plan.md`、`16-tasks.md`、`src/` 代码骨架 |
| 架构风格 | 四层分层架构 + MVC + 依赖倒置（Repository 接口在领域层） |
| 顶层包 | `com.example.library.{common, presentation, application, domain, infrastructure}`（**固定 5 个**，见 §5） |
| 同步要求 | 依第十一条第 4 款：§5 的包结构是 `08-package-diagram.puml` 的**唯一绘制依据**，二者须在同一轮变更中保持一致；`13-database-design.md` 的持久化对象须与 §5 的 `infrastructure.persistence` 对齐 |

---

## 2. 架构目标与约束来源

| 目标 | 依据 |
|---|---|
| 四层分工：表现层 / 应用层 / 领域层 / 基础设施层，依赖只能自上而下，禁反向依赖与跨层直连 | `constitution.md` 第六条、`NFR-003` 第 1 款 |
| MVC 落点清晰：Controller 只做路由、参数绑定、DTO 转换、HTTP 状态码 | `constitution.md` 第六条、`NFR-003` 第 2 款 |
| 领域层承载**全部**业务规则；引入 Repository（接口在领域层、实现在基础设施层） | `constitution.md` 第六条、`T2`、`NFR-003` 第 3 款 |
| 业务规则**禁止**写在 Controller，规则数值**禁止**硬编码（必须查配置表） | `constitution.md` 第七条、`BR-014`、`R-4` |
| 权限校验在**应用层统一入口**，越权返回 403 并写审计 | `constitution.md` 第八条、`NFR-004` 第 4 款 |
| 三层测试齐全，核心用例自动化，H2 内存库 + 种子数据 | `constitution.md` 第九条、`NFR-005` |
| Java + Spring Boot、H2 唯一方言、Vue 3 + Vite SPA、单机零配置 | `constitution.md` 第十二条、`NFR-002`、`NFR-006`、`R-13` |

---

## 3. 架构总览

### 3.1 分层与 MVC 落点

```
┌──────────────────────────────────────────────────────────────┐
│ V：View —— 前端 SPA（Vue 3 + Vite），禁 Thymeleaf 服务端渲染   │
│    路由 / 组件 / 表单 / 列表；仅按接口返回渲染，不做权限判定    │
└───────────────┬──────────────────────────────────────────────┘
                │ HTTPS + JSON（REST 契约见 14-api-spec.md）
┌───────────────▼──────────────────────────────────────────────┐
│ LAY-01 Presentation Layer（Controller Layer）                 │
│   路由映射 / 参数绑定 / 格式校验 / DTO 转换 / HTTP 状态码       │
│   C = Controller；M 的对外投影 = 应用层 DTO                    │
└───────────────┬──────────────────────────────────────────────┘
                │ 依赖（单向、自上而下）
┌───────────────▼──────────────────────────────────────────────┐
│ LAY-02 Application Service Layer                              │
│   用例编排 / 事务边界 / 权限校验入口 / 审计编排 / DTO 映射       │
└───────────────┬──────────────────────────────────────────────┘
                │ 依赖
┌───────────────▼──────────────────────────────────────────────┐
│ LAY-03 Domain Layer（M：Model 的核心）                         │
│   实体 / 值对象 / 枚举 / 领域服务 / 策略对象 / Repository 接口   │
│   承载 BR-001~BR-032 全部业务规则与不变量                       │
└───────────────▲──────────────────────────────────────────────┘
                │ 实现接口（依赖倒置 DIP）
┌───────────────┴──────────────────────────────────────────────┐
│ LAY-04 Infrastructure / Persistence Layer                      │
│   Repository 实现 / JPA 映射 / H2 数据源 / BCrypt / 会话存储     │
│   Spring @Scheduled 调度 / 种子数据 / 策略供应者与审计写入实现    │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ LAY-05 Test Layer（src/test/java，镜像主源码包结构）            │
│   领域单元测试 / 集成测试（H2 mem + 种子） / API 接口测试         │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 依赖规则（硬性）

1. **允许**的依赖方向：`presentation → application → domain ← infrastructure`（基础设施层实现领域层接口，属于依赖倒置，不算反向依赖）。
2. **禁止**：`domain` 依赖 `application` / `presentation` / `infrastructure`；`presentation` 直接依赖 `domain` 或 `infrastructure`（Controller 不得直接调用领域服务或 Repository）；`infrastructure` 被 `presentation` 直连。
3. **禁止跨层直连**：表现层只能调用**应用服务接口**，不得跨过应用层直接组合两个领域服务来编排用例。
4. **DTO 归属应用层**（`application.<module>.dto`），表现层依赖应用层的 DTO 类型；领域层**不依赖任何 DTO**（DTO ↔ 领域对象的转换由应用层 Assembler 完成）。
5. 领域层**禁止**出现 Spring / JPA / HTTP / JSON 注解与依赖（保持可纯单元测试）。
6. 校验手段：以包结构与代码评审为准；如需编译期强制，可另议引入 `ArchUnit`（**第三方依赖，须先取得人类同意**，第十二条第 4 款），见 §14 `ASM-05`。

### 3.3 MVC 落点对照

| MVC 角色 | 本项目落点 | 禁止 |
|---|---|---|
| **View（V）** | Vue 3 + Vite SPA 页面与组件；后端以 JSON 响应作为视图数据 | 禁止 Thymeleaf / JSP 等服务端渲染（`R-13`） |
| **Controller（C）** | `presentation.<module>.*Controller` + `presentation.advice.GlobalExceptionHandler` | 禁止业务判断、禁止直接访问 Repository、禁止事务注解 |
| **Model（M）** | `domain.**`（实体 / 值对象 / 领域服务 / 策略）+ 应用层 DTO 作为对外投影 | 禁止把持久化 PO 当作模型泄漏到表现层 |

---

## 4. 层职责详解

### 4.1 LAY-01 表现层（Presentation / Controller Layer）

| 项 | 内容 |
|---|---|
| 职责 | ① 路由与端点映射（URI / Method，与 `14-api-spec.md` 契约逐条对应）；② 请求参数绑定与**格式**校验（非空、长度、枚举合法性、分页参数）；③ 从会话解析操作者并封装为 `ActorContext`；④ 调用**本模块**应用服务；⑤ 将应用服务返回的 DTO 转为响应 JSON 与 HTTP 状态码；⑥ 全局异常转译（`@RestControllerAdvice`） |
| 组成 | `presentation.<module>.<Xxx>Controller`、`presentation.advice.GlobalExceptionHandler`、`presentation.common.ApiResponse` / `PageResponse` |
| 事务 | **无**（禁止 `@Transactional`） |
| 权限 | 只做「是否已登录」的**入口**判定（未登录 → 401）；**不做**角色与业务权限判定（第七条、第八条第 3 款） |
| 硬禁止 | 任何业务规则判断（借阅前置校验、可借数量与期限取值、罚款费率与计提、预约触发与队列、副本状态流转、角色归属）；规则数值字面量；try-catch 吞掉业务异常；跨模块编排 |

> 违例判定（第七条第 4 款）：任一业务规则若能在 Controller 源码中被定位到（格式校验与 DTO 映射除外），即判定违约，必须下沉至领域层并补单元测试。

### 4.2 LAY-02 应用层（Application Service Layer）

| 项 | 内容 |
|---|---|
| 职责 | ① **用例编排**：按 `UC-01`~`UC-24` 组织领域服务 / 实体方法的调用顺序；② **事务边界**：公开用例方法标注 `@Transactional`（见 §9）；③ **权限校验统一入口**：`AccessGuard.require(actor, action)`；④ **审计编排**：成功 / 失败两条路径调用 `AuditAppender`；⑤ **DTO ↔ 领域对象转换**（Assembler）；⑥ 分页与查询条件组装（读模型，只读事务）；⑦ 定时任务的**触发入口**（Runner）；⑧ 会话与认证流程（`AuthApplicationService` / `SessionService`） |
| 组成 | `application.<module>.<Xxx>ApplicationService`、`application.<module>.dto.*`、`application.<module>.assembler.*`、`application.support.{AccessGuard, AuditAppender, ActorContext, ClockProvider}` |
| 事务 | **唯一**开启事务的层（`@Transactional` / `@Transactional(readOnly = true)`） |
| 硬禁止 | 承载或复制业务规则（不得出现 `if (读者类型 == 本科生) max = 5` 之类分支）；规则数值字面量；把领域异常改写为泛化错误而丢失「具体失败原因」（`FR-009` 验收标准第 2 条） |

### 4.3 LAY-03 领域层（Domain Layer）

| 项 | 内容 |
|---|---|
| 职责 | 承载 `BR-001`~`BR-032` 的**全部**业务规则与聚合不变式（`05` §4、§6、§7）：实体状态迁移、跨聚合协调（领域服务）、策略计算（`BorrowPolicy` / `FineRule`）、领域异常定义、Repository 与策略供应者**接口**定义 |
| 组成 | `domain.<module>.*` 实体 / 值对象 / 枚举 / 领域服务 / 策略；`domain.<module>.<Xxx>Repository`（**接口**）；`domain.exception.*`；`BorrowPolicyProvider` / `FineRuleProvider` / `AuditRecorder`（**接口**） |
| 事务 | **无**（不开事务；一致性由应用层事务边界保证） |
| 依赖 | 仅依赖 JDK 与 `common`（异常基类、值对象基元）；禁 Spring / JPA / HTTP |
| 硬禁止 | 按 `ReaderType` / `ItemType` 写 `switch` / `if` 费率或期限分支（`BR-014`）；`instanceof` 按读者子类分支（`05` §3.2 D-1）；规则数值常量 |

### 4.4 LAY-04 基础设施 / 持久层（Infrastructure / Persistence Layer）

| 项 | 内容 |
|---|---|
| 职责 | ① Repository **实现**（Spring Data JPA + H2，唯一方言）；② 持久化对象 PO 与领域对象的双向映射（`XxxMapper`，默认手写，避免新增第三方依赖）；③ `BorrowPolicyProviderImpl` / `FineRuleProviderImpl`（读 `borrowing_rule` / `fine_rule` 配置表）；④ `AuditRecorderImpl`（独立事务只追加写入）；⑤ BCrypt 口令哈希、会话存储与 30 分钟超时；⑥ Spring `@Scheduled` 调度器（仅负责**触发**应用层的 Runner）；⑦ H2 数据源配置（文件 / 内存双模式，禁混用）、种子数据初始化（`TBD-008`）；⑧ Jackson / OpenAPI 配置 |
| 组成 | `infrastructure.persistence.*`、`infrastructure.policy.*`、`infrastructure.security.*`、`infrastructure.scheduling.*`、`infrastructure.audit.*`、`infrastructure.config.*`、`infrastructure.seed.*` |
| 事务 | 不开业务事务；**例外**：审计写入使用 `REQUIRES_NEW`（见 §9.4） |
| 硬禁止 | 承载业务规则；MySQL / PostgreSQL 兼容层或双方言适配（`NFR-002` 第 5 款）；把 PO 直接返回给表现层 |

### 4.5 LAY-05 测试层（Test Layer）

| 项 | 内容 |
|---|---|
| 职责 | ① **领域单元测试**：Policy / 领域服务 / 实体状态机与不变式（纯 JUnit，无 Spring 上下文、无数据库）；② **集成测试**：Repository 与 H2 交互、PO↔领域映射、事务回滚与并发、定时任务幂等；③ **API 接口测试**：`MockMvc` 端到端 HTTP 行为与状态码（`14-api-spec.md` 契约） |
| 组成 | `src/test/java/com/example/library/domain/**`、`.../integration/**`、`.../api/**`；`src/test/resources/application-test.yml`、`seed-test.sql` |
| 数据 | **必须**使用 H2 内存库 `jdbc:h2:mem:library` + 种子数据，零污染、可重复运行；**禁止**与运行 / 演示的文件模式 `jdbc:h2:file:./data/library` 混用（`R-8`、`NFR-002` 第 3 款） |
| 门禁 | 五个核心场景**必须**有自动化测试：**借书、还书、预约、超期罚款、权限失败**；全部通过方可判定交付（`constitution.md` 第九条、`NFR-005`）；定时任务必须可被测试单独触发（注入 `Clock` / 直接调用 Runner） |
| 硬禁止 | 为让测试通过而修改期望、跳过或禁用用例（第九条第 5 款）；测试命名与断言出现「损坏 / 续借 / 挂失」等已排除概念（第九条第 7 款） |

---

## 5. 包结构与目录

### 5.1 顶层包（固定，禁止新增）

`com.example.library` 下**顶层包固定为 5 个**，与 `08-package-diagram.puml` 一一对应；新增顶层包须走 `constitution.md` 第五条变更流程：

| 顶层包 | 层 | 说明 |
|---|---|---|
| `common` | 共享内核 | 异常基类、错误码、响应封装、通用技术工具（**无业务语义**）。任一层均可依赖 |
| `presentation` | LAY-01 | 表现层 |
| `application` | LAY-02 | 应用层 |
| `domain` | LAY-03 | 领域层 |
| `infrastructure` | LAY-04 | 基础设施层 |

**二级包** = 7 个业务模块（`reader` / `card` / `catalog` / `circulation` / `reservation` / `fine` / `admin`）+ 横切子包（`support` / `advice` / `exception` / `persistence` 等）。

### 5.2 源码目录树

```
library-management-ai/
├── src/main/java/com/example/library/
│   ├── common/
│   │   ├── exception/          LibraryException（基类）、ErrorCode
│   │   ├── result/             ApiResponse、PageResponse（表现层契约封装）
│   │   └── util/               ClockProvider 抽象、分页参数（禁放业务常量）
│   ├── presentation/
│   │   ├── reader/             ReaderController（UC-01 注册、UC-05 查询本人）
│   │   ├── card/               BorrowCardController（UC-16 注销借阅证）
│   │   ├── catalog/            CatalogQueryController（UC-04）、CatalogMaintenanceController（UC-13/14/15）
│   │   ├── circulation/        CirculationController（UC-09/10/12）、LoanQueryController（UC-11）
│   │   ├── reservation/        ReservationController（UC-06/07/20）
│   │   ├── fine/               FineController（UC-08 缴纳、UC-19 减免）
│   │   ├── admin/              AuthController（UC-03）、AdminAccountController（UC-17）、
│   │   │                       RuleController（UC-18）、AuditLogController（只读）
│   │   ├── advice/             GlobalExceptionHandler（@RestControllerAdvice）
│   │   └── common/             ApiResponse 装配、分页参数绑定
│   ├── application/
│   │   ├── reader/             ReaderApplicationService + dto/ + assembler/
│   │   ├── card/               BorrowCardApplicationService + dto/
│   │   ├── catalog/            CatalogQueryApplicationService、CatalogMaintenanceApplicationService
│   │   ├── circulation/        BorrowApplicationService、ReturnApplicationService、
│   │   │                       CompensationApplicationService、LoanQueryApplicationService
│   │   ├── reservation/        ReservationApplicationService、ReservationQueueApplicationService、
│   │   │                       ReservationExpiryRunner（UC-22 触发入口）
│   │   ├── fine/               FinePaymentApplicationService、FineReductionApplicationService、
│   │   │                       FineAccrualRunner（UC-21 触发入口）、FineQueryApplicationService
│   │   ├── admin/              AuthApplicationService、SessionService（UC-24 会话超时）、
│   │   │                       AdminAccountApplicationService、RuleApplicationService、AuditQueryApplicationService
│   │   └── support/            AccessGuard（权限统一入口）、AuditAppender、ActorContext、ClockProvider
│   ├── domain/
│   │   ├── shared/             Money、DateRange、标识基元等公共值对象
│   │   ├── reader/             Reader、StudentReader、专科生/本科生/研究生/博士生 Reader、
│   │   │                       TeacherReader、ReaderType、ReaderStatus、ReaderRepository
│   │   ├── card/               BorrowCard、CardNumber、CardStatus（随 Reader 聚合持久化，H-4）
│   │   ├── catalog/            LibraryItem、Book、Magazine、Thesis、ItemType、Language、
│   │   │                       ItemCopy、CopyStatus、CopyNumber、ISBN、CatalogService、
│   │   │                       LibraryItemRepository、ItemCopyRepository
│   │   ├── circulation/        Loan、LoanStatus、LoanRepository、BorrowPolicy、BorrowingService、
│   │   │                       ReturningService、CompensationService、BorrowPolicyProvider（接口）
│   │   ├── reservation/        Reservation、ReservationStatus、ReservationRepository、ReservationService
│   │   ├── fine/               FineRecord、FineType、FineStatus、FineRule、FineAccrualService、
│   │   │                       FineRecordRepository、FineRuleRepository、FineRuleProvider（接口）
│   │   ├── admin/              AdminAccount、Role、AdminStatus、Action、AdminAccountRepository/
│   │   │                       AuditLog、AuditEntry、ActionType、Result、AuditLogRepository、AuditRecorder（接口）
│   │   └── exception/          BorrowRejectedException、ReservationRejectedException、
│   │                           CopyStateIllegalException、FineRuleMissingException 等领域异常
│   └── infrastructure/
│       ├── persistence/        Jpa*Repository（接口实现）、*PO、*Mapper（PO ↔ 领域对象）
│       ├── policy/             BorrowPolicyProviderImpl、FineRuleProviderImpl（读配置表）
│       ├── security/           BCryptPasswordHasher、SessionStore（30 分钟滑动超时）
│       ├── scheduling/         SchedulingConfig（@Scheduled → 调用 application 的 Runner）
│       ├── audit/              AuditRecorderImpl（REQUIRES_NEW 只追加）
│       ├── config/             DataSourceConfig（H2 文件 / 内存）、Jackson、OpenAPI、CORS
│       └── seed/               SeedDataInitializer（初始系统管理员 TBD-008、规则 5+5 行、演示数据）
├── src/main/resources/
│   ├── application.yml（运行：jdbc:h2:file:./data/library）
│   └── data/（文件模式产物，**必须**加入 .gitignore，NFR-002 第 4 款）
├── src/test/java/com/example/library/
│   ├── domain/                 领域单元测试（无 Spring、无 DB）
│   ├── integration/            集成测试（@SpringBootTest + H2 mem）
│   └── api/                    API 接口测试（MockMvc，覆盖五大核心场景）
├── src/test/resources/         application-test.yml（jdbc:h2:mem:library）、seed-test.sql
└── web/                        前端：Vue 3 + Vite SPA（dev 代理 → 后端；构建产物由 Spring Boot 静态托管）
```

### 5.3 模块间依赖规则

1. **表现层**：每个 Controller 只调用**本模块**的应用服务；例外 —— 鉴权与登录走 `admin` 模块，书目检索（`catalog`）由前端分别调用，**禁止**在后端串联两个模块的 Controller。
2. **应用层**：可跨模块调用**其他模块的领域服务与 Repository**（向下依赖），但**禁止**重复实现他模块的规则（如应用层不得自行计算逾期金额）。
3. **领域层**：跨模块协作只经 **Repository 接口 / 领域服务接口**加载对象；对他模块实体的状态变更**必须**调用该模块的实体方法或领域服务，**禁止**直接改字段（例：借书改副本状态须走 `ItemCopy.markOnLoan()`，队首取书出队须走 `Reservation.fulfill()`）。
4. **基础设施层**：PO 与 Mapper 按模块划分，禁止跨模块共享可变 PO。

---

## 6. 业务模块（MOD-01 ~ MOD-07）

### 6.1 模块总表

| 编号 | 模块 | 领域核心类（`05` §4） | 主要应用服务 | 表现层 | 对应用例 |
|---|---|---|---|---|---|
| MOD-01 | `reader` | `Reader` + `StudentReader` + 4 个学生子类 + `TeacherReader`、`ReaderType` | `ReaderApplicationService`（注册发证、注销读者、查询本人） | `ReaderController` | UC-01、UC-05 |
| MOD-02 | `card` | `BorrowCard`、`CardNumber`、`CardStatus` | `BorrowCardApplicationService`（自动发证、注销） | `BorrowCardController` | UC-02（随 UC-01 原子触发）、UC-16 |
| MOD-03 | `catalog` | `LibraryItem` / `Book` / `Magazine` / `Thesis`、`ItemType`、`ItemCopy`、`CopyStatus`、`CatalogService` | `CatalogQueryApplicationService`、`CatalogMaintenanceApplicationService` | `CatalogQueryController`、`CatalogMaintenanceController` | UC-04、UC-13、UC-14、UC-15 |
| MOD-04 | `circulation` | `Loan`、`LoanStatus`、`BorrowPolicy`、`BorrowingService`、`ReturningService`、`CompensationService` | `BorrowApplicationService`、`ReturnApplicationService`、`CompensationApplicationService`、`LoanQueryApplicationService` | `CirculationController`、`LoanQueryController` | UC-09、UC-10、UC-11、UC-12 |
| MOD-05 | `reservation` | `Reservation`、`ReservationStatus`、`ReservationService` | `ReservationApplicationService`、`ReservationQueueApplicationService`、`ReservationExpiryRunner` | `ReservationController` | UC-06、UC-07、UC-20、UC-22 |
| MOD-06 | `fine` | `FineRecord`、`FineType`、`FineStatus`、`FineRule`、`FineAccrualService` | `FinePaymentApplicationService`、`FineReductionApplicationService`、`FineAccrualRunner`、`FineQueryApplicationService` | `FineController` | UC-08、UC-19、UC-21 |
| MOD-07 | `admin` | `AdminAccount`、`Role`、`Action`、`AuditLog`、`AuditRecorder` | `AuthApplicationService`、`SessionService`、`AdminAccountApplicationService`、`RuleApplicationService`、`AuditQueryApplicationService` | `AuthController`、`AdminAccountController`、`RuleController`、`AuditLogController` | UC-03、UC-17、UC-18、UC-23、UC-24 |

### 6.2 MOD-01 `reader`（读者）

- **职责**：读者账号的注册与认证、读者类型的持有与修正、读者注销（逻辑删除）、本人借阅与罚款信息的只读查询。
- **规则落点**：`Reader.register()`（口令强度 + 用户名唯一 + 类型合法，且与 `BorrowCard` **原子创建**，`BR-024` / `FR-001`）；`Reader.authenticate()`（BCrypt 比对）；`Reader.isBorrowAllowed()`；`Reader.hasUnpaidFine()` / `isBlacklisted()`（**派生状态**，不持久化，`H-1`）。
- **跨模块**：借书资格判定由 `circulation` 的 `BorrowingService` 读取；本模块**不**自行计算可借数量。
- **禁止**：子类中出现 `3 / 5 / 7 / 10 / 15` 等数值；`instanceof` 分支（`05` §3.2 D-1）。

### 6.3 MOD-02 `card`（借阅证）

- **职责**：借阅证的自动发放、状态（`正常 / 已注销`）与注销；提供姓名、系别、借阅证号。
- **规则落点**：`BorrowCard.issueFor(reader)`（注册时由系统自动发放，**无独立触发入口**，`R-11`）；`BorrowCard.cancel()`（系统管理员执行，逻辑删除，`TBD-011`）。
- **边界**：与 `Reader` 同属一个聚合（1:1、原子创建），**不设**独立的写 Repository，随 `Reader` 聚合一并持久化（`H-4`）。
- **禁止**：有效期字段、`挂失 / 冻结 / 补办` 状态与方法（`R-12`、`U1`）。

### 6.4 MOD-03 `catalog`（馆藏与书目）

- **职责**：图书标题与馆藏副本的维护、报废、书目检索；承载副本五态状态机。
- **规则落点**：`LibraryItem.addCopy/removeCopy/logicalDelete/hasAvailableCopy/hasActiveReservation/matches`；`ItemCopy.markOnLoan/holdForReservation/release/markLost/scrap/isBorrowable/isReservationExpired`（`BR-019`~`BR-023`）；`CatalogService`（系统管理员权限校验、新增副本初始 `在馆`、报废为显式动作）。
- **检索**：书名 / 作者 / ISBN 的**精确或前缀**匹配 + 分页，**不做**分词、相关性排序与跨字段模糊匹配（`FR-023`、`00` §8）。
- **禁止**：「损坏」状态；「遗失 → 在馆」找回；报废自动触发。

### 6.5 MOD-04 `circulation`（借还流通）

- **职责**：办理借书、还书、查询全部借阅记录、登记遗失赔偿；持有借阅规则策略 `BorrowPolicy`。
- **规则落点**：`BorrowPolicy.computeDueDate()` / `exceedsLimit()`（`BR-001`、`BR-004`）；`BorrowingService`（四项前置校验 + 副本可借校验 + 未取预约仅提示 + 队首预约者取书，`BR-003`~`BR-006`）；`ReturningService`（记录归还时间、按有效预约决定副本去向 `在馆` / `预约保留(+3 天)`，**不结算罚款**，`BR-009`、`BR-017`）；`CompensationService`（已借出 → 遗失 + 生成人工录入金额的赔偿款，**不自动报废**，`BR-022`）。
- **策略供应**：`BorrowPolicyProvider`（接口在领域层，实现在 `infrastructure.policy`），**唯一**取数入口，快照语义（`05` §4.12 D-2）。
- **禁止**：续借；还书路径中插入缴费步骤（`R-9`、`R-16`）。

### 6.6 MOD-05 `reservation`（预约）

- **职责**：预约发起与取消、FIFO 队列与队首判定、管理员队列调整、保留到期扫描与顺延。
- **规则落点**：`Reservation.create/enqueue/reorder/isHead/promote/expire/cancel/fulfill`（`BR-015`~`BR-018`）；`ReservationService`（队列协调 + 到期扫描**幂等**，`NFR-009`）。
- **边界**：队列 = 同标题下「在队」（`等待中` / `保留中`）预约的有序集合（`queuePosition` 升序 + `reservedAt` 升序），**不另建** `ReservationQueue` 实体（`05` §4.11）。
- **调度**：`ReservationExpiryRunner`（应用层）被 `infrastructure.scheduling` 触发，须可被测试单独调用。
- **禁止**：到馆通知（邮件 / 短信）；系统管理员预约（`TBD-006` 默认不可）。

### 6.7 MOD-06 `fine`（罚款）

- **职责**：超期罚款的每日计提、赔偿款登记（金额由 `circulation` 侧人工录入后在此落账）、读者自助缴纳、管理员减免、封顶与黑名单重算、罚款查询。
- **规则落点**：`FineRule.accrue()`（逾期天数 × 日费率，宽限期 0，`BR-007`、`BR-008`）；`FineAccrualService`（分页扫描未还且超期、**同自然日幂等**、封顶 50 元仅计超期款、`BR-010`/`BR-013`）；`FineRecord.pay()` / `reduce()`（`BR-012`、`FR-019`）。
- **策略供应**：`FineRuleProvider`，**实时读取**语义（每次计提读当前费率，下次计提即生效；已生成记录不追溯重算，`05` §4.13 D-3）；缺费率**失败并留痕**，禁按 0 或默认值计提（`UC-21 E1`）。
- **禁止**：归还时一次结算；赔偿款计入封顶；真实支付网关。

### 6.8 MOD-07 `admin`（管理员、权限与审计）

- **职责**：登录 / 登出与会话管理、管理员账号维护、借阅与罚款规则的运行时维护、审计日志的写入与只读查询、权限矩阵与角色判定。
- **规则落点**：`AdminAccount.create/authenticate/hasPermission/canBorrow/canReserve/deactivate`（`BR-028`、`BR-029`）；`AuditLog.record()`（只追加，`BR-030`）；`AuditRecorder`（统一写入入口，实现在 `infrastructure.audit`）。
- **规则维护**：`RuleApplicationService` 位于本模块（`FR-020` 的权限归属系统管理员），写入 `circulation.BorrowPolicyRepository` 与 `fine.FineRuleRepository` —— **领域对象仍在原模块**，避免规则口径出现第二处真相。
- **会话**：30 分钟无操作自动登出（`FR-004`、`UC-24`），属**应用层**，仅以 `SessionToken` 值对象在领域侧传递，不建模为领域实体（`05` §2.3）。
- **禁止**：删除或更新审计记录；删除最后一个系统管理员账号。

---

## 7. 权限控制策略（SEC-01 ~ SEC-08）

### 7.1 认证（SEC-01）

| 项 | 策略 |
|---|---|
| 方式 | 本地用户名 + 口令，**不接入**第三方 / 统一身份认证（`P3`、`R-17`） |
| 口令 | 8 位且含数字与字母；**BCrypt** 哈希，`password_hash` 按 60 字符设计（`BR-029`、`NFR-004`） |
| 落点 | `Reader.authenticate()` / `AdminAccount.authenticate()`（领域：比对语义）→ `infrastructure.security.BCryptPasswordHasher`（哈希实现）→ `application.admin.AuthApplicationService`（编排） |
| 失败 | 返回**统一**提示（不区分「用户不存在」与「口令错误」，防账号枚举）；**登录失败必须写审计**（`FR-003` 验收标准第 5 条） |

### 7.2 会话（SEC-02）

- `application.admin.SessionService` + `infrastructure.security.SessionStore`：登录后发放会话凭证，**30 分钟无操作自动登出**，有请求则重新计时（`FR-004`）。
- 阈值 30 分钟为**配置项**（`application.yml`），允许作为默认配置值，禁止作为不可改的硬编码字面量参与业务判断（`FR-004` 验收标准第 3 条）。
- 会话失效 / 未登录访问受保护接口 → **401**。

### 7.3 鉴权统一入口（SEC-03）

1. **唯一入口**：`application.support.AccessGuard.require(actor, action)`，在每个应用服务公开方法的**首行**调用（**禁止**只在 Controller 拦截，`constitution.md` 第八条第 3 款）。
2. **领域承载角色判定**：`AccessGuard` 委托 `AdminAccount.hasPermission(Action)`（领域层方法，`05` §7）—— 满足第七条「角色归属判断必须在领域层」。
3. **实现形式**：应用服务内**显式**调用（而非仅靠 AOP 注解），便于单元测试直接断言越权行为（`NFR-005`）。AOP 可作为补充，**不得**替代显式调用。
4. 前端**禁止**以隐藏按钮 / 隐藏菜单充当权限控制（第八条第 3 款）。

### 7.4 权限矩阵落地（SEC-04）

权限矩阵以 `01` 的 `P1`（含 `R-10` / `R-5` / `R-14` / `R-16` 补齐）为**唯一来源**，`02` §3.3 逐条对应：

| 操作（Action） | 读者 | 图书管理员 | 系统管理员 | 落点 |
|---|---|---|---|---|
| 借书 / 还书 | ✗ | ✓ | **✗（硬约束）** | `circulation`：`BorrowApplicationService` / `ReturnApplicationService` |
| 查询全部借阅记录 | ✗ | ✓ | ✗ | `circulation`：`LoanQueryApplicationService` |
| 查询本人借阅信息 | ✓（仅本人） | — | — | `reader`：`ReaderApplicationService` |
| 自助缴纳罚款 | ✓（仅本人） | ✗ | ✗ | `fine`：`FinePaymentApplicationService` |
| 预约 / 取消预约 | ✓（仅本人） | ✗ | **✗（`TBD-006` 默认）** | `reservation`：`ReservationApplicationService` |
| 维护图书标题 / 馆藏副本 / 设置副本报废 | ✗ | **✗（无任何维护权限）** | ✓ | `catalog`：`CatalogMaintenanceApplicationService` |
| 减免罚款（单笔） | ✗ | ✗ | ✓ | `fine`：`FineReductionApplicationService` |
| 调整预约队列 | ✗ | ✗ | ✓ | `reservation`：`ReservationQueueApplicationService` |
| 维护借阅规则 / 罚款规则 | ✗ | ✗ | ✓ | `admin`：`RuleApplicationService` |
| 维护管理员账号（增 / 删 / 改） | ✗ | **✗（改自己被拒）** | ✓ | `admin`：`AdminAccountApplicationService` |
| 登记遗失赔偿 | ✗ | ✓ | ✗ | `circulation`：`CompensationApplicationService` |
| 注销借阅证 | ✗ | ✗ | ✓ | `card`：`BorrowCardApplicationService` |
| 查询图书（只读） | ✓ | ✓ | ✓ | `catalog`：`CatalogQueryApplicationService` |
| 审计日志只读查询 | ✗ | ✗ | ✓ | `admin`：`AuditQueryApplicationService` |

> `Action` 枚举须与 `06-domain-class-diagram.puml` 保持一致；若实现需细分（如新增「修改规则」），**必须**同步修订 `06` 并留痕（第十一条第 4 款）。

### 7.5 数据级权限（SEC-05）

- 读者侧接口（`UC-05`、`UC-08`、`UC-07`）的 `readerId` **一律取自会话** `ActorContext`，**不接受**客户端传入的读者标识作为授权依据；传入他人标识 → **403**，且不返回他人数据（`FR-011` 验收标准第 3 条、`FR-018` 验收标准第 5 条）。
- 图书管理员查询全部记录时，可传入任意 `readerId` 作为**查询条件**（非越权）。

### 7.6 越权处理（SEC-06）

1. 越权访问 → **403** + `ErrorCode.FORBIDDEN`，并**写入审计日志**（`result = 失败`）（第八条第 3 款、NFR-004 第 4 款）。
2. 硬约束三条的实现点：
   - 系统管理员不可借书 / 不可预约：`AdminAccount.canBorrow()` / `canReserve()` 恒 `false`，在 `BorrowingService` / `ReservationService` 入口前置拦截（`BR-028`、`H-4`）。
   - 图书管理员不能修改自己的账号信息：`AdminAccountApplicationService.update()` 校验「操作者 == 被改账号」时拒绝（`FR-022` 验收标准第 3 条）。
   - 图书管理员无任何馆藏维护权限：权限矩阵不含该 `Action`。

### 7.7 审计（SEC-07）

- 写入入口：`AuditRecorder`（领域接口）→ `infrastructure.audit.AuditRecorderImpl`，**只追加**，业务代码**禁止**出现 update / delete 调用（`BR-030`）。
- 覆盖操作：`FR-005` 验收标准第 1 条列举的全部写操作 + 登录失败；预约保留的每次顺延 / 取消必落审计（`FR-015` 验收标准第 7 条）。
- **不落审计**：注册与自动发证、只读查询（含 `UC-04`）、定时计提（仅留任务执行摘要，`H-1`）。
- 审计写入使用**独立事务**（`REQUIRES_NEW`），保证业务回滚后审计仍留存（见 §9.4）。

### 7.8 权限相关禁止项（SEC-08）

禁止依赖前端隐藏元素；禁止在 Controller 内做角色判断后「提前返回成功」；禁止把越权降级为「返回空列表」（必须 403 + 审计）。

---

## 8. 异常处理策略（EXC-01 ~ EXC-08）

### 8.1 异常层次（EXC-01）

```
RuntimeException
└── LibraryException（common.exception，携带 ErrorCode + 具体原因）
    ├── DomainException（domain.exception —— 业务规则拒绝）
    │   ├── BorrowRejectedException         原因：超期未还 / 未缴罚款 / 达上限 / 已借同标题（FR-009 验收 2）
    │   ├── ReservationRejectedException    原因：存在在馆副本 / 重复预约 / 已借同标题 / 超上限 3
    │   ├── CopyStateIllegalException       副本状态不允许该操作（BR-021）
    │   ├── FineRuleMissingException        缺费率即失败，禁按 0 计提（UC-21 E1）
    │   ├── InvalidAmountException          减免超未缴余额 / 金额为负（FR-019）
    │   └── RuleValidationException         规则数值非法（FR-020）
    ├── UnauthenticatedException（401）     未登录 / 会话超时
    ├── AccessDeniedException（403）        越权（含读者访问他人数据）
    ├── ValidationException（400）          参数格式校验失败（表现层绑定 + 应用层 DTO 校验）
    ├── NotFoundException（404）            标题 / 副本 / 读者 / 借阅记录不存在
    ├── ConcurrencyException（409）         同一副本并发借出、乐观锁冲突（NFR-009 第 3 条）
    └── InfrastructureException（500）      持久化 / 外部资源异常（不向客户端暴露细节）
```

### 8.2 定义与抛出位置（EXC-02）

| 异常 | 定义位置 | 抛出位置 |
|---|---|---|
| `DomainException` 及其子类 | `domain.exception` | 领域实体方法 / 领域服务（**唯一**抛出点） |
| `AccessDeniedException` / `UnauthenticatedException` | `application.support`（或 `common.exception`） | `AccessGuard` / `SessionService`（应用层） |
| `ValidationException` | `common.exception` | 表现层参数绑定、应用层 DTO 校验 |
| `ConcurrencyException` | `common.exception` | 基础设施层乐观锁 / 唯一约束冲突，由应用层转译 |
| `InfrastructureException` | `common.exception` | `infrastructure`，不向客户端暴露堆栈与 SQL |

### 8.3 转译与响应（EXC-03）

1. **统一转译**：`presentation.advice.GlobalExceptionHandler`（`@RestControllerAdvice`）将异常映射为 HTTP 状态码 + `ApiResponse{code, message, data}`，与 `14-api-spec.md` 契约一致。
2. 映射：未登录 401 / 越权 403 / 参数 400 / 不存在 404 / 并发冲突 409 / 业务规则拒绝 **422（或 409，以 `14-api-spec.md` 为准）且必须携带具体原因码** / 其余 500。
3. **禁止** Controller 内散落 `try-catch` 吞掉业务异常；**禁止**把「具体失败原因」泛化为「操作失败」（`FR-009` 验收标准第 2 条）。
4. **禁止**向客户端返回堆栈、SQL、内部类名。

### 8.4 异常与事务（EXC-04）

- 全部异常继承 `RuntimeException`，`@Transactional` 默认回滚；禁止使用受检异常绕开回滚。
- 应用服务在 `catch` 中写**失败审计**后**原样重抛**（不改写、不吞掉），由全局处理器负责转译。

### 8.5 定时任务中的异常（EXC-05）

- 罚款计提（`UC-21`）：按借阅记录**逐条**处理，单条异常记录审计与任务日志后**继续**本批；**缺 `FineRule` 视为任务失败并留痕**，禁止回退默认费率或按 0 计提（`UC-21 E1`）。
- 保留到期扫描（`UC-22`）：单条失败不影响其余顺延；整体执行结果可追溯（审计 + 任务日志）。

### 8.6 禁止项（EXC-06 ~ EXC-08）

- 禁止以异常替代正常业务分支（除规则拒绝外，如「无有效预约」应返回正常结果而非抛异常）。
- 禁止在领域层抛出 HTTP 语义异常（401 / 403 / 404 属应用层与表现层语义）。
- 禁止「捕获后静默成功」—— 审计与日志必须留痕（`BR-030`）。

---

## 9. 事务边界（TX-01 ~ TX-08）

### 9.1 总则（TX-01）

1. **事务边界 = 应用层应用服务的公开用例方法**（`@Transactional`）；领域层、基础设施层**不开**业务事务；表现层**禁止**事务注解。
2. 只读查询使用 `@Transactional(readOnly = true)`，且**必须分页**（`NFR-001` 第 2 款）。
3. 隔离级别：H2 默认 `READ_COMMITTED`，不额外升级；跨聚合一致性由**事务边界 + 行级约束**保证。

### 9.2 必须原子化的用例（TX-02）

| 用例 | 事务内的原子操作 | 依据 |
|---|---|---|
| UC-01 + UC-02 注册发证 | 创建 `Reader` + 创建 `BorrowCard`（**同一事务**，禁止「有账号无借阅证」） | `BR-024`、`FR-001` 验收 2 |
| UC-09 借书 | 生成 `Loan`（含 `dueDate` 快照） + 副本 `在馆\|预约保留 → 已借出` + 队首预约 `fulfill()` 出队 | `BR-021`、`NFR-009` 第 2 款 |
| UC-10 还书 | `Loan.return()` 记录归还时间 + 副本 → `在馆` 或 `预约保留(+3 天)` | `BR-017`、`FR-010` |
| UC-12 登记遗失赔偿 | 副本 `已借出 → 遗失` + 生成 `类型 = 赔偿` 的未缴款项 | `BR-022` |
| UC-06 预约入队 | 校验（无在馆副本 / 未借同标题 / 无重复 / 上限 3） + FIFO 入队（依赖唯一约束防并发重复） | `BR-015`、`BR-018` |
| UC-20 队列调整 | 同标题预约集合的 `queuePosition` 重排（行级锁 / 乐观锁） | `FR-016`、并发风险 `05` 附录 A |
| UC-08 缴纳 / UC-19 减免 | `FineRecord` 余额与状态更新 + 未缴累计额与黑名单重算 | `BR-010`、`BR-012` |
| UC-13 / UC-14 / UC-15 馆藏维护 | 标题 / 副本状态变更 + 前置校验（无在借副本、无有效预约） | `FR-006`、`FR-007`、`FR-008` |
| UC-18 规则维护 | `BorrowPolicy` / `FineRule` 更新 + 审计 | `FR-020` |

### 9.3 定时任务的事务（TX-03）

- `FineAccrualRunner` / `ReservationExpiryRunner`：**分页批次**，**每批一个事务**，批间提交；禁止单事务覆盖全量扫描（避免长事务与锁升级）。
- **幂等**：重复执行不产生重复罚款或重复顺延（同自然日幂等计提、到期扫描幂等），`NFR-009` 第 1 款。
- 每批提交后写任务执行摘要；失败按 §8.5 处理。

### 9.4 审计写入的独立事务（TX-04）

审计写入使用 `Propagation.REQUIRES_NEW`（`infrastructure.audit.AuditRecorderImpl`）：业务事务回滚时，审计记录**仍然留存**，从而可追溯失败操作（`FR-005`、`BR-030`）。审计写入失败**不得**阻断业务主流程，须记录任务日志并告警。

### 9.5 并发控制（TX-05）

| 场景 | 手段 |
|---|---|
| 同一副本并发借出（NFR-009 第 3 条） | 副本状态**条件更新**（`UPDATE ... WHERE id=? AND status='在馆'`）+ `@Version` 乐观锁；影响行数 0 → `ConcurrencyException`（409） |
| 并发重复预约 | `(reader_id, item_id)` + 在队状态**唯一约束**（`05` §4.11 第 6 条） |
| 队列调整与到期顺延并发 | 同标题预约行级锁 / 乐观锁，重试一次后失败即报错 |
| 规则并发修改 | `@Version` 乐观锁 |

### 9.6 事务禁止项（TX-06 ~ TX-08）

- 禁止在领域层 / 表现层开启事务；禁止在事务内调用外部资源（本项目无外部资源，**禁止**任何支付网关调用，`00` §8）。
- 禁止「一个用例多个事务」造成部分提交（用例级原子性见 TX-02）。
- 禁止在还书事务中插入缴费步骤（缴纳与归还解耦，`R-9`、`R-16`、第十条第 2 款）。

---

## 10. 业务规则归属（RULE-01 ~ RULE-08）

### 10.1 规则 → 类对照（承接 `05` §7）

| 编号 | 规则 | 承载类（**唯一**落点） | 所在模块 |
|---|---|---|---|
| RULE-01 | BR-001 / BR-002 可借数量与借阅期限、应还日期 | `BorrowPolicy.computeDueDate()`、`BorrowPolicy.exceedsLimit()`（经 `BorrowPolicyProvider`） | `circulation` |
| RULE-02 | BR-003 / BR-004 / BR-005 / BR-006 借阅前置校验、同标题限 1、未取预约仅提示、不续借 | `BorrowingService` + `Loan.belongsToSameTitleAs()` + `BorrowCard.isActive()` | `circulation` / `card` |
| RULE-03 | BR-007 / BR-008 日费率与金额计算、不区分读者类型 | `FineRule.accrue()`（经 `FineRuleProvider`） + `Loan.overdueDays()` | `fine` / `circulation` |
| RULE-04 | BR-009 / BR-010 / BR-011 / BR-013 每日计提、50 元封顶、黑名单、赔偿不计封顶 | `FineAccrualService` + `FineRecord.unpaidAmount()` | `fine` |
| RULE-05 | BR-012 / BR-032 缴纳 / 减免结清、超期与赔偿复用结清路径 | `FineRecord.pay()` / `reduce()` | `fine` |
| RULE-06 | BR-015 ~ BR-018 预约触发、FIFO、保留 3 天与顺延、上限 3 | `ReservationService` + `Reservation`（入队 / 队首 / 顺延 / 取消） + `LibraryItem.hasAvailableCopy()` | `reservation` / `catalog` |
| RULE-07 | BR-019 ~ BR-023 副本五态迁移、报废、遗失赔偿 | `ItemCopy` 状态方法 + `CatalogService` / `CompensationService` | `catalog` / `circulation` |
| RULE-08 | BR-024 ~ BR-027 注册即发证、两态、逻辑删除 | `Reader.register()` / `deactivate()` + `BorrowCard.issueFor()` / `cancel()` | `reader` / `card` |
| RULE-09 | BR-028 权限矩阵与角色归属 | `AdminAccount.hasPermission()`（领域）+ `AccessGuard`（应用层入口） | `admin` |
| RULE-10 | BR-029 / BR-030 认证与审计 | `AdminAccount.authenticate()` / `Reader.authenticate()` + `AuditLog.record()` / `AuditRecorder` | `admin` |
| RULE-11 | BR-031 借出物类型是标题属性 | `LibraryItem` + `Book` / `Magazine` / `Thesis` 的 `itemType()` | `catalog` |
| RULE-12 | BR-014 规则配置化、禁硬编码 | `BorrowPolicy` / `FineRule`（配置型聚合根）+ Provider 实现读配置表 | `circulation` / `fine` / `infrastructure.policy` |

### 10.2 明确「不归属」清单（RULE-13）

以下位置**禁止**出现任何业务规则判断与规则数值（第七条第 1、3 款）：

| 位置 | 禁止内容 |
|---|---|
| Controller / DTO / VO | 全部业务判断（格式校验与 DTO 映射除外） |
| 应用服务 | 规则分支与数值；只能调用领域方法并按结果编排 |
| SQL / 存储过程 | 规则计算（如按类型算罚金） |
| 常量类 / 枚举工具类 | `3 / 5 / 7 / 10 / 15`、`2 / 2.5 / 1 / 1.5 / 3`、`50`、`3 天` 等参与业务判断（**种子数据脚本除外**） |
| 前端 | 任何以「隐藏按钮」实现的权限控制；任何规则镜像计算 |
| 领域层 | 按 `ReaderType` / `ItemType` 的 `switch` / `if` 费率或期限分支、`instanceof` 子类分支 |

> **「规则数值须配置化」的适用边界（2026-09-24 人类裁决，选项 A；见 `17` §12 裁决 2）**
>
> 1. **入配置表**（第七条第 3 款的适用对象）：**借阅规则** `borrow_policies`（读者类型 × 最大可借数量 × 借阅期限）与**罚款规则** `fine_rules`（借出物类型 × 日费率 × 宽限期），系统管理员运行时可改（`FR-020`）。
> 2. **不入配置表**（需求常量）：**50 元封顶**（`BR-010`）、**预约保留 3 天**（`BR-017`）、**单人有效预约上限 3**（`BR-018`）、**会话 30 分钟**（`FR-004`）。
> 3. 上述四类常量的**判定逻辑**集中在领域层单一入口（`FineAccrualService` 负责封顶与黑名单、`ReservationService` 负责保留期与上限、`SessionService` 负责会话超时），数值以 `application.yml` 配置项或**唯一**常量承载。
> 4. 本表（RULE-13）的禁止要求**不变**：四类常量同样**禁止**出现在 Controller / DTO、应用服务分支、SQL / 存储过程、前端；**禁止**散落硬编码与按类型分支。
> 5. **种子数据脚本**是唯一允许出现规则字面量的位置（含借期与费率）。

---

## 11. 设计模式（DP-01 ~ DP-10）

### 11.1 必须采用的模式

| 编号 | 模式 | 意图 | 落点（类 / 包） | 依据 |
|---|---|---|---|---|
| DP-01 | **MVC** | 分离界面、控制与模型 | V = `web/`（Vue 3 + Vite SPA）；C = `presentation.<module>.*Controller`；M = `domain.**` + 应用层 DTO 投影 | `00` §7、`NFR-003` 第 2 款、`R-13` |
| DP-02 | **分层架构（Layered）+ 依赖倒置** | 单向依赖、可替换实现 | `presentation → application → domain ← infrastructure`；领域层定义接口，基础设施层实现 | `constitution.md` 第六条、`NFR-003` 第 1、3 款 |
| DP-03 | **Repository** | 持久化领域对象，隔离存储技术 | 接口：`domain.<module>.<Xxx>Repository`；实现：`infrastructure.persistence.Jpa<Xxx>Repository` + `*PO` + `*Mapper` | 第六条第 3 款、`T2` |
| DP-04 | **Service Layer** | 组织用例流程、承载事务与权限入口 | 应用层：`application.<module>.<Xxx>ApplicationService`；领域层：`BorrowingService` / `ReturningService` / `ReservationService` / `FineAccrualService` / `CatalogService` / `CompensationService` | 第六条第 2 款、`05` §6 |
| DP-05 | **Strategy（借阅规则）** | 不同**读者类型**的借阅规则可替换 | 策略：`BorrowPolicy`（`computeDueDate()` / `exceedsLimit()`）；选择与注入：`BorrowPolicyProvider`（接口在领域层、实现在基础设施层）；运行时由 `borrowing_rule` 配置表驱动 | `BR-001`、`BR-014`、`05` §4.12 D-2 |
| DP-06 | **Strategy（罚款规则）** | 不同**借出物类型**的罚款规则可替换 | 策略：`FineRule`（`accrue()` / `isAccruable()`）；选择与注入：`FineRuleProvider`；运行时由 `fine_rule` 配置表驱动 | `BR-007`、`BR-014`、`05` §4.13 D-3 |
| DP-07 | **Factory** | 创建不同类型读者 / 馆藏资源，封装构造与不变式 | `ReaderFactory`（按 `ReaderType` 创建 `JuniorCollegeReader` / `UndergraduateReader` / `MasterReader` / `DoctoralReader` / `TeacherReader`）；`LibraryItemFactory`（按 `ItemType` 创建 `Book`(中文/外文) / `Magazine`(中文/外文) / `Thesis`）；`BorrowCardFactory`（生成 `CardNumber`，随 `Reader` 原子创建）；静态工厂 `Loan.create(...)`、`FineRecord.accrueFor(...)` / `createCompensation(...)` | `02` §3.2、`BR-031`、`BR-024`、`05` §4 |
| DP-08 | **DTO（+ Assembler / Mapper）** | 隔离接口层与领域层，避免领域对象外泄 | DTO：`application.<module>.dto.*Command` / `*Result`；转换：`application.<module>.assembler.*Assembler`（DTO ↔ 领域）；PO 映射：`infrastructure.persistence.*Mapper` | `NFR-003`、第七条（禁 DTO 内业务判断） |
| DP-09 | **Value Object / Aggregate（DDD 战术，非严格 DDD）** | 以不可变值对象表达度量与标识，以聚合界定一致性边界 | `Money`、`CardNumber`、`CopyNumber`、`ISBN`、`OverdueDays`、`DateRange`、`AuditEntry`；聚合：`Reader`(+`BorrowCard`)、`LibraryItem`(+`ItemCopy`)、`Loan`、`Reservation`、`FineRecord` | `05` §3.3、§5 |
| DP-10 | **Template / Runner（定时任务的稳定骨架）** | 定时任务「分页扫描 → 逐条处理 → 幂等 → 留痕」流程固定，具体动作由领域服务注入 | `application.fine.FineAccrualRunner`、`application.reservation.ReservationExpiryRunner`（被 `infrastructure.scheduling` 的 `@Scheduled` 触发；可被测试直接调用） | `FR-017`、`FR-015`、`NFR-005` 第 5 款 |

### 11.2 策略模式的硬性约束（DP-05 / DP-06）

1. **调用点唯一**：`BorrowPolicy` 只由 `BorrowingService` 经 `BorrowPolicyProvider.forReaderType()` 取得；`FineRule` 只由 `FineAccrualService` 经 `FineRuleProvider.forItemType()` 取得。禁止在其他任何位置自行读取 `maxCopies` / `loanPeriodDays` / `dailyRate` 计算。
2. **禁止分支代替查表**：禁止 `if (itemType == 中文图书) return 2;` 之类实现（第七条第 3 款）。
3. **快照 vs 实时**：`BorrowPolicy` 为**快照语义**（`dueDate` 借书时算定写入 `Loan`，规则变更不追溯已借记录）；`FineRule` 为**实时读取语义**（每次计提读当前费率，下次计提即生效，已生成 `FineRecord` 不追溯重算）。
4. **缺策略即失败**：`FineRuleProvider` 找不到对应 `ItemType` 时抛 `FineRuleMissingException` 并留痕，禁止回退默认值或按 0 计提（`UC-21 E1`）。
5. **可测试性**：策略为对象，测试可注入任意口径（如「0 天期限」「自定义费率」）而无需改代码或改库。

### 11.3 支撑性机制（非 GoF，但必须说明）

| 机制 | 说明 |
|---|---|
| **依赖注入（Spring IoC）** | 全部装配由 Spring 容器完成，**禁止**自造单例或静态服务定位器 |
| **Unit of Work** | 由 Spring `@Transactional` 实现（见 §9） |
| **乐观锁 / 条件更新** | `@Version` + `WHERE status = ?` 条件更新，保证并发借书唯一性（§9.5） |
| **只追加日志（Append-only）** | `AuditLog` 仅提供 `record()`；`REQUIRES_NEW` 独立事务（§7.7、§9.4） |

### 11.4 明确**不采用**的模式（避免过度设计）

| 模式 | 不采用的理由 |
|---|---|
| 完整 **State 模式**（为五态各建状态类） | `BR-020` 仅五态、迁移有限，以 `CopyStatus` 枚举 + `ItemCopy` 状态方法（`markOnLoan` / `holdForReservation` / `release` / `markLost` / `scrap`）表驱动实现，避免类爆炸 |
| **CQRS / 事件溯源 / 领域事件总线** | 规模 `D3`（读者 5000 / 副本 10000）无需读写分离；引入会显著增加复杂度，超出课程范围（第十二条第 4 款） |
| **Specification 规则链（独立规则对象）** | 借阅前置校验仅 4 项且由 `BorrowingService` 编排即可，暂不引入规则对象；如需扩展再走变更流程 |
| **外观 / 远程服务（Facade、Remote Facade）** | 单体部署、单机零配置，无需 |

---

## 12. 运行与部署视图

| 项 | 策略 |
|---|---|
| 形态 | 前后端分离：Spring Boot 单体（内嵌容器）+ Vue 3 + Vite SPA（`T1`、`R-13`） |
| 数据 | H2 **唯一**方言；运行 / 演示 / 验收 = `jdbc:h2:file:./data/library`；开发 / 测试 / CI = `jdbc:h2:mem:library`，**禁止混用**（`R-8`、`NFR-002`） |
| 部署 | 单机零配置可运行；`./data/` 加入 `.gitignore`（NFR-002 第 4 款） |
| 调度 | Spring `@Scheduled` 驱动罚款计提与保留到期扫描；**仅触发**应用层 Runner，业务规则不写在调度类中 |
| 契约 | REST API 契约见 `14-api-spec.md`（OpenAPI）；实现与契约不一致即违约（第六条第 5 款） |

---

## 13. 与 `constitution.md` 的一致性自检

| # | 检查项 | 本文件落点 | 判定 |
|---|---|---|---|
| 1 | 四层分工、依赖单向、无反向与跨层直连 | §3.1 / §3.2 / §5.3 | 符合 |
| 2 | MVC 落点清晰，Controller 无业务规则 | §3.3 / §4.1 | 符合 |
| 3 | 领域层承载业务规则；Repository 接口在领域层 | §4.3 / §10 / DP-03 | 符合 |
| 4 | 包结构为 `08-package-diagram.puml` 的唯一依据 | §5.1 / §5.2 | 符合（待 `08` 同轮绘制） |
| 5 | Controller 无业务规则；数值全部来自配置表 | §4.1 / §10.2 / DP-05 / DP-06 | 符合 |
| 6 | 权限矩阵逐条落地；越权 403 + 审计 | §7.4 / §7.6 / §7.7 | 符合 |
| 7 | 三层测试齐全；五核心场景自动化；H2 mem + 种子 | §4.5 | 符合 |
| 8 | 无「不做」清单项；还书路径无缴费步骤 | §6.5 / §9.6 | 符合 |
| 9 | 术语与枚举与基线逐字一致 | 全文沿用 `02` 附录 A | 符合 |

---

## 14. 假设与待确认项

| 编号 | 内容 | 影响 | 处理 |
|---|---|---|---|
| ASM-01 | 顶层包固定 5 个（含 `common` 共享内核） | `08-package-diagram.puml` | 待 `08` 绘制时逐包对齐；不一致以本文件为准并修正 `08` |
| ASM-02 | 根包名取 `com.example.library`（示例值） | `08`、`13`、`src/` | 若需改为课程要求的包名，须在本文件与 `08` 同轮修订 |
| ASM-03 | DTO 置于**应用层**而非表现层，以保证依赖方向自上而下 | `09`、`14` | 若审查要求 DTO 独立为契约包，须走第五条变更流程 |
| ASM-04 | PO ↔ 领域对象映射默认**手写 Mapper**，不引入 MapStruct | `infrastructure.persistence`、第十二条第 4 款 | 若同意引入 MapStruct，须登记理由与依赖说明 |
| ASM-05 | 分层依赖以代码评审保证；如需编译期强制须引入 `ArchUnit` | 构建依赖 | **须人类同意**（第十二条第 4 款），当前**不引入** |
| ASM-06 | `Action` 权限枚举须与 `06-domain-class-diagram.puml` 一致 | `06`、`09`、`src/` | 若需细分，须同轮修订 `06` 并留痕（第十一条第 4 款） |
| ASM-07 | 本次生成依 NFR-007 须登记 `19-ai-usage-log.md`，但本轮人类指令限定仅修改本文件 | `19-ai-usage-log.md` | 依第一条第 2 款**指出的冲突**：登记条目尚未写入，须由人类补齐或另行授权后补记；在完成登记前，本文件**不得**作为下游文档与代码的输入（第四条第 1 款） |
| ASM-08 | 前端目录取 `web/`（Vue 3 + Vite），构建产物由 Spring Boot 静态托管 | 部署形态 | 若仓库根目录另有约定，须同步本文件 §5.2 与 §12 |

**引用的上游待确认项**：`TBD-002`（读者类型确定方式 → `ReaderFactory` / `changeReaderType`）、`TBD-004`（计提时点与补提口径 → `FineAccrualRunner`）、`TBD-005`（扫描频率 → 与计提共用调度）、`TBD-006`（系统管理员能否预约 → `canReserve()`）、`TBD-007`（未缴赔偿款是否禁止借阅 → `BorrowingService` 前置校验）、`TBD-008`（初始系统管理员 → `infrastructure.seed`）、`TBD-009`（保留取书由谁办理 → 默认图书管理员，与借书同一入口）、`TBD-010`（借阅证号格式 → `CardNumber` / `BorrowCardFactory`）、`TBD-012`（逻辑删除 → `CatalogService`）。以上均**按 `02` 的建议默认推进**，确认后须同步本文件与下游。

---

## 附录 A 修订记录

| 版本 | 日期 | 修订内容 | 来源 |
|---|---|---|---|
| v1.0 | 2026-09-23 | 首版成文：① 架构总览（四层 + MVC、依赖规则、MVC 落点）；② 五层职责（含 Test Layer）；③ 包结构与目录树（顶层包固定 5 个、7 个业务模块）；④ 7 个业务模块职责与规则落点；⑤ 权限控制策略（认证 / 会话 / 统一鉴权入口 / 权限矩阵 / 数据级权限 / 越权与审计）；⑥ 异常处理策略（层次、抛出位置、转译、事务、定时任务）；⑦ 事务边界（原子用例、定时任务分批、审计独立事务、并发控制）；⑧ 业务规则归属与「不归属」清单；⑨ 设计模式（MVC / 分层 + DIP / Repository / Service Layer / Strategy ×2 / Factory / DTO / 值对象与聚合 / Runner）与「不采用」清单；⑩ 运行部署视图、`constitution.md` 一致性自检、假设与待确认项 | `02-requirements.md`（v1.1）、`05-domain-model.md`（v1.1）、`03-use-cases.md`（v1.0）、`06-domain-class-diagram.puml`、`constitution.md`（v1.0） |

> **留痕要求（NFR-007 / 第四条第 5 款）**：本次由 Agent 生成，须在 `19-ai-usage-log.md` 新增一条记录（使用工具、使用阶段、Prompt 摘要、修改文件、输出摘要、测试结果、Git 提交），「人工审查结果」由人类补齐；审查通过后方可作为 `08-package-diagram.puml`、`09-design-model.md`、`13`~`16` 及 `src/` 的输入。
