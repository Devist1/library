# 05 领域模型（Domain Model）

- **输入**：`specs/02-requirements.md`（v1.1）、`specs/03-use-cases.md`（v1.0）、`specs/constitution.md`
- **版本**：v1.1
- **日期**：2026-09-23
- **状态**：**草稿，待人工审查**（依 `constitution.md` 第四条：未经审查的产出不得作为下游输入）
- **本次修订**：按人工审查提出的 7 项问题逐项自检并补正（见**附录 A 评审项自检对照**）：① 领域类业务来源与反「数据库表机械翻译」自检（新增 §3.4）；② 读者类型与借出物类型的对照与建模决策（§3.2）；③ 借阅 / 罚款规则抽象为 `BorrowPolicy` / `FineRule` 的策略语义（§4.12、§4.13）；④ `Loan` 借出 / 归还状态表达（§4.10）；⑤ `Reservation` 排队机制（§4.11）
- **编号约定**：类名以英文标识（供 `06-domain-class-diagram.puml` 与代码直接使用）；业务规则引用 `BR-xxx`、需求引用 `FR-xxx`、用例引用 `UC-xx`
- **约束**：本文件受 `constitution.md` 约束；术语与枚举严格遵循 `02` 附录 A 与 `01` 基线，不得改写；「不做」清单（`02` §2.3）中的概念**禁止**出现在领域模型中
- **留痕**：本次由 Agent 生成，须在 `19-ai-usage-log.md` 登记一条记录（NFR-007 / 第四条第 5 款），「人工审查结果」由人类补齐

---

## 1. 文档信息

| 项 | 内容 |
|---|---|
| 文档编号 | `05-domain-model.md` |
| 上游基线 | `02-requirements.md`（v1.1，`FR-001`~`FR-023` / `BR-001`~`BR-032`）、`03-use-cases.md`（v1.0，`UC-01`~`UC-24`） |
| 下游影响 | `06-domain-class-diagram.puml`、`07-architecture.md`、`09-design-model.md`、`10/11/12-sequence-*.puml`、`13-database-design.md`、`16-tasks.md` |
| 类的总数 | 实体 12 个（含 3 个抽象基类 + 7 个具体子类）、领域服务 6 个、值对象 / 枚举 11 个 |
| 未决事项 | 见第 10 节（引用 `TBD-002`、`TBD-003`、`TBD-006`、`TBD-007`、`TBD-010`、`TBD-012`） |
| 同步要求 | 依第十一条第 4 款，`06-domain-class-diagram.puml` 须按本文件第 3、4 节同步重绘；本次仅修改本文件 |

---

## 2. 建模范围与依据

1. **边界**：本文件只描述**领域层**（NFR-003 第 1、3 款）。表现层（Controller / Vue）、应用层编排、基础设施（Repository 实现、H2、BCrypt、Spring `@Scheduled`）不在本文件定义职责，仅在必要时标注协作关系。
2. **规则归属**：全部业务判定（BR-001~BR-032）由领域层执行（第七条第 1 款）。应用服务只做事务边界、权限拦截、审计编排与 DTO 转换，**不承载业务规则**。
3. **会话不是领域概念**：会话与 30 分钟超时（FR-004 / BR-029）属应用层与基础设施关注点，本模型仅保留 `SessionToken` 值对象用于审计与权限传递，**不**建模为领域实体。
4. **不建模的概念**（`02` §2.3）：续借、借阅证挂失 / 冻结 / 补办、副本「损坏」状态、「遗失 → 在馆」找回、真实支付扣款与人工收银、催还通知、报表统计、条码扫描、复杂全文检索、多校区调拨、第三方登录。
5. **状态图替代**：依 `E2` / `R-19`，不绘制状态图 `.puml`，副本状态迁移以第 8 节文字状态迁移表表达。

---

## 3. 领域模型总览

### 3.1 分类总表（实体 / 值对象 / 服务与策略）

| 构造型 | 类 | 判据 |
|---|---|---|
| **实体（Entity，聚合根）** | `Reader`（抽象）、`BorrowCard`、`LibraryItem`（抽象）、`ItemCopy`、`Loan`、`Reservation`、`FineRecord`、`AdminAccount`、`AuditLog` | 有唯一标识与生命周期，标识相等性；状态可变且需持久化 |
| **实体（具体子类，继承）** | `StudentReader`（抽象）、`JuniorCollegeReader`、`UndergraduateReader`、`MasterReader`、`DoctoralReader`、`TeacherReader` | `Reader` 的泛化（`02` §3.2、`A1`） |
| **实体（具体子类，继承）** | `Book`、`Magazine`、`Thesis` | `LibraryItem` 的泛化（借出物类型分类，`BR-031`） |
| **值对象（Value Object）** | `Money`、`DateRange`、`CardNumber`、`CopyNumber`、`ISBN`、`OverdueDays`、`AuditEntry`（审计条目内容） | 无独立标识，属性相等性，不可变 |
| **枚举（值对象特化）** | `ReaderType`、`ItemType`、`CopyStatus`、`CardStatus`、`ReservationStatus`、`FineType`、`FineStatus`、`Role` | 取值集合由 `02` 附录 A 固定，不可扩展 |
| **策略 / 配置实体** | `BorrowPolicy`、`FineRule` | 既是**策略对象**（封装可替换的计算口径，Strategy 模式），又是**配置型聚合根**（运行时可改、需审计，`BR-014`、`FR-020`） |
| **领域服务（Domain Service）** | `BorrowingService`、`ReturningService`、`CompensationService`、`ReservationService`、`FineAccrualService`、`CatalogService` | 跨聚合协调或需要读取外部资源的业务动作，不属于任何单一实体 |
| **领域服务（接口）** | `AuditRecorder`、`BorrowPolicyProvider`、`FineRuleProvider` | 领域层定义接口，实现 / 装配由应用层与基础设施层完成（NFR-003 第 3 款） |

> **为什么 `BorrowPolicy` / `FineRule` 是「策略 + 实体」双重身份**：其**行为**是策略（应还日期计算、罚款金额计算随配置变化），但**载体**是可被系统管理员在运行时修改并落审计的配置记录（`FR-020`、`BR-014`、`BR-030`），故建模为**配置型聚合根**，且必须由 `BorrowPolicyProvider` / `FineRuleProvider` 从仓储读取，**禁止**在代码中以常量或类型分支实现（`BR-014`、`02 FR-020` 验收标准第 4 条）。

### 3.2 泛化（继承 / 分类）关系

```
Reader（抽象，读者 —— 借阅主体，持有借阅证）
├── StudentReader（抽象，学生读者 —— 共同属性：学号、年级 / 培养层次）
│   ├── JuniorCollegeReader（专科生读者）   ReaderType = 专科生
│   ├── UndergraduateReader（本科生读者）   ReaderType = 本科生
│   ├── MasterReader（研究生读者）          ReaderType = 研究生
│   └── DoctoralReader（博士生读者）        ReaderType = 博士生
└── TeacherReader（教师读者，= 00 §5 的「教师」）  ReaderType = 教师

LibraryItem（抽象，图书标题 —— 借出物类型是其属性，BR-031）
├── Book（图书：中文图书 / 外文图书，由 language 决定具体 ItemType）
├── Magazine（杂志：中文杂志 / 外文杂志，由 language 决定具体 ItemType）
└── Thesis（论文：ItemType = 论文）
```

**设计说明**

1. **读者子类只决定「读者类型」**：`Reader` 的子类**不**重写借书、预约等行为，仅通过 `readerType()` 提供 `BorrowPolicy` 的查找键（`BR-001`、`L1`）。最大可借数量与借阅期限一律读 `BorrowPolicy`，子类**不**持有数值常量（`BR-014`）。
2. **罚款不区分读者类型**（`BR-008`）：`FineRule` 的查找键是 `ItemType`，与 `Reader` 子类**无**关联；教师读者**无**罚款减免分支（`02 FR-017` 验收标准第 4 条）。
3. **借出物类型是标题的属性**（`BR-031`）：`ItemType` 挂在 `LibraryItem` 上，**不**挂在 `ItemCopy` 上；同一书籍的中外文版是两个 `LibraryItem`（两个 `Book`，`language` 不同），不是同一标题的两个副本。
4. **`LibraryItem` 子类不改变借阅规则**：`BR-002` 规定借阅期限与借出物类型无关，故 `Book / Magazine / Thesis` 的差异**只**体现在罚款费率（`ItemType → FineRule`）与检索展示，**不**体现在借期与可借数量。

**读者类型 ↔ 类 ↔ `BorrowPolicy` 对照（`BR-001`）**

| `ReaderType` | 领域类 | `maxCopies` | `loanPeriodDays` | 罚款费率差异 |
|---|---|---|---|---|
| 专科生 | `JuniorCollegeReader` | 3 | 15 | **无**（`BR-008`） |
| 本科生 | `UndergraduateReader` | 5 | 30 | 无 |
| 研究生 | `MasterReader` | 7 | 30 | 无 |
| 博士生 | `DoctoralReader` | 10 | 60 | 无 |
| 教师 | `TeacherReader` | 15 | 60 | 无（教师**无**减免，`BR-008`、`UC-21 A1`） |

> 上表数值仅供**阅读对照**，运行期一律由 `BorrowPolicyProvider` 读配置表取得，代码中不出现这些字面量（`BR-014`）。

**借出物类型 ↔ 类 ↔ `FineRule` 对照（`BR-007`、`BR-031`）**

