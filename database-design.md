# KMS-Wind 数据库设计文档

## 1. 数据库设计概述

### 1.1 数据库选型
- **主数据库**: PostgreSQL 15+
- **缓存数据库**: Redis 7+
- **搜索引擎**: Elasticsearch 8+
- **时序数据库**: InfluxDB (用于监控和审计)

### 1.2 设计原则
- **规范化设计**: 减少数据冗余，保证数据一致性
- **性能优化**: 合理使用索引，支持高并发查询
- **扩展性**: 支持水平分片和垂直拆分
- **安全性**: 敏感数据加密，访问权限控制

### 1.3 数据分布策略
- **用户域**: 按租户ID分片
- **文档域**: 按空间ID分片
- **日志域**: 按时间分片
- **搜索域**: Elasticsearch独立集群

## 2. 核心数据模型设计

### 2.1 用户管理域

#### 2.1.1 users (用户表)
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100),
    avatar_url VARCHAR(500),
    phone VARCHAR(20),
    status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, INACTIVE, SUSPENDED
    email_verified BOOLEAN DEFAULT FALSE,
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT REFERENCES users(id),
    tenant_id BIGINT NOT NULL
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
```

#### 2.1.2 roles (角色表)
```sql
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    description TEXT,
    scope VARCHAR(20) NOT NULL, -- SYSTEM, SPACE, PAGE
    permissions JSONB,
    is_system BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    tenant_id BIGINT NOT NULL,
    UNIQUE(name, tenant_id)
);
```

#### 2.1.3 user_roles (用户角色关联表)
```sql
CREATE TABLE user_roles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    resource_type VARCHAR(20), -- SYSTEM, SPACE, PAGE
    resource_id BIGINT,
    granted_by BIGINT REFERENCES users(id),
    granted_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE,
    UNIQUE(user_id, role_id, resource_type, resource_id)
);
```

### 2.2 空间管理域

#### 2.2.1 spaces (空间表)
```sql
CREATE TABLE spaces (
    id BIGSERIAL PRIMARY KEY,
    space_key VARCHAR(50) NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    type VARCHAR(20) DEFAULT 'TEAM', -- TEAM, PERSONAL, PROJECT
    visibility VARCHAR(20) DEFAULT 'PRIVATE', -- PUBLIC, PRIVATE, RESTRICTED
    status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, ARCHIVED, DELETED
    logo_url VARCHAR(500),
    homepage_id BIGINT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT NOT NULL REFERENCES users(id),
    tenant_id BIGINT NOT NULL,
    UNIQUE(space_key, tenant_id)
);

