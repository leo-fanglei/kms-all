# KMS-Wind API 接口设计文档

## 1. API 设计概述

### 1.1 API 设计原则
- **RESTful 设计**: 遵循REST架构风格
- **版本控制**: 通过URL路径进行版本管理
- **统一响应**: 标准化的响应格式
- **安全认证**: JWT Token + OAuth2.0
- **幂等性**: 确保关键操作的幂等性
- **分页查询**: 大数据量查询支持分页

### 1.2 通用规范

#### 1.2.1 API Base URL
```
开发环境: https://dev-api.kms-wind.com/api/v1
测试环境: https://test-api.kms-wind.com/api/v1
生产环境: https://api.kms-wind.com/api/v1
```

#### 1.2.2 HTTP 状态码规范
```
200 OK - 请求成功
201 Created - 资源创建成功
202 Accepted - 请求已接受，异步处理
204 No Content - 成功但无返回内容
400 Bad Request - 请求参数错误
401 Unauthorized - 未认证
403 Forbidden - 无权限访问
404 Not Found - 资源不存在
409 Conflict - 资源冲突
422 Unprocessable Entity - 请求参数验证失败
429 Too Many Requests - 请求频率过高
500 Internal Server Error - 服务器内部错误
502 Bad Gateway - 网关错误
503 Service Unavailable - 服务不可用
```

#### 1.2.3 统一响应格式
```json
{
  "success": true,
  "code": 200,
  "message": "操作成功",
  "data": {},
  "timestamp": "2024-01-15T10:30:00Z",
  "traceId": "550e8400-e29b-41d4-a716-446655440000"
}

// 错误响应
{
  "success": false,
  "code": 400,
  "message": "请求参数错误",
  "errors": [
    {
      "field": "username",
      "code": "REQUIRED",
      "message": "用户名不能为空"
    }
  ],
  "timestamp": "2024-01-15T10:30:00Z",
  "traceId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### 1.2.4 分页响应格式
```json
{
  "success": true,
  "code": 200,
  "message": "查询成功",
  "data": {
    "content": [],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 100,
      "totalPages": 5,
      "hasNext": true,
      "hasPrevious": false
    }
  }
}
```

## 2. 认证授权服务 (Auth Service)

### 2.1 用户认证

#### 2.1.1 用户登录
```http
POST /auth/login
Content-Type: application/json

{
  "username": "john.doe@example.com",
  "password": "password123",
  "rememberMe": true
}
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "user": {
      "id": 1,
      "username": "john.doe",
      "email": "john.doe@example.com",
      "displayName": "John Doe",
      "avatar": "https://example.com/avatar.jpg"
    }
  }
}
```

#### 2.1.2 令牌刷新
```http
POST /auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### 2.1.3 用户退出
```http
POST /auth/logout
Authorization: Bearer {accessToken}
```

#### 2.1.4 OAuth2.0 授权
```http
GET /auth/oauth2/authorize/{provider}?redirect_uri={url}
# provider: google, github, microsoft, etc.
```

### 2.2 用户管理

#### 2.2.1 获取当前用户信息
```http
GET /auth/user/profile
Authorization: Bearer {accessToken}
```

#### 2.2.2 更新用户信息
```http
PUT /auth/user/profile
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "displayName": "John Doe",
  "avatar": "https://example.com/new-avatar.jpg",
  "phone": "+1234567890"
}
```

#### 2.2.3 修改密码
```http
PUT /auth/user/password
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "currentPassword": "oldPassword",
  "newPassword": "newPassword123"
}
```

### 2.3 权限管理

#### 2.3.1 获取用户权限
```http
GET /auth/user/permissions
Authorization: Bearer {accessToken}
```

#### 2.3.2 检查资源权限
```http
GET /auth/permissions/check?resourceType=SPACE&resourceId=123&permission=EDIT
Authorization: Bearer {accessToken}
```

## 3. 空间管理服务 (Space Service)

### 3.1 空间基础操作

#### 3.1.1 获取空间列表
```http
GET /spaces?page=1&size=20&type=TEAM&visibility=PUBLIC
Authorization: Bearer {accessToken}
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 1,
        "spaceKey": "TEAM",
        "name": "团队空间",
        "description": "团队协作空间",
        "type": "TEAM",
        "visibility": "PRIVATE",
        "logoUrl": "https://example.com/space-logo.jpg",
        "homepageId": 10,
        "memberCount": 25,
        "pageCount": 150,
        "createdAt": "2024-01-01T00:00:00Z",
        "createdBy": {
          "id": 1,
          "username": "admin",
          "displayName": "Administrator"
        }
      }
    ],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 5,
      "totalPages": 1
    }
  }
}
```

#### 3.1.2 创建空间
```http
POST /spaces
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "spaceKey": "PROJECT_A",
  "name": "项目A空间",
  "description": "项目A的协作空间",
  "type": "PROJECT",
  "visibility": "PRIVATE",
  "logoUrl": "https://example.com/logo.jpg"
}
```

#### 3.1.3 获取空间详情
```http
GET /spaces/{spaceId}
Authorization: Bearer {accessToken}
```