| `ItemType` | 领域类 | 判定来源 | `dailyRate` | 对借期 / 可借数量的影响 |
|---|---|---|---|---|
| 中文图书 | `Book`（`language = 中文`） | 子类 + `language` | 2 元/天 | **无**（`BR-002`） |
| 外文图书 | `Book`（`language = 外文`） | 子类 + `language` | 2.5 元/天 | 无 |
| 中文杂志 | `Magazine`（`language = 中文`） | 子类 + `language` | 1 元/天 | 无 |
| 外文杂志 | `Magazine`（`language = 外文`） | 子类 + `language` | 1.5 元/天 | 无 |
| 论文 | `Thesis` | 子类 | 3 元/天 | 无 |

> 同一书籍的中文版与外文版是**两个** `Book` 标题（`D2`、`BR-031`），因此在类图上表现为两个 `Book` 实例，而非一个 `Book` 的两种语言属性。

**建模决策 D-1：读者类型「可变更」与「固定子类」的冲突（`TBD-002`）**

1. `Reader` 持有**可变**的 `readerType` 属性，它是**规则查找的唯一依据**；子类（`StudentReader`、`TeacherReader`）表达**业务分类与身份属性**（学号 / 教职工号）以及类型的**初始取值**，而**不是**行为分支。
2. 借期与上限**只**经 `readerType` 查 `BorrowPolicy`；任何业务代码**禁止**使用 `instanceof` 按子类分支（`BR-014`、第七条第 3 款）。
3. 系统管理员修正读者类型时：**同子类内修正**（如本科生 → 研究生）仅改 `readerType`；**跨子类修正**（如学生 → 教师）以新子类**重建 `Reader` 实体并沿用原 `id` 与 `BorrowCard`**（借阅证号与状态不变，历史借阅 / 罚款记录不受影响）。
4. 已借出记录的 `dueDate` **不因**类型变更重算（借书时已快照，见 §4.12 D-2）。

### 3.3 聚合与边界

| 聚合根 | 聚合内成员 | 不变式（事务一致性边界） |
|---|---|---|
| `Reader` | `Reader` + `BorrowCard`（1:1） | 读者与借阅证**原子创建**（`BR-024`）；不存在「有账号无借阅证」；注销为逻辑删除（`BR-026`） |
| `LibraryItem` | `LibraryItem` + `ItemCopy`（1:N） | 副本编号全局唯一（`BR-019`）；标题删除时不得存在在借副本或有效预约（`TBD-012`） |
| `Loan` | `Loan` | 应还日期 = 借阅日期 + `BorrowPolicy[读者类型].loanPeriodDays`；同标题在借副本 ≤ 1（`BR-003`） |
| `Reservation` | `Reservation`（同标题的一组预约构成 FIFO 队列，由 `ReservationService` 协调） | 队列严格 FIFO（`BR-016`）；单人有效预约 ≤ 3（`BR-018`） |
| `FineRecord` | `FineRecord` | 未缴余额 ≥ 0；减免不超过未缴余额（`FR-019`）；同一自然日对同一 `Loan` 不重复计提（NFR-009） |
| `BorrowPolicy` / `FineRule` | 各自独立，按 `ReaderType` / `ItemType` 各 5 行 | 数值非负；修改即时生效无需重启（`FR-020`） |
| `AuditLog` | `AuditLog` | 只追加，不可修改 / 删除（`BR-030`、`NFR-004` 第 5 条） |

> **跨聚合引用一律使用标识（ID）**，不持有对象引用：`Loan` 持有 `readerId` + `copyId`，`Reservation` 持有 `readerId` + `itemId`，`FineRecord` 持有 `readerId` + `loanId` / `copyId`。聚合间一致性由领域服务在**事务边界内**保证（NFR-009 第 2 条）。

### 3.4 领域类的业务来源（反「数据库表机械翻译」自检）

**准入判据**：一个类进入领域模型，当且仅当满足至少一条 ——
① 来自 `02` 的术语 / 业务规则或 `03` 的用例语义；② 承载至少一个业务不变式；③ 封装一条**可变更**的策略口径。

| 领域类 | 业务来源（业务概念，非表） | 承载的不变式 / 策略 |
|---|---|---|
| `Reader` + 子类 | 「读者」是借阅主体，学生 / 教师决定借阅资格（`02` §3.2、`L1`） | 注册即发证（无中间态）；类型只作策略键 |
| `BorrowCard` | 「借阅证」是**资格凭证**而非账号附属行（`BR-024`、`BR-027`） | 与读者 1:1 原子创建；无有效期；仅两态 |
| `LibraryItem` + 子类 | 「标题」是可被检索、预约的对象，与「副本」是业务上不同的东西（`D1`、`BR-019`、`BR-031`） | 借出物类型是标题属性；中外文版是两个标题 |
| `ItemCopy` | 「馆藏副本」是**物理可借出物**，业务上需独立编号与状态（`D1`、`BR-020`） | 五态状态机（BR-021）；报废为终态 |
| `Loan` | 「借出—归还」这一业务事实（`FR-009` / `FR-010`） | 应还日期快照；不可续借；归还不结算罚款 |
| `Reservation` | 「排队等书」这一业务概念（`R1`、`R2`、`BR-015`） | FIFO 队列；单人上限 3；队列可调序不可增删 |
| `BorrowPolicy` | 「读者类型分档规则」是**可变更的业务口径**（`B1`、`R-4`） | 策略：应还日期、可借上限 |
| `FineRule` | 「借出物类型日费率」是**可变更的业务口径**（`F1`、`B1`） | 策略：逾期金额计算 |
| `FineRecord` | 「应收款项」区分超期与赔偿，共用结清路径（`BR-032`） | 未缴余额 ≥ 0；减免不超余额；封顶只算超期 |
| `AdminAccount` | 「管理员」是权限主体，与读者是不同的业务身份（`A2`、§3.3 权限矩阵） | 管理员不可借书；不可删最后一个系统管理员 |
| `AuditLog` | 「操作可追溯」是业务的合规要求（`P2`、`BR-030`） | 只追加；不可改 / 不可删 |

**明确不建模（按数据库表机械翻译会多出来的东西）**

| 表式概念 | 为何不建模为领域类 |
|---|---|
| 字典 / 码表（`reader_type_dict`、`item_type_dict`、`copy_status_dict`） | 在领域层是**枚举值对象**，取值由 `02` 附录 A 固定，无生命周期与行为 |
| 外键列与关联表（`loan.copy_id`、`reservation.item_id` 等） | 领域层表达为**跨聚合标识引用**（`CopyId` / `LibraryItemId`），不产生类 |
| 纯技术列（`created_at` / `updated_at` / `version`） | 存储与并发控制细节，不承载业务规则 |
| `deleted` 标记列 | **例外保留**：`NFR-008` / `TBD-012` 规定「逻辑删除」是**业务语义**（注销 / 报废 / 下架后仍可追溯历史），故以 `LibraryItem.deleted`、`ItemCopy.deleted`、`AdminAccount.status` 作为**领域状态**建模，并在状态迁移与删除校验中显式约束；它不是存储细节 |
| 查询结果 / 视图（检索结果、借阅与罚款汇总） | 是**读模型 DTO**（`UC-04` / `UC-05` / `UC-11`），无业务行为，不进领域层 |
| 会话 / 令牌表 | 会话与 30 分钟超时属应用层（`FR-004`），仅保留 `SessionToken` 值对象用于传递 |

---

## 4. 领域类详述

> 每个类统一给出：**类名 / 构造型 / 职责 / 关键属性 / 关键方法或行为 / 约束 / 与其他类的关系**。

---

### 4.1 `Reader`（读者，抽象基类）

| 项 | 内容 |
|---|---|
| 类名 | `Reader`（abstract） |
| 构造型 | **实体 / 聚合根** |
| 职责 | 系统的借阅主体：持有身份凭证（用户名 + BCrypt 口令）、持有 1 张借阅证、发起预约与自助缴费、查询本人借阅与罚款信息（`02` §3.1、`FR-011`、`FR-013`、`FR-018`） |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `ReaderId` | 唯一标识 |
| `username` | `String` | 登录用户名，全局唯一（`FR-003`） |
| `passwordHash` | `String` | BCrypt 哈希，60 字符；**不存明文**（`BR-029`、NFR-004） |
| `name` | `String` | 姓名（与借阅证一致，`BR-027`） |
| `department` | `String` | 系别（`BR-027`） |
| `readerType` | `ReaderType` | **规则查找的唯一依据**，可变；子类给出初始取值，5 值之一（§3.2 D-1、`02` 附录 A） |
| `card` | `BorrowCard` | 1:1，注册时同事务创建（`BR-024`） |
| `status` | `ReaderStatus`（`正常 / 已注销`） | 逻辑删除标记（`BR-026`） |

**关键方法或行为**

- `register(...)`（工厂方法）：校验用户名唯一、口令强度（8 位且含数字与字母）、读者类型合法后，**在同一事务内**创建 `Reader` 与 `BorrowCard`（`UC-01` + `UC-02`，`BR-024`）。
- `authenticate(rawPassword)`：BCrypt 比对，失败返回统一失败提示并触发审计（`FR-003`、`UC-03 E1`）。
- `readerType()`（抽象，由子类实现）：返回 `ReaderType`，供 `BorrowPolicyProvider` 查找规则。
- `isBorrowAllowed()`：借阅证状态为 `正常` 且未被注销（`BR-026`、`UC-09` 前置条件）。
- `hasUnpaidFine()` / `isBlacklisted()`：**派生状态**，由 `FineRecord` 汇总计算（当前未缴超期罚款累计额 > 0 / ≥ 50 元，`BR-010`、`BR-011`、`TBD-003`），**不**作为持久化字段冗余存储。
- `changeReaderType(type)`：系统管理员修正读者类型（`TBD-002`）；同子类内仅改 `readerType`，跨子类以新子类重建实体并沿用 `id` 与 `BorrowCard`（§3.2 D-1），借阅证号与状态不变。
- `deactivate()`：注销（逻辑删除），保留历史借阅与罚款记录（`FR-002`、`BR-026`）。

