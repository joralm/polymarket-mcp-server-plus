# Polymarket SDK Migration Report

## Stage 1 — Pre-migration audit

This audit was completed before changing application source. The repository
worktree was clean at the start of the audit.

### Architecture and dependencies

- Python package: `polymarket-mcp`, version `0.4.0`, in `pyproject.toml`.
- Runtime constraint: Python `>=3.10`.
- Runtime dependencies are minimum-version ranges, not a fully locked set:
  `mcp>=1.9.0`, `py-clob-client-v2>=1.0.1`, `websockets>=13.0`,
  `eth-account>=0.13.0`, `web3>=7.16.0`, `python-dotenv>=1.2.0`,
  `httpx>=0.28.0`, `pydantic>=2.10.0`, `pydantic-settings>=2.7.0`,
  `fastapi>=0.115.0`, `uvicorn>=0.30.0`, and `jinja2>=3.1.4`.
  Redis is an optional dependency.
- `py-clob-client-v2` is imported in `src/polymarket_mcp/auth/client.py`.
  The code uses its `ClobClient`, order and trade request types, balances,
  order management, signature settings, and API-key creation methods.
  No `py-clob-client` (v1 package) dependency was found.
- `src/polymarket_mcp/auth/signer.py` has a separate custom EIP-712 order
  signer. The wrapper also uses `web3` for Polygon allowance management.
- Gamma HTTP requests are in `tools/market_discovery.py`,
  `tools/market_analysis.py`, `tools/trading.py`, and `auth/client.py`.
  CLOB market-data HTTP requests are in `tools/market_analysis.py`.
  Data API HTTP requests to `/positions`, `/trades`, and `/activity` are in
  `tools/portfolio.py` and the fallback in `auth/client.py`.
- Direct Polymarket HTTP outside those clients is used for the geoblock
  check (`server.py`) and a USD/EUR rate lookup (`portfolio.py`).
  `utils/websocket_manager.py` maintains direct CLOB and RTDS WebSocket
  connections for realtime MCP tools.
- Authentication currently combines an API-key configuration/derivation
  flow, a `ClobClient` initialized with private key/signature type/funder,
  and custom allowance/balance synchronization. Unsupported signature types
  are silently changed to type 3 in `auth/client.py`. Configuration calls
  type 2 “POLY_PROXY” although the official SDK maps type 2 to Gnosis Safe.
- Generated CLOB API key, secret, and passphrase are currently written to
  application logs by `_log_new_credentials` in `auth/client.py`; this is
  incompatible with the requested secret-handling requirement and must be
  removed. Tool-call error responses currently echo submitted arguments.
-   Before the follow-up hardening, `ENABLE_AUTONOMOUS_TRADING` defaulted to
  `true` in config and Compose. It now defaults to `false` in config, Compose,
  the example environment file, and the setup wizard, while retaining the
  existing explicit confirmation safeguards.

### MCP tool map

All tools return MCP text content containing JSON. Public discovery/analysis
and status tools are available without authenticated trading credentials;
trading and portfolio tools are registered only when authentication verifies.
Realtime tools are controlled by `WS_ENABLED`.