#### 3.1.4 更新空间信息
```http
PUT /spaces/{spaceId}
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "name": "更新后的空间名称",
  "description": "更新后的描述",
  "logoUrl": "https://example.com/new-logo.jpg",
  "settings": {
    "enableComments": true,
    "allowAnonymousView": false
  }
}
```

#### 3.1.5 删除空间
```http
DELETE /spaces/{spaceId}
Authorization: Bearer {accessToken}
```

### 3.2 空间成员管理

#### 3.2.1 获取空间成员列表
```http
GET /spaces/{spaceId}/members?page=1&size=20&role=ADMIN
Authorization: Bearer {accessToken}
```

#### 3.2.2 邀请成员
```http
POST /spaces/{spaceId}/members/invite
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "emails": ["user1@example.com", "user2@example.com"],
  "roleId": 2,
  "message": "欢迎加入我们的团队空间"
}
```

#### 3.2.3 更新成员角色
```http
PUT /spaces/{spaceId}/members/{userId}/role
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "roleId": 3
}
```

#### 3.2.4 移除成员
```http
DELETE /spaces/{spaceId}/members/{userId}
Authorization: Bearer {accessToken}
```

### 3.3 空间权限管理

#### 3.3.1 获取空间权限配置
```http
GET /spaces/{spaceId}/permissions
Authorization: Bearer {accessToken}
```

#### 3.3.2 设置空间权限
```http
POST /spaces/{spaceId}/permissions
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "userId": 123,
  "permissions": ["VIEW", "EDIT", "COMMENT"]
}
```

## 4. 文档管理服务 (Document Service)

### 4.1 页面基础操作

#### 4.1.1 获取页面列表
```http
GET /spaces/{spaceId}/pages?page=1&size=20&parentId=0&status=PUBLISHED
Authorization: Bearer {accessToken}
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 1,
        "title": "入门指南",
        "slug": "getting-started",
        "status": "PUBLISHED",
        "versionNumber": 5,
        "parentId": null,
        "hasChildren": true,
        "childrenCount": 3,
        "labels": ["guide", "tutorial"],
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-15T10:30:00Z",
        "createdBy": {
          "id": 1,
          "displayName": "John Doe"
        },
        "updatedBy": {
          "id": 2,
          "displayName": "Jane Smith"
        }
      }
    ],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 25
    }
  }
}
```

#### 4.1.2 创建页面
```http
POST /spaces/{spaceId}/pages
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "title": "新建页面",
  "content": "# 页面标题\n\n页面内容...",
  "parentId": 10,
  "templateId": 5,
  "labels": ["draft", "documentation"],
  "status": "DRAFT"
}
```

#### 4.1.3 获取页面详情
```http
GET /pages/{pageId}?version=latest
Authorization: Bearer {accessToken}
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "API设计指南",
    "content": "# API设计指南\n\n本文档介绍了API设计的最佳实践...",
    "contentType": "MARKDOWN",
    "status": "PUBLISHED",
    "versionNumber": 3,
    "wordCount": 1250,
    "excerpt": "本文档介绍了API设计的最佳实践，包括RESTful设计原则...",
    "labels": ["api", "guide", "best-practice"],
    "metadata": {
      "readingTime": 5,
      "difficulty": "intermediate"
    },
    "space": {
      "id": 1,
      "spaceKey": "TEAM",
      "name": "团队空间"
    },
    "parent": {
      "id": 5,
      "title": "开发文档"
    },
    "attachments": [
      {
        "id": 1,
        "filename": "api-diagram.png",
        "fileSize": 102400,
        "downloadUrl": "/files/1/download"
      }
    ],
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-15T10:30:00Z",
    "createdBy": {
      "id": 1,
      "displayName": "John Doe"
    },
    "updatedBy": {
      "id": 2,
      "displayName": "Jane Smith"
    }
  }
}
```

#### 4.1.4 更新页面
```http
PUT /pages/{pageId}
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "title": "更新后的标题",
  "content": "更新后的内容...",
  "labels": ["updated", "guide"],
  "changeSummary": "更新了API示例代码"
}
```

#### 4.1.5 删除页面
```http
DELETE /pages/{pageId}
Authorization: Bearer {accessToken}
```

### 4.2 页面版本管理

#### 4.2.1 获取页面版本历史
```http
GET /pages/{pageId}/versions?page=1&size=10
Authorization: Bearer {accessToken}
```

#### 4.2.2 获取特定版本内容
```http
GET /pages/{pageId}/versions/{version}
Authorization: Bearer {accessToken}
```

#### 4.2.3 版本比较
```http
GET /pages/{pageId}/versions/{version1}/compare/{version2}
Authorization: Bearer {accessToken}
```

#### 4.2.4 版本回滚
```http
POST /pages/{pageId}/versions/{version}/restore
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "changeSummary": "回滚到版本2，撤销错误修改"
}
```

### 4.3 页面树形结构

#### 4.3.1 获取空间页面树
```http
GET /spaces/{spaceId}/page-tree?maxDepth=3
Authorization: Bearer {accessToken}
```

