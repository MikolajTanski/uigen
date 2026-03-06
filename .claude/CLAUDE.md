# Project Instructions

## Environment
- Dev server is already running on http://localhost:3000
- Logs are written to `logs.txt` in the project root — check it for runtime errors
- Database: SQLite at `prisma/dev.db` — query with `sqlite3 prisma/dev.db "<SQL>"`

## Tools available
- **MCP Playwright** — use `mcp__playwright__*` tools to launch browser, navigate, click, screenshot, etc.
- **sqlite3 CLI** — query the DB directly via Bash: `sqlite3 prisma/dev.db "SELECT * FROM User;"`

## Workflow
- Do NOT run `next dev` directly — use `npm run dev` (includes required `NODE_OPTIONS`)
- Do NOT write to `.env` — secrets are managed outside version control
- Check `logs.txt` before debugging runtime issues
- Use Playwright tools to verify UI changes visually before considering a task done
