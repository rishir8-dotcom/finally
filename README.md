# FinAlly — AI Trading Workstation

A visually stunning AI-powered trading workstation that streams live market data, simulates portfolio trading, and integrates an LLM chat assistant that can analyze positions and execute trades via natural language.

Built entirely by coding agents as a capstone project for an agentic AI coding course. See [`planning/PLAN.md`](planning/PLAN.md) for the full specification.

## Status

The market data subsystem (simulator, SSE streaming, price cache) is complete — see [`planning/MARKET_DATA_SUMMARY.md`](planning/MARKET_DATA_SUMMARY.md). The rest of the platform (portfolio, AI chat, frontend, Docker packaging) is still in development.

## Planned Architecture

Single Docker container serving everything on port 8000:

- **Frontend**: Next.js (static export) with TypeScript and Tailwind CSS
- **Backend**: FastAPI (Python/uv) with SSE streaming
- **Database**: SQLite with lazy initialization
- **AI**: Anthropic API (Claude Sonnet 5) with structured outputs
- **Market data**: Built-in GBM simulator (default) or Massive API (optional)

## Backend Quick Start

```bash
cd backend
uv sync --extra dev

uv run market_data_demo.py   # live terminal dashboard with simulated prices
uv run --extra dev pytest -v # run tests
```

See [`backend/README.md`](backend/README.md) and [`backend/CLAUDE.md`](backend/CLAUDE.md) for details.

## Environment Variables

Copy `.env.example` to `.env` and fill in as needed:

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key for AI chat |
| `MASSIVE_API_KEY` | No | Massive (Polygon.io) key for real market data; omit to use the simulator |
| `LLM_MOCK` | No | Set `true` for deterministic mock LLM responses (testing) |

## Project Structure

```
finally/
├── frontend/    # Next.js static export (planned)
├── backend/     # FastAPI uv project
├── planning/    # Project documentation and agent contracts
├── test/        # Playwright E2E tests (planned)
├── db/          # SQLite volume mount (runtime)
└── scripts/     # Start/stop helpers (planned)
```

## License

See [LICENSE](LICENSE).
