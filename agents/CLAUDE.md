[根目录](../CLAUDE.md) > **agents**

# Python Agents 模块

## 模块职责

Python Agents模块是项目的核心后端组件，实现了多智能体协作的旅行规划系统。该模块包含：
- **编排器智能体**：使用Google ADK框架，协调所有专业化智能体
- **专业化智能体**：4个不同框架的智能体，各自负责特定的旅行规划任务
- **A2A协议支持**：实现Agent-to-Agent通信协议

## 入口与启动

### 编排器 (Orchestrator)
- **入口文件**: `orchestrator.py`
- **框架**: Google ADK + Gemini 2.5 Pro
- **端口**: 9000
- **启动命令**: `python agents/orchestrator.py`

### 专业化智能体
每个智能体都通过FastAPI暴露HTTP端点，支持A2A协议通信：

| 智能体 | 文件名 | 框架 | LLM | 端口 | 启动命令 |
|--------|--------|------|-----|------|----------|
| 行程规划 | `itinerary_agent.py` | LangGraph | OpenAI GPT | 9001 | `python itinerary_agent.py` |
| 预算估算 | `budget_agent.py` | Google ADK | Gemini | 9002 | `python budget_agent.py` |
| 餐厅推荐 | `restaurant_agent.py` | LangGraph | OpenAI GPT | 9003 | `python restaurant_agent.py` |
| 天气预报 | `weather_agent.py` | Google ADK | Gemini | 9005 | `python weather_agent.py` |

## 对外接口

