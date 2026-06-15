# ai_sql

AI inference SQL functions for Supabase. Call LLMs directly from SQL — completions, classification, sentiment, extraction, translation, redaction, and embeddings. Self-deploys its own edge function from a single SQL call; no CLI required.

```sql
select ai_sentiment(body) from posts;
-- → 'positive'

select category, ai_summarize_agg(description)
from products
group by category;
-- → one LLM call per group
```

---

## How it works

```
Your SQL query
    └─► public.ai_sentiment(text)          ← SECURITY DEFINER, authenticated only
            └─► public._ai_call(jsonb)     ← reads config from ai._config
                    └─► POST /functions/v1/ai   ← edge function on your project
                              └─► OpenAI-compatible API  (or Supabase built-in)
```

`ai.deploy()` bootstraps the whole pipeline from Postgres in one call:
1. Reads your personal access token from Vault (encrypted at rest)
2. Fetches your project's anon key from the Management API
3. Deploys the self-contained edge function to `/functions/v1/ai`
4. Saves `project_ref` + `anon_key` to `ai._config`

The edge function source is embedded directly in the SQL file (zero imports, raw fetch) so it can be deployed via the Management API without the CLI.

---

## Prerequisites

- A Supabase project (any plan)
- A [personal access token](https://supabase.com/dashboard/account/tokens) (`sbp_…`)
- An API key for an OpenAI-compatible provider, **or** use the free Supabase built-in inference

---

## Bootstrap

### 1. Install the extension

From the SQL editor or `psql`:

```sql
select dbdev.install('supabase/ai-sql');
create extension "supabase/ai-sql";
```

> **Note:** [database.dev](https://database.dev) must be enabled on your project. If it isn't, follow the [dbdev setup guide](https://database.dev/installer).

### 2. Store your personal access token in Vault

```sql
select vault.create_secret('sbp_xxxxxxxxxxxx', 'ai_access_token');
```

This is the **only** secret that touches the database. It is AES-256 encrypted at rest and only ever decrypted inside `ai.deploy()` — it never appears in query logs or `pg_stat_activity`.

Get your token from **[supabase.com/dashboard/account/tokens](https://supabase.com/dashboard/account/tokens)**.

### 3. Set your API key

Use the Supabase CLI — **never pass the key as a SQL parameter**, as it would appear in `pg_stat_activity` logs:

```bash
supabase secrets set OPENAI_API_KEY=sk-...
```

Or via the dashboard: **Settings → Edge Functions → Secrets**.

> **Using a different provider?** Any OpenAI-compatible API works. Also set `OPENAI_BASE_URL` and use the same `OPENAI_API_KEY` variable for the credential:
>
> ```bash
> # Groq
> supabase secrets set OPENAI_BASE_URL=https://api.groq.com/openai/v1
> supabase secrets set OPENAI_API_KEY=gsk_...
>
> # Together AI
> supabase secrets set OPENAI_BASE_URL=https://api.together.xyz/v1
> supabase secrets set OPENAI_API_KEY=...
>
> # Azure OpenAI
> supabase secrets set OPENAI_BASE_URL=https://<resource>.openai.azure.com/openai/deployments/<deployment>
> supabase secrets set OPENAI_API_KEY=...
>
> # Local Ollama (self-hosted)
> supabase secrets set OPENAI_BASE_URL=http://localhost:11434/v1
> supabase secrets set OPENAI_API_KEY=ollama
> ```
>
> Omit `OPENAI_BASE_URL` to default to `https://api.openai.com/v1`.

### 4. Deploy

```sql
select ai.deploy('your-project-ref');
-- → { "status": "ok", "deployed": "functions/v1/ai", "project_ref": "..." }
```

This is the only time you need your project ref. Everything else is automatic.

### 5. Verify

```sql
select ai.status();
```

```json
{
  "version": "1.0.0",
  "configured": true,
  "project_ref": "abcdefghijklmnopqrst",
  "vault_token": true
}
```

### 6. Run your first query

```sql
select ai_complete2('Write a one-sentence tagline for a Postgres extension.');
```

---

## Functions

### Scalar — one LLM call per row

| Function | Returns | Description |
|---|---|---|
| `ai_complete2(input, system?, model?, provider?)` | `text` | Free-form completion |
| `ai_classify(input, categories?, model?, provider?)` | `text` | One of the supplied labels |
| `ai_sentiment(input, model?, provider?)` | `text` | `positive`, `negative`, or `neutral` |
| `ai_extract(input, schema_hint?, model?, provider?)` | `jsonb` | Structured fields as JSON |
| `ai_translate(input, target_language?, model?, provider?)` | `text` | Translated text |
| `ai_redact(input, entity_types?, model?, provider?)` | `text` | PII replaced with `[REDACTED]` |
| `ai_embed(input, model?, provider?)` | `real[]` | Embedding vector (1536 dims, OpenAI only) |

### Aggregates — one LLM call per `GROUP BY` group

| Aggregate | Returns | Description |
|---|---|---|
| `ai_summarize_agg(text)` | `text` | Summarize all rows in a group |
| `ai_extract_agg(text)` | `jsonb` | Extract entities across all rows in a group |

All functions default to `provider => 'openai'`. Pass `provider => 'supabase'` for free built-in inference (see [Providers](#providers)).

---

## Usage

```sql
-- Free-form completion
select ai_complete2('Explain JSONB indexes in one sentence.');

-- Custom system prompt
select ai_complete2(
  'What is the capital of France?',
  system_text => 'You are a geography tutor. Be brief.'
);

-- Sentiment on every row
select id, body, ai_sentiment(body) as mood
from posts;

-- Classify with custom labels
select ai_classify(
  'The login button is broken',
  array['bug', 'feature-request', 'question']
);
-- → 'bug'

-- Extract structured data
select ai_extract(
  'Jesse Vent, jesse@example.com, Supabase, joined 2024-01-15',
  schema_hint => 'name, email, company, joined_date'
);
-- → { "name": "Jesse Vent", "email": "jesse@example.com", "company": "Supabase", "joined_date": "2024-01-15" }

-- Translate
select ai_translate('Good morning', target_language => 'Japanese');
-- → おはようございます

-- Redact PII
select ai_redact('Call me at 555-123-4567 or email bob@example.com');
-- → Call me at [REDACTED] or email [REDACTED]

-- Embeddings (cast to vector if pgvector is installed)
select ai_embed('hello world')::vector;

-- GROUP BY aggregate — one LLM call per category
select category, ai_summarize_agg(description)
from products
group by category;

-- Extract entities grouped by user
select user_id, ai_extract_agg(message)
from support_tickets
group by user_id;
```

---

## Providers

### `openai` (default)

Routes to `OPENAI_BASE_URL` (default: `https://api.openai.com/v1`) with `OPENAI_API_KEY`. Works with any OpenAI-compatible API — swap the URL and key to use Groq, Together AI, Azure OpenAI, Ollama, and others without changing your SQL.

Default model: `gpt-5.4-mini`. Default embedding model: `text-embedding-3-small`.

```sql
-- Default provider
select ai_complete2('Write a haiku about Postgres.');

-- Override model per call
select ai_complete2(
  'Write a haiku about Postgres.',
  model_name => 'gpt-5.4-pro'
);
```

### `supabase` (free, no key required)

Uses [Supabase built-in inference](https://supabase.com/docs/guides/ai/quickstarts/generate-text-using-ai-models) (Mistral) at no extra cost. No API key needed — runs entirely within the Edge Function runtime.

```sql
select ai_complete2('Hello', provider => 'supabase');
select ai_sentiment('Great product!', provider => 'supabase');
select ai_classify('Bug in login', array['bug', 'feature'], provider => 'supabase');
```

> `ai_embed` requires `provider => 'openai'` — built-in inference does not expose an embeddings endpoint.

---

## Aggregate safety caps

Aggregates accumulate row values in memory before a single LLM call. Two GUCs cap input size to prevent runaway token costs:

```sql
set ai_agg.max_items = 50;    -- max rows accumulated per group (default: 100)
set ai_agg.max_chars = 8000;  -- max joined characters sent per group (default: 12000)
```

Set these at the session level before expensive aggregate queries.

---

## Security model

| Credential | Where it lives | Who can read it |
|---|---|---|
| Personal access token (`sbp_…`) | Vault (AES-256 encrypted) | `postgres` / service role only, inside `ai.deploy()` |
| OpenAI API key | Edge Function secrets (set via CLI) | Edge runtime only — never stored in DB, never a SQL parameter |
| Anon key | `ai._config` table | Readable by service role — same key shipped to browsers |

**Function access**: All `ai_*` functions are `REVOKE`d from `public` and `GRANT`ed to `authenticated` only. Unauthenticated callers cannot trigger LLM calls.

**Prompt injection**: Caller-supplied values interpolated into system prompts (`categories`, `entity_types`, `target_language`, `schema_hint`) are sanitized — control characters stripped, length-capped — before being embedded in the prompt. The `input` field is passed as the user message, which is standard LLM input separation.

---

## Re-deploying

To update the edge function after an extension upgrade:

```sql
select ai.deploy('your-project-ref');
```

The edge function source embedded in the SQL file is always in sync with the installed extension version.

---

## Troubleshooting

**`No access token found in Vault`**
Run step 2 — `vault.create_secret` with the name `ai_access_token`.

**`OPENAI_API_KEY secret is not set`**
The edge function is deployed but the secret isn't. Set it via the CLI or dashboard, then retry your query (no re-deploy needed).

**`Failed to fetch API keys (HTTP 401)`**
Your personal access token in Vault is expired or invalid. Create a new one and update Vault:
```sql
select vault.update_secret('sbp_new...', 'ai_access_token');
```

**`ai_sql extension not configured`**
`ai.deploy()` hasn't been run yet, or the config was cleared. Re-run it.

**`embed action requires provider=openai`**
Switch to `provider => 'openai'` for `ai_embed` — the Supabase built-in provider doesn't expose an embeddings endpoint.