CREATE INDEX idx_spaces_space_key ON spaces(space_key);
CREATE INDEX idx_spaces_created_by ON spaces(created_by);
CREATE INDEX idx_spaces_tenant_id ON spaces(tenant_id);
```

#### 2.2.2 space_permissions (空间权限表)
```sql
CREATE TABLE space_permissions (
    id BIGSERIAL PRIMARY KEY,
    space_id BIGINT NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT REFERENCES roles(id) ON DELETE CASCADE,
    permission_type VARCHAR(30) NOT NULL, -- VIEW, EDIT, ADMIN, DELETE
    granted_by BIGINT REFERENCES users(id),
    granted_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(space_id, user_id, role_id, permission_type)
);
```

### 2.3 页面/文档管理域

#### 2.3.1 pages (页面表)
```sql
CREATE TABLE pages (
    id BIGSERIAL PRIMARY KEY,
    space_id BIGINT NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    parent_id BIGINT REFERENCES pages(id),
    title VARCHAR(200) NOT NULL,
    content_id BIGINT, -- 指向当前版本的内容
    template_id BIGINT REFERENCES page_templates(id),
    status VARCHAR(20) DEFAULT 'DRAFT', -- DRAFT, PUBLISHED, ARCHIVED, DELETED
    version_number INTEGER DEFAULT 1,
    position INTEGER DEFAULT 0, -- 用于排序
    labels TEXT[], -- 标签数组
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT NOT NULL REFERENCES users(id),
    updated_by BIGINT NOT NULL REFERENCES users(id),
    slug VARCHAR(100),
    is_homepage BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_pages_space_id ON pages(space_id);
CREATE INDEX idx_pages_parent_id ON pages(parent_id);
CREATE INDEX idx_pages_created_by ON pages(created_by);
CREATE INDEX idx_pages_status ON pages(status);
CREATE INDEX idx_pages_labels ON pages USING GIN(labels);
CREATE UNIQUE INDEX idx_pages_slug_space ON pages(space_id, slug) WHERE slug IS NOT NULL;
```

#### 2.3.2 page_contents (页面内容表)
```sql
CREATE TABLE page_contents (
    id BIGSERIAL PRIMARY KEY,
    page_id BIGINT NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    content_type VARCHAR(20) DEFAULT 'MARKDOWN', -- MARKDOWN, HTML, CONFLUENCE_WIKI
    version_number INTEGER NOT NULL,
    word_count INTEGER DEFAULT 0,
    excerpt TEXT, -- 页面摘要
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT NOT NULL REFERENCES users(id),
    UNIQUE(page_id, version_number)
);

CREATE INDEX idx_page_contents_page_id ON page_contents(page_id);
CREATE INDEX idx_page_contents_version ON page_contents(page_id, version_number);
```

#### 2.3.3 page_templates (页面模板表)
```sql
CREATE TABLE page_templates (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    content TEXT NOT NULL,
    template_type VARCHAR(30) DEFAULT 'USER', -- SYSTEM, SPACE, USER
    space_id BIGINT REFERENCES spaces(id),
    is_global BOOLEAN DEFAULT FALSE,
    thumbnail_url VARCHAR(500),
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT NOT NULL REFERENCES users(id),
    tenant_id BIGINT NOT NULL
);
```

### 2.4 附件管理域

#### 2.4.1 attachments (附件表)
```sql
CREATE TABLE attachments (
    id BIGSERIAL PRIMARY KEY,
    page_id BIGINT REFERENCES pages(id) ON DELETE CASCADE,
    space_id BIGINT REFERENCES spaces(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(100),
    file_hash VARCHAR(64), -- SHA-256
    version_number INTEGER DEFAULT 1,
    is_current_version BOOLEAN DEFAULT TRUE,
    download_count INTEGER DEFAULT 0,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    uploaded_by BIGINT NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_attachments_page_id ON attachments(page_id);
CREATE INDEX idx_attachments_space_id ON attachments(space_id);
CREATE INDEX idx_attachments_filename ON attachments(filename);
CREATE INDEX idx_attachments_file_hash ON attachments(file_hash);
```

### 2.5 评论系统域

#### 2.5.1 comments (评论表)
```sql
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    page_id BIGINT NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    parent_id BIGINT REFERENCES comments(id), -- 回复评论
    content TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, DELETED, HIDDEN
    is_resolved BOOLEAN DEFAULT FALSE,
    position_info JSONB, -- 页面中的位置信息
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT NOT NULL REFERENCES users(id),
    updated_by BIGINT REFERENCES users(id)
);

CREATE INDEX idx_comments_page_id ON comments(page_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);
CREATE INDEX idx_comments_created_by ON comments(created_by);
```

### 2.6 通知系统域

#### 2.6.1 notifications (通知表)
```sql
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    recipient_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    sender_id BIGINT REFERENCES users(id),
    type VARCHAR(50) NOT NULL, -- PAGE_CREATED, PAGE_UPDATED, COMMENT_ADDED, etc.
    title VARCHAR(200) NOT NULL,
    message TEXT,
    resource_type VARCHAR(30), -- PAGE, SPACE, COMMENT
    resource_id BIGINT,
    is_read BOOLEAN DEFAULT FALSE,
    read_at TIMESTAMP WITH TIME ZONE,
    data JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_recipient_id ON notifications(recipient_id);
CREATE INDEX idx_notifications_is_read ON notifications(recipient_id, is_read);
CREATE INDEX idx_notifications_type ON notifications(type);
```

#### 2.6.2 subscriptions (订阅表)
```sql
CREATE TABLE subscriptions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    resource_type VARCHAR(30) NOT NULL, -- PAGE, SPACE, USER
    resource_id BIGINT NOT NULL,
    subscription_type VARCHAR(30) NOT NULL, -- WATCH, FOLLOW, MENTION
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, resource_type, resource_id, subscription_type)
);
```

### 2.7 搜索域

#### 2.7.1 search_indices (搜索索引表)
```sql
CREATE TABLE search_indices (
    id BIGSERIAL PRIMARY KEY,
    resource_type VARCHAR(30) NOT NULL, -- PAGE, SPACE, COMMENT
    resource_id BIGINT NOT NULL,
    title VARCHAR(200),
    content TEXT,
    keywords TEXT[],
    index_data JSONB DEFAULT '{}',
    last_indexed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(resource_type, resource_id)
);