**约束**

1. 读者类型取值 ∈ `专科生 / 本科生 / 研究生 / 博士生 / 教师`（`FR-001` 验收标准第 6 条）。
2. 口令 8 位且含数字与字母，BCrypt 存储（`BR-029`）。
3. 「最大可借数量 / 借阅期限」**不**作为 `Reader` 的字段，一律由 `BorrowPolicy` 提供（`BR-014`）。
4. 「已注销」读者不可借书、不可预约，但历史记录可查（`BR-026`、`FR-012` 验收标准第 4 条）。
5. 同一自然人可同时持有 `AdminAccount`；但借书须以**读者账号**发起，系统管理员账号发起借书被拒（`BR-028`、`UC-09 E1`）。
6. `readerType` 可变，且**禁止**以 `instanceof` 按子类分支实现任何业务判定（§3.2 D-1、`BR-014`）。

**与其他类的关系**

- `Reader` 1 —— 1 `BorrowCard`（组合，同生同死）。
- `Reader` 1 —— N `Loan`；`Reader` 1 —— N `Reservation`；`Reader` 1 —— N `FineRecord`。
- `Reader` 1 —— N `AdminAccount`（同一自然人的另一身份，**独立聚合**，无强关联约束）。
- `Reader`（经 `ReaderType`）→ `BorrowPolicy`：多对一查找（**非**对象引用）。

---

### 4.2 `StudentReader` / `JuniorCollegeReader` / `UndergraduateReader` / `MasterReader` / `DoctoralReader`（学生读者）

| 项 | 内容 |
|---|---|
| 类名 | `StudentReader`（abstract）→ `JuniorCollegeReader`、`UndergraduateReader`、`MasterReader`、`DoctoralReader` |
| 构造型 | **实体（`Reader` 的具体子类）** |
| 职责 | 表示学生读者的四类培养层次；其存在**唯一目的**是给出 `ReaderType`，从而决定最大可借数量与借阅期限（`02` §3.2、`BR-001`） |

**关键属性**：继承 `Reader` 全部属性；新增 `studentNo`（学号）、`grade`（年级 / 培养层次描述，可选）。`readerType` 分别固定为 `专科生 / 本科生 / 研究生 / 博士生`。

**关键方法或行为**：`readerType()` 返回各自固定枚举值；其余行为全部继承自 `Reader`，**不重写**借书 / 预约 / 缴费逻辑。

**约束**

1. 四类学生读者**只**在 `ReaderType` 上有差异，**禁止**出现按学生类型分支的业务代码（`BR-014`）。
2. 数值（3/15d、5/30d、7/30d、10/60d）来自 `borrowing_rule` 配置表，子类中不出现这些字面量（`FR-020` 验收标准第 4 条）。
3. 罚款费率与学生类型**无关**（`BR-008`）。

**与其他类的关系**：`StudentReader` ——|> `Reader`（泛化）；`JuniorCollegeReader` 等 ——|> `StudentReader`（泛化）；经 `ReaderType` 关联 `BorrowPolicy`。

---

### 4.3 `TeacherReader`（教师读者）

| 项 | 内容 |
|---|---|
| 类名 | `TeacherReader` |
| 构造型 | **实体（`Reader` 的具体子类）** |
| 职责 | 表示教师读者（`= 00 §5` 的「教师」，同一概念）；决定其借阅上限与期限（`BR-001`：15 本 / 60 天） |

**关键属性**：继承 `Reader`；新增 `employeeNo`（教职工号，可选）。`readerType = 教师`。

**关键方法或行为**：`readerType()` 返回 `教师`；其余继承 `Reader`。

**约束**

1. 教师**无**罚款减免：罚款只看 `ItemType`（`BR-008`、`UC-21 A1`）。
2. 教师若同时持有 `AdminAccount`（`图书管理员` / `系统管理员`），两个身份**互不影响**：以管理员账号发起借书 / 预约仍被拒绝（`BR-028`、`H-4`）。

**与其他类的关系**：`TeacherReader` ——|> `Reader`；经 `ReaderType` 关联 `BorrowPolicy`。

---

### 4.4 `BorrowCard`（借阅证）

| 项 | 内容 |
|---|---|
| 类名 | `BorrowCard` |
| 构造型 | **实体（`Reader` 聚合内成员，与 `Reader` 1:1）** |
| 职责 | 承载读者的借阅资格：提供姓名、系别、借阅证号（`BR-027`）；以状态 `正常 / 已注销` 控制能否借书（`BR-025`、`BR-026`） |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `BorrowCardId` | 唯一标识 |
| `readerId` | `ReaderId` | 归属读者（1:1） |
| `cardNumber` | `CardNumber`（值对象） | 借阅证号，系统生成、全局唯一（建议 `L` + 年份 + 6 位序列，`TBD-010`） |
| `ownerName` / `department` | `String` | 姓名、系别（`BR-027`） |
| `status` | `CardStatus`（`正常 / 已注销`） | 初始 `正常`，逻辑注销（`BR-025`、`FR-002`） |

**关键方法或行为**

- `issueFor(reader)`：注册时由系统**自动**发放，与 `Reader` 同事务；**无**独立触发入口（`R-11`、`UC-02`）。
- `isActive()`：状态为 `正常`。
- `cancel()`：置 `已注销`（系统管理员执行，`UC-16`、`TBD-011`），保留历史记录。
- **不提供**挂失 / 冻结 / 补办相关状态或方法（`R-12`、`U1`）。

**约束**

1. **无有效期字段**（`BR-025`）：模型中不得出现 `validUntil` / `expireDate`。
2. 状态集合严格为 `正常 / 已注销` 两态，不存在「待审核 / 待发证 / 挂失 / 冻结」（`BR-024`、`BR-025`、`R-12`）。
3. 借阅证号非空且全局唯一（`FR-001` 验收标准第 1 条）。
4. 注销为**逻辑删除**，不物理删除记录（`NFR-008`）。

**与其他类的关系**：`BorrowCard` 1 —— 1 `Reader`；`BorrowCard.status` 是 `BorrowingService` 借书前置校验的输入（`UC-09` 步骤 3）。

---

### 4.5 `LibraryItem`（图书标题，抽象基类）

| 项 | 内容 |
|---|---|
| 类名 | `LibraryItem`（abstract） |
| 构造型 | **实体 / 聚合根** |
| 职责 | 表达「可被检索、可被预约 / 借阅的**标题**」：持有书目信息与**借出物类型**，并作为多个馆藏副本的容器（`BR-019`、`BR-031`） |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `LibraryItemId` | 唯一标识 |
| `title` | `String` | 书名 / 题名 |
| `author` | `String` | 作者 |
| `isbn` | `ISBN`（值对象） | ISBN |
| `itemType` | `ItemType`（enum） | **借出物类型**，5 值之一，由子类与 `language` 共同决定（`BR-031`） |
| `copies` | `Set<ItemCopy>` | 1:N 馆藏副本（`BR-019`） |
| `deleted` | `boolean` | 逻辑删除标记（`TBD-012`） |

**关键方法或行为**

- `addCopy(copyNumber)` / `removeCopy(copy)`：新增副本（初始 `在馆`，`FR-007`）与删除副本（仅 `在馆` / `遗失` 可删，`FR-007` 验收标准第 4 条）。
- `hasAvailableCopy()`：是否存在任一 `在馆` 副本 —— 预约触发条件的**反向判定**（`BR-015`）。
- `hasActiveReservation()`：是否存在有效预约 —— 决定还书后副本去向（`BR-017`、`BR-021`）。
- `itemType()`（抽象）：返回 `ItemType`，供 `FineRuleProvider` 查找日费率（`BR-007`、`BR-031`）。
- `matches(keyword)`：书名 / 作者 / ISBN 的**精确或前缀**匹配（**不做**分词与相关性排序，`FR-023`、`00` §8）。
- `logicalDelete()`：存在在借副本或有效预约时拒绝（`FR-006` 验收标准第 4、8 条）。

**约束**

1. 借出物类型是**标题**的属性，**不是**副本的属性（`BR-031`）。
2. 同一书籍的中文版与外文版是**两个** `LibraryItem`（`D2`、`BR-031`）。
3. 同一 `ISBN` + 同一 `ItemType` 不得重复录入（`FR-006` 验收标准第 7 条）。
4. 逻辑删除后不出现在检索结果中，也不能再作为新增副本的归属标题（`FR-006` / `FR-007`、`TBD-012`）。
5. 不实现复杂全文检索（`FR-023` 验收标准第 7 条）。

**与其他类的关系**：`LibraryItem` 1 —— N `ItemCopy`；`LibraryItem` 1 —— N `Reservation`（预约针对**标题**而非副本）；`LibraryItem`（经 `ItemType`）→ `FineRule`；`Book / Magazine / Thesis` ——|> `LibraryItem`。

---

### 4.6 `Book`（图书）

| 项 | 内容 |
|---|---|
| 类名 | `Book` |
| 构造型 | **实体（`LibraryItem` 的具体子类）** |
| 职责 | 表达「图书」类标题；`language` 决定其 `ItemType` 为 `中文图书` 或 `外文图书` |

**关键属性**：继承 `LibraryItem`；新增 `language`（`中文 / 外文`）、`publisher`、`publishYear`（可选）。`itemType = (language == 中文 ? 中文图书 : 外文图书)`。

**关键方法或行为**：`itemType()` 由 `language` 推导；其余继承 `LibraryItem`。

