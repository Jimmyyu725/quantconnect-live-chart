# QuantConnect Live Chart Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Atlas 上交付一个只通过 SSH 隧道访问、由 QuantConnect Cloud 实时算法供数、支持美股/ETF与 Coinbase 加密货币、25 个指标和响应式布局的私人行情图表。

**Architecture:** QuantConnect Cloud 算法常驻订阅 `SPY` 与 `BTCUSD`，并按 Live Command 临时订阅最多一个搜索标的；它把当前活动标的的 OHLCV 写入固定图表序列。Atlas 的 FastAPI 后端负责认证、命令串行化、图表读取、SQLite 缓存、状态判断和 SSE；React 前端用 Lightweight Charts 渲染图表，并在浏览器内计算指标。云端行情算法不包含订单调用，未来模拟交易必须使用独立适配层。

**Tech Stack:** Python 3.12、uv、FastAPI、httpx、Pydantic、aiosqlite、exchange-calendars、pytest、TypeScript、React、Vite、Lightweight Charts 5.2、technicalindicators、Vitest、Playwright、QuantConnect LEAN Python API、QuantConnect REST API、SQLite、用户级 systemd、Git/GitHub 私有仓库。

---

## 固定边界与完成定义

- 数据源仅为 QuantConnect Cloud；不接入第三方行情 API。
- 第一版支持美国上市股票、ETF 和 Coinbase 加密货币；不支持 OTC、期权、期货或外汇。
- `SPY` 与 `BTCUSD` 始终保持订阅；搜索标的最多额外保留一个。
- 第一版不提供任何订单入口，不调用 `market_order`、`limit_order`、`set_holdings`、`liquidate` 或其他订单 API。
- 页面通常约每分钟更新，并显示实际采样间隔；不得使用“tick”“逐笔”“秒级实时”等误导性标签。
- Atlas 服务只绑定 `127.0.0.1`；不创建公网 DNS、TLS 或防火墙入口。
- 所有变更在隔离 worktree 的 `feature/live-chart-v1` 分支完成；测试全部通过后才允许合并 `main`。
- 创建 QuantConnect 云项目、编译和启动 Paper Trading live deployment 只能在离线测试与云端无交易回测通过之后进行。

已批准视觉基线不得重新解释：原始参考图为 `/tmp/codex-remote-attachments/01a0556a-bb4d-7c92-b017-32de77ca6077/36AC27CE-8229-46D1-84BB-A4EEC1B1DCD1/1-Photo-1.jpg`；已确认的 A 版移动端与宽版截图分别为 `/home/jingtianyu/.codex/visualizations/2026/08/31/01a0556a-bb4d-7c92-b017-32de77ca6077/live-market-dashboard-mobile.png` 和 `/home/jingtianyu/.codex/visualizations/2026/08/31/01a0556a-bb4d-7c92-b017-32de77ca6077/live-market-dashboard-wide.png`。实现时以这三张图的白色卡片、淡灰画布、红绿数值、蓝色图表与信息密度为视觉验收依据，不照搬参考图中的虚构账户盈亏数据。

## 文件结构

```text
quantconnect-live-chart/
├── pyproject.toml
├── uv.lock
├── src/qclive/
│   ├── __init__.py
│   ├── config.py
│   ├── logging.py
│   ├── domain.py
│   ├── symbols.py
│   ├── market_clock.py
│   ├── cache.py
│   ├── coordinator.py
│   ├── poller.py
│   ├── service.py
│   ├── quantconnect/
│   │   ├── auth.py
│   │   ├── client.py
│   │   └── models.py
│   └── api/
│       ├── app.py
│       ├── routes.py
│       └── sse.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── cloud/atlas-market-data/
│   ├── main.py
│   ├── protocol.py
│   └── project-manifest.json
├── scripts/
│   ├── sync_cloud_project.py
│   ├── run_read_only_backtest.py
│   ├── deploy_paper_live.py
│   ├── request_chart_switch.py
│   ├── verify_live_deployment.py
│   └── verify_no_secrets.py
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── api/
│   │   ├── chart/
│   │   ├── components/
│   │   ├── indicators/
│   │   ├── state/
│   │   ├── styles/
│   │   └── test/
│   └── tests/e2e/
├── deploy/
│   └── quantconnect-live-chart.service
└── docs/
    ├── operations.md
    └── superpowers/
```

## Execution setup: 建立隔离实施 worktree

- [ ] **确认设计仓库干净，并从 `main` 创建独立实施 worktree。**

Run:

```bash
cd /home/jingtianyu/projects/quantconnect-live-chart
git status --porcelain
git branch --show-current
git worktree add /home/jingtianyu/projects/quantconnect-live-chart-v1 \
  -b feature/live-chart-v1 main
cd /home/jingtianyu/projects/quantconnect-live-chart-v1
git status --short --branch
```

Expected: 第一条状态输出为空；新 worktree 位于 `feature/live-chart-v1`，起点包含本规格和本计划。

## Task 1: 建立 Python 项目、质量门和最小应用入口

**Files:**
- Create: `pyproject.toml`
- Create: `src/qclive/__init__.py`
- Create: `src/qclive/api/app.py`
- Create: `tests/unit/test_package.py`
- Modify: `.gitignore`

- [ ] **Step 1: 写入包导入和健康端点的失败测试。**

```python
# tests/unit/test_package.py
from fastapi.testclient import TestClient

from qclive import __version__
from qclive.api.app import create_app


def test_package_version_is_explicit() -> None:
    assert __version__ == "0.1.0"


def test_process_health_endpoint() -> None:
    response = TestClient(create_app()).get("/health/live")
    assert response.status_code == 200
    assert response.json() == {"status": "live"}
```

- [ ] **Step 2: 运行测试并确认包尚不存在。**

Run: `uv run --with pytest --with fastapi --with httpx pytest tests/unit/test_package.py -q`

Expected: FAIL，错误包含 `No module named 'qclive'`。

- [ ] **Step 3: 写入 Python 项目配置和最小入口。**

```toml
# pyproject.toml
[project]
name = "quantconnect-live-chart"
version = "0.1.0"
requires-python = ">=3.12,<3.13"
dependencies = [
  "aiosqlite>=0.21,<1",
  "exchange-calendars>=4.11,<5",
  "fastapi>=0.116,<1",
  "httpx>=0.28,<1",
  "pandas>=2.3,<3",
  "pydantic>=2.11,<3",
  "pydantic-settings>=2.10,<3",
  "tenacity>=9,<10",
  "uvicorn[standard]>=0.35,<1",
]

[dependency-groups]
dev = [
  "mypy>=1.17,<2",
  "pytest>=8.4,<9",
  "pytest-asyncio>=1.1,<2",
  "respx>=0.22,<1",
  "ruff>=0.12,<1",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/qclive"]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "ASYNC", "S"]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]

[tool.mypy]
python_version = "3.12"
strict = true
packages = ["qclive"]
```

```python
# src/qclive/__init__.py
__version__ = "0.1.0"
```

保留现有 `.gitignore`，并确认至少包含 `.venv/`、`data/`、`logs/`、`frontend/dist/`（现有全局 `dist/` 规则也可）、`node_modules/`、`playwright-report/`、`test-results/`、`*.db`、`*.db-shm` 与 `*.db-wal`。不得忽略 `cloud/`、`deploy/`、`docs/` 或 `frontend/package-lock.json`。

```python
# src/qclive/api/app.py
from fastapi import FastAPI


def create_app() -> FastAPI:
    app = FastAPI(title="Atlas Market Graph", version="0.1.0")

    @app.get("/health/live")
    async def health_live() -> dict[str, str]:
        return {"status": "live"}

    return app
```

- [ ] **Step 4: 解析依赖并运行最小质量门。**

Run:

```bash
uv lock
uv run pytest tests/unit/test_package.py -q
uv run ruff check src tests
uv run mypy src
```

Expected: `2 passed`；Ruff 与 mypy 均退出 0。

- [ ] **Step 5: 提交项目骨架。**

```bash
git add pyproject.toml uv.lock src/qclive tests/unit/test_package.py .gitignore
git commit -m "工程：建立后端项目骨架"
```

## Task 2: 固定领域模型、范围与粒度矩阵

**Files:**
- Create: `src/qclive/domain.py`
- Create: `tests/unit/test_domain.py`

- [ ] **Step 1: 写入资产、范围、粒度和 OHLCV 的失败测试。**

```python
# tests/unit/test_domain.py
import pytest
from pydantic import ValidationError

from qclive.domain import (
    AssetKind,
    BarResolution,
    ChartRange,
    OhlcvBar,
    SwitchRequest,
    allowed_resolutions,
    default_resolution,
)


def test_default_resolution_matrix() -> None:
    assert default_resolution(ChartRange.ONE_DAY) is BarResolution.M1
    assert default_resolution(ChartRange.FIVE_DAYS) is BarResolution.M5
    assert default_resolution(ChartRange.ONE_MONTH) is BarResolution.M30
    assert BarResolution.M1 not in allowed_resolutions(ChartRange.ONE_MONTH)


def test_ohlcv_rejects_invalid_high_low() -> None:
    with pytest.raises(ValidationError):
        OhlcvBar(time=1, open=10, high=9, low=8, close=9, volume=1)


def test_switch_request_is_read_only_market_selection() -> None:
    request = SwitchRequest(
        asset_kind=AssetKind.EQUITY,
        symbol="SPY",
        chart_range=ChartRange.ONE_DAY,
        resolution=BarResolution.M1,
    )
    assert request.symbol == "SPY"
```

- [ ] **Step 2: 运行测试并确认领域模块缺失。**

Run: `uv run pytest tests/unit/test_domain.py -q`

Expected: FAIL，错误包含 `No module named 'qclive.domain'`。

- [ ] **Step 3: 实现冻结的领域类型与兼容矩阵。**