#### 4.3.2 移动页面
```http
PUT /pages/{pageId}/move
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "newParentId": 15,
  "position": 2
}
```

#### 4.3.3 获取页面子页面
```http
GET /pages/{pageId}/children?includeAll=false
Authorization: Bearer {accessToken}
```

### 4.4 页面模板管理

#### 4.4.1 获取模板列表
```http
GET /templates?spaceId={spaceId}&type=USER&page=1&size=20
Authorization: Bearer {accessToken}
```

#### 4.4.2 创建模板
```http
POST /templates
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "name": "会议纪要模板",
  "description": "标准的会议纪要格式",
  "content": "# 会议纪要\n\n**时间**: \n**参与人员**: \n\n## 议题\n\n## 决议事项\n",
  "spaceId": 1,
  "isGlobal": false,
  "thumbnailUrl": "https://example.com/template-thumb.jpg"
}
```

#### 4.4.3 应用模板创建页面
```http
POST /spaces/{spaceId}/pages/from-template
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "templateId": 5,
  "title": "2024年第1季度会议纪要",
  "parentId": 10
}
```

## 5. 搜索服务 (Search Service)

### 5.1 全文搜索

#### 5.1.1 搜索内容
```http
GET /search?q=API%20design&type=PAGE&spaceId=1&page=1&size=20
Authorization: Bearer {accessToken}
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 1,
        "type": "PAGE",
        "title": "API设计指南",
        "excerpt": "本文档介绍了<em>API</em> <em>design</em>的最佳实践，包括RESTful设计原则...",
        "url": "/spaces/1/pages/1",
        "space": {
          "id": 1,
          "name": "团队空间",
          "spaceKey": "TEAM"
        },
        "score": 0.95,
        "highlights": {
          "title": ["<em>API设计</em>指南"],
          "content": ["介绍了<em>API design</em>的最佳实践"]
        },
        "lastModified": "2024-01-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 15
    },
    "facets": {
      "types": {
        "PAGE": 12,
        "COMMENT": 3
      },
      "spaces": {
        "团队空间": 10,
        "项目空间": 5
      }
    },
    "queryTime": 25,
    "suggestions": ["API design patterns", "API documentation"]
  }
}
```

#### 5.1.2 高级搜索
```http
POST /search/advanced
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "query": "API design",
  "filters": {
    "types": ["PAGE", "COMMENT"],
    "spaces": [1, 2],
    "authors": [1, 2],
    "dateRange": {
      "from": "2024-01-01",
      "to": "2024-01-31"
    },
    "labels": ["api", "documentation"]
  },
  "sort": "relevance", // relevance, created, updated, title
  "page": 1,
  "size": 20
}
```

#### 5.1.3 搜索建议
```http
GET /search/suggestions?q=api&limit=10
Authorization: Bearer {accessToken}
```

#### 5.1.4 最近搜索
```http
GET /search/recent?limit=10
Authorization: Bearer {accessToken}
```

### 5.2 智能推荐

#### 5.2.1 相关页面推荐
```http
GET /pages/{pageId}/related?limit=5
Authorization: Bearer {accessToken}
```

#### 5.2.2 个性化推荐
```http
GET /recommendations/pages?limit=10
Authorization: Bearer {accessToken}
```

## 6. 通知服务 (Notification Service)

### 6.1 通知管理

#### 6.1.1 获取通知列表
```http
GET /notifications?page=1&size=20&isRead=false&type=PAGE_UPDATED
Authorization: Bearer {accessToken}
```

#### 6.1.2 标记通知为已读
```http
PUT /notifications/{notificationId}/read
Authorization: Bearer {accessToken}
```

#### 6.1.3 批量标记已读
```http
PUT /notifications/mark-all-read
Authorization: Bearer {accessToken}
```

#### 6.1.4 删除通知
```http
DELETE /notifications/{notificationId}
Authorization: Bearer {accessToken}
```

### 6.2 订阅管理

#### 6.2.1 获取订阅列表
```http
GET /subscriptions?resourceType=PAGE&page=1&size=20
Authorization: Bearer {accessToken}
```

#### 6.2.2 订阅资源
```http
POST /subscriptions
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "resourceType": "PAGE",
  "resourceId": 123,
  "subscriptionType": "WATCH"
}
```

#### 6.2.3 取消订阅
```http
DELETE /subscriptions/{subscriptionId}
Authorization: Bearer {accessToken}
```

### 6.3 通知设置

#### 6.3.1 获取通知偏好
```http
GET /notifications/preferences
Authorization: Bearer {accessToken}
```

#### 6.3.2 更新通知偏好
```http
PUT /notifications/preferences
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "emailNotifications": {
    "pageUpdated": true,
    "commentAdded": true,
    "mentioned": true
  },
  "pushNotifications": {
    "pageUpdated": false,
    "commentAdded": true,
    "mentioned": true
  }
}
```

## 7. 文件服务 (File Service)

### 7.1 文件上传

