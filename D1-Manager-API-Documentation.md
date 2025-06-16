# D1 Manager API 接口调用说明文档

## Cloudflare Access 配置指南

在使用API之前，您需要先配置 Cloudflare Access 来保护您的 D1 Manager 应用。

### 前置要求

1. 已部署 D1 Manager 到 Cloudflare Pages
2. 拥有 Cloudflare 账户并已添加您的域名
3. 已启用 Cloudflare Zero Trust 服务

### Step 1: 启用 Zero Trust

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 选择您的账户
3. 在左侧导航栏中找到 **"Zero Trust"**
4. 点击 **"Get started"** 开始设置（如果尚未设置）

### Step 2: 配置 Access 应用

#### 2.1 创建 Access 应用

1. 进入 **Zero Trust** → **Access** → **Applications**
2. 点击 **"Add an application"**
3. 选择 **"Self-hosted"**

#### 2.2 应用基础设置

```yaml
Application Configuration:
  Application name: "D1 Manager Protection"
  Application domain: your-d1-manager.pages.dev
  Path: 
    - /api/*     # 保护所有 API 端点
    - /db/*      # 保护数据库管理页面（可选）
    - /*         # 或者保护整个应用
  
Advanced Settings:
  Session Duration: 24h
  Enable automatic cloudflared authentication: ✓
```

**具体操作：**
1. **Application name**: 输入 `D1 Manager Protection`
2. **Subdomain**: 输入您的 Pages 子域名，如 `your-d1-manager`
3. **Domain**: 选择 `pages.dev`（如果使用自定义域名，选择对应域名）
4. **Path**: 输入 `/api/*` （重要：保护所有API端点）

#### 2.3 配置访问策略

创建应用后，需要配置谁可以访问：

**策略 1: 管理员访问**
```yaml
Policy Name: "Admin Access"
Action: Allow
Rules:
  - Include:
      - Emails: admin@yourcompany.com, user@yourcompany.com
    # 或者
  - Include:
      - Email domain: yourcompany.com
```

**策略 2: Service Token 访问（用于API调用）**
```yaml
Policy Name: "API Service Tokens"
Action: Service Auth
Rules:
  - Include:
      - Service Token: [稍后创建的Service Token名称]
```

### Step 3: 创建 Service Token

Service Token 用于程序化的API访问，不需要人工登录。

#### 3.1 创建 Token

1. 进入 **Zero Trust** → **Access** → **Service Auth** → **Service Tokens**
2. 点击 **"Create Service Token"**
3. 配置如下：

```yaml
Service Token Configuration:
  Name: "D1-Manager-API-Token"
  Client Secret will expire in: Never (或选择适当的过期时间)
```

4. 点击 **"Generate token"**
5. **重要**：立即复制并安全保存生成的信息：
   ```
   Client ID: 7a9b8c6d5e4f3g2h1i.access.service.token
   Client Secret: 7a9b8c6d5e4f3g2h1i0j9k8l7m6n5o4p3q2r1s0t9u8v7w6x5y4z3a2b1c0d9e8f
   ```

#### 3.2 将 Service Token 添加到访问策略

返回到之前创建的 Access 应用：

1. 进入 **Applications** → **D1 Manager Protection** → **Policies**
2. 编辑 **"API Service Tokens"** 策略
3. 在 **Include** 规则中：
   - 选择 **"Service Token"**
   - 选择刚才创建的 **"D1-Manager-API-Token"**
4. 保存策略

### Step 4: 验证配置

#### 4.1 测试 Web 访问

在浏览器中访问您的 D1 Manager URL：
```
https://your-d1-manager.pages.dev
```

您应该会看到 Cloudflare Access 登录页面，输入您在策略中配置的邮箱进行登录。

#### 4.2 测试 API 访问

**无认证测试（应该失败）：**
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db"
# 预期响应: 403 Forbidden 或重定向到登录页面
```

**使用 Service Token 测试（应该成功）：**
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db" \
  -H "CF-Access-Client-Id: 7a9b8c6d5e4f3g2h1i.access.service.token" \
  -H "CF-Access-Client-Secret: 7a9b8c6d5e4f3g2h1i0j9k8l7m6n5o4p3q2r1s0t9u8v7w6x5y4z3a2b1c0d9e8f"
# 预期响应: JSON 数组包含数据库列表
```