| Existing tool(s) | Current underlying API |
| --- | --- |
| `search_markets` | Gamma `GET /markets` with query and filters |
| `get_trending_markets` | Gamma `GET /markets`, locally ranked by volume |
| `filter_markets_by_category` | Gamma `GET /markets` with category/tag filters |
| `get_event_markets` | Gamma `GET /events` and/or `/events/{slug-or-id}`, then event market payload |
| `get_featured_markets` | Gamma `GET /markets` with featured/promoted filters |
| `get_closing_soon_markets` | Gamma `GET /markets` with active/close-time filtering |
| `get_sports_markets` | Gamma `GET /markets` with sports filters |
| `get_crypto_markets` | Gamma `GET /markets` with crypto/category filters |
| `get_market_details` | Gamma `GET /markets` by slug/condition ID or `/markets/{id}` |
| `get_current_price` | CLOB `GET /price` (BUY and SELL) |
| `get_orderbook` | CLOB `GET /book` |
| `get_spread` | CLOB spread endpoint or values derived from the orderbook |
| `get_market_volume` | Gamma/CLOB volume data; computed/aggregated in analysis helpers |
| `get_liquidity` | Gamma market liquidity and/or CLOB orderbook depth |
| `get_price_history` | CLOB `GET /prices-history` |
| `get_market_holders` | Data API holder data |
| `analyze_market_opportunity` | Composition of market details, prices, orderbook/spread, volume, and liquidity |
| `compare_markets` | Repeated market detail/price/volume lookups |
| `get_server_status` | Local configuration/auth state and authenticated CLOB balance probe |
| `create_limit_order` | Market lookup, CLOB orderbook/positions, safety checks, CLOB signed order submission |
| `create_market_order` | Market lookup, CLOB orderbook/positions, safety checks, CLOB market-order submission |
| `create_batch_orders` | Repeated limit order creation/submission |
| `suggest_order_price` | Market/orderbook lookup and local pricing calculation |
| `get_order_status` | Authenticated CLOB order lookup |
| `get_open_orders` | Authenticated CLOB open-order listing |
| `get_order_history` | Authenticated CLOB trade history |
| `cancel_order` | Authenticated CLOB order cancellation |
| `cancel_market_orders` | Authenticated CLOB market-order cancellation |
| `cancel_all_orders` | Authenticated CLOB cancel-all |
| `execute_smart_trade` | Market/orderbook/positions, safety checks, then CLOB order submission |
| `rebalance_position` | Positions/orderbook lookups and CLOB order submission |
| `get_all_positions` | Data API `GET /positions` for the configured funder |
| `get_position_details` | Data API `/positions` and `/trades` |
| `get_portfolio_value` | Data API positions/value aggregation |
| `get_pnl_summary` | Data API `/trades` and `/positions` |
| `get_trade_history` | Data API `GET /trades` |
| `get_activity_log` | Data API `GET /activity` |
| `analyze_portfolio_risk` | Data API `/positions` plus local risk calculations |
| `suggest_portfolio_actions` | Data API `/positions` plus local suggestions |
| `subscribe_market_prices` | CLOB market WebSocket |
| `subscribe_orderbook_updates` | CLOB market WebSocket |
| `subscribe_user_orders` | Authenticated CLOB user WebSocket |
| `subscribe_user_trades` | Authenticated CLOB user WebSocket |
| `subscribe_market_resolution` | RTDS WebSocket market-resolution stream |
| `get_realtime_status` | Local WebSocket manager state |
| `unsubscribe_realtime` | Local WebSocket manager state and subscriptions |

Tool names and JSON argument/result shapes are declared in
`tools/market_discovery.py`, `tools/market_analysis.py`, `tools/trading.py`,
`tools/portfolio_integration.py`, and `tools/realtime.py`; routing is in
`server.py`. The server also exposes the read-only MCP resources
`polymarket://status`, `polymarket://config`, and `polymarket://rate-limits`.

### Configuration and deployment

Configuration is loaded by Pydantic Settings from environment variables and
`.env` (`config.py`, case-sensitive). Existing variables include:

- Wallet/auth: `DEMO_MODE`, `POLYGON_PRIVATE_KEY`, `POLYGON_ADDRESS`,
  `POLYMARKET_SIGNATURE_TYPE`, `POLYMARKET_FUNDER`, `POLYMARKET_API_KEY`,
  `POLYMARKET_API_SECRET`, `POLYMARKET_PASSPHRASE`,
  `POLYMARKET_API_KEY_NAME`, `POLYMARKET_RELAYER_KEY`,
  `POLYMARKET_RELAYER_SECRET`, `POLYMARKET_RELAYER_PASSPHRASE`.
- Network/API: `POLYMARKET_ENV`, `POLYMARKET_CHAIN_ID`,
  `POLYMARKET_TEST_CHAIN_ID`, `CLOB_API_URL`, `GAMMA_API_URL`,
  `CLOB_API_TEST_URL`, `GAMMA_API_TEST_URL`, `POLYMARKET_GEOBLOCK_URL`,
  `POLYGON_RPC_URL`, `USDC_ADDRESS`, `CTF_EXCHANGE_ADDRESS`,
  `CONDITIONAL_TOKEN_ADDRESS`.
- Safety/operations: `MAX_ORDER_SIZE_USD`, `MAX_TOTAL_EXPOSURE_USD`,
  `MAX_POSITION_SIZE_PER_MARKET`, `MIN_LIQUIDITY_REQUIRED`,
  `MAX_SPREAD_TOLERANCE`, `ENABLE_AUTONOMOUS_TRADING`,
  `REQUIRE_CONFIRMATION_ABOVE_USD`, `AUTO_CANCEL_ON_LARGE_SPREAD`,
  `WS_ENABLED`, and `LOG_LEVEL`.
- Transport/cache: `MCP_TRANSPORT`, `MCP_STREAMABLE_HTTP_HOST`,
  `MCP_STREAMABLE_HTTP_PORT`, `MCP_STREAMABLE_HTTP_PATH`, `REDIS_URL`,
  `REDIS_HOST`, and `REDIS_PORT`.

