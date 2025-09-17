# KMS-Wind 系统架构设计文档

## 1. 系统概述

KMS-Wind 是一个企业级知识管理系统（Knowledge Management System），参考 Atlassian Confluence 的核心功能和架构设计，提供协作文档编辑、知识库管理、企业级权限控制等功能。

### 1.1 技术栈选择
- **后端框架**: Spring Boot 3.x (基于 JDK 17)
- **前端框架**: React 18+ (专注于后端架构)
- **数据库**: PostgreSQL 15+
- **缓存**: Redis 7+
- **搜索引擎**: Elasticsearch 8+
- **消息队列**: RabbitMQ
- **文件存储**: MinIO / AWS S3

### 1.2 核心业务域
- **文档管理域**: 页面创建、编辑、版本控制
- **空间管理域**: 工作空间、权限管理、协作
- **用户管理域**: 认证、授权、用户配置
- **搜索域**: 全文搜索、智能推荐
- **通知域**: 实时通知、订阅管理

## 2. 系统架构设计

### 2.1 整体架构图

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Browser   │    │   Mobile App    │    │   Third Party   │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                │
                     ┌─────────┴─────────┐
                     │  API Gateway      │
                     │  (Spring Gateway) │
                     └─────────┬─────────┘
                               │
                   ┌───────────┼───────────┐
                   │           │           │
          ┌────────▼───────┐ ┌─▼─────┐ ┌──▼──────┐
          │   Auth Service │ │ Document│ │ Search  │
          │   (OAuth2/JWT) │ │ Service │ │ Service │
          └────────────────┘ └─────────┘ └─────────┘
                   │           │           │
                   └───────────┼───────────┘
                               │
                     ┌─────────▼─────────┐
                     │ Persistent Layer  │
                     │ PostgreSQL+Redis  │
                     └───────────────────┘
