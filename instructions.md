# Project Overview
Build **YouTuber Research → Script + Title + Tags Automator**: a Next.js (App Router) web app for creators to paste a topic or competitor video link, fetch YouTube metadata, run keyword analysis, and generate:
- **3 video ideas**
- **1 full 5–8 min script** (~900–1,200 words)
- **5 SEO‑optimized title variations**
- **1 description**
- **5 tags**
- **Thumbnail text suggestions**

**Tech**: Next.js + Tailwind (frontend), Next.js Route Handlers on Vercel (backend), Supabase (Postgres/Auth/Storage) with Prisma, YouTube Data API v3, multi‑provider LLMs (OpenAI default; switchable to Claude, Gemini, DeepSeek, Perplexity, Qwen), Dodo Payments for checkout & recurring.

---

## Core Functionalities

### 1) Project Setup (Next.js App Router + Tailwind + Prisma + Supabase)
**Description**
- Scaffold a Next.js **App Router** project and configure Tailwind CSS.
- Add Prisma ORM and connect to Supabase Postgres.
- Prepare environment variables and Vercel project setup.

**Implementation**
1. **Scaffold**: Create Next.js (TypeScript, App Router).  
   _Docs_: Next.js App Router (see Docs §A). :contentReference[oaicite:0]{index=0}
2. **Styling**: Install & configure Tailwind (PostCSS plugin, `tailwind.config` content paths for `app/**/*.{ts,tsx}` etc.).  
   _Docs_: Tailwind + Next.js (see Docs §B). :contentReference[oaicite:1]{index=1}
3. **Supabase Client**: Install `@supabase/supabase-js`. Add `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
   _Docs_: supabase-js intro (see Docs §H). :contentReference[oaicite:2]{index=2}
4. **Prisma**: Install `prisma @prisma/client`. Create `schema.prisma` and configure datasource with Supabase connection URL; run `prisma generate`.
   _Docs_: Prisma Schema Overview (see Docs §C). :contentReference[oaicite:3]{index=3}
5. **Vercel**: Create project, set env vars, connect Git for CI/CD.  
   _Docs_: Next.js on Vercel (see Docs §D). :contentReference[oaicite:4]{index=4}

---

### 2) Auth: Google OAuth + Email (Magic Link/OTP) via Supabase
**Description**
- Enable **Google sign‑in** and **email code/magic‑link** auth.
- Request **YouTube read scopes** when user connects channel.
- Persist session via `@supabase/auth-helpers-nextjs`.

**Implementation**
1. **Auth UI**: Add sign‑in page with buttons: “Continue with Google” and “Sign in with Email”.  
   - Use `supabase.auth.signInWithOAuth({ provider: 'google', options: { scopes: 'https://www.googleapis.com/auth/youtube.readonly openid email profile' }})`.  
   - Email flow: `supabase.auth.signInWithOtp({ email })` (magic link / OTP).  
   _Docs_: Supabase Auth, Google Login, OAuth method, provider tokens (Docs §H‑K). :contentReference[oaicite:5]{index=5}
2. **Callback**: Handle OAuth redirect (Next.js route in `/app/auth/callback/route.ts`), capture `session.provider_token` for Google (optional secure server-side exchange).  
   _Docs_: Provider tokens guidance (Docs §K). :contentReference[oaicite:6]{index=6}
3. **RLS**: Ensure Supabase RLS policies allow user‑scoped access to app tables (Projects/Outputs/Credits/Jobs).
4. **Identity Linking** (optional): Allow linking email + Google under one user.  
   _Docs_: Identity Linking (Docs §L). :contentReference[oaicite:7]{index=7}

---

### 3) Billing & Credits (Dodo Payments) + Webhooks
**Description**
- Plans: $15–$49/mo with credits; per‑script $1–$5 one‑off.  
- Store credits in `credits` table; decrement on generation; log usage to `outputs.cost_credits`.

**Implementation**
1. **Checkout**: Create pricing page; invoke Dodo Checkout; handle success/cancel routes.
2. **Webhooks**: Expose `/api/webhooks/dodo` (Route Handler) to:
   - On **subscription created/renewed**: increment `credits_remaining`.
   - On **one‑off purchase**: add credits or mark entitlement.
   - On **failed payment/cancel**: flag subscription and freeze new generations when credits depleted.
3. **Guardrails**: Middleware to block generation if `credits_remaining <= 0`.

_Docs_: Dodo Payments API (Docs §M). :contentReference[oaicite:8]{index=8}

---

### 4) Data Model with Prisma (Supabase Postgres)
**Description**
Implement the following tables (aligning to your schema) via Prisma models:

- **projects**  
  - `id` (uuid, pk), `user_id` (uuid, fk→auth.users.id), `name` (string, nullable),  
  - `input_text` (text), `status` (enum: draft|processed),  
  - `created_at`, `updated_at` (timestamps).
- **outputs**  
  - `id` (uuid), `project_id` (uuid fk→projects),  
  - `kind` (enum: ideas|script|metadata|seo),  
  - `content` (jsonb), `cost_credits` (int),  
  - `created_at`.
- **credits**  
  - `id` (uuid), `user_id` (uuid fk), `credits_remaining` (int), `updated_at`.
- **jobs**  
  - `id` (uuid), `project_id` (uuid fk), `type` (enum: fetch|analyze|generate),  
  - `status` (enum: queued|running|done|error), `meta` (jsonb), `created_at`.

**Implementation**
1. Create Prisma models and enums; run `prisma migrate dev`.  
   _Docs_: Prisma schema & types (Docs §C, §C2, §C3). :contentReference[oaicite:9]{index=9}
2. Add composite indexes: `(user_id, created_at)` on projects; `(project_id, kind)` on outputs.
3. RLS: Postgres policies to `user_id = auth.uid()` for select/insert/update.

---

### 5) YouTube Connection & Metadata Ingestion
**Description**
- **Connect Channel** (optional): with OAuth `youtube.readonly` scope; fetch uploads playlist; hydrate past videos.  
- **Competitor Links**: Parse pasted URLs, extract video IDs, fetch `videos.list`.  
- **Topic Input**: Use `search.list` to find top N relevant videos as context for keyword analysis.

**Implementation**
1. **OAuth scopes** for channel connect (Google): `https://www.googleapis.com/auth/youtube.readonly`. Store & encrypt refresh token if returned (see provider token notes). :contentReference[oaicite:10]{index=10}
2. **Fetch user uploads**:
   - `channels.list?part=contentDetails&mine=true` → get `relatedPlaylists.uploads`.
   - `playlistItems.list?part=snippet,contentDetails&playlistId=...` (page through).  
   _Docs_: YouTube implementation guide & playlistItems (Docs §E3, §E5). :contentReference[oaicite:11]{index=11}