**约束**：中外文版为**两个独立 `Book` 标题**，不得合并为同一标题的两个副本（`BR-031`）；`itemType` 取值只能落在 `中文图书 / 外文图书`。

**与其他类的关系**：`Book` ——|> `LibraryItem`；经 `ItemType` 关联 `FineRule`（中文图书 2 元/天，外文图书 2.5 元/天，`BR-007`）。

---

### 4.7 `Magazine`（杂志）

| 项 | 内容 |
|---|---|
| 类名 | `Magazine` |
| 构造型 | **实体（`LibraryItem` 的具体子类）** |
| 职责 | 表达「杂志」类标题；`language` 决定其 `ItemType` 为 `中文杂志` 或 `外文杂志` |

**关键属性**：继承 `LibraryItem`；新增 `language`（`中文 / 外文`）、`issue`（期号，可选）、`periodicalNo`（刊号，可选）。

**关键方法或行为**：`itemType()` 由 `language` 推导；其余继承 `LibraryItem`。

**约束**：借出物类型只能落在 `中文杂志 / 外文杂志`；借期与可借数量仍由**读者类型**决定，与杂志无关（`BR-002`）。

**与其他类的关系**：`Magazine` ——|> `LibraryItem`；经 `ItemType` 关联 `FineRule`（中文杂志 1 元/天，外文杂志 1.5 元/天）。

---

### 4.8 `Thesis`（论文）

| 项 | 内容 |
|---|---|
| 类名 | `Thesis` |
| 构造型 | **实体（`LibraryItem` 的具体子类）** |
| 职责 | 表达「论文」类标题；`ItemType` 恒为 `论文` |

**关键属性**：继承 `LibraryItem`；新增 `department`（所属系别 / 学位授予单位）、`degreeYear`（年份）、`supervisor`（指导教师，可选）。

**关键方法或行为**：`itemType()` 恒返回 `论文`；其余继承 `LibraryItem`。

**约束**：`ItemType` 恒为 `论文`；日费率 3 元/天（`BR-007`）；借期仍由读者类型决定，论文的借期与中文图书**相同**（`BR-002`、`FR-009` 验收标准第 8 条）。

**与其他类的关系**：`Thesis` ——|> `LibraryItem`；经 `ItemType` 关联 `FineRule`。

---

### 4.9 `ItemCopy`（馆藏副本）

| 项 | 内容 |
|---|---|
| 类名 | `ItemCopy` |
| 构造型 | **实体（`LibraryItem` 聚合内成员）** |
| 职责 | 表达一本**物理可借出物**：承载五态状态机（`BR-020` / `BR-021`）、保留期到期时间、以及「是否可被借出 / 保留」的判定 |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `ItemCopyId` | 唯一标识 |
| `itemId` | `LibraryItemId` | 所属标题（N:1） |
| `copyNumber` | `CopyNumber`（值对象） | **全局唯一**副本编号（`BR-019`、`FR-007`） |
| `status` | `CopyStatus`（enum） | `在馆 / 已借出 / 预约保留 / 遗失 / 报废`（**无「损坏」**，`BR-020`） |
| `reservedUntil` | `LocalDate` | `预约保留` 状态的保留到期时间 = 归还时间 + 3 天（`BR-017`） |
| `deleted` | `boolean` | 逻辑删除标记（`TBD-012`） |

**关键方法或行为**

- `markOnLoan()`：`在馆 | 预约保留` → `已借出`（`BR-021`）。
- `holdForReservation(expireAt)`：归还且有有效预约 → `预约保留` + 设置保留到期时间（`BR-017`）。
- `release()`：`预约保留` → `在馆`（无下一位预约者时，`FR-015` 验收标准第 4 条）。
- `markLost()`：`已借出` → `遗失`，并生成 `类型 = 赔偿` 的未缴款项（`BR-022`）。
- `scrap()`：→ `报废`（**系统管理员**显式执行，**不得**由遗失赔偿自动触发，`BR-022`、`FR-008` 验收标准第 3 条）。
- `isBorrowable()`：仅 `在馆` 可借；`预约保留` 时仅队首预约者可借（`BR-021`、`UC-09 E7`）。
- `isReservationHolding()` / `isReservationExpired(today)`：支撑到期扫描与顺延（`FR-015`）。

**约束**

1. 状态集合严格五态，**不含「损坏」**（`BR-020`、`R-5`）。
2. `报废` 是**终态**，不可逆（`BR-021`）。
3. `遗失` **不能**自动迁移回 `在馆`（不实现找回流程，`R-15`）。
4. `已借出` / `预约保留` 的副本**不可**被逻辑删除（`FR-007` 验收标准第 4 条）。
5. 报废与删除均为**逻辑删除**，历史借阅记录保留（`NFR-008`）。
6. 副本编号全局唯一，新增时校验（`FR-007` 验收标准第 1 条）。

**与其他类的关系**：`ItemCopy` N —— 1 `LibraryItem`；`ItemCopy` 1 —— N `Loan`（生命周期内可有多条借阅记录，同一时刻至多 1 条未还）；`ItemCopy` 1 —— N `FineRecord`（赔偿款）；状态迁移由 `BorrowingService` / `ReturningService` / `ReservationService` / `CatalogService` 驱动。

---

### 4.10 `Loan`（借阅记录）

| 项 | 内容 |
|---|---|
| 类名 | `Loan` |
| 构造型 | **实体 / 聚合根** |
| 职责 | 记录一次「借出 → 归还」的事实：借阅日期、应还日期、归还日期；提供逾期天数与超期判定，作为罚款计提的**唯一输入** |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `LoanId` | 唯一标识 |
| `readerId` | `ReaderId` | 借阅人（N:1） |
| `copyId` | `ItemCopyId` | 所借副本（N:1） |
| `borrowDate` | `LocalDate` | 借阅日期 = 当前日期 |
| `dueDate` | `LocalDate` | 应还日期 = `borrowDate + BorrowPolicy[readerType].loanPeriodDays`（`BR-001`、`BR-002`） |
| `returnDate` | `LocalDate`（可空） | 归还日期；为空表示**尚未归还** |
| `status` | `LoanStatus`（`在借 / 已还`） | **借出状态与归还状态的显式表达**：`在借` ⟺ `returnDate == null`；`已还` ⟺ `returnDate != null`（二者互为充要条件，见「状态不变量」） |
| `operatorId` | `AdminAccountId` | 办理借书的图书管理员（`U2`） |

**关键方法或行为**

- `create(reader, copy, policy, today)`（工厂）：计算 `dueDate`，**不改变**副本状态（状态迁移由 `BorrowingService` 在同一事务内完成）。
- `isOverdue(today)`：`returnDate == null && today > dueDate`。
- `overdueDays(today)`：逾期**自然日**数（`BR-007`，宽限期 0）。
- `return(today)`：记录归还时间，置 `已还`；**不生成、不结算**罚款（`BR-009`、`R-9`、`UC-10` 步骤 7）。
- `isOnLoan()` / `isReturned()`：`在借` / `已还` 的显式判定，供 `FineAccrualService` 筛选「未还且已超期」的计提对象。
- `belongsToSameTitleAs(otherLoan)`：支撑同标题 ≤ 1 副本校验（`BR-003`）。

**借出与归还状态的表达（状态不变量）**

1. `status = 在借` ⟺ `returnDate == null`；`status = 已还` ⟺ `returnDate != null`。**不存在**「已还但无归还时间」或「有归还时间但仍为在借」。
2. `borrowDate <= dueDate`；`returnDate == null || returnDate >= borrowDate`。
3. 状态迁移**单向**：`在借 → 已还` 一次性完成，**不可**回到 `在借`；已还记录重复归还被拒绝（`FR-010` 验收标准第 4 条）。
4. 归还**不**修改 `dueDate`（不支持续借，`BR-006`）、**不**生成罚款（`BR-009`、`R-9`）。
5. `在借` 期间可被每日计提（`UC-21`），计提**不改变** `Loan` 状态 —— 借款状态与罚款状态解耦。

**约束**

1. `dueDate` **只**由读者类型决定，与借出物类型无关（`BR-002`）。
2. **不支持续借**：不存在修改 `dueDate` 的方法（`BR-006`、`L3`）。
3. 同一副本在同一时刻最多 1 条未还记录（NFR-009 第 3 条）。
4. 重复归还被拒绝（`FR-010` 验收标准第 4 条）。
5. 归还**不阻断**于未缴罚款，也**不**触发一次性结算（`BR-012`、`FR-010` 验收标准第 2、3 条）。

**与其他类的关系**：`Loan` N —— 1 `Reader`；`Loan` N —— 1 `ItemCopy`；`Loan` 1 —— N `FineRecord`（`类型 = 超期`）；`Loan` 经 `Reader` 的 `ReaderType` → `BorrowPolicy`；经 `Copy → LibraryItem → ItemType` → `FineRule`。

---

### 4.11 `Reservation`（预约）

| 项 | 内容 |
|---|---|
| 类名 | `Reservation` |
| 构造型 | **实体 / 聚合根**（同一 `LibraryItem` 下的有效预约构成 FIFO 队列） |
| 职责 | 表达读者对某一**标题**的预约：维护 FIFO 位次与生命周期状态，支撑保留、顺延、取消与队列调整（`BR-015`~`BR-018`） |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `ReservationId` | 唯一标识 |
| `readerId` | `ReaderId` | 预约人（N:1） |
| `itemId` | `LibraryItemId` | 预约的**标题**（N:1，非副本） |
| `reservedAt` | `LocalDateTime` | 预约发起时间；FIFO 的**原始次序**与位次相同时的稳定次序（`BR-016`） |
| `queuePosition` | `int` | **队列位次**（排序键，可被系统管理员重排，`FR-016`）；调整**不改** `reservedAt` |
| `status` | `ReservationStatus` | `等待中 / 保留中 / 已完成 / 已取消 / 已失效(过期顺延)`；其中 `等待中` / `保留中` 为**在队（有效）**状态 |
| `holdUntil` | `LocalDate`（可空） | `保留中` 时的保留到期时间（`BR-017`） |

