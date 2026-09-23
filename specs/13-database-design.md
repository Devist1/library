# 13 数据库设计说明书（Database Design）

- **输入**：`specs/05-domain-model.md`（v1.1）、`specs/07-architecture.md`（v1.0）、`specs/06-domain-class-diagram.puml`、`specs/02-requirements.md`（v1.1）、`specs/constitution.md`（v1.0）
- **版本**：v1.0.1
- **日期**：2026-09-23
- **状态**：**草稿，待人工审查**（依 `constitution.md` 第四条：未经审查的产出不得作为下游输入）
- **本次修订**：按人工审查提出的 7 项核查逐条自检（见**附录 B 审查项自检对照**）：① §5.10 补「超期为派生状态、不建列」的显式说明；② 不新增 `barcode` 列并登记 `DB-ASM-09`；③ 命名口径差异登记 `DB-ASM-10`
- **编号约定**：表名 `snake_case` 复数；字段名 `snake_case`；索引 `uk_` / `idx_` 前缀；约束 `pk_` / `fk_` / `ck_` 前缀；设计决策 `DB-D-x`、待确认项 `DB-ASM-x`；业务规则引用 `BR-xxx`、需求引用 `FR-xxx`、用例 `UC-xx`
- **约束**：本文件受 `constitution.md` 约束；术语与枚举严格遵循 `02` 附录 A，**逐字一致**（第二条第 4 款）；「不做」清单（`02` §2.3、第十条）中的概念**禁止**出现在数据库设计中；逻辑删除一律保留历史（第十条第 4 款）
- **留痕**：本次由 Agent 生成，须在 `19-ai-usage-log.md` 登记一条记录（NFR-007 / 第四条第 5 款）；本轮人类指令限定**仅修改本文件**，故登记条目**尚未写入**（见 §10 `DB-ASM-08`）

---

## 0. 冲突声明（依 `constitution.md` 第一条第 2 款、第十三条，须人类裁决）

| # | 冲突点 | 上游口径 | 本次处理 | 登记 |
|---|---|---|---|---|
| 1 | 规则配置表命名 | `constitution.md` 第七条第 3 款写作 `borrowing_rule` / `fine_rule` | 本次按人类指令采用 `borrow_policies` / `fine_rules`；**语义与用途完全一致**（配置表存储、系统管理员运行时可改、代码与 SQL 禁硬编码数值）。名称差异不触及第二条需求基线，仅为表名差异 | `DB-ASM-01` |
| 2 | 管理员账号的表结构 | `05` §4.15 / §3.1：`AdminAccount` 为**单一实体**，以 `role` 枚举（`图书管理员 / 系统管理员`）区分，**无继承** | 本次按人类指令拆分为 `librarians` / `system_admins` 两张**同构**表（等价「每个具体子类一张表」，见 §7.2）。领域层仍只有 `AdminAccount` 一个类，拆分只发生在持久层 | `DB-ASM-02` |
| 3 | `item_type` 是否落库 | `05` `H-3`：`ItemType` 由「子类 + `language`」推导，未明确落库 | 落为**派生冗余列**（不可人工录入），用于 `FineRule` 查找与检索过滤；与 `H-3` 的「来源是推导而非录入」不冲突，但物化策略须确认 | `DB-ASM-03` |
| 4 | 留痕要求 | 第四条第 5 款：每次生成须登记 `19-ai-usage-log.md` | 本轮人类指令限定仅修改本文件，登记条目**未写入**；在完成登记前本文件**不得**作为下游文档与 `src/` 的输入（第四条第 1 款） | `DB-ASM-08` |

> 以上 4 项在裁决前**不得**被视为已确认结论；裁决结果须同步回写本文件与 `07` / `16` / `src/`。

---

## 1. 文档信息

| 项 | 内容 |
|---|---|
| 文档编号 | `13-database-design.md` |
| 上游基线 | `05-domain-model.md`（v1.1）、`07-architecture.md`（v1.0）、`06-domain-class-diagram.puml`、`02-requirements.md`（v1.1）、`constitution.md`（v1.0） |
| 下游影响 | `09-design-model.md`（持久化设计类）、`15-test-plan.md`（H2 种子数据）、`16-tasks.md`、`src/main/resources`（`schema.sql` / `data.sql`）、`src/main/java/.../infrastructure/persistence`（PO 与 Mapper） |
| 目标方言 | **H2 唯一**（`NFR-002`、第十二条第 3 款）；运行 `jdbc:h2:file:./data/library`，测试 `jdbc:h2:mem:library`，**禁止混用**（`R-8`） |
| 访问方式 | Spring Data JPA + 手写 `*PO` / `*Mapper`（`07` §4.4、`ASM-04`）；规则计算**禁止**写入 SQL / 存储过程 / 触发器（`07` §10.2） |
| 表总数 | 15 张（业务 14 + 审计 1） |
| 同步要求 | `13` 的持久化对象须与 `07` §5.2 的 `infrastructure.persistence` 对齐（`07` §1）；PO 命名对照见 §7.5 |

---

## 2. 设计依据与约束

| 约束 | 内容 | 依据 |
|---|---|---|
| 单一方言 | 全部 DDL 以 H2 2.x 语法书写；禁 MySQL / PostgreSQL 兼容层 | `NFR-002` 第 5 款、第十二条第 3 款 |
| 术语逐字一致 | 枚举列以**中文术语原文**存储（如 `在馆`、`中文图书`、`专科生`），并以 `CHECK` 约束取值域 | 第二条第 4 款 |
| 不建字典表 | 枚举一律「列 + CHECK」，**禁止**新建 `reader_type_dict` 之类码表 | `05` §3.4 |
| 规则不落 SQL | 日费率、借期、可借数量、封顶 50 元、保留 3 天等**禁止**出现在 CHECK / 触发器 / 视图 / 存储过程 | 第七条第 3 款、`07` §10.2 |
| 规则数值来源 | 只存于 `borrow_policies` / `fine_rules`，由种子数据初始化，系统管理员运行时可改 | `BR-014`、`FR-020` |
| 逻辑删除 | 读者 / 借阅证 / 标题 / 副本 / 管理员账号一律逻辑删除，**禁止**物理删除；`ON DELETE RESTRICT` | 第十条第 4 款、`NFR-008` |
| 只追加 | `audit_logs` 只 INSERT，不建 `version` / `updated_at`，不建外键 | `BR-030`、`FR-005` |
| 派生状态不落库 | 「未缴罚款」「黑名单」「在借副本数」「队首」「未缴累计额」不建列 | `05` §4.1、`H-1` |
| 不做清单 | 不建「损坏」状态、续借字段、挂失 / 冻结字段、催还通知表、报表统计表、支付流水表、会话表 | `02` §2.3、第十条 |
| 并发控制 | 需要并发保护的表带 `version`（`@Version`）+ 条件更新；关键唯一性以唯一索引兜底 | `07` §9.5、`NFR-009` |

---

## 3. 持久化对象识别（领域类 → 关系模型）

### 3.1 需持久化的领域类（`05` §3.1、§4）

| 领域类 | 构造型 | 持久化形态 | 对应表 | 依据 |
|---|---|---|---|---|
| `Reader`（抽象）+ `StudentReader`（抽象）+ 4 个学生子类 + `TeacherReader` | 实体 / 聚合根 | 单表 + 判别列 | `readers` | `05` §4.1~§4.3、`BR-026` 逻辑删除 |
| `BorrowCard` | 实体（`Reader` 聚合内成员） | 独立表（`reader_id` 唯一） | `borrow_cards` | `BR-024`、`BR-027`；虽与 `Reader` 同聚合（`H-4`），仍以独立表承载 1:1 |
| `AdminAccount`（角色 = 图书管理员） | 实体 / 聚合根 | 按角色分表 | `librarians` | `05` §4.15、`BR-028` |
| `AdminAccount`（角色 = 系统管理员） | 实体 / 聚合根 | 按角色分表 | `system_admins` | `05` §4.15、`BR-028` |
| `LibraryItem`（抽象） | 实体 / 聚合根 | 父表 | `library_items` | `05` §4.5、`BR-031` |
| `Book` | 实体（子类） | 子类表 | `book_titles` | `05` §4.6 |
| `Magazine` | 实体（子类） | 子类表 | `magazines` | `05` §4.7 |
| `Thesis` | 实体（子类） | 子类表 | `theses` | `05` §4.8 |
| `ItemCopy` | 实体（聚合内成员） | 独立表 | `item_copies` | `05` §4.9、`BR-019` |
| `Loan` | 实体 / 聚合根 | 独立表 | `loans` | `05` §4.10、`FR-009` |
| `Reservation` | 实体 / 聚合根 | 独立表 | `reservations` | `05` §4.11、`BR-015` |
| `BorrowPolicy` | 策略 + 配置型聚合根 | 配置表（5 行） | `borrow_policies` | `05` §4.12、`FR-020` |
| `FineRule` | 策略 + 配置型聚合根 | 配置表（5 行） | `fine_rules` | `05` §4.13、`FR-020` |
| `FineRecord` | 实体 / 聚合根 | 独立表 | `fine_records` | `05` §4.14、`BR-032` |
| `AuditLog` | 实体 / 聚合根（只追加） | 独立表 | `audit_logs` | `05` §4.16、`BR-030` |

