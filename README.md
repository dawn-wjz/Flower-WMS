# 仓库管理系统 (WMS)

## 项目搭建

### 前置条件

- Node.js (v14)
- Java (JDK 11)
- Maven

### 后端

1. 进入 `backend` 目录。
2. 运行 `mvn spring-boot:run`。
3. 服务器会在端口 8080 启动。

### 前端

1. 进入 `frontend` 目录。
2. 安装依赖：`npm install`。
3. 启动开发服务器：`npm run serve`。
4. 在浏览器打开 `http://localhost:8081`（或终端显示的端口）。

## 功能

- **仓库管理**：库存跟踪、货架管理、入库/出库操作。
- **采购计划**：创建和管理采购计划。
- **质量控制**：根据质量批准或拒收入库鲜花。
- **配送管理**：管理配送任务。

## 技术栈

- **前端**：Vue.js、Element UI、Axios。
- **后端**：Spring Boot、MyBatis Plus、MySQL。


**基本系统运行必须功能已完成，某些功能有待优化~~~~~~~~**