### Step 5: 高级配置（可选）

#### 5.1 精细化路径保护

您可以创建多个 Access 应用来实现更精细的访问控制：

**应用 1: Web UI 保护**
```yaml
Name: "D1 Manager Web UI"
Domain: your-d1-manager.pages.dev
Path: /
Policy: Interactive users only (email login)
```

**应用 2: API 保护**
```yaml
Name: "D1 Manager API"
Domain: your-d1-manager.pages.dev
Path: /api/*
Policy: Service Tokens + Authorized users
```

**应用 3: 危险操作保护**
```yaml
Name: "D1 Manager Admin API"
Domain: your-d1-manager.pages.dev
Path: 
  - /api/*/exec
  - /api/*/*/data (PUT/DELETE)
Policy: Admins only + Special service tokens
```

#### 5.2 配置多个 Service Token

为不同的用途创建不同的 Service Token：

```yaml
Token 1: "D1-Manager-Read-Only"
  Permissions: GET requests only
  
Token 2: "D1-Manager-Full-Access" 
  Permissions: All operations
  
Token 3: "D1-Manager-Analytics"
  Permissions: Read-only for analytics database
```

#### 5.3 IP 白名单配置

在策略中添加 IP 限制：

```yaml
Policy Rules:
  - Include:
      - Service Token: D1-Manager-API-Token
      - IP ranges: 
          - 192.168.1.0/24
          - 10.0.0.0/8
```

### Step 6: 监控和日志

#### 6.1 访问日志

查看 API 访问日志：
1. 进入 **Zero Trust** → **Logs** → **Access**
2. 筛选应用程序: **D1 Manager Protection**
3. 查看访问记录，包括：
   - 访问时间
   - 用户/Service Token
   - 访问的路径
   - 访问结果（允许/拒绝）

#### 6.2 设置告警

配置异常访问告警：
1. 进入 **Zero Trust** → **Settings** → **Notifications**
2. 添加通知策略：
   ```yaml
   Trigger: Failed authentication attempts > 5
   Action: Send email to admin@yourcompany.com
   ```

### 故障排除

#### 常见问题 1: 403 Forbidden
**症状**: API 调用返回 403 错误
**解决方案**:
1. 检查 Service Token 是否正确配置
2. 确认 Access 应用的路径配置包含 `/api/*`
3. 验证策略中包含了对应的 Service Token

#### 常见问题 2: 重定向到登录页面
**症状**: API 调用被重定向到 Cloudflare Access 登录页面
**解决方案**:
1. 确保请求包含正确的 `CF-Access-Client-Id` 和 `CF-Access-Client-Secret` 头部
2. 检查头部名称的拼写和大小写

#### 常见问题 3: Token 过期
**症状**: 之前有效的 Token 突然失效
**解决方案**:
1. 检查 Service Token 的过期时间
2. 重新生成新的 Token
3. 更新应用程序中的凭据

#### 常见问题 4: 策略冲突
**症状**: 访问行为不符合预期
**解决方案**:
1. 检查是否有多个 Access 应用覆盖了相同的路径
2. 确认策略优先级和执行顺序
3. 简化策略配置进行测试

### 安全最佳实践

1. **Token 管理**:
   - 定期轮换 Service Token
   - 为不同环境使用不同的 Token
   - 妥善保存 Token，避免硬编码

2. **访问控制**:
   - 遵循最小权限原则
   - 为不同操作设置不同的访问级别
   - 定期审查访问策略

3. **监控**:
   - 启用访问日志记录
   - 设置异常访问告警
   - 定期检查访问模式

4. **网络安全**:
   - 配置 IP 白名单（如适用）
   - 使用 HTTPS 进行所有通信
   - 考虑额外的网络层安全措施

---

## 认证方式

使用 Cloudflare Access Service Tokens 进行认证，所有API请求需要包含以下HTTP头：

