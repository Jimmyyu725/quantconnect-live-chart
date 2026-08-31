# QuantConnect Live Chart 设计规格

状态：已由用户在 2026-08-31 确认

## 1. 目标

在 Atlas 上建立一个私人、持续运行的行情仪表盘。它从 QuantConnect Cloud 实时算法读取数据，支持美股、ETF 和 Coinbase 加密货币，并为未来的股票与加密货币模拟交易保留清晰的扩展边界。

第一版必须做到：

- 在手机、16:9 桌面和 16:10 桌面上提供一致且无横向滚动的响应式体验。
- 显示蜡烛图或折线图、成交量、25 个技术指标、市场状态和实际更新时间。
- 支持用户不停机切换 QuantConnect 当前支持的美国上市股票、ETF 和 Coinbase 交易对。
- 提供 `1D`、`5D`、`1M` 三个历史范围与兼容的 K 线周期。
- 在数据延迟、断线、无效代码和市场休市时给出准确状态。
- 将 QuantConnect 凭据限制在 Atlas 后端，不向浏览器暴露。
- 保证行情算法不创建订单。

## 2. 非目标

第一版明确不包含：

- 期权行情、期权链或期权模拟交易。
- 真实券商连接、真实资金交易或任何下单按钮。
- 秒级、逐笔或盘口行情展示。
- 公网部署、公开分享或行情数据再分发。
- 自建上百个指标或依赖需要单独商业授权的 TradingView Charting Library。
- 邮件、短信、Telegram 或其他价格提醒。

## 3. 已确认的产品决策

| 主题 | 决策 |
| --- | --- |
| 成品形态 | Atlas 上运行的私人网页 |
| 访问方式 | 仅绑定 `127.0.0.1`，通过 SSH 隧道访问 |
| 数据源 | 只使用 QuantConnect Cloud，不增加第三方行情供应商 |
| 初始用途 | 个人盯盘；未来扩展至股票和加密货币模拟交易 |
| 资产范围 | 美国上市股票、ETF、Coinbase 加密货币；排除 OTC 与期权 |
| 默认资产 | 股票 `SPY`；加密货币 `BTCUSD`，页面默认打开 `SPY` |
| 市场时段 | 美股包含盘前、正常交易和盘后；加密货币按 24/7 处理 |
| 图表 | 蜡烛图与折线图可切换 |
| 历史范围 | `1D`、`5D`、`1M` |
| 默认粒度 | `1D → 1m`、`5D → 5m`、`1M → 30m` |
| 可选粒度 | `1m`、`5m`、`15m`、`30m`、`1h`、`1D`；不可用组合禁用 |
| 指标 | 25 个可搜索、开关、修改参数的常用指标 |
| 默认指标 | EMA 20、VWAP、RSI 14 |
| 运行方式 | QuantConnect 算法和 Atlas 服务持续运行 |
| 视觉方向 | 浅色 QuantConnect 风格：白底、细灰边框、红绿行情色、蓝色辅助图 |
| 实时性表述 | 页面通常约每分钟更新；显示实际采样间隔，绝不宣称逐笔实时 |

## 4. 方案选择

选择“QuantConnect Cloud 实时算法 + Atlas 自定义网页”。

没有选择直接嵌入 QuantConnect Live Results，因为其界面、代码搜索、指标库和响应式布局无法满足已确认的产品设计。没有选择第三方行情供应商，因为秒级展示不是第一版目标，而且额外供应商会增加费用、凭据和数据授权复杂度。

## 5. 系统架构

### 5.1 QuantConnect Cloud 算法

职责：

- 使用 QuantConnect 数据提供商持续订阅默认的 `SPY` 与 `BTCUSD`，并在需要时增加最多一个当前搜索标的。
- 股票使用 `Market.USA`、分钟数据、Raw normalization 和 extended market hours。
- 加密货币使用 `Market.COINBASE`。
- 接收 Atlas 通过 Live Command 发送的 JSON 控制命令。
- 在切换时先验证并订阅新标的，成功后才移除旧的非默认搜索标的；`SPY` 与 `BTCUSD` 常驻订阅不移除。
- 对当前“标的 + 范围 + 粒度”执行一次同步 `History` 回填。
- 将历史和后续数据写入固定的 Candlestick 与 Volume 序列。
- 用自定义 runtime statistics 报告当前请求、标的、粒度、状态和错误。
- 从不调用任何订单 API。

