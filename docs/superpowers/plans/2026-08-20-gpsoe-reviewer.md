# GPSOE Exam Reviewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` exam reviewer for the Google Professional Security Operations Engineer cert that presents lesson content before per-domain quizzes, deploys to GitHub Pages.

**Architecture:** One self-contained `index.html` with embedded CSS and JS — no build step, no external dependencies. All 131 questions categorized into 5 official GPSOE exam domains. Each domain has a Study tab (lesson content derived from Q&A) and a Practice tab (quiz). A Full Mock Exam mode runs all 131 questions.

**Tech Stack:** Vanilla HTML/CSS/JS. No frameworks, no bundlers. Dark theme reused from the source file.

**Spec:** Design approved in chat on 2026-08-20.

## Global Constraints

- All 131 questions must be preserved — no removals.
- Edit a question/answer/explanation only if it is factually wrong; mark any edit with `// EDITED:` comment.
- Single file: `index.html` at repo root.
- No external JS/CSS dependencies (inline everything).
- GitHub Pages target: `github.com/xiann-neko/secops-reviewer`, branch `main`, root `/`.
- Dark theme: bg `#0f1117`, surface `#1a1d27`, accent `#4f7cff`, green `#22c55e`, red `#f87171`, yellow `#fbbf24`.

---

## Domain → Question Mapping

Questions are 0-indexed matching their order in `ALL_QUESTIONS` in the source file (`gpsoe-practice-exam.html`, lines 130–260).

| Domain | Indices |
|--------|---------|
| 1 – Configuring the Environment | 7,11,18,25,26,40,44,52,55,57,59,65,74,78,79,81,82,83,90,100,101,102,106,110,113,115,124,128,130 |
| 2 – Managing & Configuring Security Telemetry | 3,12,16,62,71,89,99,104,118 |
| 3 – Ensuring Ongoing & Effective Detection | 0,2,4,8,13,15,17,19,20,22,24,33,34,35,38,39,41,50,51,60,64,66,70,80,91,92,93,94,95,96,105,112,122,123,127 |
| 4 – Investigating Security Incidents | 6,9,21,23,27,30,31,32,37,43,46,49,58,63,68,76,85,88,97,108,111,116,119,120,125 |
| 5 – Responding to Security Incidents | 1,5,10,14,28,29,36,42,45,47,48,53,54,56,61,67,69,72,73,75,77,84,86,87,98,103,107,109,114,117,121,126,129 |

---

## File Structure

```
secops-reviewer/
  index.html          ← entire app (HTML + embedded CSS + JS)
  README.md           ← description + GitHub Pages link
  docs/superpowers/plans/2026-08-20-gpsoe-reviewer.md
```

`index.html` internal structure:
```
<style> — dark theme CSS (all screens)
<body>
  #app — single mount point
<script>
  DOMAINS[]     — 5 domain metadata objects
  LESSONS{}     — lesson content keyed by domain id
  ALL_QUESTIONS[] — all 131 questions with .domain property added
  state{}       — current screen, domain, quiz progress
  render()      — top-level dispatcher
  renderHome()
  renderDomain(id)
  renderStudy(id)
  renderQuiz(questions, opts)
  renderResults()
  helpers: shuffle(), formatTime(), el()
```

---

## Task 1: HTML Shell + CSS

**Files:**
- Create: `index.html`

**Deliverable:** Opening `index.html` in a browser shows a dark background with no errors in the console.

- [ ] **Step 1: Create `index.html` with the shell**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>GPSOE Exam Reviewer</title>
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
:root {
  --bg: #0f1117; --surface: #1a1d27; --card: #21253a;
  --border: #2e3350; --border-light: #3d4370;
  --accent: #4f7cff; --accent-hover: #6b93ff;
  --green: #22c55e; --green-bg: #0d2818; --green-border: #166534;
  --red: #f87171; --red-bg: #2d0f0f; --red-border: #7f1d1d;
  --yellow: #fbbf24;
  --text: #e8eaf6; --text-muted: #8b92b8; --text-dim: #5a6080;
  --radius: 10px; --radius-sm: 6px;
}
body { background: var(--bg); color: var(--text); font-family: 'Segoe UI', system-ui, sans-serif; font-size: 15px; line-height: 1.6; min-height: 100vh; }

/* ── Shared ── */
.btn { padding: 10px 20px; border-radius: var(--radius-sm); font-size: 14px; font-weight: 600; cursor: pointer; border: none; transition: opacity .15s, transform .1s; font-family: inherit; }
.btn:active { transform: scale(.97); }
.btn-primary { background: var(--accent); color: #fff; }
.btn-primary:hover { background: var(--accent-hover); }
.btn-green  { background: var(--green); color: #fff; }
.btn-green:hover { opacity: .9; }
.btn-ghost  { background: var(--card); border: 1.5px solid var(--border); color: var(--text-muted); }
.btn-ghost:hover { border-color: var(--border-light); color: var(--text); }
.badge { background: var(--accent); color: #fff; font-size: 11px; font-weight: 700; letter-spacing: .06em; padding: 3px 8px; border-radius: 4px; text-transform: uppercase; }
.screen { display: none; }
.screen.active { display: block; }

/* ── Header ── */
.header { background: var(--surface); border-bottom: 1px solid var(--border); padding: 0 24px; display: flex; align-items: center; justify-content: space-between; height: 56px; position: sticky; top: 0; z-index: 100; }
.header-left { display: flex; align-items: center; gap: 12px; }
.header h1 { font-size: 15px; font-weight: 600; }
.header-back { background: none; border: none; color: var(--text-muted); cursor: pointer; font-size: 14px; font-family: inherit; padding: 6px 12px; border-radius: var(--radius-sm); }
.header-back:hover { background: var(--card); color: var(--text); }

/* ── Home screen ── */
.home-body { max-width: 900px; margin: 0 auto; padding: 40px 24px; }
.home-hero { text-align: center; margin-bottom: 48px; }
.home-hero h2 { font-size: 28px; font-weight: 700; margin-bottom: 8px; }
.home-hero p  { color: var(--text-muted); font-size: 15px; max-width: 560px; margin: 0 auto 24px; }
.domain-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 16px; margin-bottom: 32px; }
.domain-card { background: var(--card); border: 1.5px solid var(--border); border-radius: var(--radius); padding: 20px; display: flex; flex-direction: column; gap: 12px; transition: border-color .15s; }
.domain-card:hover { border-color: var(--border-light); }
.domain-card .domain-num { font-size: 11px; font-weight: 700; color: var(--accent); text-transform: uppercase; letter-spacing: .08em; }
.domain-card h3 { font-size: 15px; font-weight: 600; line-height: 1.4; }
.domain-card .domain-meta { font-size: 12px; color: var(--text-muted); }
.domain-card .card-actions { display: flex; gap: 8px; margin-top: 4px; }
.card-actions .btn { font-size: 13px; padding: 8px 14px; }
.mock-banner { background: var(--surface); border: 1.5px solid var(--border); border-radius: var(--radius); padding: 20px 24px; display: flex; align-items: center; justify-content: space-between; gap: 16px; }
.mock-banner div h3 { font-size: 16px; font-weight: 700; margin-bottom: 4px; }
.mock-banner div p  { font-size: 13px; color: var(--text-muted); }

/* ── Domain screen (Study/Practice tabs) ── */
.domain-body { max-width: 820px; margin: 0 auto; padding: 32px 24px; }
.tabs { display: flex; gap: 4px; background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius-sm); padding: 4px; width: fit-content; margin-bottom: 28px; }
.tab-btn { padding: 8px 20px; border-radius: 4px; font-size: 14px; font-weight: 500; border: none; background: none; color: var(--text-muted); cursor: pointer; font-family: inherit; transition: background .15s, color .15s; }
.tab-btn.active { background: var(--card); color: var(--text); font-weight: 600; }

/* ── Study / Lesson view ── */
.lesson-section { margin-bottom: 32px; }
.lesson-section h3 { font-size: 16px; font-weight: 700; margin-bottom: 14px; padding-bottom: 8px; border-bottom: 1px solid var(--border); color: var(--accent); }
.concept-list { list-style: none; display: flex; flex-direction: column; gap: 10px; }
.concept-list li { background: var(--card); border: 1px solid var(--border); border-radius: var(--radius-sm); padding: 12px 14px; font-size: 13.5px; line-height: 1.6; }
.concept-list li strong { color: var(--text); }
.pattern-list { list-style: none; display: flex; flex-direction: column; gap: 8px; }
.pattern-list li { background: var(--surface); border-left: 3px solid var(--accent); border-radius: 0 var(--radius-sm) var(--radius-sm) 0; padding: 10px 14px; font-size: 13.5px; line-height: 1.6; }
.pattern-list li .trigger { color: var(--text-muted); }
.pattern-list li .answer  { color: var(--green); font-weight: 600; }
.ref-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 10px; }
.ref-item { background: var(--card); border: 1px solid var(--border); border-radius: var(--radius-sm); padding: 10px 12px; font-size: 12.5px; font-family: 'Consolas', monospace; color: var(--yellow); line-height: 1.7; }
.lesson-start-btn { margin-top: 8px; }