```python
# src/qclive/domain.py
from __future__ import annotations

from enum import StrEnum

from pydantic import BaseModel, Field, model_validator


class AssetKind(StrEnum):
    EQUITY = "equity"
    CRYPTO = "crypto"


class ChartRange(StrEnum):
    ONE_DAY = "1D"
    FIVE_DAYS = "5D"
    ONE_MONTH = "1M"


class BarResolution(StrEnum):
    M1 = "1m"
    M5 = "5m"
    M15 = "15m"
    M30 = "30m"
    H1 = "1h"
    D1 = "1D"


class DashboardStatus(StrEnum):
    LIVE = "LIVE"
    LOADING = "LOADING"
    STALE = "STALE"
    MARKET_CLOSED = "MARKET_CLOSED"
    DISCONNECTED = "DISCONNECTED"
    ERROR = "ERROR"


DEFAULT_RESOLUTION = {
    ChartRange.ONE_DAY: BarResolution.M1,
    ChartRange.FIVE_DAYS: BarResolution.M5,
    ChartRange.ONE_MONTH: BarResolution.M30,
}
ALLOWED_RESOLUTIONS = {
    ChartRange.ONE_DAY: frozenset(
        {BarResolution.M1, BarResolution.M5, BarResolution.M15, BarResolution.M30, BarResolution.H1}
    ),
    ChartRange.FIVE_DAYS: frozenset(
        {BarResolution.M5, BarResolution.M15, BarResolution.M30, BarResolution.H1, BarResolution.D1}
    ),
    ChartRange.ONE_MONTH: frozenset(
        {BarResolution.M30, BarResolution.H1, BarResolution.D1}
    ),
}


def default_resolution(chart_range: ChartRange) -> BarResolution:
    return DEFAULT_RESOLUTION[chart_range]


def allowed_resolutions(chart_range: ChartRange) -> frozenset[BarResolution]:
    return ALLOWED_RESOLUTIONS[chart_range]


class SwitchRequest(BaseModel):
    asset_kind: AssetKind
    symbol: str = Field(min_length=1, max_length=20)
    chart_range: ChartRange
    resolution: BarResolution

    @model_validator(mode="after")
    def validate_matrix(self) -> "SwitchRequest":
        if self.resolution not in allowed_resolutions(self.chart_range):
            raise ValueError("resolution is not allowed for chart range")
        return self


class OhlcvBar(BaseModel):
    time: int = Field(gt=0)
    open: float
    high: float
    low: float
    close: float
    volume: float = Field(ge=0)

    @model_validator(mode="after")
    def validate_prices(self) -> "OhlcvBar":
        if self.high < max(self.open, self.close) or self.low > min(self.open, self.close):
            raise ValueError("OHLC values are inconsistent")
        return self
```

- [ ] **Step 4: 运行领域测试与类型检查。**

Run: `uv run pytest tests/unit/test_domain.py -q && uv run mypy src`

Expected: `3 passed`，mypy 退出 0。

- [ ] **Step 5: 提交领域合同。**

```bash
git add src/qclive/domain.py tests/unit/test_domain.py
git commit -m "功能：固定行情领域合同"
```

## Task 3: 实现凭据隔离与 QuantConnect REST 客户端

**Files:**
- Create: `src/qclive/config.py`
- Create: `src/qclive/logging.py`
- Create: `src/qclive/quantconnect/auth.py`
- Create: `src/qclive/quantconnect/client.py`
- Create: `src/qclive/quantconnect/models.py`
- Create: `tests/unit/test_quantconnect_auth.py`
- Create: `tests/integration/test_quantconnect_client.py`

- [ ] **Step 1: 写入凭据加载和签名的失败测试。**

```python
# tests/unit/test_quantconnect_auth.py
import base64
import hashlib
import json
from pathlib import Path

from qclive.quantconnect.auth import QuantConnectCredentials, build_headers, load_credentials


def test_load_credentials_and_build_headers(tmp_path: Path) -> None:
    path = tmp_path / "credentials"
    path.write_text(
        json.dumps({"user-id": 123456, "api-token": "secret-token"}),
        encoding="utf-8",
    )
    credentials = load_credentials(path)
    headers = build_headers(credentials, timestamp=1_700_000_000)
    digest = hashlib.sha256(b"secret-token:1700000000").hexdigest()
    expected = base64.b64encode(f"123456:{digest}".encode()).decode()
    assert credentials == QuantConnectCredentials("123456", "secret-token")
    assert headers["Authorization"] == f"Basic {expected}"
    assert "secret-token" not in json.dumps(headers)
```

- [ ] **Step 2: 运行测试并确认认证模块缺失。**

Run: `uv run pytest tests/unit/test_quantconnect_auth.py -q`

Expected: FAIL，错误包含 `No module named 'qclive.quantconnect.auth'`。

- [ ] **Step 3: 实现凭据读取、签名与统一错误模型。**

```python
# src/qclive/quantconnect/auth.py
from __future__ import annotations

import base64
import hashlib
import json
import time
from dataclasses import dataclass
from pathlib import Path


class QuantConnectAuthError(RuntimeError):
    """QuantConnect credentials are missing or malformed."""


@dataclass(frozen=True)
class QuantConnectCredentials:
    user_id: str
    api_token: str


def load_credentials(path: Path) -> QuantConnectCredentials:
    try:
        payload = json.loads(path.read_text(encoding="utf-8"))
        user_id = str(payload["user-id"])
        token = str(payload["api-token"])
    except (OSError, KeyError, TypeError, ValueError, json.JSONDecodeError) as error:
        raise QuantConnectAuthError("QuantConnect credentials are unavailable") from error
    if not user_id or not token:
        raise QuantConnectAuthError("QuantConnect credentials are incomplete")
    return QuantConnectCredentials(user_id, token)


def build_headers(
    credentials: QuantConnectCredentials, *, timestamp: int | None = None
) -> dict[str, str]:
    current = str(int(time.time()) if timestamp is None else timestamp)
    digest = hashlib.sha256(
        f"{credentials.api_token}:{current}".encode("utf-8")
    ).hexdigest()
    encoded = base64.b64encode(
        f"{credentials.user_id}:{digest}".encode("utf-8")
    ).decode("ascii")
    return {
        "Authorization": f"Basic {encoded}",
        "Timestamp": current,
        "User-Agent": "atlas-market-graph/0.1.0",
    }
```

```python
# src/qclive/quantconnect/models.py
from pydantic import BaseModel, Field


class LiveDetails(BaseModel):
    project_id: int
    deploy_id: str
    status: str
    runtime_statistics: dict[str, str] = Field(default_factory=dict)


class LoadingResult(BaseModel):
    status: str
    progress: float = 0


class QuantConnectApiError(RuntimeError):
    """A QuantConnect request failed or returned success=false."""
```

```python
# src/qclive/config.py
from pathlib import Path

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="QCLIVE_")

    credentials_path: Path = Path.home() / ".lean" / "credentials"
    data_dir: Path = Path("data")
    deployment_state_path: Path = Path("data/deployment.json")
    static_dir: Path = Path("frontend/dist")
    host: str = "127.0.0.1"
    port: int = 8765
```

- [ ] **Step 4: 写入 HTTP 合同测试，固定所需端点和错误脱敏。**

```python
# tests/integration/test_quantconnect_client.py
import httpx
import pytest
import respx

from qclive.quantconnect.auth import QuantConnectCredentials
from qclive.quantconnect.client import QuantConnectClient
from qclive.quantconnect.models import QuantConnectApiError


@respx.mock
@pytest.mark.asyncio
async def test_send_command_uses_expected_payload() -> None:
    route = respx.post("https://www.quantconnect.com/api/v2/live/commands/create").mock(
        return_value=httpx.Response(200, json={"success": True})
    )
    async with QuantConnectClient(QuantConnectCredentials("1", "secret")) as client:
        await client.send_command(42, {"action": "switch", "symbol": "SPY"})
    assert route.calls[0].request.content == (
        b'{"projectId":42,"command":{"action":"switch","symbol":"SPY"}}'
    )


@respx.mock
@pytest.mark.asyncio
async def test_api_error_never_contains_token() -> None:
    respx.post("https://www.quantconnect.com/api/v2/live/read").mock(
        return_value=httpx.Response(401, json={"success": False, "errors": ["denied"]})
    )
    async with QuantConnectClient(QuantConnectCredentials("1", "secret")) as client:
        with pytest.raises(QuantConnectApiError) as captured:
            await client.read_live(42)
    assert "secret" not in str(captured.value)
```

在 `tests/unit/test_quantconnect_auth.py` 另加日志测试：构造含 `Authorization`、`Cookie`、`api-token`、签名摘要和原始 token 的结构化字段与消息，经 `RedactingFilter` 后逐项断言只剩 `[REDACTED]`，同时保留非敏感 endpoint/status 字段。

- [ ] **Step 5: 实现仅包含已批准端点的异步客户端。**

```python
# src/qclive/quantconnect/client.py
from __future__ import annotations

from collections.abc import Mapping
from typing import Any

import httpx

from qclive.quantconnect.auth import QuantConnectCredentials, build_headers
from qclive.quantconnect.models import QuantConnectApiError


class QuantConnectClient:
    BASE_URL = "https://www.quantconnect.com/api/v2"

    def __init__(
        self,
        credentials: QuantConnectCredentials,
        *,
        transport: httpx.AsyncBaseTransport | None = None,
    ) -> None:
        self._credentials = credentials
        self._client = httpx.AsyncClient(
            base_url=self.BASE_URL,
            timeout=httpx.Timeout(60),
            transport=transport,
        )

    async def __aenter__(self) -> "QuantConnectClient":
        return self

    async def __aexit__(self, *_: object) -> None:
        await self._client.aclose()

    async def _post(self, endpoint: str, payload: Mapping[str, Any]) -> dict[str, Any]:
        response = await self._client.post(
            endpoint, headers=build_headers(self._credentials), json=dict(payload)
        )
        try:
            body = response.json()
        except ValueError as error:
            raise QuantConnectApiError(f"invalid JSON from {endpoint}") from error
        if response.is_error or not isinstance(body, dict) or not body.get("success"):
            raise QuantConnectApiError(
                f"QuantConnect rejected {endpoint}: HTTP {response.status_code}"
            )
        return body

    async def authenticate(self) -> dict[str, Any]:
        return await self._post("/authenticate", {})

    async def read_live(self, project_id: int) -> dict[str, Any]:
        return await self._post("/live/read", {"projectId": project_id})

    async def list_live(self, project_id: int) -> dict[str, Any]:
        return await self._post(
            "/live/list", {"projectId": project_id, "status": "Running"}
        )

    async def send_command(self, project_id: int, command: dict[str, Any]) -> None:
        await self._post(
            "/live/commands/create", {"projectId": project_id, "command": command}
        )

    async def read_chart(
        self, project_id: int, name: str, start: int, end: int, count: int
    ) -> dict[str, Any]:
        return await self._post(
            "/live/chart/read",
            {
                "projectId": project_id,
                "name": name,
                "count": count,
                "start": start,
                "end": end,
            },
        )
```

`src/qclive/logging.py` 提供递归 `redact(value, secrets)` 与 `RedactingFilter`：字段名大小写归一后命中 `authorization`、`cookie`、`api-token`、`token`、`password`、`signature` 就整值替换；字符串内的已加载 token、`Basic` authorization 值和 64 位签名摘要也替换。`create_app` 在创建 QuantConnect 客户端前，把同一个 filter 安装到应用、uvicorn access/error 与 httpx logger；请求/响应正文不进入 INFO 日志。

- [ ] **Step 6: 运行认证和客户端测试。**

Run: `uv run pytest tests/unit/test_quantconnect_auth.py tests/integration/test_quantconnect_client.py -q`

Expected: 至少 4 个认证、日志脱敏与客户端合同测试通过。

- [ ] **Step 7: 提交 REST 边界。**

```bash
git add src/qclive/config.py src/qclive/logging.py src/qclive/quantconnect tests/unit/test_quantconnect_auth.py tests/integration/test_quantconnect_client.py
git commit -m "功能：增加 QuantConnect 安全客户端"
```