算法状态机为：

```text
IDLE → SWITCHING → BACKFILLING → LIVE
                    ↘ ERROR ↗
```

`ERROR` 不清除最后一个成功图表。新订阅失败时，旧订阅和旧图继续保留。

### 5.2 Atlas 后端

采用 Python 3.12 与 FastAPI。模块边界如下：

- `quantconnect_client`：签名认证、请求、重试和响应解析。
- `symbol_registry`：标准化并验证美股、ETF 与 Coinbase 交易对。
- `command_coordinator`：串行化切换命令、去重并确认算法状态。
- `chart_reader`：读取 Candlestick、Volume 与 runtime statistics。
- `market_cache`：保存最近 31 天的自适应 OHLCV 图表点。
- `market_clock`：判定美股盘前、正常交易、盘后、休市及加密货币 24/7 状态。
- `api`：向浏览器提供只读 JSON、控制请求和 Server-Sent Events 更新。

每个模块对外提供窄接口；QuantConnect 响应模型不直接泄漏到浏览器协议中。

### 5.3 网页前端

采用 TypeScript、React、Vite 与开源 Lightweight Charts。前端职责：

- 渲染响应式仪表盘、蜡烛图、折线图、成交量和指标副图。
- 发起资产、范围、粒度和指标配置变更。
- 从后端接收初始快照与 Server-Sent Events 更新。
- 在浏览器中根据 OHLCV 计算指标，不占用 QuantConnect 图表序列额度。
- 将最后一次资产、范围、粒度、图表类型和指标参数保存在本地浏览器配置中。

前端只访问 Atlas 后端；它不知道 QuantConnect User ID、Token、项目 ID或部署 ID。

### 5.4 本地缓存

采用 SQLite，存储经过 QuantConnect 图表接口返回的自适应 OHLCV 点、请求元数据和最后成功时间。

- 每个资产与粒度最多保留 31 天。
- 不保存 tick、逐笔成交、盘口或原始数据包。
- 缓存文件属于运行时状态，不进入 Git。
- 数据只用于用户本人在私人 Atlas 页面查看，不提供导出或共享端点。

## 6. 数据流

### 6.1 首次加载与切换

1. 浏览器提交资产类型、代码、范围和粒度。
2. Atlas 规范化输入并验证其是否属于支持范围。
3. Atlas 生成唯一 `requestId`，把命令写入串行队列。
4. Atlas 调用 QuantConnect `POST /live/commands/create`。
5. 若目标不是已常驻的 `SPY` 或 `BTCUSD`，算法先添加新订阅，再进入 `BACKFILLING`。
6. 算法在 warm-up 结束后执行同步 `History`，使用历史 bar 自带的时间戳填充 Candlestick 与 Volume 序列。
7. Atlas 轮询 runtime statistics；只有请求 ID、标的和粒度完全匹配且状态为 `LIVE` 才确认切换成功。
8. Atlas 读取 `/live/chart/read`；接口返回 `loading` 时每 10 秒重试。
9. Atlas 写入 SQLite 并将快照返回浏览器。
10. 新订阅稳定后，算法只移除旧的非默认搜索标的；常驻的 `SPY` 与 `BTCUSD` 保留。

同步 live `History` 通常会暂停算法约 5–10 秒，因此页面显示明确的加载状态，不承诺瞬时切换。浏览器在此期间继续显示最后一个成功图表。

### 6.2 持续更新

- 算法向相同两条固定序列追加当前活动标的的数据。
- Atlas 按 QuantConnect 实际图表采样节奏读取增量结果。
- 后端去重并校验时间顺序，再写入缓存并通过 Server-Sent Events 推送。
- 页面根据新 OHLCV 点增量更新可见指标。
- 页脚显示数据源、实际粒度、最后一个完成 bar 的时间和最后一次 API 成功时间。

