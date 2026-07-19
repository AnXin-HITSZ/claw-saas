# 项目架构说明

## 总体架构

```text
gateway
  统一入口、认证前置、限流、路由

backend-for-frontend
  面向前端页面的聚合 API

agent-marketplace-service
  Agent 发布、版本、发现、安装申请

conversation-service
  对话线程、消息、展示结构

runtime-service
  Orchestrator、Agent run、call_agent、FastAPI 调用、审批恢复

billing-service
  用量、额度、套餐、账单

pyclaw-runtime-api FastAPI
  Agent 执行引擎、工具执行、workspace 操作
```