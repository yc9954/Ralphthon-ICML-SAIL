<h1 align="center">ICML SAIL with Ralph</h1>

<p align="center">
  <img src="https://img.shields.io/badge/React%2018%20%C2%B7%20TypeScript%206%20%C2%B7%20Vite%208-C15F3C?style=flat" alt="React 18, TypeScript 6, Vite 8" />
  <img src="https://img.shields.io/badge/Tailwind%203%20%C2%B7%20Recoil%20%C2%B7%20TanStack%20Query-C15F3C?style=flat" alt="Tailwind 3, Recoil, TanStack Query" />
  <img src="https://img.shields.io/badge/adapter-FastAPI%20%C2%B7%20Python%203.12-4493F8?style=flat" alt="FastAPI adapter on Python 3.12" />
  <img src="https://img.shields.io/badge/heads-Claude%20reviewers%20%2B%20VESSL%20Qwen3--8B%20LoRA-4493F8?style=flat" alt="Claude reviewer heads and a VESSL Qwen3-8B LoRA meta-review head" />
</p>

<p align="center">
  <sub><a href="docs/readme/README.ko.md">한국어</a></sub>
</p>

<p align="center">
  <strong>An Area-Chair-style review loop for your paper, run the way ICML runs it.</strong><br/>
  Submit a manuscript, get three reviewer reviews with no score attached, argue back in a rebuttal thread,<br/>
  accept or reject AI-drafted revision hunks, and only at <em>finalize</em> receive the AC meta-review, a 0–100<br/>
  selectivity score and an accept/reject decision. Resubmit and the next cycle starts fresh.
</p>

<h3 align="center"><a href="#getting-started"><ins>Getting started</ins></a> · <a href="sail-spec/00-INDEX.md">Rebuild spec</a> · <a href="sail-spec/assets/review-agent.md">Frozen review agent</a></h3>

## Features

### Reviews first, score last

A cycle mirrors a real venue: `submit` produces three reviewer reviews (ICML 1–10 rating, summary, and in live mode a full-length body plus confidence, soundness, presentation and contribution). No score exists yet. The score, `gradeTier`, feature attributions and the deficiency report appear only after `finalize`, together with the AC meta-review and the decision post. `SELECT_THRESHOLD` is 88.

### A rebuttal thread, not a chat log

Comments are structured issues (`major` / `minor` / `question`, with a section label and the reviewer who raised them). Replying to a comment sends the author's message back to that reviewer, who follows up. Every allow/deny decision on a revision hunk is logged into the thread as rebuttal text, so the AC sees the whole discussion.

### Hunk-level AI revision

`revision-draft` asks the agent for before/after hunks, each with a rationale and the comment ids it addresses. You approve or deny hunk by hunk; `revision-apply` rewrites the manuscript with the allowed ones and attaches the revised draft to your next message. The manuscript pane highlights what changed. Direct manuscript edits are supported too.

### Analysis view grounded in a real corpus

`/review/:id/analysis` draws the score bottleneck (backbone activations → score → three heads) and places your paper's cycle scores on a histogram of **47,209 real ICLR/ICML/NeurIPS/UAI submissions (2018–2026)** with measured tier medians: reject 25, poster 43, spotlight 59, oral 71.8, notable-top-5% 86.7. Hovering an attribution row highlights the manuscript sentences behind it.

### Works with or without a backend

Leave `VITE_RALPH_API_URL` unset and the built-in mock runs the entire loop deterministically (cycle scores 63 → 79 → 91 → 96), streaming step events and persisting every paper, including uploaded PDF blobs, to IndexedDB (`sail-ralph`). Set it and every call maps 1:1 onto `serve/sail_adapter.py`.

**Also included**

- **`serve/sail_adapter.py`**: the single-file FastAPI backend (contract v2). With `ANTHROPIC_API_KEY` it runs three parallel Claude reviewer agents, a reviewer-reply agent, a VESSL-served Qwen3-8B LoRA meta-review and score head, a Claude attribution head and a Claude revision agent; without it every head degrades to a deterministic fallback. It also exposes `POST /api/score` as a thin proxy to the trained score head.
- **`sail-spec/`**: a self-contained rebuild specification: verbatim source units for every front-end module, the backend contract, GCE and VESSL runbooks, the GPU training plan, golden transcripts from a live run, and the hackathon Track 2 protocol.
- **Terminal review skill**: `sail-spec/assets/review_paper.py` runs one paper through the same backend path as the web app and renders the official Track 2 review template; `agent_review_submit.py` is the Track 1 submission harness for the openagentreview.org review window.
- **Paper-tone light and dark themes**, `⌘B` sidebar collapse, and a sidebar history of every submission with status dots.

---

## How it works