```
CF-Access-Client-Id: your-client-id.access.service.token
CF-Access-Client-Secret: your-client-secret
Content-Type: application/json
```

## 基础信息

**Base URL**: `https://your-d1-manager.pages.dev`

## API 接口列表

### 1. 系统信息

#### GET /api

获取系统基本信息，包括可用数据库列表、AI助手状态和版本信息。

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
{
  "db": ["default", "test", "analytics"],
  "assistant": "OpenAI (gpt-4.1-mini)" | "Cloudflare (@cf/mistral/mistral-7b-instruct-v0.1)" | null,
  "version": "0.0.0"
}
```

### 2. 数据库管理

#### GET /api/db

获取所有可用数据库列表。

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
["default", "test", "analytics"]
```

#### GET /api/db/{database}

获取指定数据库的表信息，包括表结构和记录数。

**路径参数**:
- `database`: 数据库名称

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db/default" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
[
  {
    "name": "users",
    "columns": [
      {
        "cid": 0,
        "name": "id",
        "type": "INTEGER",
        "notnull": 1,
        "dflt_value": null,
        "pk": 1
      },
      {
        "cid": 1,
        "name": "email",
        "type": "TEXT",
        "notnull": 1,
        "dflt_value": null,
        "pk": 0
      }
    ],
    "count": 150
  }
]
```

#### POST /api/db/{database}

在指定数据库中创建新表。

**路径参数**:
- `database`: 数据库名称

**请求体**:
```json
{
  "name": "new_table",
  "columns": {
    "id": "INTEGER PRIMARY KEY",
    "name": "TEXT NOT NULL",
    "created_at": "TEXT DEFAULT CURRENT_TIMESTAMP"
  }
}
```

**请求示例**:
```bash
curl -X POST "https://your-d1-manager.pages.dev/api/db/default" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "products",
    "columns": {
      "id": "INTEGER PRIMARY KEY",
      "name": "TEXT NOT NULL",
      "price": "REAL"
    }
  }'
```

**响应格式**:
```json
{
  "success": true,
  "meta": {
    "duration": 5.2,
    "last_row_id": 0,
    "changes": 0,
    "served_by": "v1-prod",
    "internal_stats": null
  }
}
```

### 3. SQL 执行

#### POST /api/db/{database}/exec

执行任意SQL语句，支持多语句执行。

**路径参数**:
- `database`: 数据库名称

**请求体**:
```json
{
  "query": "SELECT * FROM users WHERE age > 18; UPDATE users SET status = 'active' WHERE id = 1;"
}
```

**请求示例**:
```bash
curl -X POST "https://your-d1-manager.pages.dev/api/db/default/exec" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT COUNT(*) as total FROM users"}'
```

**响应格式**:
```json
{
  "results": [
    [
      {"total": 150}
    ]
  ],
  "duration": 12.5,
  "count": 1
}
```

#### POST /api/db/{database}/all

执行参数化SQL查询，返回所有结果。

**路径参数**:
- `database`: 数据库名称

**请求体**:
```json
{
  "query": "SELECT * FROM users WHERE age > ? AND city = ?",
  "params": ["18", "New York"]
}
```

**请求示例**:
```bash
curl -X POST "https://your-d1-manager.pages.dev/api/db/default/all" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT * FROM users WHERE status = ?",
    "params": ["active"]
  }'
```

**响应格式**:
```json
{
  "results": [
    {
      "id": 1,
      "email": "user1@example.com",
      "status": "active",
      "created_at": "2024-01-01 10:00:00"
    },
    {
      "id": 2,
      "email": "user2@example.com", 
      "status": "active",
      "created_at": "2024-01-02 11:30:00"
    }
  ],
  "success": true,
  "meta": {
    "duration": 8.3,
    "last_row_id": 0,
    "changes": 0,
    "served_by": "v1-prod",
    "internal_stats": null
  }
}
```

### 4. AI 助手

#### POST /api/db/{database}/assistant

使用AI助手将自然语言查询转换为SQL语句。

**路径参数**:
- `database`: 数据库名称

**请求体**:
```json
{
  "q": "show me all users who registered last month",
  "t": "users"
}
```

**请求参数说明**:
- `q`: 自然语言查询（必需）
- `t`: 目标表名（可选，用于优化AI上下文）

**请求示例**:
```bash
curl -X POST "https://your-d1-manager.pages.dev/api/db/default/assistant" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "q": "find top 5 users by order count",
    "t": "users"
  }'
