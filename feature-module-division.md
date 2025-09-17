# KMS-Wind 功能模块划分设计文档

## 1. 功能划分概述

### 1.1 划分原则
- **领域驱动设计(DDD)**: 按业务域划分服务边界
- **独立性**: 每个模块可独立开发、测试、部署
- **松耦合**: 模块间通过API/事件通信，避免直接依赖
- **高内聚**: 相关功能聚合在同一模块内
- **渐进式**: 支持分阶段独立开发和上线

### 1.2 模块分层策略

```
┌─────────────────────────────────────────────────────────┐
│                     前端应用层                          │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                  基础设施层                             │
│    API网关 | 配置中心 | 服务发现 | 监控告警               │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                   业务服务层                            │
│   认证服务 | 空间服务 | 文档服务 | 搜索服务 | 通知服务    │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                   数据存储层                            │
│   PostgreSQL | Redis | Elasticsearch | MinIO          │
└─────────────────────────────────────────────────────────┘
```

## 2. 核心功能模块划分

### 2.1 基础设施模块 (Phase 0)

#### M-00-001: API网关模块
**模块名称**: kms-gateway-service
**核心职责**:
- 请求路由和负载均衡
- 统一认证和授权校验
- 限流和熔断保护
- 跨域和安全策略

**独立测试能力**:
- 路由规则测试
- 限流策略测试
- 认证拦截测试
- 健康检查测试

**数据依赖**:
- 配置中心（路由配置）
- Redis（限流计数器）

**API接口**:
```
GET  /health - 健康检查
GET  /metrics - 监控指标
POST /admin/routes - 路由管理
```

#### M-00-002: 配置中心模块
**模块名称**: kms-config-service
**核心职责**:
- 集中配置管理
- 配置热更新
- 环境隔离
- 配置版本控制

**独立测试能力**:
- 配置CRUD测试
- 配置推送测试
- 版本回滚测试
- 权限控制测试

**数据依赖**:
- Git仓库（配置存储）
- PostgreSQL（配置元数据）

**API接口**:
```
GET    /config/{app}/{profile} - 获取配置
POST   /config/refresh - 配置刷新
GET    /config/history - 配置历史
```

### 2.2 核心业务模块 (Phase 1 - MVP)

#### M-01-001: 用户认证模块
**模块名称**: kms-auth-service
**核心职责**:
- 用户注册和登录
- JWT Token管理
- OAuth2.0集成
- 用户基础信息管理

**独立测试能力**:
- 用户注册流程测试
- 登录认证测试
- Token生成和验证测试
- 密码加密和校验测试

**数据存储**:
- 用户表（users）
- 用户会话表（user_sessions）

**API接口**:
```
POST /auth/register - 用户注册
POST /auth/login - 用户登录
POST /auth/logout - 用户登出
GET  /auth/profile - 获取用户信息
PUT  /auth/profile - 更新用户信息
POST /auth/verify-token - Token验证
```

**测试策略**:
- 单元测试：密码加密、Token生成
- 集成测试：数据库操作、外部OAuth
- 端到端测试：完整登录流程

#### M-01-002: 基础空间模块
**模块名称**: kms-space-basic-service
**核心职责**:
- 空间CRUD操作
- 空间基础配置
- 空间成员管理（基础版）

**独立测试能力**:
- 空间创建和配置测试
- 成员邀请和移除测试
- 空间权限基础测试

**数据存储**:
- 空间表（spaces）
- 空间成员表（space_members）

**API接口**:
```
POST /spaces - 创建空间
GET  /spaces/{id} - 获取空间详情
PUT  /spaces/{id} - 更新空间配置
DELETE /spaces/{id} - 删除空间
POST /spaces/{id}/members - 添加成员
DELETE /spaces/{id}/members/{userId} - 移除成员
```

#### M-01-003: 基础文档模块
**模块名称**: kms-document-basic-service
**核心职责**:
- 文档CRUD操作
- 文档基础元数据管理
- 简单的文档组织结构

**独立测试能力**:
- 文档创建和编辑测试
- 文档保存和读取测试
- 文档删除和恢复测试
- 文档层级结构测试

**数据存储**:
- 页面表（pages）
- 页面内容表（page_contents）

**API接口**:
```
POST /documents - 创建文档
GET  /documents/{id} - 获取文档
PUT  /documents/{id} - 更新文档
DELETE /documents/{id} - 删除文档
GET  /documents/tree/{spaceId} - 获取文档树
```

#### M-01-004: 基础文件存储模块
**模块名称**: kms-file-basic-service
**核心职责**:
- 文件上传和下载
- 文件基础元数据管理
- 简单的文件权限控制

**独立测试能力**:
- 文件上传测试
- 文件下载测试
- 文件删除测试
- 文件权限测试