```text
Browser (Vite + React 18, Recoil UI state, TanStack Query server state)
   │  src/api/reviewLoop.ts — the single type contract
   ├── VITE_RALPH_API_URL unset ──▶ mock loop in the same file ──▶ IndexedDB (papers + PDF blobs)
   └── VITE_RALPH_API_URL set ────▶ serve/sail_adapter.py (FastAPI, :8100 locally)
                                       ├─ review head ........ 3 parallel Claude reviewers
                                       ├─ reply head ......... Claude, grounded in that reviewer's review + thread
                                       ├─ meta-review+score .. VESSL Qwen3-8B LoRA  (/meta-review, /score)
                                       ├─ attribution head ... Claude, verbatim evidence sentences
                                       ├─ revision agent ..... Claude, grounded hunks only
                                       └─ state ............. SAIL_STATE_PATH (sail_state.json)
```

1. **Submit** (`POST /api/loop/papers`, multipart `title` + `text` or PDF `file`). PDFs are text-extracted server-side (PyMuPDF). Cycle 1 opens with three reviews; the UI streams the agent's step events from `/api/loop/jobs/{id}`.
2. **Discuss.** `reply` adds an author message (optionally targeted at a comment) and gets reviewer follow-ups. `revision-draft` / `revision-apply` produce and apply hunks; `manuscript` accepts direct edits.
3. **Finalize.** The AC meta-review is written from the reviews plus the whole discussion. Score = calibrated `p_accept` (p^0.25, anchored to visible ratings) or, when `SAIL_SCORE_HEAD=1`, the trained regression head. Decision, tier, attributions and a deficiency report ("why the score stopped here, path to the next band") come back together.
4. **Resubmit** starts cycle N+1 on the revised manuscript with fresh reviews and an empty thread. Status moves between `in_discussion` and `decided`.

<details>
<summary><strong>Adapter environment</strong></summary>

| Variable | Default | What it does |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | unset | Turns on the live heads (`LIVE=True`). Without it all heads use deterministic fallbacks. |
| `SAIL_CLAUDE_MODEL` | `claude-opus-4-8` | Model for reviewer, reply, attribution and revision agents. |
| `VESSL_META_URL`, `VESSL_META_MODEL` | VESSL endpoint, `v2` | Meta-review and score serving. |
| `SAIL_SCORE_HEAD` | `1` | `0` reverts to the original `p_accept` calibration path. |
| `SAIL_VENUE` | `icml` | Reviewer bar: `icml` (main conference) or `workshop` (4-page events; marked unvalidated). |
| `SAIL_FIELD_CONTEXT`, `TOPIC_MATURITY_PATH` | `1`, `/opt/sail/topic_maturity.json` | Recency correction from `build_topic_maturity.py`. |
| `SAIL_STATE_PATH` | `sail_state.json` | Persistent loop state. |
| `WEB_DIST` | `/opt/sail/web` | Built SPA served by the same process in deployment. |

</details>

---

## Tech stack

<p>
  <kbd>React&nbsp;18</kbd> &nbsp; <kbd>TypeScript&nbsp;6</kbd> &nbsp; <kbd>Vite&nbsp;8</kbd> &nbsp; <kbd>Tailwind&nbsp;3</kbd> &nbsp; <kbd>Recoil</kbd> &nbsp; <kbd>TanStack&nbsp;Query&nbsp;5</kbd> &nbsp; <kbd>react-router&nbsp;7</kbd> &nbsp; <kbd>Radix&nbsp;UI</kbd> &nbsp; <kbd>react-markdown</kbd> &nbsp; <kbd>IndexedDB</kbd> &nbsp; <kbd>oxlint</kbd> &nbsp;
  <kbd>FastAPI</kbd> &nbsp; <kbd>Python&nbsp;3.12</kbd> &nbsp; <kbd>PyMuPDF</kbd> &nbsp; <kbd>Anthropic&nbsp;SDK</kbd> &nbsp; <kbd>VESSL&nbsp;serving</kbd> &nbsp; <kbd>GCE&nbsp;+&nbsp;GCS</kbd>
</p>

---

## Getting started

**Prerequisites**

- Node.js 20+ and npm for the web app.
- Python 3.10+ (the Dockerfile uses 3.12) for the adapter; an Anthropic API key and a reachable VESSL endpoint for live mode.

```bash
git clone https://github.com/yc9954/Ralphthon-ICML-SAIL.git
cd Ralphthon-ICML-SAIL
npm install
npm run dev                 # http://localhost:5199 — mock mode, no backend needed
```

To connect the adapter instead of the mock:

```bash
# backend
cd serve && pip install -r requirements.txt
ANTHROPIC_API_KEY=... uvicorn sail_adapter:app --port 8100     # omit the key for fallback heads

# frontend
echo 'VITE_RALPH_API_URL=http://localhost:8100' > .env.local   # 8000 is avoided: it collides with a local Sophy
npm run dev
```

| Process | Port | Notes |
| --- | --- | --- |
| Vite dev server | `5199` | set in `vite.config.ts` |
| `sail_adapter.py` (uvicorn) | `8100` locally, `8080` in the Docker image | `PORT` env in the container |