#### 7.1.1 上传文件
```http
POST /files/upload
Authorization: Bearer {accessToken}
Content-Type: multipart/form-data

file: (binary)
pageId: 123
spaceId: 1
description: 设计图
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "filename": "design_v2.pdf",
    "originalFilename": "设计图_v2.pdf",
    "fileSize": 2048576,
    "mimeType": "application/pdf",
    "downloadUrl": "/files/1/download",
    "previewUrl": "/files/1/preview",
    "thumbnailUrl": "/files/1/thumbnail",
    "uploadedAt": "2024-01-15T10:30:00Z"
  }
}
```

#### 7.1.2 批量上传
```http
POST /files/upload/batch
Authorization: Bearer {accessToken}
Content-Type: multipart/form-data

files[]: (binary)
files[]: (binary)
pageId: 123
spaceId: 1
```

### 7.2 文件管理

#### 7.2.1 获取文件列表
```http
GET /files?pageId=123&page=1&size=20&type=image
Authorization: Bearer {accessToken}
```

#### 7.2.2 获取文件详情
```http
GET /files/{fileId}
Authorization: Bearer {accessToken}
```

#### 7.2.3 下载文件
```http
GET /files/{fileId}/download
Authorization: Bearer {accessToken}
```

#### 7.2.4 预览文件
```http
GET /files/{fileId}/preview?width=800&height=600
Authorization: Bearer {accessToken}
```

#### 7.2.5 删除文件
```http
DELETE /files/{fileId}
Authorization: Bearer {accessToken}
```

### 7.3 图片处理

#### 7.3.1 生成缩略图
```http
GET /files/{fileId}/thumbnail?size=200x200
Authorization: Bearer {accessToken}
```

#### 7.3.2 图片裁剪
```http
GET /files/{fileId}/crop?x=10&y=10&width=300&height=200
Authorization: Bearer {accessToken}
```

## 8. 评论服务 (Comment Service)

### 8.1 评论管理

#### 8.1.1 获取页面评论
```http
GET /pages/{pageId}/comments?page=1&size=20&includeReplies=true
Authorization: Bearer {accessToken}
```

#### 8.1.2 添加评论
```http
POST /pages/{pageId}/comments
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "content": "这是一个很好的文档，建议添加更多示例",
  "parentId": null,
  "positionInfo": {
    "lineNumber": 25,
    "selection": "API设计原则"
  }
}
```

#### 8.1.3 回复评论
```http
POST /comments/{commentId}/replies
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "content": "谢谢反馈，我会在下个版本中添加更多示例"
}
```

#### 8.1.4 更新评论
```http
PUT /comments/{commentId}
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "content": "更新后的评论内容"
}
```

#### 8.1.5 删除评论
```http
DELETE /comments/{commentId}
Authorization: Bearer {accessToken}
```

#### 8.1.6 解决评论
```http
PUT /comments/{commentId}/resolve
Authorization: Bearer {accessToken}
```

## 9. WebSocket 实时API

### 9.1 连接管理

#### 9.1.1 建立WebSocket连接
```javascript
const ws = new WebSocket('wss://api.kms-wind.com/ws?token={accessToken}');
```