**排队机制（`BR-015`~`BR-018`）**

1. **队列的定义**：队列 = 同一 `LibraryItem` 下全部**在队**（`等待中` / `保留中`）`Reservation` 构成的有序集合，由 `ReservationRepository.findActiveByItemOrderByQueue(itemId)` 以「`queuePosition` 升序，`reservedAt` 升序」返回。**不**另建 `ReservationQueue` 实体 —— 它只是容器，无独立业务行为（见 §3.4 反机械翻译判据）。
2. **入队**：新预约经 `enqueue(queue)` 追加至**队尾**，`queuePosition = 当前最大位次 + 1`，`reservedAt` 记录真实发起时间（`BR-016`、`UC-06 A1`）。
3. **队首判定**：`isHead(queue)` = 在队集合中位次最小者；**仅队首**可在 `预约保留` 期内取书（`BR-021`、`UC-09 E7`、`FR-015` 验收标准第 1 条）。
4. **队列调整**（`UC-20`）：`reorder(queue, newOrder)` 仅重排 `queuePosition`，**不**新增 / 不删除预约、**不**修改 `reservedAt`（`FR-016` 验收标准第 3 条）；调整后下一次顺延按新顺序执行（`UC-06 A1`）。
5. **出队**：`fulfill()`（队首取书）、`cancel()`（读者主动取消）、`expire()`（保留期届满未取）后该预约退出队列；其余预约的 `queuePosition` **保持不变**，「依次前移」由有序查询自然体现，**不做**批量重排（避免并发写放大）。
6. **并发与容量**：同一 `readerId` + `itemId` 至多 1 条在队预约（唯一约束，防并发重复提交，`UC-06 E5`）；单人有效预约总数 ≤ 3（`BR-018`）。

**关键方法或行为**

- `create(reader, item, now)`：校验「全部副本均无 `在馆`」（`BR-015`）、未借同标题副本、无重复预约、有效预约数 < 3（`BR-018`）。
- `enqueue(queue)` / `reorder(queue, newOrder)` / `isHead(queue)`：见上「排队机制」第 2~4 条；队首判定决定 `预约保留` 副本能否被借出（`BR-021`、`UC-09` 步骤 5）。
- `promote(holdUntil)`：被顺延为当前保留者，重算 3 天保留期（`FR-015` 验收标准第 3 条）。
- `expire()`：保留期届满未取 → `已失效`，副本转下一位或转 `在馆`（`FR-015`）。
- `cancel()`：读者主动取消 → `已取消`，**不产生罚款、不记违约**（`FR-014` 验收标准第 4 条）。
- `fulfill()`：队首取书 → `已完成` 并出队（`BR-021`）。

**约束**

1. 预约对象是**标题**不是副本（`BR-015`、`R1`）。
2. 队列严格 FIFO，系统管理员可**只改顺序**，不得新增 / 删除预约（`FR-016` 验收标准第 3 条）。
3. 单人有效预约上限 3（`BR-018`）。
4. 预约**不发送**任何到馆通知（无邮件 / 短信通道，`R3`）。
5. 系统管理员默认**不可**预约（`TBD-006`、`H-4`）。
6. 取消与过期**不**产生罚款（`FR-014`、`UC-07 E3`）。

**与其他类的关系**：`Reservation` N —— 1 `Reader`；`Reservation` N —— 1 `LibraryItem`；与 `Loan` 通过「队首取书」在 `BorrowingService` 中衔接（保留中的预约出队并生成 `Loan`）；队列协调与到期扫描由 `ReservationService` 完成。

---

### 4.12 `BorrowPolicy`（借阅规则）

| 项 | 内容 |
|---|---|
| 类名 | `BorrowPolicy` |
| 构造型 | **策略 + 配置型聚合根**（Strategy 模式；`02` §6.7、`BR-001`、`BR-014`） |
| 职责 | 封装「读者类型 → 最大可借数量 + 借阅期限」的口径，并负责计算应还日期与校验在借上限 |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `BorrowPolicyId` | 唯一标识 |
| `readerType` | `ReaderType` | 唯一键，5 行（`专科生 / 本科生 / 研究生 / 博士生 / 教师`） |
| `maxCopies` | `int` | 最大可借**副本**数（3 / 5 / 7 / 10 / 15） |
| `loanPeriodDays` | `int` | 借阅期限天数（15 / 30 / 30 / 60 / 60） |

**关键方法或行为**

- `computeDueDate(borrowDate)` → `LocalDate`：`borrowDate + loanPeriodDays`（`BR-001`、`BR-002`）。
- `exceedsLimit(currentLoanedCopies)` → `boolean`：在借副本数是否达上限（`BR-004`、`R-18`）。
- `validate(values)`：数量与天数**非负**（`FR-020`、`UC-18 E2`）。
- `appliesTo(readerType)`：判定该策略行是否适用于给定读者类型（策略选择，供 `BorrowPolicyProvider` 使用）。

**建模决策 D-2：规则抽象为策略对象，而非「配置表的一行」**

1. **抽象来源**：`BorrowPolicy` 抽象的是业务口径「*某类读者最多借几本、能借多久*」（`B1`、`R-4`、`BR-001`），不是 `borrowing_rule` 表的一行；表只是它的**持久化形式**。因此它对外暴露的是**行为**（`computeDueDate` / `exceedsLimit`），不是 getter 集合。
2. **调用点唯一**：`BorrowingService` 通过 `BorrowPolicyProvider.forReaderType(readerType)` 取得策略，**不得**在其他任何位置读取 `maxCopies` / `loanPeriodDays` 自行计算（第七条第 3 款）。
3. **生效语义（快照）**：`dueDate` 在借书当时由策略计算并**快照**进 `Loan`（`Loan.create(..., policy, today)`）。规则变更后只对**新的**借阅生效，**不追溯**已借出记录（`FR-020` 验收标准第 3 条、`UC-18` 后置条件）。
4. **可替换性**：因策略是对象，测试可注入任意口径（如「0 天期限」）而无需改代码或改库（NFR-005）。

**约束**

1. 数值来自 `borrowing_rule` 配置表，代码中**禁止**出现 `3 / 5 / 7 / 10 / 15` 等规则字面量参与业务判断（种子数据脚本除外，`FR-020` 验收标准第 4 条）。
2. 修改后**立即生效**，无需重启 / 重新部署（`FR-020` 验收标准第 3 条）。
3. 只按 `ReaderType` 分档，**不**按借出物类型分档（`BR-002`）。
4. 规则变更写入审计日志（`FR-020` 验收标准第 6 条）。

**与其他类的关系**：`BorrowPolicy` 按 `ReaderType` 与 `Reader` 的子类**间接**关联（多对一查找，通过 `BorrowPolicyProvider`）；被 `BorrowingService`（计算 `Loan.dueDate`、校验上限）使用。

---

### 4.13 `FineRule`（罚款规则）

| 项 | 内容 |
|---|---|
| 类名 | `FineRule` |
| 构造型 | **策略 + 配置型聚合根**（Strategy 模式；`BR-007`、`BR-014`） |
| 职责 | 封装「借出物类型 → 日费率」的口径，并负责按逾期自然日计算罚款金额 |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `FineRuleId` | 唯一标识 |
| `itemType` | `ItemType` | 唯一键，5 行（`中文图书 / 外文图书 / 中文杂志 / 外文杂志 / 论文`） |
| `dailyRate` | `Money` | 日费率（元/天：2 / 2.5 / 1 / 1.5 / 3） |
| `gracePeriodDays` | `int` | 宽限期，恒为 **0**（`BR-007`） |

**关键方法或行为**

- `accrue(overdueDays)` → `Money`：`逾期天数 × dailyRate`，宽限期 0（`BR-007`、`FR-017` 验收标准第 2 条）。
- `isAccruable(overdueDays)`：`overdueDays > gracePeriodDays`。
- `appliesTo(itemType)`：判定该策略行是否适用于给定借出物类型（供 `FineRuleProvider` 使用）。
- `validate(values)`：费率非负（`FR-020`）。

**建模决策 D-3：罚款规则抽象为策略对象，且与 `BorrowPolicy` 的生效语义不同**

1. **抽象来源**：`FineRule` 抽象的是业务口径「*某类借出物逾期一天罚多少*」（`F1`、`B1`、`BR-007`），不是 `fine_rule` 表的一行。
2. **调用点唯一**：`FineAccrualService` 通过 `FineRuleProvider.forItemType(itemType)` 取得策略并调用 `accrue(...)`；代码中**禁止**出现按 `ItemType` 的 `switch` / `if` 费率分支（`BR-014`、`FR-020` 验收标准第 4 条）。
3. **生效语义（实时读取，与 D-2 不同）**：每次计提都**实时读取**当前 `FineRule`，故规则变更后**下一次计提**即按新费率生效（`UC-18` 后置条件、`UC-21 A3`）；已生成的 `FineRecord` 金额**不追溯**重算。
4. **缺费率即失败**：`FineRuleProvider` 找不到对应 `ItemType` 时抛错并由任务留痕，**禁止**回退默认费率或按 0 计提（`UC-21 E1`）。

**约束**

