[根目录](../CLAUDE.md) > **api**

# API Routes 模块

## 模块职责

API Routes模块作为Next.js API路由层，负责：
- **智能体代理**：为Python智能体提供Next.js API端点适配
- **协议转换**：桥接HTTP请求和Python智能体服务
- **本地开发支持**：支持Python智能体在独立进程中运行
- **部署适配**：适配Vercel等无服务器部署环境

## 入口与启动

### API路由结构
```
api/
├── orchestrator.py        # 编排器API代理 (端口9000)
├── itinerary_agent.py     # 行程智能体API代理 (端口9001)
├── budget_agent.py        # 预算智能体API代理 (端口9002)
├── restaurant_agent.py    # 餐厅智能体API代理 (端口9003)
└── weather_agent.py       # 天气智能体API代理 (端口9005)
```

### 启动方式
API路由文件本身不直接启动，而是：
1. **Next.js开发服务器**: `npm run dev:ui` 启动时自动加载
2. **Python智能体**: 通过 `npm run dev` 并行启动
3. **A2A中间件**: 在 `/api/copilotkit/route.ts` 中配置引用

## 对外接口

### HTTP API接口
所有API路由都实现标准的HTTP接口：

```python
# 统一的接口模式
@app.post("/")
async def handle_request(request: Request):
    # 转发请求到Python智能体
    # 返回智能体响应
    pass

@app.get("/agent/")
async def get_agent_info():
    # 返回智能体元信息
    pass
```

### 智能体端点配置
```typescript
// 在 CopilotKit 路由中配置
const itineraryAgentUrl = process.env.ITINERARY_AGENT_URL || `${base}/api/itinerary`;
const restaurantAgentUrl = process.env.RESTAURANT_AGENT_URL || `${base}/api/restaurant`;
const budgetAgentUrl = process.env.BUDGET_AGENT_URL || `${base}/api/budget`;
const weatherAgentUrl = process.env.WEATHER_AGENT_URL || `${base}/api/weather`;
const orchestratorUrl = process.env.ORCHESTRATOR_URL || `${base}/api/orchestrator`;
```

## 智能体代理详解

### 1. 编排器代理 (orchestrator.py)
```python
from agents.orchestrator import app

# 直接导入Python智能体应用
# Next.js API路由会自动处理HTTP到Python的桥接
```

**职责**:
- 桥接Next.js HTTP请求和Python FastAPI应用
- 保持会话状态和用户上下文
- 处理AG-UI协议消息

### 2. 行程智能体代理 (itinerary_agent.py)
```python
from agents.itinerary_agent import app
```

**职责**:
- 代理LangGraph行程智能体
- 处理A2A协议消息
- 返回结构化JSON行程数据

### 3. 预算智能体代理 (budget_agent.py)
```python
from agents.budget_agent import app
```

**职责**:
- 代理ADK预算智能体
- 处理预算估算请求
- 返回详细预算分析

### 4. 餐厅智能体代理 (restaurant_agent.py)
```python
from agents.restaurant_agent import app
```

**职责**:
- 代理LangGraph餐厅智能体
- 处理餐饮推荐请求
- 返回日式餐厅建议

### 5. 天气智能体代理 (weather_agent.py)
```python
from agents.weather_agent import app
```

**职责**:
- 代理ADK天气智能体
- 处理天气预报请求
- 返回旅行建议

## 协议支持

### A2A协议实现
API路由完全支持A2A协议标准：

```python
# A2A协议消息结构
{
    "id": "unique_message_id",
    "timestamp": "2025-11-15T00:42:19Z",
    "agent": "agent_name",
    "task": "具体任务描述",
    "parameters": {
        "city": "目的地",
        "days": 5,
        # ... 其他参数
    }
}
```

