# HelloAgents 智能旅行助手 — 项目学习文档

> 目标：读完这份文档 + 对照源码走一遍，你能**彻底讲清楚**这个项目的架构、每一行核心代码的作用、用到的技术原理，以及如何回答面试官的追问。

---

## 目录

1. [项目是什么](#一项目是什么)
2. [整体架构（最重要）](#二整体架构最重要)
3. [目录结构逐层解读](#三目录结构逐层解读)
4. [核心概念铺垫：MCP 与 Agent](#四核心概念铺垫mcp-与-agent)
5. [一次请求的完整生命周期（主干）](#五一次请求的完整生命周期主干)
6. [后端逐文件精讲](#六后端逐文件精讲)
7. [前端逐文件精讲](#七前端逐文件精讲)
8. [项目中用到的设计模式与技巧](#八项目中用到的设计模式与技巧)
9. [环境搭建与运行](#九环境搭建与运行)
10. [项目的已知问题与改进方向（面试加分点）](#十项目的已知问题与改进方向面试加分点)
11. [7天学习路线](#十一7天学习路线)
12. [高频面试题与参考答案](#十二高频面试题与参考答案)

---

## 一、项目是什么

一个 **AI 旅行规划助手**：用户在网页上填"目的地 + 日期 + 偏好"，后端用 **多个 LLM Agent 协作**（搜景点、查天气、找酒店、排行程），通过 **MCP 协议**调用**高德地图**的真实数据，最终生成一份包含景点、酒店、餐饮、天气、预算的完整行程单，前端渲染成可视化页面，支持**编辑、导出图片/PDF**。

一句话技术概括：

> **FastAPI + Vue3 前后端分离；后端基于 HelloAgents 框架实现 4 个 Agent 的流水线协作；Agent 通过 MCP 协议调用高德地图 MCP Server 获取真实 POI/天气/路线数据；最终结果以 Pydantic 模型校验后返回前端渲染。**

技术栈清单：

| 层 | 技术 |
|---|---|
| 后端框架 | FastAPI + Uvicorn |
| Agent 框架 | HelloAgents（SimpleAgent、HelloAgentsLLM、MCPTool） |
| 工具协议 | MCP（Model Context Protocol），amap-mcp-server |
| 数据校验 | Pydantic v2 + pydantic-settings |
| 前端框架 | Vue 3（`<script setup>` 组合式 API）+ TypeScript |
| UI 组件 | Ant Design Vue 4 |
| 地图 | 高德地图 JavaScript API 2.0 |
| 构建 | Vite 6 |
| HTTP | Axios（前端）/ httpx、requests（后端） |
| 导出 | html2canvas + jsPDF |

---

## 二、整体架构（最重要）

### 2.1 架构图

```
┌─────────────────────────── 浏览器 ───────────────────────────┐
│  Vue3 前端 (localhost:5173)                                  │
│  Home.vue(表单) ──Axios POST /api/trip/plan──▶               │
│  Result.vue(渲染/编辑/导出 + 高德JS地图)                       │
└──────────────────────────────┬───────────────────────────────┘
                               │ HTTP (JSON)
┌──────────────────────────────▼───────────────────────────────┐
│  FastAPI 后端 (localhost:8000)                                │
│  api/main.py ── CORS中间件 ── 3个路由模块                      │
│   ├─ routes/trip.py   POST /api/trip/plan  ★核心入口          │
│   ├─ routes/poi.py    GET  /api/poi/photo   (Unsplash图片)    │
│   └─ routes/map.py    GET  /api/map/*       (备用地图服务)     │
│                                                               │
│  agents/trip_planner_agent.py  ★★ 多智能体编排                 │
│   ┌────────────────────────────────────────────┐             │
│   │ ① 景点Agent ──┐                            │             │
│   │ ② 天气Agent ──┤ 都挂同一个 MCPTool          │             │
│   │ ③ 酒店Agent ──┘                            │             │
│   │ ④ 规划Agent(纯LLM,无工具) ← 汇总①②③的输出   │             │
│   └────────────────────────────────────────────┘             │
│  services/llm_service.py      → HelloAgentsLLM(单例)          │
│  services/amap_service.py     → AmapService(备用,大部分TODO)   │
│  services/unsplash_service.py → UnsplashService(景点配图)      │
└──────────┬─────────────────────────────┬─────────────────────┘
           │ MCP协议(stdio)              │ HTTPS
┌──────────▼──────────────┐   ┌──────────▼──────────┐
│ amap-mcp-server          │   │ LLM API (DeepSeek/  │
│ (uvx 启动的子进程)         │   │ OpenAI等)           │
│ 提供 maps_text_search、   │   └─────────────────────┘
│ maps_weather、maps_geo、  │
│ maps_direction_* 等工具   │
└──────────┬───────────────┘
           │ HTTPS
   ┌───────▼────────┐
   │ 高德开放平台API  │
   └────────────────┘
```

### 2.2 三条关键理解（吃透架构就看这三条）

1. **LLM 本身不会"上网"**。模型只能生成文本，所有"查景点、查天气"的真实动作，都是 Agent 框架解析 LLM 输出里的**工具调用指令**后，代替模型去调外部 API，再把结果喂回给模型 —— 这个循环就是 Agent 的本质（ReAct 模式）。
2. **MCP 是"工具的 USB 接口"**。高德把它的能力封装成一个标准 MCP Server，任何 MCP 客户端（本项目里是 HelloAgents 的 `MCPTool`）都能发现并调用它，不需要自己写高德 HTTP 封装。
3. **本项目是"流水线多智能体"，不是"对话式多智能体"**。4 个 Agent 不是互相聊天，而是后端 Python 代码**串行调度**：先让①跑完拿到文本，再跑②，再跑③，最后把三段文本拼进④的 prompt 里生成最终 JSON。编排逻辑是硬编码的（见 `plan_trip` 方法），这是最朴素也最容易讲清楚的多智能体形态。

---

## 三、目录结构逐层解读

```
Travel Agent/
├── README.md                    # 项目说明
├── LEARNING_GUIDE.md            # 本文档
├── _tmp_hello_agents/           # 临时目录(可忽略)
│
├── backend/                     # ========== 后端 ==========
│   ├── run.py                   # 启动脚本: 读配置→uvicorn起服务
│   ├── requirements.txt         # 依赖: hello-agents, fastapi, uvicorn...
│   ├── .env                     # 密钥配置(不进git)
│   ├── .env.example             # 配置模板
│   └── app/
│       ├── config.py            # 配置中心: pydantic-settings 读 .env
│       ├── api/
│       │   ├── main.py          # FastAPI应用工厂: CORS+注册路由+启动事件
│       │   └── routes/
│       │       ├── trip.py      # POST /api/trip/plan  ★核心API
│       │       ├── poi.py       # POI详情/搜索/Unsplash配图
│       │       └── map.py       # 天气/路线/POI(基于AmapService)
│       ├── agents/
│       │   └── trip_planner_agent.py  # ★★多智能体系统(全项目灵魂)
│       ├── services/
│       │   ├── llm_service.py   # HelloAgentsLLM单例
│       │   ├── amap_service.py  # 高德MCP工具封装(部分TODO未完成)
│       │   └── unsplash_service.py  # Unsplash图片搜索
│       └── models/
│           └── schemas.py       # 全部Pydantic模型(请求/响应/领域对象)
│
└── frontend/                    # ========== 前端 ==========
    ├── index.html               # SPA入口
    ├── vite.config.ts           # Vite配置: @别名 + /api代理
    ├── package.json
    └── src/
        ├── main.ts              # 应用入口: 创建Vue实例+路由+Antd
        ├── App.vue              # 布局骨架(Header/Footer/router-view)
        ├── router 在 main.ts 内  # 两个页面: / (Home) 和 /result (Result)
        ├── types/index.ts       # TS类型定义(与后端schemas一一对应)
        ├── services/api.ts      # Axios封装: 拦截器+generateTripPlan()
        └── views/
            ├── Home.vue         # 表单页: 目的地/日期/偏好/额外要求
            └── Result.vue       # 结果页: 概览/预算/地图/每日行程/天气/导出
```

**记忆口诀**：后端三层 —— `api` 管 HTTP 进出，`agents` 管智能编排，`services` 管外部对接；`models` 是三者的公共语言。

---

## 四、核心概念铺垫：MCP 与 Agent

### 4.1 MCP（Model Context Protocol）

MCP 是 Anthropic 提出的开放协议，标准化"LLM 应用 ↔ 外部工具/数据源"的连接方式。三个角色：

| 角色 | 本项目对应 | 说明 |
|---|---|---|
| MCP Host | FastAPI 后端 | 发起调用的 LLM 应用 |
| MCP Client | HelloAgents 的 `MCPTool` | 管理与 Server 的连接 |
| MCP Server | `amap-mcp-server` | 高德地图能力的标准封装 |

本项目用的是 **stdio 传输**：`MCPTool` 通过 `server_command=["uvx", "--with", "pydantic<2.11", "amap-mcp-server"]` **拉起一个子进程**，通过标准输入/输出与其通信。`uvx` 是 uv 的临时环境运行器，会自动安装并运行该包（`--with pydantic<2.11` 是为了规避 pydantic 2.11 与该 server 的兼容问题）。

`MCPTool(auto_expand=True)` 的含义：把 MCP Server 暴露的所有工具**自动展开为独立工具**注册给 Agent，命名格式 `amap_工具名`，如：
- `amap_maps_text_search` — 关键词搜索 POI
- `amap_maps_weather` — 查城市天气
- `amap_maps_geo` — 地址→经纬度
- `amap_maps_direction_walking_by_address` / `driving` / `transit_integrated` — 步行/驾车/公交路线
- `amap_maps_search_detail` — POI 详情

### 4.2 Agent 的工作循环（SimpleAgent）

```
用户输入
   │
   ▼
LLM(系统提示词 + 输入 + 可用工具列表)
   │
   ├─ 输出是普通文本 ──▶ 直接作为最终答案返回
   │
   └─ 输出含工具调用指令 ──▶ 框架解析指令 → 真正调用工具
                              → 把工具结果拼回对话 → 再问 LLM
                              （循环，直到 LLM 给出最终文本）
```

本项目的系统提示词里用 **自定义文本格式** 约定工具调用：

```
[TOOL_CALL:amap_maps_text_search:keywords=历史文化,city=北京]
```

框架检测到这种 `[TOOL_CALL:工具名:参数]` 格式就执行真实调用。**关键技巧**：`_build_attraction_query` 干脆在发给 Agent 的问题里**预置好这条指令**，等于"替模型写好第一步"，大幅降低模型格式出错率 —— 这是很实用的 Prompt 工程手段。

### 4.3 为什么拆成 4 个 Agent 而不是 1 个？

- **提示词聚焦**：每个 Agent 的 system prompt 只干一件事，输出格式更稳定；
- **职责解耦**：搜景点的失败不会污染天气查询的上下文；
- **可控编排**：Python 代码控制调用顺序，而非依赖模型自主决策，结果可复现、易调试。

---

## 五、一次请求的完整生命周期（主干）

这是整个项目的"故事主线"，务必能脱稿讲出：

```
① 用户在 Home.vue 填表单 → watch 自动算天数(结束-开始+1, 上限30)
② 点提交 → handleSubmit() → 显示模拟进度条(每0.5s+10%)
③ Axios POST /api/trip/plan (TripRequest JSON)
④ FastAPI 路由 trip.py 接收 → Pydantic 自动校验字段
⑤ get_trip_planner_agent() 单例获取 MultiAgentTripPlanner
   (首次会: 初始化LLM + 启动amap MCP子进程 + 创建4个Agent)
⑥ plan_trip() 四步流水线:
   步骤1 attraction_agent.run("搜索XX的YY景点 [TOOL_CALL:...]")
        → LLM 输出 TOOL_CALL → MCPTool 经 MCP 调高德 → 结果回喂 LLM
        → LLM 输出景点文本
   步骤2 weather_agent.run("查询XX天气") → 同理拿到天气文本
   步骤3 hotel_agent.run("搜索XX的经济型酒店") → 拿到酒店文本
   步骤4 planner_agent.run(拼接: 基本信息+①②③全部文本+JSON格式要求)
        → LLM 直接输出 JSON 字符串(无工具)
⑦ _parse_response() 三级降级提取 JSON:
   ```json 代码块 → ``` 代码块 → 首个{到最后}
   json.loads → TripPlan(**data) Pydantic 校验
   任何一步失败 → _create_fallback_plan() 生成占位行程
⑧ 包装为 TripPlanResponse(success/message/data) 返回
⑨ 前端存 sessionStorage.setItem('tripPlan', ...)
⑩ 跳转 /result，Result.vue onMounted:
   - 从 sessionStorage 读数据
   - 并发请求 /api/poi/photo?name=景点 给每个景点拉 Unsplash 配图
   - AMapLoader 加载高德 JS API → 打点所有景点
⑪ 用户可编辑行程(增删排序景点,存回 sessionStorage)、导出 PNG/PDF
```

---

## 六、后端逐文件精讲

### 6.1 `app/config.py` — 配置中心

- 用 **pydantic-settings** 的 `BaseSettings` 声明配置类，字段自动从环境变量/`.env` 读取（大小写不敏感：`amap_api_key` ↔ `AMAP_API_KEY`）。
- 加载顺序：先 `load_dotenv()` 读当前目录 `.env`，再尝试读上层 `HelloAgents/.env`（不覆盖已有变量）—— 兼容 HelloAgents 教程项目的环境。
- `cors_origins` 存成逗号分隔字符串再 `split(',')`，因为 CORS origins 列表不方便直接写在 env 里。
- `validate_config()`：`AMAP_API_KEY` 缺失直接抛错（**硬性依赖**）；LLM key 缺失只警告（**软性依赖**）。
- 模块末尾 `settings = Settings()` 即创建了**全局配置单例**，其他文件 `get_settings()` 拿的都是它。

### 6.2 `app/api/main.py` — 应用入口

- 创建 FastAPI 实例，`docs_url="/docs"` 自动生成 Swagger 文档。
- **CORS 中间件**：前端(5173)和后端(8000)端口不同属于跨域，必须放行 `allow_origins`。
- `include_router` 挂载 3 个路由模块，统一前缀 `/api`。
- `@app.on_event("startup")`：启动时打印配置、执行 `validate_config()`，配置缺失直接**启动失败**（fail-fast 思想）。
- 附带 `/`（服务信息）和 `/health`（健康检查）端点。

### 6.3 `app/models/schemas.py` — Pydantic 模型（数据契约）

分三组，前端 `types/index.ts` 与之**逐字段对应**：

- **请求**：`TripRequest`（city/start_date/end_date/travel_days/transportation/accommodation/preferences/free_text_input）、`POISearchRequest`、`RouteRequest`
- **领域对象**：`TripPlan → DayPlan → Attraction/Meal/Hotel`（层级嵌套），`WeatherInfo`、`Budget`、`Location`
- **响应**：统一 `success + message + data` 三段式（`TripPlanResponse` 等）

两个值得注意的细节：

1. `WeatherInfo.parse_temperature` 用 `field_validator(mode='before')` 在校验前清洗温度字符串——LLM 经常输出 `"25°C"`、`"25℃"` 甚至 `"25°"`，这里统一剥单位转 int，失败兜底为 0。**这是应对 LLM 输出不确定性的防御式编程典范。**
2. `day_temp: Union[int, str]` —— 允许两种类型，进一步放宽对 LLM 输出的约束。

### 6.4 `app/services/llm_service.py` — LLM 单例

极简：`HelloAgentsLLM()` 会**自动从环境变量**读取 `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL_ID`（所以 config.py 里那组 `openai_*` 字段实际是备用的）。全局 `_llm_instance` + None 检查实现单例，避免重复建连。

### 6.5 `app/agents/trip_planner_agent.py` — ★★ 项目灵魂

**4 段提示词**（ATTRACTION/WEATHER/HOTEL/PLANNER_PROMPT）：

- 前三个共同点：反复强调"**必须用工具、不要编造**"（对抗 LLM 幻觉），并给出 `TOOL_CALL` 的**精确格式 + 示例**（few-shot 提示）。
- PLANNER_PROMPT 是**输出格式契约**：直接在提示词里贴一份完整的 JSON Schema 样例（含 budget、weather_info 等），要求严格照抄结构。

**`MultiAgentTripPlanner.__init__`**：
- 创建**一个共享的** `MCPTool`（注意 `server_command` 带 `--with pydantic<2.11`），4 个 Agent 中前 3 个 `add_tool(self.amap_tool)` 共享它 —— **只启动一个 MCP 子进程，省资源**。
- 第 4 个 planner_agent 不挂工具（它只做文本整合，不需要真实数据）。
- 初始化全程打印日志，失败打印 traceback 并 raise —— **初始化即失败，不带着坏状态运行**。

**`plan_trip(request)`** —— 核心编排，纯串行：
```python
attraction_response = self.attraction_agent.run(attraction_query)   # 步骤1
weather_response    = self.weather_agent.run(...)                   # 步骤2
hotel_response      = self.hotel_agent.run(...)                     # 步骤3
planner_response    = self.planner_agent.run(self._build_planner_query(
                        request, attraction_response,
                        weather_response, hotel_response))          # 步骤4
return self._parse_response(planner_response, request)
```
任何异常 → `_create_fallback_plan()` 兜底（用假坐标生成占位行程，保证 API 永远有响应结构）。

**`_build_attraction_query`**：从 `preferences` 里**只取第一个**作为搜索关键词（简化处理），并在问题文本里**预置好 TOOL_CALL 指令**。

**`_parse_response`** 三级 JSON 提取（```json → ``` → 花括号），`TripPlan(**data)` 利用 Pydantic 做最终校验。

**`get_trip_planner_agent()`**：模块级全局变量 + 惰性初始化 = 单例。

### 6.6 `app/api/routes/trip.py` — 核心 API

`POST /api/trip/plan`：接收 `TripRequest`（Pydantic 自动校验/文档生成）→ 调 agent → 包 `TripPlanResponse`。异常打 traceback 并转成 `HTTPException(500)`。
⚠️ 注意 `/trip/health` 里访问了 `agent.agent.name`，但 `MultiAgentTripPlanner` 并没有 `.agent` 属性——**这个健康检查端点实际会报错**（见第十节问题清单）。

### 6.7 `app/services/amap_service.py` — 高德服务封装（备用路径）

`AmapService` 封装了 search_poi / get_weather / plan_route / geocode / get_poi_detail 五个方法，**都是通过 `mcp_tool.run({"action":"call_tool", "tool_name":..., "arguments":...})` 显式调用 MCP 工具**。但除 `get_poi_detail` 外，其余方法的返回解析全是 **TODO，固定返回空**。也就是说：**真正的数据获取走的是 Agent 自动调用 MCP 的路径，这个 Service 层是半成品/备用方案**。理解这一点非常重要，否则你会疑惑"为什么 map 路由返回空数据"。

### 6.8 `app/services/unsplash_service.py` — 配图服务

标准 requests 调 Unsplash Search API，`search_photos(query)` 返回图片 URL 列表，`get_photo_url` 取第一张。被 `routes/poi.py` 的 `GET /api/poi/photo?name=景点名` 使用（搜索词是 `"{name} China landmark"`，失败降级为纯名称）。

---

## 七、前端逐文件精讲

### 7.1 `main.ts` + `App.vue`

- 手动创建 vue-router（`createWebHistory`），只有两条路由：`/` → Home，`/result` → Result。
- `app.use(Antd)` 全量注册 Ant Design Vue。
- `App.vue` 用 `a-layout` 搭 Header/Footer 骨架，中间 `<router-view />`。

### 7.2 `services/api.ts` — HTTP 层

- `axios.create({ baseURL: VITE_API_BASE_URL || 'http://localhost:8000', timeout: 120000 })` —— **2 分钟超时**，因为多 Agent 串行 + LLM 多轮调用很慢。
- 请求/响应拦截器只做日志。
- `generateTripPlan()` POST `/api/trip/plan`，错误时提取后端 `detail` 字段抛出。

### 7.3 `views/Home.vue` — 表单页

要点：
- `TripFormState` 用 `Omit<TripFormData,'start_date'|'end_date'>` 改造类型，日期字段存 **Dayjs 对象**（Ant Design 的 DatePicker 返回 Dayjs），提交时 `.format('YYYY-MM-DD')`。
- `watch([start, end])` 自动算 `travel_days = end.diff(start,'day') + 1`，非法区间清空 end_date 并提示。
- 提交时启动**模拟进度条**（`setInterval` 每 0.5s +10%，封顶 90%，文案分四阶段：搜景点→查天气→荐酒店→生成行程）—— 因为后端是一次性长请求、无流式接口，前端只能"表演"进度。
- 成功后 `sessionStorage.setItem('tripPlan', JSON.stringify(data))`，延迟 500ms 跳 `/result`。

### 7.4 `views/Result.vue` — 结果页（最复杂）

按功能块拆解：

1. **数据加载**：`onMounted` 从 sessionStorage 读 plan → 并发 `fetch /api/poi/photo?name=` 给每个景点拉真实图片存 `attractionPhotos` 映射 → `nextTick()` 后初始化地图。
2. **图片降级策略**：`getAttractionImage()` —— 有真实图用真实图；没有就用**内联 SVG 渐变占位图**（base64 编码，`btoa(unescape(encodeURIComponent(svg)))` 处理中文字符）；加载出错 `@error` 换灰色占位图。
3. **高德地图**：`AMapLoader.load({ key: VITE_AMAP_WEB_JS_KEY, version:'2.0', plugins:[...] })` → 遍历所有天的景点 `new AMap.Marker(...)` 打点，并把**不同天用不同颜色**区分，最后 `map.setFitView()` 自适应视野。
4. **编辑模式**：`toggleEditMode` 深拷贝原始数据（`JSON.parse(JSON.stringify())`）作为取消还原的快照；支持上移/下移/删除景点（至少保留 1 个）；保存时写回 sessionStorage 并 `map.destroy()` 后重建地图。
5. **导出**：`exportAsImage` / `exportAsPDF` —— 克隆 DOM 到隐藏容器 → 把地图 canvas 截图替换为 `<img>` → 手动内联关键样式 → `html2canvas` 截图 → PNG 下载或 `jsPDF` 分页写 A4。

---

## 八、项目中用到的设计模式与技巧

| 模式/技巧 | 位置 | 一句话解释 |
|---|---|---|
| 单例模式（惰性初始化） | `get_llm` / `get_amap_service` / `get_unsplash_service` / `get_trip_planner_agent` | 全局变量 + None 检查，首次调用才创建，进程内复用 |
| 流水线编排（Pipeline） | `plan_trip` | 4 个 Agent 串行，前序输出作为后序输入 |
| Fail-fast | `validate_config` + startup 事件 | 配置缺失直接拒绝启动 |
| 兜底降级（Fallback） | `_create_fallback_plan`、前端 SVG 占位图 | 任一环节失败都保证有可用输出 |
| 多级容错解析 | `_parse_response` 三级 JSON 提取 | 应对 LLM 输出格式不稳定 |
| 防御式校验 | `parse_temperature` validator | 校验前清洗 LLM 输出的脏数据 |
| Prompt 工程 | 4 段提示词 | few-shot 格式示例 + "必须用工具不许编造" + 预置 TOOL_CALL |
| 资源共享 | 共享单个 MCPTool | 一个 MCP 子进程服务 3 个 Agent |
| 数据快照 | Result.vue `originalPlan` | 深拷贝支持"取消编辑"还原 |

---

## 九、环境搭建与运行

### 9.1 准备 3 类密钥

| 密钥 | 去哪申请 | 用在哪 |
|---|---|---|
| `AMAP_API_KEY`（**Web 服务**类型） | 高德开放平台 lbs.amap.com | 后端 MCP Server 调高德 HTTP API |
| `VITE_AMAP_WEB_JS_KEY`（**Web端(JS API)** 类型） | 同上，注意是另一种 Key | 前端浏览器渲染地图 |
| `LLM_API_KEY` + `LLM_BASE_URL` + `LLM_MODEL_ID` | DeepSeek / 智谱 / OpenAI 等 | HelloAgentsLLM |
| （可选）`UNSPLASH_ACCESS_KEY` | unsplash.com/developers | 景点配图 |

> 高德的"Web 服务"和"Web端(JS API)"是**两种不同类型的 Key**，不能混用 —— 这是最常见的踩坑点。

### 9.2 后端启动

```bash
cd backend
python -m venv venv
venv\Scripts\activate          # Windows
pip install -r requirements.txt
# 编辑 .env（参考 .env.example）
python run.py                  # 或 uvicorn app.api.main:app --reload --port 8000
```

验证：浏览器开 `http://localhost:8000/docs` 能看到 Swagger 即成功。

> 依赖说明：`hello-agents[protocols]` 是 Agent 框架本体；`uv` 提供 `uvx` 命令用于拉起 MCP Server（**必须保证 `uvx` 在 PATH 里可用**，否则 MCP 初始化失败）；`fastmcp` 是 MCP 的 Python 实现。

### 9.3 前端启动

```bash
cd frontend
npm install
# .env 里配置 VITE_AMAP_WEB_JS_KEY（可选 VITE_API_BASE_URL）
npm run dev    # http://localhost:5173
```

### 9.4 联调自测顺序（建议照做一遍）

1. `GET http://localhost:8000/health` → 后端活着
2. `GET http://localhost:8000/api/map/weather?city=北京` → 看 MCP 链路是否通（注意：返回 data 可能为空，因为解析是 TODO，只看日志有没有真实调用）
3. Swagger 里直接 `POST /api/trip/plan`（用 schemas 里的 example）→ 看后端控制台四步日志
4. 再走前端页面完整流程

---

## 十、项目的已知问题与改进方向（面试加分点）

主动指出这些问题 = 你真的读懂了代码。每一条都可以反过来答"我会怎么改"：

| # | 问题 | 位置 | 改进方案 |
|---|---|---|---|
| 1 | **async 端点里跑同步重代码**：`async def plan_trip` 内直接调用同步的 `agent.plan_trip()`，LLM 调用期间（可达数十秒）**阻塞整个事件循环**，期间所有请求（包括健康检查）都卡住 | `routes/trip.py` | 方案A：改成 `def plan_trip`（FastAPI 自动丢线程池）；方案B：`await run_in_threadpool(...)`；方案C：改异步 LLM 客户端 |
| 2 | **无流式输出**：前端只能"假装"进度条 | Home.vue | 用 SSE/WebSocket 把 4 个步骤的真实进度推给前端 |
| 3 | **AmapService 大半是 TODO**，map 路由基本返回空数据 | `amap_service.py` | 补齐 JSON 解析，或直接删掉改走 Agent 路径 |
| 4 | `/trip/health` 访问不存在的 `agent.agent` 属性，必然 500 | `routes/trip.py` | 改为检查 planner 实例是否初始化 |
| 5 | 前端两处硬编码 `http://localhost:8000`（api.ts 有 env 但图片加载用 fetch 硬编码） | Result.vue `loadAttractionPhotos` | 统一走 `API_BASE_URL` |
| 6 | vite 配了 `/api` 代理但 axios 用完整 URL，**代理实际不生效** | `vite.config.ts` | baseURL 改为 `''` 走相对路径，让代理生效，顺便解决 CORS |
| 7 | 计划数据只存 sessionStorage，**刷新编辑会丢**（sessionStorage 标签页级） | Result.vue | 落库/存 localStorage/后端持久化 |
| 8 | LLM 返回 JSON 靠字符串截取，脆弱 | `_parse_response` | 用 OpenAI 的 JSON mode / structured outputs，或 function calling 输出 |
| 9 | 高德天气 API **只提供未来 4 天预报**，行程超过 4 天天气必缺 | 后端整体 | 超出部分标注"暂无预报"而非留空 |
| 10 | Agent 串行执行慢（1+2+3 可并行） | `plan_trip` | `asyncio.gather` 并行跑步骤1-3 |
| 11 | key 直接打进 MCP 子进程 env，无密钥轮转/脱敏日志 | 全局 | secrets 管理、日志脱敏 |

---

## 十一、7天学习路线

| 天 | 任务 | 检验标准 |
|---|---|---|
| D1 | 跑通环境；手点一遍全流程；开着后端控制台看完整日志 | 能说清日志里四步分别发生了什么 |
| D2 | 精读 `config.py`、`schemas.py`、`api/main.py` | 能默画请求/响应模型嵌套关系 |
| D3 | 精读 `trip_planner_agent.py`（灵魂文件，逐行） | 能脱稿讲 plan_trip 四步 + 两处容错 |
| D4 | 花半天理解 MCP：读 hello-agents 库里 `MCPTool` 源码（venv 里能找到），观察 uvx 子进程 | 能解释 MCP 三角色与 stdio 传输 |
| D5 | 精读前端 `Home.vue` + `api.ts` | 能讲清 Dayjs 处理与 sessionStorage 传递 |
| D6 | 精读 `Result.vue`（地图、编辑、导出三块） | 能讲清图片三级降级与导出流程 |
| D7 | 动手改造（任选）：修第十节问题 #1 或 #5；把步骤1-3并行化；给 map 路由补解析 | 改完能跑通 = 真吃透 |

---

## 十二、高频面试题与参考答案

**Q1：介绍一下这个项目？**
> 一个 AI 旅行规划助手，前后端分离。用户提交目的地和偏好后，后端用 FastAPI 接收请求，由基于 HelloAgents 框架的 4 个 LLM Agent 流水线协作：景点/天气/酒店三个 Agent 通过 MCP 协议调用高德地图 MCP Server 获取真实数据，第四个规划 Agent 汇总生成结构化 JSON 行程，经 Pydantic 校验后返回。前端 Vue3 渲染行程、高德地图打点、支持编辑和导出 PNG/PDF。我在其中重点解决了 LLM 输出不稳定（多级 JSON 容错+Pydantic 清洗）、工具调用格式可靠性（Prompt 预置指令）等问题。

**Q2：MCP 是什么？为什么用它而不是直接写高德 API 封装？**
> MCP 是标准化 LLM 应用与外部工具间连接的开放协议，分 Host/Client/Server 三角色。好处：① 工具以标准协议暴露，即插即用，换地图供应商只要换 Server；② 能力发现机制——客户端可动态列出 Server 的全部工具，新增工具零代码；③ 解耦——鉴权、限流、版本都收敛在 Server 一侧。本项目 MCPTool 通过 uvx 拉起 amap-mcp-server 子进程，用 stdio 通信，auto_expand 展开后 Agent 直接获得十几个高德工具。

**Q3：多 Agent 比单 Agent 好在哪？你的编排是怎样的？**
> 我的编排是**代码驱动的串行流水线**：三个带工具的 Agent 分别产出景点/天气/酒店文本，规划 Agent 拿到三段文本和用户偏好生成最终 JSON。好处：每个 Agent 的 system prompt 职责单一、输出更稳；编排顺序确定、可调试可复现；单点失败不影响其他环节。局限是串行慢，景点/天气/酒店查询其实无依赖，可以用 asyncio 并行——这是我已识别的改进点。

**Q4：LLM 输出的 JSON 不合法怎么办？（必问）**
> 四层防御：① Prompt 里给完整 JSON 模板并强调严格遵循；② 解析时三级降级提取（```json 块 → ``` 块 → 首尾花括号截取）；③ Pydantic 模型校验 + field_validator 清洗脏数据（如剥掉温度里的 °C）；④ 全链路 try/except，最终兜底一个静态占位行程，保证接口永远返回合法结构。

**Q5：FastAPI 的 async 端点里能直接调同步阻塞代码吗？**
> 不能。`async def` 端点运行在事件循环上，同步长调用会阻塞整个循环，期间所有请求排队。我这个项目恰好有这个问题，两种修法：把端点改成普通 `def`（FastAPI 自动放线程池），或者 `from starlette.concurrency import run_in_threadpool` 包一层。这也是我读源码时发现并想优先修复的点。

**Q6：单例模式在这里为什么必要？怎么实现的？**
> LLM 客户端、MCP 子进程连接、Agent 系统的创建成本高（MCP 要拉起子进程），且无状态可安全复用，必须全进程一份。实现是模块级全局变量 + 惰性初始化：首次调用 get_xxx() 时创建，后续直接返回。相比 `__new__` 或元类实现，Python 里这种函数级单例最简单直观。

**Q7：为什么用 sessionStorage 而不是 localStorage？跨页面怎么传大对象？**
> 行程计划是会话级临时数据，关标签页即失效符合业务语义；localStorage 会长期残留。传大对象不用 URL 参数，直接 JSON 序列化进 sessionStorage，结果页 onMounted 读回。但 sessionStorage 不跨标签页且仅 5MB，改进方向是后端持久化 + 计划 ID 路由。

**Q8：前端跨域怎么解决的？**
> 开发期后端用 CORSMiddleware 白名单放行 5173 端口。其实 vite.config 里也配了 /api 代理，但因 axios baseURL 写了完整地址导致代理没生效——正确做法是 baseURL 留空走相对路径让代理转发，这样连 CORS 都不需要。生产环境则应同域部署或走网关。

**Q9：如何防止 Agent 编造景点信息（幻觉）？**
> 三招：① 提示词强约束"必须用工具、不许编造"；② 用预置 TOOL_CALL 指令把模型"第一反应"引导到真实调用上；③ 数据源只有高德 MCP 返回的真实 POI，规划 Agent 只做组织不做检索。彻底解决还需要对 Agent 产出与 MCP 原始结果做交叉校验，这是可迭代方向。

**Q10：如果让你把这个项目做成支持 10 万用户的生产系统，改什么？**
> ① 架构：LLM 调用改为异步 + 任务队列（生成计划耗时分钟级，应改为提交任务+轮询/SSE 取结果）；② 并发：三路查询并行化、MCP 连接池化；③ 成本：加缓存（同城市+日期的天气/POI 结果 Redis 缓存）、控制上下文长度；④ 可靠性：LLM 结构化输出、重试与超时策略、限流；⑤ 存储：行程落库支持分享链接；⑥ 可观测：结构化日志+链路追踪替代 print。

---

## 附：一页纸速记

- **一个入口**：`POST /api/trip/plan`
- **一个灵魂类**：`MultiAgentTripPlanner`（4 Agent 串行流水线 + JSON 三级解析 + fallback 兜底）
- **一个协议**：MCP（MCPTool → uvx 子进程 amap-mcp-server → 高德 API）
- **一个框架**：HelloAgents（SimpleAgent + HelloAgentsLLM + MCPTool）
- **两条数据链**：LLM 链（4 次 agent.run）；真实数据链（高德 MCP + Unsplash 配图）
- **两个 Key 别搞混**：后端"Web服务"key ↔ 前端"JS API"key
- **三层防御**：Prompt 约束 → 解析降级 → Fallback 兜底