1. 费率读 `fine_rule` 配置表，**禁止**硬编码或按类型写 `if/switch` 分支（`BR-014`、第七条第 3 款）。
2. 罚款**不区分读者类型**，教师无减免（`BR-008`）。
3. 缺少某 `ItemType` 费率时计提任务**失败并留痕**，**禁止**按 0 或默认值静默计提（`UC-21 E1`）。
4. 修改即时生效（`FR-020`）。

**与其他类的关系**：`FineRule` 按 `ItemType` 与 `LibraryItem` 间接关联（多对一查找，通过 `FineRuleProvider`）；被 `FineAccrualService` 使用于 `UC-21`。

---

### 4.14 `FineRecord`（罚款记录 / 款项记录）

| 项 | 内容 |
|---|---|
| 类名 | `FineRecord` |
| 构造型 | **实体 / 聚合根** |
| 职责 | 记录一笔应收款项：区分 `超期` 与 `赔偿`，复用同一套 `未缴 / 已缴` 状态与结清路径（`BR-032`）；支撑余额减免、封顶与黑名单判定 |

**关键属性**

| 属性 | 类型 | 说明 |
|---|---|---|
| `id` | `FineRecordId` | 唯一标识 |
| `readerId` | `ReaderId` | 欠款人（N:1） |
| `type` | `FineType`（`超期 / 赔偿`） | 罚款类型（`BR-032`） |
| `amount` | `Money` | 原始金额 |
| `unpaidAmount` | `Money` | **未缴余额**（减免后减少，`FR-019`） |
| `status` | `FineStatus`（`未缴 / 已缴`） | 结清状态（`BR-012`） |
| `accrualDate` | `LocalDate` | 计提日（超期）/ 登记日（赔偿） |
| `settledAt` | `LocalDateTime`（可空） | 结清时间（`FR-018`） |
| `loanId` | `LoanId`（可空） | `超期` 罚款关联的借阅记录 |
| `copyId` | `ItemCopyId`（可空） | `赔偿` 款关联的副本 |

**关键方法或行为**

- `accrueFor(loan, rule, today)`（工厂 / 幂等更新）：同一自然日对同一 `Loan` **不重复计提**（NFR-009、`UC-21` 步骤 4）。
- `createCompensation(copy, amount)`：金额由图书管理员**人工录入**，系统**不做**定价计算（`R-15`、`FR-021`）。
- `pay()`：读者自助缴纳 → `已缴` + 记录结清时间（记账式，**不调用**支付网关，`FR-018`、`00` §8）。
- `reduce(amount)`：系统管理员减免；超出未缴余额拒绝；减至 0 视为结清（`FR-019`）。
- `isSettled()` / `unpaidAmount()`：供借阅限制与黑名单重算使用（`BR-010`、`BR-011`）。

**约束**

1. 类型严格 `超期 / 赔偿` 两值，状态严格 `未缴 / 已缴` 两值（`BR-032`、`02` 附录 A）。
2. **赔偿款不计入 50 元封顶**（`BR-013`）；未缴赔偿款按「存在未缴款项」处理并禁止借阅（`TBD-007`、`H-3`）。
3. 缴纳与归还**解耦**：还书路径**不**包含缴费步骤（`R-16`、第十条第 2 款）。
4. 不可重复缴纳已 `已缴` 的款项；不可代他人缴纳（`FR-018` 验收标准第 5、6 条）。
5. 减免后 `unpaidAmount >= 0`，且金额精度以 `Money`（定点小数）表达，避免浮点误差。

**与其他类的关系**：`FineRecord` N —— 1 `Reader`；`FineRecord` N —— 0..1 `Loan`（超期）；`FineRecord` N —— 0..1 `ItemCopy`（赔偿）；产生金额时经 `FineRule` 计算；参与 `Reader.hasUnpaidFine()` / `isBlacklisted()` 的汇总计算。

---

### 4.15 `AdminAccount`（管理员账号）

| 项 | 内容 |
|---|---|
| 类名 | `AdminAccount` |
| 构造型 | **实体 / 聚合根** |
| 职责 | 承载图书管理员与系统管理员的身份与角色，作为权限矩阵（`02` §3.3、`BR-028`）与审计「操作者」的来源 |

**关键属性**：`id`、`username`（唯一）、`passwordHash`（BCrypt）、`role`（`Role`：`图书管理员 / 系统管理员`）、`status`（`正常 / 已停用`，逻辑删除）、`createdAt`。

**关键方法或行为**

- `create(username, rawPassword, role)`：口令强度校验 + BCrypt 存储（`FR-022` 验收标准第 1、9 条）。
- `authenticate(rawPassword)`：BCrypt 比对（`BR-029`）。
- `hasPermission(action)`：按权限矩阵判定（借还 / 维护 / 减免 / 队列调整 / 账号维护）。
- `canBorrow()` / `canReserve()`：**恒为 false**（`BR-028`、`H-4`）。
- `deactivate()`：逻辑删除；**禁止**删除最后一个系统管理员账号（`FR-022` 验收标准第 4 条）。

**约束**

1. 系统管理员**不可借书**、默认不可预约（`BR-028`、`TBD-006`）。
2. 图书管理员**不能修改自己的账号信息**（`BR-028`、`FR-022` 验收标准第 3 条）。
3. 图书管理员**无任何**图书 / 标题 / 副本维护权限（`BR-028`）。
4. 删除为逻辑删除，其历史审计记录**不得**被删除（`BR-030`、`FR-022` 验收标准第 5 条）。
5. 初始系统管理员账号由种子数据提供（`TBD-008`）。

**与其他类的关系**：`AdminAccount` 是 `AuditLog.actorId` 的来源之一；作为 `Loan.operatorId`（办理借还的图书管理员）；与 `Reader` 无继承关系（同一自然人可分别持有两个身份）。

---

### 4.16 `AuditLog`（审计日志）

| 项 | 内容 |
|---|---|
| 类名 | `AuditLog` |
| 构造型 | **实体 / 聚合根（只追加，Append-only）** |
| 职责 | 记录关键写操作，保证操作可追溯（`FR-005`、`BR-030`、`NFR-004`） |

**关键属性**：`id`、`actorId`（操作者标识，指向 `Reader` 或 `AdminAccount`）、`actorRole`、`actionType`（登录失败 / 借书 / 还书 / 预约 / 取消预约 / 调整预约队列 / 登记遗失赔偿 / 缴纳罚款 / 减免罚款 / 维护标题 / 维护副本 / 报废 / 维护管理员 / 修改规则）、`targetType` + `targetId`、`result`（`成功 / 失败`）、`occurredAt`。

**关键方法或行为**

- `record(entry)`：只追加写入；**不提供** `update` / `delete` 方法（`BR-030`）。
- `queryByTarget(targetId)`：支持追溯至**已注销**读者与**报废**副本的历史操作（`FR-005` 验收标准第 4 条）。

**约束**

1. 业务代码中**不得**出现更新 / 删除审计表的调用（`FR-005` 验收标准第 3 条）。
2. **不落审计**的操作：注册与自动发证（`FR-005` 未列举）、只读查询（`UC-04` / `UC-05` / `UC-11`）、定时计提（仅留任务执行摘要，`H-1`）。
3. 登录失败**必须**落审计，且成功登录不落审计（`FR-003` 验收标准第 5 条、`UC-23`）。
4. 预约保留的每次顺延 / 取消**必须**落审计（`FR-015` 验收标准第 7 条）。

**与其他类的关系**：`AuditLog` 弱关联（仅以 `actorId` / `targetId` 引用）`Reader`、`AdminAccount`、`Loan`、`Reservation`、`FineRecord`、`LibraryItem`、`ItemCopy`、`BorrowPolicy`、`FineRule`；写入由 `AuditRecorder`（领域服务接口）统一入口完成。

---

## 5. 值对象与枚举

| 名称 | 构造型 | 取值 / 成员 | 约束与依据 |
|---|---|---|---|
| `ReaderType` | 枚举 | `专科生 / 本科生 / 研究生 / 博士生 / 教师` | `BorrowPolicy` 的查找键；决定可借数量与期限，**不影响**费率（`BR-001`、`BR-008`） |
| `ItemType` | 枚举 | `中文图书 / 外文图书 / 中文杂志 / 外文杂志 / 论文` | `FineRule` 的查找键；**标题**的属性（`BR-031`、`BR-007`） |
| `CopyStatus` | 枚举 | `在馆 / 已借出 / 预约保留 / 遗失 / 报废` | 五态，**不含「损坏」**（`BR-020`） |
| `CardStatus` | 枚举 | `正常 / 已注销` | 两态，无挂失 / 冻结（`BR-025`） |
| `ReservationStatus` | 枚举 | `等待中 / 保留中 / 已完成 / 已取消 / 已失效` | 支撑 FIFO 队列与顺延（`BR-016`、`BR-017`） |
| `FineType` | 枚举 | `超期 / 赔偿` | 复用同一结清路径（`BR-032`） |
| `FineStatus` | 枚举 | `未缴 / 已缴` | 结清判定（`BR-012`） |
| `Role` | 枚举 | `读者 / 图书管理员 / 系统管理员` | 权限矩阵依据（`BR-028`） |
| `Money` | 值对象 | `BigDecimal amount` + 币种（人民币） | 定点小数，不可变；支持加 / 减 / 比较，禁止浮点 |
| `CardNumber` | 值对象 | `String`（建议 `L` + 年份 + 6 位序列） | 全局唯一、非空（`BR-027`、`TBD-010`） |
| `CopyNumber` | 值对象 | `String` | 全局唯一（`BR-019`） |
| `ISBN` | 值对象 | `String` | 与 `ItemType` 联合去重（`FR-006` 验收标准第 7 条） |
| `DateRange` | 值对象 | `LocalDate from` / `to` | 借阅区间、查询区间 |
| `OverdueDays` | 值对象 | `long days` | 逾期**自然日**，宽限期 0（`BR-007`） |
| `SessionToken` | 值对象 | `String` | 仅供应用层 / 审计传递，**非**领域状态（FR-004 属应用层） |