/* ── Quiz engine ── */
.quiz-layout  { display: flex; height: calc(100vh - 56px); overflow: hidden; }
.quiz-sidebar { width: 240px; flex-shrink: 0; background: var(--surface); border-right: 1px solid var(--border); display: flex; flex-direction: column; overflow: hidden; }
.sidebar-header { padding: 14px; border-bottom: 1px solid var(--border); }
.progress-label { display: flex; justify-content: space-between; font-size: 12px; color: var(--text-muted); margin-bottom: 6px; }
.progress-bar  { height: 4px; background: var(--border); border-radius: 2px; overflow: hidden; }
.progress-fill { height: 100%; background: var(--accent); border-radius: 2px; transition: width .3s ease; }
.stats-row { display: flex; gap: 6px; margin-top: 10px; }
.stat { flex: 1; background: var(--card); border: 1px solid var(--border); border-radius: var(--radius-sm); padding: 6px; text-align: center; }
.stat .num { font-size: 16px; font-weight: 700; }
.stat .lbl { font-size: 9px; color: var(--text-muted); text-transform: uppercase; letter-spacing: .05em; }
.stat.correct .num { color: var(--green); }
.stat.wrong   .num { color: var(--red); }
.stat.remain  .num { color: var(--accent); }
.q-grid  { flex: 1; overflow-y: auto; padding: 10px; display: grid; grid-template-columns: repeat(5, 1fr); gap: 4px; align-content: start; }
.q-dot   { aspect-ratio: 1; border-radius: 4px; background: var(--card); border: 1px solid var(--border); display: flex; align-items: center; justify-content: center; font-size: 9px; font-weight: 600; color: var(--text-dim); cursor: pointer; transition: all .15s; user-select: none; }
.q-dot:hover   { border-color: var(--accent); color: var(--accent); }
.q-dot.active  { border-color: var(--accent); background: rgba(79,124,255,.15); color: var(--accent); }
.q-dot.correct { background: var(--green-bg); border-color: var(--green-border); color: var(--green); }
.q-dot.wrong   { background: var(--red-bg); border-color: var(--red-border); color: var(--red); }
.quiz-main { flex: 1; overflow-y: auto; padding: 32px 40px; display: flex; flex-direction: column; gap: 20px; }
.q-meta  { display: flex; align-items: center; gap: 10px; font-size: 12px; color: var(--text-muted); }
.q-num   { background: var(--accent); color: #fff; font-size: 11px; font-weight: 700; padding: 2px 8px; border-radius: 4px; }
.multi-tag { background: rgba(251,191,36,.15); color: var(--yellow); border: 1px solid rgba(251,191,36,.3); font-size: 11px; font-weight: 600; padding: 2px 8px; border-radius: 4px; }
.q-text  { font-size: 16px; font-weight: 500; line-height: 1.65; max-width: 700px; }
.options { display: flex; flex-direction: column; gap: 10px; max-width: 700px; }
.opt     { background: var(--card); border: 1.5px solid var(--border); border-radius: var(--radius); padding: 13px 16px; cursor: pointer; display: flex; align-items: flex-start; gap: 12px; transition: border-color .15s, background .15s; text-align: left; width: 100%; color: var(--text); font-size: 14px; line-height: 1.55; font-family: inherit; }
.opt:hover:not(:disabled) { border-color: var(--accent); background: rgba(79,124,255,.06); }
.opt:disabled { cursor: default; }
.opt-letter { width: 24px; height: 24px; border-radius: 50%; background: var(--border); color: var(--text-muted); font-size: 11px; font-weight: 700; display: flex; align-items: center; justify-content: center; flex-shrink: 0; margin-top: 1px; }
.opt.selected .opt-letter { background: var(--accent); color: #fff; }
.opt.selected { border-color: var(--accent); background: rgba(79,124,255,.1); }
.opt.correct-ans { border-color: var(--green) !important; background: var(--green-bg) !important; }
.opt.correct-ans .opt-letter { background: var(--green); color: #fff; }
.opt.wrong-ans   { border-color: var(--red) !important; background: var(--red-bg) !important; }
.opt.wrong-ans   .opt-letter { background: var(--red); color: #fff; }
.feedback-box { max-width: 700px; border-radius: var(--radius); padding: 14px 16px; font-size: 13.5px; line-height: 1.6; }
.feedback-box.correct { background: var(--green-bg); border: 1px solid var(--green-border); color: #86efac; }
.feedback-box.wrong   { background: var(--red-bg);   border: 1px solid var(--red-border);   color: #fca5a5; }
.quiz-actions { display: flex; gap: 10px; max-width: 700px; }

/* ── Results screen ── */
.results-body { max-width: 640px; margin: 0 auto; padding: 48px 24px; text-align: center; }
.results-body h2 { font-size: 26px; font-weight: 700; margin-bottom: 6px; }
.results-body .sub { color: var(--text-muted); margin-bottom: 32px; }
.score-circle { width: 150px; height: 150px; border-radius: 50%; border: 6px solid var(--border); display: flex; flex-direction: column; align-items: center; justify-content: center; margin: 0 auto 28px; }
.score-circle.pass { border-color: var(--green); box-shadow: 0 0 40px rgba(34,197,94,.2); }
.score-circle.fail { border-color: var(--red);   box-shadow: 0 0 40px rgba(248,113,113,.15); }
.score-pct  { font-size: 38px; font-weight: 800; }
.score-circle.pass .score-pct { color: var(--green); }
.score-circle.fail .score-pct { color: var(--red); }
.score-lbl  { font-size: 11px; color: var(--text-muted); text-transform: uppercase; letter-spacing: .08em; }
.verdict    { font-size: 17px; font-weight: 700; margin-bottom: 28px; }
.verdict.pass { color: var(--green); }
.verdict.fail { color: var(--red); }
.results-grid { display: grid; grid-template-columns: repeat(3,1fr); gap: 12px; margin-bottom: 28px; }
.res-card { background: var(--card); border: 1px solid var(--border); border-radius: var(--radius); padding: 16px; }
.res-card .big { font-size: 26px; font-weight: 700; }
.res-card .sml { font-size: 11px; color: var(--text-muted); text-transform: uppercase; letter-spacing: .05em; margin-top: 3px; }
.res-card.c .big { color: var(--green); }
.res-card.w .big { color: var(--red); }
.results-actions { display: flex; gap: 10px; justify-content: center; }
</style>
</head>
<body>
<div id="app"></div>
<script>
// DATA, STATE, and RENDERERS go here (Tasks 2–7)
</script>
</body>
</html>
```

- [ ] **Step 2: Open `index.html` in browser, verify**
  - Dark background renders
  - No console errors
  - Page is blank (expected — no render call yet)

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HTML shell and full CSS"
```

---

## Task 2: QUESTIONS Data

**Files:**
- Modify: `index.html` — replace the `// DATA...` comment in `<script>` with the QUESTIONS array and domain mapping.

**Deliverable:** `console.log(ALL_QUESTIONS.length)` in browser console returns `131`. Each question has a `.domain` property (1–5).

- [ ] **Step 1: Read source questions from the download**

Open `C:\Users\ross.anne.ilagan\Downloads\gpsoe-practice-exam.html` and copy the entire `ALL_QUESTIONS` array (lines 129–261).

- [ ] **Step 2: Paste into `index.html` and add `.domain` property to each question**

Insert after `<script>` opening tag. Add a `domain` field to every question object using the mapping table at the top of this plan. Example:

```js
// Domain index map (0-based question index → domain 1-5)
const DOMAIN_MAP = {
  // Domain 1 – Configuring the Environment
  7:1, 11:1, 18:1, 25:1, 26:1, 40:1, 44:1, 52:1, 55:1, 57:1,
  59:1, 65:1, 74:1, 78:1, 79:1, 81:1, 82:1, 83:1, 90:1,
  100:1, 101:1, 102:1, 106:1, 110:1, 113:1, 115:1, 124:1, 128:1, 130:1,
  // Domain 2 – Managing & Configuring Security Telemetry
  3:2, 12:2, 16:2, 62:2, 71:2, 89:2, 99:2, 104:2, 118:2,
  // Domain 3 – Ensuring Ongoing & Effective Detection
  0:3, 2:3, 4:3, 8:3, 13:3, 15:3, 17:3, 19:3, 20:3, 22:3, 24:3,
  33:3, 34:3, 35:3, 38:3, 39:3, 41:3, 50:3, 51:3, 60:3, 64:3,
  66:3, 70:3, 80:3, 91:3, 92:3, 93:3, 94:3, 95:3, 96:3,
  105:3, 112:3, 122:3, 123:3, 127:3,
  // Domain 4 – Investigating Security Incidents
  6:4, 9:4, 21:4, 23:4, 27:4, 30:4, 31:4, 32:4, 37:4, 43:4,
  46:4, 49:4, 58:4, 63:4, 68:4, 76:4, 85:4, 88:4, 97:4,
  108:4, 111:4, 116:4, 119:4, 120:4, 125:4,
  // Domain 5 – Responding to Security Incidents
  1:5, 5:5, 10:5, 14:5, 28:5, 29:5, 36:5, 42:5, 45:5, 47:5, 48:5,
  53:5, 54:5, 56:5, 61:5, 67:5, 69:5, 72:5, 73:5, 75:5, 77:5,
  84:5, 86:5, 87:5, 98:5, 103:5, 107:5, 109:5, 114:5, 117:5,
  121:5, 126:5, 129:5
};

const ALL_QUESTIONS = [
  /* paste the 131 question objects here, then tag each with domain */
];

// Tag each question with its domain
ALL_QUESTIONS.forEach((q, i) => { q.domain = DOMAIN_MAP[i] || 3; });
```

> Note: Any question index not in DOMAIN_MAP falls back to domain 3 (detection) as a safe default. Verify the total after pasting: `ALL_QUESTIONS.filter(q=>q.domain===1).length` etc.

- [ ] **Step 3: Verify in browser console**

```js
console.log(ALL_QUESTIONS.length); // must be 131
[1,2,3,4,5].forEach(d => console.log(`D${d}:`, ALL_QUESTIONS.filter(q=>q.domain===d).length));
// Expected totals: D1:29, D2:9, D3:35, D4:25, D5:33
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add all 131 questions with domain assignments"
```

---

## Task 3: DOMAINS + LESSONS Data

**Files:**
- Modify: `index.html` — add `DOMAINS` array and `LESSONS` object after `ALL_QUESTIONS`.

**Deliverable:** `console.log(DOMAINS.length)` → 5. Each domain has `id`, `name`, `pct`, `desc`. `LESSONS[1].concepts` is a non-empty array.

- [ ] **Step 1: Add DOMAINS array**

```js
const DOMAINS = [
  { id:1, name:'Configuring the Environment',              pct:14, desc:'Log ingestion, parsers, collection agents, IAM, ingestion health monitoring.' },
  { id:2, name:'Managing & Configuring Security Telemetry',pct:22, desc:'UDM search, dashboards, data access scopes, entity enrichment, reference lists.' },
  { id:3, name:'Ensuring Ongoing & Effective Detection',   pct:22, desc:'YARA-L rules, curated detections, Applied Threat Intelligence, rule tuning.' },
  { id:4, name:'Investigating Security Incidents',         pct:26, desc:'Threat hunting, IOC analysis, GTI, Risk Analytics, Entity Explorer, SCC findings.' },
  { id:5, name:'Responding to Security Incidents',        pct:16, desc:'SOAR playbooks, case management, automated response, SCC response, remediation.' },
];
```

- [ ] **Step 2: Add LESSONS object**

Each domain entry has three arrays: `concepts` (key concept items), `patterns` (exam-tip trigger→answer pairs), `refs` (quick reference strings).

```js
const LESSONS = {
  1: {
    concepts: [
      { term:'Log Ingestion Methods', detail:'Direct ingestion (Google Cloud → SecOps native), Google SecOps Forwarder (on-premises structured logs), Bindplane agent (Syslog + structured logs, third-party sources), Ops Agent → Cloud Logging → direct ingestion (built into Google-managed GCE images), Cloud Run + Ingestion API (cross-project Pub/Sub workaround), Feed management (pull-based from APIs/buckets).' },
      { term:'Parsers & Extensions', detail:'Default parsers: Google-maintained, cannot be edited directly. Parser extensions: additive field mappings ON TOP of a default parser — fastest, least change management. Custom parsers: full replacement — use only when default is absent. Extract Additional Fields tool: fast no-code way to surface new fields from raw logs without a parser change.' },
      { term:'Collection Agents', detail:'Bindplane: for Syslog and third-party on-prem sources. Google SecOps Forwarder: for on-premises structured log sources (e.g., MySQL). Ops Agent: embedded in Google-managed GCE images — no install needed, feeds Cloud Logging. Remote Agent: for SOAR integrations in private networks that block inbound connections (agent initiates outbound).' },
      { term:'IAM & Access Control', detail:'roles/chronicle.viewer: full read-only including detection rules. roles/chronicle.limitedViewer: read-only EXCLUDING detection rules. roles/chronicle.editor: read-write. Data access scopes: SIEM-level data visibility filter. Workforce identity federation: federate with external IdP — MSP/partner manages their own user lifecycle. allowedPolicyMemberDomains org policy: restricts IAM bindings to specific domains — blocks external identities.' },
      { term:'Ingestion Health Monitoring', detail:'Cloud Monitoring metric-absence condition: fires when metrics stop arriving within a specified time window (e.g., 5 min or 30 min). This is the native purpose-built approach for silent source alerting. Bindplane has its own Observability Pipeline for agent-side metrics.' },
      { term:'Log Sink Configuration', detail:'Aggregated log sinks collect logs from all projects in a folder or org without project-by-project config. Folder-level sink + Pub/Sub connector = low-latency delivery with minimal setup for folder-scoped logs (e.g., Cloud NAT). Ingestion labels: tag individual log sources to avoid IP aliasing across multiple NAS/collectors. Namespaces: logical data separation (different purpose from ingestion labels).' },
    ],
    patterns: [
      { trigger:'Fastest/least effort to add missing UDM fields to an existing parser', answer:'Parser extension (no-code or code snippet)', reason:'Additive — no change management, no parser replacement.' },
      { trigger:'On-premises Syslog source → SecOps', answer:'Deploy Bindplane agent, set as Syslog destination', reason:'Bindplane is the standard third-party on-prem Syslog collector.' },
      { trigger:'On-premises MySQL → SecOps, minimize effort', answer:'Google SecOps Forwarder', reason:'Forwarder natively supports structured on-prem sources.' },
      { trigger:'Google-managed GCE VM logs → SecOps, minimize cost/time', answer:'Ops Agent → Cloud Logging → direct ingestion', reason:'Ops Agent is pre-embedded; no extra agent install or infra.' },
      { trigger:'Detect when a data source goes silent', answer:'Cloud Monitoring metric-absence condition', reason:'Triggers when metrics stop appearing within a time window.' },
      { trigger:'Minimize involvement in external user lifecycle management', answer:'Workforce identity federation with the external IdP', reason:'They manage their own users; you only manage IAM roles.' },
      { trigger:'External users authenticate fine but cannot access SecOps', answer:'Check allowedPolicyMemberDomains org policy constraint', reason:'This policy blocks IAM bindings for out-of-domain identities.' },
      { trigger:'Read-only access including detection rules', answer:'roles/chronicle.viewer', reason:'limitedViewer excludes detection rules.' },
      { trigger:'SOAR integration with system in private DC (no inbound connections allowed)', answer:'Remote agent', reason:'Initiates outbound from private network — no inbound needed.' },
      { trigger:'Avoid IP aliasing across NAS sources with feed management', answer:'Ingestion labels (not namespaces)', reason:'Ingestion labels tag the specific source; namespaces separate data logically.' },
      { trigger:'Cloud logs from an entire folder → SecOps', answer:'Aggregated log sink at folder level + Pub/Sub connector', reason:'Captures all projects in folder without per-project config.' },
      { trigger:'Cross-project Pub/Sub to SecOps (push fails)', answer:'Cloud Run function + SecOps Ingestion API key', reason:'Works across project boundaries; event-driven, low-latency.' },
      { trigger:'Audit logs for SecOps data feeds', answer:'Ingest the Google SecOps audit logs into SecOps SIEM', reason:'SecOps generates its own audit logs capturing feed-related admin actions.' },
      { trigger:'Windows log source needed for principal.user.userid coverage', answer:'Ingest Windows Sysmon logs', reason:'Sysmon provides detailed process and network events with user context.' },
    ],
    refs: [
      'chronicle.viewer = read-only incl. rules',
      'chronicle.limitedViewer = read-only excl. rules',
      'Parser extension = additive, no replacement',
      'Extract Additional Fields = fast no-code field surfacing',
      'Bindplane = Syslog / third-party on-prem',
      'Ops Agent = GCE built-in → Cloud Logging',
      'SecOps Forwarder = on-prem structured logs',
      'Remote Agent = private DC (outbound only)',
      'metric-absence condition = silent source detection',
      'Ingestion labels = avoid IP aliasing',
      'Namespaces = logical data separation',
      'Folder-level aggregated sink = all projects in folder',
      'allowedPolicyMemberDomains = blocks external identities',
      'Workforce identity federation = external IdP lifecycle',
    ],
  },
  2: {
    concepts: [
      { term:'UDM Search', detail:'Searches across ALL normalized (parsed) events. Supports field-level filters, aggregations, grouping. Use YARA-L 2.0 syntax for complex queries. Columns feature: customize which UDM fields appear in results view. Faster for ad-hoc investigation than building dashboards first.' },
      { term:'Dashboards & Visualization', detail:'SecOps SIEM dashboards: built-in, no additional cost. Looker Studio connected to BigQuery: best for historical data, JOINs between SCC findings and audit logs, rich charts. Cloud Monitoring: metric-based alerting and dashboards. Use BigQuery when you need to JOIN data sources or query large historical datasets.' },
      { term:'Data Access Scopes', detail:'SIEM-level controls that filter which data a user or group can see. Configured in SecOps SIEM settings, assigned via IAM. Use to give Group A all data and Group B all data except a restricted namespace. Each group gets a separate scope object.' },
      { term:'Entity Enrichment', detail:'HVA/CMDB data: ingest as entity context to add asset sensitivity — helps PRIORITIZE and FILTER alerts (reduces false positives by focusing on what matters). AD context: ingest as user/asset entity context, enriches events at SIEM level automatically. Reference lists: curated lists (IPs, domains, hashes) referenced in detection rules. Watchlists: track specific entities over time for ongoing monitoring.' },
      { term:'Integration Patterns', detail:'Logs (on-prem and cloud) are ingested as EVENTS into SIEM — not as entities. GTI enrichment is done via SOAR integrations during investigation — not ingested as SIEM events. Reference lists for sharing IOCs with other teams integrate directly into operational processes (rules, playbooks).' },
    ],
    patterns: [
      { trigger:'Most efficient way to identify common processes across many servers', answer:'UDM search with aggregations for process-related UDM fields', reason:'Fastest summary without building dashboards first.' },
      { trigger:'Historical data + JOIN between SCC findings and audit logs for leadership dashboard', answer:'Export to BigQuery + Looker Studio', reason:'BigQuery supports large datasets and JOINs; Looker for rich viz.' },
      { trigger:'Add/modify a filter on an existing SIEM dashboard', answer:'Add custom filter directly to the dashboard', reason:'Native feature — no need to copy, export, or contact support.' },
      { trigger:'Group A = all data, Group B = all data except restricted namespace', answer:'Create data access scopes in SIEM settings; assign via IAM', reason:'Scopes enforce data visibility; one scope per group requirement.' },
      { trigger:'Reduce false positives by adding context about asset sensitivity', answer:'Ingest HVA/CMDB data as entity enrichment', reason:'Helps prioritize alerts based on what assets actually matter.' },
      { trigger:'Share IOCs operationally with other teams for use in rules/playbooks', answer:'Create a SecOps reference list and grant access', reason:'Reference lists integrate directly into detection and response workflows.' },
      { trigger:'Customize which UDM fields appear in search result columns', answer:'Use the columns feature in the UDM search results view', reason:'Direct column selection — fastest way to show only relevant data.' },
      { trigger:'Monitor data ingestion AND receive alerts when a source goes silent, minimize cost', answer:'SecOps SIEM dashboards (visualization) + Cloud Monitoring alerting (alerts)', reason:'SIEM dashboards are free; Cloud Monitoring is cost-effective for alerting.' },
      { trigger:'Integration design: logs + GTI with SecOps', answer:'Logs as events in SIEM; GTI enrichment via SOAR integrations', reason:'Two separate integration points — SIEM for storage, SOAR for enrichment.' },
    ],
    refs: [
      'UDM search = all normalized events',
      'Columns feature = customize result fields',
      'BigQuery + Looker = historical + JOINs',
      'SIEM dashboards = free, built-in',
      'Data access scope = SIEM-level data filter',
      'HVA/CMDB = asset sensitivity enrichment',
      'Reference list = curated list for rules/playbooks',
      'Watchlist = monitor specific entities over time',
      'Logs → SIEM as events; GTI → SOAR for enrichment',
    ],
  },
  3: {
    concepts: [
      { term:'YARA-L Rule Structure', detail:'events section: define event variables ($e) and field filters. match section: group events by entity over a time window (e.g., $e.principal.ip over 1h). condition section: boolean logic, thresholds (e.g., #events > 10). outcome section: compute risk scores and variables (risk_score = ...) — these inform alert severity and are available in subsequent detections.' },
      { term:'Rule Types', detail:'Single-event: one matching event triggers the rule. Multi-event (frequency): correlates multiple events over a time window — use for volume anomalies, behavioral patterns. Retrohunt: run any rule against historical ingested data — use to test rules or hunt past threats without waiting for new events.' },
      { term:'Curated Detections', detail:'Google-maintained, pre-built rule sets. Enable immediately — fastest path to coverage. Key categories: Applied Threat Intelligence (ATI) for IOC-based, Cloud Threats for multi-cloud, Windows Threats for Windows-specific, UEBA (User and Endpoint Behavioral Analytics) for behavioral baselines. For fastest threat coverage after a breach, enable curated detections first.' },
      { term:'Applied Threat Intelligence (ATI)', detail:'Fusion Feed: continuously updated threat intelligence. Correlated with your events in real time. IC-Score (Indicator Confidence Score): raise minimum threshold (e.g., ≥60%) to filter low-confidence IOC matches and reduce false positives. Join network events with Fusion Feed data in YARA-L and filter for specific threat actor associations.' },
      { term:'Rule Tuning & False Positive Reduction', detail:'net.ip_in_range_cidr(any $e.principal.ip, "CIDR"): matches if ANY IP in field is in range. not net.ip_in_range_cidr(any $e.principal.ip, "CIDR"): exclude internal IPs — use this form (not "all"). principal.user.type != "service_account": exclude service accounts without maintaining a list. principal.user.email != "bot@domain.com": exclude specific automation accounts. network.asset.ip exclusion: for proxy servers (asset = the proxy itself). Asset group exceptions: for sandboxing environments — preserve detection for production while suppressing test environment noise.' },
      { term:'Entity Graph & Threat Intel Integration', detail:'Entity graph stores first_seen_time for hashes, domains, IPs. Join with entity graph to detect when a hash is new to the environment (compare event timestamp vs. first_seen_time). source_type = "ENTITY_CONTEXT": for threat intel ingested from external platforms (MISP, STIX/TAXII) via native integrations. source_type = "GLOBAL_CONTEXT": for Google-native GTI data.' },
      { term:'SCC Detection', detail:'SCC Posture: combines org policies (preventative controls) + SHA modules (detective controls) in one scoped deployment — correct for both guardrail types. SHA PUBLIC_IP_ADDRESS detector: detects external IPs on VMs — filter results by compliance tag rather than building custom modules. ETD Configurable Bad IP: ingest external IOC-based IP threat intel into SCC. ETD Unexpected Cloud API Call: detect specific principal + method + resource combinations in near real-time.' },
    ],
    patterns: [
      { trigger:'Detect unusual download volume per user vs. baseline, least effort', answer:'Enable UEBA curated detections + Risk Analytics dashboard', reason:'Purpose-built for behavioral baseline analysis — no custom rule needed.' },
      { trigger:'Fastest path to IOC detection coverage after a breach', answer:'Enable curated detection rule sets', reason:'Google-maintained, pre-built, enable immediately — no development time.' },
      { trigger:'TTP-based detection for APT actor (novel, campaign-specific infrastructure)', answer:'Search GTI for actor TTPs → design YARA-L rules based on TTPs', reason:'TTPs persist across campaigns; IOC lists are campaign-specific and stale quickly.' },
      { trigger:'Hunt for APT behavior in historical data when hash capture is unreliable', answer:'Multi-event YARA-L rule on process relationship + run retrohunt', reason:'Behavioral correlation survives even when hashes are inconsistently captured.' },
      { trigger:'Detect when a binary hash appears for the first time in your environment', answer:'YARA-L rule joining file events with entity graph; compare event timestamp vs. hash first_seen_time', reason:'Entity graph tracks first_seen_time per hash.' },
      { trigger:'Reduce false positives from internal IP range', answer:'not net.ip_in_range_cidr(any $e.principal.ip, "192.168.0.0/16") in condition', reason:'"any" checks each IP individually; "not any" excludes if ANY IP is internal.' },
      { trigger:'Reduce false positives from service accounts logging in at odd hours', answer:'Add principal.user.type != "service_account" to condition', reason:'Uses UDM entity-level context — no reference list maintenance needed.' },
      { trigger:'Reduce false positives from a specific automation account', answer:'Add principal.user.email != "backup-bot@domain.com" to condition', reason:'Precisely excludes one account without affecting others.' },
      { trigger:'Reduce false positives from on-premises proxy servers', answer:'Rule exclusion on network.asset.ip (the asset = the proxy)', reason:'network.asset.ip targets the asset (proxy) not principal or target.' },
      { trigger:'Reduce false positives from sandboxing environment for IOC rule', answer:'Add exception for the sandboxing asset group in the detection rule', reason:'Preserves detection for production; suppresses test environment noise.' },
      { trigger:'Store risk score in YARA-L that informs alert severity', answer:'Compute risk_score in the outcomes section', reason:'Outcomes section variables inform alert severity and are available downstream.' },
      { trigger:'Apply external TI (MISP) in YARA-L entity graph join', answer:'Filter $ioc.graph.metadata.source_type = "ENTITY_CONTEXT"', reason:'ENTITY_CONTEXT = external platform ingestion; GLOBAL_CONTEXT = Google-native GTI.' },
      { trigger:'Use IOCs across multiple detection rules efficiently', answer:'Add IOCs to a reference list; reference the list in YARA-L logic', reason:'Reusable, maintainable — update list in one place, all rules benefit.' },
      { trigger:'Detect when privileged Google Group is made public', answer:'Enable Workspace Admin Audit logs + Event Threat Detection', reason:'ETD has built-in Group Open Access detection via Admin Audit logs.' },
      { trigger:'Monitor PCI VMs with external IPs, minimize custom development', answer:'Use existing PUBLIC_IP_ADDRESS SHA detector; filter results by compliance tag', reason:'Reuse existing detector rather than building custom SHA module.' },
      { trigger:'SCC detector for external bad IP threat intel', answer:'ETD Configurable Bad IP template', reason:'Purpose-built template for IOC-based IP signals in SCC.' },
      { trigger:'Detect specific Cloud API call by specific principal in near real-time', answer:'ETD custom Unexpected Cloud API Call rule (principal + method + resource)', reason:'ETD processes Cloud Audit Logs in near real-time for this exact pattern.' },
      { trigger:'Both preventative and detective guardrails in SCC for a business unit', answer:'SCC Posture = org policies (preventative) + SHA modules (detective)', reason:'Posture combines both types in one scoped deployment.' },
      { trigger:'Reduce excessive network connection alerts without reducing effectiveness', answer:'Add a threshold in the YARA-L condition section', reason:'Only alerts after a meaningful count — preserves detection, reduces noise.' },
    ],
    refs: [
      'YARA-L sections: events → match → condition → outcome',
      'not net.ip_in_range_cidr(any $e.ip, "CIDR") ← use any, not all',
      'source_type = ENTITY_CONTEXT ← external TI (MISP)',
      'source_type = GLOBAL_CONTEXT ← Google-native GTI',
      'Curated: ATI | Cloud Threats | Windows Threats | UEBA',
      'IC-Score ≥ 60% → reduce ATI false positives',
      'Retrohunt = run rule against historical data',
      'outcomes section → risk_score variable',
      'network.asset.ip = the asset itself (e.g., proxy)',
      'ETD Bad IP = external IOC-based signals into SCC',
      'ETD Unexpected API Call = principal+method+resource',
      'SCC Posture = org policies + SHA modules',
    ],
  },
  4: {
    concepts: [
      { term:'UDM Investigation Search', detail:'Query all normalized logs. Key investigation fields: principal.ip, principal.user.userid, principal.location.country, target.ip, target.port, network.dns.questions.name, metadata.event_type, network.application_protocol. Group by user+country for impossible travel. Filter target domains with low rolling prevalence for C2 hunting. Use metadata.event_type to find unusual process launches (post-exploitation).' },
      { term:'Entity Explorer & Risk Analytics', detail:'Entity Explorer: opened from the entity identifier in Entity Highlights widget — fastest way (fewest steps) to see enrichment history + related historical cases. Risk Analytics Dashboard: cross-entity risk scoring and correlation — first stop for identifying coordinated attacks across multiple users/assets simultaneously.' },
      { term:'Alerts and IOCs Page', detail:'Shows IOC matches from threat intelligence feeds (including GTI and ATI feeds) against your ingested data. First stop to check if a known-bad IP, domain, or hash appeared in your environment. Also provides reputation context for IP addresses.' },
      { term:'Google Threat Intelligence (GTI)', detail:'Curated threat intelligence platform. Private Scanning: analyze a malware sample without public disclosure (vs. VirusTotal which shares publicly) — use when you must not alert the threat actor. Reports and Analysis: extract IOCs from threat actor reports for detection. Digital Threat Monitoring: dark web + credential monitoring. Vulnerability Intelligence: CVE and patch information. Search a malware hash in GTI for the fastest, most reliable IOC and behavioral data.' },
      { term:'SCC Investigation', detail:'Filter SCC console by resource/service/cluster for targeted findings. Attack path simulations show exposure scores and lateral movement paths — use for GKE lateral movement investigation to prioritize actions before examining raw logs. ETD findings + associated network connection logs: direct evidence for malware/C2 investigation. Container Threat Detection: added binary executed → investigate pod and related resources, research attack methods, notify workload owner. Do NOT delete pod immediately (destroys forensic evidence).' },
      { term:'C2 & Threat Hunting Patterns', detail:'C2 hunting: UDM search for NETWORK_CONNECTION or NETWORK_HTTP events with low rolling prevalence for TARGET domains (outbound to C2). 14-day window for historical baseline. Impossible travel: UDM search for login events, group by principal.user.userid + principal.location.country. Low-prevalence domains: recently registered + rarely contacted = strong C2 indicator. Watchlist: add uncertain-risk entities for ongoing heightened scrutiny without premature escalation.' },
    ],
    patterns: [
      { trigger:'Identify all assets a user interacted with over past 7 days', answer:'UDM Search, query user, group/filter results by asset', reason:'UDM Search covers all normalized event types and log sources.' },
      { trigger:'Check if known-bad IP/domain/hash appeared in your environment', answer:'Alerts and IOCs page in SecOps', reason:'Shows IOC matches from GTI and other TI feeds against your ingested data.' },
      { trigger:'Fastest: enrichment history + related cases for an entity', answer:'Select entity identifier in Entity Highlights → opens Entity Explorer', reason:'Fewest steps; Entity Explorer shows both enrichment history and related cases.' },
      { trigger:'Quickly identify coordinated attack across multiple user accounts', answer:'Risk Analytics dashboard (cross-entity correlation)', reason:'Provides immediate multi-entity view — fastest first step.' },
      { trigger:'Analyze malware without alerting the threat group', answer:'GTI Private Scanning', reason:'Does not share the sample publicly, unlike VirusTotal.' },
      { trigger:'Get reliable IOCs + behavior for a known malware variant quickly', answer:'Search the malware hash in Google Threat Intelligence', reason:'Curated IOCs, behavioral data, and context — faster and more reliable than manual analysis.' },
      { trigger:'Determine if an external identity took any actions in your environment', answer:'Query Cloud Logging/BigQuery filtering by principal email matching the external identity', reason:'Most targeted approach — directly filters all log sources by that identity.' },
      { trigger:'Investigate potential malware on Compute Engine (unusual outbound connections)', answer:'Analyze Event Threat Detection findings + review associated outbound network connections', reason:'ETD findings provide direct evidence and investigation context.' },
      { trigger:'C2 hunting: identify least common network communications (14-day window)', answer:'UDM search for NETWORK_CONNECTION/HTTP events with low rolling prevalence for target domains', reason:'Target domain = outbound destination; low prevalence = rarely contacted = C2 indicator.' },
      { trigger:'Detect impossible travel / compromised account login from multiple countries', answer:'UDM search for login events; group by principal.user.userid + principal.location.country', reason:'Groups reveal geographic dispersion of logins from same account.' },
      { trigger:'Identify post-exploitation activity on an exposed web server', answer:'Search metadata.event_type for unusual process launches / rarely-seen commands', reason:'Process launches reveal command execution — the key indicator of successful compromise.' },
      { trigger:'Container Threat Detection: added binary executed in production workload', answer:'Notify workload owner + follow playbook; investigate pod and related resources. Do NOT delete pod.', reason:'Deleting the pod destroys forensic evidence; investigation must precede containment.' },
      { trigger:'GKE lateral movement: prioritize investigative actions before raw log analysis', answer:'SCC console → filter for cluster → analyze findings timeline + attack path simulations', reason:'Attack path simulations show exposure scores and lateral movement paths for prioritization.' },
      { trigger:'Suppress IOC false positives from a red team exercise', answer:'Navigate to IOC Matches page and mute the specific IOCs', reason:'Targeted muting suppresses false positives without affecting other detections.' },
      { trigger:'Entity with uncertain risk: ensure ongoing scrutiny for weeks', answer:'Add entity to a SecOps watchlist', reason:'Watchlist is designed for monitoring uncertain-risk entities over time.' },
      { trigger:'High-volume phishing: investigate user before taking containment action', answer:'Review user timeline in SecOps first', reason:'Investigation precedes containment — need context before concluding malicious intent.' },
    ],
    refs: [
      'Entity Explorer: opened from entity identifier in Entity Highlights',
      'Risk Analytics: cross-entity risk + correlation',
      'Alerts and IOCs: IOC matches from GTI + ATI feeds',
      'GTI Private Scanning: confidential (not public)',
      'C2 hunting: low rolling prevalence + target domain',
      'Impossible travel: group by userid + location.country',
      'metadata.event_type: find unusual process launches',
      'ETD findings: direct malware/C2 evidence',
      'Container Threat Detection: investigate, do NOT delete pod',
      'Attack path simulations: exposure scores + lateral movement',
      'Watchlist: ongoing monitoring for uncertain-risk entities',
    ],
  },
  5: {
    concepts: [
      { term:'SOAR Environments', detail:'Logical case isolation within one SOAR tenant. Each environment has its own cases. Playbooks can be reused across environments. Use for: MSSP multi-customer isolation, acquired companies (minimizes effort vs. new tenant), Tier 3 restricted cases (move via Cross Environment Policy). Creating an environment is the first step for any new logical boundary requirement.' },
      { term:'SOAR Playbooks: Structure', detail:'Flow conditions: up to 5 branches + Else. Nest a second flow condition on the Else branch to get 8 total paths. Multi-choice questions: alternative for analyst-driven branching. Views: role-based content presentation within a playbook — each role sees their relevant view. Playbook priority: higher priority NUMBER = evaluated LATER — set the All/catch-all playbook to the highest number so specific playbooks attach first.' },
      { term:'SOAR Playbooks: Triggers & Scheduling', detail:'Cron Scheduled Connector: native time-based recurring automation (daily, weekly) — no external infrastructure. Request + Playbook Trigger: for on-demand analyst-initiated workflows with structured inputs. Approval links: allow non-SOAR users to approve/deny actions via email click — playbook continues automatically. Remote agent: for integrations in private networks (initiates outbound, no inbound needed).' },
      { term:'SOAR Case Management', detail:'Case Stages: track time per stage — Change Case Stage action in playbooks captures timestamps at each transition. Case Tags: label cases for filtering and differentiation (e.g., PenTest tag). Pending Actions widget: consolidates ALL manual actions across alerts into one location in Default Case View. Close Case dialog: customizable root cause options — enforce required categorization at closure (e.g., DLP event types). Cross Environment Policy: move cases between environments.' },
      { term:'SOAR Reports', detail:'ROI – Analysts Benchmark: time saved and efficiency gains per analyst. Configure for a time period, filter by analyst. Management – SOC Status: case performance data by priority and environment. Use Statistics in Search + a SOAR job for custom CSV/zip report delivery on a schedule.' },
      { term:'SCC Response & Remediation', detail:'SCC finding lifecycle: when the underlying resource is deleted → finding is automatically marked inactive (do not manually mute or mark fixed — that defeats automated posture management). securitycenter.findingsEditor: minimum IAM role to update finding states (not findingsBulkMuteEditor). SCC Pub/Sub + Cloud Run: automated event-driven ticket creation in external ticketing systems. SCC Marketplace integration (SOAR): install to pull findings into SOAR actions. Remediate at the source: remove the external IP from the VM (not mute the finding) for PCI DSS compliance drift.' },
      { term:'EDR & Endpoint Response', detail:'EDR integration quarantine: isolates asset from network WHILE preserving volatile memory and forensic artifacts. Rebooting destroys forensic evidence. SOAR EDR integrations (not jobs, not remote agents) enable automated containment from playbooks. For ransomware with compromised privileged accounts: revoke OAuth tokens + suspend sessions (fastest automated containment to minimize dwell time).' },
    ],
    patterns: [
      { trigger:'Contain endpoint immediately while preserving forensic artifacts', answer:'EDR integration to quarantine the asset', reason:'Quarantine isolates from network; reboot destroys volatile forensic data.' },
      { trigger:'Logical case separation for multiple customers (MSSP) or acquired company', answer:'Create a SOAR environment for each customer/entity', reason:'Environments provide hard case isolation; playbooks can be reused across them.' },
      { trigger:'Automate a daily task in SOAR, minimize maintenance overhead', answer:'Cron Scheduled Connector + playbook trigger', reason:'Native time-based scheduling — no external VMs or scripts needed.' },
      { trigger:'Track time spent in each incident response stage', answer:'Configure Case Stages + use Change Case Stage action in playbooks', reason:'Captures timestamps automatically at each stage transition.' },
      { trigger:'Label and differentiate cases from a penetration test', answer:'Case tagging with a unique tag (e.g., PenTest)', reason:'Tags enable real-time filtering and differentiation from real incidents.' },
      { trigger:'All analyst manual actions in one place', answer:'Pending Actions widget in the Default Case View', reason:'Centralizes all pending manual actions across all alerts.' },
      { trigger:'Playbook with 8 different paths', answer:'Flow condition (5 branches + Else) → nest second flow condition on Else branch (3 more = 8 total)', reason:'Flow conditions support max 5+Else; nesting gives additional paths.' },
      { trigger:'Non-SOAR user (external) needs to approve a containment action', answer:'Generate approval link, embed in Send Email action body', reason:'Approval links allow non-SOAR users to approve/deny; playbook auto-continues.' },
      { trigger:'SOC director must be emailed before a case is closed (only if escalated)', answer:'Playbook conditional block checking escalation status at closure', reason:'Two branches: escalated → email + close; not escalated → close only.' },
      { trigger:'Ensure specific playbook runs before the catch-all All playbook', answer:'Set higher priority NUMBER on the All playbook', reason:'Higher number = evaluated later; specific playbooks attach first.' },
      { trigger:'SCC finding should be auto-cleared after resource deletion', answer:'Delete the resource; let SCC auto-mark finding inactive', reason:'SCC automatically marks findings inactive when underlying resource is gone.' },
      { trigger:'Enforce DLP root cause categorization when closing a case', answer:'Customize the Close Case dialog with DLP event type root cause options', reason:'Forces categorization at closure; ensures compliance for every DLP case.' },
      { trigger:'Role-based content in a SOAR playbook', answer:'Add Views to the playbook (one per role)', reason:'Views present role-specific content from the same playbook.' },
      { trigger:'Internal vs. external IP classification in SOAR upon ingestion', answer:'Configure internal CIDR ranges in Environment Networks settings', reason:'Auto-classifies IP entities at ingestion — no custom code needed.' },
      { trigger:'Ransomware: minimize dwell time for compromised privileged accounts', answer:'SOAR playbook step: revoke OAuth tokens + suspend sessions for high-risk accounts', reason:'Most direct automated containment action for privileged account compromise.' },
      { trigger:'New playbook needed fast, deployable by junior analysts', answer:'Use Gemini playbook creation feature', reason:'Generates a deployable playbook from a description quickly.' },
      { trigger:'ROI and efficiency metrics for individual analysts', answer:'ROI – Analysts Benchmark report in SOAR Reports', reason:'Built-in report for time saved and efficiency gains per analyst.' },
      { trigger:'Automatically add UDM query result users as entities in an alert', answer:'Create Entity action with Expression Builder (Siemplify integration)', reason:'Automates entity creation from query results — no manual analyst input.' },
      { trigger:'Update SCC finding states from SOAR (permission denied)', answer:'Grant roles/securitycenter.findingsEditor (not findingsBulkMuteEditor)', reason:'findingsEditor is the minimum role to update finding states.' },
    ],
    refs: [
      'SOAR environment = hard case isolation per customer/BU',
      'Flow condition: max 5+Else (nest for 8 paths)',
      'Case Stages = timestamp at each stage transition',
      'Cron Scheduled Connector = time-based recurring SOAR automation',
      'Remote agent = private DC outbound-only integration',
      'Approval links = non-SOAR user approve/deny via email',
      'Playbook priority: higher number = evaluated later',
      'Pending Actions widget = all manual actions in one place',
      'Close Case dialog = enforce root cause at closure',
      'EDR quarantine = isolates + preserves forensics',
      'securitycenter.findingsEditor = update finding states',
      'SCC Pub/Sub + Cloud Run = automated ticket creation',
      'SCC finding auto-inactive when resource deleted',
    ],
  },
};
```

- [ ] **Step 3: Verify in browser console**

```js
console.log(Object.keys(LESSONS)); // ["1","2","3","4","5"]
console.log(LESSONS[1].concepts.length);  // 6
console.log(LESSONS[3].patterns.length);  // 19
console.log(LESSONS[5].refs.length);      // 13
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add DOMAINS and LESSONS content for all 5 domains"
```

---

## Task 4: State + Navigation + Home Screen

**Files:**
- Modify: `index.html` — add `state`, `navigate()`, `renderHome()`, and initial call.

**Deliverable:** Opening `index.html` shows the home screen with 5 domain cards and a Full Mock Exam button.

- [ ] **Step 1: Add state and navigation skeleton**

```js
// ── STATE ──────────────────────────────────────────────────────────────────
const state = {
  screen: 'home',       // 'home' | 'domain' | 'quiz' | 'results'
  domainId: null,       // 1-5 for domain screens; null for full mock
  view: 'study',        // 'study' | 'practice' (domain screen tabs)
  // Quiz state
  questions: [],        // active question list (shuffled)
  current: 0,
  answers: [],          // {selected:[], correct:bool} per question
  // Results
  score: 0,
};

// ── HELPERS ────────────────────────────────────────────────────────────────
function shuffle(arr) {
  const a = [...arr];
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

function el(id) { return document.getElementById(id); }

function setScreen(html) { el('app').innerHTML = html; }

function navigate(screen, params = {}) {
  Object.assign(state, { screen, ...params });
  render();
}

function render() {
  switch (state.screen) {
    case 'home':    renderHome();           break;
    case 'domain':  renderDomain();         break;
    case 'quiz':    renderQuiz();           break;
    case 'results': renderResults();        break;
  }
}
```

- [ ] **Step 2: Add renderHome()**

```js
function renderHome() {
  const cards = DOMAINS.map(d => {
    const count = ALL_QUESTIONS.filter(q => q.domain === d.id).length;
    return `
    <div class="domain-card">
      <div class="domain-num">Domain ${d.id} · ${d.pct}% of exam</div>
      <h3>${d.name}</h3>
      <p class="domain-meta">${count} questions · ${d.desc}</p>
      <div class="card-actions">
        <button class="btn btn-ghost" onclick="navigate('domain',{domainId:${d.id},view:'study'})">📖 Study</button>
        <button class="btn btn-primary" onclick="startQuiz(${d.id})">▶ Practice</button>
      </div>
    </div>`;
  }).join('');

  setScreen(`
    <div class="header">
      <div class="header-left">
        <span class="badge">GPSOE</span>
        <h1>Exam Reviewer</h1>
      </div>
    </div>
    <div class="home-body">
      <div class="home-hero">
        <h2>Google Professional Security Operations Engineer</h2>
        <p>Study by domain, then test yourself. All ${ALL_QUESTIONS.length} official practice questions organized by exam domain.</p>
      </div>
      <div class="domain-grid">${cards}</div>
      <div class="mock-banner">
        <div>
          <h3>Full Mock Exam</h3>
          <p>All ${ALL_QUESTIONS.length} questions in random order · Timed · Simulates real exam conditions</p>
        </div>
        <button class="btn btn-green" onclick="startQuiz(null)">Start Full Exam →</button>
      </div>
    </div>`);
}
```

- [ ] **Step 3: Add startQuiz() and init call**

```js
function prepareQuestions(domainId) {
  const pool = domainId
    ? ALL_QUESTIONS.filter(q => q.domain === domainId)
    : [...ALL_QUESTIONS];
  return shuffle(pool).map(q => {
    const indices = q.opts.map((_, i) => i);
    const shuffled = shuffle(indices);
    return { ...q, opts: shuffled.map(i => q.opts[i]), ans: q.ans.map(a => shuffled.indexOf(a)) };
  });
}

function startQuiz(domainId) {
  const questions = prepareQuestions(domainId);
  navigate('quiz', {
    domainId,
    questions,
    current: 0,
    answers: new Array(questions.length).fill(null).map(() => ({ selected: [], submitted: false, correct: false })),
    score: 0,
  });
}

// ── INIT ───────────────────────────────────────────────────────────────────
render();
```

- [ ] **Step 4: Verify in browser**
  - Home screen renders with 5 domain cards
  - Each card shows correct question count and domain metadata
  - Full Mock Exam banner shows 131 questions
  - Clicking "Study" on a card does not crash (domain screen not rendered yet — OK for now)
  - Clicking "Practice" does not crash (quiz screen not rendered yet — OK)

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add state management, navigation, and home screen"
```

---

## Task 5: Domain Screen (Study Tab)

**Files:**
- Modify: `index.html` — add `renderDomain()` and `renderStudy()`.

**Deliverable:** Clicking "Study" on a domain card shows the lesson view with Key Concepts, Key Patterns, and Quick Reference sections.

- [ ] **Step 1: Add renderDomain()**

```js
function renderDomain() {
  const d = DOMAINS.find(x => x.id === state.domainId);
  const count = ALL_QUESTIONS.filter(q => q.domain === d.id).length;
  const studyContent = state.view === 'study' ? renderStudy() : renderPracticeTab(d, count);

  setScreen(`
    <div class="header">
      <div class="header-left">
        <button class="header-back" onclick="navigate('home')">← Home</button>
        <span class="badge">Domain ${d.id}</span>
        <h1>${d.name}</h1>
      </div>
    </div>
    <div class="domain-body">
      <div class="tabs">
        <button class="tab-btn ${state.view==='study'?'active':''}"
          onclick="navigate('domain',{domainId:state.domainId,view:'study'})">📖 Study</button>
        <button class="tab-btn ${state.view==='practice'?'active':''}"
          onclick="navigate('domain',{domainId:state.domainId,view:'practice'})">▶ Practice</button>
      </div>
      ${studyContent}
    </div>`);
}
```

- [ ] **Step 2: Add renderStudy()**

```js
function renderStudy() {
  const lesson = LESSONS[state.domainId];

  const concepts = lesson.concepts.map(c =>
    `<li><strong>${c.term}:</strong> ${c.detail}</li>`
  ).join('');

  const patterns = lesson.patterns.map(p =>
    `<li>
      <span class="trigger">When asked: <em>${p.trigger}</em></span><br>
      → <span class="answer">${p.answer}</span><br>
      <span style="color:var(--text-muted);font-size:12.5px">Because: ${p.reason}</span>
    </li>`
  ).join('');

  const refs = lesson.refs.map(r =>
    `<div class="ref-item">${r}</div>`
  ).join('');

  const count = ALL_QUESTIONS.filter(q => q.domain === state.domainId).length;

  return `
    <div class="lesson-section">
      <h3>📚 Key Concepts</h3>
      <ul class="concept-list">${concepts}</ul>
    </div>
    <div class="lesson-section">
      <h3>💡 Key Patterns (Exam Tips)</h3>
      <ul class="pattern-list">${patterns}</ul>
    </div>
    <div class="lesson-section">
      <h3>⚡ Quick Reference</h3>
      <div class="ref-grid">${refs}</div>
    </div>
    <div class="lesson-start-btn">
      <button class="btn btn-primary" onclick="startQuiz(${state.domainId})" style="width:100%;padding:14px;font-size:15px">
        Ready? Start ${count}-Question Practice →
      </button>
    </div>`;
}
```

- [ ] **Step 3: Add renderPracticeTab() placeholder (replaced in Task 6)**

```js
function renderPracticeTab(d, count) {
  return `
    <div style="text-align:center;padding:48px 0;color:var(--text-muted)">
      <p style="font-size:17px;margin-bottom:16px">${count} questions in this domain</p>
      <button class="btn btn-primary" onclick="startQuiz(${d.id})" style="padding:14px 28px;font-size:15px">
        Start Practice →
      </button>
    </div>`;
}
```

- [ ] **Step 4: Verify in browser**
  - Click "Study" on any domain card
  - Header shows domain name + back button
  - Study tab is active
  - Lesson sections render: Key Concepts (cards), Key Patterns (bordered list), Quick Reference (grid)
  - "Ready? Start X-Question Practice" button appears at bottom
  - Clicking "Practice" tab shows placeholder with question count
  - Clicking back arrow returns to home screen

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add domain screen with study lesson view"
```

---

## Task 6: Quiz Engine

**Files:**
- Modify: `index.html` — add `renderQuiz()`, `selectOption()`, `submitAnswer()`, `nextQuestion()`.

**Deliverable:** Clicking "Practice" on a domain card → quiz renders with sidebar, question, options. Selecting an option and submitting shows green/red feedback with explanation. Navigation dots work.

- [ ] **Step 1: Add renderQuiz()**

```js
function renderQuiz() {
  const { questions, current, answers } = state;
  const q = questions[current];
  const ans = answers[current];

  // Sidebar dots
  const dots = questions.map((_, i) => {
    let cls = i === current ? 'active' : '';
    if (answers[i].submitted) cls = answers[i].correct ? 'correct' : 'wrong';
    return `<div class="q-dot ${cls}" onclick="jumpTo(${i})">${i+1}</div>`;
  }).join('');

  const correct = answers.filter(a => a.submitted && a.correct).length;
  const wrong   = answers.filter(a => a.submitted && !a.correct).length;
  const remain  = questions.length - answers.filter(a => a.submitted).length;
  const pct     = Math.round((answers.filter(a=>a.submitted).length / questions.length) * 100);

  // Options HTML
  let optCls = (i) => {
    if (!ans.submitted) return ans.selected.includes(i) ? 'selected' : '';
    if (q.ans.includes(i)) return 'correct-ans';
    if (ans.selected.includes(i) && !q.ans.includes(i)) return 'wrong-ans';
    return '';
  };

  const opts = q.opts.map((text, i) => `
    <button class="opt ${optCls(i)}" onclick="selectOption(${i})" ${ans.submitted ? 'disabled' : ''}>
      <span class="opt-letter">${String.fromCharCode(65+i)}</span>
      <span class="opt-text">${text}</span>
    </button>`).join('');

  // Feedback
  let feedback = '';
  if (ans.submitted) {
    const correct_ans = q.ans.map(a => String.fromCharCode(65+a)).join(', ');
    feedback = `<div class="feedback-box ${ans.correct ? 'correct' : 'wrong'}">
      <strong>${ans.correct ? '✓ Correct' : `✗ Correct answer: ${correct_ans}`}</strong><br>${q.exp}
    </div>`;
  }

  // Action buttons
  const canSubmit = ans.selected.length > 0 && !ans.submitted;
  const isLast    = current === questions.length - 1;
  const actions = ans.submitted
    ? `<button class="btn btn-primary" onclick="${isLast ? 'showResults()' : 'nextQuestion()'}">${isLast ? 'View Results' : 'Next →'}</button>
       ${current > 0 ? '<button class="btn btn-ghost" onclick="prevQuestion()">← Prev</button>' : ''}`
    : `<button class="btn btn-primary" onclick="submitAnswer()" ${canSubmit ? '' : 'disabled'}>Submit</button>
       ${current > 0 ? '<button class="btn btn-ghost" onclick="prevQuestion()">← Prev</button>' : ''}`;

  const domainLabel = state.domainId
    ? `Domain ${state.domainId} Practice`
    : 'Full Mock Exam';

  setScreen(`
    <div class="header">
      <div class="header-left">
        <button class="header-back" onclick="confirmExit()">← Exit</button>
        <span class="badge">GPSOE</span>
        <h1>${domainLabel}</h1>
      </div>
      <span style="font-size:13px;color:var(--text-muted)">${current+1} / ${questions.length}</span>
    </div>
    <div class="quiz-layout">
      <div class="quiz-sidebar">
        <div class="sidebar-header">
          <div class="progress-label"><span>Progress</span><span>${pct}%</span></div>
          <div class="progress-bar"><div class="progress-fill" style="width:${pct}%"></div></div>
          <div class="stats-row">
            <div class="stat correct"><div class="num">${correct}</div><div class="lbl">Correct</div></div>
            <div class="stat wrong"><div class="num">${wrong}</div><div class="lbl">Wrong</div></div>
            <div class="stat remain"><div class="num">${remain}</div><div class="lbl">Left</div></div>
          </div>
        </div>
        <div class="q-grid">${dots}</div>
      </div>
      <div class="quiz-main">
        <div class="q-meta">
          <span class="q-num">Q${current+1}</span>
          ${q.multi ? '<span class="multi-tag">Choose 2</span>' : ''}
        </div>
        <div class="q-text">${q.q}</div>
        <div class="options">${opts}</div>
        ${feedback}
        <div class="quiz-actions">${actions}</div>
      </div>
    </div>`);
}
```

- [ ] **Step 2: Add quiz interaction functions**

```js
function selectOption(i) {
  const { questions, current, answers } = state;
  if (answers[current].submitted) return;
  const q = questions[current];
  const sel = answers[current].selected;

  if (q.multi) {
    // Toggle multi-select (max 2)
    const idx = sel.indexOf(i);
    if (idx === -1 && sel.length < q.ans.length) sel.push(i);
    else if (idx !== -1) sel.splice(idx, 1);
  } else {
    answers[current].selected = [i];
  }
  renderQuiz();
}

function submitAnswer() {
  const { questions, current, answers } = state;
  const q   = questions[current];
  const ans = answers[current];
  if (!ans.selected.length || ans.submitted) return;

  ans.submitted = true;
  const sortedSel = [...ans.selected].sort();
  const sortedAns = [...q.ans].sort();
  ans.correct = sortedSel.length === sortedAns.length &&
    sortedSel.every((v, i) => v === sortedAns[i]);
  if (ans.correct) state.score++;
  renderQuiz();
}

function nextQuestion() {
  if (state.current < state.questions.length - 1) {
    state.current++;
    renderQuiz();
  }
}

function prevQuestion() {
  if (state.current > 0) {
    state.current--;
    renderQuiz();
  }
}

function jumpTo(i) {
  state.current = i;
  renderQuiz();
}

function confirmExit() {
  if (confirm('Exit quiz? Your progress will be lost.')) {
    state.domainId
      ? navigate('domain', { domainId: state.domainId, view: 'practice' })
      : navigate('home');
  }
}

function showResults() {
  navigate('results');
}
```

- [ ] **Step 3: Verify in browser**
  - Click "Practice" on Domain 1 card
  - Quiz renders with sidebar dots, question text, and 4 option buttons
  - Submit button is disabled until an option is selected
  - Selecting an option enables Submit; clicking Submit shows green/red feedback with explanation
  - Correct answer highlighted green; wrong selection highlighted red
  - Next → button advances to next question; sidebar dot updates to green/red
  - Progress bar and stats update correctly
  - Clicking a sidebar dot jumps to that question
  - Clicking Exit shows confirm dialog

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add full quiz engine with feedback, navigation, and multi-select support"
```

---

## Task 7: Results Screen + Practice Tab Wiring

**Files:**
- Modify: `index.html` — add `renderResults()`, update `renderPracticeTab()`.

**Deliverable:** Completing a quiz shows a score circle (green if ≥70%, red otherwise), correct/wrong/skipped counts, and retry/home buttons. The Practice tab on a domain screen launches the quiz directly.

- [ ] **Step 1: Add renderResults()**

```js
function renderResults() {
  const { questions, answers, score, domainId } = state;
  const total   = questions.length;
  const wrong   = answers.filter(a => a.submitted && !a.correct).length;
  const skipped = answers.filter(a => !a.submitted).length;
  const pct     = Math.round((score / total) * 100);
  const pass    = pct >= 70;
  const label   = domainId ? DOMAINS.find(d=>d.id===domainId).name : 'Full Mock Exam';

  setScreen(`
    <div class="header">
      <div class="header-left">
        <span class="badge">GPSOE</span>
        <h1>Results</h1>
      </div>
    </div>
    <div class="results-body">
      <h2>${label}</h2>
      <p class="sub">${total} questions · ${pass ? 'Great work!' : 'Keep studying — you\'ve got this.'}</p>
      <div class="score-circle ${pass?'pass':'fail'}">
        <div class="score-pct">${pct}%</div>
        <div class="score-lbl">Score</div>
      </div>
      <div class="verdict ${pass?'pass':'fail'}">${pass ? '✓ Pass (≥70%)' : '✗ Below passing (70%)'}</div>
      <div class="results-grid">
        <div class="res-card c"><div class="big">${score}</div><div class="sml">Correct</div></div>
        <div class="res-card w"><div class="big">${wrong}</div><div class="sml">Wrong</div></div>
        <div class="res-card"><div class="big" style="color:var(--yellow)">${skipped}</div><div class="sml">Skipped</div></div>
      </div>
      <div class="results-actions">
        <button class="btn btn-primary" onclick="startQuiz(${domainId})">Retry →</button>
        ${domainId
          ? `<button class="btn btn-ghost" onclick="navigate('domain',{domainId:${domainId},view:'study'})">← Review Lesson</button>`
          : ''}
        <button class="btn btn-ghost" onclick="navigate('home')">Home</button>
      </div>
    </div>`);
}
```

- [ ] **Step 2: Replace renderPracticeTab() with real content**

```js
function renderPracticeTab(d, count) {
  const answered = 0; // fresh start from domain tab
  return `
    <div style="text-align:center;padding:40px 0">
      <h3 style="font-size:20px;margin-bottom:10px">${count} Questions</h3>
      <p style="color:var(--text-muted);margin-bottom:28px;font-size:14px">
        Questions are shuffled each session. Answer each question, then see the explanation immediately.
      </p>
      <button class="btn btn-primary" onclick="startQuiz(${d.id})" style="padding:14px 32px;font-size:15px">
        Start Practice →
      </button>
      <div style="margin-top:20px">
        <button class="btn btn-ghost" onclick="navigate('domain',{domainId:${d.id},view:'study'})" style="font-size:13px">
          ← Review Lesson First
        </button>
      </div>
    </div>`;
}
```

- [ ] **Step 3: Verify full flow in browser**
  - Home → Study Domain 3 → read lesson → click "Ready? Start Practice" → answer all questions → View Results
  - Score circle is green if ≥70%, red otherwise
  - Correct/wrong/skipped counts are accurate
  - Retry button starts a fresh shuffled quiz
  - Review Lesson goes back to study tab
  - Home button returns to home screen
  - Full Mock Exam: Home → Start Full Exam → complete → results show 131 questions total
  - All 5 domain Study tabs render without errors

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add results screen, wire practice tab, complete full app flow"
```

---

## Task 8: README + GitHub Pages

**Files:**
- Create: `README.md`

**Deliverable:** `README.md` committed. `git push origin main` succeeds. GitHub Pages is enabled and the site URL is accessible.

- [ ] **Step 1: Create README.md**

```markdown
# GPSOE Exam Reviewer

A study tool for the **Google Professional Security Operations Engineer** certification exam.

## Features

- **131 practice questions** organized into all 5 official GPSOE exam domains
- **Study mode**: Key concepts, exam-tip patterns, and quick reference before each domain quiz
- **Practice mode**: Shuffled per-domain quiz with immediate feedback and explanations
- **Full Mock Exam**: All 131 questions in random order
- Single HTML file — works offline, no install needed

## Exam Domains

| # | Domain | Exam Weight |
|---|--------|-------------|
| 1 | Configuring the Environment | 14% |
| 2 | Managing & Configuring Security Telemetry | 22% |
| 3 | Ensuring Ongoing & Effective Detection | 22% |
| 4 | Investigating Security Incidents | 26% |
| 5 | Responding to Security Incidents | 16% |

## Live Site

🔗 **https://xiann-neko.github.io/secops-reviewer/**

## Local Use

Download `index.html` and open it in any browser. No server required.

## Source

Questions adapted from the GPSOE practice exam. All questions preserved as-is; explanations edited only where factually incorrect.
```

- [ ] **Step 2: Commit README**

```bash
git add README.md
git commit -m "docs: add README with domain table and GitHub Pages link"
```

- [ ] **Step 3: Push to GitHub**

```bash
git push origin main
```

- [ ] **Step 4: Enable GitHub Pages**

1. Go to `https://github.com/xiann-neko/secops-reviewer/settings/pages`
2. Under **Source**, select: Branch = `main`, folder = `/ (root)`
3. Click **Save**
4. Wait ~60 seconds, then visit `https://xiann-neko.github.io/secops-reviewer/`

- [ ] **Step 5: Verify**
  - Site loads at GitHub Pages URL
  - Home screen renders correctly on mobile (resize browser to verify responsive grid)
  - All 5 domain Study tabs render lesson content
  - Quiz and results flow works end-to-end

---

## Self-Review

**Spec coverage:**
- ✅ All 131 questions preserved (Task 2)
- ✅ Categorized into 5 official GPSOE domains (Task 2, domain map)
- ✅ Study mode with lesson content derived from Q&A (Task 3)
- ✅ Practice mode per domain (Task 6)
- ✅ Full Mock Exam mode (Task 4 startQuiz(null))
- ✅ Immediate feedback + explanation after each answer (Task 6)
- ✅ Sidebar navigation dots (Task 6)
- ✅ Results screen with score, pass/fail, retry (Task 7)
- ✅ Single index.html, no external dependencies (Tasks 1–7)
- ✅ GitHub Pages deployment (Task 8)
- ✅ Dark theme preserved from source (Task 1 CSS)
- ✅ Multi-select questions handled (Task 6 selectOption)

**Placeholder scan:** No TBDs or TODOs. All code blocks are complete and runnable.

**Type consistency:** `state.domainId` is used consistently as `null` (full mock) or `1–5` (domain). `state.questions[i].domain` is always set via `DOMAIN_MAP`. `answers[i].selected`, `.submitted`, `.correct` are consistent across renderQuiz, selectOption, submitAnswer, renderResults.