#### 9.1.2 心跳保持
```json
{
  "type": "PING",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### 9.2 实时事件

#### 9.2.1 页面协作编辑
```json
{
  "type": "PAGE_EDIT",
  "pageId": 123,
  "userId": 456,
  "data": {
    "operation": "INSERT",
    "position": 150,
    "content": "新增内容",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

#### 9.2.2 实时通知
```json
{
  "type": "NOTIFICATION",
  "data": {
    "id": 789,
    "title": "页面被更新",
    "message": "John Doe 更新了页面 'API设计指南'",
    "resourceType": "PAGE",
    "resourceId": 123,
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

#### 9.2.3 在线状态
```json
{
  "type": "USER_PRESENCE",
  "data": {
    "userId": 456,
    "status": "ONLINE", // ONLINE, AWAY, OFFLINE
    "pageId": 123,
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

## 10. API 安全设计

### 10.1 认证机制

#### 10.1.1 JWT Token 格式
```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user123",
    "exp": 1640995200,
    "iat": 1640908800,
    "iss": "kms-wind",
    "aud": "kms-wind-api",
    "tenantId": "tenant001",
    "roles": ["USER", "SPACE_ADMIN"],
    "permissions": ["READ", "WRITE"]
  }
}
```

#### 10.1.2 请求头认证
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-Tenant-ID: tenant001
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
```

### 10.2 权限控制

#### 10.2.1 权限级别
- **SYSTEM**: 系统级权限
- **TENANT**: 租户级权限
- **SPACE**: 空间级权限
- **PAGE**: 页面级权限

#### 10.2.2 权限检查中间件
```java
@PreAuthorize("hasPermission(#spaceId, 'SPACE', 'EDIT')")
public ResponseEntity<Space> updateSpace(@PathVariable Long spaceId, @RequestBody SpaceUpdateRequest request) {
    // 业务逻辑
}
```

### 10.3 API 限流

#### 10.3.1 限流规则
- **认证接口**: 10次/分钟
- **查询接口**: 100次/分钟
- **上传接口**: 5次/分钟
- **批量操作**: 10次/分钟

#### 10.3.2 限流响应
```json
{
  "success": false,
  "code": 429,
  "message": "请求过于频繁，请稍后重试",
  "retryAfter": 60
}
```

## 11. API 测试示例

### 11.1 Postman Collection
```json
{
  "info": {
    "name": "KMS-Wind API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "https://api.kms-wind.com/api/v1"
    },
    {
      "key": "accessToken",
      "value": ""
    }
  ]
}
```

### 11.2 cURL 示例
```bash
# 用户登录
curl -X POST "${baseUrl}/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "user@example.com",
    "password": "password123"
  }'

# 创建页面
curl -X POST "${baseUrl}/spaces/1/pages" \
  -H "Authorization: Bearer ${accessToken}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "新建页面",
    "content": "# 页面标题\n\n页面内容..."
  }'

# 搜索内容
curl -X GET "${baseUrl}/search?q=API%20design&type=PAGE" \
  -H "Authorization: Bearer ${accessToken}"
```

## 12. API 性能优化

### 12.1 缓存策略
- **Redis 缓存**: 热点数据缓存
- **CDN 缓存**: 静态资源缓存
- **浏览器缓存**: 客户端缓存
- **应用缓存**: 业务数据缓存

### 12.2 分页和过滤
- **默认分页**: 每页20条记录
- **最大分页**: 每页100条记录
- **字段过滤**: 只返回需要的字段
- **关系加载**: 按需加载关联数据

### 12.3 批量操作
- **批量查询**: 减少网络往返
- **批量更新**: 提高操作效率
- **异步处理**: 耗时操作异步化
- **状态轮询**: 长时间任务状态查询

## 13. API 监控和日志

### 13.1 监控指标
- **响应时间**: P50, P95, P99响应时间
- **请求量**: QPS和并发数
- **错误率**: 4xx和5xx错误率
- **可用性**: 服务可用性SLA

### 13.2 日志格式
```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "traceId": "550e8400-e29b-41d4-a716-446655440000",
  "service": "document-service",
  "method": "GET",
  "uri": "/api/v1/pages/123",
  "userId": "user123",
  "tenantId": "tenant001",
  "responseTime": 125,
  "statusCode": 200,
  "userAgent": "Mozilla/5.0...",
  "ip": "192.168.1.100"
}
```

## 14. 详细错误码定义

### 14.1 错误码规范

#### 14.1.1 错误码格式
```
错误码格式: {服务代码}{模块代码}{错误类型}{序号}
- 服务代码: 2位，01=用户服务, 02=文档服务, 03=空间服务等
- 模块代码: 2位，01=认证, 02=权限, 03=CRUD等
- 错误类型: 1位，1=参数错误, 2=权限错误, 3=业务错误, 4=系统错误
- 序号: 3位，具体错误的序列号