## Task 4: 实现代码规范化、市场阶段与 stale 判定

**Files:**
- Create: `src/qclive/symbols.py`
- Create: `src/qclive/market_clock.py`
- Create: `tests/unit/test_symbols.py`
- Create: `tests/unit/test_market_clock.py`

- [ ] **Step 1: 写入输入规范化与市场阶段测试。**

```python
# tests/unit/test_symbols.py
import pytest

from qclive.domain import AssetKind
from qclive.symbols import SymbolValidationError, normalize_symbol


def test_equity_and_crypto_normalization() -> None:
    assert normalize_symbol(AssetKind.EQUITY, " spy ") == "SPY"
    assert normalize_symbol(AssetKind.CRYPTO, "btc-usd") == "BTCUSD"


@pytest.mark.parametrize("value", ["", "../../etc/passwd", "SPY<script>", "BTCUSDT"])
def test_invalid_or_out_of_scope_symbols_are_rejected(value: str) -> None:
    kind = AssetKind.CRYPTO if value == "BTCUSDT" else AssetKind.EQUITY
    with pytest.raises(SymbolValidationError):
        normalize_symbol(kind, value)
```

```python
# tests/unit/test_market_clock.py
from datetime import datetime, timezone

from qclive.domain import AssetKind, BarResolution, DashboardStatus
from qclive.market_clock import classify_status, market_phase


def test_crypto_is_open_on_weekend() -> None:
    instant = datetime(2026, 8, 30, 12, tzinfo=timezone.utc)
    assert market_phase(AssetKind.CRYPTO, instant) == "OPEN_24_7"


def test_equity_extended_session() -> None:
    instant = datetime(2026, 8, 31, 12, tzinfo=timezone.utc)
    assert market_phase(AssetKind.EQUITY, instant) == "PRE_MARKET"


def test_stale_requires_open_market_and_old_bar() -> None:
    now = datetime(2026, 8, 31, 15, tzinfo=timezone.utc)
    assert classify_status(
        AssetKind.EQUITY, BarResolution.M1, now, now, now
    ) is DashboardStatus.LIVE
```

- [ ] **Step 2: 运行测试并确认模块缺失。**

Run: `uv run pytest tests/unit/test_symbols.py tests/unit/test_market_clock.py -q`

Expected: FAIL，错误指向缺少 `symbols` 与 `market_clock`。

- [ ] **Step 3: 实现两阶段验证和市场时钟。**

```python
# src/qclive/symbols.py
import re

from qclive.domain import AssetKind


EQUITY_PATTERN = re.compile(r"^[A-Z][A-Z0-9.\-]{0,9}$")
CRYPTO_PATTERN = re.compile(r"^[A-Z0-9]{3,15}USD$")


class SymbolValidationError(ValueError):
    """The requested symbol is syntactically invalid or outside product scope."""


def normalize_symbol(kind: AssetKind, value: str) -> str:
    normalized = value.strip().upper()
    if kind is AssetKind.CRYPTO:
        normalized = normalized.replace("-", "").replace("/", "")
        if not CRYPTO_PATTERN.fullmatch(normalized) or normalized.endswith("USDT"):
            raise SymbolValidationError("only Coinbase USD pairs are supported")
        return normalized
    if not EQUITY_PATTERN.fullmatch(normalized):
        raise SymbolValidationError("invalid US equity or ETF symbol")
    return normalized
```

```python
# src/qclive/market_clock.py
from datetime import datetime, time, timedelta
from zoneinfo import ZoneInfo

import exchange_calendars as xcals
import pandas as pd

from qclive.domain import AssetKind, BarResolution, DashboardStatus


NEW_YORK = ZoneInfo("America/New_York")
XNYS = xcals.get_calendar("XNYS")
RESOLUTION_SECONDS = {
    BarResolution.M1: 60,
    BarResolution.M5: 300,
    BarResolution.M15: 900,
    BarResolution.M30: 1_800,
    BarResolution.H1: 3_600,
    BarResolution.D1: 86_400,
}


def market_phase(kind: AssetKind, instant: datetime) -> str:
    if kind is AssetKind.CRYPTO:
        return "OPEN_24_7"
    local = instant.astimezone(NEW_YORK)
    if not XNYS.is_session(pd.Timestamp(local.date())):
        return "CLOSED"
    clock = local.timetz().replace(tzinfo=None)
    if time(4) <= clock < time(9, 30):
        return "PRE_MARKET"
    if time(9, 30) <= clock < time(16):
        return "REGULAR"
    if time(16) <= clock < time(20):
        return "POST_MARKET"
    return "CLOSED"


def classify_status(
    kind: AssetKind,
    resolution: BarResolution,
    now: datetime,
    last_bar: datetime | None,
    last_api_success: datetime | None,
) -> DashboardStatus:
    if market_phase(kind, now) == "CLOSED":
        return DashboardStatus.MARKET_CLOSED
    if last_bar is None or last_api_success is None:
        return DashboardStatus.LOADING
    bar_stale_after = timedelta(
        seconds=max(3 * RESOLUTION_SECONDS[resolution], 180)
    )
    api_stale_after = timedelta(seconds=180)
    return (
        DashboardStatus.STALE
        if (
            now - last_bar > bar_stale_after
            or now - last_api_success > api_stale_after
        )
        else DashboardStatus.LIVE
    )
```

- [ ] **Step 4: 补齐节假日、夏令时和边界分钟测试后运行。**

Run: `uv run pytest tests/unit/test_symbols.py tests/unit/test_market_clock.py -q`

Expected: 所有参数化用例通过，且 2026-09-07 Labor Day 被判定为 `MARKET_CLOSED`。

- [ ] **Step 5: 提交代码与市场时钟。**

```bash
git add src/qclive/symbols.py src/qclive/market_clock.py tests/unit/test_symbols.py tests/unit/test_market_clock.py
git commit -m "功能：增加代码验证与市场时钟"
```

## Task 5: 实现 SQLite 图表缓存

**Files:**
- Create: `src/qclive/cache.py`
- Create: `tests/unit/test_cache.py`

- [ ] **Step 1: 写入幂等写入、顺序读取和 31 天清理测试。**

```python
# tests/unit/test_cache.py
from pathlib import Path

import pytest

from qclive.cache import MarketCache
from qclive.domain import AssetKind, BarResolution, OhlcvBar


@pytest.mark.asyncio
async def test_upsert_is_idempotent_and_sorted(tmp_path: Path) -> None:
    cache = MarketCache(tmp_path / "market.db")
    await cache.initialize()
    bars = [
        OhlcvBar(time=2, open=2, high=3, low=1, close=2, volume=20),
        OhlcvBar(time=1, open=1, high=2, low=1, close=2, volume=10),
    ]
    await cache.upsert_bars(AssetKind.EQUITY, "SPY", BarResolution.M1, bars)
    await cache.upsert_bars(AssetKind.EQUITY, "SPY", BarResolution.M1, bars)
    loaded = await cache.read_bars(
        AssetKind.EQUITY, "SPY", BarResolution.M1, start=0, end=3
    )
    assert [bar.time for bar in loaded] == [1, 2]
```

- [ ] **Step 2: 运行测试并确认缓存模块缺失。**

Run: `uv run pytest tests/unit/test_cache.py -q`

Expected: FAIL，错误包含 `No module named 'qclive.cache'`。

- [ ] **Step 3: 实现固定 schema 和事务写入。**

```sql
CREATE TABLE IF NOT EXISTS bars (
  asset_kind TEXT NOT NULL,
  symbol TEXT NOT NULL,
  resolution TEXT NOT NULL,
  time INTEGER NOT NULL,
  open REAL NOT NULL,
  high REAL NOT NULL,
  low REAL NOT NULL,
  close REAL NOT NULL,
  volume REAL NOT NULL,
  PRIMARY KEY (asset_kind, symbol, resolution, time)
);
CREATE INDEX IF NOT EXISTS bars_lookup
ON bars(asset_kind, symbol, resolution, time);

CREATE TABLE IF NOT EXISTS runtime_state (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at INTEGER NOT NULL
);
```

```python
# src/qclive/cache.py
from pathlib import Path

import aiosqlite

from qclive.domain import AssetKind, BarResolution, OhlcvBar


SCHEMA = """
CREATE TABLE IF NOT EXISTS bars (
  asset_kind TEXT NOT NULL, symbol TEXT NOT NULL, resolution TEXT NOT NULL,
  time INTEGER NOT NULL, open REAL NOT NULL, high REAL NOT NULL,
  low REAL NOT NULL, close REAL NOT NULL, volume REAL NOT NULL,
  PRIMARY KEY (asset_kind, symbol, resolution, time)
);
CREATE INDEX IF NOT EXISTS bars_lookup
ON bars(asset_kind, symbol, resolution, time);
CREATE TABLE IF NOT EXISTS runtime_state (
  key TEXT PRIMARY KEY, value TEXT NOT NULL, updated_at INTEGER NOT NULL
);
"""


class MarketCache:
    def __init__(self, path: Path) -> None:
        self._path = path

    async def initialize(self) -> None:
        self._path.parent.mkdir(parents=True, exist_ok=True)
        async with aiosqlite.connect(self._path) as connection:
            await connection.executescript(SCHEMA)
            await connection.commit()

    async def upsert_bars(
        self,
        kind: AssetKind,
        symbol: str,
        resolution: BarResolution,
        bars: list[OhlcvBar],
    ) -> None:
        rows = [
            (
                kind.value, symbol, resolution.value, bar.time, bar.open,
                bar.high, bar.low, bar.close, bar.volume,
            )
            for bar in bars
        ]
        async with aiosqlite.connect(self._path) as connection:
            await connection.executemany(
                """
                INSERT INTO bars VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT(asset_kind, symbol, resolution, time) DO UPDATE SET
                  open=excluded.open, high=excluded.high, low=excluded.low,
                  close=excluded.close, volume=excluded.volume
                """,
                rows,
            )
            await connection.commit()

    async def read_bars(
        self,
        kind: AssetKind,
        symbol: str,
        resolution: BarResolution,
        *,
        start: int,
        end: int,
    ) -> list[OhlcvBar]:
        async with aiosqlite.connect(self._path) as connection:
            cursor = await connection.execute(
                """
                SELECT time, open, high, low, close, volume FROM bars
                WHERE asset_kind=? AND symbol=? AND resolution=?
                  AND time>=? AND time<=?
                ORDER BY time ASC
                """,
                (kind.value, symbol, resolution.value, start, end),
            )
            rows = await cursor.fetchall()
        return [
            OhlcvBar(
                time=row[0], open=row[1], high=row[2], low=row[3],
                close=row[4], volume=row[5],
            )
            for row in rows
        ]

    async def prune(self, cutoff: int) -> int:
        async with aiosqlite.connect(self._path) as connection:
            cursor = await connection.execute(
                "DELETE FROM bars WHERE time < ?", (cutoff,)
            )
            await connection.commit()
            return cursor.rowcount
```

