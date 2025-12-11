# 联脉（ReachFlow）前端交付

本仓库提供联脉（ReachFlow）的两套前端交付件：

- **Landing Page**：面向市场与销售的品牌着陆页，覆盖价值传达、线索收集、A/B 试验与埋点。
- **Research 子页面**：AI 背调实验台（`research.html`），实时消费后端 `/research/stream` SSE，帮助内测用户验证 Deep Research Agent 的推理日志与报告输出。
- **Docs**：支撑交付的部署、流式对接、规划说明，位于 `docs/`。

---

## 目录结构

```
.
├── index.html           # Marketing Landing Page
├── research.html        # Research 流式聊天子页面
├── app.js               # 共用交互、埋点、SSE 逻辑
├── styles.css           # 设计系统 + Chat UI 样式
├── assets/              # favicon、OG、logo/图标占位
│   ├── favicon.svg
│   ├── icon.png
│   └── logos/
├── docs/
│   ├── deployment-readiness.md
│   ├── frontend_streaming_handoff.md
│   └── research-subpage-plan.md
└── README.md
```

---

## 本地开发 / 预览

项目为纯静态资源，无需构建：

```bash
# 任意静态服务器均可
python3 -m http.server 3000
# 或
npx serve .
```

- 访问 `http://localhost:3000/index.html` 查看 Landing Page。
- 访问 `http://localhost:3000/research.html` 查看 Research 子页面。需要设置后端地址时，可：
  1. 在 `research.html` 的 `<body data-api-base-url="https://your-domain/api">` 中写入。
  2. 或部署阶段注入 `window.__RESEARCH_CONFIG__.apiBaseUrl`（见文档与模板脚本）。

直接以 `file://` 打开也能浏览，但由于无法发起 SSE，Research 页面需运行在本地服务器以调试真实请求。

---

## Landing Page 能力

### 信息架构 & 交互
- Hero、痛点对比、三步工作流、场景标签页、关键能力、指标、客户证言、价格方案、FAQ、CTA 与合规页脚等区块完整覆盖。
- 移动端菜单、场景 Tab、模版 Modal、合规 Drawer、FAQ 埋点、Toast 提示等交互均在 `app.js` 中控制。
- 所有 CTA / 关键交互通过 `data-track` 触发 `trackEvent`，统一写入 `window.dataLayer`。

### 表单与埋点对接
1. `app.js` 的 `heroForm()` 负责前端校验；默认只显示成功 Toast。
2. 接入真实后端时在该函数内补充 `fetch`/`XMLHttpRequest`，并分别触发 `trackEvent("form_submit_success")` 与 `trackEvent("form_submit_fail")`。
3. `validateContact()` 同时支持邮箱、电话、微信号校验，可按策略调整正则。

### A/B 变量
通过 URL 参数开启实验，命中后会记录 `ab_variant_applied` 事件。

| 参数             | 示例取值                 | 作用说明                                   |
|------------------|--------------------------|--------------------------------------------|
| `ab_h1`          | `B`                      | 切换首屏标题为 “首批可联对象，T+24 必达”。 |
| `ab_secondary`   | `demo` / `whitepaper`    | 调整次级 CTA 文案及埋点。                   |
| `ab_form_fields` | `extended` / `5`         | 显示额外的“预估月外联量”字段。             |
| `ab_trust`       | `logos` / `metrics`      | 切换首屏信任呈现：Logo 墙或指标徽章。       |
| `ab_pricing`     | `hidden`                 | 将价格改为“联系我们获取报价”。             |

示例：`https://example.com/?ab_h1=B&ab_secondary=whitepaper&ab_trust=metrics`。

### 品牌与设计交付
- **字体**：系统字体 + 思源黑体 / 阿里巴巴普惠体，若需 Web Font 请在 `styles.css` 中追加 `@font-face`。
- **配色**：主色 #2F6FED、辅色 #1BBF72、警示 #F59E0B、错误 #EF4444，详见 `:root` 变量。
- **响应式**：≥1200 / 1024 / 768 / 640 断点，自适配移动端，按钮高度 ≥44px。
- **资产替换**：`assets/` 中留有 favicon、OG、logo 占位，可按品牌物料替换。

---

## Research 子页面（AI 背调实验台）