示例: 01011001 = 用户服务认证模块参数错误001
```

#### 14.1.2 通用错误码
```json
{
  "commonErrors": {
    "00001000": {
      "code": "INVALID_REQUEST",
      "message": "请求格式错误",
      "httpStatus": 400,
      "category": "CLIENT_ERROR"
    },
    "00001001": {
      "code": "MISSING_PARAMETER",
      "message": "缺少必需参数: {parameter}",
      "httpStatus": 400,
      "category": "CLIENT_ERROR"
    },
    "00001002": {
      "code": "INVALID_PARAMETER",
      "message": "参数格式错误: {parameter}",
      "httpStatus": 400,
      "category": "CLIENT_ERROR"
    },
    "00002000": {
      "code": "UNAUTHORIZED",
      "message": "未授权访问",
      "httpStatus": 401,
      "category": "AUTH_ERROR"
    },
    "00002001": {
      "code": "TOKEN_EXPIRED",
      "message": "访问令牌已过期",
      "httpStatus": 401,
      "category": "AUTH_ERROR"
    },
    "00002002": {
      "code": "TOKEN_INVALID",
      "message": "访问令牌无效",
      "httpStatus": 401,
      "category": "AUTH_ERROR"
    },
    "00003000": {
      "code": "FORBIDDEN",
      "message": "无权限访问此资源",
      "httpStatus": 403,
      "category": "PERMISSION_ERROR"
    },
    "00004000": {
      "code": "RESOURCE_NOT_FOUND",
      "message": "请求的资源不存在",
      "httpStatus": 404,
      "category": "CLIENT_ERROR"
    },
    "00005000": {
      "code": "RESOURCE_CONFLICT",
      "message": "资源冲突",
      "httpStatus": 409,
      "category": "CLIENT_ERROR"
    },
    "00006000": {
      "code": "RATE_LIMIT_EXCEEDED",
      "message": "请求频率超限，请稍后重试",
      "httpStatus": 429,
      "category": "CLIENT_ERROR"
    },
    "00009000": {
      "code": "INTERNAL_SERVER_ERROR",
      "message": "服务器内部错误",
      "httpStatus": 500,
      "category": "SERVER_ERROR"
    },
    "00009001": {
      "code": "SERVICE_UNAVAILABLE",
      "message": "服务暂时不可用",
      "httpStatus": 503,
      "category": "SERVER_ERROR"
    }
  }
}
```

### 14.2 业务模块错误码

#### 14.2.1 用户认证模块 (01xx)
```json
{
  "authErrors": {
    "01011001": {
      "code": "INVALID_USERNAME_FORMAT",
      "message": "用户名格式错误，只允许字母、数字、下划线，长度3-50字符",
      "httpStatus": 400
    },
    "01011002": {
      "code": "INVALID_EMAIL_FORMAT",
      "message": "邮箱格式错误",
      "httpStatus": 400
    },
    "01011003": {
      "code": "WEAK_PASSWORD",
      "message": "密码强度不足，至少8位包含大小写字母和数字",
      "httpStatus": 400
    },
    "01013001": {
      "code": "USER_NOT_FOUND",
      "message": "用户不存在",
      "httpStatus": 404
    },
    "01013002": {
      "code": "INVALID_CREDENTIALS",
      "message": "用户名或密码错误",
      "httpStatus": 401
    },
    "01013003": {
      "code": "USER_ACCOUNT_LOCKED",
      "message": "账户已被锁定，请联系管理员",
      "httpStatus": 403
    },
    "01013004": {
      "code": "USER_ACCOUNT_SUSPENDED",
      "message": "账户已被暂停",
      "httpStatus": 403
    },
    "01015001": {
      "code": "USERNAME_ALREADY_EXISTS",
      "message": "用户名已存在",
      "httpStatus": 409
    },
    "01015002": {
      "code": "EMAIL_ALREADY_EXISTS",
      "message": "邮箱已被注册",
      "httpStatus": 409
    }
  }
}
```

#### 14.2.2 文档管理模块 (02xx)
```json
{
  "documentErrors": {
    "02011001": {
      "code": "INVALID_PAGE_TITLE",
      "message": "页面标题不能为空且长度不超过200字符",
      "httpStatus": 400
    },
    "02011002": {
      "code": "INVALID_PAGE_CONTENT",
      "message": "页面内容格式错误",
      "httpStatus": 400
    },
    "02013001": {
      "code": "PAGE_NOT_FOUND",
      "message": "页面不存在",
      "httpStatus": 404
    },
    "02013002": {
      "code": "PAGE_VERSION_NOT_FOUND",
      "message": "页面版本不存在",
      "httpStatus": 404
    },
    "02013003": {
      "code": "TEMPLATE_NOT_FOUND",
      "message": "模板不存在",
      "httpStatus": 404
    },
    "02023001": {
      "code": "PAGE_EDIT_PERMISSION_DENIED",
      "message": "无权限编辑此页面",
      "httpStatus": 403
    },
    "02023002": {
      "code": "PAGE_DELETE_PERMISSION_DENIED",
      "message": "无权限删除此页面",
      "httpStatus": 403
    },
    "02033001": {
      "code": "PAGE_EDIT_CONFLICT",
      "message": "页面正在被其他用户编辑",
      "httpStatus": 409
    },
    "02033002": {
      "code": "PAGE_CIRCULAR_REFERENCE",
      "message": "页面层级不能形成循环引用",
      "httpStatus": 409
    },
    "02043001": {
      "code": "PAGE_CONTENT_TOO_LARGE",
      "message": "页面内容超过大小限制",
      "httpStatus": 413
    }
  }
}
```

#### 14.2.3 空间管理模块 (03xx)
```json
{
  "spaceErrors": {
    "03011001": {
      "code": "INVALID_SPACE_KEY",
      "message": "空间标识格式错误，只允许大写字母、数字、下划线，长度2-20字符",
      "httpStatus": 400
    },
    "03011002": {
      "code": "INVALID_SPACE_NAME",
      "message": "空间名称长度必须在2-100字符之间",
      "httpStatus": 400
    },
    "03013001": {
      "code": "SPACE_NOT_FOUND",
      "message": "空间不存在",
      "httpStatus": 404
    },
    "03023001": {
      "code": "SPACE_ACCESS_DENIED",
      "message": "无权限访问此空间",
      "httpStatus": 403
    },
    "03023002": {
      "code": "SPACE_ADMIN_PERMISSION_REQUIRED",
      "message": "需要空间管理员权限",
      "httpStatus": 403
    },
    "03035001": {
      "code": "SPACE_KEY_ALREADY_EXISTS",
      "message": "空间标识已存在",
      "httpStatus": 409
    },
    "03035002": {
      "code": "SPACE_HAS_ACTIVE_CONTENT",
      "message": "空间包含内容，无法删除",
      "httpStatus": 409
    }
  }
}
```

#### 14.2.4 文件管理模块 (04xx)
```json
{
  "fileErrors": {
    "04011001": {
      "code": "INVALID_FILE_TYPE",
      "message": "不支持的文件类型",
      "httpStatus": 400
    },
    "04011002": {
      "code": "FILE_SIZE_EXCEEDED",
      "message": "文件大小超过限制(最大50MB)",
      "httpStatus": 400
    },
    "04011003": {
      "code": "INVALID_FILE_NAME",
      "message": "文件名包含非法字符",
      "httpStatus": 400
    },
    "04013001": {
      "code": "FILE_NOT_FOUND",
      "message": "文件不存在",
      "httpStatus": 404
    },
    "04043001": {
      "code": "FILE_UPLOAD_FAILED",
      "message": "文件上传失败",
      "httpStatus": 500
    },
    "04043002": {
      "code": "FILE_VIRUS_DETECTED",
      "message": "检测到恶意文件，上传被拒绝",
      "httpStatus": 400
    }
  }
}
```

### 14.3 错误响应格式

#### 14.3.1 标准错误响应
```json
{
  "success": false,
  "code": "01013002",
  "message": "用户名或密码错误",
  "timestamp": "2024-01-15T10:30:00Z",
  "traceId": "550e8400-e29b-41d4-a716-446655440000",
  "path": "/api/v1/auth/login",
  "errors": [
    {
      "field": "password",
      "code": "INVALID_CREDENTIALS",
      "message": "密码错误"
    }
  ],
  "suggestions": [
    "请检查用户名和密码是否正确",
    "如果忘记密码，请使用找回密码功能"
  ]
}
```

#### 14.3.2 字段验证错误
```json
{
  "success": false,
  "code": "00001002",
  "message": "请求参数验证失败",
  "timestamp": "2024-01-15T10:30:00Z",
  "traceId": "550e8400-e29b-41d4-a716-446655440000",
  "path": "/api/v1/spaces",
  "errors": [
    {
      "field": "spaceKey",
      "code": "INVALID_SPACE_KEY",
      "message": "空间标识格式错误，只允许大写字母、数字、下划线，长度2-20字符",
      "rejectedValue": "invalid-key"
    },
    {
      "field": "name",
      "code": "MISSING_PARAMETER",
      "message": "空间名称不能为空"
    }
  ]
}
```

## 15. API版本管理策略

### 15.1 版本控制方案

#### 15.1.1 版本命名规范
```
版本格式: v{major}.{minor}.{patch}
- major: 主版本号，不兼容的API修改
- minor: 次版本号，向后兼容的功能性新增
- patch: 修订号，向后兼容的问题修正

