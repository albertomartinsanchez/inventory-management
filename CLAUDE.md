# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — Vue 3 frontend + Python FastAPI backend + in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for any `.vue` file creation or significant modification
- **code-reviewer**: After writing significant code
- **Explore**: Codebase searches and structure questions
- **backend-api-test** skill: When writing or modifying tests in `tests/backend/`

### MCP Tools
- **Always use `mcp__github__*`** for all GitHub operations (exception: local branch creation → `git checkout -b`)
- **Always use `mcp__playwright__*`** for browser testing against `http://localhost:3000` (frontend) and `http://localhost:8001` (API)

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI + Pydantic (port 8001)
- **Data**: JSON files in `server/data/` loaded into memory at startup via `server/mock_data.py`

## Commands

```bash
# Backend
cd server && uv run python main.py

# Frontend
cd client && npm install && npm run dev

# Backend tests (run from tests/ directory)
cd tests && uv run pytest backend/ -v

# Single test
cd tests && uv run pytest backend/test_inventory.py::test_filter_by_warehouse -v

# Frontend build
cd client && npm run build
```

## Architecture

### Global Filter System
The 4 filters (Time Period, Warehouse, Category, Order Status) are global state managed by `client/src/composables/useFilters.js` as a **singleton** (shared across all views via module-level refs). Flow:

1. `FilterBar.vue` component updates the singleton via `useFilters()`
2. Each view watches `getCurrentFilters()` and re-fetches on change
3. `api.js` passes filters as `URLSearchParams` query params to FastAPI
4. Backend applies `apply_filters()` (warehouse/category/status) and `filter_by_month()` (supports both direct month `2025-01` and quarters `Q1-2025`)
5. Responses validated by Pydantic models before returning

**Important**: `GET /api/inventory` does **not** support `month` filtering (inventory has no time dimension).

### Frontend State Pattern
- Raw API data lives in `ref()`: `allOrders`, `inventoryItems`, etc.
- Display data lives in `computed()`: filtered views, aggregations, formatted values
- Never mutate raw data refs directly — always replace them with API responses

### Composables
- `useFilters.js` — singleton filter state, `getCurrentFilters()`, `resetFilters()`, `hasActiveFilters`
- `useAuth.js` — mock user/tasks, `getInitials()`, `logout()`
- `useI18n.js` — locale switching (en/ja), translations in `client/src/locales/`

### Backend Data Loading
`mock_data.py` loads all 7 JSON files once at startup. Data is read-only during runtime — changes to JSON require a server restart. The `purchase_orders.json` file exists but the purchase order feature is partially implemented.

### i18n
Two locales supported: English (`en.js`) and Japanese (`ja.js`) in `client/src/locales/`. All user-facing strings should use `useI18n()` — do not hardcode display text in components.

### Currency
`client/src/utils/currency.js` handles USD/JPY formatting with a fixed 150 exchange rate. Use `formatCurrency()` for display values.

## API Endpoints
- `GET /api/inventory` — filters: `warehouse`, `category`
- `GET /api/orders` — filters: `warehouse`, `category`, `status`, `month`
- `GET /api/dashboard/summary` — all filters
- `GET /api/demand`, `GET /api/backlog` — no filters
- `GET /api/spending/summary|monthly|categories|transactions`
- `GET /api/reports/quarterly|monthly-trends`
- `GET /api/orders/{order_id}`, `GET /api/inventory/{item_id}` — return 404 if not found

## Data Reference

**Warehouses**: San Francisco, London, Tokyo (filters use `warehouse` param)  
**Categories**: Circuit Boards, Sensors, Actuators, Controllers, Power Supplies  
**Order statuses**: Delivered, Shipped, Processing, Backordered  
**Revenue goals**: $800K/month (single month selected), $9.6M YTD (all months)

## Tests
- 55 tests in `tests/backend/` using pytest + FastAPI `TestClient`
- `conftest.py` provides `client` fixture (TestClient) + sample data fixtures
- Tests validate actual calculations, not hardcoded values — assertions check ranges and logic
- No frontend tests configured

## Design System
- Colors: `#0f172a` (dark), `#64748b` (mid), `#e2e8f0` (light)
- Status colors: green (Delivered), blue (Shipped), yellow (Processing), red (Backordered)
- Charts: Custom SVG with `viewBox` for responsiveness — no charting library
- Layouts: CSS Grid for dashboards, flexbox for components
- No emojis in UI

## Code Style
- Add a short inline comment whenever logic is non-obvious — workarounds, subtle invariants, unexpected constraints, or calculations that would surprise a reader

## Common Pitfalls
1. Use unique keys in `v-for` — use `sku`, `order_id`, `month`, never `index`
2. Validate dates before calling `.getMonth()` — JSON dates may be null
3. Update Pydantic models in `main.py` whenever JSON data structure changes
4. `useFilters` is a singleton — calling it in multiple components shares the same state (intentional)