**数据存储**:
- 附件表（attachments）
- MinIO对象存储

**API接口**:
```
POST /files/upload - 文件上传
GET  /files/{id}/download - 文件下载
DELETE /files/{id} - 删除文件
GET  /files/{id}/info - 文件信息
```

### 2.3 增强功能模块 (Phase 2)

#### M-02-001: 权限管理模块
**模块名称**: kms-permission-service
**核心职责**:
- 细粒度权限控制
- 角色和权限管理
- 权限继承和覆盖
- 权限审计

**独立测试能力**:
- 权限分配和回收测试
- 角色管理测试
- 权限继承测试
- 权限审计测试

**数据存储**:
- 角色表（roles）
- 权限表（permissions）
- 用户角色表（user_roles）
- 资源权限表（resource_permissions）

**API接口**:
```
GET  /permissions/check - 权限检查
POST /permissions/grant - 授权
DELETE /permissions/revoke - 撤权
GET  /permissions/audit - 权限审计
POST /roles - 创建角色
GET  /roles/{id}/permissions - 角色权限
```

#### M-02-002: 文档版本模块
**模块名称**: kms-version-service
**核心职责**:
- 文档版本管理
- 版本比较和差异
- 版本回滚和恢复
- 版本分支和合并

**独立测试能力**:
- 版本创建和保存测试
- 版本比较测试
- 版本回滚测试
- 版本合并测试

**数据存储**:
- 页面版本表（page_versions）
- 版本差异表（version_diffs）

**API接口**:
```
GET  /versions/{pageId} - 获取版本列表
GET  /versions/{pageId}/{version} - 获取特定版本
POST /versions/{pageId}/compare - 版本比较
POST /versions/{pageId}/rollback - 版本回滚
```

#### M-02-003: 基础搜索模块
**模块名称**: kms-search-basic-service
**核心职责**:
- 基础全文搜索
- 搜索索引管理
- 简单的搜索过滤

**独立测试能力**:
- 搜索索引构建测试
- 关键词搜索测试
- 搜索结果排序测试
- 搜索权限过滤测试

**数据存储**:
- Elasticsearch索引
- 搜索日志表（search_logs）

**API接口**:
```
GET  /search?q={keyword} - 基础搜索
POST /search/index/{documentId} - 索引文档
DELETE /search/index/{documentId} - 删除索引
GET  /search/suggest?q={partial} - 搜索建议
```

#### M-02-004: 基础通知模块
**模块名称**: kms-notification-basic-service
**核心职责**:
- 基础消息通知
- 通知模板管理
- 通知发送策略

**独立测试能力**:
- 通知发送测试
- 通知模板测试
- 通知订阅测试
- 通知历史测试

**数据存储**:
- 通知表（notifications）
- 通知模板表（notification_templates）

**API接口**:
```
POST /notifications/send - 发送通知
GET  /notifications/user/{userId} - 用户通知
PUT  /notifications/{id}/read - 标记已读
GET  /notifications/subscribe - 订阅设置
```

### 2.4 协作功能模块 (Phase 3)

#### M-03-001: 实时编辑模块
**模块名称**: kms-realtime-edit-service
**核心职责**:
- 实时协作编辑
- 冲突检测和解决
- 编辑状态同步
- 编辑锁管理

**独立测试能力**:
- 多用户编辑测试
- 冲突解决测试
- WebSocket连接测试
- 编辑锁测试

**数据存储**:
- 编辑会话表（edit_sessions）
- 编辑锁表（edit_locks）
- Redis（实时状态）

**API接口**:
```
WebSocket /ws/edit/{documentId} - 实时编辑连接
POST /edit/lock/{documentId} - 获取编辑锁
DELETE /edit/lock/{documentId} - 释放编辑锁
GET  /edit/status/{documentId} - 编辑状态
```

#### M-03-002: 评论系统模块
**模块名称**: kms-comment-service
**核心职责**:
- 文档评论管理
- 嵌套回复功能
- 评论解决状态
- @提及功能

**独立测试能力**:
- 评论CRUD测试
- 嵌套回复测试
- 评论通知测试
- @提及解析测试

**数据存储**:
- 评论表（comments）
- 提及表（mentions）

**API接口**:
```
POST /comments - 添加评论
GET  /comments/{documentId} - 获取评论列表
PUT  /comments/{id} - 更新评论
DELETE /comments/{id} - 删除评论
POST /comments/{id}/resolve - 解决评论
```

### 2.5 高级功能模块 (Phase 4)

#### M-04-001: 智能搜索模块
**模块名称**: kms-search-advanced-service
**核心职责**:
- 智能搜索推荐
- 个性化搜索
- 搜索分析统计
- 语义搜索

**独立测试能力**:
- 智能推荐测试
- 个性化算法测试
- 搜索统计测试
- 语义匹配测试