### HTTP方法支持
- **POST /** - 主要的消息处理端点
- **GET /agent/** - 智能体信息查询
- **GET /tasks/** - 任务状态查询 (可选)

### 响应格式
```python
# 成功响应
{
    "success": true,
    "data": {
        # 智能体返回的特定数据
    },
    "agent": "agent_name",
    "timestamp": "2025-11-15T00:42:19Z"
}

# 错误响应
{
    "success": false,
    "error": "错误描述",
    "timestamp": "2025-11-15T00:42:19Z"
}
```

## 部署配置

### 开发环境
```bash
# 本地开发时，Python智能体运行在独立进程
npm run dev  # 启动所有服务
```

端口分配：
- 编排器: http://localhost:9000
- 行程智能体: http://localhost:9001
- 预算智能体: http://localhost:9002
- 餐厅智能体: http://localhost:9003
- 天气智能体: http://localhost:9005

### 生产环境 (Vercel)
```javascript
// vercel.json
{
  "functions": {
    "api/orchestrator.py": {
      "maxDuration": 30
    },
    "api/itinerary_agent.py": {
      "maxDuration": 30
    },
    "api/budget_agent.py": {
      "maxDuration": 30
    },
    "api/restaurant_agent.py": {
      "maxDuration": 30
    },
    "api/weather_agent.py": {
      "maxDuration": 30
    }
  }
}
```

### 环境变量
```bash
# API端点配置
ORCHESTRATOR_URL=/api/orchestrator
ITINERARY_AGENT_URL=/api/itinerary
BUDGET_AGENT_URL=/api/budget
RESTAURANT_AGENT_URL=/api/restaurant
WEATHER_AGENT_URL=/api/weather

# 外部服务URL (可选)
EXTERNAL_ORCHESTRATOR_URL=https://your-service.com/orchestrator
```

## 性能考虑

### 请求处理
- **异步处理**: 所有API路由使用async/await
- **连接池**: 复用Python智能体连接
- **超时控制**: 设置合理的请求超时时间
- **错误重试**: 实现智能重试机制

### 内存管理
- **会话存储**: 使用内存存储会话状态
- **响应缓冲**: 控制响应大小
- **垃圾回收**: 及时清理临时数据

### 监控指标
- **响应时间**: 监控API响应延迟
- **错误率**: 跟踪失败请求比例
- **并发数**: 监控同时处理的请求数
- **资源使用**: CPU和内存使用情况

## 安全考虑

### 输入验证
```python
from pydantic import BaseModel, validator

class AgentRequest(BaseModel):
    agent: str
    task: str
    parameters: dict

    @validator('agent')
    def validate_agent(cls, v):
        allowed_agents = ['itinerary_agent', 'budget_agent', 'restaurant_agent', 'weather_agent']
        if v not in allowed_agents:
            raise ValueError(f'Agent {v} not allowed')
        return v
```

### 错误处理
```python
from fastapi import HTTPException
import logging

logger = logging.getLogger(__name__)

@app.post("/")
async def handle_request(request: Request):
    try:
        # 处理请求
        result = await process_agent_request(request)
        return result
    except Exception as e:
        logger.error(f"Agent request failed: {str(e)}")
        raise HTTPException(status_code=500, detail="Internal server error")
```

### 访问控制
- **CORS配置**: 限制跨域访问
- **速率限制**: 防止API滥用
- **认证**: 可选的API密钥认证

## 测试策略

### 单元测试
```python
import pytest
from fastapi.testclient import TestClient
from api.itinerary_agent import app

client = TestClient(app)

def test_itinerary_agent():
    response = client.post("/", json={
        "agent": "itinerary_agent",
        "task": "Create a 3-day itinerary for Paris",
        "parameters": {"city": "Paris", "days": 3}
    })
    assert response.status_code == 200
    assert "data" in response.json()
```

### 集成测试
- 测试API路由与Python智能体的集成
- 验证A2A协议消息传递
- 测试错误处理和恢复机制

### 负载测试
- 并发请求处理能力
- 内存和CPU使用效率
- 长时间运行稳定性

## 故障排除

### 常见问题

#### 1. 智能体连接失败
**症状**: API返回502错误
**解决方案**:
- 检查Python智能体是否正在运行
- 验证端口配置是否正确
- 检查防火墙设置

#### 2. 响应超时
**症状**: API请求超时
**解决方案**:
- 增加超时时间配置
- 优化智能体响应速度
- 检查网络连接

#### 3. 内存泄漏
**症状**: 内存使用持续增长
**解决方案**:
- 检查会话存储清理
- 优化数据结构使用
- 重启服务释放内存

### 调试工具
```python
import logging
import sys

# 配置日志
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/tmp/api.log'),
        logging.StreamHandler(sys.stdout)
    ]
)
```

## 监控和日志

### 日志配置
```python
import structlog

logger = structlog.get_logger()

@app.post("/")
async def handle_request(request: Request):
    logger.info("Processing agent request", agent=request.json().get("agent"))
    # 处理请求
    logger.info("Request completed successfully")
```

### 健康检查
```python
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": "2025-11-15T00:42:19Z",
        "version": "1.0.0"
    }
```

### 指标收集
- 请求计数和响应时间
- 错误率和异常统计
- 系统资源使用情况

## 最佳实践

### 代码组织
- 保持API路由简洁明了
- 使用类型提示提高代码质量
- 实现全面的错误处理
- 添加详细的日志记录

### 性能优化
- 使用异步编程模式
- 实现连接池和缓存
- 优化数据库查询
- 监控和调优性能瓶颈

### 安全措施
- 验证所有输入数据
- 实现适当的认证机制
- 使用HTTPS加密传输
- 定期更新依赖包

## 相关文件清单

### API路由文件
- `orchestrator.py` - 编排器API代理
- `itinerary_agent.py` - 行程智能体API代理
- `budget_agent.py` - 预算智能体API代理
- `restaurant_agent.py` - 餐厅智能体API代理
- `weather_agent.py` - 天气智能体API代理

### 相关配置
- `../app/api/copilotkit/route.ts` - A2A中间件配置
- `../vercel.json` - Vercel部署配置
- `../package.json` - 构建和启动脚本

### Python智能体引用
- `../agents/orchestrator.py` - 编排器实现
- `../agents/itinerary_agent.py` - 行程智能体实现
- `../agents/budget_agent.py` - 预算智能体实现
- `../agents/restaurant_agent.py` - 餐厅智能体实现
- `../agents/weather_agent.py` - 天气智能体实现

## 变更记录 (Changelog)

### 2025-11-15 00:42:19
- 创建API Routes模块文档
- 分析Next.js API路由架构
- 记录Python智能体代理模式
- 识别性能和安全改进点

---

*此文档由AI自动生成，如有不准确之处请手动修正*