`MarketCache.initialize` 创建目录与 schema；`prune` 只删除早于调用方传入 cutoff 的 bar，不删除 runtime state。测试另加 `PRAGMA integrity_check`，并断言返回 `ok`。

- [ ] **Step 4: 运行缓存测试与 SQLite 完整性检查。**

Run: `uv run pytest tests/unit/test_cache.py -q`

Expected: 幂等、隔离不同资产/粒度、清理边界和 `PRAGMA integrity_check` 全部通过。

- [ ] **Step 5: 提交缓存实现。**

```bash
git add src/qclive/cache.py tests/unit/test_cache.py
git commit -m "功能：增加行情图表缓存"
```

## Task 6: 实现命令串行化、确认与最后请求优先

**Files:**
- Create: `src/qclive/coordinator.py`
- Create: `tests/unit/test_coordinator.py`

- [ ] **Step 1: 写入命令 payload、确认条件和队列折叠测试。**

```python
# tests/unit/test_coordinator.py
from unittest.mock import AsyncMock

import pytest

from qclive.coordinator import CommandCoordinator
from qclive.domain import AssetKind, BarResolution, ChartRange, SwitchRequest


@pytest.mark.asyncio
async def test_command_is_not_confirmed_until_runtime_request_matches() -> None:
    client = AsyncMock()
    client.read_live.side_effect = [
        {"runtimeStatistics": {"Dashboard Request": "old", "Dashboard Status": "LIVE"}},
        {"runtimeStatistics": {"Dashboard Request": "req-1", "Dashboard Status": "LIVE"}},
    ]
    coordinator = CommandCoordinator(client, project_id=42, sleep=AsyncMock())
    request = SwitchRequest(
        asset_kind=AssetKind.EQUITY,
        symbol="SPY",
        chart_range=ChartRange.ONE_DAY,
        resolution=BarResolution.M1,
    )
    result = await coordinator.switch(request, request_id="req-1")
    assert result.request_id == "req-1"
    assert client.send_command.await_args.args[1]["action"] == "switch"
```

- [ ] **Step 2: 运行测试并确认协调器缺失。**

Run: `uv run pytest tests/unit/test_coordinator.py -q`

Expected: FAIL，错误包含 `No module named 'qclive.coordinator'`。

- [ ] **Step 3: 实现单活动命令协调器。**

```python
# src/qclive/coordinator.py
from __future__ import annotations

import asyncio
import random
from collections.abc import Awaitable, Callable
from dataclasses import dataclass
from typing import Protocol

from qclive.domain import SwitchRequest


class CommandClient(Protocol):
    async def send_command(
        self, project_id: int, command: dict[str, object]
    ) -> None:
        raise NotImplementedError

    async def read_live(self, project_id: int) -> dict[str, object]:
        raise NotImplementedError


@dataclass(frozen=True)
class SwitchConfirmation:
    request_id: str
    symbol: str


class CommandCoordinator:
    def __init__(
        self,
        client: CommandClient,
        project_id: int,
        sleep=asyncio.sleep,
        jitter=random.uniform,
    ) -> None:
        self._client = client
        self._project_id = project_id
        self._sleep = sleep
        self._jitter = jitter
        self._switch_lock = asyncio.Lock()
        self._pending_lock = asyncio.Lock()
        self._pending: tuple[SwitchRequest, str] | None = None
        self._pending_event = asyncio.Event()

    async def submit(self, request: SwitchRequest, *, request_id: str) -> str | None:
        async with self._pending_lock:
            superseded = self._pending[1] if self._pending is not None else None
            self._pending = (request, request_id)
            self._pending_event.set()
            return superseded

    async def run(
        self,
        on_complete: Callable[
            [str, SwitchConfirmation | Exception], Awaitable[None]
        ],
    ) -> None:
        while True:
            await self._pending_event.wait()
            async with self._pending_lock:
                pending = self._pending
                self._pending = None
                self._pending_event.clear()
            if pending is None:
                continue
            request, request_id = pending
            try:
                outcome: SwitchConfirmation | Exception = await self.switch(
                    request, request_id=request_id
                )
            except Exception as error:
                outcome = error
            await on_complete(request_id, outcome)

    async def switch(
        self, request: SwitchRequest, *, request_id: str
    ) -> SwitchConfirmation:
        command: dict[str, object] = {
            "action": "switch",
            "requestId": request_id,
            "assetKind": request.asset_kind.value,
            "symbol": request.symbol,
            "range": request.chart_range.value,
            "resolution": request.resolution.value,
        }
        async with self._switch_lock:
            await self._client.send_command(self._project_id, command)
            for retry_delay in (1, 2, 4, 8, 16, 32, 60, 57):
                details = await self._client.read_live(self._project_id)
                statistics = details.get("runtimeStatistics", {})
                if isinstance(statistics, dict) and (
                    statistics.get("Dashboard Request") == request_id
                    and statistics.get("Dashboard Status") == "LIVE"
                ):
                    return SwitchConfirmation(request_id, request.symbol)
                if isinstance(statistics, dict) and (
                    statistics.get("Dashboard Request") == request_id
                    and statistics.get("Dashboard Status") == "ERROR"
                ):
                    raise RuntimeError(str(statistics.get("Dashboard Error", "switch failed")))
                await self._sleep(
                    min(60.0, retry_delay + self._jitter(0.0, 1.0))
                )
        raise TimeoutError("QuantConnect switch confirmation exhausted retry budget")
```

- [ ] **Step 4: 增加并发测试，确认后到请求替代尚未开始的请求，而活动请求不中断。**

测试启动 `run` worker：先让请求 A 停在云端确认轮询中，再依次 `submit` B、C；断言 A 完成后只发送 C，B 的 request ID 由 `submit` 作为 `superseded` 返回。另断言 A 的异常经 `on_complete` 上报，但 worker 继续处理 C。测试结束时取消并等待 worker，避免遗留异步任务。

Run: `uv run pytest tests/unit/test_coordinator.py -q`

Expected: 确认匹配、云端 ERROR、关闭 jitter 后约 180 秒重试预算、并发折叠四类用例全部通过。

- [ ] **Step 5: 提交命令协调器。**

```bash
git add src/qclive/coordinator.py tests/unit/test_coordinator.py
git commit -m "功能：增加行情切换协调器"
```

## Task 7: 实现图表解析、轮询状态和 SSE

**Files:**
- Create: `src/qclive/poller.py`
- Create: `src/qclive/api/sse.py`
- Create: `tests/unit/test_poller.py`
- Create: `tests/integration/test_sse.py`

- [ ] **Step 1: 写入 QuantConnect candle/volume 解析与 loading 测试。**

```python
# tests/unit/test_poller.py
from qclive.poller import parse_chart


def test_parse_chart_merges_candle_and_volume_by_time() -> None:
    payload = {
        "success": True,
        "chart": {
            "series": {
                "Price": {"values": [{"x": 100, "o": 1, "h": 3, "l": 1, "c": 2}]},
                "Volume": {"values": [{"x": 100, "y": 20}]},
            }
        },
    }
    bars = parse_chart(payload)
    assert bars[0].model_dump() == {
        "time": 100,
        "open": 1.0,
        "high": 3.0,
        "low": 1.0,
        "close": 2.0,
        "volume": 20.0,
    }
```

- [ ] **Step 2: 运行测试并确认 poller 缺失。**

Run: `uv run pytest tests/unit/test_poller.py -q`

Expected: FAIL，错误包含 `No module named 'qclive.poller'`。

- [ ] **Step 3: 实现严格解析、去重和状态快照。**

```python
# src/qclive/poller.py
from __future__ import annotations

from dataclasses import dataclass

from qclive.domain import (
    AssetKind,
    BarResolution,
    ChartRange,
    DashboardStatus,
    OhlcvBar,
)


def parse_chart(payload: dict[str, object]) -> list[OhlcvBar]:
    chart = payload.get("chart")
    if not isinstance(chart, dict):
        raise ValueError("chart payload is missing")
    series = chart.get("series")
    if not isinstance(series, dict):
        raise ValueError("chart series are missing")
    price = series.get("Price")
    volume = series.get("Volume")
    if not isinstance(price, dict) or not isinstance(volume, dict):
        raise ValueError("Price and Volume series are required")
    price_values = price.get("values")
    volume_values = volume.get("values")
    if not isinstance(price_values, list) or not isinstance(volume_values, list):
        raise ValueError("chart values must be arrays")
    volume_by_time = {
        int(point["x"]): float(point["y"])
        for point in volume_values
        if isinstance(point, dict) and "x" in point and "y" in point
    }
    bars = []
    for point in price_values:
        if not isinstance(point, dict):
            raise ValueError("candle point must be an object")
        timestamp = int(point["x"])
        bars.append(
            OhlcvBar(
                time=timestamp,
                open=float(point["o"]),
                high=float(point["h"]),
                low=float(point["l"]),
                close=float(point["c"]),
                volume=volume_by_time.get(timestamp, 0.0),
            )
        )
    return sorted({bar.time: bar for bar in bars}.values(), key=lambda bar: bar.time)


@dataclass(frozen=True)
class PollSnapshot:
    status: DashboardStatus
    request_id: str
    asset_kind: AssetKind
    symbol: str
    chart_range: ChartRange
    resolution: BarResolution
    market_phase: str
    bars: tuple[OhlcvBar, ...]
    last_api_success: int | None
    error: str | None
```

`parse_chart` 时间戳不匹配时成交量置 0，并记录结构化 warning。`ChartPoller` 正常时每 15 秒读取 `live/read`，只在算法状态为 `Running` 或 `History` 时读取图表；`loading` 结果保持 `LOADING`。失败重试间隔固定为 `min(60, 2 ** failure_count) + uniform(0, 1)` 秒；第一次和第二次失败保留最后快照并标为 `STALE`，第三次转为 `DISCONNECTED`；成功后失败计数归零并重置 15 秒周期。`classify_status` 同时检查最后完成 bar 和 `last_api_success`，任一超过允许窗口即为 `STALE`。

- [ ] **Step 4: 实现 SSE 编码和慢客户端隔离。**

```python
# src/qclive/api/sse.py
import json
from collections.abc import AsyncIterator


def encode_sse(event: str, data: dict[str, object]) -> bytes:
    payload = json.dumps(data, separators=(",", ":"), ensure_ascii=False)
    return f"event: {event}\ndata: {payload}\n\n".encode("utf-8")


async def heartbeat(interval_seconds: float) -> AsyncIterator[bytes]:
    import asyncio

    while True:
        await asyncio.sleep(interval_seconds)
        yield b": keep-alive\n\n"
```

- [ ] **Step 5: 运行 poller 与 SSE 测试。**

Run: `uv run pytest tests/unit/test_poller.py tests/integration/test_sse.py -q`