### 核心能力
- 聊天面板实时渲染 Markdown，自动滚动并区分用户 / Agent。
- 侧栏“研究日志”按事件流展示搜索、抓取、工具输出与最终汇总，顶部状态指示 `idle | active | streaming | error`。
- 表单包含 Provider 切换（OpenAI/Anthropic/Gemini）、高级设置折叠区（API Key、模型、OpenAI Base URL、EXA Key）。
- `sessionStorage`（键 `reachflow_research_chat_history`）持久化单 Tab 会话，可清空或刷新恢复。
- “停止”按钮借助 `AbortController` 中断本地 SSE 连接，不影响其他用户。

### 接入 `/research/stream`
- `initResearchPage()` 会读取：
  - `<body data-api-base-url>` 属性；
  - 或 `window.__RESEARCH_CONFIG__.apiBaseUrl`；
  - 若都未提供，默认 `/api`。
- 提交表单后调用 `fetch(${apiBaseUrl}/research/stream)` 并解析 `text/event-stream`，逐条分发 `search_start`、`search_results`、`open_url_result`、`tool_result`、`assistant_message`、`final`、`error`、`ping`、`done` 等事件（详见 `docs/frontend_streaming_handoff.md`）。
- UI 在 `handleStreamEvent()` 中维护：
  - 聊天区：增量更新/流式写入助手气泡；
  - Timeline：每个事件带时间戳；
  - 状态条：`data-status` + 文案提示；
  - Toast：复用全局 `#form-toast`；
  - 清空/停止：`data-clear-chat`、`data-stop-stream` 控制按钮态。

### 自定义提示 / 鉴权
- 高级设置允许用户传入 Provider API Key / 模型 / Base URL，如无需暴露可在后端忽略。
- 如果部署在受限环境，可在反代层加 Token，再通过额外输入框写入请求体。
- 需要引用更多 Provider 或工具事件时，可在 `handleStreamEvent()` switch 中追加分支。

---

## 埋点事件
- Landing Page 已为导航、CTA、FAQ、合规抽屉、场景 Tab、表单 focus/submit 等埋点（`data-track="*"`）。
- Research 页面新增事件 `research_submit`、`research_stop_stream`、`research_clear_chat`、`research_toggle_advanced`、`research_back_home` 等，均复用同一 `trackEvent()`。
- 统一输出结构：`{ event: name, ...payload }` → `window.dataLayer`。

---

## 部署建议
1. **静态资源**：`index.html` / `research.html` 与 `styles.css`、`app.js` 可直接托管在 Vercel、Netlify、Cloudflare Pages、OSS 等；HTML 建议短缓存或 `no-cache`，CSS/JS 可长缓存并带指纹。
2. **Landing Page**：若追加三方分析脚本，在 `index.html` `<head>` 或 `</body>` 前异步引入即可。A/B 变体通过 URL 控制，无需额外构建。
3. **Research 页面**：
   - 确保反向代理允许 `text/event-stream`，并设置 `Cache-Control: no-cache`、`Connection: keep-alive`。
   - 后端需开启 CORS 允许部署域名；心跳 `ping` 频率 15s，可据此设置超时。
   - 多用户并发、日志脱敏、鉴权/限流、自查清单详见 `docs/deployment-readiness.md`。

---

## 配套文档
- `docs/frontend_streaming_handoff.md`：SSE 对接说明、事件字段与前端解析模板。
- `docs/research-subpage-plan.md`：Research 页面交互/技术规划与验收标准。
- `docs/deployment-readiness.md`：上线自查、并发/安全建议与 Checklist。

---

## 后续迭代参考
- **M1（上线 +3 天）**：落地表单提交 → 邮件或企微 Webhook；Landing Page Hero/Workflow/CTA 首屏曝光；Research 页面联调真实 API，完成基础监控。
- **M2（+7 天）**：Landing Page 价格、FAQ、合规抽屉与 A/B 实验全量启用；Research Timeline 增加抓取摘要预览，接入埋点告警。
- **M3（+14 天）**：替换真实客户素材、完善 SEO/OG、性能优化（LCP ≤ 2.5s）；Research 支持多会话导出、权限/配额管理。

以上可根据团队节奏调整，推荐在 `docs/` 目录持续沉淀协作记录。