```

**响应格式**:
```json
{
  "sql": "SELECT u.*, COUNT(o.id) as order_count FROM users u JOIN orders o ON u.id = o.user_id GROUP BY u.id ORDER BY order_count DESC LIMIT 5"
}
```

**错误响应**:
```json
{
  "error": "no backend"
}
```

### 5. 表管理

#### GET /api/db/{database}/{table}

获取指定表的元数据信息。

**路径参数**:
- `database`: 数据库名称
- `table`: 表名

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db/default/users" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
{
  "count": 150
}
```

#### DELETE /api/db/{database}/{table}

删除指定表。

**路径参数**:
- `database`: 数据库名称
- `table`: 表名

**请求示例**:
```bash
curl -X DELETE "https://your-d1-manager.pages.dev/api/db/default/old_table" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
{
  "success": true,
  "meta": {
    "duration": 3.2,
    "last_row_id": 0,
    "changes": 0,
    "served_by": "v1-prod",
    "internal_stats": null
  }
}
```

### 6. 表数据操作

#### GET /api/db/{database}/{table}/data

查询表数据，支持分页、排序、筛选。

**路径参数**:
- `database`: 数据库名称
- `table`: 表名

**查询参数**:
- `offset`: 偏移量，默认 0
- `limit`: 限制数量，默认 100
- `order`: 排序字段
- `dir`: 排序方向，ASC 或 DESC，默认 ASC
- `select`: 选择字段，默认 *
- `where`: WHERE 条件

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db/default/users/data?offset=0&limit=10&order=created_at&dir=DESC&where=status='active'" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
[
  {
    "id": 1,
    "email": "user1@example.com",
    "status": "active",
    "created_at": "2024-01-01 10:00:00"
  },
  {
    "id": 2,
    "email": "user2@example.com",
    "status": "active", 
    "created_at": "2024-01-02 11:30:00"
  }
]
```

#### POST /api/db/{database}/{table}/data

向表中插入新记录。

**路径参数**:
- `database`: 数据库名称
- `table`: 表名

**请求体**:
```json
{
  "email": "newuser@example.com",
  "name": "New User",
  "status": "active"
}
```

**请求示例**:
```bash
curl -X POST "https://your-d1-manager.pages.dev/api/db/default/users/data" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newuser@example.com",
    "name": "John Doe", 
    "status": "active"
  }'
```

**响应格式**:
```json
{
  "success": true,
  "meta": {
    "duration": 4.1,
    "last_row_id": 151,
    "changes": 1,
    "served_by": "v1-prod",
    "internal_stats": null
  }
}
```

#### PUT /api/db/{database}/{table}/data

更新表中的记录。

**路径参数**:
- `database`: 数据库名称
- `table`: 表名

**查询参数**:
WHERE 条件参数，用于指定要更新的记录。

**请求体**:
要更新的字段和值。

**请求示例**:
```bash
curl -X PUT "https://your-d1-manager.pages.dev/api/db/default/users/data?id=1" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "inactive",
    "updated_at": "2024-01-03 15:30:00"
  }'
```

**响应格式**:
```json
{
  "success": true,
  "meta": {
    "duration": 2.8,
    "last_row_id": 0,
    "changes": 1,
    "served_by": "v1-prod",
    "internal_stats": null
  }
}
```

#### DELETE /api/db/{database}/{table}/data

删除表中的记录。

**路径参数**:
- `database`: 数据库名称
- `table`: 表名

**查询参数**:
WHERE 条件参数，用于指定要删除的记录。

**请求示例**:
```bash
curl -X DELETE "https://your-d1-manager.pages.dev/api/db/default/users/data?id=1" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应格式**:
```json
{
  "success": true,
  "meta": {
    "duration": 1.9,
    "last_row_id": 0,
    "changes": 1,
    "served_by": "v1-prod",
    "internal_stats": null
  }
}
```