> 值对象一律**不可变**、以属性相等性比较；持久化时作为所属实体的嵌入列或独立值表（见 `13-database-design.md`）。

---

## 6. 领域服务与策略

> 领域服务承载**跨聚合**的业务动作与规则编排；实体承载单聚合内部的不变式。应用服务负责事务、权限拦截与审计编排，**不得**复制以下规则。

| 服务 | 构造型 | 职责（对应用例） | 关键行为 | 依赖 |
|---|---|---|---|---|
| `BorrowingService` | 领域服务 | 办理借书（`UC-09`），含保留副本取书（`TBD-009`） | 校验借阅证 `正常`；执行 `BR-004` 四项前置校验；校验副本可借（`BR-021`）；未取预约**仅提示**（`BR-005`）；同事务生成 `Loan` + 置副本 `已借出` + 队首预约出队 | `ReaderRepository`、`ItemCopyRepository`、`LoanRepository`、`ReservationRepository`、`BorrowPolicyProvider`、`AuditRecorder` |
| `ReturningService` | 领域服务 | 办理还书（`UC-10`） | 校验存在未还记录且副本 `已借出`；记录归还时间；按「是否存在有效预约」决定副本去向 `在馆` / `预约保留(+3 天)`；**不生成罚款**、**不阻断于未缴罚款**（`BR-012`、`R-9`） | `LoanRepository`、`ItemCopyRepository`、`ReservationRepository`、`AuditRecorder` |
| `CompensationService` | 领域服务 | 登记遗失赔偿（`UC-12`） | 校验操作者为图书管理员；副本 `已借出` → `遗失`；按**人工录入**金额生成 `类型 = 赔偿` 的未缴款项；**不自动报废**（`BR-022`） | `LoanRepository`、`ItemCopyRepository`、`FineRecordRepository`、`AuditRecorder` |
| `ReservationService` | 领域服务 | 预约（`UC-06`）、取消（`UC-07`）、队列调整（`UC-20`）、保留到期扫描与顺延（`UC-22`） | 校验 `BR-015` 触发条件、同标题未借、无重复预约、上限 3（`BR-018`）；FIFO 入队与位次调整；到期取消 + 顺延并重算 3 天保留期，**幂等**（NFR-009） | `ReservationRepository`、`ItemCopyRepository`、`LibraryItemRepository`、`AuditRecorder` |
| `FineAccrualService` | 领域服务 | 每日定时计提超期罚款（`UC-21`） | 分页扫描「未还且应还日期 < 今日」；按 `ItemType` 取 `FineRule` 计算金额；**同自然日幂等**；缺费率则失败留痕；重算未缴累计额与黑名单（50 元封顶，仅超期款，`BR-010`、`BR-013`） | `LoanRepository`、`FineRecordRepository`、`FineRuleProvider`、`LibraryItemRepository` |
| `CatalogService` | 领域服务 | 维护标题（`UC-13`）、维护副本（`UC-14`）、报废（`UC-15`） | 校验系统管理员权限；新增副本初始 `在馆`；删除时校验无在借副本 / 无有效预约；报废为显式动作（`BR-023`） | `LibraryItemRepository`、`ItemCopyRepository`、`ReservationRepository`、`AuditRecorder` |
| `AuditRecorder` | 领域服务（接口） | 统一写入 `AuditLog`（`UC-23`） | 收集操作者、时间、类型、对象、结果；只追加 | 实现在应用层 / 基础设施层 |
| `BorrowPolicyProvider` | 策略供应者（接口） | 按 `ReaderType` 返回 `BorrowPolicy` | 读配置表，屏蔽硬编码（`BR-014`） | 实现在基础设施层 |
| `FineRuleProvider` | 策略供应者（接口） | 按 `ItemType` 返回 `FineRule` | 同上（`BR-014`、`BR-007`） | 实现在基础设施层 |

**仓储接口（领域层定义，基础设施层实现）**：`ReaderRepository`、`BorrowCardRepository`（或随 `Reader` 聚合一并持久化）、`LibraryItemRepository`、`ItemCopyRepository`、`LoanRepository`、`ReservationRepository`、`FineRecordRepository`、`BorrowPolicyRepository`、`FineRuleRepository`、`AuditLogRepository`、`AdminAccountRepository`。

---

## 7. 关键业务计算的归属

| 计算 | 归属 | 依据 |
|---|---|---|
| 应还日期 = 借阅日期 + 借阅期限 | `BorrowPolicy.computeDueDate()` | `BR-001`、`BR-002`、`BR-014` |
| 在借副本是否达上限 | `BorrowPolicy.exceedsLimit()` | `BR-004`、`R-18` |
| 逾期天数 | `Loan.overdueDays(today)` | `BR-007`（自然日、宽限期 0） |
| 罚款金额 = 逾期天数 × 日费率 | `FineRule.accrue(overdueDays)` | `BR-007`、`BR-008` |
| 未缴累计额 / 50 元封顶 / 黑名单 | `FineAccrualService` 汇总 `FineRecord.unpaidAmount`（仅 `超期`） | `BR-010`、`BR-013`、`TBD-003` |
| 副本下一状态 | `ItemCopy` 状态方法 + `BR-021` 迁移表 | `BR-020`、`BR-021` |
| 是否允许预约 | `LibraryItem.hasAvailableCopy()` 取反 + `Reservation` 校验链 | `BR-015`、`BR-018` |
| 队首判定 | `Reservation.isHead(queue)` | `BR-016`、`BR-021` |
| 权限能否执行某动作 | `AdminAccount.hasPermission()`（领域）+ 应用层统一拦截 | `BR-028`、`NFR-004` 第 4 条 |

---

## 8. 状态迁移表（替代状态图，依 `E2` / `R-19`）

### 8.1 `ItemCopy.status`（`BR-021`，须全部实现）

| 现态 | 事件 | 次态 | 副作用 | 执行者 |
|---|---|---|---|---|
| `在馆` | 借书 | `已借出` | 生成 `Loan`（含应还日期） | 图书管理员（柜台） |
| `在馆` | 报废 | `报废` | 写审计日志 | 系统管理员 |
| `已借出` | 归还（无有效预约） | `在馆` | 记录归还时间 | 图书管理员 |
| `已借出` | 归还（有有效预约） | `预约保留` | 设置保留到期时间 = 归还时间 + 3 天 | 图书管理员 |
| `已借出` | 登记遗失赔偿 | `遗失` | 生成 `类型 = 赔偿` 的未缴款项 | 图书管理员 |
| `预约保留` | 队首预约者取书 | `已借出` | 生成 `Loan`，该预约出队 | 图书管理员（`TBD-009`） |
| `预约保留` | 保留期届满未取 | `预约保留`（下一位） | 取消当前预约，重算 3 天保留期 | 系统（定时，`UC-22`） |
| `预约保留` | 保留期届满且无下一位 | `在馆` | 恢复对外可借 | 系统（定时，`UC-22`） |
| `预约保留` | 报废 | `报废` | 写审计日志 | 系统管理员 |
| `遗失` | 报废 | `报废` | 写审计日志（**非自动触发**） | 系统管理员 |
| `报废` | — | **终态** | 不可借、不可预约保留 | — |

> 禁止的迁移：`遗失 → 在馆`（无找回流程，`R-15`）；`已借出 → 报废` 由遗失自动触发（`BR-022`）；任何出现「损坏」态（`R-5`）。

### 8.2 `Loan.status`

| 现态 | 事件 | 次态 | 副作用 |
|---|---|---|---|
| （无） | 借书成功 | `在借` | 副本置 `已借出`，在借副本数 +1 |
| `在借` | 归还 | `已还` | 记录归还时间；**不生成罚款**（`R-9`） |
| `在借` | 每日计提（超期） | `在借` | 生成 / 更新 `类型 = 超期` 的 `FineRecord`（幂等） |
| `已还` | — | 终态 | 重复归还被拒绝 |

### 8.3 `Reservation.status`

| 现态 | 事件 | 次态 | 副作用 |
|---|---|---|---|
| （无） | 预约成功（`BR-015` 成立） | `等待中` | FIFO 入队，写审计日志 |
| `等待中` | 副本归还且为队首 | `保留中` | 副本置 `预约保留`，`holdUntil = 归还日 + 3 天` |
| `等待中` / `保留中` | 读者主动取消 | `已取消` | 出队；**不产生罚款**、不记违约 |
| `保留中` | 保留期届满未取 | `已失效` | 顺延下一位或副本转 `在馆`；写审计日志 |
| `等待中` / `保留中` | 队列调整（`UC-20`） | 不变（仅改 `queuePosition`） | 仅改顺序，不增不减预约；写审计日志 |
| `保留中` | 队首取书 | `已完成` | 出队并生成 `Loan`，副本置 `已借出` |

### 8.4 `BorrowCard.status` / `FineRecord.status`

| 类 | 迁移 | 说明 |
|---|---|---|
| `BorrowCard` | `正常 → 已注销` | 系统管理员执行（`FR-002`）；**无**挂失 / 冻结 / 补办（`R-12`） |
| `FineRecord` | `未缴 → 已缴` | 读者自助缴纳（`FR-018`）或管理员减免至 0（`FR-019`）；已缴后不可重复缴纳 / 减免 |

---

## 9. 需求与业务规则 ↔ 领域类 对照