示例: v1.2.3
```

#### 15.1.2 版本传递方式
```http
# 方式1: URL路径版本（推荐）
GET /api/v1/pages/123
GET /api/v2/pages/123

# 方式2: 请求头版本
GET /api/pages/123
Accept-Version: v1

# 方式3: 查询参数版本
GET /api/pages/123?version=v1
```

### 15.2 版本兼容性策略

#### 15.2.1 向后兼容原则
```yaml
# 兼容性变更（无需升级主版本）
compatible_changes:
  - 添加新的API端点
  - 添加可选的请求参数
  - 添加响应字段
  - 放宽验证规则
  - 修复bug

# 不兼容变更（需要升级主版本）
breaking_changes:
  - 删除API端点
  - 删除请求参数或响应字段
  - 修改字段类型
  - 修改字段语义
  - 严格验证规则
```

#### 15.2.2 API废弃策略
```java
// API废弃注解示例
@Deprecated
@ApiOperation(value = "获取页面详情", notes = "此接口将在v2.0中移除，请使用 GET /pages/{id}/detail")
@GetMapping("/pages/{id}")
public ResponseEntity<Page> getPage(@PathVariable Long id) {
    // 实现
}

// 响应头标记废弃
response.setHeader("Deprecation", "true");
response.setHeader("Sunset", "2024-12-31T23:59:59Z");
response.setHeader("Link", "</api/v2/pages/{id}/detail>; rel=\"successor-version\"");
```

### 15.3 版本迁移指南

#### 15.3.1 v1 to v2 迁移
```json
{
  "migration_guide": {
    "version": "v1 -> v2",
    "release_date": "2024-06-01",
    "deprecation_date": "2024-12-31",
    "changes": [
      {
        "type": "REMOVED",
        "endpoint": "GET /pages/{id}",
        "replacement": "GET /pages/{id}/detail",
        "reason": "统一详情接口命名规范"
      },
      {
        "type": "MODIFIED",
        "endpoint": "POST /spaces",
        "changes": [
          {
            "field": "type",
            "old_type": "string",
            "new_type": "enum",
            "values": ["TEAM", "PERSONAL", "PROJECT"],
            "migration": "原有字符串值需要转换为枚举值"
          }
        ]
      },
      {
        "type": "ADDED",
        "endpoint": "GET /spaces/{id}/analytics",
        "description": "新增空间分析数据接口"
      }
    ]
  }
}
```

#### 15.3.2 自动迁移工具
```bash
# API迁移检查工具
./kms-wind-cli api-migration check --from=v1 --to=v2 --config=migration.json

# 生成迁移报告
./kms-wind-cli api-migration report --output=migration-report.html

