# AGENTS.md

## Project Mission

Rewrite this repository into a cryptocurrency-market-focused version of TradingAgents **on top of the existing project**. Do not rebuild the framework from scratch.

The target product is a multi-agent cryptocurrency market research and trading-analysis framework for:

- `crypto_spot`
- `crypto_perp`

Preserve the existing multi-agent workflow, LLM-provider orchestration, LangGraph collaboration pattern, debate/risk-management mechanism, memory/reflection path, vendor-routing style, and report-generation mechanism wherever possible. Rewrite stock-market assumptions into crypto-native analysis only where the current code proves they are stock-coupled.

## Current Repository Facts

These facts were confirmed from the current codebase and should be treated as the starting point for future rewrite work.

- The simple programmatic entry point is `main.py`; it builds `DEFAULT_CONFIG`, constructs `TradingAgentsGraph`, and calls `propagate("NVDA", "2024-05-10")`.
- The interactive CLI entry point is `cli/main.py`, a Typer application named `TradingAgents`.
- `TradingAgentsGraph` in `tradingagents/graph/trading_graph.py` initializes config, LLM clients, memory, tool nodes, graph setup, propagation, reflection, signal processing, and checkpoint support.
- The graph is built in `tradingagents/graph/setup.py` with LangGraph `StateGraph(AgentState)`.
- The graph runs selected analyst nodes first, then the bull/bear research debate, then research manager, trader, risk analysts, and portfolio manager.
- State is defined in `tradingagents/agents/utils/agent_states.py`. It carries `company_of_interest`, `asset_type`, `instrument_context`, `trade_date`, analyst reports, investment debate state, trader plan, risk debate state, final decision, and memory context.
- Initial state is created in `tradingagents/graph/propagation.py`. The current default `asset_type` is `stock`.
- Report output is centralized in `tradingagents/reporting.py`, which writes per-section markdown under numbered report directories and a consolidated `complete_report.md`.
- Data tools are exposed through `tradingagents/agents/utils/agent_utils.py`, which re-exports market, technical indicator, fundamentals, news, macro, prediction-market, and verification tools.
- Vendor routing is centralized in `tradingagents/dataflows/interface.py`. Current configured vendor categories include `core_stock_apis`, `technical_indicators`, `fundamental_data`, `news_data`, `macro_data`, and `prediction_markets`.
- Default vendor configuration in `tradingagents/default_config.py` routes price and technical data to `yfinance`, fundamentals to `yfinance`, news to `yfinance`, macro data to `fred`, and prediction markets to `polymarket`.
- Existing tests cover crypto asset detection, symbol normalization, CLI symbol handling, instrument identity, vendor routing, reporting, data validation, and analyst execution.

## Existing Workflow to Preserve

Preserve the existing workflow shape unless a future PR proves a narrower change is required:

1. Entry point collects ticker, date, selected analysts, config, LLM provider, and output options.
2. `TradingAgentsGraph` initializes config, LLM clients, memory, tools, workflow graph, propagation, reflection, signal processing, and optional checkpointing.
3. `Propagator.create_initial_state()` creates graph state with ticker, date, asset type, instrument context, reports, and debate states.
4. Analyst nodes create market, sentiment, news, and fundamentals reports according to selected analysts.
5. Bull and bear researchers debate the opportunity.
6. Research manager converts the bull/bear debate into an investment plan.
7. Trader converts the investment plan into a transaction proposal.
8. Aggressive, conservative, and neutral risk analysts debate the trader proposal.
9. Portfolio manager emits the final trading decision.
10. Signal processing extracts the final action, memory stores the decision, JSON state is logged, and report writers can save markdown reports.

The crypto rewrite should adapt this flow to spot and perpetual crypto markets instead of replacing it.

## Existing Crypto Spot Support

Current crypto support is present but limited and partly routed through stock-named abstractions.

Confirmed current behavior:

- `cli.models.AssetType` defines `STOCK` and `CRYPTO`.
- `cli.utils.detect_asset_type()` normalizes the ticker first and classifies canonical symbols ending with `-USD`, `-USDT`, `-USDC`, `-BTC`, or `-ETH` as crypto.
- `cli.utils.filter_analysts_for_asset_type()` removes the fundamentals analyst for crypto.
- `tradingagents.dataflows.symbol_utils.normalize_symbol()` maps known crypto bases quoted in `USD`, `USDT`, or `USDC` to Yahoo-style `BASE-USD`.
- Current known crypto bases in `symbol_utils.py` are `BTC`, `ETH`, `SOL`, `XRP`, `ADA`, `DOGE`, `LTC`, `BCH`, `DOT`, `AVAX`, and `LINK`.
- Supported normalized crypto inputs include forms such as `BTCUSD`, `BTCUSDT`, `BTC-USDT`, `BTC-USDC`, `ETHUSD`, and `ethusdt`, all currently normalized to Yahoo `BASE-USD` where the base is known.
- Slash-pair format such as `BTC/USDT` is not accepted by current `is_yahoo_safe()` validation and is not documented by tests as supported.
- `TradingAgentsGraph.propagate(..., asset_type="crypto")` exists and injects `asset_type` into state.
- `build_instrument_context()` changes wording for crypto assets and explicitly warns agents not to assume company fundamentals are available.
- Bull and bear researchers already adjust labels from `stock` to `asset` when `asset_type` is not `stock`, but their detailed prompt bullets still contain company/competitive-advantage language.
- The current market-analysis path still uses tools named `get_stock_data`, `core_stock_apis`, and Yahoo OHLCV data; crypto spot price history is therefore supported through normalized Yahoo Finance symbols, not through a dedicated crypto exchange data layer.
- No confirmed current code path provides exchange-native spot order book, liquidity depth, exchange volume, stablecoin liquidity, token unlocks, on-chain metrics, or protocol metrics.

Migration guidance: preserve the existing Yahoo-based crypto spot capability first, then organize and strengthen it. Do not remove working `BTC-USD`/`ETH-USD` style analysis while adding exchange-native crypto data.

## Stock-Market Assumptions to Migrate

Do not delete stock-market code blindly. Migrate by coupling level.

### Mostly prompt/text coupling; good early rewrite candidates

- `fundamentals_analyst.py` is explicitly company-fundamentals oriented: financial documents, company profile, financial history, balance sheet, cash flow, and income statement.
- `bull_researcher.py` and `bear_researcher.py` already have partial asset-type labels, but still emphasize company growth potential, competitive advantages, scalability, market saturation, financial instability, and competitors.
- Risk debator prompts still refer to company fundamentals and competitive advantages.
- Trader, research manager, and portfolio manager are more generic but still use investment-plan and market language that should be made crypto-native.
- Report section names in `reporting.py` are generic enough to preserve, but the fundamentals section should be renamed, replaced, or made asset-type-aware when crypto becomes primary.

### Data-layer and schema coupling; isolate before replacing

- Price data tools are named `get_stock_data` and routed through `core_stock_apis`, even though normalized Yahoo crypto symbols can currently be priced.
- Technical indicators are computed through stockstats/Yahoo OHLCV paths, which can remain useful for crypto spot but should be renamed or wrapped so crypto analysis is not described as stock data.
- Fundamental data tools route to company financial statements and should not run in the crypto main path unless a token/protocol-specific replacement exists.
- `TradingAgentsGraph._fetch_returns()` uses yfinance and benchmark alpha logic. It defaults to `SPY` via `_resolve_benchmark()`, which is stock-market-specific and should be rewritten or made asset-type-aware for crypto memory/reflection.
- Default config uses `benchmark_map` with equity regional benchmarks and `SPY` as the default. Crypto should not inherit this as its main benchmark logic.

### Stock-specific data sources to remove, replace, or isolate from crypto main path

- SEC/company-financial style fundamentals from yfinance/Alpha Vantage.
- Insider transactions in the news tool node; crypto may need protocol treasury, unlock, governance, or exchange-flow context instead.
- Equity benchmarks, sectors, analyst/company valuation concepts, dividends, buybacks, EPS, revenue, balance sheets, and cash-flow statements.
- U.S. trading-day assumptions in indicator output such as `N/A: Not a trading day (weekend or holiday)` require review because crypto trades continuously.

## Target Crypto Market Model

Target state, not current fact:

- `crypto_spot`: spot-market analysis for crypto assets and spot trading pairs.
- `crypto_perp`: perpetual-futures analysis for crypto contracts.

The final system should distinguish spot and derivatives evidence in state, prompts, reports, and final decisions. Do not force perpetual futures into stock/fundamentals schemas.

Recommended internal meaning:

- `crypto_spot` focuses on spot price, volume, liquidity, volatility, market structure, BTC/ETH leadership, altcoin beta, stablecoin liquidity, crypto news, regulatory events, protocol events, token unlocks, and on-chain/protocol metrics when reliable data exists.
- `crypto_perp` focuses on perp price, mark price, index price, funding rate, open interest, long/short crowding, basis/premium, liquidation context when reliable, leverage crowding, price/OI divergence, squeeze risk, funding cost, and whether derivatives confirm or diverge from spot.

## Crypto Spot Rewrite Guidance

Preserve and strengthen current spot support in this order:

