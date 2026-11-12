# Paper-to-Strategy Research Orchestrator: Plan

Goal: systematically mine academic working papers for trading strategies on index futures, and
produce three things per paper:

1. **The idea**: why the effect should exist, kept in a reusable library (`ideas/`).
2. **The strategy**: coded in Python, replicated against the paper and tested out of sample.
3. **A PineScript port**: only for strategies that survive validation.

## Decisions taken

| Topic | Decision |
|---|---|
| Sources | SSRN, arXiv q-fin, NBER, Fed / central bank papers, open-access journals and author sites |
| Scope | Index futures (ES/NQ style), intraday to swing (minutes to days) |
| Validation | Backtest, replicate the paper, then out-of-sample and cost checks; Pine only for survivors |
| Run mode | Backlog sweep of ~50 best papers, then a weekly scheduled run that opens a pull request |
| Idea library | One idea card per paper, plus cross-paper theme pages |
| SSRN-only papers | Triage from the abstract, queue for manual PDF download, process when the PDF arrives |

## Pipeline

Each paper moves through a state machine recorded in `registry/papers.jsonl`:

`discovered → triaged → (needs_pdf) → ready → extracted → coded → validated → ported → done`
(with `rejected` possible at triage, and a verdict attached at `validated`).

| # | Stage | What happens | Output |
|---|---|---|---|
| 1 | Discover | One scout per source pulls titles, abstracts, dates, links. Deduplicate across sources by DOI / title. | registry rows |
| 2 | Triage | Score each abstract 0–5 on: fits scope, rules can be specified, data obtainable, has an economic rationale, evidence quality. | score + reason; top N advance |
| 3 | Acquire | Look for free full text (arXiv, NBER, author site, open-access copy). If none: add to `queue/needs_pdf.md`. | text, or queue entry |
| 4 | Extract | A reader agent reads the full paper once and writes **two separate artefacts**: the strategy spec and the idea card. | `spec.yaml`, idea card |
| 5 | Code | A coder agent implements the spec: pure signal logic in one module, plus a QuantConnect (LEAN, Python) wrapper. | `strategies/<id>/python/` |
| 6 | Validate | A *different* agent reviews for look-ahead bias, then runs the checks below. | `report.md` + verdict |
| 7 | Port | For passing, single-chart strategies: Pine v6 `strategy()` script mirroring the signal module. | `strategies/<id>/pine/` |
| 8 | Synthesise | A theme agent folds new idea cards into theme pages and updates the index. | `ideas/themes/` |
| 9 | Report | Run summary: papers processed, verdicts, what needs your attention. | `reports/<date>.md`, PR |

The orchestrator itself only manages state and dispatches agents; it never reads papers, so its
context stays small across a 50-paper run.

### Validation checks (stage 6)

1. **Replication**: same sample period as the paper. Pass if the sign matches and the Sharpe ratio
   is within roughly half of the reported value. If it fails, the spec is re-read once before the
   paper is marked `not_replicated`.
2. **Out of sample**: the period after the paper's sample ends (ideally after publication).
3. **Costs**: default 1 tick slippage per side plus commission; report gross and net.
4. **Sensitivity**: vary each main parameter around the paper's value; flag results that only work
   at one setting.
5. **Verdict**: `survives`, `decayed` (worked in sample, not after), `not_replicated`, or
   `untestable` (data unavailable). Thresholds live in `orchestrator/rubric.yaml` and are tunable.

Every verdict is kept, including failures: a decayed strategy is still useful evidence in the idea library.

## The idea library

**Idea card** (`ideas/cards/<paper_id>.md`), written for every fully-read paper regardless of whether the strategy works:

- **Claim**: the effect in one sentence.
- **Mechanism**: why it should exist, classified as risk premium, behavioural bias, structural flow
  (hedging, rebalancing, mandates), information, or microstructure.
- **Counterparty**: who is on the losing side and why they keep trading that way.
- **Persistence**: what stops it being arbitraged away.
- **Conditions**: when it should be strongest, and what would make it stop working.
- **Evidence**: sample, robustness, data-mining risk, known post-publication decay.
- **Transfer**: how it maps to ES/NQ if the paper studied something else.
- **Spin-offs**: untested variations the mechanism suggests.
- Links: paper, strategy folder, validation verdict, related themes.

**Theme pages** (`ideas/themes/<theme>.md`) merge cards that share a mechanism (for example
overnight drift, dealer hedging flows, intraday momentum, macro announcement effects): the common
logic, which papers support or contradict it, which implementations survived, and combination ideas.
`ideas/INDEX.md` lists every card by mechanism, horizon and verdict.

## Repository layout

```
orchestrator/      agent prompts, triage rubric, spec + card schemas, run script
registry/          papers.jsonl (one row per paper, current state)
queue/             needs_pdf.md (papers waiting for you)
inbox/pdfs/        drop downloaded PDFs here (not committed)
ideas/             cards/, themes/, INDEX.md
strategies/<id>/   spec.yaml, python/, pine/, backtests/, report.md
reports/           one summary per run
```

PDFs are never committed (copyright); the registry stores links and metadata only.

## Rollout

| Phase | Content | Checkpoint |
|---|---|---|
| 0 | Scaffold repo, schemas, prompts, source scouts; confirm network and data access | – |
| 1 | Screen ~2–3k abstracts, shortlist ~150, fully process the first 10 | You review cards, code and reports; rubric and prompts get adjusted |
| 2 | Process the remaining ~40 | Theme pages built; summary report |
| 3 | Weekly scheduled run: new papers since last run, plus anything in `inbox/pdfs/` | Draft PR each week |

## Known risks and open points

- **SSRN access**: SSRN restricts automated access. Plan is to get SSRN metadata through open
  scholarly indexes where possible and fall back to the manual PDF queue. How well this covers
  SSRN is unverified until phase 0.
- **Network allowlist**: the cloud environment must allow the source hosts (arXiv, NBER, Fed sites,
  metadata indexes). To be requested in phase 0.
- **Futures data**: replication depends on QuantConnect providing ES/NQ intraday history on your
  account tier. To be confirmed in phase 0; papers needing unavailable data (e.g. options flow) get
  `untestable` and an idea card only.
- **Pine cannot be run by agents**: ports are checked by code review against the Python signal
  module. A true parity check needs you to export a TradingView trade list for comparison.
- **Publication bias**: most published effects weaken after publication, so expect the majority of
  verdicts to be `decayed` or `not_replicated`. The out-of-sample step exists for that reason.
