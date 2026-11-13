# Railwise — Supplier Risk Intelligence for Rail Procurement

Railwise is a supplier-risk and sourcing decision-support prototype for rail procurement. It combines an interactive web application, a FastAPI service, and a 10,000-supplier dataset to help buyers explore risk, compare shortlists, record evidence, and prepare commercial reviews.

**Track:** Siemens Supplier Risk Intelligence · **Repository:** [Ganeshlakshmana/Nebius-hack](https://github.com/Ganeshlakshmana/Nebius-hack)

## Project layout

| Path | Purpose |
|---|---|
| `app/` | Web application pages, API routes, and shared styling. |
| `components/` | Dashboard workspaces and reusable interface components. |
| `backend/app/` | FastAPI routes, data stores, services, and specialist agents. |
| `backend/tests/` | Backend API, component-screening, and guardrail tests. |
| `data/` | Supplier CSV snapshot and risk-criteria documentation. |
| `public/` | Static browser data and public assets. |
| `db/`, `drizzle/` | Application database schema and migrations/configuration. |
| `integrations/n8n/` | Optional commercial-monitor workflow templates and setup notes. |
| `docs/` | Architecture, challenge readiness, audits, and demo guidance. |
| `deploy/` | Deployment and runtime configuration. |
| `tests/` | Frontend and integration-level tests. |

## Demo walkthrough

1. **Portfolio → Suppliers:** show the risk distribution, search/filter suppliers, and open one risk profile. The profile comes from the supplied CSV snapshot.
2. **Compare:** select three suppliers and compare risk dimensions side by side. Ask the assistant: `Which of SUPP_00012, SUPP_00450 and SUPP_00731 has the lowest financial risk?` It routes the question to a deterministic dataset comparison and returns the values with provenance.
3. **Component → Sourcing:** ask for a component screen, then show the mandatory evidence gates. Missing or failed qualification evidence prevents a supplier from becoming ready for authorized review, regardless of its risk rank.
4. **Commercial:** load the clearly marked synthetic quotes, set contract status and buyer constraints, and run the purchasing-set comparison. Show excluded candidates, modeled allocations, contract alerts, and negotiation questions. The result is preparation for a buyer, not a negotiated price or award.
5. **Research (if time):** research a real company, inspect the linked source evidence and model-review status, and explain why results remain partial until a person verifies them.

For a narrated script and presentation notes, see [docs/demo-script.md](docs/demo-script.md).

## How one request flows through Railwise

The chart shows the app request path and the optional n8n path. **Not every request runs every agent.** A chat turn enters the LangGraph supervisor, which may use the configured Nebius model to propose a bounded plan, validates that plan, and calls only relevant allowlisted tools. If the model is offline or unavailable, the deterministic data-answer path is used where supported. Dedicated workspaces call their domain endpoint through the same API. Supplier arithmetic stays deterministic; the model plans chat actions and can provide bounded, quote-checked research review when configured.

```mermaid
flowchart TB
  subgraph Buyer[Buyer workspace]
    USER[Buyer]
    UI[Railwise web dashboard]
    SNAP[CSV-derived browser snapshot<br/>public/suppliers.json]
    USER --> UI
    UI -->|Portfolio and comparison views| SNAP
  end

  subgraph Edge[Public app server]
    NEXT[Server API routes<br/>rate limits, timeouts, secret handling]
    COMM[POST /api/commercial<br/>configured n8n webhook<br/>or direct backend fallback]
    PROXY[Authenticated internal commercial proxy<br/>/api/internal/commercial/*]
    UI -->|Chat, research, sourcing| NEXT
    UI --> COMM
  end

  subgraph Core[Private FastAPI backend]
    API[FastAPI API]
    CHAT[POST /chat]
    PLAN[LangGraph plan node]
    VALIDATE[Validate schema, IDs and limits]
    TOOLS[Allowlisted specialist tools<br/>profile · compare · explain · filter<br/>component · alternatives · memo · sourcing · commercial · research]
    FALLBACK[Deterministic dataset fallback<br/>for supported questions]
    RESPOND[Compose grounded answer<br/>and provenance]
    DIRECT[Dedicated workspace endpoints<br/>research · sourcing · component · commercial]
    API --> CHAT --> PLAN --> VALIDATE --> TOOLS
    TOOLS -->|Next validated action, up to 3| TOOLS
    TOOLS -->|Results or partial failures| RESPOND
    PLAN -->|Model unavailable or offline| FALLBACK --> RESPOND
    API --> DIRECT --> TOOLS
  end

  NEXT -->|Private backend URL| API
  COMM -.->|No n8n webhook configured| API
  COMM -->|HTTPS + server-held key<br/>when configured| N8N
  PROXY -->|HTTPS + separate server-held key| API

  subgraph Data[Data and state]
    CSV[(Provided supplier_risk_dataset.csv<br/>10,000 anonymized suppliers)]
    DUCK[(In-memory DuckDB<br/>read-only analytics view)]
    SQLITE[(SQLite<br/>research reports · Tavily cache<br/>commercial cases · events · audit log)]
    CSV --> DUCK
  end
  TOOLS -->|Risk lookups and rankings| DUCK
  TOOLS -->|Persist reports, alerts and cases| SQLITE

  subgraph Research[Optional model and public research providers]
    TAVILY[Tavily Search and Extract]
    TF[Nebius Token Factory<br/>Nemotron model, if configured]
  end
  TOOLS -->|Real-company research| TAVILY
  TAVILY -->|Source-linked excerpts| TOOLS
  PLAN -->|Optional bounded JSON plan| TF
  TF -->|Validated plan| PLAN
  TOOLS -->|Optional bounded advisory evidence review| TF
  TF -->|Quote-checked advisory findings| TOOLS

  subgraph Automation[Optional n8n Cloud contract monitor]
    N8N[Authenticated webhook and daily schedule]
  end
  N8N -.->|Commercial evaluation and scheduled scan<br/>only after imported, configured and activated| PROXY
  SQLITE -->|Saved alerts| API
  RESPOND --> API --> NEXT --> UI
```

### Explaining the assistant in a demo

For the question *“Which supplier has the lowest financial risk?”*, the path is:

`Dashboard chat → /api/chat → FastAPI /chat → LangGraph plan and validation → compare tool → DuckDB supplier snapshot → grounded answer and comparison table → dashboard`

The supervisor is a bounded orchestrator, not a group of agents debating autonomously. With a model configured, it proposes up to three allowlisted actions; the application validates supplier IDs and the full plan before execution. A deterministic fallback handles supported dataset questions when the model is unavailable, and partial failures are reported instead of claiming a tool succeeded. In the chat UI, internal graph execution details are hidden from the buyer.

For *“Research Knorr-Bremse”*, the research service asks Tavily for public sources, extracts selected pages, checks company/domain context and incident-vs-policy signals, then may ask Nemotron on Nebius Token Factory for a constrained, quote-backed review. Sources and excerpts are retained with the report. Model findings are advisory and unverified; research does not alter the CSV risk scores.

## What the system does

- **Supplier portfolio:** search, filter, rank and compare up to 20 anonymized suppliers across supplied risk dimensions and operating metrics.
- **Conversational assistant:** routes dataset questions to profile, comparison, explanation, ranking, component, sourcing, research, memo or commercial tools.
- **Component screening:** applies transparent component risk factors to supplier records. The challenge data has no bill-of-materials mapping, so component assignments and component scores are **simulated screening overlays**, not evidence that a supplier makes or is qualified for a part.
- **Sourcing workspace:** scores the buyer's shortlist against buyer-authored requirements and evidence. Missing mandatory evidence remains open; a failed mandatory requirement blocks review readiness.
- **Commercial workspace:** checks entered quotes and contract assumptions for comparability, validity, delivery lead time, capacity, minimum order, risk ceiling and supplier concentration. It enumerates feasible allocations and drafts negotiation questions.
- **Company research:** gathers public evidence with source links and labels identity, extraction and interpretation limits. No anonymous `SUPP_...` ID is assumed to identify a real company.
- **Decision artifacts:** export a shortlist summary and review packets for human use.

## Data integrity and decision boundaries

| Information | Source and status |
|---|---|
| Supplier IDs, names, risk factors, Total Risk Index, delivery and purchasing metrics | Supplied `supplier_risk_dataset.csv`; original values are retained. |
| Total Risk Index explanation | The source does not include its formula or weights. Driver contributions are estimates, not the original formula. |
| Component assignment and component risk | Deterministic **synthetic overlay** because the CSV has no component/BOM relationships. |
| Qualification evidence and contract facts | Buyer-entered records; authenticity, legal interpretation and applicability are not independently verified. |
| Commercial quotes and operation feed | Synthetic demo inputs unless replaced with reviewed buyer data. |
| Company research | Public web sources retrieved on demand; identity and claims require human verification. No results do not mean low risk. |

Risk values use 0 = lower risk and 100 = higher risk; delivery/confirmation rates retain their performance meaning. The dataset has no snapshot date. Railwise does not certify compliance, infer engineering capability, send supplier communications, commit purchase orders, or automatically award contracts.

Challenge input files: [`data/supplier_risk_dataset.csv`](data/supplier_risk_dataset.csv) and [`data/risk_criteria_description.md`](data/risk_criteria_description.md).

## Stack and backend endpoints

- **Frontend:** React 19, TypeScript, Tailwind CSS, and shadcn-style components, served through Vinext on Vite.
- **API:** FastAPI and Pydantic; **chat orchestration:** LangGraph StateGraph; optional model planning uses an OpenAI-compatible endpoint such as Nebius Token Factory.
- **Supplier analytics:** in-memory DuckDB reads the supplied CSV through a read-only analytics view.
- **Application state:** SQLite stores reports, search cache, operation events, commercial cases and audit events.
- **Research:** Tavily Search and Extract, with bounded result counts and cached search results.
- **Hosting:** deployment assets support a Nebius CPU VM, Docker Compose, and a Caddy HTTPS reverse proxy; the backend is intended to remain on a private Docker network.

Useful backend endpoints:

| Endpoint | Purpose |
|---|---|
| `GET /suppliers` and `GET /suppliers/{id}` | Filtered portfolio and supplier profiles |
| `POST /compare` and `GET /suppliers/{id}/explain` | Risk comparison and score explanation |
| `POST /chat` | LangGraph-supervised assistant |
| `POST /components/analyze` | Component-specific screening |
| `POST /sourcing/evaluate` | Requirement and evidence gate evaluation |
| `POST /commercial/evaluate` | Quote, contract-assumption and allocation analysis |
| `POST /research` and `GET /research/{id}` | Start and retrieve a company research report |
| `POST /commercial/scan` and `GET /commercial/alerts` | Refresh and read saved commercial alerts |

## Run locally

Requires Node.js 22.13 or newer, npm, and Python 3.11 or newer. Run the backend and web app in separate terminals from the repository root.

### 1. Configure and run the backend

```powershell
python -m pip install -e "backend[test]"
Copy-Item .env.example .env
python -m uvicorn app.main:app --app-dir backend --host 127.0.0.1 --port 8000
```

The API health check is [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health), and interactive OpenAPI documentation is available at [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs).

### 2. Install and run the web app

In a second terminal at the repository root:

```powershell
npm run install:ci
npm run dev
```

Open the local URL printed by Vite (normally [http://127.0.0.1:5173](http://127.0.0.1:5173)). The frontend chat proxy uses `BACKEND_URL`, which defaults to `http://127.0.0.1:8000`.

### Environment configuration

The checked-in `.env.example` documents the supported settings. The core app runs in offline mode by default; live provider access is optional.

| Variable | Purpose |
|---|---|
| `OFFLINE_MODE` | Keep external research and model calls disabled when `true` (default). Set to `false` only when providers are configured. |
| `DATASET_PATH` | Optional path to the supplier CSV; defaults to `data/supplier_risk_dataset.csv`. |
| `BACKEND_URL` | Backend URL used by the frontend server-side API proxy. |
| `TAVILY_API_KEY` | Optional key for live company research and search. |
| `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL` | Optional OpenAI-compatible model settings for bounded planning/review. |
| `INTERNAL_AGENT_API_KEY`, `INTERNAL_AGENT_SECRET` | Credentials for signed internal agent messages; use private, strong values outside local development. |
| `COMMERCIAL_API_KEY`, `N8N_COMMERCIAL_WEBHOOK`, `N8N_WEBHOOK_KEY` | Optional server-side commercial workflow integration settings. |
| `FEED_SPEED_SECONDS` | Simulated operations-feed timing setting. |

Keep `.env` local and never commit real credentials. Browsing the demo and deterministic dataset calculations do not require provider keys.

## n8n negotiation-preparation workflow

The repository includes an authenticated app-to-n8n bridge and importable workflow templates in [`integrations/n8n/`](integrations/n8n/). **The templates are not yet imported or activated in n8n Cloud.** Once connected, n8n will orchestrate commercial evaluation and a scheduled alert refresh; it will not negotiate with suppliers or send messages. See [`integrations/n8n/README.md`](integrations/n8n/README.md) for setup, credential separation, and test order.

## Tests and references

```powershell
python -m pip install -e "backend[test]"
python -m pytest backend/tests -q
npm run lint
npm run build
```

The backend tests use pytest. `npm run lint` and `npm run build` validate the web application. See [challenge readiness and gaps](docs/challenge-readiness.md), [data and assistant audit](docs/DATA-AND-ASSISTANT-AUDIT.md), [LangGraph supervisor controls](docs/LANGGRAPH-SUPERVISOR.md), and [the demo script](docs/demo-script.md).
