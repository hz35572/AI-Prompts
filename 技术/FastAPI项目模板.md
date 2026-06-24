# FastAPI 生产级项目模板生成器

请根据以下规范生成一个完整的 FastAPI 项目基础框架。该模板应适用于从小型 API 到大型复杂系统的各种规模项目，并遵循现代 Python 异步最佳实践。

## 1. 项目目录结构

生成以下目录结构：

```text
project_root/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI 应用工厂 + lifespan 管理
│   ├── agent/                     # (可选) AI Agent / 工作流编排
│   │   ├── __init__.py
│   │   ├── state.py
│   │   └── workflow.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py                # 共享依赖注入：DB Session、当前用户等
│   │   ├── router.py              # API 路由聚合器（版本控制入口）
│   │   ├── utils.py               # API 层通用工具函数
│   │   └── routers/
│   │       ├── __init__.py
│   │       └── v1/
│   │           ├── __init__.py    # v1 路由注册中心
│   │           └── *.py           # 各模块路由文件
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py              # Pydantic Settings 统一配置
│   │   ├── database.py            # 数据库兼容层/连接导出
│   │   ├── security.py            # 密码哈希、JWT、加密工具
│   │   └── logging_config.py      # 统一日志配置（含脱敏、轮转、队列）
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py                # SQLAlchemy DeclarativeBase + 命名规范
│   │   ├── session.py             # 数据库会话管理器
│   │   └── models/
│   │       ├── __init__.py
│   │       ├── mixins.py          # 通用模型 Mixin（如 TimestampMixin）
│   │       └── *.py               # 各业务模型
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── common.py              # 通用响应结构：ApiResponse, ApiErrorResponse
│   │   └── *.py                   # 各模块 Pydantic Schema
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── *.py                   # 数据访问层（纯数据库操作）
│   ├── services/
│   │   ├── __init__.py
│   │   └── *.py                   # 业务逻辑层（编排 repository）
│   └── tasks/
│       └── *.py                   # 后台任务辅助
├── alembic/                       # 数据库迁移
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── tests/
│   ├── integration/               # 集成测试（API 全链路）
│   └── unit/                      # 单元测试（纯函数/服务/校验器）
├── pyproject.toml                 # 项目配置 + 依赖 + pytest 配置
├── docker-compose.yml             # 本地开发依赖（Postgres/Redis等）
└── README.md
```

## 2. 核心配置文件模板

### 2.1 pyproject.toml

- 使用 `setuptools` 构建系统
- Python >= 3.11
- 核心依赖：`fastapi>=0.110`, `uvicorn[standard]>=0.27`, `pydantic>=2.6`, `pydantic-settings>=2.2`, `sqlalchemy>=2.0`, `asyncpg>=0.29`, `alembic>=1.13`, `python-jose[cryptography]>=3.3`, `passlib[bcrypt]>=1.7`, `redis>=5.0`
- 测试依赖：`pytest>=8.0`, `pytest-asyncio>=0.23`, `httpx>=0.27`
- pytest 配置：`asyncio_mode = "auto"`, `testpaths = ["tests"]`, `pythonpath = ["."]`

### 2.2 app/core/config.py

- 使用 `pydantic-settings` 的 `BaseSettings`
- 环境变量统一使用 `{PROJECT_PREFIX}_` 前缀（如 `APP_`），避免冲突
- 支持 `.env` 文件自动加载（从当前目录或父目录查找）
- 配置分类：项目基础、数据库、Redis、日志、认证、CORS、邮件、外部服务
- 使用 `@field_validator` 进行安全配置校验（如生产环境禁止默认密钥）
- 使用 `@model_validator(mode="after")` 动态构建连接 URL
- 提供 `@lru_cache` 缓存的 `get_settings()` 工厂函数
- 小写属性名作为大写字段的兼容别名

### 2.3 docker-compose.yml

- 包含：PostgreSQL（端口映射到宿主机非默认端口，如 5433:5432）、Redis、（可选）向量数据库/对象存储
- 使用命名卷持久化数据

## 3. 数据库连接配置