### 7. 数据库导出

#### GET /api/db/{database}/dump

重定向到数据库文件下载链接。

**路径参数**:
- `database`: 数据库名称

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db/default/dump" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret"
```

**响应**: 301 重定向到 `/api/db/{database}/dump/db-{database}.sqlite3`

#### GET /api/db/{database}/dump/{filename}

下载数据库文件的二进制数据。

**路径参数**:
- `database`: 数据库名称
- `filename`: 文件名

**请求示例**:
```bash
curl -X GET "https://your-d1-manager.pages.dev/api/db/default/dump/db-default.sqlite3" \
  -H "CF-Access-Client-Id: your-client-id.access.service.token" \
  -H "CF-Access-Client-Secret: your-client-secret" \
  -o "database.sqlite3"
```

**响应**: 二进制数据流，Content-Type: `application/octet-stream`

## 错误处理

API 使用标准 HTTP 状态码：

- `200`: 请求成功
- `301`: 重定向（用于导出接口）
- `400`: 请求参数错误
- `401`: 认证失败
- `403`: 访问被拒绝
- `404`: 资源不存在
- `500`: 服务器内部错误

**错误响应格式**:
```json
{
  "message": "Database not found",
  "code": 404
}
```

## 使用注意事项

1. **认证**: 所有API请求必须包含有效的 Cloudflare Access Service Token
2. **SQL安全**: `/exec` 接口支持执行任意SQL，包括DDL和DML操作，请谨慎使用
3. **参数化查询**: 推荐使用 `/all` 接口进行参数化查询以防止SQL注入
4. **分页**: 查询大量数据时使用 `offset` 和 `limit` 参数进行分页
5. **AI助手**: AI功能需要配置 OpenAI API Key 或 Cloudflare AI Worker
6. **开发环境**: 开发模式下API会代理到生产环境

## SDK 示例

### JavaScript/Node.js
```javascript
class D1ManagerAPI {
  constructor(baseURL, clientId, clientSecret) {
    this.baseURL = baseURL;
    this.headers = {
      'CF-Access-Client-Id': clientId,
      'CF-Access-Client-Secret': clientSecret,
      'Content-Type': 'application/json'
    };
  }

  async getDatabases() {
    const response = await fetch(`${this.baseURL}/api/db`, {
      headers: this.headers
    });
    return response.json();
  }

  async executeSQL(database, query) {
    const response = await fetch(`${this.baseURL}/api/db/${database}/exec`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify({ query })
    });
    return response.json();
  }

  async queryData(database, table, options = {}) {
    const params = new URLSearchParams(options);
    const response = await fetch(`${this.baseURL}/api/db/${database}/${table}/data?${params}`, {
      headers: this.headers
    });
    return response.json();
  }

  async insertData(database, table, data) {
    const response = await fetch(`${this.baseURL}/api/db/${database}/${table}/data`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify(data)
    });
    return response.json();
  }

  async updateData(database, table, data, where) {
    const params = new URLSearchParams(where);
    const response = await fetch(`${this.baseURL}/api/db/${database}/${table}/data?${params}`, {
      method: 'PUT',
      headers: this.headers,
      body: JSON.stringify(data)
    });
    return response.json();
  }

  async deleteData(database, table, where) {
    const params = new URLSearchParams(where);
    const response = await fetch(`${this.baseURL}/api/db/${database}/${table}/data?${params}`, {
      method: 'DELETE',
      headers: this.headers
    });
    return response.json();
  }

  async getAIAssistant(database, question, table = null) {
    const body = { q: question };
    if (table) body.t = table;
    
    const response = await fetch(`${this.baseURL}/api/db/${database}/assistant`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify(body)
    });
    return response.json();
  }
}

// 使用示例
const api = new D1ManagerAPI(
  'https://your-d1-manager.pages.dev',
  'your-client-id.access.service.token',
  'your-client-secret'
);

// 获取数据库列表
const databases = await api.getDatabases();
console.log('Available databases:', databases);

