<h1 align="center">ICML SAIL with Ralph</h1>

<p align="center">
  <img src="https://img.shields.io/badge/React%2018%20%C2%B7%20TypeScript%206%20%C2%B7%20Vite%208-C15F3C?style=flat" alt="React 18, TypeScript 6, Vite 8" />
  <img src="https://img.shields.io/badge/Tailwind%203%20%C2%B7%20Recoil%20%C2%B7%20TanStack%20Query-C15F3C?style=flat" alt="Tailwind 3, Recoil, TanStack Query" />
  <img src="https://img.shields.io/badge/adapter-FastAPI%20%C2%B7%20Python%203.12-4493F8?style=flat" alt="FastAPI adapter on Python 3.12" />
  <img src="https://img.shields.io/badge/heads-Claude%20reviewers%20%2B%20VESSL%20Qwen3--8B%20LoRA-4493F8?style=flat" alt="Claude reviewer heads and a VESSL Qwen3-8B LoRA meta-review head" />
</p>

<p align="center">
  <sub><a href="../../README.md">English</a></sub>
</p>

<p align="center">
  <strong>실제 ICML 절차 그대로 돌아가는 AC(Area Chair)식 논문 리뷰 루프.</strong><br/>
  원고를 제출하면 점수 없이 리뷰어 3인의 리뷰가 먼저 오고, 리버탈 스레드에서 반론하고,<br/>
  AI가 초안한 수정 헝크를 허용·거부한 뒤, <em>finalize</em> 시점에만 AC 메타리뷰와 0–100 선택도 점수,<br/>
  accept/reject 결정을 받는다. resubmit하면 다음 사이클은 fresh context로 시작한다.
</p>

<h3 align="center"><a href="#시작하기"><ins>시작하기</ins></a> · <a href="../../sail-spec/00-INDEX.md">재구현 스펙</a> · <a href="../../sail-spec/assets/review-agent.md">동결된 리뷰 에이전트</a></h3>

## 기능

### 리뷰가 먼저, 점수는 마지막

한 사이클은 실제 학회를 그대로 따른다. `submit`은 리뷰어 3인의 리뷰(ICML식 1–10 평점, 요약, 라이브 모드에서는 전체 본문과 confidence·soundness·presentation·contribution)를 만든다. 이 시점에 점수는 존재하지 않는다. 점수, `gradeTier`, 피처 기여도, 부족점(deficiency) 리포트는 `finalize` 이후에만 AC 메타리뷰·결정문과 함께 나타난다. `SELECT_THRESHOLD`는 88이다.

### 채팅 로그가 아닌 리버탈 스레드

코멘트는 구조화된 이슈다(`major` / `minor` / `question`, 섹션 라벨, 제기한 리뷰어). 코멘트에 답글을 달면 그 리뷰어에게 전달되고 리뷰어가 후속 답변을 한다. 수정 헝크의 allow/deny 결정은 모두 리버탈 텍스트로 스레드에 기록되어 AC가 토론 전체를 본다.

### 헝크 단위 AI 수정

`revision-draft`는 에이전트에게 before/after 헝크를 요청하며, 각 헝크에는 근거와 대응하는 코멘트 id가 붙는다. 헝크별로 허용·거부하면 `revision-apply`가 허용된 헝크만 반영해 원고를 다시 쓰고, 수정본은 다음 메시지에 첨부된다. 원고 패널은 바뀐 부분을 하이라이트한다. 원고 직접 편집도 지원한다.

### 실제 코퍼스에 기반한 Analysis 뷰

`/review/:id/analysis`는 점수 병목(백본 활성 → 점수 → 3-헤드)을 그리고, **실제 ICLR/ICML/NeurIPS/UAI 제출 47,209편(2018–2026)**의 히스토그램 위에 내 논문의 사이클별 점수를 올린다. 실측 티어 중앙값: reject 25 / poster 43 / spotlight 59 / oral 71.8 / notable-top-5% 86.7. 기여도 행에 호버하면 그 근거가 된 원고 문장이 하이라이트된다.

### 백엔드가 있어도, 없어도

`VITE_RALPH_API_URL`을 비워두면 내장 mock이 전체 루프를 결정적으로 실행하고(사이클 점수 63 → 79 → 91 → 96), step 이벤트를 스트리밍하며, 업로드한 PDF Blob까지 포함해 모든 논문을 IndexedDB(`sail-ralph`)에 저장한다. 설정하면 모든 호출이 `serve/sail_adapter.py`에 1:1로 매핑된다.