### 3.1 app/db/base.py

SQLAlchemy 2.x `DeclarativeBase` 基类，定义统一的 `NAMING_CONVENTION` 字典，规范约束/索引命名：

```python
{
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}
```

### 3.2 app/db/session.py

- 使用 `create_async_engine` 创建异步引擎，配置连接池（pool_size, max_overflow, pool_timeout）
- `async_sessionmaker` 配置 `expire_on_commit=False`
- 提供三种会话获取方式：
  1. `get_db_session()` - FastAPI `Depends()` 用的异步生成器（自动 commit/rollback）
  2. `get_db_context()` - `async with` 上下文管理器（用于 WebSocket 等手动管理场景）
  3. `get_worker_db_context()` - 后台工作者专用，使用 `NullPool` 避免跨进程/事件循环问题
- `close_db()` 优雅关闭连接

### 3.3 app/db/models/mixins.py

- `TimestampMixin`：自动管理 `created_at`（server_default=func.now）和 `updated_at`（onupdate=func.now）

## 4. 路由管理方式

### 4.1 分层路由聚合

- `app/api/routers/v1/__init__.py`：注册各模块路由，统一设置 `prefix` 和 `tags`
- `app/api/router.py`：版本路由聚合（如 `v1_router`）
- `app/main.py`：通过 `settings.API_V1_STR`（如 `/api/v1`）挂载总路由

### 4.2 路由文件规范

- 每个模块独立文件（如 `auth.py`, `users.py`）
- 使用 `APIRouter()` 创建路由器
- 路由函数只负责：协议适配、依赖注入、参数校验、序列化
- **禁止在路由层直接操作数据库**，必须通过 Service 层

## 5. 中间件设置

### 5.1 CORS

- 在 `app/main.py` 中通过 `CORSMiddleware` 配置
- 配置项来自 `settings.CORS_ORIGINS` 等字段
- 生产环境校验：禁止 `*` 通配符

### 5.2 其他中间件

- 预留请求 ID 追踪中间件位置
- 预留请求耗时日志中间件位置

## 6. 异常处理机制

在 `app/main.py` 的 `create_app()` 中注册三个全局异常处理器：

1. **StarletteHTTPException**：统一包装为 `{"code": "ERR_HTTP", "message": "...", "details": {}}`
2. **RequestValidationError**（422）：统一包装为 `{"code": "ERR_INVALID_ARGUMENT", "message": "参数错误", "details": {"errors": [...]}}`
   - 需处理 `ctx` 中的非 JSON 序列化值（如使用 `_json_safe_validation_errors`）
3. **Exception**（500）：记录堆栈，返回 `{"code": "ERR_INTERNAL_SERVER_ERROR", "message": "服务器内部错误", "details": {}}`

## 7. 依赖注入模式

### 7.1 app/api/deps.py

- `get_db`：从 `app.db.session` 导入异步会话依赖
- `get_current_user`：JWT 校验 + 数据库查询当前用户
- `OAuth2PasswordBearer` 配置 `tokenUrl`
- 使用 `Annotated` 类型别名简化依赖声明（如 `CurrentUser = Annotated[User, Depends(get_current_user)]`）

### 7.2 Service 依赖

- 每个 Service 类接收 `AsyncSession` 作为构造函数参数
- 在 `deps.py` 中提供 `get_xxx_service` 工厂函数
- 示例：

```python
def get_user_service(db: DBSession) -> UserService:
    return UserService(db)

UserSvc = Annotated[UserService, Depends(get_user_service)]
```

## 8. API 文档配置

- FastAPI 初始化配置：`title`, `version`, `debug`, `lifespan`
- 自动生成 `/docs`（Swagger UI）和 `/redoc`
- 路由 tags 统一命名，便于文档分组

## 9. 测试文件组织

### 9.1 目录结构

- `tests/integration/`：API 端到端流程测试（使用 `httpx.AsyncClient` + `create_app()`）
- `tests/unit/`：纯函数、服务、校验器、解析器的单元测试

### 9.2 集成测试规范

