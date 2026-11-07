# fks-mcp

The FKS edge ladder as MCP tools, so an agent (Claude Code, Claude Desktop) can
do edge research through the platform's API v1 (fks-web `/api/v1`) with a
personal access token.

| Tool | What it does |
|---|---|
| `ladder` | edges, hypotheses (searched or not), experiments with verdicts, deployments with health, trials spent |
| `hypothesis_get` / `experiment_get` | one record in full |
| `datasets` | the lake folders a hypothesis can name, and the dataset string format |
| `alerts` | open alerts |
| `create_edge` | a new edge, status research |
| `register_hypothesis` | a hypothesis, **frozen at registration** |
| `start_search` | the `job.edge-search` instance `edge-search-h<id>` (family inferred: `raw/cme/…` → orb; a perpetual folder joined with its venue's spot folder, `raw/kucoin/futures/…+raw/kucoin/spot/…` → carry; else trend-band. Pass `family` when unsure: a wrong one fails at load and spends the instance name) |
| `search_status` | container state, the last log lines, the experiment once written |

Nothing deletes, promotes a rung, arms live trading or touches a key: those
stay in the web UI. The token's role caps the rest (tokens are never admin),
and fks-web caps a token's edge searches per day (`FKS_TOKEN_SEARCHES_PER_DAY`,
default 5).

## Install

1. Web UI → Settings → API tokens → create an **operator** token (`fks_pat_…`).
2. Install and register with Claude Code (user scope):

```sh
python3 -m venv ~/.local/share/fks-mcp
~/.local/share/fks-mcp/bin/pip install git+https://github.com/nuniesmith/fks-mcp
claude mcp add fks --scope user -e FKS_TOKEN=fks_pat_… -- ~/.local/share/fks-mcp/bin/fks-mcp
```

(`pipx install git+https://github.com/nuniesmith/fks-mcp` works too; then the
command is just `fks-mcp`.) Update with `pip install -U git+…` into the same
venv. `FKS_URL` defaults to `https://oryx.tailfef10.ts.net/api/v1` (tailnet
only). The token lands in `~/.claude.json`: revoke it in the web UI if that
file leaks.

## Develop

```sh
python3 -m venv .venv && .venv/bin/pip install -e '.[test]' && .venv/bin/pytest -q
```

The tests run against `httpx.MockTransport`; nothing reaches the platform.
`docs/MUSE_TEST_PLAN.md` is the live test against the real platform.