```

### 2.2 微服务架构设计

#### 2.2.1 核心服务列表

1. **Gateway Service** (API 网关服务)
   - 请求路由与负载均衡
   - 认证与授权统一处理
   - 限流与熔断保护
   - 跨域处理

2. **Auth Service** (认证授权服务)
   - JWT Token 管理
   - OAuth2.0 集成
   - 用户身份验证
   - 权限控制

3. **Document Service** (文档管理服务)
   - 页面 CRUD 操作
   - 版本控制管理
   - 文档模板管理
   - 附件文件管理

4. **Space Service** (空间管理服务)
   - 工作空间管理
   - 权限分配管理
   - 成员邀请管理
   - 空间配置管理

5. **Search Service** (搜索服务)
   - Elasticsearch 集成
   - 全文检索功能
   - 智能搜索建议
   - 搜索结果排序

6. **Notification Service** (通知服务)
   - 实时消息推送
   - 邮件通知
   - 订阅管理
   - 消息队列处理

7. **File Service** (文件存储服务)
   - 文件上传下载
   - 文件版本管理
   - 图片处理服务
   - CDN 集成

### 2.3 分层架构设计

```
┌─────────────────────────────────────────┐
│              Presentation Layer         │
│         (REST Controllers)              │
├─────────────────────────────────────────┤
│              Business Layer             │
│            (Service Layer)              │
├─────────────────────────────────────────┤
│             Persistence Layer           │
│    (Repository + JPA + Spring Data)     │
├─────────────────────────────────────────┤
│              Database Layer             │
│       (PostgreSQL + Redis + ES)        │
└─────────────────────────────────────────┘
```

#### 2.3.1 Presentation Layer (表现层)
- **REST Controllers**: 处理HTTP请求响应
- **DTO Classes**: 数据传输对象
- **Request/Response Models**: 请求响应模型
- **Validation**: 参数校验
- **Exception Handlers**: 统一异常处理

#### 2.3.2 Business Layer (业务层)
- **Service Interfaces**: 业务接口定义
- **Service Implementations**: 业务逻辑实现
- **Business Objects**: 业务对象
- **Event Publishers**: 事件发布
- **Transaction Management**: 事务管理

#### 2.3.3 Persistence Layer (持久层)
- **Repository Interfaces**: 数据访问接口
- **JPA Entities**: JPA实体类
- **Custom Queries**: 自定义查询
- **Database Migrations**: 数据库版本管理

## 3. 架构设计原则

### 3.1 设计原则
- **单一职责原则**: 每个服务只负责一个业务域
- **开闭原则**: 对扩展开放，对修改封闭
- **接口隔离原则**: 依赖接口而不是具体实现
- **依赖倒置原则**: 高层模块不依赖低层模块

### 3.2 架构模式
- **领域驱动设计 (DDD)**: 按业务域划分服务边界
- **CQRS 模式**: 读写分离，提高系统性能
- **Event Sourcing**: 事件溯源，保证数据一致性
- **Saga 模式**: 分布式事务管理

### 3.3 性能设计
- **缓存策略**: 多级缓存设计，Redis + 本地缓存
- **数据库优化**: 读写分离、分库分表
- **异步处理**: 消息队列处理耗时操作
- **CDN 加速**: 静态资源全球分发

## 4. 安全架构设计

### 4.1 认证授权
- **JWT Token**: 无状态认证
- **OAuth2.0**: 第三方登录集成
- **RBAC**: 基于角色的权限控制
- **Multi-Factor Authentication**: 多因子认证

### 4.2 数据安全
- **数据加密**: 敏感数据加密存储
- **传输安全**: HTTPS + TLS1.3
- **SQL 注入防护**: 参数化查询
- **XSS 防护**: 输入输出过滤

### 4.3 审计日志
- **操作审计**: 记录用户关键操作
- **访问审计**: 记录API访问日志
- **安全事件**: 异常登录告警
- **合规性**: 满足企业合规要求

## 5. 部署架构设计

### 5.1 容器化部署
- **Docker**: 应用容器化
- **Kubernetes**: 容器编排
- **Helm**: 应用包管理
- **Istio**: 服务网格

### 5.2 云原生架构
- **微服务**: 独立部署扩容
- **服务发现**: Eureka/Consul
- **配置中心**: Spring Cloud Config
- **API网关**: Spring Cloud Gateway

### 5.3 监控运维
- **应用监控**: Prometheus + Grafana
- **日志管理**: ELK Stack
- **链路追踪**: Sleuth + Zipkin
- **健康检查**: Spring Actuator

## 6. 扩展性设计

### 6.1 水平扩展
- **无状态设计**: 服务实例可任意扩缩
- **负载均衡**: 请求均匀分配
- **数据分片**: 支持数据水平分割
- **缓存分布式**: Redis Cluster

### 6.2 插件化设计
- **模板引擎**: 支持自定义页面模板
- **插件系统**: 支持第三方功能扩展
- **Webhook**: 支持外部系统集成
- **Open API**: 开放API供第三方调用

## 7. 配置管理和治理

### 7.1 配置中心设计

#### 7.1.1 配置管理架构
```
┌─────────────────┐    ┌─────────────────┐
│   Config Admin  │    │   Config Client │
│     Portal      │    │   (Services)    │
└─────────┬───────┘    └─────────┬───────┘
          │                      │
          └──────────┬───────────┘
                     │
            ┌────────▼────────┐
            │  Config Center  │
            │ (Spring Cloud   │
            │    Config)      │
            └─────────────────┘
                     │
            ┌────────▼────────┐
            │   Git Repository│
            │ (Configuration  │
            │     Storage)    │
            └─────────────────┘
```

#### 7.1.2 配置分类管理
- **应用配置**: 业务参数、功能开关
- **环境配置**: 数据库连接、外部服务地址
- **基础配置**: 日志级别、监控参数
- **安全配置**: 密钥、证书（加密存储）

#### 7.1.3 配置热更新机制
```yaml
# 应用配置示例
application:
  features:
    realtime-edit: true
    ai-recommendation: false
  limits:
    max-file-size: 50MB
    max-page-size: 100
  cache:
    user-info-ttl: 1800
    page-content-ttl: 3600