// 执行SQL查询
const result = await api.executeSQL('default', 'SELECT COUNT(*) FROM users');
console.log('Query result:', result);

// 查询表数据
const users = await api.queryData('default', 'users', { 
  limit: 10, 
  order: 'created_at', 
  dir: 'DESC' 
});
console.log('Users:', users);

// 插入新数据
const insertResult = await api.insertData('default', 'users', {
  email: 'newuser@example.com',
  name: 'New User',
  status: 'active'
});
console.log('Insert result:', insertResult);

// 使用AI助手
const aiResult = await api.getAIAssistant('default', 'show me active users', 'users');
console.log('AI generated SQL:', aiResult.sql);
```

### Python
```python
import requests
import json

class D1ManagerAPI:
    def __init__(self, base_url, client_id, client_secret):
        self.base_url = base_url
        self.headers = {
            'CF-Access-Client-Id': client_id,
            'CF-Access-Client-Secret': client_secret,
            'Content-Type': 'application/json'
        }
    
    def get_databases(self):
        response = requests.get(f"{self.base_url}/api/db", headers=self.headers)
        return response.json()
    
    def execute_sql(self, database, query):
        data = {'query': query}
        response = requests.post(
            f"{self.base_url}/api/db/{database}/exec",
            headers=self.headers,
            json=data
        )
        return response.json()
    
    def query_data(self, database, table, **options):
        params = {k: v for k, v in options.items() if v is not None}
        response = requests.get(
            f"{self.base_url}/api/db/{database}/{table}/data",
            headers=self.headers,
            params=params
        )
        return response.json()
    
    def insert_data(self, database, table, data):
        response = requests.post(
            f"{self.base_url}/api/db/{database}/{table}/data",
            headers=self.headers,
            json=data
        )
        return response.json()

# 使用示例
api = D1ManagerAPI(
    'https://your-d1-manager.pages.dev',
    'your-client-id.access.service.token',
    'your-client-secret'
)

# 获取数据库列表
databases = api.get_databases()
print(f"Available databases: {databases}")

# 执行SQL查询
result = api.execute_sql('default', 'SELECT COUNT(*) FROM users')
print(f"Query result: {result}")

# 查询表数据
users = api.query_data('default', 'users', limit=10, order='created_at', dir='DESC')
print(f"Users: {users}")
```

## 常见用例

### 1. 数据迁移
```bash
# 1. 导出源数据库
curl -X GET "https://your-d1-manager.pages.dev/api/db/source/dump/db-source.sqlite3" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -o "backup.sqlite3"

# 2. 查询源数据
curl -X POST "https://your-d1-manager.pages.dev/api/db/source/all" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -d '{"query": "SELECT * FROM users"}'

# 3. 在目标数据库插入数据
curl -X POST "https://your-d1-manager.pages.dev/api/db/target/users/data" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -d '{"email": "user@example.com", "name": "User"}'
```

### 2. 数据分析
```bash
# 使用AI助手生成分析SQL
curl -X POST "https://your-d1-manager.pages.dev/api/db/analytics/assistant" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -d '{"q": "show monthly user registration trend", "t": "users"}'

# 执行生成的SQL
curl -X POST "https://your-d1-manager.pages.dev/api/db/analytics/exec" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -d '{"query": "SELECT DATE_TRUNC(\"month\", created_at) as month, COUNT(*) as registrations FROM users GROUP BY month ORDER BY month"}'
```

### 3. 批量操作
```bash
# 批量更新用户状态
curl -X POST "https://your-d1-manager.pages.dev/api/db/default/exec" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -d '{"query": "UPDATE users SET status = \"inactive\" WHERE last_login < \"2024-01-01\""}'

# 批量删除过期数据
curl -X POST "https://your-d1-manager.pages.dev/api/db/default/exec" \
  -H "CF-Access-Client-Id: xxx" \
  -H "CF-Access-Client-Secret: xxx" \
  -d '{"query": "DELETE FROM logs WHERE created_at < DATE(\"now\", \"-30 days\")"}'
```

---

**版本**: v1.0  
**更新日期**: $(date)  
**支持**: 基于 D1 Manager 源码分析生成，涵盖所有API接口 