| 业务规则 / 需求 | 承载类 |
|---|---|
| BR-001 / BR-002 / BR-003 / BR-004 / BR-005 / BR-006 | `BorrowPolicy`、`Loan`、`BorrowingService` |
| BR-007 / BR-008 / BR-010 / BR-011 / BR-013 / BR-014 | `FineRule`、`FineRecord`、`FineAccrualService` |
| BR-009 / BR-012 / BR-032 | `FineRecord`、`FineAccrualService`、`ReturningService` |
| BR-015 / BR-016 / BR-017 / BR-018 | `Reservation`、`LibraryItem`、`ReservationService` |
| BR-019 / BR-020 / BR-021 / BR-022 / BR-023 | `LibraryItem`、`ItemCopy`、`CatalogService`、`CompensationService` |
| BR-024 / BR-025 / BR-026 / BR-027 | `Reader`、`BorrowCard` |
| BR-028 / BR-029 / BR-030 | `AdminAccount`、`AuditLog`、`AuditRecorder` |
| BR-031 | `LibraryItem` + `Book / Magazine / Thesis` |
| FR-001 / FR-002 | `Reader`、`BorrowCard` |
| FR-006 / FR-007 / FR-008 / FR-023 | `LibraryItem`、`ItemCopy`、`CatalogService` |
| FR-009 / FR-010 / FR-012 | `Loan`、`BorrowingService`、`ReturningService` |
| FR-013 ~ FR-016 | `Reservation`、`ReservationService` |
| FR-017 ~ FR-021 | `FineRule`、`FineRecord`、`FineAccrualService`、`CompensationService` |
| FR-022 / FR-003 / FR-004 / FR-005 | `AdminAccount`、`AuditLog`（会话属应用层，不进领域模型） |

---

## 10. 假设与待确认项

| 编号 | 内容 | 影响 |
|---|---|---|
| H-1 | 「黑名单」建模为 `FineRecord` 汇总的**派生状态**，不设独立实体与持久化标志位；避免与「当前未缴累计额」口径（`TBD-003`）产生第二处真相 | `Reader.isBlacklisted()`、`FineAccrualService` |
| H-2 | 学生读者四类建模为 `StudentReader` 的**四个具体子类**（对应 `02` §3.2 层次）；子类只提供 `readerType` 初始取值与身份属性，规则查找一律走可变 `readerType`（§3.2 D-1）。若实现期认为过重，可退化为 `Reader` + 可变 `ReaderType` 属性并取消子类，但**不得**引入按类型分支的业务代码 | `Reader` 继承体系、`06-domain-class-diagram.puml` |
| H-3 | `ItemType` 由「子类 + `language`」推导而非直接持久化枚举列，以同时满足 `BR-031`（标题属性）与中外文版为两个标题（`D2`） | `LibraryItem` 子类 |
| H-4 | `BorrowCard` 与 `Reader` 同属一个聚合（1:1、原子创建），故不设独立的 `BorrowCardRepository` 写接口，随 `Reader` 聚合一并持久化 | `Reader`、`BorrowCard` |
| TBD-002 | 注册时读者类型的确定方式（自选 / 管理员修正） | `Reader.register()`、`changeReaderType()` |
| TBD-003 | 50 元封顶的计数口径（当前未缴累计额 / 历史累计额） | `FineAccrualService`、`Reader.isBlacklisted()` |
| TBD-006 | 系统管理员能否预约 | `AdminAccount.canReserve()`、`ReservationService` |
| TBD-007 | 未缴赔偿款是否禁止借阅 | `BorrowingService` 前置校验、`FineRecord` |
| TBD-010 | 借阅证号生成规则 | `CardNumber` |
| TBD-012 | 标题 / 副本删除方式（一律逻辑删除为默认） | `LibraryItem.deleted`、`ItemCopy.deleted` |

---

## 附录 A 评审项自检对照（7 项）

> 针对本轮审查提出的 7 个问题逐项自检；「结论」为**符合 / 已补正**，「证据」指向本文件的具体位置。

| # | 审查问题 | 结论 | 证据 / 补正内容 |
|---|---|---|---|
| 1 | 领域类是否来自业务，而不是数据库表的机械翻译？ | **符合（v1.1 已补正）** | §3.4 给出准入判据（来自术语 / 承载不变式 / 封装可变更策略），逐类说明业务来源；并列明**被排除**的表式概念：字典码表 → 枚举值对象、外键与关联表 → 跨聚合标识引用、`created_at/updated_at/version` → 不建模、查询视图 → 读模型 DTO、会话表 → 应用层。唯一保留的 `deleted` 标记已论证为 **`NFR-008` / `TBD-012` 规定的业务语义「逻辑删除」**，非存储细节 |
| 2 | 是否体现不同读者类型？ | **符合（v1.1 已强化）** | §3.2 读者类型 ↔ 类 ↔ `BorrowPolicy` 对照表（5 类 → 5 个类 → 数量 / 期限）；`Reader` → `StudentReader`（含 4 个学生子类）→ `TeacherReader` 继承树；新增 **D-1** 处理「类型可变更（`TBD-002`）与固定子类」的冲突：`readerType` 可变并作为唯一规则键，子类表达业务分类与初始取值，禁止 `instanceof` 分支 |
| 3 | 是否体现不同借出物类型？ | **符合（v1.1 已强化）** | §3.2 借出物类型 ↔ 类 ↔ `FineRule` 对照表（5 类 → `Book` / `Magazine` / `Thesis` → 日费率）；`ItemType` 挂在 `LibraryItem`（`BR-031`），由「子类 + `language`」推导，中外文版为两个 `Book` 标题；并明确**不影响**借期与可借数量（`BR-002`） |
| 4 | 借阅规则是否抽象为 `BorrowPolicy`？ | **符合（v1.1 已强化）** | §4.12：`BorrowPolicy` = 策略 + 配置型聚合根；新增 **D-2** 说明它抽象的是业务口径而非配置表的一行，对外暴露 `computeDueDate()` / `exceedsLimit()` 行为，调用点唯一（经 `BorrowPolicyProvider`），并规定**快照语义**（`dueDate` 在借书时快照，规则变更不追溯已借出记录） |
| 5 | 罚款规则是否抽象为 `FineRule`？ | **符合（v1.1 已强化）** | §4.13：`FineRule` = 策略 + 配置型聚合根；新增 **D-3** 说明其抽象来源、唯一调用点（经 `FineRuleProvider`，禁 `switch` 费率分支）、**实时读取语义**（每次计提读当前费率，下次计提即生效；已计提金额不追溯重算），以及缺费率即失败（`UC-21 E1`） |
| 6 | `Loan` 是否能表达借出和归还状态？ | **符合（v1.1 已强化）** | §4.10：`LoanStatus`（`在借 / 已还`）+ `returnDate` 的**充要不变量**、4 条状态不变量、单向迁移（不可回退、不可续借）、`isOverdue()` / `overdueDays()` 支撑计提；§8.2 给出状态迁移表（含「计提不改变 `Loan` 状态」的解耦说明） |
| 7 | `Reservation` 是否支持排队？ | **符合（v1.1 已强化）** | §4.11 新增「排队机制」6 条：队列 = 同标题下**在队**（`等待中` / `保留中`）预约的有序集合（`queuePosition` 升序 + `reservedAt` 升序）；队尾入队、队首判定、仅重排不增删的队列调整、出队后位次不变（前移由有序查询体现）、唯一约束防重复预约 + 单人上限 3。明确**不**另建无行为的 `ReservationQueue` 实体 |

**遗留风险（须人工确认）**

1. **D-1 的跨子类修正**：学生 → 教师的类型修正需「重建实体」，实现期须保证 `id`、`BorrowCard` 与历史记录连续；若审查认为过重，可退化为「`Reader` + 可变 `ReaderType` 属性、取消子类」（`H-2`）。
2. **队列位次的并发重排**：`UC-20` 调整顺序与 `UC-22` 顺延可能并发，须在实现层以「同标题预约的行级锁或乐观锁」保证（`NFR-009`）。
3. **数值对照表的时效性**：§3.2 两张对照表的数值来自 `02` 附录 A 的**默认种子值**，规则变更后以配置表为准（`BR-014`）。

---

## 附录 B 修订记录

| 版本 | 日期 | 修订内容 | 来源 |
|---|---|---|---|
| v1.0 | 2026-09-23 | 首版成文：12 个实体（含 `Reader` / `LibraryItem` 两棵继承树）、6 个领域服务、11 个值对象与枚举、聚合边界、状态迁移表、BR/FR↔类对照、假设与待确认项 | `02-requirements.md`（v1.1）、`03-use-cases.md`（v1.0）、`constitution.md` |
| v1.1 | 2026-09-23 | 按 7 项评审意见补正：① 新增 §3.4 领域类业务来源与反「数据库表机械翻译」自检；② §3.2 新增读者类型 / 借出物类型两张对照表与建模决策 **D-1**（类型可变更 vs 固定子类）；③ §4.12 / §4.13 新增 **D-2** / **D-3**（策略抽象来源、调用点唯一性与快照 / 实时两种生效语义）；④ §4.10 补充 `Loan` 借出 / 归还状态表达与不变量；⑤ §4.11 补充 `Reservation` 排队机制 6 条；⑥ §4.1 补充 `readerType` 可变与禁 `instanceof` 约束；⑦ 新增附录 A 评审项自检对照与遗留风险 | 人工审查意见（7 项）、`02` 附录 A、`03` §4 |

> **留痕要求（NFR-007 / 第四条第 5 款）**：本次由 Agent 生成，须在 `19-ai-usage-log.md` 新增一条记录（使用工具、使用阶段、Prompt 摘要、修改文件、输出摘要、测试结果、Git 提交），「人工审查结果」由人类补齐；审查通过后方可作为 `06-domain-class-diagram.puml` 及下游设计文档的输入。