1. Keep the existing normalized Yahoo spot path working for known assets such as BTC and ETH.
2. Expand symbol handling only with tests. If slash pairs such as `BTC/USDT` are added, update CLI validation, normalization, and tests together.
3. Rename or wrap stock-named abstractions only after confirming all call sites. A non-breaking bridge from current `get_stock_data`/`core_stock_apis` to asset-neutral naming is safer than an immediate mass rename.
4. Make market prompts explicitly crypto-aware: continuous trading, exchange fragmentation, spot liquidity, volatility regimes, BTC/ETH leadership, altcoin beta, and stablecoin liquidity.
5. Replace company fundamentals in the crypto main path with token/protocol fundamentals only when reliable data tools exist. Until then, explicitly mark those fields unavailable rather than fabricating them.
6. Keep the no-data sentinel behavior. Missing OHLCV, indicators, news, on-chain metrics, unlocks, or tokenomics must be reported as unavailable.

## Crypto Perpetual Rewrite Guidance

Current repository gap: no confirmed tools or agent path for perpetual-specific data such as funding, open interest, mark price, index price, basis, long/short ratio, liquidation heatmaps, or leverage crowding.

Recommended target state:

- Add perpetual-futures analysis using the existing agent-creation and tool-routing patterns.
- Keep spot and perp evidence separate in state and reports.
- Add data tools only for vendor-supported, verifiable metrics. Funding, open interest, liquidation data, and long/short ratios must never be inferred from spot OHLCV.
- Design the perp analyst around the relationship between derivatives and spot:
  - Does perp price confirm or diverge from spot trend?
  - Is funding supportive, crowded, or costly?
  - Is open interest expanding with trend or diverging from price?
  - Is basis/premium stretched?
  - Is there squeeze risk from crowded longs or shorts?
- If a chosen vendor lacks a metric, return an explicit unavailable marker and require the agent to state the gap.
- Integrate the new capability into the current graph according to the existing `GraphSetup` pattern rather than creating a separate orchestration framework.

## Agent Rewrite Guidance

Preserve these framework roles:

- Market analyst: rewrite as crypto market/technical/structure analyst.
- Sentiment analyst: preserve multi-source sentiment structure, but review Reddit and StockTwits source relevance for crypto tickers and avoid equity-only subreddit assumptions.
- News analyst: preserve macro/news/prediction-market pattern, but add crypto-native event context when data tools exist.
- Bull researcher and bear researcher: preserve debate mechanism; rewrite thesis criteria for crypto assets and derivatives.
- Research manager: preserve structured synthesis; rewrite rating rationale for crypto spot/perp context.
- Trader: preserve transaction-proposal role; include spot/perp instrument distinction and avoid equity-only position language.
- Risk analysts: preserve aggressive/conservative/neutral debate; rewrite risk criteria for volatility, liquidity, exchange risk, liquidation/squeeze risk, funding cost, and data gaps.
- Portfolio manager: preserve final decision role; require final output to distinguish spot and derivatives evidence.
- Memory/reflection: preserve the mechanism, but replace stock benchmark alpha defaults with crypto-aware benchmark logic before trusting crypto performance lessons.

Agents requiring replacement or isolation:

- Fundamentals analyst should not remain a company-financial analyst in the crypto main path. Replace it with a token/protocol/on-chain/fundamentals analyst when reliable data exists, or isolate it from crypto runs until then.
- Insider-transaction context should be removed from the crypto main path unless replaced with crypto-native, verifiable analogues.

## Data Layer Rewrite Guidance

Use the existing routing and tool mechanism as the migration base.

Recommendations:

- Add asset-market distinctions explicitly rather than overloading `stock` and `crypto` forever. The target markets are `crypto_spot` and `crypto_perp`.
- Preserve current vendor fallback/error behavior. The router currently avoids silent unconfigured-vendor fallback and returns explicit no-data sentinels for market-data gaps; keep that principle.
- Do not force perpetual-futures data into `core_stock_apis` or `fundamental_data`.
- Prefer adding clear crypto categories to the existing `data_vendors`/`tool_vendors` pattern after confirming how config precedence tests should change.
- Keep ticker normalization as the single source of truth, but extend it carefully for exchange-native symbols and slash pairs only with tests.
- Mark missing data explicitly. Funding, open interest, liquidation data, tokenomics, on-chain metrics, and unlock schedules must not be fabricated.
- Review continuous-market date handling. Any “not a trading day” or weekend/holiday assumption should be asset-type-aware for crypto.

## Prompt Rewrite Guidance

Current prompts are embedded in Python files, not external prompt files.

Prompt migration rules:

- Rewrite prompts by agent responsibility, not by simple keyword replacement.
- Preserve report structure and collaboration instructions where they still fit.
- Make all exact numeric claims tool-grounded, following the current verified-market-snapshot principle.
- For crypto spot, emphasize price trend, volume, liquidity, volatility, market structure, BTC/ETH leadership, stablecoins, crypto news, regulatory catalysts, protocol events, and token-specific risks when data exists.
- For crypto perps, emphasize funding, open interest, mark/index, basis/premium, crowding, liquidation context, leverage risk, and spot/perp divergence when data exists.
- Remove or gate company-specific concepts such as earnings, revenue, EPS, dividends, buybacks, SEC filings, balance sheets, and cash-flow statements from the crypto main path.
- Prompt agents to say “data unavailable” when tools return missing data rather than filling gaps with general market knowledge.