- 使用 `pytest.mark.asyncio`
- 数据库可用性检查：连接测试失败时 `pytest.skip`
- 使用 Alembic `command.upgrade(alembic_cfg, "head")` 初始化测试数据库
- 测试数据清理：利用事务回滚或独立测试数据库

### 9.3 pytest 配置

- `asyncio_mode = "auto"`
- `addopts = "-p no:cacheprovider"`（避免缓存干扰）

## 10. 部署相关配置

### 10.1 启动命令

```bash
uv run uvicorn app.main:app --reload --port 8000
```

### 10.2 Lifespan 管理

- `lifespan` 异步上下文管理器处理启动/关闭
- 启动：创建必要的本地目录（storage、upload tmp 等）
- 关闭：关闭数据库连接、停止日志监听器

### 10.3 日志配置

- 统一通过 `app.core.logging_config.configure_logging(settings)` 初始化
- 支持通过环境变量 `LOG_CONFIG_FILE` 加载 JSON dictConfig
- 默认包含：控制台日志（开发彩色）、文件日志（按日轮转）、队列异步写入
- 敏感数据脱敏过滤器（自动处理 password/token/api_key 等）
- 统一格式：`timestamp level [logger] [pid=process tid=thread] message`

## 11. 代码风格与命名规范

### 11.1 通用规范

- 遵循 PEP 8，使用 `ruff` 或类似工具格式化
- 类型注解：全面使用 Python 3.11+ 语法（如 `str | None`, `list[str]`）
- 异步优先：数据库操作、HTTP 请求全部使用 `async/await`

### 11.2 命名规范

- 模块/包：全小写，下划线分隔（如 `logging_config.py`）
- 类名：PascalCase（如 `NotificationService`, `TimestampMixin`）
- 函数/变量：snake_case
- 常量：UPPER_SNAKE_CASE
- 配置字段：大写字母（如 `DATABASE_URL`），小写为兼容别名
- 环境变量：`{PREFIX}_UPPER_SNAKE_CASE`

### 11.3 导入规范

- 绝对导入优先：`from app.db.base import Base`
- 避免循环导入：模型间使用字符串引用或 `TYPE_CHECKING`

## 12. 分层架构最佳实践

严格遵循以下分层规则：

| 层级 | 职责 | 禁止 |
|------|------|------|
| `api/routers` | 协议适配、依赖注入、参数校验、序列化 | 直接数据库操作 |
| `services` | 业务编排、事务管理、跨模块协调 | 直接处理 HTTP 请求/响应 |
| `repositories` | 纯数据库访问、SQL 构建 | 业务逻辑判断 |
| `schemas` | Pydantic 模型定义、数据校验 | 数据库查询 |
| `db/models` | SQLAlchemy ORM 模型定义 | 业务逻辑 |

## 13. 安全最佳实践

- 密码使用 `bcrypt` 哈希，限制最大 72 UTF-8 字节
- JWT 使用 `python-jose`，生产环境强制校验密钥长度和默认值
- 日志默认脱敏敏感字段，禁止记录密码、验证码、Token
- CORS 生产环境禁止通配符
- API Key / Secret Key 生产环境强制非默认值校验

## 14. 扩展指南

新增 API Endpoint 的标准流程：

1. `app/schemas/{module}.py` - 定义请求/响应 Schema
2. `app/db/models/{module}.py` - 定义数据库模型（继承 `Base` + `TimestampMixin`）
3. `app/repositories/{module}.py` - 实现数据访问
4. `app/services/{module}_service.py` - 实现业务逻辑
5. `app/api/deps.py` - 注册 Service 依赖
6. `app/api/routers/v1/{module}.py` - 实现路由
7. `app/api/routers/v1/__init__.py` - 注册路由
8. `alembic revision --autogenerate -m "Add {module} table"` - 创建迁移

---

## 生成要求

请生成完整的项目代码，确保：

1. 所有文件可直接运行（`import app.main` 成功）
2. 包含一个示例模块（如 `users` 或 `items`）展示完整分层
3. 包含示例测试展示集成测试和单元测试写法
4. 代码注释使用中文，保持与现有项目一致