# 批量更新API调用
./kms-wind-cli api-migration update --project=/path/to/project --target=v2
```

## 16. OpenAPI文档生成

### 16.1 Swagger配置

#### 16.1.1 Spring Doc配置
```java
@Configuration
@EnableWebMvc
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("KMS-Wind API")
                .version("v1.0.0")
                .description("企业级知识管理系统API文档")
                .contact(new Contact()
                    .name("KMS-Wind Team")
                    .email("support@kms-wind.com")
                    .url("https://kms-wind.com"))
                .license(new License()
                    .name("MIT License")
                    .url("https://opensource.org/licenses/MIT")))
            .servers(Arrays.asList(
                new Server().url("https://api.kms-wind.com").description("生产环境"),
                new Server().url("https://test-api.kms-wind.com").description("测试环境"),
                new Server().url("http://localhost:8080").description("开发环境")
            ))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }

    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public")
            .pathsToMatch("/api/v1/**")
            .build();
    }
}
```

#### 16.1.2 API文档注解示例
```java
@RestController
@RequestMapping("/api/v1/pages")
@Tag(name = "页面管理", description = "页面的增删改查操作")
public class PageController {

    @Operation(
        summary = "创建页面",
        description = "在指定空间中创建新页面",
        tags = {"页面管理"}
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "201",
            description = "页面创建成功",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = PageResponse.class)
            )
        ),
        @ApiResponse(
            responseCode = "400",
            description = "请求参数错误",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = ErrorResponse.class)
            )
        ),
        @ApiResponse(
            responseCode = "403",
            description = "无权限在此空间创建页面"
        )
    })
    @PostMapping
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<PageResponse> createPage(
        @Parameter(description = "页面创建请求", required = true)
        @Valid @RequestBody PageCreateRequest request,

        @Parameter(description = "空间ID", required = true, example = "1")
        @RequestParam Long spaceId
    ) {
        // 实现
    }
}
```

### 16.2 文档自动化生成

#### 16.2.1 CI/CD集成
```yaml
# .github/workflows/api-docs.yml
name: Generate API Documentation

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  generate-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build application
        run: ./mvnw clean compile

      - name: Generate OpenAPI spec
        run: ./mvnw spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=docs" &

      - name: Wait for application startup
        run: sleep 30

      - name: Download OpenAPI spec
        run: |
          curl http://localhost:8080/v3/api-docs > openapi.json
          curl http://localhost:8080/v3/api-docs.yaml > openapi.yaml

      - name: Generate HTML documentation
        uses: Legion2/swagger-ui-action@v1
        with:
          output: docs
          spec-file: openapi.yaml

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: docs
```

#### 16.2.2 多格式文档导出
```bash
# 导出Postman Collection
curl -s http://localhost:8080/v3/api-docs | \
  npx openapi-to-postman -s - -o kms-wind-api.postman.json

# 导出Insomnia Collection
curl -s http://localhost:8080/v3/api-docs | \
  npx openapi-to-insomnia -s - -o kms-wind-api.insomnia.json

# 生成客户端SDK
openapi-generator generate \
  -i http://localhost:8080/v3/api-docs \
  -g typescript-axios \
  -o ./sdk/typescript

# 生成API测试代码
openapi-generator generate \
  -i http://localhost:8080/v3/api-docs \
  -g java-junit \
  -o ./test/generated
```

### 16.3 文档质量保证

#### 16.3.1 文档验证规则
```yaml
# api-docs-rules.yaml
rules:
  required_fields:
    - summary
    - description
    - tags
    - responses

  response_codes:
    required: [200, 400, 401, 403, 500]

  parameter_rules:
    - name: required
    - description: required
    - example: recommended

  schema_rules:
    - all_fields_documented: true
    - examples_provided: true
```

#### 16.3.2 文档测试
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class OpenApiDocumentationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void shouldGenerateValidOpenApiSpec() {
        ResponseEntity<String> response = restTemplate.getForEntity("/v3/api-docs", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

        // 验证OpenAPI规范格式
        ObjectMapper mapper = new ObjectMapper();
        JsonNode spec = mapper.readTree(response.getBody());

        assertThat(spec.get("openapi").asText()).isEqualTo("3.0.1");
        assertThat(spec.get("info").get("title").asText()).isEqualTo("KMS-Wind API");
        assertThat(spec.get("paths")).isNotNull();
    }

    @Test
    public void shouldHaveAllEndpointsDocumented() {
        // 验证所有端点都有完整的文档
        ResponseEntity<String> response = restTemplate.getForEntity("/v3/api-docs", String.class);
        JsonNode spec = mapper.readTree(response.getBody());

        JsonNode paths = spec.get("paths");
        paths.fields().forEachRemaining(entry -> {
            String path = entry.getKey();
            JsonNode methods = entry.getValue();

            methods.fields().forEachRemaining(methodEntry -> {
                String method = methodEntry.getKey();
                JsonNode operation = methodEntry.getValue();

                // 验证必需字段
                assertThat(operation.get("summary")).isNotNull();
                assertThat(operation.get("description")).isNotNull();
                assertThat(operation.get("tags")).isNotNull();
                assertThat(operation.get("responses")).isNotNull();
            });
        });
    }
}