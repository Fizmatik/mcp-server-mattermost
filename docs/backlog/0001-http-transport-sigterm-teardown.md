---
title: "HTTP-транспорт: teardown lifespan не выполняется при SIGTERM"
summary: На `--http` шатдаун по SIGTERM не доводит lifespan до конца, поэтому
  общий HTTP-пул и auth-провайдер не закрываются, а логи шатдауна не пишутся.
  `docker stop` шлёт именно SIGTERM. На stdio и по SIGINT всё отрабатывает.
category: bug
created: 2026-09-04
files:
  - src/mcp_server_mattermost/__init__.py
  - src/mcp_server_mattermost/server.py
  - src/mcp_server_mattermost/http_pool.py
---

Баг не наш: воспроизводится на голом `FastMCP` с пустым lifespan, без единой
строки кода проекта. Проверен на 3.4.4.

**Матрица:**

| Запуск | Сигнал | Teardown |
| --- | --- | --- |
| `mcp.run(transport="http")` — то, что делает `main()` | SIGTERM | не выполняется вообще, даже `finally` |
| `mcp.run(transport="http")` | SIGINT | выполняется полностью |
| `uvicorn.run(mcp.http_app())` | SIGTERM | выполняется полностью |
| stdio (`mcp.run(transport="stdio")`) | EOF на stdin | выполняется полностью |

По SIGTERM uvicorn пишет `Waiting for application shutdown.`, затем сразу
`Finished server process` — lifespan он не дожидается.

**Следствия:** `HTTP connection pool closed` и `Mattermost MCP server shutdown
complete` не пишутся; `_close_auth_provider` не вызывается. Сокеты забирает ОС
при выходе процесса, так что утечки за пределы процесса нет — ломается именно
graceful shutdown, и для всего, что кто-либо положит в lifespan, а не только
для пула.

**Проверка:**

```bash
MATTERMOST_URL=http://127.0.0.1:59999 MATTERMOST_TOKEN=t MATTERMOST_LOG_FORMAT=text \
  uv run mcp-server-mattermost --http --port 8811 2>&1 | tee /tmp/probe.log &
sleep 6 && kill -TERM $(pgrep -f "mcp-server-mattermost --http" | tail -1)
grep "connection pool" /tmp/probe.log   # ожидается пара created/closed, будет только created
```

**Починку проверять на всех трёх путях:** `--http`, stdio, `fastmcp run`.
Кандидаты — поднимать uvicorn самим через `mcp.http_app()` вместо
`mcp.run(transport="http")`, либо завести issue в PrefectHQ/fastmcp.