```

### 7.2 分布式锁设计

#### 7.2.1 锁的应用场景
- **文档编辑锁**: 防止并发编辑冲突
- **空间配置锁**: 避免权限配置竞争
- **批量操作锁**: 防止重复执行
- **缓存更新锁**: 避免缓存击穿

#### 7.2.2 Redis分布式锁实现
```java
@Component
public class DistributedLockService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public boolean tryLock(String key, String value, long expireTime) {
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, value, expireTime, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(result);
    }

    public void unlock(String key, String value) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                       "return redis.call('del', KEYS[1]) else return 0 end";
        redisTemplate.execute(new DefaultRedisScript<>(script, Long.class),
                            Collections.singletonList(key), value);
    }
}
```

### 7.3 事件驱动架构

#### 7.3.1 事件总线设计
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Producer   │    │  Event Bus  │    │  Consumer   │
│  Service    │───▶│ (RabbitMQ)  │───▶│  Service    │
└─────────────┘    └─────────────┘    └─────────────┘
                         │
                    ┌────▼────┐
                    │ Message │
                    │ Store   │
                    └─────────┘
```

#### 7.3.2 事件类型定义
```java
public abstract class DomainEvent {
    private String eventId;
    private String eventType;
    private LocalDateTime occurredOn;
    private String aggregateId;
    private Integer version;
}

@Data
public class PageCreatedEvent extends DomainEvent {
    private Long pageId;
    private Long spaceId;
    private Long authorId;
    private String title;
}

@Data
public class UserPermissionChangedEvent extends DomainEvent {
    private Long userId;
    private Long resourceId;
    private String resourceType;
    private List<String> permissions;
}
```

### 7.4 灰度发布策略

#### 7.4.1 发布流程设计
```
开发环境 → 测试环境 → 预发布环境 → 灰度发布 → 全量发布
    ↓         ↓          ↓          ↓         ↓
  单元测试   集成测试    验收测试    小流量验证  全量监控
```

#### 7.4.2 灰度规则配置
```yaml
# 灰度发布配置
canary:
  enabled: true
  rules:
    - type: USER_ID
      condition: "mod 100 < 5"  # 5%用户
    - type: SPACE_ID
      condition: "in (1,2,3)"   # 特定空间
    - type: FEATURE_FLAG
      condition: "beta_tester = true"  # Beta用户
  monitoring:
    error-threshold: 0.1%
    response-time-threshold: 2s
    rollback-enabled: true
```

### 7.5 容器化和编排

#### 7.5.1 Docker容器设计
```dockerfile
# 应用容器示例
FROM openjdk:17-jre-slim

# 添加应用监控代理
ADD agent/javaagent.jar /app/agent.jar

# 应用JAR包
ADD target/kms-wind-service.jar /app/app.jar

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-javaagent:/app/agent.jar", "-jar", "/app/app.jar"]
```

#### 7.5.2 Kubernetes部署配置
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: document-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: document-service
  template:
    metadata:
      labels:
        app: document-service
    spec:
      containers:
      - name: document-service
        image: kms-wind/document-service:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### 7.6 服务网格架构

#### 7.6.1 Istio服务网格
```yaml
# 服务网格配置
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: document-service
spec:
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: document-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: document-service
        subset: v1
      weight: 100
```

#### 7.6.2 流量管理策略
- **负载均衡**: 轮询、权重、一致性哈希
- **熔断器**: 自动故障切换
- **重试机制**: 指数退避重试
- **超时控制**: 请求超时保护

## 8. 非功能性需求

### 8.1 性能指标
- **响应时间**: 页面加载 < 2秒
- **并发用户**: 支持1万在线用户
- **数据容量**: 支持TB级文档存储
- **可用性**: 99.9% 系统可用性

### 8.2 可扩展性
- **用户规模**: 支持10万+企业用户
- **存储扩展**: 支持PB级数据存储
- **功能扩展**: 插件化功能扩展
- **地域扩展**: 多地域部署支持

### 8.3 可维护性
- **代码质量**: SonarQube代码扫描
- **单元测试**: 覆盖率 > 80%
- **文档完善**: API文档自动生成
- **日志规范**: 统一日志格式

### 8.4 运维监控
- **应用监控**: APM性能监控
- **基础监控**: CPU、内存、磁盘、网络
- **业务监控**: 关键业务指标
- **日志监控**: 错误日志告警