CREATE INDEX idx_search_indices_resource ON search_indices(resource_type, resource_id);
CREATE INDEX idx_search_indices_keywords ON search_indices USING GIN(keywords);
```

### 2.8 审计日志域

#### 2.8.1 audit_logs (审计日志表)
```sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    action VARCHAR(50) NOT NULL, -- CREATE, UPDATE, DELETE, VIEW, LOGIN, etc.
    resource_type VARCHAR(30), -- PAGE, SPACE, USER, etc.
    resource_id BIGINT,
    details JSONB DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    session_id VARCHAR(64),
    result VARCHAR(20) DEFAULT 'SUCCESS', -- SUCCESS, FAILURE, ERROR
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    tenant_id BIGINT NOT NULL
);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
CREATE INDEX idx_audit_logs_tenant_id ON audit_logs(tenant_id);
```

## 3. 高级数据结构设计

### 3.1 多租户数据隔离
```sql
-- 租户表
CREATE TABLE tenants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    domain VARCHAR(100) UNIQUE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    settings JSONB DEFAULT '{}',
    storage_limit BIGINT DEFAULT 10737418240, -- 10GB
    user_limit INTEGER DEFAULT 100,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Row Level Security (RLS) 示例
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY users_tenant_policy ON users FOR ALL TO application_role
    USING (tenant_id = current_setting('app.tenant_id')::BIGINT);
```

### 3.2 页面树形结构优化
```sql
-- 使用 Materialized Path 模式优化页面层级查询
ALTER TABLE pages ADD COLUMN path VARCHAR(1000);
ALTER TABLE pages ADD COLUMN depth INTEGER DEFAULT 0;

CREATE INDEX idx_pages_path ON pages(path);
CREATE INDEX idx_pages_depth ON pages(depth);

-- 页面层级查询示例
-- 获取某页面的所有子页面: WHERE path LIKE 'parent_path%'
-- 获取某层级的页面: WHERE depth = ?
```

### 3.3 版本控制优化
```sql
-- 页面版本关系表
CREATE TABLE page_versions (
    id BIGSERIAL PRIMARY KEY,
    page_id BIGINT NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    content_id BIGINT NOT NULL REFERENCES page_contents(id),
    change_summary VARCHAR(500),
    is_minor_edit BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT NOT NULL REFERENCES users(id),
    UNIQUE(page_id, version_number)
);
```

## 4. 缓存设计

### 4.1 Redis 缓存结构
```
用户缓存:
- user:profile:{user_id} -> 用户基本信息
- user:permissions:{user_id} -> 用户权限缓存
- user:spaces:{user_id} -> 用户可访问空间列表

空间缓存:
- space:info:{space_id} -> 空间基本信息
- space:pages:{space_id} -> 空间页面树
- space:members:{space_id} -> 空间成员列表

页面缓存:
- page:content:{page_id}:{version} -> 页面内容
- page:metadata:{page_id} -> 页面元数据
- page:children:{page_id} -> 子页面列表