### 6.3 范围与粒度矩阵

默认组合为：

- `1D`：`1m`
- `5D`：`5m`
- `1M`：`30m`

用户可从 `1m`、`5m`、`15m`、`30m`、`1h`、`1D` 中选择，但前端必须禁用会超过 QuantConnect 图表点限额或无法产生有意义显示的组合。切换组合视为一次新的可取消请求；成功结果写入独立缓存键。

## 7. 指标库

第一版提供以下 25 个指标：

### 趋势

1. SMA（20）
2. EMA（20）
3. WMA（20）
4. HMA（20）
5. VWAP（按交易时段重置）
6. Ichimoku（9、26、52）
7. Supertrend（10、3）
8. Parabolic SAR（0.02、0.2）

### 动量

9. RSI（14）
10. Stochastic（14、3、3）
11. Stoch RSI（14、14、3、3）
12. MACD（12、26、9）
13. CCI（20）
14. ROC（12）
15. Williams %R（14）

### 波动

16. Bollinger Bands（20、2）
17. ATR（14）
18. Keltner Channels（20、2）
19. Donchian Channels（20）
20. Standard Deviation（20）

### 成交量

21. OBV
22. MFI（14）
23. CMF（20）
24. Volume SMA（20）
25. Accumulation/Distribution

主图叠加指标和副图指标分别管理。参数必须有合理上下限，错误参数不能破坏其他已启用指标。

## 8. 界面设计

### 8.1 共同元素

- 顶部：品牌、LIVE/STALE/DISCONNECTED 状态。
- 控制区：股票/加密货币切换、代码搜索、范围、粒度、蜡烛/折线切换。
- 摘要区：最后价格、当日涨跌、成交量。
- 主图：OHLCV、当前价格线和最多数个主图指标。
- 指标区：搜索、开关、参数和独立副图。
- 页脚：市场阶段、最后更新时间、数据源和实际采样间隔。

### 8.2 手机

宽度从 320px 起支持。所有区块单列排列；触控目标不小于约 44px；不允许页面横向滚动。次要信息折叠而不是缩小到不可读。

### 8.3 16:9 与 16:10 桌面

页面展开为约 70/30 的两栏结构。主图位于左侧，指标与数据状态位于右侧。16:10 可增加纵向图表高度；16:9 优先保持主图和关键状态在首屏内。布局通过内容断点重排，不按比例拉伸手机界面。

## 9. 状态与错误处理

- `LIVE`：算法运行、请求状态匹配、数据在该市场应有的时间窗口内更新。
- `LOADING`：历史回填或 `/live/chart/read` 尚在生成结果。
- `STALE`：市场应开放，但最后完成 bar 或 API 成功时间超过允许窗口。
- `MARKET CLOSED`：市场按日历休市，不作为故障。
- `DISCONNECTED`：连续三次后端读取失败或 live deployment 不再运行。
- `ERROR`：无效代码、命令拒绝、历史请求失败、图表额度错误或无法解析的响应。

规则：

- 同一时刻只执行一个切换；队列中仅保留最后一个未开始请求。
- QuantConnect 命令 API 返回成功只表示“已接受”，不能直接显示为切换完成。
- 所有重试使用带随机抖动的指数退避，上限 60 秒。
- 失败时继续展示最后一个成功快照，并把失败原因与发生时间显示给用户。
- 浏览器刷新不重启算法，也不清空缓存。

## 10. 安全与隐私

- 后端从 Atlas 现有用户级 LEAN credentials 读取 QuantConnect 凭据。
- Token 不进入项目文件、前端 bundle、浏览器存储、URL、错误页面或日志。
- 后端仅绑定 `127.0.0.1`，默认拒绝跨域访问。
- 不生成公开 Live Stream 或 iframe，不开放行情导出接口。
- 运行日志对 Authorization、Cookie 和请求签名做字段级脱敏。
- 云端算法在代码层不包含下单方法；测试同时断言订单列表始终为空。
- 未来模拟交易功能必须作为独立适配层和单独设计，不复用图表算法的订阅移除流程。

