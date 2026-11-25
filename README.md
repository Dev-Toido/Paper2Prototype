# Paper2Prototype

**From Research Paper to Working Prototype.**

Paper2Prototype is a full-stack AI research engineering platform that helps students, researchers, and developers transform academic PDF papers into structured analyses, editable experiment blueprints, verified dataset/library recommendations, modular Python starter code, sandboxed executions, reproducibility audits, and downloadable PDF technical reports.

---

## Architecture & Tech Stack

- **Frontend:** React 18, TypeScript, Tailwind CSS, Radix/shadcn-style UI, Lucide React, React Router
- **PDF Processing:** `pdfjs-dist` (page-by-page extraction, canvas preview, password/corruption handling) + optional in-browser OCR (`tesseract.js`) for scanned PDFs
- **Code Editor & Export:** Monaco Editor (`@monaco-editor/react`), `jszip` (full project ZIP export), `jspdf` (12-section PDF report export)
- **Charts:** Recharts
- **Backend & Auth:** Supabase Auth, Supabase PostgreSQL (9 tables with strict Row Level Security), Supabase Storage (`research-papers` private bucket), Supabase Edge Functions (Deno)
- **AI Engine:** Google Gemini API (`GEMINI_API_KEY` stored strictly on Supabase Edge Functions — never in frontend code)
- **Execution Isolation:** External containerized Python sandbox (`sandbox-runner/`) with CPU/RAM/timeout limits, `--network none`, stripped env vars, and ephemeral `/tmp` storage

---

## 9-Stage Workflow

1. **PDF Upload** — Drag & drop, magic-byte validation, page preview, corrupted/password handling, scanned PDF detection + optional OCR, secure Supabase Storage upload
2. **Research Paper Analysis** — 14 structured fields extracted across all paper pages (chunked synthesis without silent truncation), page citations, and strict separation of extracted facts vs. AI suggestions
3. **Methodology Extraction** — Pipeline steps, architecture details, algorithms, mathematical formulations, and hyperparameters
4. **Experiment Blueprint Generation** — Editable Beginner, Intermediate, and Advanced plans
5. **Dataset & Resource Finder** — Live Hugging Face Hub & PyPI lookups + transparent official search links for Kaggle, UCI, and Google Dataset Search (zero fabricated URLs)
6. **Python Code Generation** — Multi-file starter project in Monaco Editor with `requirements.txt`, `README.md`, Explain Code, and ZIP download
7. **Experiment Execution** — Delegates strictly to an external isolated sandbox (`SANDBOX_API_URL`) or guides local execution via ZIP download
8. **Result Analysis** — Displays actual measured execution metrics & charts alongside paper-reported reference benchmarks
9. **Final Report Generation** — 12-section downloadable PDF report + Research Reproducibility Score (100-point documentation completeness audit)

---

## Setup & Deployment Instructions

### 1. Prerequisites
- Node.js 20+
- [Supabase CLI](https://supabase.com/docs/guides/cli) (`npm install -g supabase` or `npx supabase`) authenticated via `supabase login` (or `SUPABASE_ACCESS_TOKEN`)
- A [Supabase](https://supabase.com) project and its database password
- A [Google AI Studio Gemini API key](https://aistudio.google.com/apikey) (new format starts with `AQ.`; legacy `AIza...` keys also work)
- Docker (optional, for running the isolated `sandbox-runner/`)

### 2. Database & Storage Migration
Link your Supabase project and push the SQL migration in `supabase/migrations/001_initial_schema.sql`:

```bash
supabase login
supabase link --project-ref <your-project-ref>
supabase db push
```

This creates all 9 tables (`profiles`, `research_projects`, `uploaded_papers`, `paper_analyses`, `experiment_blueprints`, `generated_code`, `experiment_runs`, `experiment_results`, `generated_reports`), RLS policies, auth profile triggers, and the private `research-papers` storage bucket.

### 3. Deploy Supabase Edge Functions & Secrets
Set server-side secrets (NEVER put `GEMINI_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, or `SANDBOX_API_KEY` in `frontend/.env`):

```bash
supabase secrets set GEMINI_API_KEY=AQ.your_actual_gemini_api_key
supabase secrets set GEMINI_MODEL=gemini-3.8-flash
# Optional external isolated sandbox:
# supabase secrets set SANDBOX_API_URL=https://your-sandbox-host.example.com
# supabase secrets set SANDBOX_API_KEY=your_strong_sandbox_secret
```

Deploy all 6 Edge Functions:

```bash
supabase functions deploy analyze-paper
supabase functions deploy generate-blueprint
supabase functions deploy recommend-resources
supabase functions deploy generate-code
supabase functions deploy explain-code
supabase functions deploy execute-experiment
```

### 4. Configure & Run Frontend

```bash
cd frontend
cp .env.example .env
# Edit frontend/.env and fill in VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY
npm install
npm run dev
```

### 5. Deploy Frontend to Vercel or Netlify
SPA routing rewrites are pre-configured in `frontend/vercel.json` and `frontend/public/_redirects` so deep links (such as `/app/dashboard` and `/auth?mode=update-password`) work on reload.

- **Vercel:** Import the repository with Root Directory set to `frontend`, Build Command `npm run build`, Output Directory `dist`, and add `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` in Project Environment Variables.
- **Netlify:** Set Base directory to `frontend`, Build command `npm run build`, Publish directory `frontend/dist`, and add `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` in Site Environment Variables.
- **Supabase Auth Redirects:** In Supabase Dashboard -> Authentication -> URL Configuration, add your deployed Vercel/Netlify URL (`https://your-app.vercel.app/**`) to **Redirect URLs**.

### 6. Build & Run Automated Tests

```bash
cd frontend
npm test
npm run build
```

And for the isolated sandbox runner:

```bash
cd sandbox-runner
python -m pytest -v
```

### 7. Optional: Run Isolated Sandbox Container (`sandbox-runner/`)
See [`sandbox-runner/README.md`](sandbox-runner/README.md) for Docker build & hardened execution flags (`--network none`, `--cpus=1.0`, `--memory=512m`, `--read-only`).