`POLYMARKET_FUNDER_ADDRESS` and `POLYMARKET_SIGNER_ADDRESS` have been added as
explicit wallet settings. `POLYMARKET_SIGNER_ADDRESS` takes precedence over
the legacy `POLYGON_ADDRESS`, which remains supported as a fallback.
`POLYMARKET_FUNDER_ADDRESS` takes precedence over the legacy
`POLYMARKET_FUNDER`. `POLYGON_PRIVATE_KEY` remains unchanged.

The HTTP server is MCP `StreamableHTTPSessionManager` via Starlette and
Uvicorn, serving the configured `/mcp` route. `MCP_TRANSPORT=stdio` remains
an optional local mode; default transport is Streamable HTTP. Docker uses a
single Python 3.12 image, starts `python -m polymarket_mcp.server`, and
Compose preserves a logs bind mount and named `polymarket-data` volume. The
Compose port is configurable; its default is 8000 (set it to 53130 in the
Portainer stack to retain the supplied OpenClaw URL).

### Official SDK verification and migration impact

The official repository `https://github.com/Polymarket/py-sdk` reports the
latest stable release as `polymarket-client 0.18.0`, tag
`polymarket-client-v0.18.0`, published Nov. 9, 2026. Its `pyproject.toml`
requires Python `>=3.11`; this project must raise its declared minimum while
the current Docker Python 3.12 base can remain.

The SDK package imports as `polymarket`. Its public API provides public
market/event discovery, orderbook/price data, positions, trade/activity
history, and portfolio data. Its secure client supports API credential
creation/derivation, balance and allowance, signed order construction,
order posting/cancellation, and authenticated account workflows. SDK models
are typed objects, so the existing dictionary-based tool outputs need an
explicit normalization/compatibility layer.

The official SDK source `src/polymarket/_internal/wallet.py` maps
`DEPOSIT_WALLET` to signature type `3`, recognizes signer and wallet
separately, and provides deposit-wallet signature handling. The official SDK
order examples construct `SecureClient` with `private_key` and `wallet`
separately. Therefore type 3 is verified from the current SDK implementation,
not inferred from an older client. Migration should use its secure wallet
detection/creation APIs and fail closed on unsupported wallet configurations;
it must not implement an EOA fallback. The provided production addresses are
configuration values and must not be hardcoded into source.

The new SDK can replace Gamma/Data/CLOB HTTP calls where corresponding
public/secure methods exist. The existing geoblock check is a separate
Polymarket web endpoint; realtime WebSocket features must be kept compatible,
using the SDK stream API where it fully covers the existing subscriptions or
retaining only the technically necessary transport adapter. Existing
Streamable HTTP, endpoint, tool names, input/output schemas, and volume
configuration must remain unchanged.

### Stage 1 decision

The official SDK exposes the required deposit-wallet signature type and the
main market, account, and trading workflows. There is no blocker to proceeding
with a staged adapter-based migration. Compatibility work remains necessary
for SDK model normalization, existing API URL/testnet configuration, realtime
subscription behavior, and preservation of existing MCP output shapes.

## Migration status

Stage 1 audit is complete and recorded above. `pyproject.toml` now declares
`polymarket-client>=0.18.0,<0.19.0` and Python `>=3.11`, retaining
`py-clob-client-v2` during the planned transition. The new SDK was verified
against the official repository source and release metadata, but installation
into this environment was denied by its permission gate and cannot be retried
or worked around in this session. The Python environment also lacks `pytest`
and `pydantic`, so the test suite and config runtime checks cannot run here.

Independent hardening changes completed after the audit:

- Added validated signer/funder address configuration with precedence for
  `POLYMARKET_SIGNER_ADDRESS` and `POLYMARKET_FUNDER_ADDRESS`, preserving the
  `POLYGON_ADDRESS` and `POLYMARKET_FUNDER` aliases. Explicit signer addresses
  are checked against the configured private key.
- Removed plaintext CLOB credential logging and redacted configured secrets
  from CLOB error logs.
- Preserved the existing MCP error response structure while redacting secret
  arguments and error-message values.
- Changed autonomous trading defaults to disabled in source, Compose,
  documentation, and the setup wizard.
- Added focused tests for the above behavior; they are not executable here
  because the test and runtime dependencies are unavailable.

The actual migration from `py-clob-client-v2` to the SDK — including wallet
detection/authentication, API methods, portfolio mapping, stream compatibility,
and release/test verification — remains incomplete. The existing Streamable
HTTP implementation and tool definitions have not been changed.