## 11. 部署与运行

- QuantConnect：使用现有 Quant Researcher 组织和一台空闲 `L-MICRO` live node，部署 QuantConnect Paper Trading 行情算法并启用自动恢复。
- Atlas：使用用户级 systemd 服务 `quantconnect-live-chart.service` 持续运行 FastAPI 与静态前端。
- 健康检查：`/health/live` 只检查进程；`/health/ready` 同时检查数据库、QuantConnect 认证和 live deployment 状态。
- 缓存与日志保存在项目外的运行时目录或被 `.gitignore` 排除的目录中。
- 通过 SSH 本地端口转发访问，不创建公网 DNS、TLS 终止或公开防火墙规则。

## 12. 测试策略

### 12.1 单元测试

- QuantConnect 请求签名、时间戳和错误解析。
- 股票与 Coinbase 代码规范化和拒绝规则。
- 范围/粒度兼容矩阵和 OHLCV 时间顺序。
- 缓存去重、保留期和恢复。
- 25 个指标的标准样本、边界参数和空数据行为。
- 市场阶段、节假日、扩展时段和 stale 判定。

### 12.2 集成测试

- 使用假 QuantConnect 服务覆盖成功、`loading`、限流、超时、断线和格式错误。
- 验证命令串行化、最后请求优先、重复 request ID 和失败回退。
- 验证 SSE 重连不会重复蜡烛或倒序更新。
- 云端编译并执行无交易回测，确认 Candlestick、Volume 和 runtime statistics 的结构。

### 12.3 实际部署验证

- 部署后确认 `L-MICRO` 节点运行且订单数为零。
- 验证 `SPY → BTCUSD → SPY` 不停机切换。
- 验证首次历史回填、随后分钟更新和休市状态。
- 人为中断 Atlas 到 QuantConnect 的访问，验证 STALE、DISCONNECTED 和自动恢复。
- 在 360px、16:9 和 16:10 视口执行浏览器截图与交互检查。
- 检查 Git 历史、前端资源、浏览器网络记录与服务日志中没有 Token。

## 13. 完成标准

以下条件全部满足才可称第一版完成：

1. 所有自动化测试通过。
2. 云端算法编译、部署并持续运行，订单数始终为零。
3. SPY 与 BTCUSD 的切换、历史回填和持续更新通过实际验证。
4. 25 个指标可启用、关闭和修改参数，失败互相隔离。
5. 手机、16:9 和 16:10 页面无重叠、截断或横向滚动。
6. 断线、数据过期、加载和休市状态与真实条件一致。
7. QuantConnect 凭据不出现在浏览器、项目历史或日志中。
8. Atlas 服务已配置自动启动和失败重启。
9. 项目文档说明访问、运维、停止和恢复步骤。

## 14. 官方依据

- [QuantConnect Live Results](https://www.quantconnect.com/docs/v2/cloud-platform/live-trading/results)
- [Read Live Chart API](https://www.quantconnect.com/docs/v2/cloud-platform/api-reference/live-management/read-live-algorithm/charts)
- [Create Live Command API](https://www.quantconnect.com/docs/v2/cloud-platform/api-reference/live-management/live-commands/create-live-command)
- [Live Algorithm Commands](https://www.quantconnect.com/docs/v2/writing-algorithms/live-trading/commands)
- [Charting and Candlestick Series](https://www.quantconnect.com/docs/v2/writing-algorithms/charting)
- [Historical Data in Live Trading](https://www.quantconnect.com/docs/v2/writing-algorithms/historical-data/live-trading)
- [US Equities Data](https://www.quantconnect.com/docs/v2/cloud-platform/datasets/quantconnect/us-equities)
- [Crypto Data](https://www.quantconnect.com/docs/v2/cloud-platform/datasets/quantconnect/crypto)
- [Dataset Licensing](https://www.quantconnect.com/docs/v2/cloud-platform/datasets/licensing)