### A2A协议接口
所有智能体都实现A2A协议标准接口：
- **POST /** - 主消息处理端点
- **GET /agent/** - 智能体信息查询
- **GET /tasks/** - 任务状态查询

### 消息格式
```python
# 智能体间通信的标准消息格式
{
    "id": "message_id",
    "timestamp": "2025-11-15T00:42:19Z",
    "agent": "agent_name",
    "task": "具体任务描述",
    "parameters": {...}
}
```

### 编排器接口
编排器使用AG-UI协议与前端通信：
- **ADKAgent包装器**: 将ADK智能体包装为AG-UI兼容的接口
- **会话管理**: 支持用户会话和状态持久化
- **工具注入**: A2A中间件自动注入通信工具

## 关键依赖与配置

### 核心依赖 (requirements.txt)
```python
# AG-UI和A2A协议
ag-ui-adk>=0.0.1          # AG-UI协议ADK适配器
a2a>=0.1.0                # A2A协议SDK
a2a-sdk[http-server]      # A2A HTTP服务器支持

# 智能体框架
google-adk>=0.1.0         # Google ADK框架
langgraph>=0.2.0          # LangGraph工作流框架
langchain>=0.3.0          # LangChain核心
langchain-openai>=0.2.0   # OpenAI集成

# Web服务
fastapi>=0.115.0          # API框架
uvicorn>=0.30.0           # ASGI服务器
python-dotenv>=1.0.0      # 环境变量管理
```

### 环境变量配置
```bash
# 必需的API密钥
GOOGLE_API_KEY=your_google_api_key    # Google Gemini API
OPENAI_API_KEY=your_openai_api_key    # OpenAI GPT API

# 智能体端口配置
ORCHESTRATOR_PORT=9000
ITINERARY_PORT=9001
BUDGET_PORT=9002
RESTAURANT_PORT=9003
WEATHER_PORT=9005
```

### 智能体配置
- **编排器**: 使用内存服务 (`use_in_memory_services=True`) 适配Vercel部署
- **超时设置**: 会话超时3600秒
- **并发控制**: 编排器一次只调用一个智能体，避免并发问题

## 数据模型

### 行程数据结构
```python
class DayItinerary(BaseModel):
    day: int                           # 天数
    title: str                         # 当天主题
    morning: TimeSlot                  # 上午活动
    afternoon: TimeSlot                # 下午活动
    evening: TimeSlot                  # 晚上活动
    meals: Meals                       # 餐饮推荐

class TimeSlot(BaseModel):
    activities: List[str]              # 活动列表
    location: str                      # 主要地点

class Meals(BaseModel):
    breakfast: str                     # 早餐推荐
    lunch: str                         # 午餐推荐
    dinner: str                        # 晚餐推荐
```

### 预算数据结构
```python
class BudgetCategory(BaseModel):
    category: str                      # 预算类别
    amount: number                     # 金额
    percentage: number                 # 占总预算百分比

class BudgetData(BaseModel):
    totalBudget: number                # 总预算
    currency: str                      # 货币类型
    breakdown: BudgetCategory[]        # 分类明细
    notes: str                         # 备注说明
```

### 天气数据结构
```python
class DailyWeather(BaseModel):
    day: int                           # 天数
    date: str                          # 日期
    condition: str                     # 天气状况
    highTemp: number                   # 最高温度
    lowTemp: number                    # 最低温度
    precipitation: number              # 降水量
    humidity: number                   # 湿度
    windSpeed: number                  # 风速
    description: str                   # 天气描述
```

## 测试与质量

### 当前状态
- **测试覆盖率**: 0% (缺少自动化测试)
- **代码质量**: 遵循PEP 8规范
- **类型检查**: 使用Python类型提示
- **错误处理**: 基本的异常处理机制

### 建议改进
- 添加单元测试覆盖智能体核心逻辑
- 实现集成测试验证A2A通信
- 添加性能监控和日志记录
- 完善错误处理和重试机制

## 工作流程

### 1. 编排器工作流
```
用户请求 → 收集需求 → 行程规划 → 天气查询 → 餐厅推荐 → 预算估算 → 人工审批 → 完整计划
```

### 2. 智能体调用顺序
1. **gather_trip_requirements** - 收集旅行基本信息
2. **Itinerary Agent** - 创建基础行程
3. **Weather Agent** - 获取天气信息
4. **Restaurant Agent** - 推荐餐厅
5. **Budget Agent** - 预算估算
6. **request_budget_approval** - 预算审批

### 3. A2A通信流程
```python
# 编排器调用智能体的标准模式
send_message_to_a2a_agent(
    agent_name="itinerary_agent",
    task=f"Create a {days}-day itinerary for {city}"
)
```

## 常见问题 (FAQ)

### Q: 智能体无法启动怎么办？
A: 检查以下几点：
1. Python虚拟环境是否正确激活
2. requirements.txt依赖是否完整安装
3. API密钥是否正确配置
4. 端口是否被占用

### Q: A2A通信失败如何调试？
A: 查看智能体日志，确认：
1. 所有智能体服务都在运行
2. A2A协议端点可访问
3. 消息格式符合标准
4. 网络连接正常

### Q: 如何添加新的智能体？
A: 按以下步骤：
1. 创建新的智能体文件
2. 实现A2A协议接口
3. 在编排器中注册新智能体
4. 更新前端组件支持新功能

## 相关文件清单

### 核心智能体文件
- `orchestrator.py` - 主编排器 (ADK + Gemini)
- `itinerary_agent.py` - 行程规划智能体 (LangGraph + OpenAI)
- `budget_agent.py` - 预算估算智能体 (ADK + Gemini)
- `restaurant_agent.py` - 餐厅推荐智能体 (LangGraph + OpenAI)
- `weather_agent.py` - 天气预报智能体 (ADK + Gemini)

### 配置和依赖
- `requirements.txt` - Python依赖包列表
- `__init__.py` - Python模块初始化

### API适配器
- `../api/orchestrator.py` - 编排器API适配器
- `../api/itinerary_agent.py` - 行程智能体API适配器
- `../api/budget_agent.py` - 预算智能体API适配器
- `../api/restaurant_agent.py` - 餐厅智能体API适配器
- `../api/weather_agent.py` - 天气智能体API适配器

## 变更记录 (Changelog)

### 2025-11-15 00:42:19
- 创建agents模块文档
- 分析智能体架构和工作流程
- 记录A2A协议实现细节
- 识别测试覆盖缺口

---

*此文档由AI自动生成，如有不准确之处请手动修正*