Expected: 图表解析、`loading`、三次断线、恢复、心跳和客户端取消全部通过。

- [ ] **Step 6: 提交读取与推送层。**

```bash
git add src/qclive/poller.py src/qclive/api/sse.py tests/unit/test_poller.py tests/integration/test_sse.py
git commit -m "功能：增加图表轮询与实时推送"
```

## Task 8: 组装 FastAPI 只读服务

**Files:**
- Create: `src/qclive/service.py`
- Create: `src/qclive/api/routes.py`
- Modify: `src/qclive/api/app.py`
- Create: `tests/fixtures/__init__.py`
- Create: `tests/fixtures/fake_service.py`
- Create: `tests/integration/test_api.py`

- [ ] **Step 1: 写入 snapshot、switch、stream 和 readiness 合同测试。**

```python
# tests/integration/test_api.py
from fastapi.testclient import TestClient

from qclive.api.app import create_app
from tests.fixtures.fake_service import FakeMarketService


def test_snapshot_and_switch_contract() -> None:
    service = FakeMarketService()
    client = TestClient(create_app(service=service))
    response = client.post(
        "/api/v1/switch",
        json={
            "asset_kind": "equity",
            "symbol": "spy",
            "chart_range": "1D",
            "resolution": "1m",
        },
    )
    assert response.status_code == 202
    assert response.json()["symbol"] == "SPY"
    snapshot = client.get("/api/v1/snapshot")
    assert snapshot.status_code == 200
    assert "bars" in snapshot.json()
```

- [ ] **Step 2: 运行测试并确认路由尚未装配。**

Run: `uv run pytest tests/integration/test_api.py -q`

Expected: FAIL，错误指向缺少 `qclive.service` 或 404。

- [ ] **Step 3: 实现应用服务接口与依赖注入。**

```python
@dataclass(frozen=True)
class SwitchAccepted:
    request_id: str
    symbol: str
    superseded_request_id: str | None = None


class MarketService(Protocol):
    async def snapshot(self) -> PollSnapshot:
        raise NotImplementedError

    async def request_switch(self, request: SwitchRequest) -> SwitchAccepted:
        raise NotImplementedError

    async def subscribe(self) -> AsyncIterator[PollSnapshot]:
        raise NotImplementedError

    async def ready(self) -> bool:
        raise NotImplementedError
```

生产 `MarketService` 组合 `MarketCache`、`CommandCoordinator` 与 `ChartPoller`；应用 lifespan 启动且只启动一个 coordinator worker 与一个 poller task，关闭时取消并等待两者后再关闭 HTTP/SQLite 资源。`request_switch` 由服务端生成 UUID request ID，调用 `submit` 后立即返回 202；若替代了尚未开始的请求，在 `superseded_request_id` 返回旧 ID 并发布最新 `LOADING` 快照。`FakeMarketService` 只存在于 `tests/fixtures/fake_service.py`，不进入生产包。上述代码块同时从 `collections.abc` 导入 `AsyncIterator`、从 `dataclasses` 导入 `dataclass`、从 `typing` 导入 `Protocol`。

- [ ] **Step 4: 实现固定 HTTP 行为。**

- `GET /health/live`：进程存活即 200。
- `GET /health/ready`：数据库可读、QuantConnect 认证成功且项目有运行 deployment 时 200，否则 503。
- `GET /api/v1/snapshot`：返回最后成功快照。
- `POST /api/v1/switch`：规范化后返回 202、request ID 和可空的 superseded request ID；无效输入返回 422；后端未 ready 时返回 503。
- `GET /api/v1/stream`：`text/event-stream`，发送 `snapshot`、`status`、`error` 与心跳。
- 所有响应设置 `Cache-Control: no-store`；CORS 中间件不启用。
- 在注册所有 `/api` 与 `/health` 路由后，把 `Settings.static_dir` 以 `StaticFiles(html=True)` 挂载到 `/`；静态目录不存在时启动失败并报告可读错误，避免返回空白页面。

`GET /api/v1/snapshot` 与 SSE `snapshot` 事件共享以下 snake_case 合同，不另造前端专用字段：`status`、`request_id`、`asset_kind`、`symbol`、`chart_range`、`resolution`、`market_phase`、`bars`、`last_api_success`、`error`。数据源固定由前端显示为 `QuantConnect`，实际采样间隔由 `resolution` 映射；最后完成 bar 时间从 `bars` 末项获得。

- [ ] **Step 5: 运行 API 合同和完整后端测试。**

Run: `uv run pytest tests/unit tests/integration -q && uv run ruff check src tests && uv run mypy src`

Expected: 所有测试通过，Ruff 与 mypy 退出 0。

- [ ] **Step 6: 提交后端服务。**

```bash
git add src/qclive tests/integration/test_api.py tests/fixtures
git commit -m "功能：组装私人行情后端"
```

## Task 9: 实现无交易 QuantConnect Cloud 算法

**Files:**
- Create: `cloud/atlas-market-data/protocol.py`
- Create: `cloud/atlas-market-data/main.py`
- Create: `cloud/atlas-market-data/project-manifest.json`
- Create: `tests/unit/test_cloud_protocol.py`
- Create: `tests/unit/test_cloud_source_guard.py`

- [ ] **Step 1: 写入命令协议和禁止订单源码测试。**

```python
# tests/unit/test_cloud_source_guard.py
from pathlib import Path


def test_cloud_algorithm_contains_no_order_calls() -> None:
    source = (
        Path(__file__).resolve().parents[2]
        / "cloud"
        / "atlas-market-data"
        / "main.py"
    ).read_text(encoding="utf-8")
    forbidden = (
        "market_order(",
        "limit_order(",
        "stop_market_order(",
        "set_holdings(",
        "liquidate(",
        "exercise_option(",
    )
    assert all(marker not in source for marker in forbidden)
```

```python
# tests/unit/test_cloud_protocol.py
import pytest

from protocol import SwitchCommand, parse_command


def test_parse_valid_switch_command() -> None:
    parsed = parse_command(
        {
            "action": "switch",
            "requestId": "req-1",
            "assetKind": "crypto",
            "symbol": "BTCUSD",
            "range": "1D",
            "resolution": "1m",
        }
    )
    assert parsed == SwitchCommand("req-1", "crypto", "BTCUSD", "1D", "1m")


def test_order_shaped_command_is_rejected() -> None:
    with pytest.raises(ValueError):
        parse_command({"$type": "OrderCommand", "quantity": 1})
```

- [ ] **Step 2: 运行测试并确认 cloud 文件缺失。**

Run: `uv run pytest tests/unit/test_cloud_protocol.py tests/unit/test_cloud_source_guard.py -q`

Expected: FAIL，错误为缺少 cloud protocol 或 `main.py`。

- [ ] **Step 3: 实现纯 Python 命令协议。**

```python
# cloud/atlas-market-data/protocol.py
from dataclasses import dataclass


@dataclass(frozen=True)
class SwitchCommand:
    request_id: str
    asset_kind: str
    symbol: str
    chart_range: str
    resolution: str


def parse_command(data: object) -> SwitchCommand:
    if not isinstance(data, dict) or data.get("action") != "switch":
        raise ValueError("unsupported command")
    required = ("requestId", "assetKind", "symbol", "range", "resolution")
    if any(not isinstance(data.get(key), str) or not data[key] for key in required):
        raise ValueError("invalid switch command")
    if data["assetKind"] not in {"equity", "crypto"}:
        raise ValueError("unsupported asset kind")
    return SwitchCommand(
        data["requestId"],
        data["assetKind"],
        data["symbol"],
        data["range"],
        data["resolution"],
    )
```

- [ ] **Step 4: 实现云端算法状态机与常驻订阅。**

`initialize` 必须：

```python
from AlgorithmImports import *
from protocol import parse_command


class AtlasMarketData(QCAlgorithm):
    CHART_NAME = "Atlas Market"
    BASELINE = {"SPY", "BTCUSD"}

    def initialize(self) -> None:
        self.set_time_zone(TimeZones.UTC)
        if not self.live_mode:
            self.set_start_date(2026, 8, 24)
            self.set_end_date(2026, 8, 28)
        self.set_cash(100_000)
        self.settings.daily_precise_end_time = True
        self._spy = self.add_equity(
            "SPY",
            Resolution.MINUTE,
            Market.USA,
            extended_market_hours=True,
            data_normalization_mode=DataNormalizationMode.RAW,
        ).symbol
        self._btc = self.add_crypto(
            "BTCUSD", Resolution.MINUTE, Market.COINBASE
        ).symbol
        self._active_symbol = self._spy
        self._dynamic_symbol = None
        self._request_id = "startup"
        self._price = CandlestickSeries("Price", "USD")
        self._volume = Series("Volume", SeriesType.BAR, 1, "")
        chart = Chart(self.CHART_NAME)
        chart.add_series(self._price)
        chart.add_series(self._volume)
        self.add_chart(chart)
        self._set_state("LOADING", "")
        self.set_warm_up(1, Resolution.MINUTE)
```

`on_warmup_finished` 对默认 `SPY/1D/1m` 调用与命令相同的回填路径；不得在 warm-up 期间画图。`on_command` 必须捕获所有异常并返回布尔值；先把 runtime status 设为 `SWITCHING`，成功订阅后设为 `BACKFILLING`，清空固定两条 series，用同步 `history` 获取所需范围的 minute 或 daily bar，按命令粒度合并并用 bar 原始时间戳 `add_point`，再设为 `LIVE`。新标的完整回填成功之前不改 `_active_symbol`、不拆旧 consolidator；失败时保持旧活动标的并写入 `Dashboard Error`。切换提交成功之后，只有旧标的是非默认搜索标的时才拆旧 consolidator 并调用 `remove_security`。

`on_data` 只处理 `_active_symbol`，按当前粒度 consolidator 形成完整 bar 后追加 Price 与 Volume。`on_order_event` 若收到任何事件，必须 `quit("unexpected order event")`，形成第二层无交易保护。

- [ ] **Step 5: 固定 runtime statistics 键和 project manifest。**

```json
{
  "name": "Atlas Live Market Data",
  "language": "Py",
  "mode": "read-only-market-data",
  "baseline_symbols": ["SPY", "BTCUSD"],
  "live_trading": true,
  "orders_allowed": false
}
```

Runtime keys 必须精确包含 `Dashboard Request`、`Dashboard Status`、`Dashboard Symbol`、`Dashboard Asset`、`Dashboard Range`、`Dashboard Resolution`、`Dashboard Subscriptions`、`Dashboard Error`。`Dashboard Subscriptions` 始终按字母顺序列出当前订阅，供只读验证器确认 `BTCUSD` 与 `SPY` 未被移除。

- [ ] **Step 6: 运行协议、源码安全和全后端测试。**

Run:

```bash
PYTHONPATH=cloud/atlas-market-data uv run pytest \
  tests/unit/test_cloud_protocol.py \
  tests/unit/test_cloud_source_guard.py -q
uv run pytest tests/unit tests/integration -q
```

