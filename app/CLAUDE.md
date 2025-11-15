[根目录](../CLAUDE.md) > **app**

# Next.js Frontend 模块

## 模块职责

Next.js Frontend模块是项目的用户界面层，负责：
- **用户交互界面**：提供现代化的旅行规划聊天界面
- **A2A中间件集成**：通过CopilotKit集成A2A协议通信
- **实时数据展示**：动态显示智能体生成的行程、预算、天气等信息
- **人机交互循环**：实现需求收集和预算审批等HITL工作流

## 入口与启动

### 主要入口文件
- **layout.tsx** - 根布局组件，配置主题和全局样式
- **page.tsx** - 主页面组件，展示旅行规划界面
- **globals.css** - 全局样式定义

### A2A中间件入口
- **api/copilotkit/route.ts** - CopilotKit API路由，配置A2A中间件

### 启动配置
```typescript
// next.config.ts
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    turbo: {
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js',
        },
      },
    },
  },
};
```

## 对外接口

### CopilotKit API路由
**路径**: `/api/copilotkit`
**方法**: POST
**功能**: 连接前端UI与后端智能体，处理A2A协议转换

主要功能：
1. **智能体URL配置** - 动态配置各智能体端点
2. **A2A中间件设置** - 桥接AG-UI和A2A协议
3. **工作流指令** - 定义编排器工作流程
4. **会话管理** - 管理用户会话状态

### 页面组件接口
```typescript
interface HomePageProps {
  // 无props，使用内部状态管理
}

// 状态更新回调
onItineraryUpdate: (data: ItineraryData | null) => void
onBudgetUpdate: (data: BudgetData | null) => void
onWeatherUpdate: (data: WeatherData | null) => void
onRestaurantUpdate: (data: RestaurantData | null) => void
```

## 关键依赖与配置

### 核心依赖
```json
{
  "dependencies": {
    "@a2a-js/sdk": "latest",              // A2A协议SDK
    "@ag-ui/a2a-middleware": "0.0.2",     // A2A中间件
    "@ag-ui/client": "0.0.40",            // AG-UI客户端
    "@copilotkit/react-core": "latest",   // CopilotKit核心
    "@copilotkit/react-ui": "latest",     // CopilotKit UI组件
    "@copilotkit/runtime": "latest",      // CopilotKit运行时
    "next": "15.1.6",                     // Next.js框架
    "react": "^19.0.0",                   // React核心
    "tailwindcss": "^3.4.1"               // Tailwind CSS
  }
}
```

### TypeScript配置
```json
{
  "compilerOptions": {
    "target": "ES2017",
    "strict": true,
    "jsx": "preserve",
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

### Tailwind CSS配置
```typescript
// tailwind.config.ts
export default {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      fontFamily: {
        sans: ['var(--font-plus-jakarta-sans)'],
        mono: ['var(--font-spline-sans-mono)'],
      },
    },
  },
  plugins: [require('@tailwindcss/typography')],
}
```

## 数据模型

### 页面状态管理
```typescript
interface HomePageState {
  itineraryData: ItineraryData | null;
  budgetData: BudgetData | null;
  weatherData: WeatherData | null;
  restaurantData: RestaurantData | null;
}
```

### A2A中间件配置
```typescript
interface A2AMiddlewareConfig {
  description: string;
  agentUrls: string[];           // 智能体URL列表
  orchestrationAgent: HttpAgent; // 编排器智能体
  instructions: string;          // 工作流指令
}
```

### 环境变量配置
```bash
# 智能体端点配置
ORCHESTRATOR_URL=http://localhost:9000
ITINERARY_AGENT_URL=http://localhost:9001
BUDGET_AGENT_URL=http://localhost:9002
RESTAURANT_AGENT_URL=http://localhost:9003
WEATHER_AGENT_URL=http://localhost:9005

# Vercel部署配置
VERCEL_URL=your-app.vercel.app
SELF_BASE_URL=https://your-app.vercel.app
```

## UI架构

### 布局结构
```
主页面 (page.tsx)
├── 背景装饰 (圆形渐变元素)
├── 左侧聊天区域 (450px)
│   ├── 标题栏
│   └── TravelChat组件
└── 右侧展示区域 (flex-1)
    ├── 计划标题
    ├── 空状态提示
    ├── ItineraryCard (行程卡片)
    └── 网格布局
        ├── WeatherCard (天气卡片)
        └── BudgetBreakdown (预算明细)