### 3.2 明确**不**持久化为表的领域概念

| 概念 | 处理 | 依据 |
|---|---|---|
| 值对象 `Money` | 落为 `DECIMAL(10,2)` 列（定点小数，禁浮点） | `05` §5 |
| 值对象 `CardNumber` / `CopyNumber` / `ISBN` | 落为列 + 唯一约束 | `BR-027`、`BR-019`、`FR-006` |
| 值对象 `DateRange` / `OverdueDays` | 落为 `DATE` 列 / 不落库（计算得出） | `05` §5、`BR-007` |
| 枚举 `ReaderType` / `ItemType` / `CopyStatus` / `CardStatus` / `ReservationStatus` / `FineType` / `FineStatus` / `Role` | 落为 `VARCHAR` 列 + `CHECK`，**不建码表** | `05` §3.4 |
| 派生状态 `hasUnpaidFine()` / `isBlacklisted()` / 在借副本数 / 未缴累计额 | **不建列**，由 `fine_records`、`loans` 聚合查询 | `05` §4.1、`H-1` |
| `ReservationQueue`（队列容器） | **不建表**；队列 = `reservations` 中「在队」行的有序集合 | `05` §4.11 第 1 条 |
| 领域服务 `BorrowingService` 等 6 个 + 3 个接口 | 无状态，不持久化 | `05` §6 |
| `SessionToken` / 会话 | 属应用层，不建会话表 | `05` §2.3、§3.4、`FR-004` |
| 检索结果 / 借阅与罚款汇总 | 读模型 DTO，不建视图、不建物化表 | `05` §3.4 |
| 纯技术列 `created_at` / `updated_at` | **默认不加**；仅在领域属性明确要求处保留（如 `AdminAccount.createdAt`、`AuditLog.occurredAt`），并发保护用 `version` | `05` §3.4、`07` §9.5 |

---

## 4. 关系模型总览

### 4.1 表清单（15 张）

| # | 表名 | 对应领域类 | 模块（`07` §6） | 说明 |
|---|---|---|---|---|
| 1 | `readers` | `Reader` 继承体系 | `reader` | 单表继承，判别列 `reader_type` |
| 2 | `borrow_cards` | `BorrowCard` | `card` | 与 `readers` 1:1（`reader_id` 唯一） |
| 3 | `librarians` | `AdminAccount`（`role = 图书管理员`） | `admin` | 账号表（每个具体角色一张表） |
| 4 | `system_admins` | `AdminAccount`（`role = 系统管理员`） | `admin` | 账号表（每个具体角色一张表） |
| 5 | `library_items` | `LibraryItem`（抽象父） | `catalog` | JOINED 父表，承载检索与预约关联 |
| 6 | `book_titles` | `Book` | `catalog` | JOINED 子类表 |
| 7 | `magazines` | `Magazine` | `catalog` | JOINED 子类表 |
| 8 | `theses` | `Thesis` | `catalog` | JOINED 子类表 |
| 9 | `item_copies` | `ItemCopy` | `catalog` | 五态状态机 |
| 10 | `loans` | `Loan` | `circulation` | 借还事实 + `due_date` 快照 |
| 11 | `reservations` | `Reservation` | `reservation` | FIFO 队列行 |
| 12 | `borrow_policies` | `BorrowPolicy` | `circulation` | 配置表，种子 5 行 |
| 13 | `fine_rules` | `FineRule` | `fine` | 配置表，种子 5 行 |
| 14 | `fine_records` | `FineRecord` | `fine` | 超期 / 赔偿共用结清路径 |
| 15 | `audit_logs` | `AuditLog` | `admin` | 只追加 |

### 4.2 外键关系矩阵

| 子表.列 | → 父表.列 | 基数 | 删除策略 | 说明 |
|---|---|---|---|---|
| `borrow_cards.reader_id` | `readers.id` | 1:1 | RESTRICT | 注册即发证、同生同死（`BR-024`） |
| `item_copies.item_id` | `library_items.id` | N:1 | RESTRICT | 标题是副本容器（`BR-019`） |
| `book_titles.item_id` | `library_items.id` | 1:1（子类） | RESTRICT | JOINED 连接键，同时是子类表主键 |
| `magazines.item_id` | `library_items.id` | 1:1（子类） | RESTRICT | 同上 |
| `theses.item_id` | `library_items.id` | 1:1（子类） | RESTRICT | 同上 |
| `loans.reader_id` | `readers.id` | N:1 | RESTRICT | 历史借阅须可追溯已注销读者（`BR-026`） |
| `loans.copy_id` | `item_copies.id` | N:1 | RESTRICT | 历史借阅须可追溯报废副本（`NFR-008`） |
| `loans.operator_id` | `librarians.id` | N:1 | RESTRICT | 办理借还的图书管理员（`U2`） |
| `reservations.reader_id` | `readers.id` | N:1 | RESTRICT | 预约人 |
| `reservations.item_id` | `library_items.id` | N:1 | RESTRICT | 预约对象是**标题**不是副本（`BR-015`） |
| `fine_records.reader_id` | `readers.id` | N:1 | RESTRICT | 欠款人 |
| `fine_records.loan_id` | `loans.id` | N:0..1 | RESTRICT | 仅 `类型 = 超期` 有值 |
| `fine_records.copy_id` | `item_copies.id` | N:0..1 | RESTRICT | 仅 `类型 = 赔偿` 有值 |
| （无外键） | `borrow_policies` / `fine_rules` | — | — | **不被外键引用**：按枚举键查找（`reader_type` / `item_type`），跨聚合只以标识引用（`05` §3.3） |
| （无外键） | `audit_logs.actor_id` / `target_id` | — | — | 弱引用，保留已注销读者 / 报废副本的历史（`FR-005` 验收 4） |
| （无外键） | `librarians` / `system_admins` 被引用 | — | — | 仅 `loans.operator_id` 引用 `librarians`（见 §5.10 说明） |

> 全部外键均为 `RESTRICT`：**禁止**物理删除主表记录（`NFR-008`、第十条第 4 款）；「删除」一律通过状态列 / `deleted` 标记完成。

---

## 5. 表结构详细设计

> 字段表列头统一为：**字段名 / 类型（H2） / 是否为空 / 主键 / 外键 / 唯一约束 / 默认值 / 说明**。
> `U` = 单列唯一约束；`U1…` = 组合唯一约束（名称见每张表下方的约束清单）；`—` = 无。

---

### 5.1 `readers`（读者）

**领域来源**：`Reader`（抽象）+ `StudentReader`（抽象）+ `JuniorCollegeReader` / `UndergraduateReader` / `MasterReader` / `DoctoralReader` + `TeacherReader`（`05` §4.1~§4.3）
**继承策略**：单表继承（SINGLE_TABLE），判别列 = `reader_type`（见 §7.2）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 读者标识 `ReaderId` |
| `username` | `VARCHAR(50)` | NOT NULL | — | — | U | — | 登录用户名，全局唯一（`FR-003`） |
| `password_hash` | `CHAR(60)` | NOT NULL | — | — | — | — | BCrypt 哈希，定长 60，**禁存明文**（`BR-029`、NFR-004） |
| `name` | `VARCHAR(50)` | NOT NULL | — | — | — | — | 姓名，与借阅证一致（`BR-027`） |
| `department` | `VARCHAR(100)` | NOT NULL | — | — | — | — | 系别（`BR-027`） |
| `reader_type` | `VARCHAR(10)` | NOT NULL | — | — | — | — | **继承判别列 + `BorrowPolicy` 查找键**；CHECK ∈ 专科生 / 本科生 / 研究生 / 博士生 / 教师（`BR-001`、`FR-001` 验收 6） |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `正常` | `ReaderStatus`：正常 / 已注销；逻辑删除（`BR-026`、第十条第 4 款） |
| `student_no` | `VARCHAR(32)` | NULL | — | — | — | — | 学号（学生子类）；教师读者恒为 NULL |
| `grade` | `VARCHAR(32)` | NULL | — | — | — | — | 年级 / 培养层次描述（可选） |
| `employee_no` | `VARCHAR(32)` | NULL | — | — | — | — | 教职工号（教师子类，可选）；学生读者恒为 NULL |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（类型修正 / 注销并发，`07` §9.5） |

**约束与说明**

- `ck_readers_type`：`reader_type IN ('专科生','本科生','研究生','博士生','教师')`
- `ck_readers_status`：`status IN ('正常','已注销')`
- `ck_readers_identity`：`(reader_type = '教师' AND student_no IS NULL) OR (reader_type <> '教师' AND employee_no IS NULL)` —— 保证判别列与子类身份列不出现「两类身份并存」
- `uk_readers_username`：`username` 唯一
- **禁止**列：`max_copies` / `loan_period_days`（一律查 `borrow_policies`，`BR-014`）、`has_unpaid_fine` / `blacklisted`（派生状态，`H-1`）、`valid_until`（借阅证无有效期，`BR-025`）
- 跨子类类型修正（如本科生 → 教师）在 DB 层即 `UPDATE readers SET reader_type='教师', student_no=NULL, employee_no=? WHERE id=?`，`id` 与 `borrow_cards` 不变 —— 与 `05` §3.2 D-1 第 3 条一致