Expected: 协议与禁止订单测试通过；既有后端测试无回归。

- [ ] **Step 7: 提交云端算法。**

```bash
git add cloud/atlas-market-data tests/unit/test_cloud_protocol.py tests/unit/test_cloud_source_guard.py
git commit -m "功能：增加无交易云端行情算法"
```

## Task 10: 建立云项目同步、编译、部署与只读验证工具

**Files:**
- Create: `scripts/sync_cloud_project.py`
- Create: `scripts/run_read_only_backtest.py`
- Create: `scripts/deploy_paper_live.py`
- Create: `scripts/request_chart_switch.py`
- Create: `scripts/verify_live_deployment.py`
- Create: `tests/unit/test_cloud_scripts.py`

- [ ] **Step 1: 写入部署前置条件和 payload 测试。**

```python
# tests/unit/test_cloud_scripts.py
from scripts.deploy_paper_live import build_live_payload


def test_paper_payload_has_no_holdings_and_uses_quantconnect_data() -> None:
    payload = build_live_payload(
        project_id=42,
        compile_id="compile-1",
        node_id="LN-live-1",
    )
    assert payload["brokerage"] == {
        "id": "QuantConnectBrokerage",
        "holdings": [],
        "cash": [{"currency": "USD", "amount": 100000}],
    }
    assert payload["dataProviders"] == {
        "QuantConnectBrokerage": {"id": "QuantConnectBrokerage"}
    }
```

- [ ] **Step 2: 实现幂等云项目同步。**

`sync_cloud_project.py` 必须使用 `/projects/read` 按精确名称寻找项目；不存在时用 `{"name": "Atlas Live Market Data", "language": "Py"}` 调用 `/projects/create`；随后逐个比较并用 `/files/create` 或 `/files/update` 同步 `main.py` 与 `protocol.py`。脚本将 `projectId` 写入被 Git 忽略的 `data/deployment.json`，不把 User ID 或 Token 写入该文件。

- [ ] **Step 3: 实现强制的无交易云端回测门。**

`run_read_only_backtest.py` 读取 `projectId`，调用 `/compile/create` 并轮询 `/compile/read`；编译成功后调用 `/backtests/create`，轮询 `/backtests/read` 到 Completed，再读取 `Atlas Market` 的 Price/Volume 图表和 `/backtests/orders/read`。只有算法正常结束、两条 series 均非空、runtime statistics 含 Dashboard keys 且订单为 0 时输出匹配 `^BACKTEST_GATE_OK project=[0-9]+ orders=0$` 的单行结果；任何编译警告中的 error、RuntimeError 或非零订单都退出非零。测试 transport 固定覆盖 compiling、build error、runtime error、loading orders 与非零订单。

- [ ] **Step 4: 实现受保护的 Paper deployment。**

```python
def build_live_payload(project_id: int, compile_id: str, node_id: str) -> dict:
    return {
        "versionId": "-1",
        "projectId": project_id,
        "compileId": compile_id,
        "nodeId": node_id,
        "brokerage": {
            "id": "QuantConnectBrokerage",
            "holdings": [],
            "cash": [{"currency": "USD", "amount": 100_000}],
        },
        "dataProviders": {
            "QuantConnectBrokerage": {"id": "QuantConnectBrokerage"}
        },
    }
```

`deploy_paper_live.py` 必须依次：认证、确认项目没有 Running deployment、编译、等待编译成功、读取 nodes、只选择唯一 `busy=false` 的 `L-MICRO` 节点、调用 `/live/create`、写入 `deployId`。若存在运行 deployment、节点不唯一、编译失败或 brokerage payload 不是 QuantConnect Paper，立即退出且不改变远端状态。

- [ ] **Step 5: 实现命令工具与只读 live 验证器。**

`request_chart_switch.py` 只接受 `--symbol`、`--asset`、`--range` 与 `--resolution`；复用 `normalize_symbol`、`SwitchRequest` 和 `CommandCoordinator`，发送不含 `$type`、数量或价格的 `switch` command，等待匹配 request ID 的 `LIVE` 确认。成功只打印 request ID 与规范化 symbol，不打印响应正文。

`verify_live_deployment.py` 必须读取 `/live/read`、`/live/chart/read` 与 `/live/orders/read`；对 chart/orders 的 `loading` 使用带 jitter 的指数退避并在单次等待 60 秒处封顶。验证：状态为 `Running`、runtime status 为 `LIVE`、活动 symbol 与参数一致、`Dashboard Subscriptions` 包含 `SPY` 和 `BTCUSD`、图表包含 Price 与 Volume、订单总数为 0、`isPublicStreaming` 与 `public` 都为 false。成功只打印：

输出必须匹配正则 `^LIVE_VERIFY_OK project=[0-9]+ deploy=L-[a-f0-9]+ symbol=SPY orders=0 public=false$`。

验证输出不得包含 Authorization、Token、完整 API 响应或其他凭据。

- [ ] **Step 6: 运行纯测试，暂不调用 QuantConnect。**

Run: `uv run pytest tests/unit/test_cloud_scripts.py -q`

Expected: payload、回测门、已有部署拒绝、节点选择、命令 schema、编译失败、loading 轮询和脱敏测试全部通过；没有网络请求离开测试 transport。

- [ ] **Step 7: 提交云端工具，但仍不部署。**

```bash
git add scripts/sync_cloud_project.py scripts/run_read_only_backtest.py scripts/deploy_paper_live.py scripts/request_chart_switch.py scripts/verify_live_deployment.py tests/unit/test_cloud_scripts.py
git commit -m "工具：增加云端部署与验证流程"
```

## Task 11: 建立 React 前端与已确认的响应式视觉骨架

**Files:**
- Create: `frontend/package.json`
- Create: `frontend/vite.config.ts`
- Create: `frontend/src/main.tsx`
- Create: `frontend/src/App.tsx`
- Create: `frontend/src/styles/tokens.css`
- Create: `frontend/src/styles/dashboard.css`
- Create: `frontend/src/test/App.test.tsx`

- [ ] **Step 1: 用 Vite 建立 React TypeScript 项目并安装固定能力依赖。**

Run:

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install lightweight-charts@^5.2.0 technicalindicators@^3.1.0
npm install -D @playwright/test @testing-library/jest-dom \
  @testing-library/react @testing-library/user-event jsdom vitest
```

Expected: `frontend/package-lock.json` 生成；不得安装商业 TradingView Charting Library。

- [ ] **Step 2: 写入响应式骨架失败测试。**

```tsx
// frontend/src/test/App.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import App from '../App';

describe('App', () => {
  it('renders the confirmed read-only dashboard controls', () => {
    render(<App />);
    expect(screen.getByRole('button', { name: '股票' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: '加密货币' })).toBeInTheDocument();
    expect(screen.getByRole('textbox', { name: '代码' })).toHaveValue('SPY');
    expect(screen.queryByRole('button', { name: /买入|卖出|下单/ })).toBeNull();
  });
});
```

- [ ] **Step 3: 运行测试并确认视觉骨架尚未实现。**

Run: `cd frontend && npm test -- --run`

Expected: FAIL，找不到已确认控件。

- [ ] **Step 4: 实现页面结构和固定视觉 token。**

`App.tsx` 必须渲染：顶部品牌和连接状态、资产切换与搜索、范围和粒度、三项摘要、主图容器、指标选择区、指标副图容器、市场/更新时间/数据源页脚。所有文本从组件 props 或领域状态获得，不写入虚假实时值。

`tokens.css` 固定使用参考图方向：

```css
:root {
  --qc-canvas: #eef1f6;
  --qc-surface: #ffffff;
  --qc-panel: #fbfcfe;
  --qc-border: #dfe3ea;
  --qc-text: #232832;
  --qc-muted: #9299a6;
  --qc-blue: #668fd9;
  --qc-green: #22b65f;
  --qc-red: #df2c37;
  color: var(--qc-text);
  background: var(--qc-canvas);
}
```

`dashboard.css` 在 767px 以下单列；768px 以上两栏；宽屏主图区使用 `minmax(0, 7fr) minmax(280px, 3fr)`。所有 input 字体至少 16px，触控控件最小高度 44px；页面根节点不得产生横向滚动。

同时把 `frontend/package.json` 的 scripts 固定为：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:e2e": "playwright test"
  }
}
```

- [ ] **Step 5: 添加 Lightweight Charts 官方 attribution。**

页脚包含可访问链接 `https://www.tradingview.com/`，文字为 `Charts by TradingView`；仓库保留依赖包 NOTICE 要求，不隐藏 attribution logo。

- [ ] **Step 6: 运行前端单元测试和构建。**

Run: `cd frontend && npm test -- --run && npm run build`

Expected: 测试通过，Vite production build 成功。

- [ ] **Step 7: 提交视觉骨架。**

```bash
git add frontend
git commit -m "界面：建立响应式行情仪表盘"
```

## Task 12: 实现前端 API、状态机和图表交互

**Files:**
- Create: `frontend/src/api/client.ts`
- Create: `frontend/src/state/market.ts`
- Create: `frontend/src/state/useMarketStream.ts`
- Create: `frontend/src/chart/MarketChart.tsx`
- Create: `frontend/src/components/MarketControls.tsx`
- Create: `frontend/src/test/market-state.test.ts`
- Create: `frontend/src/test/MarketChart.test.tsx`

- [ ] **Step 1: 写入 snapshot 合并、SSE 去重和控件状态测试。**

```ts
// frontend/src/test/market-state.test.ts
import { describe, expect, it } from 'vitest';
import { mergeBars } from '../state/market';

describe('mergeBars', () => {
  it('keeps ascending unique bars and replaces matching timestamps', () => {
    const first = [{ time: 1, open: 1, high: 2, low: 1, close: 2, volume: 10 }];
    const next = [{ time: 1, open: 1, high: 3, low: 1, close: 3, volume: 20 }];
    expect(mergeBars(first, next)).toEqual(next);
  });
});
```

- [ ] **Step 2: 运行测试并确认状态层缺失。**

Run: `cd frontend && npm test -- --run src/test/market-state.test.ts`

Expected: FAIL，无法导入 `mergeBars`。

- [ ] **Step 3: 实现浏览器领域类型和单一状态 reducer。**

```ts
export type AssetKind = 'equity' | 'crypto';
export type ChartRange = '1D' | '5D' | '1M';
export type Resolution = '1m' | '5m' | '15m' | '30m' | '1h' | '1D';
export type ConnectionStatus =
  | 'LIVE'
  | 'LOADING'
  | 'STALE'
  | 'MARKET_CLOSED'
  | 'DISCONNECTED'
  | 'ERROR';

export interface Bar {
  time: number;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
}
```

`mergeBars` 使用时间戳 Map 合并并升序返回；reducer 只接受 request ID 等于当前请求的事件，防止旧 SSE 响应覆盖新选择。localStorage 键固定为 `atlas-market-graph:v1:preferences`，只保存资产、代码、范围、粒度、图形类型和指标参数。