**함께 포함된 것**

- **`serve/sail_adapter.py`**: 단일 파일 FastAPI 백엔드(계약 v2). `ANTHROPIC_API_KEY`가 있으면 Claude 리뷰어 에이전트 3인 병렬, 리뷰어 답변 에이전트, VESSL 서빙 Qwen3-8B LoRA 메타리뷰·점수 헤드, Claude 기여도 헤드, Claude 수정 에이전트를 실행하고, 없으면 각 헤드가 결정적 폴백으로 강등된다. 학습된 점수 헤드로 가는 얇은 프록시 `POST /api/score`도 노출한다.
- **`sail-spec/`**: 자기완결적 재구현 명세. 프론트 모듈별 verbatim 소스, 백엔드 계약, GCE·VESSL 런북, GPU 학습 플랜, 라이브 실측 골든 트랜스크립트, 해커톤 Track 2 프로토콜.
- **터미널 리뷰 스킬**: `sail-spec/assets/review_paper.py`는 논문 한 편을 웹과 동일한 백엔드 경로로 돌려 공식 Track 2 리뷰 템플릿으로 렌더한다. `agent_review_submit.py`는 openagentreview.org 리뷰 창을 위한 Track 1 제출 하네스다.
- **paper-tone 라이트/다크 테마**, `⌘B` 사이드바 접기, 상태 점이 붙은 제출 히스토리 사이드바.

---

## 동작 방식

```text
Browser (Vite + React 18, Recoil UI state, TanStack Query server state)
   │  src/api/reviewLoop.ts — 단일 타입 계약
   ├── VITE_RALPH_API_URL 미설정 ──▶ 같은 파일의 mock 루프 ──▶ IndexedDB (papers + PDF blobs)
   └── VITE_RALPH_API_URL 설정 ────▶ serve/sail_adapter.py (FastAPI, 로컬 :8100)
                                       ├─ review head ........ Claude 리뷰어 3인 병렬
                                       ├─ reply head ......... Claude, 해당 리뷰어의 리뷰 + 스레드에 근거
                                       ├─ meta-review+score .. VESSL Qwen3-8B LoRA  (/meta-review, /score)
                                       ├─ attribution head ... Claude, 원문 verbatim 근거 문장
                                       ├─ revision agent ..... Claude, 근거 있는 헝크만
                                       └─ state ............. SAIL_STATE_PATH (sail_state.json)
```

1. **제출** (`POST /api/loop/papers`, multipart `title` + `text` 또는 PDF `file`). PDF는 서버에서 텍스트 추출(PyMuPDF). 사이클 1이 리뷰 3건으로 열리고, UI는 `/api/loop/jobs/{id}`로 에이전트 step 이벤트를 스트리밍한다.
2. **토론.** `reply`는 저자 메시지(특정 코멘트 대상 가능)를 올리고 리뷰어 후속 답변을 받는다. `revision-draft` / `revision-apply`가 헝크를 만들고 반영하며, `manuscript`는 직접 편집을 받는다.
3. **Finalize.** AC 메타리뷰는 리뷰와 토론 전체로부터 작성된다. 점수 = 캘리브레이션된 `p_accept`(p^0.25, 노출된 평점에 앵커) 또는 `SAIL_SCORE_HEAD=1`이면 학습된 회귀 헤드. 결정, 티어, 기여도, 부족점 리포트("왜 여기서 멈췄고 다음 밴드로 가는 길")가 함께 돌아온다.
4. **Resubmit**은 수정된 원고로 사이클 N+1을 새 리뷰·빈 스레드로 시작한다. 상태는 `in_discussion`과 `decided` 사이를 오간다.

<details>
<summary><strong>어댑터 환경변수</strong></summary>

| 변수 | 기본값 | 역할 |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | 미설정 | 라이브 헤드 활성화(`LIVE=True`). 없으면 모든 헤드가 결정적 폴백. |
| `SAIL_CLAUDE_MODEL` | `claude-opus-4-8` | 리뷰어·답변·기여도·수정 에이전트 모델. |
| `VESSL_META_URL`, `VESSL_META_MODEL` | VESSL 엔드포인트, `v2` | 메타리뷰·점수 서빙. |
| `SAIL_SCORE_HEAD` | `1` | `0`이면 원래 `p_accept` 캘리브레이션 경로로 복귀. |
| `SAIL_VENUE` | `icml` | 리뷰어 심사 기준: `icml`(메인 학회) 또는 `workshop`(4쪽 행사, 미검증 표시). |
| `SAIL_FIELD_CONTEXT`, `TOPIC_MATURITY_PATH` | `1`, `/opt/sail/topic_maturity.json` | `build_topic_maturity.py`로 만든 리센시 보정. |
| `SAIL_STATE_PATH` | `sail_state.json` | 루프 상태 영속화. |
| `WEB_DIST` | `/opt/sail/web` | 배포 시 같은 프로세스가 서빙하는 빌드된 SPA. |

