# Incident Retrieval Schema Repair — LU 2.11 Assignment

## What This Is

The platform is shipping its first AI feature: surface the **three most similar past incidents** when a responder opens a new one. A colleague drafted the **retrieval schema** — the `incident_chunks` table that holds embedded incident text for vector similarity search — in `migrations/001_retrieval_schema.sql`. It has **intentional errors**: the embedding is stored as plain `TEXT` instead of a `vector`, the foreign key is the wrong type with no `ON DELETE`, the filter-metadata columns are missing, there are no indexes, and a rejected shape (a single embedding column on `incidents`) made it in.

Your job: **fix the retrieval schema so it correctly implements the retrieval spec** in the assignment question. The spec is the source of truth.

When you're done, all **20 pgTAP tests** must pass.

---

## Prerequisites

| Tool | Why | Check |
|------|-----|-------|
| PostgreSQL (v14+) | Runs the test database | `psql --version` |
| **pgvector** | Provides the `vector` type and HNSW index | `SELECT '[1,2,3]'::vector;` |
| pgTAP | Test framework for schema validation | `pg_prove --version` |
| pg_prove | pgTAP test runner | comes with pgTAP |
| Node.js + npm | Runs the helper scripts | `node --version` |

### Easiest setup — Docker (Postgres + pgvector preinstalled)

```bash
docker run --name incident-pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d pgvector/pgvector:pg16
```

### Installing pgvector manually

**macOS (Homebrew):**
```bash
brew install pgvector
```

**Ubuntu / Debian:**
```bash
sudo apt-get install postgresql-16-pgvector   # match your postgres version
```

### Installing pgTAP

**macOS:** `brew install pgtap` · **Ubuntu:** `sudo apt-get install postgresql-16-pgtap`

Both extensions are enabled automatically by `npm run db:reset` (it runs `CREATE EXTENSION IF NOT EXISTS vector;` and `CREATE EXTENSION IF NOT EXISTS pgtap;`).

---

## Setup

### Step 1: Clone or fork the repo

```bash
git clone <repo-url>
cd incident-retrieval-repair
```

### Step 2: Make sure PostgreSQL (with pgvector) is running

```bash
# Docker
docker start incident-pg
# or local service: brew services start postgresql / sudo service postgresql start
```

### Step 3: Create the test database and apply the schema

```bash
npm run db:reset
```

This drops `incident_retrieval_test`, recreates it, enables the `vector` and `pgtap` extensions, and applies `migrations/001_retrieval_schema.sql`.

> ⚠️ "role does not exist" → add `-U postgres` to each `psql` command in the `db:reset` script in `package.json`.

### Step 4: Run the tests

```bash
npm test
```

The first run **fails most tests** — the schema is broken. Fix it until all 20 pass.

---

## What the Tests Check

| Category | Tests | What they check |
|----------|-------|-----------------|
| Table existence | 2 | `incidents` and `incident_chunks` exist |
| Column types | 7 | `id`/`incident_id`/`team_id` are BIGINT, `embedding` is `vector(1536)` (not TEXT), `created_at` is TIMESTAMPTZ |
| NOT NULL | 6 | `incident_id`, `chunk_text`, `embedding`, `team_id`, `severity`, `created_at` are required |
| Foreign key | 1 | `incident_chunks.incident_id` references `incidents.id` |
| Indexes | 2 | HNSW ANN index on `embedding`; B-tree index on `incident_id` |
| Rejected shape | 1 | `incidents.embedding` column is gone |
| ON DELETE CASCADE | 1 | deleting an incident removes its chunks |

---

## The File to Fix

**Only edit this file:**

```
migrations/001_retrieval_schema.sql
```

Every broken line has a `-- wrong:` or `-- MISSING:` comment. Read each one.

---

## Hints

Work top to bottom.

**Hint 1 — Enable pgvector first.**
Add `CREATE EXTENSION IF NOT EXISTS vector;` at the very top. Without it, `vector(1536)` will error with `type "vector" does not exist`.