---

### 5.2 `borrow_cards`（借阅证）

**领域来源**：`BorrowCard`（`05` §4.4）。与 `Reader` 同属一个聚合（`H-4`），此处以独立表承载 1:1，写入仍随 `Reader` 聚合在同一事务内完成（`BR-024`）。

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 借阅证标识 `BorrowCardId` |
| `reader_id` | `BIGINT` | NOT NULL | — | → `readers(id)` | U | — | 归属读者，1:1 |
| `card_number` | `VARCHAR(20)` | NOT NULL | — | — | U | — | 借阅证号 `CardNumber`，系统生成、全局唯一（建议 `L`+年份+6 位序列，`TBD-010`） |
| `owner_name` | `VARCHAR(50)` | NOT NULL | — | — | — | — | 证面姓名（`BR-027`） |
| `department` | `VARCHAR(100)` | NOT NULL | — | — | — | — | 证面系别（`BR-027`） |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `正常` | `CardStatus`：正常 / 已注销（`BR-025`、`FR-002`） |

**约束与说明**

- `ck_cards_status`：`status IN ('正常','已注销')` —— **严格两态**
- `uk_cards_reader`：`reader_id` 唯一（保证 1:1）
- `uk_cards_number`：`card_number` 唯一
- **禁止**列：`valid_until` / `expire_date`（无有效期，`BR-025`）、`挂失` / `冻结` / `补办` 相关列（`R-12`、`U1`）

---

### 5.3 `librarians`（图书管理员账号）

**领域来源**：`AdminAccount`，`role = 图书管理员`（`05` §4.15）。持久层按角色分表（TABLE_PER_CLASS 等价形态，见 §7.2）。

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 管理员标识 `AdminAccountId` |
| `username` | `VARCHAR(50)` | NOT NULL | — | — | U | — | 登录用户名（表内唯一；**跨管理员表唯一**由应用层保证，见 `DB-ASM-02`） |
| `password_hash` | `CHAR(60)` | NOT NULL | — | — | — | — | BCrypt 哈希（`BR-029`） |
| `role` | `VARCHAR(10)` | NOT NULL | — | — | — | `图书管理员` | `Role` 枚举常量列，与表名互为校验；供审计 `actor_role` 取用 |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `正常` | `AdminStatus`：正常 / 已停用；逻辑删除（`BR-030`） |
| `created_at` | `TIMESTAMP` | NOT NULL | — | — | — | `CURRENT_TIMESTAMP` | 创建时间（`05` §4.15 领域属性） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（账号维护并发，`07` §9.5） |

**约束与说明**

- `ck_librarians_role`：`role = '图书管理员'`
- `ck_librarians_status`：`status IN ('正常','已停用')`
- `uk_librarians_username`：`username` 唯一
- 权限语义（`canBorrow()` / `canReserve()` 恒 `false`）在**领域层**判定，DB 不建权限列（`BR-028`、`07` §7.6）

---

### 5.4 `system_admins`（系统管理员账号）

**领域来源**：`AdminAccount`，`role = 系统管理员`（`05` §4.15）。与 `librarians` 同构。

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 管理员标识 `AdminAccountId` |
| `username` | `VARCHAR(50)` | NOT NULL | — | — | U | — | 登录用户名（表内唯一） |
| `password_hash` | `CHAR(60)` | NOT NULL | — | — | — | — | BCrypt 哈希（`BR-029`） |
| `role` | `VARCHAR(10)` | NOT NULL | — | — | — | `系统管理员` | `Role` 枚举常量列 |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `正常` | 正常 / 已停用；逻辑删除 |
| `created_at` | `TIMESTAMP` | NOT NULL | — | — | — | `CURRENT_TIMESTAMP` | 创建时间 |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁 |

**约束与说明**

- `ck_admins_role`：`role = '系统管理员'`
- `ck_admins_status`：`status IN ('正常','已停用')`
- `uk_admins_username`：`username` 唯一
- 「不可删除最后一个系统管理员」（`FR-022` 验收 4）在**领域层**校验：`SELECT COUNT(*) FROM system_admins WHERE status='正常'` 须 > 1 才允许停用；**禁止**用触发器实现（第七条第 3 款）
- 初始账号由种子数据提供（`TBD-008`，见 §9）

---

### 5.5 `library_items`（图书标题 —— JOINED 父表）

**领域来源**：`LibraryItem`（抽象，`05` §4.5）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 标题标识 `LibraryItemId` |
| `title` | `VARCHAR(200)` | NOT NULL | — | — | — | — | 书名 / 题名；检索字段（`FR-023`） |
| `author` | `VARCHAR(100)` | NOT NULL | — | — | — | — | 作者；检索字段（`FR-023`） |
| `isbn` | `VARCHAR(32)` | NOT NULL | — | — | U1 | — | `ISBN` 值对象；与 `item_type` 联合去重（`FR-006` 验收 7） |
| `item_type` | `VARCHAR(10)` | NOT NULL | — | — | U1 | — | **派生冗余列**：由子类 + `language` 推导后写入（`H-3`），是 `FineRule` 的查找键（`BR-007`、`BR-031`） |
| `deleted` | `BOOLEAN` | NOT NULL | — | — | — | `FALSE` | 逻辑删除标记（`TBD-012`）；下架后仍可追溯历史（`NFR-008`） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（维护并发，`07` §9.5） |

**约束与说明**

- `ck_items_type`：`item_type IN ('中文图书','外文图书','中文杂志','外文杂志','论文')`
- `uk_items_isbn_type`：`(isbn, item_type)` 唯一 —— 同一 ISBN + 同一借出物类型不得重复录入；中外文版是**两个**标题，故二者可并存（`D2`、`BR-031`）
- `item_type` 与子类的一致性（`Book` → 中文图书 / 外文图书 等）由 `LibraryItemFactory` 在写入时保证；DB 仅约束取值域，不以触发器强制（避免规则落 SQL，`07` §10.2）
- 索引见 §6；检索为**精确或前缀**匹配，**不做**分词、相关性排序与全文索引（`FR-023`、`00` §8）

---

### 5.6 `book_titles`（图书 —— JOINED 子类表）

**领域来源**：`Book`（`05` §4.6）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `item_id` | `BIGINT` | NOT NULL | PK | → `library_items(id)` | — | — | 连接键（JOINED）：既是本表主键也是父表外键 |
| `language` | `VARCHAR(10)` | NOT NULL | — | — | — | — | 语言：中文 / 外文；与子类共同推导 `item_type`（`BR-031`） |
| `publisher` | `VARCHAR(100)` | NULL | — | — | — | — | 出版社 |
| `publish_year` | `INT` | NULL | — | — | — | — | 出版年（可选） |

**约束与说明**

- `ck_books_language`：`language IN ('中文','外文')`
- 与父表的 `item_type` 派生关系：`language = '中文' → 中文图书`，`language = '外文' → 外文图书`
- 日费率分别为 2 / 2.5 元每天，但**不**在本表落任何费率列（一律查 `fine_rules`，`BR-014`）

---

### 5.7 `magazines`（杂志 —— JOINED 子类表）

**领域来源**：`Magazine`（`05` §4.7）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `item_id` | `BIGINT` | NOT NULL | PK | → `library_items(id)` | — | — | 连接键 |
| `language` | `VARCHAR(10)` | NOT NULL | — | — | — | — | 中文 / 外文；推导 `item_type` 为中文杂志 / 外文杂志 |
| `issue` | `VARCHAR(32)` | NULL | — | — | — | — | 期号（可选） |
| `periodical_no` | `VARCHAR(32)` | NULL | — | — | — | — | 刊号（可选） |

**约束与说明**：`ck_magazines_language`：`language IN ('中文','外文')`；借期与可借数量与杂志无关（`BR-002`），本表不含任何借期列。

---

### 5.8 `theses`（论文 —— JOINED 子类表）

**领域来源**：`Thesis`（`05` §4.8）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `item_id` | `BIGINT` | NOT NULL | PK | → `library_items(id)` | — | — | 连接键 |
| `department` | `VARCHAR(100)` | NULL | — | — | — | — | 所属系别 / 学位授予单位 |
| `degree_year` | `INT` | NULL | — | — | — | — | 年份 |
| `supervisor` | `VARCHAR(50)` | NULL | — | — | — | — | 指导教师（可选） |

**约束与说明**：本表无 `language` 列 —— `item_type` 恒为 `论文`（`05` §4.8）。

---

### 5.9 `item_copies`（馆藏副本）