```

### 设计系统
- **字体**: Plus Jakarta Sans (正文字体), Spline Sans Mono (等宽字体)
- **颜色**:
  - 主色: #DEDEE9 (背景), #010507 (主文字)
  - 强调色: #1B936F (成功), #BEC2FF (信息)
- **圆角**: 统一使用 `rounded-lg` (8px)
- **阴影**: 自定义 `shadow-elevation-*` 系列

## A2A中间件实现

### 中间件工作流
```typescript
const workflowInstructions = `
  1. 收集旅行需求 (gather_trip_requirements)
  2. 创建行程 (Itinerary Agent)
  3. 查询天气 (Weather Agent)
  4. 推荐餐厅 (Restaurant Agent)
  5. 预算估算 (Budget Agent)
  6. 预算审批 (request_budget_approval)
  7. 展示完整计划
`;
```

### 智能体注册
```typescript
const agentUrls = [
  itineraryAgentUrl,    // LangGraph + OpenAI
  restaurantAgentUrl,   // LangGraph + OpenAI
  budgetAgentUrl,       // ADK + Gemini
  weatherAgentUrl,      // ADK + Gemini
];
```

### 协议桥接
- **AG-UI协议**: CopilotKit ↔ 编排器
- **A2A协议**: 编排器 ↔ 专业化智能体
- **中间件职责**: 注入通信工具，路由消息，转换协议

## 性能优化

### Next.js优化
- **增量构建**: 使用 `incremental: true`
- **Turbopack**: 开发环境启用 `npm run dev:ui`
- **代码分割**: 自动按路由分割代码
- **图片优化**: Next.js Image组件

### 渲染优化
- **客户端组件**: 使用 `"use client"` 标记
- **React 19**: 利用最新的并发特性
- **CSS优化**: Tailwind CSS按需生成
- **字体优化**: 使用 `next/font` 优化字体加载

## 测试与质量

### 当前状态
- **测试覆盖率**: 0% (缺少自动化测试)
- **TypeScript**: 严格模式，完整类型检查
- **ESLint**: Next.js预设规则
- **代码质量**: 函数式组件，现代React模式

### 建议改进
- 添加Jest + Testing Library测试
- 实现E2E测试 (Playwright/Cypress)
- 添加性能监控和分析
- 完善错误边界和异常处理

## 部署配置

### Vercel配置
```json
// vercel.json
{
  "functions": {
    "app/api/copilotkit/route.ts": {
      "maxDuration": 30
    }
  }
}
```

### 环境要求
- **Node.js**: 18+ (当前使用18.17.0)
- **Platform**: Vercel (推荐) 或自托管Node.js环境
- **域名**: 支持自定义域名配置

## 常见问题 (FAQ)

### Q: CopilotKit连接失败怎么办？
A: 检查以下几点：
1. 所有智能体服务是否正常运行
2. API密钥是否正确配置
3. 网络连接是否正常
4. Vercel环境变量是否设置

### Q: 如何调试A2A通信？
A: 使用浏览器开发者工具：
1. 查看Network标签页的API请求
2. 检查Console日志输出
3. 验证智能体响应格式
4. 确认数据解析是否正确

### Q: 如何自定义UI样式？
A: 修改以下文件：
1. `app/globals.css` - 全局样式
2. `tailwind.config.ts` - Tailwind配置
3. 组件内部样式 - 使用Tailwind类名

## 相关文件清单

### 核心应用文件
- `layout.tsx` - 根布局，主题配置
- `page.tsx` - 主页面，UI布局
- `globals.css` - 全局CSS样式

### API路由
- `api/copilotkit/route.ts` - A2A中间件配置

### 配置文件
- `next-env.d.ts` - Next.js类型定义
- `next.config.ts` - Next.js配置
- `tsconfig.json` - TypeScript配置
- `tailwind.config.ts` - Tailwind CSS配置
- `postcss.config.mjs` - PostCSS配置

### 样式资源
- `styles/typography.css` - 排版样式

## 变更记录 (Changelog)

### 2025-11-15 00:42:19
- 创建app模块文档
- 分析Next.js架构和A2A集成
- 记录UI组件结构和设计系统
- 识别性能优化机会

---

*此文档由AI自动生成，如有不准确之处请手动修正*