搜索缓存:
- search:recent:{user_id} -> 用户最近搜索
- search:suggest:{keyword} -> 搜索建议
```

### 4.2 缓存过期策略
- **用户信息**: 30分钟
- **权限信息**: 5分钟
- **页面内容**: 1小时
- **空间信息**: 2小时
- **搜索结果**: 15分钟

## 5. 数据库性能优化

### 5.1 索引策略
```sql
-- 复合索引示例
CREATE INDEX idx_pages_space_status_updated ON pages(space_id, status, updated_at);
CREATE INDEX idx_audit_logs_tenant_action_date ON audit_logs(tenant_id, action, created_at);

-- 部分索引示例
CREATE INDEX idx_pages_active ON pages(space_id, updated_at) WHERE status = 'PUBLISHED';

-- 函数索引示例
CREATE INDEX idx_pages_title_lower ON pages(LOWER(title));
```

### 5.2 分区表设计
```sql
-- 审计日志按月分区
CREATE TABLE audit_logs_2024_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- 通知按季度分区
CREATE TABLE notifications_2024_q1 PARTITION OF notifications
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
```

### 5.3 读写分离配置
```yaml
# 数据源配置
datasource:
  master:
    url: jdbc:postgresql://master-db:5432/kms_wind
    username: master_user
    password: ${DB_MASTER_PASSWORD}

  slave:
    url: jdbc:postgresql://slave-db:5432/kms_wind
    username: slave_user
    password: ${DB_SLAVE_PASSWORD}
```

## 6. 数据迁移与备份

### 6.1 版本迁移脚本
```sql
-- V1.0.0 - 初始化数据库结构
-- V1.0.1 - 添加页面模板功能
-- V1.1.0 - 添加多租户支持
-- V1.2.0 - 添加审计日志分区