**领域来源**：`ItemCopy`（`05` §4.9）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 副本标识 `ItemCopyId` |
| `item_id` | `BIGINT` | NOT NULL | — | → `library_items(id)` | — | — | 所属标题（N:1，`BR-019`） |
| `copy_number` | `VARCHAR(32)` | NOT NULL | — | — | U | — | `CopyNumber`，**全局唯一**（`BR-019`、`FR-007` 验收 1） |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `在馆` | `CopyStatus` 五态：在馆 / 已借出 / 预约保留 / 遗失 / 报废（`BR-020`） |
| `reserved_until` | `DATE` | NULL | — | — | — | — | `预约保留` 到期时间 = 归还日期 + 3 天（`BR-017`）；非保留态恒 NULL |
| `deleted` | `BOOLEAN` | NOT NULL | — | — | — | `FALSE` | 逻辑删除（`TBD-012`） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁；配合条件更新 `WHERE id=? AND status=?` 防并发借出（`07` §9.5、`NFR-009` 第 3 条） |

**约束与说明**

- `ck_copies_status`：`status IN ('在馆','已借出','预约保留','遗失','报废')` —— **不含「损坏」**（`BR-020`、`R-5`）
- `ck_copies_hold`：`(status = '预约保留' AND reserved_until IS NOT NULL) OR (status <> '预约保留' AND reserved_until IS NULL)`
- `uk_copies_number`：`copy_number` 唯一
- 状态**迁移合法性**（报废终态、遗失不可回在馆、已借出不可删）由 `ItemCopy` 状态方法在领域层保证（`05` §8.1）；**禁止**用触发器实现（规则不得落 SQL，第七条第 3 款）
- 「3 天保留期」「报废」等规则数值一律不落 CHECK（`07` §10.2）

---

### 5.10 `loans`（借阅记录）

**领域来源**：`Loan`（`05` §4.10）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 借阅记录标识 `LoanId` |
| `reader_id` | `BIGINT` | NOT NULL | — | → `readers(id)` | — | — | 借阅人（N:1） |
| `copy_id` | `BIGINT` | NOT NULL | — | → `item_copies(id)` | — | — | 所借副本（N:1） |
| `borrow_date` | `DATE` | NOT NULL | — | — | — | `CURRENT_DATE` | 借阅日期 |
| `due_date` | `DATE` | NOT NULL | — | — | — | — | 应还日期 = `borrow_date + BorrowPolicy[reader_type].loanPeriodDays`，借书时**快照**（`BR-001`、`05` §4.12 D-2） |
| `return_date` | `DATE` | NULL | — | — | — | — | 归还日期；NULL = 尚未归还 |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `在借` | `LoanStatus`：在借 / 已还；**与 `return_date` 互为充要**（`05` §4.10） |
| `operator_id` | `BIGINT` | NOT NULL | — | → `librarians(id)` | — | — | 办理借书的图书管理员（`U2`） |
| `open_copy_key` | `BIGINT`（计算列） | NULL | — | — | U | — | `GENERATED ALWAYS AS (CASE WHEN return_date IS NULL THEN copy_id END)`；保证「同一副本同一时刻至多 1 条未还记录」（`NFR-009` 第 3 条） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（并发归还 / 计提写入） |

**约束与说明**

- `ck_loans_status`：`status IN ('在借','已还')`
- `ck_loans_return`：`(status = '在借' AND return_date IS NULL) OR (status = '已还' AND return_date IS NOT NULL)` —— 状态不变量
- `ck_loans_dates`：`due_date >= borrow_date AND (return_date IS NULL OR return_date >= borrow_date)`
- **超期为派生状态，不建列**：`超期` = `status = '在借' AND due_date < 当前日期`，由 `Loan.isOverdue(today)` / `overdueDays(today)` 计算（`05` §4.10、`BR-007`）；理由是超期为「当前日期」与 `due_date` 的函数，落列需每日批量更新且会与 `status` 形成**第二处真相**（`H-1`）。查询侧由 `idx_loans_status_due (status, due_date)` 支撑「在借且已超期」扫描（`UC-21`），计提幂等由 `fine_records.uk_fine_loan_date` 保证（`NFR-009` 第 1 款）
- `uk_loans_open_copy`：`open_copy_key` 唯一（NULL 不参与唯一性判定，已还记录可有多条）
- **禁止**列：`renew_times` / 续借相关列（`BR-006`、`L3`）、`fine_amount`（归还不结算罚款，`BR-009`、`R-9`）
- 不支持续借 ⇒ **不存在**任何会更新 `due_date` 的写路径（`BR-006`）
- `operator_id` 外键指向 `librarians`：借还只能由图书管理员办理（`BR-028`、`07` §7.4）

---

### 5.11 `reservations`（预约）

**领域来源**：`Reservation`（`05` §4.11）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 预约标识 `ReservationId` |
| `reader_id` | `BIGINT` | NOT NULL | — | → `readers(id)` | — | — | 预约人（N:1） |
| `item_id` | `BIGINT` | NOT NULL | — | → `library_items(id)` | — | — | 预约的**标题**（N:1，非副本，`BR-015`） |
| `reserved_at` | `TIMESTAMP` | NOT NULL | — | — | — | `CURRENT_TIMESTAMP` | 预约发起时间；FIFO 原始次序与同位次时的稳定次序（`BR-016`） |
| `queue_position` | `INT` | NOT NULL | — | — | — | — | 队列位次（排序键，可被系统管理员重排，`FR-016`）；调整**不改** `reserved_at` |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `等待中` | 等待中 / 保留中 / 已完成 / 已取消 / 已失效；前两者为**在队（有效）** |
| `hold_until` | `DATE` | NULL | — | — | — | — | `保留中` 时的保留到期时间（`BR-017`） |
| `active_slot` | `VARCHAR(64)`（计算列） | NULL | — | — | U | — | `GENERATED ALWAYS AS (CASE WHEN status IN ('等待中','保留中') THEN reader_id || ':' || item_id END)`；防并发重复预约（`05` §4.11 第 6 条、`UC-06 E5`） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（队列重排与到期顺延并发，`05` 附录 A 遗留风险 2） |

**约束与说明**

- `ck_resv_status`：`status IN ('等待中','保留中','已完成','已取消','已失效')`
- `ck_resv_hold`：`(status = '保留中' AND hold_until IS NOT NULL) OR (status <> '保留中' AND hold_until IS NULL)`
- `uk_resv_active_slot`：`active_slot` 唯一 —— 同一 `reader_id` + `item_id` 至多 1 条在队预约；已取消 / 已失效 / 已完成行为 NULL，可并存多条历史
- **不唯一**：`queue_position` 在标题内**不**建唯一约束 —— 队列重排过程中允许瞬时重复，最终一致性由事务 + 乐观锁保证
- **不建** `reservation_queues` 表：队列 = 同标题下在队行的有序集合（`05` §4.11 第 1 条）
- 单人有效预约上限 3（`BR-018`）在**领域层**计数校验，DB 不建行数约束（规则不得落 SQL，`07` §10.2）

---

### 5.12 `borrow_policies`（借阅规则配置）

**领域来源**：`BorrowPolicy`（`05` §4.12）；即 `constitution` 第七条第 3 款所称 `borrowing_rule`（见 §0 冲突 1）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 规则行标识 `BorrowPolicyId` |
| `reader_type` | `VARCHAR(10)` | NOT NULL | — | — | U | — | 唯一键，按读者类型分档，共 5 行（`BR-001`） |
| `max_copies` | `INT` | NOT NULL | — | — | — | — | 最大可借副本数（种子：3 / 5 / 7 / 10 / 15） |
| `loan_period_days` | `INT` | NOT NULL | — | — | — | — | 借阅期限天数（种子：15 / 30 / 30 / 60 / 60） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（规则并发修改，`07` §9.5） |

**约束与说明**

- `ck_policy_type`：`reader_type IN ('专科生','本科生','研究生','博士生','教师')`
- `ck_policy_values`：`max_copies >= 0 AND loan_period_days >= 0`（非负校验，`FR-020`、`UC-18 E2`）
- `uk_policy_reader_type`：`reader_type` 唯一
- **只按读者类型分档**，不按借出物类型（`BR-002`）⇒ 无 `item_type` 列
- **不建** `effective_from` / `deleted` 列：规则变更**即时生效**、只改值不删行；`due_date` 在借书时快照进 `loans`，变更不追溯已借记录（`05` §4.12 D-2、`FR-020` 验收 3）
- 修改必须写审计（应用层保证，`FR-020` 验收 6）

---

### 5.13 `fine_rules`（罚款规则配置）

**领域来源**：`FineRule`（`05` §4.13）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 规则行标识 `FineRuleId` |
| `item_type` | `VARCHAR(10)` | NOT NULL | — | — | U | — | 唯一键，按借出物类型分档，共 5 行（`BR-007`） |
| `daily_rate` | `DECIMAL(10,2)` | NOT NULL | — | — | — | — | 日费率（元/天；种子：2 / 2.5 / 1 / 1.5 / 3）；定点小数，禁浮点 |
| `grace_period_days` | `INT` | NOT NULL | — | — | — | `0` | 宽限期，恒为 0（`BR-007`、`02` 附录 A） |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁 |

**约束与说明**

- `ck_fine_rule_type`：`item_type IN ('中文图书','外文图书','中文杂志','外文杂志','论文')`
- `ck_fine_rule_values`：`daily_rate >= 0 AND grace_period_days >= 0`
- `uk_fine_rule_item_type`：`item_type` 唯一
- **不区分读者类型**（`BR-008`）⇒ 无 `reader_type` 列、无教师减免列
- **5 行必须齐全**：缺任一 `item_type` 时计提任务失败并留痕，**禁止**回退默认费率或按 0 计提（`UC-21 E1`、`05` §4.13 D-3）
- **实时读取**语义：每次计提读当前行，规则变更后**下一次计提**即生效，已生成的 `fine_records` 不追溯重算（`05` §4.13 D-3）

---

### 5.14 `fine_records`（罚款 / 款项记录）

**领域来源**：`FineRecord`（`05` §4.14）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 款项标识 `FineRecordId` |
| `reader_id` | `BIGINT` | NOT NULL | — | → `readers(id)` | — | — | 欠款人（N:1） |
| `type` | `VARCHAR(10)` | NOT NULL | — | — | — | — | `FineType`：超期 / 赔偿（`BR-032`） |
| `amount` | `DECIMAL(10,2)` | NOT NULL | — | — | — | — | 原始金额（`Money`，定点小数） |
| `unpaid_amount` | `DECIMAL(10,2)` | NOT NULL | — | — | — | — | 未缴余额；减免后减少，减至 0 视为结清（`FR-019`） |
| `status` | `VARCHAR(10)` | NOT NULL | — | — | — | `未缴` | `FineStatus`：未缴 / 已缴（`BR-012`） |
| `accrual_date` | `DATE` | NOT NULL | — | — | U1 | `CURRENT_DATE` | 计提日（超期）/ 登记日（赔偿） |
| `settled_at` | `TIMESTAMP` | NULL | — | — | — | — | 结清时间（`FR-018`） |
| `loan_id` | `BIGINT` | NULL | — | → `loans(id)` | U1 | — | 仅 `类型 = 超期` 有值 |
| `copy_id` | `BIGINT` | NULL | — | → `item_copies(id)` | — | — | 仅 `类型 = 赔偿` 有值 |
| `version` | `INT` | NOT NULL | — | — | — | `0` | 乐观锁（并发缴纳 / 减免） |

**约束与说明**

- `ck_fine_type`：`type IN ('超期','赔偿')`
- `ck_fine_status`：`status IN ('未缴','已缴')`
- `ck_fine_amount`：`amount >= 0 AND unpaid_amount >= 0 AND unpaid_amount <= amount`
- `ck_fine_settled`：`(status = '已缴' AND settled_at IS NOT NULL AND unpaid_amount = 0) OR (status = '未缴' AND settled_at IS NULL)`
- `ck_fine_link`：`(type = '超期' AND loan_id IS NOT NULL AND copy_id IS NULL) OR (type = '赔偿' AND copy_id IS NOT NULL AND loan_id IS NULL)`
- `uk_fine_loan_date`：`(loan_id, accrual_date)` 唯一 —— 同一自然日对同一 `Loan` **不重复计提**（`NFR-009` 第 1 款、`UC-21` 步骤 4）；赔偿行 `loan_id` 为 NULL，不受约束
- 未缴累计额 / 50 元封顶 / 黑名单**不落列**：由 `fine_records` 按 `(reader_id, type, status)` 聚合计算，其中封顶**只计** `超期`（`BR-010`、`BR-013`、`H-1`）
- 记账式自助结算：无支付流水表、无收银记录表（`00` §8、第十条第 2 款）

---

### 5.15 `audit_logs`（审计日志）

**领域来源**：`AuditLog`（`05` §4.16）

| 字段名 | 类型（H2） | 是否为空 | 主键 | 外键 | 唯一约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|---|
| `id` | `BIGINT` | NOT NULL | PK | — | — | `GENERATED BY DEFAULT AS IDENTITY` | 日志标识 `AuditLogId` |
| `actor_id` | `VARCHAR(64)` | NOT NULL | — | —（弱引用） | — | — | 操作者标识（读者 / 图书管理员 / 系统管理员 id）；**不建外键**，以保留已注销账号的可追溯性 |
| `actor_role` | `VARCHAR(10)` | NOT NULL | — | — | — | — | `Role`：读者 / 图书管理员 / 系统管理员 |
| `action_type` | `VARCHAR(32)` | NOT NULL | — | — | — | — | 操作类型：登录失败 / 借书 / 还书 / 预约 / 取消预约 / 调整预约队列 / 登记遗失赔偿 / 缴纳罚款 / 减免罚款 / 维护标题 / 维护副本 / 报废 / 维护管理员 / 修改规则（`05` §4.16） |
| `target_type` | `VARCHAR(32)` | NULL | — | — | — | — | 操作对象类型（弱引用） |
| `target_id` | `VARCHAR(64)` | NULL | — | — | — | — | 操作对象标识（弱引用，支持追溯已注销读者与报废副本，`FR-005` 验收 4） |
| `result` | `VARCHAR(10)` | NOT NULL | — | — | — | — | 成功 / 失败（`FR-005`） |
| `occurred_at` | `TIMESTAMP` | NOT NULL | — | — | — | `CURRENT_TIMESTAMP` | 发生时间 |

**约束与说明**

- `ck_audit_role`：`actor_role IN ('读者','图书管理员','系统管理员')`
- `ck_audit_result`：`result IN ('成功','失败')`
- `ck_audit_action`：`action_type IN (14 个取值，见上)` —— 新增操作类型须同步修订本文件与 `05` §4.16（`ASM-06`）
- **只追加**：不建 `version` / `updated_at`；业务代码**禁止**出现 UPDATE / DELETE（`BR-030`、`FR-005` 验收 3）；建议对应用账号仅授予 INSERT + SELECT
- 写入走**独立事务** `REQUIRES_NEW`（`07` §9.4），故本表不参与业务事务回滚
- 不落审计的操作（注册与自动发证、只读查询、定时计提）由应用层控制，**不**在 DB 层限制（`05` §4.16 第 2 条）

---

## 6. 索引设计

### 6.1 设计原则

1. **唯一性优先用约束表达**：业务不变式（用户名、借阅证号、副本编号、`(isbn, item_type)`、`(loan_id, accrual_date)`、在队预约互斥、同一副本未还记录互斥）一律以唯一索引落地 —— 它是并发下**最后一道防线**（`NFR-009`），应用层校验不可替代。
2. **查询驱动**：只为确定的查询路径建索引 —— 书目检索、在借校验、计提扫描、保留到期扫描、FIFO 队首判定、未缴汇总、审计追溯。
3. **复合索引列序**：等值列在前，范围 / 排序列在后（`07` §9.1 只读查询必须分页）。
4. **外键列必建索引**：H2 不会为外键自动建索引（`readers` 侧的历史追溯、`item_copies.item_id` 等）。
5. **低基数列不单独建索引**：`status` / `type` / `deleted` 只作为复合索引前导列或由应用层过滤；数据规模 `D3`（读者 5000、副本 10000、累计流水 10 万）下单独建索引收益极低。
6. **控制写放大**：单表二级索引 ≤ 4 个（日均借还 300，`02` 附录 A）。
7. **禁止**函数索引与全文索引（不做复杂全文检索，`FR-023`）。

### 6.2 索引清单