- [ ] **Step 4: 实现 API 与 SSE 生命周期。**

`client.ts` 只使用同源相对路径 `/api/v1/*`；在唯一的 `decodeSnapshot` 边界把后端 snake_case 字段映射成浏览器 camelCase 状态，POST switch 发送 snake_case 后端模型；`useMarketStream` 在 mount 建立一个 `EventSource('/api/v1/stream')`，unmount 时关闭；浏览器原生重连期间显示 STALE，不创建重复 EventSource。

- [ ] **Step 5: 实现 Lightweight Charts 价格、成交量和图形切换。**

`MarketChart.tsx` 创建一个 chart；蜡烛模式使用 `CandlestickSeries`，折线模式使用 `LineSeries`，成交量使用 pane index 1 的 `HistogramSeries`。ResizeObserver 只调整当前容器；切换模式时移除旧价格 series，不销毁整个 chart；每次数据更新只对最后一根 bar 调用 `update`，首次快照才调用 `setData`。

- [ ] **Step 6: 运行状态、图表与构建测试。**

Run: `cd frontend && npm test -- --run && npm run build`

Expected: 数据去重、旧 request 拒绝、SSE 清理、蜡烛/折线切换和容器 resize 全部通过。

- [ ] **Step 7: 提交数据交互。**

```bash
git add frontend/src/api frontend/src/state frontend/src/chart frontend/src/components frontend/src/test
git commit -m "功能：连接行情状态与图表交互"
```

## Task 13: 实现并验证 25 个指标注册表

**Files:**
- Create: `frontend/src/indicators/types.ts`
- Create: `frontend/src/indicators/registry.ts`
- Create: `frontend/src/indicators/library.ts`
- Create: `frontend/src/indicators/custom.ts`
- Create: `frontend/src/components/IndicatorPicker.tsx`
- Create: `frontend/src/chart/IndicatorLayers.tsx`
- Create: `frontend/src/test/indicators.test.ts`

- [ ] **Step 1: 写入注册表完整性和数值基准失败测试。**

```ts
// frontend/src/test/indicators.test.ts
import { describe, expect, it } from 'vitest';
import { indicatorRegistry } from '../indicators/registry';
import { computeIndicator } from '../indicators/library';

const bars = Array.from({ length: 60 }, (_, index) => ({
  time: index + 1,
  open: index + 1,
  high: index + 2,
  low: index,
  close: index + 1,
  volume: 100 + index,
}));

describe('indicator registry', () => {
  it('contains the exact approved set', () => {
    expect(indicatorRegistry.map((item) => item.id)).toEqual([
      'sma', 'ema', 'wma', 'hma', 'vwap', 'ichimoku', 'supertrend', 'psar',
      'rsi', 'stochastic', 'stoch-rsi', 'macd', 'cci', 'roc', 'williams-r',
      'bollinger', 'atr', 'keltner', 'donchian', 'stddev',
      'obv', 'mfi', 'cmf', 'volume-sma', 'adl',
    ]);
  });

  it('computes aligned SMA20 values', () => {
    const result = computeIndicator('sma', bars, { period: 20 });
    expect(result.series[0].points[0]).toEqual({ time: 20, value: 10.5 });
  });
});
```

- [ ] **Step 2: 运行测试并确认指标模块缺失。**

Run: `cd frontend && npm test -- --run src/test/indicators.test.ts`

Expected: FAIL，无法导入指标注册表。

- [ ] **Step 3: 实现固定注册表、参数 schema 和输出合同。**

```ts
export type IndicatorPane = 'overlay' | 'oscillator' | 'volume';

export interface IndicatorPoint {
  time: number;
  value: number;
}

export interface IndicatorSeries {
  key: string;
  points: IndicatorPoint[];
}

export interface IndicatorResult {
  pane: IndicatorPane;
  series: IndicatorSeries[];
}

export interface IndicatorDefinition {
  id: string;
  label: string;
  pane: IndicatorPane;
  defaults: Record<string, number>;
  limits: Record<string, readonly [number, number]>;
}
```

`registry.ts` 必须按测试中的顺序定义 25 项，并使用设计规格中的默认参数。参数边界：period 为 2–500，标准差倍数为 0.1–10，PSAR step 为 0.001–1，Supertrend multiplier 为 0.1–20。

- [ ] **Step 4: 包装 technicalindicators 直接提供的 18 项并组合三项。**

`library.ts` 使用 technicalindicators 的 SMA、EMA、WMA、VWAP、IchimokuCloud、PSAR、RSI、Stochastic、MACD、CCI、ROC、WilliamsR、BollingerBands、ATR、SD、OBV、MFI 和 ADL。Keltner 使用 EMA 与 ATR 组合，Donchian 使用滚动 Highest/Lowest，Volume SMA 复用 SMA；这三项仍通过同一个 `computeIndicator` 入口暴露。所有输出用 `alignToBars` 按结果长度对齐到原 bar 时间戳；任何 `NaN` 或无限值丢弃，不修改输入数组。

- [ ] **Step 5: 实现四个剩余自定义指标。**

`custom.ts` 必须包含：

- HMA：`WMA(2*WMA(n/2)-WMA(n), sqrt(n))`。
- Supertrend：ATR 基础上下轨，趋势反转只在 close 穿越最终轨时发生。
- Stoch RSI：先算 RSI，再在窗口内归一化并分别平滑 K、D。
- CMF：`sum(((close-low)-(high-close))/(high-low)*volume, n) / sum(volume, n)`；`high==low` 的 money flow multiplier 为 0。

每个函数为纯函数并返回与 `IndicatorResult` 相同的时间对齐合同。

- [ ] **Step 6: 加入每个指标至少一个固定样本或性质测试。**

Run: `cd frontend && npm test -- --run src/test/indicators.test.ts`

Expected: 25 个注册项均有输出或明确的 warm-up 空数组；输入不变；参数越界抛出具名错误；没有 `NaN`。

- [ ] **Step 7: 实现选择器、主图叠加和独立副图。**

`IndicatorPicker` 支持搜索、开关和数值参数；默认启用 EMA20、VWAP、RSI14。`IndicatorLayers` 将 overlay series 放到主 pane，将 oscillator/volume 按启用顺序放到独立可调整 pane；单个指标失败只显示该指标错误，不清除其他 series。

- [ ] **Step 8: 运行完整前端测试和构建。**

Run: `cd frontend && npm test -- --run && npm run build`

Expected: 所有测试通过，构建成功，bundle 中不包含 QuantConnect Token 字符串。

- [ ] **Step 9: 提交指标系统。**

```bash
git add frontend/src/indicators frontend/src/components/IndicatorPicker.tsx frontend/src/chart/IndicatorLayers.tsx frontend/src/test
git commit -m "功能：增加二十五个技术指标"
```

## Task 14: 完成状态呈现、移动端和宽屏视觉验证

**Files:**
- Create: `frontend/src/components/ConnectionStatus.tsx`
- Create: `frontend/src/components/MarketSummary.tsx`
- Create: `frontend/src/components/MarketFooter.tsx`
- Create: `frontend/playwright.config.ts`
- Create: `frontend/tests/e2e/dashboard.spec.ts`
- Modify: `frontend/src/App.tsx`
- Modify: `frontend/src/styles/dashboard.css`

- [ ] **Step 1: 写入状态和无误导标签测试。**

```tsx
const staleFixture = {
  status: 'STALE' as const,
  requestId: 'req-fixture',
  assetKind: 'equity' as const,
  symbol: 'SPY',
  chartRange: '1D' as const,
  resolution: '1m' as const,
  marketPhase: 'REGULAR',
  bars: [{ time: 1_788_199_940, open: 99, high: 101, low: 98, close: 100, volume: 10 }],
  lastApiSuccess: 1_788_200_000,
  error: null,
};

it('shows actual source and stale state without claiming tick data', () => {
  render(<App initialState={staleFixture} />);
  expect(screen.getByText('STALE')).toBeInTheDocument();
  expect(screen.getByText('QuantConnect · 1 min')).toBeInTheDocument();
  expect(screen.queryByText(/tick|逐笔|秒级实时/i)).toBeNull();
});
```

- [ ] **Step 2: 实现六种状态和最后成功快照保留。**

`ConnectionStatus` 使用文字、形状和颜色共同编码；`LOADING` 显示当前请求；`STALE` 与 `DISCONNECTED` 显示最后成功时间；`MARKET_CLOSED` 不使用错误色；`ERROR` 显示可读原因。所有状态更新放入 `aria-live="polite"`，只有无法继续的错误使用 `role="alert"`。

- [ ] **Step 3: 写入 Playwright 视口、键盘和横向溢出测试。**

```ts
// frontend/playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  use: { baseURL: 'http://127.0.0.1:4173' },
  webServer: {
    command: 'npm run preview -- --host 127.0.0.1 --port 4173',
    url: 'http://127.0.0.1:4173',
    reuseExistingServer: false,
  },
});
```

```ts
// frontend/tests/e2e/dashboard.spec.ts
import { expect, test } from '@playwright/test';

const staleFixture = {
  status: 'STALE',
  request_id: 'req-fixture',
  asset_kind: 'equity',
  symbol: 'SPY',
  chart_range: '1D',
  resolution: '1m',
  market_phase: 'REGULAR',
  bars: [
    { time: 1_788_199_940, open: 99, high: 101, low: 98, close: 100, volume: 10 },
  ],
  last_api_success: 1_788_200_000,
  error: null,
};

for (const viewport of [
  { name: 'mobile-320', width: 320, height: 800 },
  { name: 'mobile', width: 360, height: 900 },
  { name: 'desktop-16-9', width: 1920, height: 1080 },
  { name: 'desktop-16-10', width: 1920, height: 1200 },
]) {
  test(`${viewport.name} has no horizontal overflow`, async ({ page }) => {
    await page.setViewportSize(viewport);
    await page.route('**/api/v1/snapshot', async (route) => {
      await route.fulfill({
        contentType: 'application/json',
        body: JSON.stringify(staleFixture),
      });
    });
    await page.goto('/');
    const overflow = await page.evaluate(
      () => document.documentElement.scrollWidth - document.documentElement.clientWidth,
    );
    expect(overflow).toBe(0);
    await expect(page).toHaveScreenshot(`${viewport.name}.png`, { fullPage: true });
  });
}
```

- [ ] **Step 4: 运行浏览器验证。**

Run:

```bash
cd frontend
npx playwright install chromium
npm run build
npx playwright test --update-snapshots
npx playwright test
```

在第一次基线生成后，逐张检查 `frontend/tests/e2e/dashboard.spec.ts-snapshots/` 中的 PNG，确认内容与已批准 A 版视觉一致且没有 loading 占位遮住主界面，再执行第二次无更新测试。

Expected: 320px、360px、16:9、16:10 四个视口截图通过，无横向滚动；股票/加密货币、范围、粒度、图形类型和指标可用键盘操作。

