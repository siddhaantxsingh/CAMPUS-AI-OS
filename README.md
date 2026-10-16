<div align="center">

# 🎓 Campus AI OS

### A multi-tenant course knowledge platform — every answer cited, every course isolated, every student anonymous

**Grounded Q&A with sentence-level citations · abstains instead of inventing · flashcards & quizzes from the course's own text · k-anonymous instructor analytics**

<br/>

![React](https://img.shields.io/badge/React_19-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_+_pgvector-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Playwright](https://img.shields.io/badge/Playwright-2EAD33?style=for-the-badge&logo=playwright&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-2E7D32?style=for-the-badge)

![Backend tests](https://img.shields.io/badge/backend_tests-39%2F39_on_real_Postgres-2E7D32?style=flat-square)
![E2E](https://img.shields.io/badge/browser_E2E-passing-2E7D32?style=flat-square)
![Audit](https://img.shields.io/badge/security_audit-10%2F10_defects_fixed-6A1B9A?style=flat-square)
![Grounded](https://img.shields.io/badge/answers-100%25_cited-1565C0?style=flat-square)
![Privacy](https://img.shields.io/badge/analytics-k--anonymous_(k%3D5)-E65100?style=flat-square)
![Docker](https://img.shields.io/badge/docker_image-built_+_smoke--tested-00838F?style=flat-square)

<br/>

<img src="docs/screenshots/ask.png" alt="A student asks a question and gets an answer with a page citation" width="85%"/>

<sub>A student asks a question. Every sentence of the answer is quoted from the course material and cites the document, page and section.</sub>

</div>

---

## ⚡ TL;DR

Instructors upload lecture notes, slides (PDF), handouts and syllabi. Students ask questions and get answers **built only from that material**:

- 📌 **Every sentence carries a citation** — document, page, section — plus a grounding score.
- 🛑 **It abstains** when the material doesn't contain the answer, instead of making something up.
- 🃏 **Study tools** — flashcards, cloze quizzes, a research view and document comparison — are extracted from the same text, so every item points back to a page.
- 🕶️ **Instructors see analytics, never individuals.** Every figure needs at least 5 distinct eligible students behind it.
- 🧱 **Each course is an isolated tenant**, enforced twice: in the service layer *and* in the retrieval engine.
- 🤖 **Optional local LLM answers** (Qwen2.5-0.5B via llama.cpp, CPU) — with every generated sentence checked against its cited source; unsupported sentences are dropped.

> [!NOTE]
> **Security was attacked, not assumed.** An independent adversarial review reproduced **10 defects** against a live server — including 2 high-severity ones (a demoted faculty member keeping instructor rights, and analytics that could identify a single student). All 10 are fixed, and each has its own regression test.

---

## 🏗️ Architecture

```mermaid
flowchart TB
    UI["⚛️ React SPA (same origin)<br/>Ask · Research · Study · Compare · Documents · Analytics"]:::ui
    API["⚡ FastAPI routes<br/>routing only — no authorization logic"]:::api
    SVC["🛡️ Campus service<br/>platform role · course membership · quotas · k-anonymity"]:::svc
    STUDY["🃏 study.py<br/>flashcards · quizzes · research grouping · compare"]:::study
    GW["🔎 Groundwork engine (library)<br/>tenant = course:‹id› · page-bounded chunks<br/>BM25 + LSA · RM3 · rerank · citation checks · injection filter"]:::gw
    LLM["🤖 llama.cpp · Qwen2.5-0.5B<br/>(opt-in)"]:::llm
    DB[("🐘 PostgreSQL + pgvector<br/>app_* tables + Groundwork index")]:::db

    UI -- "HttpOnly session cookie<br/>+ Origin check on writes" --> API --> SVC
    SVC --> STUDY
    SVC --> GW
    STUDY --> GW
    GW -. "generate → verify each sentence" .-> LLM
    SVC --> DB
    GW --> DB

    classDef ui fill:#20232A,stroke:#61DAFB,color:#61DAFB
    classDef api fill:#009688,stroke:#00695C,color:#fff
    classDef svc fill:#6A1B9A,stroke:#4A148C,color:#fff
    classDef study fill:#F9A825,stroke:#F57F17,color:#000
    classDef gw fill:#1565C0,stroke:#0D47A1,color:#fff
    classDef llm fill:#E65100,stroke:#BF360C,color:#fff
    classDef db fill:#4169E1,stroke:#1A3FA0,color:#fff
```

The retrieval and grounding engine is **[Groundwork](https://github.com/siddhaantxsingh/GROUNDWORK)**, my separate RAG engine, used here as a library. Campus AI OS adds everything a real multi-user product needs on top: accounts, roles, tenancy, quotas, privacy, study tools and a UI.

---

## 💬 How a question is answered

```mermaid
sequenceDiagram
    autonumber
    actor S as 🎓 Student
    participant A as ⚡ FastAPI
    participant C as 🛡️ Campus service
    participant DB as 🐘 Postgres
    participant G as 🔎 Groundwork

    S->>A: POST question (session cookie)
    A->>C: ask(user, course, question)
    C->>C: member of this course? (else 404)
    C->>DB: lock user row · check + reserve daily quota (one transaction)
    C->>G: retrieve in tenant course:‹id› only
    G->>G: BM25 + LSA · RM3 · rerank · drop injection-like chunks
    alt material covers the question
        G-->>C: extractive answer · citation per sentence · grounding score
    else nothing relevant
        G-->>C: abstain
    end
    C->>DB: record question for analytics
    C-->>S: answer + citations (doc · page · section) or "not in the material"
```

### Opt-in LLM answers — generated, then verified

```mermaid
flowchart LR
    Q["❓ Question"]:::q --> R["🔎 Retrieve course chunks"]:::r --> L["🤖 Local LLM drafts<br/>an answer with [n] citations"]:::l --> V{"🔍 Each sentence<br/>supported by its<br/>cited source?"}:::v
    V -- "yes" --> K["✅ keep sentence"]:::ok
    V -- "no" --> D["🗑️ drop sentence"]:::bad
    K & D --> E{"Anything left?"}:::v
    E -- "yes" --> OUT["📌 Cited answer"]:::ok
    E -- "no" --> AB["🛑 Abstain"]:::ab

    classDef q fill:#37474F,stroke:#263238,color:#fff
    classDef r fill:#1565C0,stroke:#0D47A1,color:#fff
    classDef l fill:#E65100,stroke:#BF360C,color:#fff
    classDef v fill:#F9A825,stroke:#F57F17,color:#000
    classDef ok fill:#2E7D32,stroke:#1B5E20,color:#fff
    classDef bad fill:#C62828,stroke:#B71C1C,color:#fff
    classDef ab fill:#78909C,stroke:#455A64,color:#fff
```

The checker would rather abstain than show an unsupported sentence — with a 0.5B model that is a frequent outcome, and a deliberate one.

<div align="center">
<img src="docs/screenshots/ask-llm.png" alt="An answer written by the local LLM, with every sentence checked against its cited source" width="80%"/>
</div>

---

## 🧱 Tenancy, roles and privacy

```mermaid
flowchart LR
    subgraph PLAT["🏛️ Platform roles"]
        direction LR
        AD["👑 Admin<br/>users · roles · quotas · stats<br/>❌ cannot read course content"]:::admin
        FA["👩‍🏫 Faculty<br/>granted by an admin only"]:::fac
        ST["🎓 Student<br/>self-registration is student-only"]:::stu
    end

    subgraph C1["📘 Course A · tenant course:1"]
        direction LR
        I1["Instructor(s)<br/>never removes the last one"]:::fac
        M1["Students<br/>joined with a join code"]:::stu
        D1["Documents · chunks · flashcards · quizzes"]:::data
    end

    subgraph C2["📗 Course B · tenant course:2"]
        direction LR
        D2["Documents · chunks · flashcards · quizzes"]:::data
    end

    FA --> I1
    ST --> M1
    M1 -. "any course B id → 404<br/>retrieval never crosses tenants" .-x D2

    classDef admin fill:#6A1B9A,stroke:#4A148C,color:#fff
    classDef fac fill:#1565C0,stroke:#0D47A1,color:#fff
    classDef stu fill:#2E7D32,stroke:#1B5E20,color:#fff
    classDef data fill:#ECEFF1,stroke:#90A4AE,color:#000
    style PLAT fill:transparent,stroke:#6A1B9A,stroke-dasharray:5 5
    style C1 fill:transparent,stroke:#1565C0,stroke-dasharray:5 5
    style C2 fill:transparent,stroke:#2E7D32,stroke-dasharray:5 5
```

### k-anonymous instructor analytics

```mermaid
flowchart LR
    F["📊 Requested figure<br/>topics · volume · abstention rate<br/>cited docs · feedback · quiz scores"]:::f --> E["🧮 Count distinct eligible students<br/>behind it"]:::e
    E --> EL["Eligible = member for > 1 day before asking<br/>instructors excluded"]:::note
    EL --> K{"≥ 5 students?"}:::k
    K -- "yes" --> SHOW["✅ Show the aggregate"]:::ok
    K -- "no" --> HIDE["🕶️ Show only the total question count"]:::hide

    classDef f fill:#1565C0,stroke:#0D47A1,color:#fff
    classDef e fill:#6A1B9A,stroke:#4A148C,color:#fff
    classDef note fill:#ECEFF1,stroke:#90A4AE,color:#000
    classDef k fill:#F9A825,stroke:#F57F17,color:#000
    classDef ok fill:#2E7D32,stroke:#1B5E20,color:#fff
    classDef hide fill:#78909C,stroke:#455A64,color:#fff
```

Students should be able to ask "naive" questions without being identifiable. The one-day membership rule exists because the audit showed **4 fresh sock-puppet accounts** could push a single student's topic over the threshold ([ADR 0003](docs/decisions/0003-k-anonymous-analytics.md)).

---

## ✨ Features

| Area | What exists |
|---|---|
| 🔐 **Accounts** | Email + password (scrypt, N=2¹⁵) with HttpOnly session cookies; logout and password change revoke sessions; self-service account deletion |
| 👥 **Roles (RBAC)** | Platform: admin, faculty, student. Course: instructor, student. Join codes for students; instructors can be added and removed, but never the last one |
| 🧱 **Course workspaces** | Each course is an isolated retrieval tenant — members only, any other id returns 404 |
| 📄 **Documents** | PDF (page-aware), Markdown, HTML, text. Idempotent upload. Deleting a document also deletes its flashcards and any quiz that quotes it |
| 💬 **Ask** | Grounded extractive answers with per-sentence citations and a grounding score, or opt-in local-LLM answers with the same checks. Injection-like chunks are excluded |
| 🔬 **Research** | RM3-expanded retrieval with reranking; evidence grouped by document, with pages |
| 🃏 **Study** | Flashcards from definitional sentences; cloze multiple-choice quizzes whose answers stay hidden until submission, with source pages in the results |
| ⚖️ **Compare** | Key terms shared by and distinctive to two documents, plus matching sections with similarity scores |
| 📊 **Analytics** | k-anonymous (k = 5) volume, abstention rate, topics, most-cited documents, feedback, quiz scores |
| 🚦 **Quotas** | Daily questions per user and storage per course — correct under concurrent requests |
| 👍 **Feedback** | Thumbs up/down and a comment, on your own answers only |
| 🛠️ **Admin** | Users, roles, disable (revokes sessions), per-user quotas, platform stats — but no access to course content |

<details>
<summary><b>📸 More screenshots</b></summary>

<br/>

**Analytics below the k threshold — the instructor sees only a total:**

<img src="docs/screenshots/analytics.png" alt="Instructor analytics hidden below k = 5" width="80%"/>

**Quiz with source pages after submission:** [`docs/screenshots/quiz.png`](docs/screenshots/quiz.png)

</details>

---

## 🔐 Security audit — 10 defects found, reproduced and fixed

An independent adversarial reviewer with read-only access reproduced every claim with a script against a live server and database. Full report: [`docs/audit.md`](docs/audit.md).

```mermaid
pie showData
    title Audit findings by severity (all fixed)
    "High" : 2
    "Medium" : 5
    "Low-medium" : 1
    "Low" : 2
```

| ID | Severity | Defect (reproduced) | Fix |
|---|---|---|---|
| C1 | 🔴 high | Faculty demoted to student kept instructor rights | Instructor actions need membership **and** a faculty/admin platform role; demotion converts memberships |
| C2 | 🔴 high | Analytics exposed single students; 4 sock-puppets pushed one student over k=5 | Every breakdown needs ≥ k eligible students; eligibility needs > 1 day of membership |
| C3 | 🟠 medium | Account deletion kept feedback comments and question text | Free text erased; only anonymous counts kept |
| C4 | 🟠 medium | 20 parallel questions with quota 1 recorded 4 | Quota check + reservation in one transaction with a row lock |
| C5 | 🟠 medium | 10 parallel uploads reached 2× the storage quota | Check–ingest–record serialized per course |
| C6 | 🟠 medium | Instructor quizzes vanished when the instructor deleted their account | `official` stored at creation; last instructor can't delete their account |
| C7 | 🟡 low-med | Quizzes kept sentences of a deleted document | Deleting a document deletes quizzes quoting it |
| C8 | 🟠 medium | Compare built an \|A\|×\|B\| matrix under the global lock (2.3 s stall) | Snapshot under lock, blockwise compute outside, ≤ 2000 chunks per side |
| C9 | 🟢 low | Quiz prompt leaked later occurrences of the answer | Blank every whole-word occurrence |
| C10 | 🟢 low | Register / password change / deletion unlimited | Per-IP and per-user rate limits |

**Checked and clean:** CSRF (missing/`null` Origin, look-alike Referers), SPA path traversal, XSS sinks, cross-course document access, session fixation, quiz-answer exposure.

The browser E2E test also caught a **stemming bug in Groundwork** ("resolved" vs "resolves") that made a question miss the right sentence — fixed upstream with a regression test.

<details>
<summary><b>🛡️ Threat model (from <code>SECURITY.md</code>)</b></summary>

| Threat | Control |
|---|---|
| Cross-course access (IDOR) | Membership check on every operation + per-course retrieval tenant; foreign ids → 404 |
| Privilege escalation | Self-registration is student-only; faculty granted by admin |
| CSRF | SameSite=Lax cookie + exact Origin/Referer allow-list on every non-GET call |
| Session theft | Only SHA-256 of tokens stored; HttpOnly; 7-day expiry; revoked on logout, password change, disable |
| Password attacks | scrypt, constant-time compare, dummy hash for unknown emails, rate limits |
| Prompt injection in uploads | Groundwork excludes flagged chunks; extractive answers can't follow instructions |
| XSS | React escaping only; no raw HTML; CSP `default-src 'self'` |
| Resource exhaustion | Upload size, atomic storage and question quotas, compare caps, query length limits |

</details>

---

## 🧪 Testing

```mermaid
flowchart LR
    B["🐍 Backend API · 39 tests<br/>real PostgreSQL + pgvector<br/>incl. 10 audit regressions"]:::b --> FE["⚛️ Type check + build"]:::f --> E2E["🎭 Playwright E2E<br/>real server · fresh DB"]:::e --> LL["🤖 LLM E2E<br/>(opt-in)"]:::l --> DK["🐳 Docker smoke test"]:::d

    classDef b fill:#1565C0,stroke:#0D47A1,color:#fff
    classDef f fill:#3178C6,stroke:#1E4E8C,color:#fff
    classDef e fill:#2EAD33,stroke:#1B7A1F,color:#fff
    classDef l fill:#E65100,stroke:#BF360C,color:#fff
    classDef d fill:#2496ED,stroke:#1565C0,color:#fff
```

| Suite | Last result |
|---|---|
| Backend API (`cd backend && pytest`) | ✅ **39 passed** against real PostgreSQL + pgvector |
| Frontend type check + build | ✅ ok |
| Browser E2E (Chromium, real server, fresh DB) | ✅ passed |
| Browser E2E with a local LLM (opt-in) | ✅ passed |
| Docker image build + smoke test | ✅ `SMOKE OK` |

**The E2E story:** an admin promotes a registered user to faculty → faculty creates a course and uploads a document → a student registers, joins with the code, asks a question and gets a cited answer → gets an abstention on an off-topic question → browses flashcards → takes a quiz → the instructor's analytics stay hidden below k.

| Invariant | Covered by |
|---|---|
| Non-members get 404 for every course resource; admin role grants no content access | `test_non_members_cannot_see_course` |
| Demoted faculty lose instructor rights immediately | `test_c1_*` |
| Retrieval never crosses courses, even with foreign document ids | `test_courses_are_isolated_in_retrieval` |
| Quiz answers are never sent before an attempt | `test_quiz_hides_answers_and_grades` |
| Quotas hold under concurrency | `test_c4_*`, `test_c5_*` |
| Course deletion is resumable after a crash and invisible while in progress | `test_course_deletion_is_complete_and_resumable` |

Logs: [`test-logs/`](test-logs/) and [`docs/evidence/2026-10-15/`](docs/evidence/2026-10-15/).

---

## 🚀 Build, test, run

Requires the [Groundwork](https://github.com/siddhaantxsingh/GROUNDWORK) repo checked out next to this one, and PostgreSQL with pgvector.

```bash
pip install -e ../groundwork -e 'backend[dev]'
(cd frontend && npm ci && npm run build)

cd backend && pytest                      # 39 tests · PostgreSQL + pgvector on 127.0.0.1:5432
cd ../frontend && npx playwright test     # browser E2E against the real server and a fresh DB
```

**Run it:**

```bash
export CAMPUS_DATABASE_URL=postgresql://user:pass@db:5432/campus
export CAMPUS_ALLOWED_ORIGINS=https://campus.example.edu
export CAMPUS_BOOTSTRAP_ADMIN='admin@example.edu:<long random password>'   # first start only
export CAMPUS_STATIC_DIR=$PWD/frontend/dist CAMPUS_SECURE_COOKIES=1
cd backend && uvicorn --factory campus.app:create_app --host 0.0.0.0 --port 8000
```

**Docker:** `scripts/docker_build.sh campus-ai-os` (passes Groundwork as a named build context) → multi-stage image (Node build + Python 3.12-slim, non-root user, `/healthz` healthcheck). Optional LLM answers: `CAMPUS_LLM_PROVIDER=llamacpp`. More in [`docs/deployment.md`](docs/deployment.md).

---

## ⚖️ Engineering trade-offs

| Decision | Why | Record |
|---|---|---|
| **Extraction by default, generation opt-in** | Answers and study items are real sentences from the material, so nothing is invented | [ADR 0002](docs/decisions/0002-extractive-study-tools.md) |
| **Course = tenant** | Isolation enforced twice: service membership checks *and* engine tenant scoping | [ADR 0001](docs/decisions/0001-course-as-tenant.md) |
| **Privacy over insight** | Small courses see little analytics — by design | [ADR 0003](docs/decisions/0003-k-anonymous-analytics.md) |
| **Single process** | Simple and correct today; multi-replica needs advisory locks and a shared index | [architecture](docs/architecture.md) |

---

## 📁 Repository layout

```text
CAMPUS-AI-OS/
├── backend/
│   ├── campus/        # app.py (routes) · service.py (authz, quotas, k-anon) · study.py · auth.py · db.py
│   └── tests/         # test_campus.py · test_audit_regressions.py
├── frontend/
│   ├── src/           # React 19 + TypeScript SPA
│   └── e2e/           # Playwright specs (default + LLM)
├── scripts/           # docker_build · docker_smoke · e2e-server
├── docs/              # architecture · audit · testing · deployment · beta-plan · decisions/ · screenshots/ · evidence/
├── Dockerfile
└── SECURITY.md
```

---

## 🧭 Limitations and roadmap

**Honest status:** a working, end-to-end-verified vertical slice. Not yet deployed or used by real students.

| Next | Why |
|---|---|
| 🧑‍🎓 Beta with 1–2 real courses ([plan](docs/beta-plan.md)) | No usage or satisfaction data exists yet |
| 🤖 Larger LLM (local 3B or hosted), same citation checks | The 0.5B model is slow on CPU and abstains often after enforcement |
| 🔑 OIDC / university SSO, email verification, MFA | Accounts are email + password today |
| 📈 Advisory locks + shared index | Run multiple replicas |
| 🧠 Spaced-repetition flashcards | Study tools currently rely on definitional sentences |
| 🛡️ Differential privacy / verified enrolment | k-anonymity slows, but doesn't fully stop, a patient instructor creating accounts in advance |

Full list: [`docs/limitations.md`](docs/limitations.md).

---

<div align="center">

**Built so a student can trust every answer — and ask any question without being watched.**

<sub>MIT © siddhaantxsingh · Security reports: see <a href="SECURITY.md">SECURITY.md</a></sub>

</div>