| 索引名 | 表 | 列（顺序） | 类型 | 支撑场景 | 依据 |
|---|---|---|---|---|---|
| `pk_readers`（隐式） | `readers` | `id` | PK | 主键查找 | — |
| `uk_readers_username` | `readers` | `username` | UNIQUE | 登录按用户名定位、用户名唯一（`FR-003`） | `FR-001` 验收 1 |
| `idx_readers_type` | `readers` | `reader_type` | INDEX | 按类型统计 / 名单筛选 | `BR-001` |
| `idx_readers_status` | `readers` | `status` | INDEX | 注销读者过滤（借书前置校验） | `BR-026`、`UC-09` |
| `pk_cards`（隐式） | `borrow_cards` | `id` | PK | 主键查找 | — |
| `uk_cards_reader` | `borrow_cards` | `reader_id` | UNIQUE | 1:1 约束（`BR-024`） | `H-4` |
| `uk_cards_number` | `borrow_cards` | `card_number` | UNIQUE | 借阅证号全局唯一（`FR-001` 验收 1） | `BR-027` |
| `uk_librarians_username` | `librarians` | `username` | UNIQUE | 登录定位 | `FR-003` |
| `uk_admins_username` | `system_admins` | `username` | UNIQUE | 登录定位 | `FR-003` |
| `pk_items`（隐式） | `library_items` | `id` | PK | 主键查找 | — |
| `uk_items_isbn_type` | `library_items` | `isbn`, `item_type` | UNIQUE | 同 ISBN + 同类型不得重复录入 | `FR-006` 验收 7 |
| `idx_items_title` | `library_items` | `title` | INDEX | 书名精确 / 前缀检索（`FR-023`、`UC-04`） | `05` §4.5 |
| `idx_items_author` | `library_items` | `author` | INDEX | 作者检索 | `FR-023` |
| `idx_items_isbn` | `library_items` | `isbn` | INDEX | ISBN 检索 | `FR-023` |
| `pk_copies`（隐式） | `item_copies` | `id` | PK | 主键查找 | — |
| `uk_copies_number` | `item_copies` | `copy_number` | UNIQUE | 副本编号全局唯一（`BR-019`） | `FR-007` 验收 1 |
| `idx_copies_item_status` | `item_copies` | `item_id`, `status` | INDEX | `hasAvailableCopy()`：某标题是否存在 `在馆` 副本（`BR-015` 反向判定） | `05` §4.5 |
| `idx_copies_reserved_until` | `item_copies` | `reserved_until` | INDEX | 保留到期扫描（`UC-22`） | `BR-017`、`FR-015` |
| `pk_loans`（隐式） | `loans` | `id` | PK | 主键查找 | — |
| `uk_loans_open_copy` | `loans` | `open_copy_key`（计算列） | UNIQUE | 同一副本至多 1 条未还记录（并发借出） | `NFR-009` 第 3 条 |
| `idx_loans_reader_status` | `loans` | `reader_id`, `status` | INDEX | 在借副本数上限校验、本人借阅查询（`UC-05`、`UC-11`） | `BR-004`、`FR-011` |
| `idx_loans_status_due` | `loans` | `status`, `due_date` | INDEX | 每日计提扫描「在借且已超期」（`UC-21`） | `BR-007`、`FR-017` |
| `idx_loans_copy` | `loans` | `copy_id` | INDEX | 按副本查借阅历史；同标题限 1 副本校验（`BR-003`） | `FR-010` |
| `idx_loans_borrow_date` | `loans` | `borrow_date` | INDEX | 借阅记录分页排序 | `FR-012` |
| `pk_resv`（隐式） | `reservations` | `id` | PK | 主键查找 | — |
| `uk_resv_active_slot` | `reservations` | `active_slot`（计算列） | UNIQUE | 防并发重复预约（同读者 + 同标题仅 1 条在队） | `05` §4.11 第 6 条、`UC-06 E5` |
| `idx_resv_queue` | `reservations` | `item_id`, `queue_position`, `reserved_at` | INDEX | **FIFO 队列核心索引**：取在队预约并排序、队首判定、队列重排 | `BR-016`、`05` §4.11 |
| `idx_resv_reader_status` | `reservations` | `reader_id`, `status` | INDEX | 单人有效预约 ≤ 3 计数、本人预约查询 | `BR-018`、`UC-07` |
| `idx_resv_hold_until` | `reservations` | `hold_until` | INDEX | 保留到期扫描与顺延（`UC-22`） | `BR-017`、`FR-015` |
| `pk_policy`（隐式） | `borrow_policies` | `id` | PK | 主键查找 | — |
| `uk_policy_reader_type` | `borrow_policies` | `reader_type` | UNIQUE | `BorrowPolicyProvider.forReaderType()` 唯一查找键 | `BR-001`、`BR-014` |
| `pk_fine_rule`（隐式） | `fine_rules` | `id` | PK | 主键查找 | — |
| `uk_fine_rule_item_type` | `fine_rules` | `item_type` | UNIQUE | `FineRuleProvider.forItemType()` 唯一查找键 | `BR-007`、`BR-014` |
| `pk_fine`（隐式） | `fine_records` | `id` | PK | 主键查找 | — |
| `uk_fine_loan_date` | `fine_records` | `loan_id`, `accrual_date` | UNIQUE | 同自然日幂等计提（`UC-21` 步骤 4） | `NFR-009` 第 1 款 |
| `idx_fine_reader_type_status` | `fine_records` | `reader_id`, `type`, `status` | INDEX | 未缴累计额、50 元封顶（仅 `超期`）、黑名单、`hasUnpaidFine()` | `BR-010`、`BR-013`、`H-1` |
| `idx_fine_copy` | `fine_records` | `copy_id` | INDEX | 按副本追溯赔偿款 | `BR-022` |
| `idx_fine_accrual_date` | `fine_records` | `accrual_date` | INDEX | 按日期区间查询款项 | `FR-017` |
| `pk_audit`（隐式） | `audit_logs` | `id` | PK | 主键查找 | — |
| `idx_audit_occurred_at` | `audit_logs` | `occurred_at` | INDEX | 审计日志按时间倒序分页（`UC-23`） | `FR-005` |
| `idx_audit_target` | `audit_logs` | `target_type`, `target_id` | INDEX | 按对象追溯（含已注销读者、报废副本） | `FR-005` 验收 4 |
| `idx_audit_actor` | `audit_logs` | `actor_id` | INDEX | 按操作者追溯 | `FR-005` |
| `idx_audit_action` | `audit_logs` | `action_type` | INDEX | 按操作类型筛选 | `FR-005` |

### 6.3 不建索引的显式说明

| 场景 | 不建索引的理由 |
|---|---|
| `readers.department` / `name` | 无按系别 / 姓名的检索用例（`03` 未定义）；避免低选择性索引 |
| `item_copies.deleted` / `library_items.deleted` | 低基数列；规模 `D3` 下过滤代价可接受，且逻辑删除过滤常与高选择性条件组合 |
| `reservations.queue_position` 单列 | 位次只在标题内有意义，已被 `idx_resv_queue` 覆盖 |
| `fine_records.unpaid_amount` / `amount` | 无按金额范围检索用例；封顶与黑名单走 `idx_fine_reader_type_status` 后聚合 |
| 全文 / 模糊索引（`title` 的 `%kw%`） | 只做**精确或前缀**匹配（`FR-023` 验收 7），前缀匹配可由 `idx_items_title` 的范围扫描完成 |

---

## 7. 领域继承到关系数据库的映射策略

### 7.1 三种标准策略对照

| 策略（JPA） | 表结构 | 判别方式 | 优点 | 缺点 |
|---|---|---|---|---|
| **单表继承** `SINGLE_TABLE` | 一张表容纳全部属性 | 判别列（discriminator） | 无 JOIN、查询最快；改类型只改判别列；跨子类迁移最容易 | 子类专属列必须可空；列稀疏；无法对子类列加非空约束 |
| **连接表继承** `JOINED` | 父表 + 每张子类一张表（子类主键 = 外键） | 子类表是否存在对应行 | 无冗余列；子类列可非空；父类可被独立引用（预约 / 副本只引用父表） | 读取需 JOIN；写入需两表；多态查询成本较高 |
| **每具体类一表** `TABLE_PER_CLASS` | 每个具体子类一张完整表，**无父表** | 表名即类型 | 单类型查询零 JOIN；类型边界天然隔离 | 跨子类唯一性无法由 DB 保证；多态查询需 UNION；公共列重复定义 |

### 7.2 本项目的三处映射决策

| # | 领域继承树（`05` §3.2） | 采用策略 | 落库结果 | 决策理由 |
|---|---|---|---|---|
| `DB-D-1` | `Reader` → `StudentReader` → 4 个学生子类；`Reader` → `TeacherReader` | **单表继承** | `readers` 一张表；判别列 = `reader_type`（**复用业务枚举，不另设 `dtype`**） | ① 子类**不重写任何行为**，只提供 `readerType` 初始取值与身份属性（`05` §3.2 D-1 第 1 条）；② `reader_type` **可变**且是 `BorrowPolicy` 的唯一查找键 —— 若另设 `dtype`，会出现「`dtype` 与 `reader_type` 不一致」的**第二处真相**；③ 跨子类修正（本科生 → 教师）退化为一次 `UPDATE`（改判别列 + 身份列），`id` 与 `borrow_cards` 不变，正是 D-1 第 3 条要求的「重建实体并沿用原 id」；④ 避免 5 张表 JOIN 换取的零收益（子类独有列仅 3 个） |
| `DB-D-2` | `LibraryItem` → `Book` / `Magazine` / `Thesis` | **连接表继承** | 父表 `library_items` + `book_titles` / `magazines` / `theses`；连接键 = 主键兼外键 `item_id` | ① 子类属性差异大且**稀疏互斥**（`publisher` / `issue` / `supervisor` 互不通用），单表会产生大量 NULL 列；② 预约、副本、罚款查找只需引用**父表**（`reservations.item_id`、`item_copies.item_id`），父表独立存在可让这些高频路径**免 JOIN**；③ 公共检索列（书名 / 作者 / ISBN）集中在父表，检索无需 UNION |
| `DB-D-3` | `AdminAccount`（`05` §4.15：**无继承**，仅 `role` 枚举） | **每具体类一表（按角色分表）** | `librarians` + `system_admins`（同构表，各自带 `role` 常量列） | ① 两类账号在权限矩阵上**完全互斥**（`07` §7.4），分表使「系统管理员不可借书」等硬约束在存储层即可核查；② `loans.operator_id` 可直连 `librarians`，语义明确；③ 领域层仍**只有一个** `AdminAccount` 类，分表只发生在持久层，由 `AdminAccountMapper` 按 `role` 双向路由（**无领域继承、有持久层继承**的特例，见 `DB-ASM-02`） |