</details>

---

## 기술 스택

<p>
  <kbd>React&nbsp;18</kbd> &nbsp; <kbd>TypeScript&nbsp;6</kbd> &nbsp; <kbd>Vite&nbsp;8</kbd> &nbsp; <kbd>Tailwind&nbsp;3</kbd> &nbsp; <kbd>Recoil</kbd> &nbsp; <kbd>TanStack&nbsp;Query&nbsp;5</kbd> &nbsp; <kbd>react-router&nbsp;7</kbd> &nbsp; <kbd>Radix&nbsp;UI</kbd> &nbsp; <kbd>react-markdown</kbd> &nbsp; <kbd>IndexedDB</kbd> &nbsp; <kbd>oxlint</kbd> &nbsp;
  <kbd>FastAPI</kbd> &nbsp; <kbd>Python&nbsp;3.12</kbd> &nbsp; <kbd>PyMuPDF</kbd> &nbsp; <kbd>Anthropic&nbsp;SDK</kbd> &nbsp; <kbd>VESSL&nbsp;serving</kbd> &nbsp; <kbd>GCE&nbsp;+&nbsp;GCS</kbd>
</p>

---

## 시작하기

**사전 준비**

- 웹앱: Node.js 20+ 와 npm.
- 어댑터: Python 3.10+ (Dockerfile은 3.12). 라이브 모드는 Anthropic API 키와 접근 가능한 VESSL 엔드포인트가 필요.

```bash
git clone https://github.com/yc9954/Ralphthon-ICML-SAIL.git
cd Ralphthon-ICML-SAIL
npm install
npm run dev                 # http://localhost:5199 — mock 모드, 백엔드 불필요
```

mock 대신 어댑터를 연결하려면:

```bash
# 백엔드
cd serve && pip install -r requirements.txt
ANTHROPIC_API_KEY=... uvicorn sail_adapter:app --port 8100     # 키를 빼면 폴백 헤드

# 프론트
echo 'VITE_RALPH_API_URL=http://localhost:8100' > .env.local   # 8000은 로컬 Sophy와 충돌
npm run dev
```

| 프로세스 | 포트 | 비고 |
| --- | --- | --- |
| Vite dev 서버 | `5199` | `vite.config.ts`에 지정 |
| `sail_adapter.py` (uvicorn) | 로컬 `8100`, Docker 이미지 `8080` | 컨테이너는 `PORT` env |

**논문 한 편 터미널 리뷰** (웹과 같은 파이프라인, 공식 Track 2 템플릿):

```bash
python3 sail-spec/assets/review_paper.py papers/foo.md --base http://localhost:8100 [--selfreview papers/foo.selfreview.md] [--keep]
```

---

## 빌드와 테스트

```bash
npm run build     # tsc -b && vite build → dist/
npm run lint      # oxlint
npm run preview
```

자동 테스트는 없다. 수용 게이트는 수동 체크리스트다: `sail-spec/golden/flows.md` §A(mock 플로우 8항목)와 §B(어댑터 폴백 e2e). `sail-spec/golden/`의 JSON은 라이브 실측 트랜스크립트로, 값이 아니라 필드·불변식이 대조 기준이다.

---

## 화면

| 경로 | 역할 |
| --- | --- |
| `/review` | 제목 + 텍스트 붙여넣기 또는 PDF 제출, 이전 제출 목록. |
| `/review/:paperId` | 루프 본체: 리뷰, 코멘트별 답글이 있는 리버탈 스레드, 수정 헝크, finalize·resubmit. 우측에 원고 패널(세리프 텍스트 렌더 또는 PDF embed, 드래그 리사이즈, 수정부 하이라이트). |
| `/review/:paperId/analysis` | 병목 다이어그램과 실코퍼스 분포 위의 사이클별 궤적. |
| `/settings` | 개인정보·데이터 흐름 카드, 원격 컴퓨트·Modal 카드(데스크톱 앱 플레이스홀더). |

---

## 저장소 구조