3. **Fetch competitor metadata**:
   - Extract `v` parameter / youtu.be ID.
   - Call `videos.list?part=snippet,contentDetails,statistics&id=...` (quota 1).  
   _Docs_: videos.list (Docs §E1, §E2). :contentReference[oaicite:12]{index=12}
4. **Topic search**:
   - `search.list?part=snippet&type=video&q=...` (quota 100) to collect comparable videos.  
   _Docs_: search.list (Docs §E4). :contentReference[oaicite:13]{index=13}
5. **Normalization**:
   - Persist minimal fields to `outputs(kind='metadata')`: title, description, channel, tags (if available), duration (ISO 8601), view/like counts, publish date, categoryId.
   - Optional: map `videoCategories.list` for human-readable category. :contentReference[oaicite:14]{index=14}

---

### 6) Embeddings Store (Past Data) + Keyword Analysis
**Description**
- Store embeddings of user’s past titles/descriptions to bias new generations.
- Use **pgvector** in Supabase or **Pinecone** (toggle via config).

**Implementation**
1. **pgvector path** (default):
   - Enable `pgvector` in Supabase; add `vector` column to a new table `video_chunks` (`project_id/user_id`, `text`, `embedding vector(1536)` etc.).  
   - Upsert embeddings for user uploads / collected competitor snippets.  
   _Docs_: Supabase pgvector & AI guides (Docs §I, §I2, §I4). :contentReference[oaicite:15]{index=15}
2. **Pinecone path** (optional):
   - Create index, namespace per user, upsert vectors, metadata includes `videoId`, `kind`.  
   _Docs_: Pinecone (Docs §J). :contentReference[oaicite:16]{index=16}
3. **Keyword extraction**:
   - Prompt LLM to produce: `{primary_keywords[], secondary_keywords[], entities[], questions[], angle}` from pooled metadata.
   - Use embeddings similarity to retrieve top K user‑relevant snippets to include in the prompt.

---

### 7) Provider‑Agnostic Generation Router (Model Switch + Temperature)
**Description**
- A single interface to call multiple providers (OpenAI default).  
- UI controls: **Model dropdown**, **Temperature slider** (0.0–1.5).  
- Persist provider, model, temperature per output.

**Implementation**
1. **Provider registry**: `providers/{openai, anthropic, gemini, deepseek, perplexity, qwen}.ts` each exporting `generate(payload, settings)` and `embed(text)`.
2. **Environment**: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `DEEPSEEK_API_KEY`, `PERPLEXITY_API_KEY`, `QWEN_API_KEY`. Fail gracefully if unset.
3. **Default models** (configure constants; allow user override):
   - OpenAI: `gpt-4o-mini` (gen), `text-embedding-3-large` (embed).  
     _Docs_: OpenAI Responses & Embeddings (Docs §F). :contentReference[oaicite:17]{index=17}
   - Anthropic: latest Claude models (text). _Docs §N_. :contentReference[oaicite:18]{index=18}
   - Google Gemini: latest text model. _Docs §O_. :contentReference[oaicite:19]{index=19}
   - DeepSeek: text chat. _Docs §P_. :contentReference[oaicite:20]{index=20}
   - Perplexity API: text models. _Docs §Q_. :contentReference[oaicite:21]{index=21}
   - Qwen: text chat/completions. _Docs §R_. :contentReference[oaicite:22]{index=22}
4. **Payload schema (generation)**:
   - **Inputs**: `topic | competitorVideoIds[] | uploadRefs[]`, retrieved `metadata[]`, `retrievedSnippets[]`, `stylePrefs?`.
   - **Controls**: `model`, `temperature`, `max_tokens`, `language`.
5. **Outputs** (stored in `outputs.content` JSON):
   ```json
   {
     "ideas": [{ "title": "...", "angle": "...", "why": "..." }, ...3],
     "script": { "title": "...", "sections": [{ "heading": "...", "body": "..." }], "cta": "..." },
     "seo": { "titles": ["x 5"], "description": "...", "tags": ["x 5"], "thumbnail_text": ["x 2-3"] }
   }