- [ ] **Step 5: 提交状态与响应式完善。**

```bash
git add frontend/src frontend/playwright.config.ts frontend/tests/e2e
git commit -m "界面：完善状态与多尺寸适配"
```

## Task 15: 配置 Atlas 用户级服务与运维文档

**Files:**
- Create: `deploy/quantconnect-live-chart.service`
- Create: `docs/operations.md`
- Create: `tests/unit/test_deployment_files.py`
- Modify: `README.md`

- [ ] **Step 1: 写入 service 安全合同测试。**

```python
# tests/unit/test_deployment_files.py
from pathlib import Path


def test_systemd_service_binds_loopback_and_reads_no_token() -> None:
    service = (
        Path(__file__).resolve().parents[2]
        / "deploy"
        / "quantconnect-live-chart.service"
    ).read_text(encoding="utf-8")
    assert "--host 127.0.0.1" in service
    assert "Restart=on-failure" in service
    assert "api-token" not in service.lower()
    assert "user-id" not in service.lower()
```

- [ ] **Step 2: 写入用户级 systemd unit。**

```ini
[Unit]
Description=QuantConnect Live Chart
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/jingtianyu/projects/quantconnect-live-chart
ExecStart=/home/jingtianyu/projects/quantconnect-live-chart/.venv/bin/uvicorn qclive.api.app:create_app --factory --host 127.0.0.1 --port 8765
Restart=on-failure
RestartSec=5
Environment=QCLIVE_DATA_DIR=/home/jingtianyu/projects/quantconnect-live-chart/data
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/home/jingtianyu/projects/quantconnect-live-chart/data

[Install]
WantedBy=default.target
```

- [ ] **Step 3: 写入可执行运维说明。**

`docs/operations.md` 必须包含：

- 安装依赖与前端构建命令。
- 把 unit 链接到 `~/.config/systemd/user/`、daemon-reload、enable 与 start。
- 从电脑访问的 SSH 隧道命令：`ssh -L 8765:127.0.0.1:8765 jingtianyu@atlas-ewr`。
- `curl http://127.0.0.1:8765/health/live` 与 `/health/ready`。
- 查看脱敏日志、重启、停止、禁用和恢复服务。
- QuantConnect live deployment 的只读检查、停止前检查与重新部署步骤。
- 明确“合并不是部署”“停止 Atlas 服务不会停止 QuantConnect 云算法”。

- [ ] **Step 4: 运行 service 合同和 Markdown 链接检查。**

Run: `uv run pytest tests/unit/test_deployment_files.py -q && uv run ruff check src tests`

Expected: unit 文件不含凭据，绑定回环地址，运维链接均解析到现有文件。

- [ ] **Step 5: 提交部署文件。**

```bash
git add deploy docs/operations.md README.md tests/unit/test_deployment_files.py
git commit -m "运维：增加 Atlas 服务与操作手册"
```

## Task 16: 完成离线总验收、真实云部署和端到端验证

**Files:**
- Create: `scripts/verify_no_secrets.py`
- Create: `tests/integration/test_failure_recovery.py`
- Modify: `README.md`

- [ ] **Step 1: 实现仓库与构建产物凭据扫描器。**

`verify_no_secrets.py` 从 `~/.lean/credentials` 读取 Token 仅用于内存比较；扫描 `git ls-files`、`frontend/dist` 和测试日志。脚本只输出 `SECRET_SCAN_OK` 或具名文件路径，不打印匹配内容。Token 缺失时退出非零，避免产生虚假的安全通过结论。

- [ ] **Step 2: 增加断线、恢复和旧请求竞争的端到端后端测试。**

测试用可控制的 fake QuantConnect server 依次返回：`loading`、成功图表、三次 503、新图表；断言状态顺序为 `LOADING → LIVE → STALE → DISCONNECTED → LIVE`，最后成功 bars 在失败期间保持不变，恢复后不产生重复时间戳。

- [ ] **Step 3: 运行全部离线质量门。**

Run:

```bash
uv run pytest -q
uv run ruff check src tests scripts
uv run mypy src
cd frontend
npm test -- --run
npm run build
npx playwright test
cd ..
uv run python scripts/verify_no_secrets.py
git diff --check
```

Expected: 所有命令退出 0；Playwright 四个目标视口通过；输出包含 `SECRET_SCAN_OK`。

- [ ] **Step 4: 创建并编译 QuantConnect 云项目，先运行无交易回测。**

Run:

```bash
uv run python scripts/sync_cloud_project.py
uv run python scripts/run_read_only_backtest.py
```

Expected: 输出 `BACKTEST_GATE_OK`；云编译与 backtest 成功；结果包含 Price、Volume 与 runtime statistics；订单数量为 0。若任何订单出现，停止，不执行 live deployment。

- [ ] **Step 5: 部署 QuantConnect Paper live algorithm。**

Run: `uv run python scripts/deploy_paper_live.py`

Expected: 脚本确认无现有 Running deployment、选择唯一空闲 `L-MICRO`、创建 QuantConnect Paper deployment，并把非敏感 project/deploy ID 写入 `data/deployment.json`。

- [ ] **Step 6: 验证 SPY、BTCUSD 与一个额外美股切换。**

Run:

```bash
uv run python scripts/verify_live_deployment.py --symbol SPY
uv run python scripts/request_chart_switch.py --symbol BTCUSD --asset crypto --range 1D --resolution 1m
uv run python scripts/verify_live_deployment.py --symbol BTCUSD --asset crypto
uv run python scripts/request_chart_switch.py --symbol AAPL --asset equity --range 1D --resolution 1m
uv run python scripts/verify_live_deployment.py --symbol AAPL
uv run python scripts/request_chart_switch.py --symbol SPY --asset equity --range 1D --resolution 1m
uv run python scripts/verify_live_deployment.py --symbol SPY
```

Expected: 每次输出 `LIVE_VERIFY_OK`；`SPY` 与 `BTCUSD` 仍在订阅；AAPL 切回后被移除；订单始终为 0；public streaming 始终为 false。

- [ ] **Step 7: 在隔离 worktree 启动临时验收服务。**

Run:

```bash
cd /home/jingtianyu/projects/quantconnect-live-chart-v1
uv sync --frozen
cd frontend && npm ci && npm run build && cd ..
uv run uvicorn qclive.api.app:create_app --factory \
  --host 127.0.0.1 --port 18765
```

Expected: 验收进程保持运行；`curl --fail http://127.0.0.1:18765/health/live` 与 `/health/ready` 返回 200；`ss -ltnp` 显示仅监听 `127.0.0.1:18765`。

- [ ] **Step 8: 做实际浏览器验收与最终凭据检查。**

在 320×800、360×900、1920×1080、1920×1200 四个视口打开 Atlas 页面，验证 SPY/BTCUSD 切换、1D/5D/1M、兼容粒度、蜡烛/折线、25 个指标、LOADING、MARKET CLOSED 或当前市场阶段。随后运行：

```bash
uv run python scripts/verify_no_secrets.py
git status --short --branch
git diff --check
```

Expected: 页面无溢出；所有交互可用；凭据扫描通过；工作区只包含本任务预期修改。

- [ ] **Step 9: 更新 README 的当前运行状态并提交完成结果。**

```bash
git add README.md scripts/verify_no_secrets.py tests/integration/test_failure_recovery.py
git commit -m "完成：交付私人实时行情图表"
```

- [ ] **Step 10: 推送功能分支前重新验证仓库私有性与远程提交。**

Run:

```bash
gh repo view Jimmyyu725/quantconnect-live-chart --json visibility,nameWithOwner,url
git push -u origin feature/live-chart-v1
git rev-parse HEAD
git ls-remote origin refs/heads/feature/live-chart-v1
```

Expected: GitHub 返回 `PRIVATE`；本地 HEAD 与远程功能分支哈希一致。此步骤只备份功能分支，不合并 `main`，也不把分支推送等同于部署。

- [ ] **Step 11: 调用 `superpowers:finishing-a-development-branch` 完成主线集成，并在用户选择合并后验证 `main`。**

用户选择合并后，在主 worktree 执行：

```bash
cd /home/jingtianyu/projects/quantconnect-live-chart
git status --porcelain
git pull --ff-only
git merge --no-ff feature/live-chart-v1
uv sync --frozen
cd frontend && npm ci && npm test -- --run && npm run build && cd ..
uv run pytest -q
gh repo view Jimmyyu725/quantconnect-live-chart --json visibility,nameWithOwner,url
git push origin main
git rev-parse HEAD
git ls-remote origin refs/heads/main
```

Expected: 合并前主 worktree 干净；全套关键测试通过；GitHub 返回 `PRIVATE`；本地 `main` 与远程 `main` 哈希一致。若用户选择不合并，本步骤停止，不能声称生产路径部署完成。

- [ ] **Step 12: 从已合并的主项目安装并启动 Atlas 用户服务。**

Run:

```bash
mkdir -p /home/jingtianyu/.config/systemd/user
ln -sfn \
  /home/jingtianyu/projects/quantconnect-live-chart/deploy/quantconnect-live-chart.service \
  /home/jingtianyu/.config/systemd/user/quantconnect-live-chart.service
systemctl --user daemon-reload
systemctl --user enable --now quantconnect-live-chart.service
systemctl --user is-active quantconnect-live-chart.service
curl --fail http://127.0.0.1:8765/health/live
curl --fail http://127.0.0.1:8765/health/ready
ss -ltnp | rg '127\.0\.0\.1:8765'
```

Expected: service 为 `active`；两个健康端点返回 200；只监听 `127.0.0.1:8765`。

- [ ] **Step 13: 运行生产路径最终审计。**

Run:

```bash
cd /home/jingtianyu/projects/quantconnect-live-chart
uv run python scripts/verify_live_deployment.py --symbol SPY
uv run python scripts/verify_no_secrets.py
git status --short --branch
```

Expected: live 验证与凭据扫描通过；`main` 与 `origin/main` 一致；工作区干净。

## 规格覆盖核对表

| 规格要求 | 实施任务 |
| --- | --- |
| QuantConnect-only 数据源与安全认证 | Tasks 3、9、10 |
| SPY/BTCUSD 常驻和动态代码切换 | Tasks 6、9、16 |
| 1D/5D/1M 与自适应粒度 | Tasks 2、9、12 |
| 蜡烛、折线、成交量 | Tasks 7、9、12 |
| 25 个指标与参数 | Task 13 |
| 手机、16:9、16:10 | Tasks 11、14 |
| LIVE/LOADING/STALE/休市/断线/错误 | Tasks 4、7、14、16 |
| SQLite 31 天缓存 | Task 5 |
| 回环地址与 SSH 隧道 | Tasks 15、16 |
| 无订单保证 | Tasks 9、10、16 |
| Token 不进浏览器、Git 或日志 | Tasks 3、15、16 |
| 持续运行与自动恢复 | Tasks 15、16 |
| 私有 GitHub 备份 | Task 16 |
