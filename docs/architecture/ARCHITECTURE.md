# 项目架构说明

## 项目定位

`claw-saas` 是产品级 monorepo，用于承载 Claw SaaS 的前端、Java 后端、Python Runtime、部署配置与工程脚本。

根目录不直接作为某一种技术栈的工程根目录。各技术栈放在各自目录下独立管理：

```text
claw-saas/
  backend/    Java / Spring Boot 后端服务
  frontend/   前端应用
  runtime/    Python / FastAPI Agent 执行引擎
  deploy/     本地与生产部署配置
  scripts/    工程脚本
  docs/       架构、数据库、接口与运维文档
```

## 目标目录结构

```text
claw-saas/
  README.md

  docs/
    architecture/
      ARCHITECTURE.md
      database-schema.md

  backend/
    pom.xml

    gateway/
      pom.xml
      src/main/java/com/clawsaas/gateway/
      src/main/resources/application.properties

    backend-for-frontend/
      pom.xml
      src/main/java/com/clawsaas/backendforfrontend/
      src/main/resources/application.properties

    agent-marketplace-service/
      pom.xml
      src/main/java/com/clawsaas/agentmarketplace/
      src/main/resources/application.properties

    conversation-service/
      pom.xml
      src/main/java/com/clawsaas/conversationservice/
      src/main/resources/application.properties

    runtime-service/
      pom.xml
      src/main/java/com/clawsaas/runtimeservice/
      src/main/resources/application.properties

    billing-service/
      pom.xml
      src/main/java/com/clawsaas/billingservice/
      src/main/resources/application.properties

  frontend/
    package.json
    src/
    public/

  runtime/
    pyclaw-runtime-api/
      pyproject.toml
      app/
        main.py
        api/
        runtime/
        tools/
        workspace/
        schemas/

  deploy/
    docker-compose.yml
    env/
    k8s/

  scripts/
```

## 总体架构

```text
Frontend
  -> gateway
  -> backend-for-frontend
  -> domain services
  -> database / pyclaw-runtime-api / external systems
```

模块职责如下：

```text
gateway
  统一入口、认证前置、限流、路由、CORS、请求追踪、健康检查

backend-for-frontend
  面向前端页面的聚合 API，负责页面级数据组装、裁剪和格式转换

agent-marketplace-service
  Agent 发布、版本、发现、搜索、安装申请、用户已安装 Agent

conversation-service
  对话线程、消息、展示结构、会话状态

runtime-service
  Orchestrator、Agent run、call_agent、FastAPI 调用、审批恢复

billing-service
  用量、额度、套餐、账单、计费规则

pyclaw-runtime-api
  Python / FastAPI Agent 执行引擎、工具执行、workspace 操作
```

## 分层边界

### Gateway

Gateway 是系统公网入口，职责是入口治理，不承载业务聚合。

应该放在 Gateway 的能力：

```text
路由转发
认证前置
CORS
限流
请求 ID
访问日志
健康检查
统一入口错误响应
```

不应该放在 Gateway 的能力：

```text
页面数据聚合
Agent 发布规则
对话状态流转
套餐额度扣减
Agent 执行编排
```

### BFF

`backend-for-frontend` 是 Backend For Frontend，面向前端页面提供 API。

BFF 负责把多个领域服务的数据聚合成前端需要的形状，例如：

```text
GET /api/workbench/home
  -> conversation-service 最近会话
  -> agent-marketplace-service 推荐 Agent
  -> billing-service 当前额度
  -> runtime-service 最近运行状态
```

BFF 不负责保存领域事实，也不替代领域服务的业务规则。

### Domain Services

领域服务负责稳定的业务能力和业务事实。

```text
conversation-service
  负责会话与消息事实

runtime-service
  负责运行编排与执行状态

agent-marketplace-service
  负责 Agent 市场与安装事实

billing-service
  负责额度、用量、账单事实
```

领域服务默认不直接暴露给前端，由 BFF 调用。

### Runtime API

`runtime/pyclaw-runtime-api` 是 Python / FastAPI 服务，负责 Agent 执行层能力。

Java 后端通过 `runtime-service` 调用它，不建议由 Gateway 或 BFF 直接调用。

## 后端工程结构

`backend/` 是 Maven 多模块父工程，不是 Spring Boot 应用。

```text
backend/pom.xml
  packaging=pom
  管理后端统一 groupId、version、Java 版本、Spring Boot / Spring Cloud 版本
  聚合各个 Spring Boot 子模块
```

后端子模块均是独立 Spring Boot 应用：

```text
backend/gateway
backend/backend-for-frontend
backend/agent-marketplace-service
backend/conversation-service
backend/runtime-service
backend/billing-service
```

推荐端口：

```text
gateway                    8080
backend-for-frontend       8081
conversation-service       8082
runtime-service            8083
agent-marketplace-service  8084
billing-service            8085
pyclaw-runtime-api         8090
```

## 分阶段重构计划

### 第一阶段：补齐后端骨架