**`DB-D-3` 的代价与补偿（必须说明）**

| 代价 | 补偿措施 |
|---|---|
| 跨表 `username` 全局唯一无法由 DB 约束保证 | 各表内唯一索引 + `AdminAccountRepository.findByUsername()` 双表查询后判重（应用层唯一入口）；冲突登记 `DB-ASM-02` |
| 认证需定位账号所在表 | 由 `AuthApplicationService` 先按角色路由，或一次 `UNION ALL` 查询（小规模数据无性能问题） |
| 公共列重复定义（7 列） | 两表 DDL 同构，变更须**同轮修改**；若后续难以维护，可退化为 `JOINED`（新增父表 `admin_accounts`） |

### 7.3 判别列与派生列的取舍

| 项 | 处理 | 理由 |
|---|---|---|
| `readers.reader_type` | 业务枚举**兼作**判别列，不额外加 `dtype` | 避免第二处真相（`DB-D-1` 理由②） |
| `library_items.item_type` | 落为**派生冗余列**（由子类 + `language` 推导后写入，不人工录入） | `FineRule` 查找键需高频读取；物化后可避免「每次算费率都要 JOIN 三张子类表」。与 `05` `H-3`（来源是推导而非枚举录入）不冲突，物化策略待确认（`DB-ASM-03`） |
| `library_items.item_type` 与子类的一致性 | 由 `LibraryItemFactory`（`07` DP-07）在写入时保证；DB 只做取值域 `CHECK` | 规则不得落 SQL / 触发器（第七条第 3 款、`07` §10.2） |
| `magazines` / `theses` 表 | 一并建立，保证 JOINED 策略完整 | 否则 `Magazine` / `Thesis` 无落库位置，违反 `BR-031` 五类型覆盖 |

### 7.4 值对象与枚举的落库

| 领域构造 | 落库形态 | 说明 |
|---|---|---|
| `Money` | `DECIMAL(10,2)` | 定点小数，**禁止** `DOUBLE` / `FLOAT`（`05` §5） |
| `CardNumber` / `CopyNumber` / `ISBN` | `VARCHAR` 列 + 唯一约束 | 全局唯一由 DB 兜底（`BR-027`、`BR-019`） |
| `DateRange` | 拆为两列 / 由查询条件承载 | 无独立表 |
| `OverdueDays` | **不落库**，由 `due_date` 与当前日期计算 | `BR-007` |
| 8 个枚举 | `VARCHAR` 列 + `CHECK`（中文术语逐字） | **不建码表**（`05` §3.4）；取值与 `02` 附录 A 逐字一致（第二条第 4 款） |
| `AuditEntry` | 展开为 `audit_logs` 的 `action_type` / `target_type` / `target_id` / `result` 列 | 值对象嵌入（`05` §5） |

### 7.5 PO 与表的对应（对齐 `07` §5.2 `infrastructure.persistence`）

| PO（`infrastructure.persistence`） | 表 | 领域对象 |
|---|---|---|
| `ReaderPO` | `readers` | `Reader` + 全部子类 |
| `BorrowCardPO` | `borrow_cards` | `BorrowCard` |
| `LibrarianPO` | `librarians` | `AdminAccount`（`role = 图书管理员`） |
| `SystemAdminPO` | `system_admins` | `AdminAccount`（`role = 系统管理员`） |
| `LibraryItemPO` | `library_items` | `LibraryItem`（抽象父） |
| `BookPO` | `book_titles` | `Book` |
| `MagazinePO` | `magazines` | `Magazine` |
| `ThesisPO` | `theses` | `Thesis` |
| `ItemCopyPO` | `item_copies` | `ItemCopy` |
| `LoanPO` | `loans` | `Loan` |
| `ReservationPO` | `reservations` | `Reservation` |
| `BorrowPolicyPO` | `borrow_policies` | `BorrowPolicy` |
| `FineRulePO` | `fine_rules` | `FineRule` |
| `FineRecordPO` | `fine_records` | `FineRecord` |
| `AuditLogPO` | `audit_logs` | `AuditLog` |

> PO **禁止**上溯至表现层（`07` §4.4 硬禁止）；PO ↔ 领域对象由手写 `*Mapper` 转换（`ASM-04`，不引入 MapStruct）。

---

## 8. 完整性与一致性的分工

| 层 | 承担 | 不承担 |
|---|---|---|
| **数据库**（本文件） | 取值域（`CHECK`）、唯一性（唯一索引）、引用完整性（`FK` + `RESTRICT`）、列间不变量（状态 ↔ 日期的充要关系）、并发互斥（计算列唯一索引、`version`） | 状态**迁移**合法性、规则数值、业务计数上限 |
| **领域层** | 状态迁移（五态、`05` §8）、应还日期与上限（`BorrowPolicy`）、罚款金额（`FineRule`）、预约上限 3、队列 FIFO 与重排 | 存储细节 |
| **应用层** | 事务边界、权限拦截、审计编排、乐观锁重试与 409 转译 | 任何业务规则 |

> 明确**禁止**用触发器 / 存储过程 / 视图承载上述业务规则（第七条第 3 款、`07` §10.2）；计算列（`loans.open_copy_key`、`reservations.active_slot`）仅用于**唯一性判定**，不参与任何金额或日期计算。

---

## 9. 种子数据与初始化

| 数据 | 内容 | 依据 |
|---|---|---|
| `borrow_policies`（5 行） | 专科生 3 / 15；本科生 5 / 30；研究生 7 / 30；博士生 10 / 60；教师 15 / 60 | `BR-001`、`02` 附录 A、`07` §5.2 `SeedDataInitializer` |
| `fine_rules`（5 行） | 中文图书 2；外文图书 2.5；中文杂志 1；外文杂志 1.5；论文 3；宽限期均 0 | `BR-007`、`02` 附录 A |
| `system_admins`（≥ 1 行） | 初始系统管理员账号（`TBD-008`） | `05` §4.15 第 5 条 |
| 演示数据 | 读者 / 标题 / 副本 / 借阅记录若干（运行模式） | `07` §4.4 第 ⑦ 项 |
| 测试种子 | `src/test/resources/seed-test.sql`，与 `jdbc:h2:mem:library` 配套 | 第九条第 3 款 |
| 数值出现位置 | **仅**种子数据脚本允许出现 `3 / 5 / 7 / 10 / 15`、`2 / 2.5 / …`、`50`、`3 天` 等字面量 | `07` §10.2 |

> `borrow_policies` / `fine_rules` 的 5 行**必须齐全**：缺失即导致计提失败并留痕（`UC-21 E1`），不得静默按 0 处理。

---

## 10. 假设与待确认项

| 编号 | 内容 | 影响 | 处理 |
|---|---|---|---|
| `DB-ASM-01` | 规则配置表命名采用 `borrow_policies` / `fine_rules`，与 `constitution` 第七条第 3 款写作的 `borrowing_rule` / `fine_rule` 不一致 | `constitution`、`16-tasks.md`、`src/` | **须人类裁决**：若判定以 `constitution` 为准，须回写本文件全部表名与索引名；若以本文件为准，须按第五条走变更流程修订 `constitution` |
| `DB-ASM-02` | `AdminAccount` 按角色拆分为 `librarians` / `system_admins`，与 `05` §4.15 单一实体 + `role` 枚举的口径不同；跨表 `username` 唯一仅由应用层保证 | `05` §4.15、`07` §5.2 PO 命名、`AdminAccountRepository` | 备选方案：改为 `JOINED`（新增父表 `admin_accounts` 换取 DB 级唯一）。裁决前按本文件推进 |
| `DB-ASM-03` | `library_items.item_type` 作为派生冗余列落库 | `05` `H-3`、`FineRuleProvider` 实现 | 若严格按 `H-3` 不落列，则费率查找须 JOIN 子类表并按 `CASE` 推导（禁止落 SQL 计算，需在 Java 侧完成） |
| `DB-ASM-04` | `library_items.isbn` 设为 NOT NULL；论文无 ISBN 时的登记口径未定义 | `theses`、`FR-006` 验收 7 | 建议：论文登记校内学位论文档案号。**须人类确认**，否则改为可空并调整 `uk_items_isbn_type` |
| `DB-ASM-05` | 计算列（`GENERATED ALWAYS AS`）语法与「计算列可建唯一索引」需在 H2 2.x 实测 | `loans.open_copy_key`、`reservations.active_slot` | 集成测试须验证；若 H2 不支持，退化为「应用层维护可空列 + 唯一索引」，或退化为行级锁（`07` §9.5） |
| `DB-ASM-06` | `readers.student_no` 是否必填（`05` §4.2 未明确「可选」归属 `studentNo` 还是仅 `grade`） | `readers`、`ck_readers_identity` | 暂按「学号必填于学生、教职工号可空」设计，以 `CHECK` 保证教师无学号 |
| `DB-ASM-07` | 会话不建表（`05` §2.3、`07` §7.2）：会话状态由 `infrastructure.security.SessionStore` 内存管理 | 单机部署、重启即失效 | 若需集群或多实例，须走第五条变更流程 |
| `DB-ASM-08` | 本轮生成依 NFR-007 须登记 `19-ai-usage-log.md`，但人类指令限定仅修改本文件 | `19-ai-usage-log.md`、第四条第 1 款 | 依第一条第 2 款**指出的冲突**：登记条目尚未写入，须由人类补齐或另行授权；完成登记前本文件**不得**作为 `09` / `15` / `16` 与 `src/` 的输入 |
| `DB-ASM-09` | 审查提出「`library_item.barcode` 是否唯一」：本设计**不建** `barcode` 列 | `library_items`、`item_copies`、第二条第 4 款、第十条第 1 款 | 理由：① `05` §5 的副本标识值对象是 `CopyNumber`，`LibraryItem` 无条码属性；② 副本唯一编号落在 `item_copies.copy_number`（`uk_copies_number`，`BR-019`）；③「图书条码扫描」属 `02` §2.3 不做清单。**若裁决须新增 `barcode`**，属新增术语，须先按第五条变更流程修订 `05` / `02`，不得在 `13` 直接加列 |
| `DB-ASM-10` | 命名口径：审查项写作单数表名与缩写列名（`borrow_card.card_no`、`library_item.barcode`），本文件统一为**复数表名 + snake_case 全称**（`borrow_cards.card_number`、`item_copies.copy_number`） | 全部 15 张表、索引名、PO 与 `src/` | 语义等价（借阅证号 = `card_number`；副本唯一编号 = `copy_number`）。**须人类确认以哪套命名为准**；若改用 `card_no` 等缩写，须同轮全文件重命名并同步 `07` §7.5 PO 对照 |