CREATE TABLE schema_version (
    version VARCHAR(20) PRIMARY KEY,
    description TEXT,
    installed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 6.2 备份策略
- **全量备份**: 每日凌晨2点
- **增量备份**: 每4小时一次
- **日志备份**: 实时备份WAL日志
- **跨地域备份**: 异地灾备

### 6.3 数据归档策略
```sql
-- 归档超过1年的审计日志
CREATE TABLE audit_logs_archive AS
SELECT * FROM audit_logs
WHERE created_at < CURRENT_DATE - INTERVAL '1 year';

-- 归档已删除页面的历史版本
CREATE TABLE page_contents_archive AS
SELECT pc.* FROM page_contents pc
JOIN pages p ON pc.page_id = p.id
WHERE p.status = 'DELETED'
  AND pc.created_at < CURRENT_DATE - INTERVAL '6 months';
```

## 7. 安全设计

### 7.1 数据加密
- **静态加密**: 使用PostgreSQL TDE (Transparent Data Encryption)
- **传输加密**: SSL/TLS连接
- **字段加密**: 敏感字段使用AES-256加密

### 7.2 访问控制
- **数据库用户**: 最小权限原则
- **连接池**: 限制最大连接数
- **SQL注入**: 使用参数化查询

### 7.3 审计要求
- **数据访问**: 记录所有数据访问
- **权限变更**: 记录权限分配和撤销
- **系统变更**: 记录系统配置变更

## 8. 数据库约束和完整性

### 8.1 详细约束条件

#### 8.1.1 用户表约束
```sql
-- 用户表添加详细约束
ALTER TABLE users
ADD CONSTRAINT chk_users_email_format
CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

ALTER TABLE users
ADD CONSTRAINT chk_users_username_length
CHECK (LENGTH(username) >= 3 AND LENGTH(username) <= 50);

ALTER TABLE users
ADD CONSTRAINT chk_users_status
CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED', 'DELETED'));

ALTER TABLE users
ADD CONSTRAINT chk_users_phone_format
CHECK (phone IS NULL OR phone ~* '^\+?[0-9]{10,15}$');

-- 用户名不能包含特殊字符
ALTER TABLE users
ADD CONSTRAINT chk_users_username_format
CHECK (username ~* '^[a-zA-Z0-9._-]+$');
```

#### 8.1.2 空间表约束
```sql
ALTER TABLE spaces
ADD CONSTRAINT chk_spaces_key_format
CHECK (space_key ~* '^[A-Z0-9_]{2,20}$');

ALTER TABLE spaces
ADD CONSTRAINT chk_spaces_type
CHECK (type IN ('TEAM', 'PERSONAL', 'PROJECT', 'DEPARTMENT'));

ALTER TABLE spaces
ADD CONSTRAINT chk_spaces_visibility
CHECK (visibility IN ('PUBLIC', 'PRIVATE', 'RESTRICTED'));

ALTER TABLE spaces
ADD CONSTRAINT chk_spaces_name_length
CHECK (LENGTH(name) >= 2 AND LENGTH(name) <= 100);
```

#### 8.1.3 页面表约束
```sql
ALTER TABLE pages
ADD CONSTRAINT chk_pages_title_length
CHECK (LENGTH(title) >= 1 AND LENGTH(title) <= 200);

ALTER TABLE pages
ADD CONSTRAINT chk_pages_status
CHECK (status IN ('DRAFT', 'PUBLISHED', 'ARCHIVED', 'DELETED'));

ALTER TABLE pages
ADD CONSTRAINT chk_pages_version_positive
CHECK (version_number > 0);

ALTER TABLE pages
ADD CONSTRAINT chk_pages_position_non_negative
CHECK (position >= 0);

-- 页面不能成为自己的父页面
ALTER TABLE pages
ADD CONSTRAINT chk_pages_not_self_parent
CHECK (id != parent_id);
```

#### 8.1.4 文件附件约束
```sql
ALTER TABLE attachments
ADD CONSTRAINT chk_attachments_file_size
CHECK (file_size > 0 AND file_size <= 52428800); -- 50MB max

ALTER TABLE attachments
ADD CONSTRAINT chk_attachments_version_positive
CHECK (version_number > 0);

ALTER TABLE attachments
ADD CONSTRAINT chk_attachments_filename_length
CHECK (LENGTH(filename) >= 1 AND LENGTH(filename) <= 255);
```

### 8.2 外键约束策略

#### 8.2.1 级联删除规则
```sql
-- 用户被删除时，相关数据处理
ALTER TABLE pages
ADD CONSTRAINT fk_pages_created_by
FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL;

-- 空间被删除时，页面也被删除
ALTER TABLE pages
ADD CONSTRAINT fk_pages_space_id
FOREIGN KEY (space_id) REFERENCES spaces(id) ON DELETE CASCADE;

-- 页面被删除时，评论也被删除
ALTER TABLE comments
ADD CONSTRAINT fk_comments_page_id
FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE;

-- 用户角色关联的级联删除
ALTER TABLE user_roles
ADD CONSTRAINT fk_user_roles_user_id
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;

ALTER TABLE user_roles
ADD CONSTRAINT fk_user_roles_role_id
FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE;
```

#### 8.2.2 软删除机制
```sql
-- 为关键表添加软删除字段
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP WITH TIME ZONE;
ALTER TABLE spaces ADD COLUMN deleted_at TIMESTAMP WITH TIME ZONE;
ALTER TABLE pages ADD COLUMN deleted_at TIMESTAMP WITH TIME ZONE;

-- 软删除索引
CREATE INDEX idx_users_deleted_at ON users(deleted_at) WHERE deleted_at IS NOT NULL;
CREATE INDEX idx_spaces_deleted_at ON spaces(deleted_at) WHERE deleted_at IS NOT NULL;
CREATE INDEX idx_pages_deleted_at ON pages(deleted_at) WHERE deleted_at IS NOT NULL;
```

### 8.3 数据一致性保证

#### 8.3.1 触发器设计
```sql
-- 自动更新 updated_at 字段
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 为相关表创建触发器
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_pages_updated_at
    BEFORE UPDATE ON pages
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- 页面内容变更时更新页面的更新时间
CREATE OR REPLACE FUNCTION update_page_on_content_change()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE pages SET updated_at = CURRENT_TIMESTAMP
    WHERE id = NEW.page_id;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_page_on_content_insert
    AFTER INSERT ON page_contents
    FOR EACH ROW EXECUTE FUNCTION update_page_on_content_change();
```

#### 8.3.2 数据校验存储过程
```sql
-- 校验空间权限一致性
CREATE OR REPLACE FUNCTION validate_space_permissions()
RETURNS BOOLEAN AS $$
DECLARE
    inconsistent_count INTEGER;
BEGIN
    -- 检查是否存在不一致的权限配置
    SELECT COUNT(*) INTO inconsistent_count
    FROM space_permissions sp
    LEFT JOIN spaces s ON sp.space_id = s.id
    LEFT JOIN users u ON sp.user_id = u.id
    WHERE s.id IS NULL OR u.id IS NULL;

    IF inconsistent_count > 0 THEN
        RAISE NOTICE '发现 % 个不一致的权限配置', inconsistent_count;
        RETURN FALSE;
    END IF;

    RETURN TRUE;
END;
$$ language 'plpgsql';
```

## 9. 数据库监控与运维

### 9.1 关键监控指标

#### 9.1.1 性能监控指标
```sql
-- 数据库性能监控视图
CREATE VIEW db_performance_metrics AS
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation,
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE schemaname = 'public'
ORDER BY tablename, attname;

-- 慢查询监控
CREATE VIEW slow_queries AS
SELECT
    query,
    calls,
    total_time,
    mean_time,
    max_time,
    rows
FROM pg_stat_statements
WHERE mean_time > 1000  -- 超过1秒的查询
ORDER BY mean_time DESC;
```

#### 9.1.2 业务监控指标
```sql
-- 业务数据监控视图
CREATE VIEW business_metrics AS
SELECT
    'total_users' as metric_name,
    COUNT(*) as metric_value,
    CURRENT_TIMESTAMP as measured_at
FROM users WHERE deleted_at IS NULL
UNION ALL
SELECT
    'active_users_today',
    COUNT(DISTINCT user_id),
    CURRENT_TIMESTAMP
FROM audit_logs
WHERE created_at >= CURRENT_DATE
UNION ALL
SELECT
    'total_pages',
    COUNT(*),
    CURRENT_TIMESTAMP
FROM pages WHERE status != 'DELETED'
UNION ALL
SELECT
    'pages_created_today',
    COUNT(*),
    CURRENT_TIMESTAMP
FROM pages
WHERE created_at >= CURRENT_DATE;
```

### 9.2 连接池配置

#### 9.2.1 HikariCP 配置
```yaml
spring:
  datasource:
    hikari:
      # 连接池基本配置
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 600000        # 10分钟
      max-lifetime: 1800000       # 30分钟
      connection-timeout: 30000   # 30秒

      # 连接测试
      connection-test-query: SELECT 1
      validation-timeout: 5000

      # 连接泄露检测
      leak-detection-threshold: 60000  # 1分钟

      # 连接池名称
      pool-name: KmsWindCP

      # 数据库连接属性
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        useLocalSessionState: true
        rewriteBatchedStatements: true
        cacheResultSetMetadata: true
        cacheServerConfiguration: true
        elideSetAutoCommits: true
        maintainTimeStats: false
```

#### 9.2.2 数据源路由配置
```java
@Configuration
public class DatabaseConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(DatabaseType.MASTER, masterDataSource());
        dataSourceMap.put(DatabaseType.SLAVE, slaveDataSource());
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        return routingDataSource;
    }
}
```

### 9.3 数据库健康检查

#### 9.3.1 健康检查脚本
```sql
-- 数据库健康检查函数
CREATE OR REPLACE FUNCTION health_check()
RETURNS TABLE(
    check_name TEXT,
    status TEXT,
    message TEXT,
    checked_at TIMESTAMP WITH TIME ZONE
) AS $$
BEGIN
    -- 检查数据库连接
    RETURN QUERY SELECT
        'database_connection'::TEXT,
        'OK'::TEXT,
        'Database connection is healthy'::TEXT,
        CURRENT_TIMESTAMP;

    -- 检查表空间使用率
    RETURN QUERY
    WITH tablespace_usage AS (
        SELECT
            spcname,
            pg_size_pretty(pg_tablespace_size(spcname)) as size,
            CASE
                WHEN pg_tablespace_size(spcname) > 85 * 1024 * 1024 * 1024 THEN 'WARNING'
                ELSE 'OK'
            END as status
        FROM pg_tablespace
    )
    SELECT
        'tablespace_usage'::TEXT,
        tu.status,
        ('Tablespace ' || tu.spcname || ' size: ' || tu.size)::TEXT,
        CURRENT_TIMESTAMP
    FROM tablespace_usage tu;

    -- 检查长时间运行的查询
    RETURN QUERY
    SELECT
        'long_running_queries'::TEXT,
        CASE
            WHEN COUNT(*) > 5 THEN 'WARNING'
            ELSE 'OK'
        END,
        ('Found ' || COUNT(*) || ' long running queries')::TEXT,
        CURRENT_TIMESTAMP
    FROM pg_stat_activity
    WHERE state = 'active'
      AND query_start < CURRENT_TIMESTAMP - INTERVAL '5 minutes';

END;
$$ language 'plpgsql';
```

#### 9.3.2 监控告警规则
```yaml
# Prometheus 监控规则
groups:
  - name: postgresql.rules
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: PostgreSQL instance is down

      - alert: PostgreSQLHighConnections
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: PostgreSQL high connections usage

      - alert: PostgreSQLSlowQueries
        expr: pg_stat_statements_mean_time_ms > 1000
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: PostgreSQL slow queries detected

      - alert: PostgreSQLDiskSpaceUsage
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes > 0.85
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: PostgreSQL disk space usage is high
```

### 9.4 定期维护任务

#### 9.4.1 数据库维护脚本
```sql
-- 定期维护任务
CREATE OR REPLACE FUNCTION daily_maintenance()
RETURNS VOID AS $$
BEGIN
    -- 更新表统计信息
    ANALYZE;

    -- 清理过期的审计日志（保留1年）
    DELETE FROM audit_logs
    WHERE created_at < CURRENT_DATE - INTERVAL '1 year';

    -- 清理过期的通知（保留3个月）
    DELETE FROM notifications
    WHERE created_at < CURRENT_DATE - INTERVAL '3 months'
      AND is_read = true;

    -- 清理过期的搜索缓存
    DELETE FROM search_indices
    WHERE last_indexed_at < CURRENT_DATE - INTERVAL '7 days';

    -- 重建碎片化严重的索引
    REINDEX INDEX CONCURRENTLY idx_pages_space_status_updated;
    REINDEX INDEX CONCURRENTLY idx_audit_logs_tenant_action_date;

    RAISE NOTICE '每日维护任务完成';
END;
$$ language 'plpgsql';

-- 创建定时任务（需要pg_cron扩展）
SELECT cron.schedule('daily-maintenance', '0 2 * * *', 'SELECT daily_maintenance();');
```

#### 9.4.2 性能优化建议
```sql
-- 性能优化分析函数
CREATE OR REPLACE FUNCTION performance_analysis()
RETURNS TABLE(
    table_name TEXT,
    size_pretty TEXT,
    seq_scan BIGINT,
    seq_tup_read BIGINT,
    idx_scan BIGINT,
    suggestion TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        t.schemaname||'.'||t.tablename,
        pg_size_pretty(pg_total_relation_size(t.schemaname||'.'||t.tablename)),
        s.seq_scan,
        s.seq_tup_read,
        s.idx_scan,
        CASE
            WHEN s.seq_scan > s.idx_scan AND s.seq_tup_read > 10000
            THEN '建议添加索引优化查询'
            WHEN pg_total_relation_size(t.schemaname||'.'||t.tablename) > 100*1024*1024
            THEN '大表，考虑分区'
            ELSE '性能良好'
        END
    FROM pg_tables t
    LEFT JOIN pg_stat_user_tables s ON s.relname = t.tablename
    WHERE t.schemaname = 'public'
    ORDER BY pg_total_relation_size(t.schemaname||'.'||t.tablename) DESC;
END;
$$ language 'plpgsql';
```