目标：让 `backend/` 成为完整的 Maven 多模块后端工程。

任务：

```text
1. 保持 backend/pom.xml 作为父 POM
2. 统一所有子模块继承 backend/pom.xml
3. 补齐 backend/agent-marketplace-service
4. 确认 backend/pom.xml modules 完整
5. 为每个 Spring Boot 服务设置 application.name 和 server.port
6. 保持每个服务可以独立启动
```

完成标准：

```text
cd backend
mvn validate
```

可以成功识别所有后端模块。

### 第二阶段：跑通最小后端链路

目标：验证请求可以从 Gateway 进入 BFF，并具备继续调用领域服务的基础。

任务：

```text
1. gateway 引入 Spring Cloud Gateway
2. gateway 配置 /api/** -> backend-for-frontend
3. backend-for-frontend 提供最小测试接口
4. conversation-service / runtime-service 提供最小健康或测试接口
5. BFF 通过 HTTP client 调用一个领域服务
6. 验证 Gateway -> BFF -> Service 链路
```

完成标准：

```text
GET http://localhost:8080/api/ping
```

可以经 Gateway 转发到 BFF。

后续再验证：

```text
GET http://localhost:8080/api/conversations/ping
```

可以经 Gateway -> BFF -> conversation-service 返回。

### 第三阶段：迁移核心业务服务

目标：从旧项目 `D:\projects\personal\pyclaw` 迁移核心业务能力到新的服务边界。

建议顺序：

```text
1. conversation-service
2. runtime-service
3. agent-marketplace-service
4. billing-service
```

迁移原则：

```text
先迁移领域模型和用例
再迁移接口
最后迁移持久化与外部依赖
```

每个服务内部建议采用：

```text
controller/
  暴露内部 HTTP API

application/
  编排业务用例

domain/
  表达业务对象、状态和值对象

repository/
  访问数据库

client/
  调用其他服务或外部系统

dto/
  请求与响应对象

config/
  服务配置
```

### 第四阶段：补齐 Python Runtime

目标：建立 `runtime/pyclaw-runtime-api`，承接 Agent 执行引擎能力。

任务：

```text
1. 创建 runtime/pyclaw-runtime-api
2. 建立 FastAPI 应用入口
3. 迁移 Agent 执行、工具执行、workspace 操作
4. 定义 runtime-service -> pyclaw-runtime-api 的接口契约
5. 明确审批恢复、执行状态、错误响应协议
```

调用关系：

```text
backend/runtime-service
  -> runtime/pyclaw-runtime-api
```

不建议：

```text
gateway -> pyclaw-runtime-api
backend-for-frontend -> pyclaw-runtime-api
```

### 第五阶段：补齐前端工程

目标：建立 `frontend/`，通过 Gateway 访问后端能力。

任务：

```text
1. 创建 frontend/
2. 建立前端构建与本地开发脚本
3. 所有后端请求统一访问 Gateway
4. 前端不直接访问领域服务
5. 前端不直接访问 pyclaw-runtime-api
```

前端调用规范：

```text
Frontend
  -> http://localhost:8080/api/**
```

### 第六阶段：补齐部署与工程脚本

目标：让本地开发、联调、部署具备统一入口。

任务：

```text
1. 创建 deploy/
2. 创建 deploy/docker-compose.yml
3. 创建 deploy/env/ 环境变量模板
4. 预留 deploy/k8s/
5. 创建 scripts/ 存放本地开发与初始化脚本
```

`docker-compose.yml` 最终应覆盖：

```text
gateway
backend-for-frontend
conversation-service
runtime-service
agent-marketplace-service
billing-service
pyclaw-runtime-api
frontend
postgres
redis
```

### 第七阶段：补齐平台治理能力

目标：在核心链路跑通后补齐生产化能力。

任务：

```text
认证与授权
租户上下文
请求 ID 与链路追踪
统一错误码
限流
服务间鉴权
配置管理
日志规范
数据库迁移
接口文档
CI 校验
```

治理能力优先落在合适层：

```text
gateway
  入口认证、限流、CORS、请求 ID

backend-for-frontend
  页面级鉴权、聚合错误处理

domain services
  领域权限、业务规则、数据一致性

runtime-service
  执行权限、审批恢复、运行状态一致性
```

## 当前缺失目录

截至当前阶段，已存在：

```text
backend/
  gateway/
  backend-for-frontend/
  conversation-service/
  runtime-service/
  billing-service/

docs/
```

仍需补齐：

```text
backend/agent-marketplace-service/
frontend/
runtime/pyclaw-runtime-api/
deploy/
scripts/
```

## 当前优先级

近期优先级建议：

```text
1. 补齐 backend/agent-marketplace-service
2. 验证 Gateway -> BFF 真实转发链路
3. 为 BFF 和领域服务增加最小 ping 接口
4. 跑通 Gateway -> BFF -> conversation-service
5. 开始从旧项目 pyclaw 迁移 conversation-service 和 runtime-service
```