| 경로 | 내용 |
| --- | --- |
| `src/api/reviewLoop.ts` | 도메인 계약(타입, `SELECT_THRESHOLD`, `loopApi`)과 결정적 mock. |
| `src/api/reviewLoopQueries.ts`, `loopStorage.ts` | TanStack Query 훅, mock 모드용 IndexedDB 영속화. |
| `src/app/` | `router.tsx`, `layout/AppShell.tsx`, `providers/ThemeProvider.tsx`, `routes/` (`ReviewLoopPage`, `AnalysisPage`, `SettingsPage`, `NotFound`). |
| `src/components/` | `review/ManuscriptPane`, `analysis/` (`BottleneckDiagram`, `CorpusDistribution`, `ReviewTabs`), `sidebar/`, `settings/`, `ui/`. |
| `src/data/corpusDistribution.ts` | 47,209편 히스토그램(50 bin)과 티어 중앙값. |
| `serve/` | `sail_adapter.py`, `requirements.txt`, `Dockerfile`. |
| `sail-spec/` | `00-INDEX.md`(지도와 공통 원칙 10개), `CLAUDE.md`(세션 진입점), `STATE.md`(살아있는 체크리스트), `GOAL.md`(미션 프롬프트), 번호별 유닛(토대, 디자인 토큰, 계약, API, 데이터, 상태, UI, 와이어링, 백엔드, ops(GCE·VESSL), training(GPU 플랜), 하네스, 해커톤 Track 2), `golden/` 트랜스크립트, `assets/` (`review_paper.py`, `agent_review_submit.py`, `build_topic_maturity.py`, `topic_maturity.json`, `startup.sh`, `review-agent.md`). |
| `LICENSES/open-science-MIT.txt` | 이 포트가 따르는 업스트림 Open Science Desktop UI의 라이선스. |
| `.env.example` | `VITE_RALPH_API_URL`. |

---

## 프로젝트 상태

**배경.** Ralphthon ICML 해커톤(이벤트 킷: `github.com/team-attention/ralphthon-icml`)을 위해 2026년 7월 11–12일에 danro, juneyoon, yc9954가 만들었다. Track 2 딜리버러블은 동결된 [`review-agent.md`](../../sail-spec/assets/review-agent.md)와 그것으로 렌더한 구조화 리뷰이며, Track 1 제출은 `agent_review_submit.py`를 거친다.

**지금 동작하는 것.** IndexedDB 영속화가 있는 mock 모드의 전체 사이클 루프, 어댑터를 상대로 한 같은 루프(키 없으면 폴백 헤드, 있으면 라이브 헤드), PDF·텍스트 제출, 헝크 단위 수정, 실코퍼스 데이터 위의 Analysis 뷰, 커밋 로그상 실제 논문으로 검증된 터미널 리뷰 스크립트.

**없을 수 있는 인프라에 의존.** 라이브 모드는 Anthropic 키와 VESSL 서빙 메타리뷰·점수 모델이 필요하다. `sail-spec/ops/80`의 GCE 배포와 `training/85`의 GPU 플랜은 팀의 GCP 프로젝트, VESSL org, gitignore된 비공개 `vendor/ac-competition-kit`을 전제한다. `sail-spec/SECRETS.local.md`는 저장소에 없다. `review-agent.md`에 인용된 지표(예: 점수 헤드 test_2023 Spearman 0.872)는 팀 자체 측정값이며 여기서 재검증하지 않는다.

**알려진 공백.** 자동 테스트 없음. 라이브 모드에서 PDF는 추출 텍스트(`manuscript.kind: "text"`)로 돌아오고, 인라인 PDF embed는 PDF URL이 있을 때(mock)만 동작한다. `venue=workshop`은 A/B 회귀 이후 미검증으로 표시되어 있다. `.gstack/` 폴더는 로컬 브라우즈 세션 파일이며 제품의 일부가 아니다.

**크레딧.** UI는 MIT 라이선스 [Open Science Desktop](https://github.com/ai4s-research/open-science)(디자인 토큰, 레이아웃 상수, 인터랙션 패턴)의 픽셀 단위 웹 포트다. `LICENSES/open-science-MIT.txt` 참조.

---

## 라이선스

이 저장소 자체 코드에는 LICENSE 파일이 커밋되어 있지 않으므로 기본 저작권이 적용된다(all rights reserved). 포팅된 UI는 업스트림 [MIT 라이선스](../../LICENSES/open-science-MIT.txt)를 따른다.
