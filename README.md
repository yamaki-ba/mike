# Mike

Mike is an open-source legal document assistant. Upload contracts and other legal documents into projects, chat with them using Claude, Gemini, or OpenAI models, run tabular reviews across many documents at once, and reuse prompts as workflows.

Website: [mikeoss.com](https://mikeoss.com)

> **Before using real documents**, read [`docs/safe-local-testing.md`](docs/safe-local-testing.md). Test with disposable infrastructure and synthetic documents until you have reviewed your deployment's storage, logging, and model-provider behavior.

## Features

- **Assistant chat** – ask questions about uploaded documents, with citations back to the source text. Chats can be standalone or scoped to a project.
- **Projects** – organize documents into folders, share projects with other users, and keep project-level chat history.
- **Document handling** – upload PDF, DOC, and DOCX files, view them in the browser, track document versions, and review suggested edits as DOCX tracked changes.
- **Tabular reviews** – extract the same set of fields (columns) from many documents into a table, with per-cell citations and Excel export.
- **Workflows** – built-in and custom prompt templates that can be shared and reused in chats and tabular reviews.
- **Bring your own keys** – model provider keys can be set once for the whole instance or per user (stored encrypted).

## Architecture

| Part | Stack |
| --- | --- |
| `frontend/` | Next.js 16 / React 19, Tailwind CSS 4; deployable to Cloudflare via OpenNext |
| `backend/` | Express 4 + TypeScript API: chat, document processing, LLM provider integrations, signed downloads |
| Auth & database | Supabase Auth + Postgres (`backend/schema.sql`) |
| File storage | Cloudflare R2 or any S3-compatible bucket (MinIO, AWS S3, …) |
| Document conversion | LibreOffice (DOC/DOCX → PDF) |

### Supported models

| Provider | Chat models | Tabular review | Lightweight tasks (titles etc.) |
| --- | --- | --- | --- |
| Anthropic | `claude-opus-4-7`, `claude-sonnet-4-6` | `claude-sonnet-4-6` | `claude-haiku-4-5` |
| Google Gemini | `gemini-3.1-pro-preview`, `gemini-3-flash-preview` | `gemini-3-flash-preview` | `gemini-3.1-flash-lite-preview` |
| OpenAI | `gpt-5.5`, `gpt-5.4-mini` | `gpt-5.4-mini` | `gpt-5.4-nano` |

Gemini models are the defaults. The model list lives in `backend/src/lib/llm/models.ts`.

## Prerequisites

- Node.js 20 or newer
- npm
- git
- A Supabase project
- A Cloudflare R2 bucket, MinIO bucket, or another S3-compatible bucket
- At least one model provider API key: Anthropic, Google Gemini, or OpenAI
- LibreOffice installed locally if you need DOC/DOCX to PDF conversion

## Database Setup

For a new Supabase database, open the Supabase SQL editor and run the full contents of [`backend/schema.sql`](backend/schema.sql).

Do not re-run the full schema file over an existing production database. When upgrading an existing deployment, compare your database with the current `backend/schema.sql` and apply only the changes you need.

## Environment

Create `backend/.env`:

```bash
PORT=3001
FRONTEND_URL=http://localhost:3000

# Required secrets – generate each with: openssl rand -hex 32
DOWNLOAD_SIGNING_SECRET=replace-with-a-random-32-byte-hex-string
USER_API_KEYS_ENCRYPTION_SECRET=replace-with-a-long-random-secret

SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SECRET_KEY=your-supabase-service-role-key

R2_ENDPOINT_URL=https://your-account-id.r2.cloudflarestorage.com
R2_ACCESS_KEY_ID=your-r2-access-key
R2_SECRET_ACCESS_KEY=your-r2-secret-key
R2_BUCKET_NAME=mike

# Optional instance-wide model provider keys
GEMINI_API_KEY=your-gemini-key
ANTHROPIC_API_KEY=your-anthropic-key
OPENAI_API_KEY=your-openai-key
```

Create `frontend/.env.local`:

```bash
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY=your-supabase-anon-key
NEXT_PUBLIC_API_BASE_URL=http://localhost:3001
```

### Variable reference

| Variable | Required | Purpose |
| --- | --- | --- |
| `PORT` | No (default `3001`) | Backend port |
| `FRONTEND_URL` | No (default `http://localhost:3000`) | Allowed CORS origin for the frontend |
| `DOWNLOAD_SIGNING_SECRET` | **Yes** | HMAC secret for document download links. Downloads fail without it |
| `USER_API_KEYS_ENCRYPTION_SECRET` | **Yes**, for per-user keys | Encrypts API keys users save in **Account > Models & API Keys** |
| `SUPABASE_URL`, `SUPABASE_SECRET_KEY` | **Yes** | Supabase project URL and service role key (backend only) |
| `R2_ENDPOINT_URL`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME` | **Yes** | S3-compatible object storage for uploaded documents |
| `ANTHROPIC_API_KEY` (or `CLAUDE_API_KEY`), `GEMINI_API_KEY`, `OPENAI_API_KEY` | No | Instance-wide provider keys |
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY` | **Yes** | Supabase URL and anon/public key for the browser |
| `NEXT_PUBLIC_API_BASE_URL` | **Yes** | URL of the backend API |

Supabase values come from the project dashboard. If your Supabase project shows multiple key formats, use the legacy JWT-style anon and service role keys expected by the Supabase client libraries. Never put the service role key or provider keys in the frontend env file; only `NEXT_PUBLIC_` variables are exposed to the browser.

Provider keys can be configured in `backend/.env` for the whole instance, or per user in **Account > Models & API Keys**. If a provider key is present in `backend/.env`, that provider is available to every user and the matching browser API key field is read-only.

## Install

```bash
npm install --prefix backend
npm install --prefix frontend
```

## Run Locally

Start the backend (http://localhost:3001):

```bash
npm run dev --prefix backend
```

Start the frontend in another terminal:

```bash
npm run dev --prefix frontend
```

Open http://localhost:3000.

## First Run

1. Sign up in the app.
2. If you did not set provider keys in `backend/.env`, open **Account > Models & API Keys** and add an Anthropic, Gemini, or OpenAI API key.
3. Create or open a project, upload documents, and start chatting, or create a tabular review.

## Deployment

- **Backend** – build with `npm run build --prefix backend` and start with `npm run start --prefix backend`. `backend/nixpacks.toml` installs LibreOffice for Nixpacks-based hosts (e.g. Railway). On other hosts, make sure LibreOffice is on the process `PATH`.
- **Frontend** – deploy anywhere Next.js runs, or to Cloudflare Workers with OpenNext:

  ```bash
  npm run preview --prefix frontend   # build and preview locally on the Workers runtime
  npm run deploy --prefix frontend    # build and deploy to Cloudflare
  ```

Set `FRONTEND_URL` on the backend to the deployed frontend origin, and `NEXT_PUBLIC_API_BASE_URL` on the frontend to the deployed backend URL.

## Troubleshooting

**Sign-up confirmation email never arrives.** Confirmation emails are sent by Supabase Auth, not by Mike. For local development, the simplest fix is to disable email confirmation in **Supabase > Authentication > Providers > Email**. For production, configure custom SMTP in Supabase; the built-in mailer is heavily rate-limited and may be restricted on newer projects.

**The model picker shows a missing-key warning.** Add a key for that provider in **Account > Models & API Keys**, or configure the provider key in `backend/.env` and restart the backend.

**Saving an API key in the browser fails.** Set `USER_API_KEYS_ENCRYPTION_SECRET` in `backend/.env` and restart the backend.

**Document downloads fail with a `DOWNLOAD_SIGNING_SECRET` error.** Set `DOWNLOAD_SIGNING_SECRET` (e.g. `openssl rand -hex 32`) in `backend/.env` and restart the backend.

**DOC or DOCX conversion fails.** Install LibreOffice and restart the backend so the conversion commands are available on the process path.

## Useful Checks

```bash
npm run build --prefix backend
npm run build --prefix frontend
npm run lint --prefix frontend
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Report security vulnerabilities privately through [GitHub's private vulnerability reporting](https://github.com/willchen96/mike/security/advisories/new), not as public issues.

## License

[AGPL-3.0-only](LICENSE)