**Hint 2 — The embedding is a vector, not text.**
`embedding TEXT` → `embedding vector(1536) NOT NULL`. A stringified vector cannot be indexed or compared with the `<=>` cosine-distance operator. The whole point of retrieval storage is the native vector type.

**Hint 3 — The FK type and ON DELETE.**
`incidents.id` is `BIGSERIAL` (a `bigint`), so the FK must be `BIGINT`, not `INTEGER`. A chunk has no meaning without its incident, so the behaviour is `ON DELETE CASCADE`:
`incident_id BIGINT NOT NULL REFERENCES incidents(id) ON DELETE CASCADE`.

**Hint 4 — Add the filter metadata.**
Hybrid retrieval filters by team and severity alongside the similarity ranking. Add `team_id BIGINT NOT NULL` and `severity TEXT NOT NULL` to `incident_chunks`. These are denormalised copies — a deliberate, access-pattern-driven choice (LU16).

**Hint 5 — Timestamps.**
`created_at` should be `TIMESTAMPTZ NOT NULL DEFAULT now()`.

**Hint 6 — Remove the rejected shape.**
Delete the `ALTER TABLE incidents ADD COLUMN embedding TEXT;` line. One embedding per incident cannot represent a long, multi-paragraph description — you need multiple chunks per incident, each its own vector. That is exactly why `incident_chunks` exists (one-to-many from `incidents`).

**Hint 7 — Add both indexes.**
```sql
CREATE INDEX idx_incident_chunks_embedding
  ON incident_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_incident_chunks_incident_id
  ON incident_chunks (incident_id);
```

---

## Workflow

```bash
# 1. Edit the schema
nano migrations/001_retrieval_schema.sql

# 2. Re-apply from scratch
npm run db:reset

# 3. Run the tests
npm test

# 4. Repeat until all 20 pass
```

---

## Common Errors

**`ERROR: type "vector" does not exist`**
→ Add `CREATE EXTENSION IF NOT EXISTS vector;` at the top of the SQL file (and make sure pgvector is installed — use the Docker image if unsure).

**`ERROR: access method "hnsw" does not exist`**
→ Your pgvector version is too old (HNSW needs pgvector ≥ 0.5.0). Update pgvector, or use `ivfflat` only if your environment requires it — but the tests expect the index named `idx_incident_chunks_embedding` to exist.

**`ERROR: foreign key constraint ... cannot be implemented` / type mismatch**
→ The FK column type must match the referenced PK type. `incidents.id` is `bigint`, so `incident_id` must be `BIGINT`, not `INTEGER`.

**`ERROR: column "embedding" of relation "incidents" does not exist`** (in tests)
→ Good — that means you removed the rejected shape. The `hasnt_column` test wants it gone.

**`pg_prove: command not found`** → pgTAP isn't installed or on PATH. See prerequisites.

**`psql: error: connection to server failed`** → PostgreSQL isn't running.

---

## Submission

Once all 20 tests pass:

1. Commit to a branch called `retrieval-schema-repair`:
```bash
git checkout -b retrieval-schema-repair
git add migrations/001_retrieval_schema.sql README.md
git commit -m "fix: repair the incident retrieval schema to match the spec"
```

2. Documented retrieval-model decisions and rejected shape:

## Retrieval-Model Decisions

### Decision 1: Storing Embeddings as Native pgvector `vector(1536)`
Spec said: `embedding vector(1536) NOT NULL` — the embedding vector (pgvector type).
Schema decided: Enabled the `vector` extension (`CREATE EXTENSION IF NOT EXISTS vector;`) and defined `embedding vector(1536) NOT NULL` on `incident_chunks` with an HNSW Approximate Nearest Neighbor (ANN) index using `vector_cosine_ops`.
Reason: Retrieval storage is fundamentally designed for vector similarity search. Storing vectors as generic `TEXT`, `JSON`, or `float8[]` arrays prevents the database from performing native vector operations or using vector distance operators such as cosine distance (`<=>`). The native `vector(1536)` type from `pgvector` enforces strict dimensionality constraints at write time, optimizes contiguous storage layout for dense float arrays, and enables fast sub-linear similarity search via the HNSW index without needing full-table sequential scans or application-level distance math.