## CLI and Configuration Guidance

Current CLI support already detects crypto from normalized symbols and filters out the fundamentals analyst for crypto. Preserve this behavior during migration.

Recommended changes:

- Move CLI defaults and examples from stock-first examples toward crypto-first examples only after tests cover the new defaults.
- Add explicit market selection for `crypto_spot` and `crypto_perp` when the graph and data layer can support both.
- Keep backward compatibility for existing stock mode only if maintainers want it; otherwise isolate stock mode outside the crypto main path.
- Ensure selected analysts and report sections remain consistent when fundamentals is replaced, disabled, or renamed.

## Output Guidance

Preserve the existing markdown report tree and consolidated report writer, but make content crypto-aware.

Target output should:

- Distinguish `crypto_spot` and `crypto_perp` evidence.
- Identify which data came from spot markets versus derivative markets.
- Explicitly state unavailable metrics.
- Avoid company-fundamentals language for crypto assets unless analyzing an equity-like issuer, which is not the primary target.
- Keep final recommendations compatible with the existing signal-processing path until a safer crypto-native action schema is added.

## Testing Guidance

Future rewrite PRs should update or add tests around:

- Symbol normalization for all supported crypto formats.
- Asset/market-type detection for `crypto_spot` and `crypto_perp`.
- CLI analyst filtering and defaults.
- Data-router category selection and vendor fallback/no-data behavior.
- Crypto prompt content checks that prevent company-fundamentals assumptions in crypto mode.
- Report output that distinguishes spot and derivatives sections.
- Graph execution with selected crypto analysts.
- Continuous-market date handling.
- No fabrication behavior for missing funding, open interest, liquidation, on-chain, tokenomics, and unlock data.

Run focused tests after each rewrite slice rather than waiting for a full migration.

## Suggested Rewrite Sequence

1. **Document and lock current behavior.** Add or update tests for existing crypto normalization, crypto asset detection, fundamentals filtering, and `asset_type="crypto"` propagation.
2. **Make naming asset-neutral at the edges.** Introduce wrappers or aliases around stock-named tools/categories without breaking existing call sites.
3. **Rewrite crypto prompts.** Update market, sentiment, news, bull/bear, risk, trader, research-manager, and portfolio-manager prompts to be crypto-aware while preserving graph structure.
4. **Replace or isolate fundamentals.** Remove company-financial fundamentals from the crypto main path; add token/protocol/on-chain analysis only with reliable data tools.
5. **Add `crypto_spot`/`crypto_perp` market selection.** Evolve `asset_type` or add a compatible market-type field after confirming state and CLI coupling.
6. **Add perp data tools and analyst integration.** Use the existing tool-node and graph-setup pattern. Keep spot and perp data separate.
7. **Rewrite benchmark/reflection logic.** Replace SPY/equity benchmark defaults for crypto memory outcomes.
8. **Update reports and docs.** Ensure final reports and CLI text present the project as crypto-market research and analysis.

## Phase 1 Definition of Done

Phase 1 is complete when all of the following are true:

- The project’s primary positioning has migrated from stock markets to crypto markets.
- The original multi-agent analysis workflow still runs end to end.
- Existing crypto spot capability has been preserved, organized, or strengthened.
- The crypto spot analysis path no longer depends on stock-market prompt context.
- The crypto perpetual analysis path has been incorporated into the existing workflow.
- Perpetual-futures analysis can express funding, open interest, basis, mark/index relationship, and squeeze risk when data is available.
- Stock-fundamentals-style logic has been replaced, isolated, or removed from the crypto main path.
- Prompts, reports, and CLI defaults are oriented toward crypto markets.
- Final output can distinguish spot and derivatives markets.
- Missing data is not fabricated.
- Key paths have test coverage.
- Documentation can guide future agents in continuing the rewrite.

## Open Questions

These require maintainer confirmation before large code changes:

- Should stock-market mode remain supported as a secondary mode, or should it be isolated/removed after crypto migration?
- Which crypto data vendors should be authoritative for spot exchange data and perpetual metrics?
- Which ticker formats must be first-class: Yahoo `BTC-USD`, exchange `BTCUSDT`, slash `BTC/USDT`, venue-qualified symbols, or all of them?
- Should perpetual contracts be represented as a separate market type, a separate asset type, or a field in instrument identity?
- What benchmark should memory/reflection use for crypto: BTC, ETH, a crypto index, stablecoin cash return, or configurable per asset?
- Should the final action schema remain Buy/Hold/Sell-style, or should it become spot/perp-aware with direction, leverage, funding tolerance, and invalidation fields?
