# FKS MCP: test plan for Muse

**What this is:** a guided test of the `fks` MCP server and the FKS edge ladder behind it.
**Where:** oryx. The server is the `fks-mcp` package from [nuniesmith/fks-mcp](https://github.com/nuniesmith/fks-mcp), installed in the venv `~/.local/share/fks-mcp`. Claude Code runs it at user scope as `fks`, with the command `~/.local/share/fks-mcp/bin/fks-mcp`. The source checkout is `~/github/fks-mcp`, and this file is `docs/MUSE_TEST_PLAN.md` in it.
**Before phase 0:** the `fks` tools only load when a session starts, so run this plan in a **new** Claude Code session on oryx.
**How long:** about 20 minutes for phases 0–2. Phase 3 runs one real search, which takes a few minutes.

---

## Before you start: the rules

The ladder is the platform's permanent research record, not a sandbox. A few writes can't be undone, so take these seriously.

1. **A hypothesis is frozen and permanent.** Once it is registered, nothing can edit or delete it. The web UI can only *withdraw* it. Only register what this plan tells you to.
2. **Every search spends trial budget.** Each search adds its trials to the effort count for that family and symbol. The deflated-Sharpe gate (U7) uses that count, so a careless test makes later real research harder to pass. Phase 3 runs **one** search with **4** trials, and only after the user says go.
3. **Never touch the sealed futures holdout,** which runs 2026-04-01..2026-09-11. Don't register any hypothesis whose dataset covers those dates on `raw/cme/massive-cache`. Don't run any futures (orb) search at all: that family is closed (experiment 1).
4. **Don't try to use up the daily search cap.** Tokens get 5 searches a day, and testing the cap would burn real searches.
5. **Report verdicts exactly as they come back.** `DO NOT PASTE` is a pass of the *process*, not a failure of the test.
6. **Never paste, log or echo the token.** It lives in `~/.claude.json`.

There are no tools for deleting, promoting, going live or handling keys. That is deliberate. If a step seems to need one, stop and report it.

---

## Phase 0 — Connection (no writes)

| # | Do | Expect |
|---|---|---|
| 0.1 | Run `claude mcp get fks` | `Status: ✔ Connected`, `Command: /home/jordan/.local/share/fks-mcp/bin/fks-mcp`, no args, and only `FKS_TOKEN` in the environment (don't print its value) |
| 0.1b | Run `~/.local/share/fks-mcp/bin/pip show fks-mcp` | `Version: 0.1.0` |
| 0.2 | List your tools | Exactly 9: `ladder`, `hypothesis_get`, `experiment_get`, `datasets`, `alerts`, `create_edge`, `register_hypothesis`, `start_search`, `search_status` |
| 0.3 | Read the server's instructions | They describe the process: edge → frozen hypothesis → search → verdict |

## Phase 1 — Reads (no writes)

| # | Do | Expect |
|---|---|---|
| 1.1 | `ladder` | At least 1 edge (`orb-cme`), hypothesis 1, experiment 1 with verdict `DO NOT PASTE`, and 0 deployments |
| 1.2 | `hypothesis_get(1)` | The full frozen spec. The dataset is `raw/cme/databento+raw/cme/massive/cme 1m 2014-09-02..2024-09-06`, the symbols are MES/MNQ/MGC, and the costs include `contracts`/`account_usd`/`trail_usd` |
| 1.3 | `experiment_get(1)` | `verdict_reasons` names U5, U7 and U9. There are 50 trials, the selection score is negative, and the holdout Sharpe is about 0.84 |
| 1.4 | `search_status(1)` | Instance `edge-search-h1`, state `exited`, a `log_tail` of real log lines, and experiment 1 listed |
| 1.5 | `search_status(1, log_lines=500)` | Works: the server caps it at 200 |
| 1.6 | `datasets` | Two `cme_futures` folders (the second one carries the sealed-holdout warning), any recorder folders under `recorded_by_instances`, and the dataset string format |
| 1.7 | `alerts` | A JSON list (it may be empty). Note anything that is firing |
| 1.8 | `hypothesis_get(9999)`, `experiment_get(9999)` | `FKS refused: no hypothesis 9999` / `no experiment 9999`, as text, not a crash |

## Phase 2 — Refusals (no writes should land)

Each of these must come back as `FKS refused: …` in the platform's own words. Afterwards, `ladder` must look exactly as it did in phase 1.

| # | Do | Expect |
|---|---|---|
| 2.1 | `create_edge(edge_id="mcp-bad", name="Bad", edge_type="janus", assets=[])` | Refused: `unknown edge_type 'janus' (supported: adaptive, rule)` |
| 2.2 | `register_hypothesis` on `orb-cme` with `metric: "sharpe"`, `evidence: "because"`, `idea: "short"` | Refused, listing **several** errors at once (idea too short, metric, evidence) |
| 2.3 | `register_hypothesis` on an edge that doesn't exist (`no-such-edge`), with an otherwise valid body | Refused: the edge isn't known |
| 2.4 | `start_search(9999)` | `FKS refused: no hypothesis 9999`. No instance is created |
| 2.5 | Run `ladder` again | Same counts as in 1.1. Nothing landed |

## Phase 3 — One real write path (ask the user first)

**Stop and ask the user before 3.1.** These steps create one edge, one permanent hypothesis and one search (4 trials, on SOLUSDT, which nothing else in the ladder uses).

**3.1** `create_edge`:

```json
{"edge_id": "mcp-smoke", "name": "MCP smoke test", "edge_type": "rule", "assets": ["SOLUSDT"]}
```

Expect the edge back with status `research`.

**3.2** `register_hypothesis` on `mcp-smoke`:

```json
{
  "idea": "MCP smoke test: the trend-band rule on SOL daily, a 4-point grid around the live 200d/5% rule. Not a research claim.",
  "symbols": ["SOLUSDT"],
  "dataset": "raw/binance/spot/spot-history 1d 2020-09-01..2026-10-31",
  "holdout_share": 0.25,
  "params": [
    {"name": "lookback", "choices": [100, 200]},
    {"name": "band", "choices": [0.03, 0.05]}
  ],
  "baseline": "trend-200d-5pct",
  "metric": "sharpe after fees",
  "costs": {"maker_bps": 40, "taker_bps": 80, "slippage_bps": 5},
  "evidence": "trades"
}
```

Expect:
- a new hypothesis id (call it **N**);
- `registered_by` set to the token's user, whatever you sent;
- `ladder` shows it with `searched: false`.

**3.3** `start_search(N)`. Expect `instance: edge-search-hN`, `family: trend-band` (inferred from the dataset) and `started: true`.

**3.4** Poll `search_status(N)` about once a minute. Expect:
- the state goes `running` → `exited`;
- the log tail shows the search loading bars and scoring the grid;
- in the end, one experiment for hypothesis N with 4 trials.

**3.5** `experiment_get(<that id>)`. Report the verdict and the `verdict_reasons` word for word. **Most likely `DO NOT PASTE`**, because one grid point *is* the baseline and the grid is tiny. That is the expected and correct outcome.

**3.6** Call `start_search(N)` a second time. Expect a refusal, because the instance already exists. A search can't be silently re-run under the same name. It must not create a second experiment.

**3.7** Run `ladder`. Expect hypothesis N with `searched: true`, and `trials_per_family` showing trend-band SOLUSDT with 4 trials.

After you're done, the user can withdraw hypothesis N in the web UI (Edges → mcp-smoke).

## Phase 4 — Things to look for

Report anything you notice, especially:
- a tool description that disagrees with what the server accepted or refused;
- a tool that hung (`search_status` reads a log stream for about 4 seconds at most), or an answer far too long to be useful;
- a refusal message that didn't tell you how to fix the request;
- a problem worth a fix: write it up as an issue on nuniesmith/fks-mcp, or for the platform itself on fks-web or fks-spawner, and quote the exact tool call and reply;
- anything you wanted to do that had no tool and isn't on the deliberately-excluded list (delete, promote, live, keys).

---

## Report template

```
FKS MCP test — <date>, Muse
Phase 0: pass/fail — notes
Phase 1: 1.1 … 1.8 pass/fail — notes
Phase 2: 2.1 … 2.5 pass/fail — exact refusal texts
Phase 3: run? (user OK'd: yes/no) — hypothesis N=…, experiment id=…, verdict=…, reasons=…
         3.6 second start: …
Phase 4: findings
Ladder before/after: edges a→b, hypotheses c→d, experiments e→f
```