**Terminal review of one paper** (same pipeline as the web app, official Track 2 template):

```bash
python3 sail-spec/assets/review_paper.py papers/foo.md --base http://localhost:8100 [--selfreview papers/foo.selfreview.md] [--keep]
```

---

## Building and testing

```bash
npm run build     # tsc -b && vite build → dist/
npm run lint      # oxlint
npm run preview
```

There is no automated test suite. The acceptance gates are manual checklists: `sail-spec/golden/flows.md` §A (mock flow, eight items) and §B (adapter fallback end-to-end), with the JSON files in `sail-spec/golden/` as live transcripts whose fields and invariants, not values, are the reference.

---

## Screens

| Route | What it does |
| --- | --- |
| `/review` | Submit a title plus pasted text or a PDF; lists previous submissions. |
| `/review/:paperId` | The loop: reviews, rebuttal thread with per-comment replies, revision hunks, finalize and resubmit, with the manuscript pane on the right (serif text render or PDF embed, 360–960 px resizable, change highlights). |
| `/review/:paperId/analysis` | Bottleneck diagram and the real-corpus distribution with your paper's cycle trajectory. |
| `/settings` | Privacy and data-flow card; remote compute and Modal cards (desktop-app placeholders). |

---

## Repository structure

| Path | What lives there |
| --- | --- |
| `src/api/reviewLoop.ts` | The domain contract (types, `SELECT_THRESHOLD`, `loopApi`) and the deterministic mock. |
| `src/api/reviewLoopQueries.ts`, `loopStorage.ts` | TanStack Query hooks; IndexedDB persistence for mock mode. |
| `src/app/` | `router.tsx`, `layout/AppShell.tsx`, `providers/ThemeProvider.tsx`, `routes/` (`ReviewLoopPage`, `AnalysisPage`, `SettingsPage`, `NotFound`). |
| `src/components/` | `review/ManuscriptPane`, `analysis/` (`BottleneckDiagram`, `CorpusDistribution`, `ReviewTabs`), `sidebar/`, `settings/`, `ui/`. |
| `src/data/corpusDistribution.ts` | The 47,209-paper histogram (50 bins) and tier medians. |
| `serve/` | `sail_adapter.py`, `requirements.txt`, `Dockerfile`. |
| `sail-spec/` | `00-INDEX.md` (map and ten common principles), `CLAUDE.md` (session entry point), `STATE.md` (living checklist), `GOAL.md` (mission prompts), numbered units for foundation, design tokens, contracts, API, data, state, UI, wiring, backend, ops (GCE, VESSL), training (GPU plan), harness, hackathon Track 2, `golden/` transcripts, `assets/` (`review_paper.py`, `agent_review_submit.py`, `build_topic_maturity.py`, `topic_maturity.json`, `startup.sh`, `review-agent.md`). |
| `LICENSES/open-science-MIT.txt` | License of the upstream Open Science Desktop UI that this port follows. |
| `.env.example` | `VITE_RALPH_API_URL`. |

---

## Project status

**Context.** Built for the Ralphthon ICML hackathon (event kit: `github.com/team-attention/ralphthon-icml`), 11–12 July 2026, by danro, juneyoon and yc9954. Track 2 deliverables are the frozen [`review-agent.md`](sail-spec/assets/review-agent.md) and structured reviews rendered with it; Track 1 submission goes through `agent_review_submit.py`.

**Working today.** The full cycle loop in mock mode with IndexedDB persistence; the same loop against the adapter (fallback heads without a key, live heads with one); PDF and text submission; hunk-level revision; analysis view on real corpus data; the terminal review script, verified on a real paper per the commit log.

**Depends on infrastructure you may not have.** Live mode needs an Anthropic key and the VESSL-served meta-review/score models; the GCE deployment in `sail-spec/ops/80` and the GPU plan in `training/85` assume the team's GCP project, VESSL org and a private `vendor/ac-competition-kit` that is git-ignored. `sail-spec/SECRETS.local.md` is not in the repo. Metrics quoted in `review-agent.md` (e.g. score-head Spearman 0.872 on test_2023) are the team's own measurements; nothing here re-verifies them.

**Known gaps.** No automated tests. In live mode PDFs are returned as extracted text (`manuscript.kind: "text"`); inline PDF embed only works with a PDF URL (mock). `venue=workshop` is marked unvalidated after an A/B regression. The `.gstack/` folder contains local browse-session files and is not part of the product.

**Credits.** The UI is a pixel-faithful web port of the MIT-licensed [Open Science Desktop](https://github.com/ai4s-research/open-science) (design tokens, layout constants, interaction patterns); see `LICENSES/open-science-MIT.txt`.

---

## License

No LICENSE file is committed for this repository's own code, so default copyright applies: all rights reserved. The ported UI remains under the upstream [MIT license](LICENSES/open-science-MIT.txt).
