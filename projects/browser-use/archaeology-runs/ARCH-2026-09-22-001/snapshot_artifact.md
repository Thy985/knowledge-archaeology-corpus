# Repository Snapshot — ARCH-2026-09-22-001 (browser-use)

## 快照记录
- **repository**: https://github.com/browser-use/browser-use.git
- **commit SHA**: `d8110c5ff87ccba887aaa726cdb780f2f84bef8d`
- **commit message**: "docs: add PZERO OpenAI-compatible provider example (#5579) (#5648)"
- **branch**: main（浅克隆 depth=1）
- **analysis timestamp**: 2026-09-22T02:05:00+08:00
- **repository version**: browser-use 0.13.10（pyproject.toml，Python >=3.11,<4.0）
- **skill version**: knowledge-archaeology v3.2
- **规模**: 15MB / 518 files（tracked）/ browser_use 包 66,541 行 Python / tests 35,131 行

## 项目基础地图
```
browser-use/
├── browser_use/              # 主包（66,541 行）
│   ├── agent/                # Agent 循环核心（service.py 4,163 行）
│   │   ├── message_manager/  # 消息管理（token 预算/历史裁剪）
│   │   ├── system_prompts/   # 系统提示词模板
│   │   ├── judge.py          # 任务完成判定
│   │   └── variable_detector.py
│   ├── beta/                 # Beta 功能（service.py 6,810 行——最大文件）
│   ├── browser/              # 浏览器控制（CDP 会话 4,153 行）
│   │   ├── session.py        # CDP 会话管理
│   │   ├── chrome.py         # Chrome 启动/配置
│   │   ├── profile.py        # 用户 profile 管理
│   │   ├── watchdogs/        # 动作监视器（default 3,752 行 + downloads 1,503 行）
│   │   ├── cloud/            # Browser Use Cloud 集成
│   │   └── demo_mode.py      # 演示模式
│   ├── controller/           # 动作控制器（注册/执行）
│   ├── dom/                  # DOM 处理
│   │   ├── serializer/       # DOM 序列化（1,406 行）
│   │   ├── markdown_extractor.py  # HTML→Markdown 提取
│   │   └── views.py          # DOM 视图模型
│   ├── llm/                  # LLM 抽象（多供应商）
│   │   ├── anthropic/ aws/ azure/ google/ groq/ mistral/ deepseek/ cerebras/ ollama/ litellm/ browser_use/
│   │   ├── base.py           # LLM 基类
│   │   └── models.py         # 模型注册
│   ├── mcp/                  # MCP 服务器（1,294 行）
│   ├── sandbox/              # 沙箱（隔离执行）
│   ├── tools/                # 工具层（service.py 2,327 行）
│   ├── telemetry/            # 遥测（posthog）
│   ├── tokens/               # token 统计
│   ├── actor/                # 角色/actor 抽象（element.py 1,182 行）
│   ├── filesystem/           # 文件系统工具
│   ├── cli.py                # CLI 入口（基于 Browser Harness）
│   ├── config.py             # 配置（pydantic-settings Config）
│   └── observability.py      # 可观测性
├── tests/                    # 测试（35,131 行：agent_tasks/ ci/ mind2web_data/ scripts/）
├── examples/                 # 示例
├── skills/                   # 技能目录
├── docker/ + Dockerfile      # 容器化
├── AGENTS.md                 # 治理规则 v2
├── CLAUDE.md                 # Claude 开发指引
└── pyproject.toml            # 构建/依赖
```

## 关键信息
### 主要语言
Python（>=3.11），pydantic v2 全栈类型安全；uv 构建管理

### 主要运行入口
1. 库 API：`from browser_use import Agent, Browser`（异步 Agent.run()）
2. CLI：`browser_use`（基于 browser-harness，`BH_CLIENT=browser-use-cli` 环境标记）
3. MCP 服务器：`browser_use/mcp/server.py`
4. Beta 服务：`browser_use/beta/service.py`（6,810 行，最大文件）

### 核心模块
- agent/service.py（Agent 主循环：task→observe→decide→act）
- browser/session.py（CDP 会话：页面导航/网络/截图）
- dom/serializer/serializer.py（DOM 快照序列化——上下文压缩关键）
- controller/（动作注册表：click/type/scroll 等 + 自定义动作）
- beta/service.py（多步规划/高自动化 beta 流程）
- browser/watchdogs/（默认动作监视器 + 下载监视器——状态守护）

### 核心数据结构
- AgentOutput（动作决策结构：current_state + actions）
- BrowserState（DOM 快照 + 截图 + 错误）
- ActionResult（动作结果结构化返回）
- BrowserSessionContext / TabInfo（会话/标签模型）
- DOMElementNode / DOMTextNode（DOM 视图模型）
- LLMResponse（供应商响应归一化）

### 核心状态
- 浏览器会话状态（CDP connection / tabs / pages）
- Agent 历史消息（message_manager 管理 token 预算）
- Watchdog 状态（默认动作监视：任务完成检测/失败检测）
- 文件系统状态（browser_use/filesystem）

### 主要测试体系
- tests/agent_tasks/（agent 任务级测试）
- tests/mind2web_data/（mind2web 基准数据——网页导航基准）
- tests/ci/（CI 测试）
- tests/scripts/（脚本）
- 35,131 行测试（大量多供应商 mock）

### 主要配置
- config.py（pydantic-settings Config，env 前缀 BROWSER_USE_）
- `.env`（API keys：ANTHROPIC_API_KEY/OPENAI_API_KEY/GROQ_API_KEY 等）
- LLM 多供应商：anthropic/openai/groq/google/ollama/mistral/deepseek/cerebras/aws/azure/litellm/browser_use

### 主要权限 / policy / governance 机制
- AGENTS.md v2 治理规则（uv 强制/类型安全/pre-commit/不建随机示例/推荐 ChatBrowserUse 模型/推荐 use_cloud）
- sandbox/ 沙箱模块（隔离执行）
- watchdogs（动作监视与状态守护）
- CLAUDE.md 开发指引
- 浏览器 profile 权限边界（profile.py）

### 主要外部依赖
- aiohttp/httpx（网络）、pydantic 2/pydantic-settings（模型/配置）、posthog（遥测）、mcp（MCP 协议）、cdp-use（CDP 封装）、browser-harness（CLI 后端）、browser-use-sdk（云端）、google-genai/openai/anthropic/groq/ollama（LLM）、pyotp（TOTP/2FA）、markdownify（HTML→MD）、psutil/screeninfo（系统）

### 治理/商业化信号（考古关注点）
- AGENTS.md 明确推荐自有模型 ChatBrowserUse + Browser Use Cloud（use_cloud=True）+ BROWSER_USE_API_KEY——**OSS 漏斗产品化**模式（与 mem0 同构，跨项目候选证据）
- telemetry/ posthog 遥测
- cloud/ 云端浏览器集成