#### M-04-002: 模板管理模块
**模块名称**: kms-template-service
**核心职责**:
- 文档模板管理
- 模板版本控制
- 模板分类和标签
- 模板使用统计

#### M-04-003: 高级通知模块
**模块名称**: kms-notification-advanced-service
**核心职责**:
- 实时推送通知
- 邮件通知集成
- 通知偏好设置
- 通知聚合和摘要

#### M-04-004: 数据分析模块
**模块名称**: kms-analytics-service
**核心职责**:
- 用户行为分析
- 内容访问统计
- 系统性能监控
- 业务指标报表

## 3. 模块依赖关系

### 3.1 依赖层次图

```
Phase 4: [智能搜索] [模板管理] [高级通知] [数据分析]
           ↑           ↑           ↑           ↑
Phase 3: [实时编辑] ← [评论系统] ← [基础通知] ← [权限管理]
           ↑                       ↑           ↑
Phase 2: [基础搜索] ← [文档版本] ← [权限管理] ← [基础通知]
           ↑           ↑           ↑           ↑
Phase 1: [基础文档] ← [基础空间] ← [用户认证] ← [文件存储]
           ↑           ↑           ↑           ↑
Phase 0: [API网关] ← [配置中心] ← [服务发现] ← [监控告警]
```

### 3.2 核心依赖规则

1. **Phase 0为所有模块提供基础设施支持**
2. **Phase 1模块彼此可以通过API调用**
3. **高层模块可以依赖低层模块**
4. **同层模块之间通过事件通信**
5. **不允许反向依赖（高层→低层）**

## 4. 数据库划分策略

### 4.1 数据库分离原则

每个核心模块都有独立的数据库实例：

```
kms_auth_db      - 用户认证模块
kms_space_db     - 空间管理模块
kms_document_db  - 文档管理模块
kms_file_db      - 文件存储模块
kms_permission_db - 权限管理模块
kms_search_db    - 搜索模块
kms_notification_db - 通知模块
kms_analytics_db - 分析模块
```

### 4.2 共享数据处理

对于需要跨模块的数据：
- 通过API调用获取
- 通过事件同步更新
- 建立数据视图（只读）
- 使用分布式事务（必要时）

## 5. 接口设计规范

### 5.1 REST API规范

```
# 资源操作规范
GET    /{module}/{resource}        - 获取资源列表
GET    /{module}/{resource}/{id}   - 获取单个资源
POST   /{module}/{resource}        - 创建资源
PUT    /{module}/{resource}/{id}   - 完整更新资源
PATCH  /{module}/{resource}/{id}   - 部分更新资源
DELETE /{module}/{resource}/{id}   - 删除资源

# 子资源操作规范
GET    /{module}/{resource}/{id}/{subresource}
POST   /{module}/{resource}/{id}/{subresource}
```

### 5.2 事件通信规范

```
# 事件命名规范
{domain}.{entity}.{action}

# 示例事件
user.account.created      - 用户账户创建
space.member.added        - 空间成员添加
document.content.updated  - 文档内容更新
file.attachment.uploaded  - 文件附件上传
```

### 5.3 错误处理规范

```json
{
  "error": {
    "code": "DOC_001",
    "message": "文档不存在",
    "details": "Document with id 123 not found",
    "timestamp": "2024-01-15T10:30:00Z",
    "module": "kms-document-service"
  }
}
```

## 6. 测试策略

### 6.1 模块独立测试

每个模块必须包含：
- **单元测试**: 覆盖率 > 80%
- **集成测试**: API接口测试
- **契约测试**: 模块间接口契约
- **端到端测试**: 核心业务流程

### 6.2 模块间集成测试

- **API契约测试**: 确保接口兼容性
- **事件消息测试**: 确保事件正确传递
- **数据一致性测试**: 跨模块数据同步
- **性能测试**: 模块间调用性能

### 6.3 系统级测试

- **功能回归测试**: 完整业务流程
- **性能压力测试**: 系统负载测试
- **安全渗透测试**: 安全漏洞测试
- **容灾恢复测试**: 故障恢复能力

## 7. 部署策略

### 7.1 独立部署能力

每个模块支持：
- **独立Docker镜像**
- **独立Kubernetes部署**
- **独立扩缩容策略**
- **独立监控告警**

### 7.2 灰度发布策略

```
开发环境 → 测试环境 → 预发布环境 → 5%用户 → 50%用户 → 全量发布
```

### 7.3 回滚策略

- **版本标记**: 每个模块独立版本
- **快速回滚**: 5分钟内完成回滚
- **数据兼容**: 保证数据向前兼容
- **依赖检查**: 回滚前检查依赖影响

---

**文档版本**: v1.0.0
**创建时间**: 2024年1月15日
**维护团队**: KMS-Wind Architecture Team