### Decision 2: Foreign Key Integrity with `ON DELETE CASCADE`
Spec said: `incident_id BIGINT NOT NULL REFERENCES incidents(id) ON DELETE CASCADE` ("a chunk has no meaning without its incident").
Schema decided: Defined `incident_id BIGINT NOT NULL REFERENCES incidents(id) ON DELETE CASCADE` on `incident_chunks` along with a supporting B-tree index `idx_incident_chunks_incident_id`.
Reason: `incident_chunks` represents derived data produced by chunking and embedding the authoritative incident text. Chunks have no lifecycle or meaning independent of the incident they describe. If an incident is deleted, orphaned chunks left in the vector store would surface "ghost" matches pointing to missing incidents, corrupting AI summaries and breaking foreign key lookups. Enforcing `ON DELETE CASCADE` guarantees referential integrity at the database engine level, ensuring all chunks and their vectors are automatically pruned atomically when an incident is deleted.

### Decision 3: Denormalising Filter Metadata (`team_id` and `severity`) onto Chunks
Spec said: `team_id BIGINT NOT NULL` and `severity TEXT NOT NULL` as denormalised filter metadata on `incident_chunks`.
Schema decided: Denormalised and stored `team_id BIGINT NOT NULL` and `severity TEXT NOT NULL` directly in `incident_chunks`, copied from `incidents`.
Reason: The production query pattern is hybrid retrieval: filtering incidents by relational attributes (e.g. searching only for a specific team's past incidents or matching severity levels such as P1/P2) combined with ANN similarity ranking. If metadata resided solely on `incidents`, vector search would either require expensive relational joins during index traversal or post-filtering after retrieving candidate vector neighbors (which causes over-fetching or under-fetching where valid results are filtered out, returning fewer than the requested top-k items). Denormalising these columns allows pre-filtering and combined query execution directly at the vector store index level.

## Rejected Shape

### Shape: Single embedding column bolted onto `incidents` (`ALTER TABLE incidents ADD COLUMN embedding TEXT;`)
What it was: A single `embedding TEXT` column added directly to the parent `incidents` table.
Why rejected: Incidents contain lengthy, multi-paragraph text across titles, descriptions, timelines, and resolution notes. A 1:1 mapping between an incident and an embedding fails due to token limits and loss of retrieval granularity. Compressing an entire incident into a single vector dilutes specific technical signals and details. Furthermore, storing it as `TEXT` instead of `vector` prevented native vector indexing and distance calculations.
What would break:
1. **Retrieval Granularity & Needle-in-a-Haystack Loss**: Compressing multi-paragraph postmortems into one vector dilutes specific error messages, stack traces, and resolution steps. A responder querying with a specific symptom will get a weak similarity score against a diluted monolithic embedding, causing relevant past incidents to be missed entirely.
2. **Context Length Truncation**: Long incidents exceed embedding model token limits (e.g., 8,192 tokens), silently truncating critical resolution steps or error outputs located further down in the text.
3. **Grounded Generation Failure**: The AI assistant needs concise, relevant text passages to draft accurate summaries. Surfacing an entire monolithic incident forces the LLM to process excessive, noisy context, increasing latency, costs, and hallucinations.


3. Push and open a PR:
```bash
git push origin retrieval-schema-repair
```

4. Paste your `npm test` output (all 20 passing) into the PR description.

---

## What You May Not Do

- Change the retrieval spec (it is the source of truth)
- Add columns not in the spec
- Store the embedding as `TEXT`, `JSON`, or `float8[]` — it must be `vector(1536)`
- Skip `ON DELETE` on the FK — it must be explicit `ON DELETE CASCADE`
- Use `SERIAL`/`INTEGER` for the PK or FK
- Leave the `incidents.embedding` rejected column in place
- Skip either required index

---

## Project Structure

```
incident-retrieval-repair/
├── migrations/
│   └── 001_retrieval_schema.sql   ← the broken schema — only file you edit
├── tests/
│   └── schema.test.sql            ← 20 pgTAP tests — do not edit
├── README.md                      ← this file — update with your decisions
└── package.json                   ← npm scripts — do not edit
```
