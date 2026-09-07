# 鲜花仓库管理系统（Flower-WMS）

基于 **Spring Boot + Vue** 前后端分离架构的鲜花仓储全流程管理系统。系统覆盖「采购计划 → 品质检验 → 入库上架 → 库存盘点 → 智能出库 → 配送执行 → 异常上报」的完整业务闭环，并按不同岗位（管理员、采购、仓库、配送、质检）划分独立工作台，实现了角色化、流程化、可视化的鲜花仓储管理。

---

## 目录

1. [项目简介](#一项目简介)
2. [技术栈](#二技术栈)
3. [系统角色](#三系统角色)
4. [功能模块总览](#四功能模块总览)
5. [核心业务流程](#五核心业务流程)
6. [特色功能：出库优先级评分算法](#六特色功能出库优先级评分算法)
7. [后端接口一览](#七后端接口一览)
8. [数据库设计概览](#八数据库设计概览)
9. [项目目录结构](#九项目目录结构)
10. [本地运行指南](#十本地运行指南)
11. [默认测试账号](#十一默认测试账号)
12. [补充说明与已知问题](#十二补充说明与已知问题)

---

## 一、项目简介

鲜花属于**易腐、保鲜期短、对温度敏感**的商品，其仓储管理在时效性与环境要求上远高于普通货品。本系统围绕鲜花的这一行业特性，将鲜花按品类维护适配温区与保鲜期，并据此设计了「温区匹配的货架管理」「保鲜优先的出库调度」等特色能力。

系统主要完成以下工作：

- **采购管理**：采购员录入/批量上传采购计划，对到货鲜花上报缺货、品质等异常；
- **品质检验**：仓库管理员对待入库鲜花评定品质等级（优 / 良 / 差），实现入库前的质量把关；
- **入库与库存**：审核通过的鲜花执行入库/一键入库，自动流转到货架与待出库队列，支持库存盘点与损耗预警；
- **出库调度**：对待出库订单进行多维加权**优先级评分**，指导仓库按重要性/时效性安排出库；
- **配送执行**：出库自动拆分生成配送任务，配送员完成到货接收、完成、异常反馈等全流程操作；
- **系统管理**：管理员维护用户与角色、货架等基础数据。

---

## 二、技术栈

| 分层        | 技术选型                           | 说明                                          |
| ----------- | ---------------------------------- | --------------------------------------------- |
| 后端框架    | Spring Boot 2.7.18                 | 核心 Web 框架                                 |
| ORM         | MyBatis-Plus 3.5.3                 | Mapper 层持久化，含逻辑删除、分页能力         |
| 数据库      | MySQL 8.x                          | 业务数据存储（库名 `flower_wms`）             |
| 安全 / 鉴权 | Spring Security + JWT（jjwt）+ MD5 | 登录签发 JWT，密码 MD5 存储（见「已知问题」） |
| 接口文档    | springdoc-openapi (Swagger UI 1.7) | 在线文档依赖已集成                            |
| 文件上传    | Spring Multipart                   | 本地磁盘存储，静态映射 `/uploads/**`          |
| 前端框架    | Vue 2.6 + Vue Router 3 + Vuex 3    | SPA 单页应用                                  |
| UI 组件库   | Element UI 2.15                    | 页面组件与表单                                |
| HTTP        | Axios                              | 统一封装，请求拦截器自动携带 Token            |
| 构建工具    | Maven / Vue CLI 5                  | 前后端分别构建                                |

> 说明：`package.json` 声明了 `echarts@5` 依赖，但当前前端统计仪表盘实际由 Element UI 的环形进度条（`el-progress type="circle"`）与数据卡片实现，并未在页面中真正调用 ECharts。

---

## 三、系统角色

系统内置 5 类角色（`sys_role`），不同角色登录后进入各自独立的工作台页面：

| roleId | 角色       | 登录入口                             | 主要职责                                  |
| ------ | ---------- | ------------------------------------ | ----------------------------------------- |
| 1      | 系统管理员 | `/admin-login`（管理员专用登录页）   | 用户管理、货架管理、数据总览              |
| 2      | 采购员     | `/` 通用登录页                       | 创建/上传采购计划、异常上报、查看采购记录 |
| 3      | 仓库管理员 | `/` 通用登录页                       | 品质审核、库存管理、货架管理、出库管理    |
| 4      | 仓库配送员 | `/` 通用登录页                       | 配送任务接收与执行、配送记录              |
| 6      | 质检员     | （角色已建，前端暂无独立工作台页面） | —                                         |

数据库初始化脚本内置了**权限点表（`sys_permission`）与角色-权限关联表（`sys_role_permission`）**，如 `user:manage`、`warehouse:audit`、`delivery:receive` 等 14 个权限编码，用于描述各角色的权限边界。

---

## 四、功能模块总览

### 4.1 系统管理员（admin）

| 页面     | 路由           | 功能说明                                                     |
| -------- | -------------- | ------------------------------------------------------------ |
| 首页     | `/admin/index` | 欢迎页与统计总览卡片（用户总数、货架总数、鲜花品类、待处理订单） |
| 用户管理 | `/admin/users` | 用户**新增 / 编辑 / 删除**；表单含用户名、密码、电话、角色下拉、启用/停用开关 |
| 货架管理 | `/admin/shelf` | 复用仓库管理员的货架页面，可查看空闲/占用货架、启停货架      |

### 4.2 采购员（buyer）

| 页面     | 路由                | 功能说明                                                     |
| -------- | ------------------- | ------------------------------------------------------------ |
| 首页     | `/buyer/index`      | 采购通过率圆环、待审核/已通过/已拒绝/总计划数指标、快捷操作、最近动态 |
| 采购计划 | `/buyer/plans`      | 新建采购计划（花名/数量/省市区级联收货地址/金额）；**批量上传采购计划同步至仓库**；对某计划**上报异常**（缺货/品质异常，品质异常可上传照片） |
| 采购记录 | `/buyer/records`    | 历史采购上传记录查询，按状态（待审核/已通过/已驳回/已入库）标签展示 |
| 异常记录 | `/buyer/exceptions` | 查看**本人**上报的异常列表，含照片预览与处理状态             |

### 4.3 仓库管理员（warehouse）

| 页面              | 路由                   | 功能说明                                                     |
| ----------------- | ---------------------- | ------------------------------------------------------------ |
| 首页              | `/warehouse/index`     | 待品质审核数量、**货架占用环形图**（空闲/占用/停用）、待出库数量、仓库动态 |
| 库存管理/品质审核 | `/warehouse/inventory` | 对采购上传的库存记录行内评定**品质等级（优/良/差）**：优/良→审核通过，差→审核不通过；合格记录**一键入库**流转至待出库队列 |
| 待出库管理        | `/warehouse/outbound`  | 查看按优先级排序的待出库订单，执行**出库**；查看并清除历史出库记录 |
| 货架管理          | `/warehouse/shelf`     | 货架增删、**启用/停用**；占用货架视图按货架展示鲜花与数量，支持**单个清仓 / 一键清仓** |

### 4.4 仓库配送员（delivery）

| 页面     | 路由                | 功能说明                                                     |
| -------- | ------------------- | ------------------------------------------------------------ |
| 首页     | `/delivery/index`   | 待配送/已完成任务统计、快捷操作、系统信息                    |
| 配送任务 | `/delivery/tasks`   | 查看待配送任务，执行**到货接收**（自动生成配送记录）→ **完成**；配送中可上报**异常原因** |
| 配送记录 | `/delivery/records` | 查看配送执行记录，支持一键清除                               |

### 4.5 登录 / 注册

- 通用登录页 `/`（`Login.vue`）：普通员工登录，**自动按角色跳转到对应工作台**；提供**自助注册**（选择部门→选择职位/角色，校验手机号、邮箱、密码一致性）。
- 管理员登录页 `/admin-login`（`LoginAdmin.vue`）：仅允许系统管理员（roleId=1）登录，跳转 `/admin`。

---

## 五、核心业务流程

系统主链路采用「数据在 `采购计划 → 库存(待审核) → 出库单 → 配送任务` 多张业务表间流转」的方式实现状态机：

```
采购员创建采购计划(purchase_plan, pending)
        │
        │ 到货后：品质报送 / 或 批量「上传采购计划」
        ▼
生成采购记录(purchase_record) 并同步生成库存(inventory, 状态=0 待审核)
        │
        ▼  仓库管理员：品质审核（行内评定优/良/差）
品质合格(状态=1) 或 品质不合格(状态=3)
        │
        ▼  仓库管理员：一键入库
合格记录删除并自动生成出库单(outbound_order, 状态=0 待出库)
        │
        ▼  仓库：出库优先级评分排序 → 执行出库
出库完成(状态=1)后按货架拆分生成配送任务(delivery_task, type=2)
        │
        ▼  配送员：到货接收 → 完成任务 / 上报异常
配送完成释放货架资源，产生配送记录与出库记录
```

> 说明：采购订单也可直接走 `purchase/approve → inStorageWithQuality` 的自动入库分支（为指定货架入库时自动分配空闲货架），同时将采购单置为 `stored`。

**异常处理支线**：采购员对采购计划上报「缺货 / 品质异常」，可附照片，形成异常记录（`purchase_exception`），供后续跟进处理。

**库存盘点支线**：仓库对某库存执行盘点（录入实盘数），系统自动计算损耗率；当损耗率 > 5% 时提示预警，并保留盘点历史（`inventory_check`）。

---

## 六、特色功能：出库优先级评分算法

系统实现了对待出库订单的**六维加权优先级评分**（`OutboundOrderServiceImpl.calculatePriority()`），用于指导仓库优先处理"更易损耗、更紧急、价值更高"的订单，评分维度与权重如下：

| 维度         | 权重 | 评分口径                                                     |
| ------------ | ---- | ------------------------------------------------------------ |
| 运输方式     | 25%  | 按收货地区匹配运输方式：偏远地区空运 25 分 / 一般陆运 18 分 / 默认 10 分 |
| 订单金额     | 25%  | 金额每 1000 元 +1 分，上限 25 分                             |
| 入库停留时间 | 15%  | 等待时间越长得分越低：基础 15 分，每等待 2 小时 -1 分（下限 0） |
| 鲜花保鲜期   | 15%  | 保鲜期 ≤5 天 15 分 / ≤7 天 10 分 / ≤14 天 5 分 / >14 天 1 分（按品类保鲜期查表） |
| 气温影响     | 10%  | 按目的地区实时气温（未填则用地区默认气温）：>30℃ 10 分 / ≥25℃ 7 分 / ≥15℃ 4 分 / <15℃ 1 分 |
| 订单数量     | 10%  | 数量每 10 件 +1 分，上限 10 分                               |

总分按 **100 分制**计，`GET /api/outbound/priority` 对待出库订单实时评分并**降序返回**。

此外，出库完成时系统会**自动在空闲货架间按每架 ≤50 束拆分配送任务**，货架不足时回滚提示；配送完成后自动释放货架并回减已用容量（有并发任务时延迟释放）。

---

## 七、后端接口一览

后端统一返回封装体 `ApiResponse{ code, message, data }`，业务接口前缀 `/api`。按模块主要接口如下：

### 认证与用户管理（AuthController / RoleController）

| 方法                | 路径                 | 说明                                                    |
| ------------------- | -------------------- | ------------------------------------------------------- |
| POST                | `/api/login`         | 用户登录（用户名/手机号/邮箱均可），返回 JWT 与角色信息 |
| GET                 | `/api/user/info`     | 根据 Token 获取当前用户信息                             |
| GET / POST          | `/api/users`         | 用户列表 / 新增用户（密码 MD5 后存储）                  |
| PUT / DELETE        | `/api/users/{id}`    | 更新 / 删除用户                                         |
| GET                 | `/api/roles`         | 角色列表                                                |
| GET/POST/PUT/DELETE | `/api/admin/role(s)` | 角色增删改查                                            |

### 基础数据（CategoryController / UploadController）

| 方法       | 路径                   | 说明                                     |
| ---------- | ---------------------- | ---------------------------------------- |
| GET/POST   | `/api/categories`      | 鲜花品类列表 / 新增                      |
| PUT/DELETE | `/api/categories/{id}` | 品类更新 / 删除                          |
| POST       | `/api/upload`          | 文件上传，返回 `/uploads/xxx` 可访问 URL |

### 采购管理（PurchaseController）

| 方法 | 路径                    | 说明                                                         |
| ---- | ----------------------- | ------------------------------------------------------------ |
| GET  | `/api/purchase/plans`   | 采购计划列表（支持按状态筛选）                               |
| GET  | `/api/purchase/pending` | 待审批计划                                                   |
| POST | `/api/purchase/plan`    | 创建采购计划                                                 |
| POST | `/api/purchase/quality` | 报送品质信息（附图片）                                       |
| POST | `/api/purchase/approve` | 审核采购/库存（按 inventoryId 或 orderId；approved + qualityLevel） |
| POST | `/api/purchase/upload`  | 上传采购计划→生成采购记录并同步库存                          |
| GET  | `/api/purchase/records` | 采购记录列表（支持状态筛选）                                 |

### 仓库管理（WarehouseController）

| 方法       | 路径                               | 说明                                          |
| ---------- | ---------------------------------- | --------------------------------------------- |
| GET        | `/api/inventory`                   | 库存列表（含品质审核状态）                    |
| GET / POST | `/api/shelf`                       | 货架列表 / 新增货架                           |
| PUT        | `/api/shelf/{id}/capacity`         | 调整货架已用容量                              |
| PUT        | `/api/shelf/status`                | 启用/停用货架                                 |
| POST       | `/api/inventory/in`                | 按采购订单执行入库（可带品质等级）            |
| POST       | `/api/inventory/out`               | 直接出库                                      |
| POST       | `/api/inventory/outbound`          | 出库并生成出库单                              |
| POST       | `/api/inventory/check`             | 库存盘点（自动算损耗率）                      |
| GET        | `/api/inventory/check/history`     | 盘点历史                                      |
| POST       | `/api/inventory/one-click-storage` | 一键入库（审核通过且品质优/良的记录批量流转） |

### 出库管理（OutboundController）

| 方法         | 路径                     | 说明                         |
| ------------ | ------------------------ | ---------------------------- |
| GET          | `/api/outbound/pending`  | 待出库订单                   |
| GET          | `/api/outbound/priority` | 按优先级评分降序的出库订单   |
| POST         | `/api/outbound`          | 创建出库单                   |
| POST         | `/api/outbound/complete` | 完成出库（自动拆分配送任务） |
| GET / DELETE | `/api/outbound/records`  | 出库记录 / 清除记录          |

### 配送管理（DeliveryController）

| 方法            | 路径                          | 说明                                      |
| --------------- | ----------------------------- | ----------------------------------------- |
| GET             | `/api/delivery/tasks/pending` | 待接收任务（自动关联展示源/目标货架编号） |
| GET             | `/api/delivery/tasks`         | 按配送员查询任务                          |
| POST            | `/api/delivery/task/receive`  | 接收任务（到货确认）                      |
| POST            | `/api/delivery/task/complete` | 完成任务（释放货架）                      |
| POST            | `/api/delivery/task/abnormal` | 异常反馈                                  |
| GET/POST/DELETE | `/api/delivery/record(s)`     | 配送记录的查询 / 保存 / 清除              |

### 异常上报（ExceptionController）

| 方法 | 路径                    | 说明                                  |
| ---- | ----------------------- | ------------------------------------- |
| GET  | `/api/exceptions/my`    | 本人异常记录                          |
| GET  | `/api/exceptions`       | 全部异常记录                          |
| POST | `/api/exception/report` | 上报异常（缺货 / 品质异常，支持照片） |

---

## 八、数据库设计概览

数据库 `flower_wms`，初始化脚本 `backend/src/main/resources/sql/init.sql` 内置建表与演示数据。共 15 张表，按作用域分为两组：

### 系统权限组

| 表名                  | 说明                                                 |
| --------------------- | ---------------------------------------------------- |
| `sys_user`            | 用户表（用户名唯一、MD5 密码、角色、部门、启用状态） |
| `sys_role`            | 角色表（内置管理员/采购员/仓库管理员/配送员/质检员） |
| `sys_permission`      | 权限点表（14 个权限编码）                            |
| `sys_role_permission` | 角色-权限关联表                                      |

### 业务数据组

| 表名                                 | 说明                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `flower_category`                    | 鲜花品类表（温区、保鲜期）                                   |
| `purchase_plan`                      | 采购计划表（商家、收货地、金额、紧急度/客户等级、状态机 pending→delivered/approved/rejected/stored） |
| `purchase_record`                    | 采购记录表（批量上传后的历史单据）                           |
| `purchase_exception`                 | 采购异常上报记录表（缺货/品质异常、照片、处理状态）          |
| `rejected_flower_record`             | 驳回鲜花记录表（品质驳回留痕）                               |
| `inventory` / `inventory_check`      | 库存表 + 库存盘点记录表（损耗率预警）                        |
| `shelf`                              | 货架表（库区、容量、温区、占用状态、距出库口距离）           |
| `delivery_task` / `delivery_record`  | 配送任务表（入库/出库两种类型）+ 配送执行记录表              |
| `outbound_order` / `outbound_record` | 出库订单表（含 priority_score）+ 出库记录表                  |

> 库存相关的主要状态约定：库存 `status`：0 待品质审核 / 1 审核通过 / 2 已出库 / 3 审核不通过；品质 `quality_level`：1 优 / 2 良 / 3 差；货架 `status`：1 空闲 / 2 占用 / 3 停用。

---

## 九、项目目录结构

```
wms_test
├─ backend/                          # Spring Boot 后端
│  ├─ pom.xml
│  └─ src/main/
│     ├─ java/com/example/flower/
│     │  ├─ controller/              # 接口层（Auth/Purchase/Warehouse/Outbound/Delivery/Category/Role/Exception/Upload）
│     │  ├─ service/ + service/impl/ # 业务层（14 接口与实现一一对应）
│     │  ├─ mapper/                  # MyBatis-Plus Mapper（18 个）
│     │  ├─ entity/                  # 实体（15 个，@TableName 映射）
│     │  ├─ dto/                     # 请求/响应封装（ApiResponse、Login、各类 Request）
│     │  ├─ config/                  # 配置（CORS、Security、MVC、Swagger）
│     │  └─ utils/                   # 工具（JwtUtil、MD5Util）
│     └─ resources/
│        ├─ application.yml          # 端口 8080、数据源、上传目录等配置
│        └─ sql/init.sql             # 数据库初始化脚本
└─ frontend/                         # Vue 前端
   ├─ package.json / vue.config.js   # Vue CLI 5；dev 端口 3000，/api 代理至 8080
   └─ src/
      ├─ api/                        # axios 封装 + 各模块请求函数
      ├─ router/ store/              # 路由 / Vuex（登录态存于 localStorage）
      ├─ views/
      │  ├─ admin/                   # 系统管理员：首页、用户管理
      │  ├─ buyer/                   # 采购员：仪表盘、采购计划、采购记录、异常记录
      │  ├─ warehouse/               # 仓库：仪表盘、库存/品质审核、出库、货架
      │  ├─ delivery/                # 配送：仪表盘、配送任务、配送记录
      │  └─ Login.vue / LoginAdmin.vue   # 通用登录(含注册) / 管理员登录
```

---

## 十、本地运行指南

### 环境要求

- JDK 17+、Maven 3.6+
- MySQL 8.x
- Node.js 16+（npm）

### 1. 初始化数据库

```sql
-- 使用 MySQL 客户端执行初始化脚本（建库 + 建表 + 演示数据）
mysql -u root -p < backend/src/main/resources/sql/init.sql
```

如连接信息与默认不同，请修改 `backend/src/main/resources/application.yml` 中的数据源账号密码（默认 `root/root`，连接 `localhost:3306/flower_wms`）。

### 2. 启动后端

```bash
cd backend
mvn spring-boot:run        # 或 mvn clean package && java -jar target/flower-wms-1.0.0.jar
```

后端默认监听 `http://localhost:8080`，可访问 Swagger 文档 `http://localhost:8080/swagger-ui.html`。

### 3. 启动前端

```bash
cd frontend
npm install
npm run serve              # 开发服务器监听 3000，已将 /api 代理到 8080
```

### 4. 访问系统

- 管理员登录页：`http://localhost:3000/admin-login`
- 通用登录/注册页：`http://localhost:3000/`

生产部署时，前端 `npm run build` 产物放置到可访问静态目录，并将 `/api` 与 `/uploads` 反向代理到后端 8080 即可。

---

## 十一、默认测试账号

初始脚本预置账号密码均为 **123456**：

| 用户名       | 角色       | 建议登录入口              |
| ------------ | ---------- | ------------------------- |
| `admin`      | 系统管理员 | `/admin-login`            |
| `buyer1`     | 采购员     | `/`（自动跳转采购工作台） |
| `warehouse1` | 仓库管理员 | `/`                       |
| `delivery1`  | 仓库配送员 | `/`                       |

---

## 十二、补充说明与已知问题

> 以下内容基于对当前仓库代码的通读整理，供二次开发与答辩说明参考。

1. **数据库脚本与实体表名不一致**：`Inventory` 实体通过 `@TableName("inbound_record")` 映射到 `inbound_record` 表，而 `init.sql` 中建表名为 `inventory`（`inbound_record` 未在建表脚本中出现）。直接执行 `init.sql` 后启动会因缺表报错，**需二选一保持一致**（如将实体注解改为 `inventory`，或在库中 `RENAME TABLE inventory TO inbound_record`）。
2. **前端已声明 echarts 依赖但未被调用**：当前统计仪表盘由 Element UI 环形进度条实现，如需真实图表可直接使用已引入的 echarts。
3. **存在未接线的页面**：`views/warehouse/Audit.vue`（按订单审核）与 `views/warehouse/Rejected.vue`（驳回记录）尚未注册路由与菜单（品质审核实际在 `Inventory.vue` 行内完成）；其中 `Rejected.vue` 引用了 `api/modules.js` 中未导出的 `getRejectedRecords`，接入前需补充该封装。`/admin/categories` 路由复用 `Users.vue` 组件但未区分业务，目前展示的仍是用户管理页。
4. **鉴权处于"前端拦截、后端放行"阶段**：`SecurityConfig` 目前对 `/api/**` 全部 `permitAll()`，后端未强制校验 JWT；登录态校验主要依赖前端 Axios 拦截器（401 清空登录态跳回登录页）。若要形成严格权限体系，需在后端补充 JWT 过滤器与基于 `sys_permission` 的接口鉴权。
5. **前端未配置全局路由守卫**：登录态存于 localStorage，未登录直访业务路由不会被重定向回登录页，可按需补充。
6. **质检员（roleId=6）**角色与部门"质检部"已内置并可在注册时选择，但前端暂无对应工作台页面与路由。