**引用的上游待确认项**：`TBD-002`（读者类型确定方式 → `readers.reader_type` 写入路径）、`TBD-003`（50 元封顶口径 → 由 `idx_fine_reader_type_status` 聚合实现，两种口径均不新增列）、`TBD-006`（系统管理员能否预约 → 不新增列，领域层判定）、`TBD-007`（未缴赔偿款是否禁借 → 同上）、`TBD-008`（初始系统管理员 → 种子数据）、`TBD-010`（借阅证号格式 → `card_number VARCHAR(20)`）、`TBD-012`（逻辑删除 → `deleted` 列）。以上均按 `02` 建议默认推进。

---

## 11. 与上游文档的一致性自检

| # | 检查项 | 本文件落点 | 判定 |
|---|---|---|---|
| 1 | H2 唯一方言，无双方言适配 | §1、§2、字段类型全部为 H2 类型 | 符合（`NFR-002`） |
| 2 | 术语与枚举与 `02` 附录 A 逐字一致 | 全部 `CHECK` 取值（`在馆 / 预约保留 / 中文图书 / 专科生 / 超期` 等） | 符合（第二条第 4 款） |
| 3 | 不建字典码表；枚举以列 + CHECK 表达 | §2、§7.4 | 符合（`05` §3.4） |
| 4 | 规则数值只存配置表，SQL / CHECK 中不出现 | §5.12 / §5.13 / §8 | 符合（`BR-014`、第七条第 3 款） |
| 5 | 派生状态不落列 | §3.2、`readers` 无黑名单列、`fine_records` 无累计额列 | 符合（`H-1`） |
| 6 | 全部删除为逻辑删除，外键 `RESTRICT` | §4.2、`deleted` / `status` 列 | 符合（第十条第 4 款、`NFR-008`） |
| 7 | 审计只追加、无外键、无 version | §5.15 | 符合（`BR-030`） |
| 8 | 不含「不做」清单概念（损坏 / 续借 / 挂失 / 催还 / 报表 / 支付 / 会话表） | §2、§5.2、§5.10 | 符合（第十条） |
| 9 | 与 `07` §5.2 `infrastructure.persistence` 对齐 | §7.5 PO ↔ 表对照 | 符合（`07` §1 同步要求） |
| 10 | 并发控制手段与 `07` §9.5 一致 | `version` 列 + 计算列唯一索引 | 符合（`NFR-009`） |
| 11 | 三棵继承树均有明确映射策略 | §7.2 `DB-D-1` ~ `DB-D-3` | 符合（`05` §3.2） |
| 12 | 冲突已声明而非静默绕过 | §0 四项冲突声明 | 符合（第一条第 2 款） |

---

## 附录 A 修订记录

| 版本 | 日期 | 修订内容 | 来源 |
|---|---|---|---|
| v1.0 | 2026-09-23 | 首版成文：① 冲突声明（4 项）；② 持久化对象识别与「不持久化」清单；③ 15 张表的字段级设计（类型 / 空值 / 主键 / 外键 / 唯一 / 默认值 / 说明）；④ 外键关系矩阵；⑤ 索引设计（原则 + 39 条索引清单 + 不建索引说明）；⑥ 继承映射策略（三种策略对照 + `DB-D-1`~`DB-D-3` 决策 + 值对象 / 枚举落库 + PO 对照）；⑦ 完整性分工（DB / 领域 / 应用）；⑧ 种子数据；⑨ 待确认项 `DB-ASM-01`~`DB-ASM-08`；⑩ 与上游一致性自检 | `05-domain-model.md`（v1.1）、`07-architecture.md`（v1.0）、`06-domain-class-diagram.puml`、`02-requirements.md`（v1.1）、`constitution.md`（v1.0）、人类指令（11 张必需表） |
| v1.0.1 | 2026-09-23 | 按 7 项审查核查补正：① §5.10 新增「超期为派生状态、不建列」条目（判定口径 + 理由 + 索引与幂等支撑）；② §10 新增 `DB-ASM-09`（不建 `barcode` 列的三条依据与变更路径）与 `DB-ASM-10`（单数表名 / 缩写列名 vs 复数表名 / snake_case 全称的命名口径）；③ 新增附录 B 审查项自检对照 | 人工审查意见（7 项）、`05` §4.10 / §5、`02` §2.3、`constitution` 第二条第 4 款 |

---

## 附录 B 审查项自检对照（7 项）

| # | 审查问题 | 结论 | 证据 / 处理 |
|---|---|---|---|
| 1 | `borrow_card.card_no` 是否唯一？ | **唯一** | `borrow_cards.card_number` `VARCHAR(20) NOT NULL` + `uk_cards_number` UNIQUE（§5.2、§6.2）；并列 `uk_cards_reader` 保证 1:1。命名差异见 `DB-ASM-10` |
| 2 | `library_item.barcode` 是否唯一？ | **不建该列**（唯一性由副本编号承担） | `item_copies.copy_number` `VARCHAR(32) NOT NULL` + `uk_copies_number` UNIQUE（`BR-019`、`FR-007` 验收 1）；不建 `barcode` 的三条依据登记为 `DB-ASM-09` |
| 3 | `loans` 能否区分借出 / 已还 / 超期？ | **能**（超期为派生） | `status`（在借 / 已还）+ `return_date` 由 `ck_loans_return` 强制充要；超期 = `status='在借' AND due_date < 今日`，由 `Loan.isOverdue()` 计算、不落列（§5.10 新增条目）；扫描走 `idx_loans_status_due`，计提幂等走 `uk_fine_loan_date` |
| 4 | `reservations` 是否支持排队？ | **支持** | `queue_position` + `reserved_at` 双排序键、`status` 前两态为在队、`idx_resv_queue (item_id, queue_position, reserved_at)`、`active_slot` 唯一索引防重复；不建 `reservation_queues` 表（`BR-016`、`05` §4.11） |
| 5 | `fine_rules` 是否支持不同借出物类型？ | **支持** | `item_type` 唯一键 + CHECK 5 值 + 每行 `daily_rate` / `grace_period_days`；种子 5 行齐全，缺行计提失败留痕（`BR-007`、`UC-21 E1`） |
| 6 | `borrow_policies` 是否支持不同读者类型？ | **支持** | `reader_type` 唯一键 + CHECK 5 值 + 每行 `max_copies` / `loan_period_days`；只按读者类型分档（`BR-001`、`BR-002`） |
| 7 | 是否存在必要外键？ | **齐全（13 处）** | §4.2 外键矩阵；全部 `RESTRICT`（逻辑删除）；故意不建外键的三处：`audit_logs`（弱引用）、`borrow_policies` / `fine_rules`（按枚举键查找）、`system_admins`（未被引用） |

> **留痕要求（NFR-007 / 第四条第 5 款）**：本次由 Agent 生成，须在 `19-ai-usage-log.md` 新增一条记录（使用工具、使用阶段、Prompt 摘要、修改文件、输出摘要、测试结果、Git 提交），「人工审查结果」由人类补齐；**登记完成且审查通过前**，本文件不得作为 `09-design-model.md`、`15-test-plan.md`、`16-tasks.md` 与 `src/` 的输入（见 §0 冲突 4、`DB-ASM-08`）。
