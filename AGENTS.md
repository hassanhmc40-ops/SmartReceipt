# AGENTS.md — Expense Assistant
> AI-Assisted Development Log & Agent Configuration  
> Project: **Expense Assistant — Intelligent Receipt Extraction with Laravel & AI**  
> Author: Hassan  
> Start: Monday 08/06/2026 — Deadline: Friday 12/06/2026 at 14:30  
> Mode: Individual | Duration: 5 days

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Summary](#2-architecture-summary)
3. [Domain Model](#3-domain-model)
4. [Tech Stack](#4-tech-stack)
5. [AI Agent Configuration](#5-ai-agent-configuration)
6. [Feature Breakdown & Agent Workflow](#6-feature-breakdown--agent-workflow)
7. [JSON Contract](#7-json-contract)
8. [Queue & Job Architecture](#8-queue--job-architecture)
9. [Structured Output Strategy](#9-structured-output-strategy)
10. [OpenSpec Discipline](#10-openspec-discipline)
11. [Commit Convention](#11-commit-convention)
12. [Day-by-Day Plan](#12-day-by-day-plan)
13. [Performance Constraints](#13-performance-constraints)
14. [What the Agent Generated vs. What Was Modified](#14-what-the-agent-generated-vs-what-was-modified)
15. [Architectural Decisions Log](#15-architectural-decisions-log)

---

## 1. Project Overview

**Expense Assistant** is a Laravel 11 web application that solves a real-world problem for small shopkeepers like Brahim — a local grocery owner drowning in poorly formatted supplier receipts written in Darija, abbreviations, and handwritten shorthand.

### The Problem

Brahim accumulates dozens of supplier receipts every month. These are:
- Written in Darija (Moroccan Arabic dialect) mixed with French abbreviations
- Poorly formatted, scribbled, with non-standard units or item names
- Impossible to categorise manually at scale
- Never entered into any tracking system → he has no visibility on spending

### The Solution

Expense Assistant gives Brahim (and any user) the ability to:
1. **Paste the raw text** of a receipt into a form
2. **Let the AI extract** items automatically: description, quantity, unit price, and category
3. **View clean, structured expenses** saved to the database — categorised, typed, queryable
4. **Track all receipts** over time with status indicators (Pending / Processed / Failed)
5. **Filter expenses** by category (Food / Drinks / Hygiene / Maintenance / Other)

### Why This Is Non-Trivial

The AI call is slow (Groq API latency). A naive implementation would freeze the page. The extraction must produce **guaranteed structured JSON** — a `json_decode` crash is unacceptable in production. The data must be validated, typed (Eloquent casts), and stored reliably. This requires:
- **Asynchronous queue processing** so the UI never blocks
- **Structured output from laravel/ai SDK** to enforce the JSON contract
- **Form validation before the AI call** to avoid wasting API tokens on garbage input
- **Status tracking** so the user always knows what's happening

---

## 2. Architecture Summary

```
┌──────────────────────────────────────────────────────────────┐
│                        Browser (Blade + Alpine.js)           │
│  ┌────────────┐   POST /recus   ┌─────────────────────────┐  │
│  │  Receipt   │ ─────────────►  │  RecuController         │  │
│  │  Form      │                 │  @store                 │  │
│  └────────────┘                 └──────────┬──────────────┘  │
│                                            │                  │
│                               StoreRecuRequest (validation)  │
│                                            │                  │
│                               Recu::create(status=pending)   │
│                                            │                  │
│                        ExtractExpensesFromReceipt::dispatch() │
│                                            │                  │
│                         ◄── 200 OK (immediate redirect) ──── │
└──────────────────────────────────────────────────────────────┘
                                             │
                              ┌──────────────▼──────────────┐
                              │      Queue Worker            │
                              │  php artisan queue:work     │
                              └──────────┬──────────────────┘
                                         │
                              ┌──────────▼──────────────┐
                              │  ExtractExpensesFrom     │
                              │  Receipt (Job)           │
                              │                          │
                              │  laravel/ai SDK          │
                              │  → Groq API              │
                              │  → Structured Output     │
                              │  → JSON Contract         │
                              └──────────┬───────────────┘
                                         │
                        ┌────────────────▼──────────────────┐
                        │          MySQL Database            │
                        │                                    │
                        │  recus      ──hasMany──  depenses  │
                        │  (status: pending/processed/failed)│
                        └────────────────────────────────────┘
```

---

## 3. Domain Model

### MCD (Modèle Conceptuel de Données)

```
┌───────────────────┐          ┌───────────────────────┐
│      USER         │          │        RECU            │
│─────────────────  │          │──────────────────────  │
│ id                │  1    *  │ id                     │
│ name              │──────────│ user_id (FK)           │
│ email             │          │ texte_brut (TEXT)      │
│ password          │          │ statut (enum)          │
│ timestamps        │          │ payload_brut (json)    │
└───────────────────┘          │ timestamps             │
                               └──────────┬─────────────┘
                                          │
                                     1        *
                                          │
                               ┌──────────▼─────────────┐
                               │       DEPENSE           │
                               │──────────────────────── │
                               │ id                      │
                               │ recu_id (FK)            │
                               │ libelle (string)        │
                               │ quantite (integer)      │
                               │ prix_unitaire (decimal) │
                               │ categorie (enum)        │
                               │ timestamps              │
                               └─────────────────────────┘
```

### MLD (Modèle Logique de Données)

```
users(
  id              INT UNSIGNED PK AUTO_INCREMENT,
  name            VARCHAR(255) NOT NULL,
  email           VARCHAR(255) UNIQUE NOT NULL,
  password        VARCHAR(255) NOT NULL,
  remember_token  VARCHAR(100),
  created_at      TIMESTAMP,
  updated_at      TIMESTAMP
)

recus(
  id              INT UNSIGNED PK AUTO_INCREMENT,
  user_id         INT UNSIGNED NOT NULL FK → users(id) ON DELETE CASCADE,
  texte_brut      TEXT NOT NULL,
  statut          ENUM('pending','processed','failed') DEFAULT 'pending',
  payload_brut    JSON NULL,
  created_at      TIMESTAMP,
  updated_at      TIMESTAMP
)

depenses(
  id              INT UNSIGNED PK AUTO_INCREMENT,
  recu_id         INT UNSIGNED NOT NULL FK → recus(id) ON DELETE CASCADE,
  libelle         VARCHAR(255) NOT NULL,
  quantite        INT UNSIGNED NOT NULL,
  prix_unitaire   DECIMAL(10,2) NOT NULL,
  categorie       ENUM('alimentaire','boissons','hygiene','entretien','autre') NOT NULL,
  created_at      TIMESTAMP,
  updated_at      TIMESTAMP
)
```

### Eloquent Relationships

```php
// Recu model
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function depenses(): HasMany
{
    return $this->hasMany(Depense::class);
}

// Depense model
public function recu(): BelongsTo
{
    return $this->belongsTo(Recu::class);
}
```

### Eloquent Casts

```php
// Recu model
protected $casts = [
    'statut'       => StatutRecu::class,    // enum cast
    'payload_brut' => 'array',             // json → array cast
];

// Depense model
protected $casts = [
    'categorie'    => CategorieDepense::class,  // enum cast
    'quantite'     => 'integer',
    'prix_unitaire' => 'decimal:2',
];
```

### PHP Enums

```php
// app/Enums/StatutRecu.php
enum StatutRecu: string {
    case Pending    = 'pending';
    case Processed  = 'processed';
    case Failed     = 'failed';
}

// app/Enums/CategorieDepense.php
enum CategorieDepense: string {
    case Alimentaire = 'alimentaire';
    case Boissons    = 'boissons';
    case Hygiene     = 'hygiene';
    case Entretien   = 'entretien';
    case Autre       = 'autre';

    public function label(): string {
        return match($this) {
            self::Alimentaire => 'Food',
            self::Boissons    => 'Drinks',
            self::Hygiene     => 'Hygiene',
            self::Entretien   => 'Maintenance',
            self::Autre       => 'Other',
        };
    }
}
```

---

## 4. Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Framework | Laravel 11 | Backend MVC, routing, ORM |
| PHP | 8.2+ | Enums, named arguments, match expressions |
| Database | MySQL 8 | Relational storage |
| ORM | Eloquent | Models, relationships, casts |
| AI SDK | `laravel/ai` (official) | Structured output, provider abstraction |
| AI Provider | Groq API | Fast LLM inference (Llama 3 or Mixtral) |
| Queue | Laravel Queue (database driver) | Async job dispatching |
| Auth | Laravel Breeze | Authentication scaffolding |
| Frontend | Blade + TailwindCSS + Alpine.js | Server-rendered UI with reactivity |
| Testing | Pest PHP | Unit + feature tests |
| Dev Tools | Laravel Debugbar | N+1 detection, query profiling |
| Specs | OpenSpec | Feature documentation |
| Agent | OpenCode | AI-assisted coding |

---

## 5. AI Agent Configuration

### Agent: OpenCode

**Mode workflow:** Always **Plan Mode first**, then **Build Mode**.

- In Plan Mode: describe the feature, list files to create/edit, identify risks
- In Build Mode: generate the code, scaffold the classes, write the migration

**Never switch to Build Mode without a validated Plan.**

### Model Used for Coding Agent

- Provider: Groq (via API key in `.env`)
- Model recommendation: `llama-3.1-70b-versatile` or `mixtral-8x7b-32768`

### laravel/ai SDK Configuration

```php
// config/ai.php
return [
    'default' => env('AI_PROVIDER', 'groq'),

    'providers' => [
        'groq' => [
            'driver' => 'groq',
            'api_key' => env('GROQ_API_KEY'),
            'model'   => env('GROQ_MODEL', 'llama-3.1-70b-versatile'),
        ],
    ],
];
```

```dotenv
# .env
AI_PROVIDER=groq
GROQ_API_KEY=your_groq_api_key_here
GROQ_MODEL=llama-3.1-70b-versatile
QUEUE_CONNECTION=database
```

### What the Agent Is Allowed to Generate

| Task | Agent generates | Human reviews |
|---|---|---|
| Migrations | ✅ Full scaffold | Column types, enum values |
| Models + casts | ✅ Full | Cast types, relationships |
| Controllers | ✅ Full | Authorization, N+1 checks |
| Form Requests | ✅ Full | Validation rules correctness |
| Jobs | ✅ Full structure | AI prompt quality, error handling |
| Blade views | ✅ Full | UX flow, status display |
| Tests (Pest) | ✅ Full | Fake data accuracy |
| Prompt engineering | ❌ Human-written | Agent should not write the AI system prompt |

### What the Human Must Always Do

- Write the AI **system prompt** for receipt extraction (critical quality gate)
- Validate the JSON contract enforcement in the Job
- Review all `catch` blocks for proper status update to `failed`
- Verify Debugbar shows **zero N+1** on list pages
- Commit with accurate AI-usage annotations

---

## 6. Feature Breakdown & Agent Workflow

### Feature 1 — Authentication (`feature/auth`)

**Scope:** Sign up, log in, log out via Laravel Breeze.

**Plan Mode checklist:**
- [ ] `php artisan breeze:install blade`
- [ ] Verify routes: `/register`, `/login`, `/logout`
- [ ] Protect all receipt routes with `auth` middleware

**Build Mode tasks:**
- Scaffold Breeze
- Add `auth` middleware group to receipt routes in `routes/web.php`

**Agent prompt example:**
```
Plan: Scaffold Laravel Breeze with Blade for authentication.
List all files that will be created. Do not generate code yet.
```

---

### Feature 2 — Receipt CRUD (`feature/recus-crud`)

**Scope:** US2, US3, US4, US5 — list, create, show, delete receipts.

**Plan Mode checklist:**
- [ ] `RecuController` with index, create, store, show, destroy
- [ ] `StoreRecuRequest` with validation rules
- [ ] `Recu` model with casts and relationship
- [ ] Migration for `recus` table
- [ ] Blade views: index (list + status badges), create (form), show (details)

**Key validation rules in `StoreRecuRequest`:**
```php
public function rules(): array
{
    return [
        'texte_brut' => ['required', 'string', 'min:20', 'max:5000'],
    ];
}
```

**Why validation before the AI call:**  
Every AI call to Groq costs latency and potentially API credits. Submitting an empty string or a 2-character input must be rejected at the HTTP layer, never reaching the queue. This is a cost-saving and correctness decision, not a tick-box.

**N+1 prevention:**
```php
// RecuController@index — always eager load
$recus = auth()->user()
    ->recus()
    ->withCount('depenses')
    ->latest()
    ->get();
```

---

### Feature 3 — AI Extraction via Queue (`feature/extraction-ia` + `feature/queue-traitement`)

**Scope:** US6, US7 — async AI processing, status tracking, structured output.

**Plan Mode checklist:**
- [ ] `ExtractExpensesFromReceipt` Job class
- [ ] Job dispatched in `RecuController@store` after receipt creation
- [ ] Job calls laravel/ai SDK with structured output schema
- [ ] Job saves `Depense` records on success, updates `statut`
- [ ] Job catches exceptions → updates `statut` to `failed`
- [ ] Queue configured with `database` driver

**Job skeleton:**
```php
// app/Jobs/ExtractExpensesFromReceipt.php

class ExtractExpensesFromReceipt implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 2;
    public int $timeout = 60;

    public function __construct(public Recu $recu) {}

    public function handle(AiManager $ai): void
    {
        try {
            $response = $ai->structured(
                prompt: $this->buildPrompt(),
                schema: $this->jsonSchema(),
            );

            $payload = $response->data(); // guaranteed array

            $this->recu->update([
                'statut'       => StatutRecu::Processed,
                'payload_brut' => $payload,
            ]);

            foreach ($payload['articles'] as $article) {
                $this->recu->depenses()->create([
                    'libelle'        => $article['libellé'],
                    'quantite'       => $article['quantité'],
                    'prix_unitaire'  => $article['prix_unitaire'],
                    'categorie'      => CategorieDepense::from($article['catégorie']),
                ]);
            }
        } catch (\Throwable $e) {
            $this->recu->update(['statut' => StatutRecu::Failed]);
            throw $e; // re-throw so Laravel marks job as failed
        }
    }

    private function buildPrompt(): string
    {
        return <<<PROMPT
        You are an expert at extracting purchase items from supplier receipts.
        The receipt may be written in Darija (Moroccan Arabic), French, or a mix.
        Extract every item and return ONLY the structured JSON — no explanation, no markdown.

        Receipt:
        {$this->recu->texte_brut}
        PROMPT;
    }

    private function jsonSchema(): array
    {
        return [
            'type' => 'object',
            'required' => ['articles', 'total_estimé', 'devise'],
            'properties' => [
                'articles' => [
                    'type' => 'array',
                    'items' => [
                        'type' => 'object',
                        'required' => ['libellé','quantité','prix_unitaire','catégorie'],
                        'properties' => [
                            'libellé'        => ['type' => 'string'],
                            'quantité'       => ['type' => 'integer'],
                            'prix_unitaire'  => ['type' => 'number'],
                            'catégorie'      => [
                                'type' => 'string',
                                'enum' => ['alimentaire','boissons','hygiène','entretien','autre'],
                            ],
                        ],
                    ],
                ],
                'total_estimé' => ['type' => 'number'],
                'devise'       => ['type' => 'string'],
            ],
        ];
    }
}
```

**Why the Queue is mandatory (not a tick-box):**  
Groq API calls can take 3–15 seconds. Running this synchronously in a controller would block the PHP-FPM process for the duration, degrading server responsiveness for all concurrent users and producing a frozen browser for Brahim. The Queue decouples HTTP response time (< 200ms, instant redirect) from AI processing time (async, handled by a separate worker process). This is a standard scalable architecture for slow external I/O.

**Why Structured Output (not raw `json_decode`):**  
`json_decode` on LLM output is fragile. Models can return markdown fences (` ```json `), trailing commas, extra keys, or hallucinated fields. The `laravel/ai` SDK's structured output mode enforces the schema at the API level (via Groq's JSON mode or function calling), guaranteeing the response matches the contract. If it doesn't, the SDK throws — which we catch and mark as `Failed`. Brahim never sees malformed data.

---

### Feature 4 — Expense Tracking (`feature/depenses`)

**Scope:** US8 — filterable expense list by category.

**Plan Mode checklist:**
- [ ] `DepenseController@index` — fetch all depenses for auth user, with optional category filter
- [ ] Eager load: `Depense::with('recu')` to avoid N+1
- [ ] Blade view: table with category badges, filter dropdown
- [ ] Category labels in French/English using enum `label()` method

**Controller logic:**
```php
public function index(Request $request): View
{
    $query = Depense::whereHas('recu', fn($q) => $q->where('user_id', auth()->id()))
        ->with('recu')
        ->latest();

    if ($request->filled('categorie')) {
        $query->where('categorie', CategorieDepense::from($request->categorie));
    }

    $depenses = $query->get();

    return view('depenses.index', compact('depenses'));
}
```

---

### Bonus Feature — Image Upload (optional)

**Scope:** Upload a photo of a receipt instead of pasting text.

**Plan Mode checklist:**
- [ ] Add `image` column to `recus` table (nullable string)
- [ ] Update `StoreRecuRequest` to accept `image` (mimes:jpg,png,webp, max:5120) OR `texte_brut`
- [ ] Store file via `Storage::disk('public')->store('recus')`
- [ ] In Job: if `recu->image` exists, send as base64 multimodal content to Groq
- [ ] Use a multimodal-capable model (e.g., `llava` or Groq's vision endpoint)

---

### Bonus Feature — Pest Tests

```php
// tests/Feature/ExtractionTest.php

it('extracts expenses from a receipt using fake AI response', function () {
    AiFacade::fake([
        'articles' => [
            [
                'libellé'       => 'Coca-Cola 1L',
                'quantité'      => 12,
                'prix_unitaire' => 8.50,
                'catégorie'     => 'boissons',
            ],
        ],
        'total_estimé' => 102.00,
        'devise'       => 'MAD',
    ]);

    $user = User::factory()->create();
    $recu = Recu::factory()->for($user)->create(['texte_brut' => 'koka kola 12x8.50']);

    ExtractExpensesFromReceipt::dispatchSync($recu);

    expect($recu->fresh()->statut)->toBe(StatutRecu::Processed);
    expect($recu->depenses)->toHaveCount(1);
    expect($recu->depenses->first()->categorie)->toBe(CategorieDepense::Boissons);
});
```

**Why fake AI in tests:**  
Real Groq calls add ~5 seconds per test, make the suite non-deterministic, and cost API tokens. The `laravel/ai` SDK provides a `fake()` method that intercepts the call and returns a predetermined response. Tests run in milliseconds, are deterministic, and never hit the network. This is the correct approach for unit-testing AI-dependent code.

---

## 7. JSON Contract

The AI extraction must always return data conforming to this schema. The `laravel/ai` SDK enforces it.

```json
{
  "articles": [
    {
      "libellé": "string",
      "quantité": "integer",
      "prix_unitaire": "number",
      "catégorie": "enum: alimentaire | boissons | hygiène | entretien | autre"
    }
  ],
  "total_estimé": "number",
  "devise": "string (ex: MAD)"
}
```

**Validation chain:**
1. `StoreRecuRequest` — validates `texte_brut` (not empty, min 20 chars, max 5000 chars)
2. `laravel/ai` structured output — enforces the JSON schema at API level
3. `CategorieDepense::from()` — throws `ValueError` if enum value is invalid (caught → `Failed`)
4. Eloquent casts — final type enforcement at persistence layer

---

## 8. Queue & Job Architecture

### Queue Setup

```bash
# Run migrations to create jobs table
php artisan queue:table
php artisan migrate

# Start worker (keep running during development)
php artisan queue:work --tries=2 --timeout=60
```

### Job Lifecycle

```
RecuController@store
  └── Recu::create(statut: pending)
  └── ExtractExpensesFromReceipt::dispatch($recu)
  └── return redirect()->route('recus.show', $recu)
                        ↓ (async)
Queue Worker picks up job
  └── Job::handle()
      ├── SUCCESS → statut: processed, depenses saved
      └── FAILURE → statut: failed, job marked failed in jobs table
```

### Failed Jobs

```bash
# View failed jobs
php artisan queue:failed

# Retry a specific failed job
php artisan queue:retry {id}
```

---

## 9. Structured Output Strategy

The `laravel/ai` SDK's structured output is the core correctness guarantee of this application. Here is why each part of the strategy matters:

| Decision | Reason |
|---|---|
| Use `$ai->structured()` not `$ai->chat()` | `chat()` returns raw text. LLMs can include markdown, explanations, or malformed JSON. `structured()` enforces the schema. |
| Store `payload_brut` as JSON cast | Keeps the original AI response for debugging without re-querying the API. |
| Cast `categorie` to enum | Prevents invalid category strings from entering the database. |
| `tries = 2` on the Job | Transient API failures (rate limits, network) get one retry before marking as Failed. |
| `timeout = 60` | Groq calls can be slow under load. 60s is generous but prevents zombie jobs. |
| Re-throw exception after `Failed` update | Laravel's queue system needs the exception to mark the job as failed and log it correctly. |

---

## 10. OpenSpec Discipline

### Directory Structure

```
specs/
├── US2_liste_recus.md
├── US3_soumettre_recu.md
├── US4_detail_recu.md
├── US6_extraction_ia.md
├── US7_suivi_statut.md
└── US8_liste_depenses.md
```

### OpenSpec Workflow (per feature)

```
1. proposal  → describe the feature in plain language
2. specs     → break into acceptance criteria + edge cases
3. tasks     → concrete dev tasks (controller method, migration column, etc.)
```

### Example: `specs/US6_extraction_ia.md`

```markdown
# US6 — Structured AI Extraction

## Proposal
When a receipt is submitted, the AI must extract all items and save them as typed Depense records. The extraction must never crash due to malformed JSON.

## Specs
- The AI is called via the laravel/ai SDK with a strict JSON schema
- The response must contain: articles[], total_estimé, devise
- Each article must have: libellé (string), quantité (integer), prix_unitaire (number), catégorie (enum)
- If the AI response does not match the schema, the receipt status is set to 'failed'
- Depenses are only created on success

## Tasks
- [ ] Create Job: ExtractExpensesFromReceipt
- [ ] Define jsonSchema() method on the Job
- [ ] Call $ai->structured(prompt, schema) in handle()
- [ ] Loop over $payload['articles'] and create Depense records
- [ ] Wrap in try/catch, update statut on failure
- [ ] Write Pest test with AiFacade::fake()
```

---

## 11. Commit Convention

All commits must follow this format:

```
<type>(<scope>): <short description> [ai:<tool>]
```

### Types
- `feat` — new feature
- `fix` — bug fix
- `refactor` — code improvement without feature change
- `test` — adding or updating tests
- `docs` — documentation
- `chore` — config, tooling, migrations

### AI Usage Tags
- `[ai:opencode]` — code generated by OpenCode agent
- `[ai:plan]` — architecture or plan produced by agent
- `[ai:groq]` — prompt or AI config written with AI assistance
- `[ai:manual]` — this commit was written entirely by hand

### Examples

```
feat(auth): scaffold Laravel Breeze authentication [ai:opencode]
feat(recus): add Recu model with statut/payload_brut casts [ai:opencode]
feat(queue): implement ExtractExpensesFromReceipt job with structured output [ai:opencode]
fix(job): catch ValueError on invalid enum and mark receipt as failed [ai:manual]
refactor(recus): eager load depenses_count on index to fix N+1 [ai:manual]
test(extraction): add Pest test with AiFacade fake for US6 [ai:opencode]
docs(agents): add AGENTS.md with full project architecture [ai:manual]
chore(queue): configure database queue driver and run queue:table migration [ai:manual]
```

### Branch Strategy

```
main
├── feature/auth
├── feature/recus-crud
├── feature/extraction-ia
├── feature/queue-traitement
└── feature/depenses
```

Minimum **15 commits**, with **daily commits mandatory**.

---

## 12. Day-by-Day Plan

### Day 1 — Monday 08/06 (Setup + Auth + MCD/MLD)

**Goals:**
- [ ] Laravel 11 project created, pushed to GitHub
- [ ] `AGENTS.md` committed (first commit of the project)
- [ ] Laravel Breeze installed and configured
- [ ] `recus` and `depenses` migrations written and run
- [ ] Eloquent models created with casts and relationships
- [ ] Enums created: `StatutRecu`, `CategorieDepense`
- [ ] MCD and MLD drawn and committed
- [ ] OpenSpec: `specs/` directory created, first 2 spec files written
- [ ] Debugbar installed

**Commits target: 3–4**

---

### Day 2 — Tuesday 09/06 (Receipt CRUD)

**Goals:**
- [ ] `RecuController` with index, create, store, show, destroy
- [ ] `StoreRecuRequest` with validation rules
- [ ] Blade views: recus/index, recus/create, recus/show
- [ ] Status badges (Pending / Processed / Failed) on index
- [ ] Delete with confirmation
- [ ] N+1 check on index with Debugbar

**Commits target: 3–4**

---

### Day 3 — Wednesday 10/06 (Queue + AI Extraction)

**Goals:**
- [ ] Queue configured (database driver), `jobs` table migrated
- [ ] `ExtractExpensesFromReceipt` Job created
- [ ] Job dispatched on receipt creation
- [ ] laravel/ai SDK integrated, Groq provider configured
- [ ] System prompt and JSON schema written in Job
- [ ] Structured output tested manually with a real Darija receipt
- [ ] Status correctly updated to Processed / Failed
- [ ] `queue:work` running and processing jobs

**Commits target: 4–5**

---

### Day 4 — Thursday 11/06 (Expense Tracking + Pest Tests)

**Goals:**
- [ ] `DepenseController@index` with category filter
- [ ] Blade view: depenses/index with category badges and dropdown filter
- [ ] Eager loading verified (zero N+1 with Debugbar)
- [ ] Pest test for AI extraction with `AiFacade::fake()`
- [ ] OpenSpec: remaining spec files written
- [ ] Bonus: image upload (if time permits)

**Commits target: 3–4**

---

### Day 5 — Friday 12/06 (Polish + Demo Prep)

**Goals:**
- [ ] README.md complete (setup, env config, `queue:work` instructions)
- [ ] Final N+1 audit with Debugbar
- [ ] All feature branches merged to `main`
- [ ] Jira tickets reviewed and closed
- [ ] Live demo rehearsed (can explain Queue + Structured Output decisions verbally)
- [ ] Final commit pushed before 14:30

**Commits target: 2–3**

---

## 13. Performance Constraints

### Zero N+1 — Verified with Debugbar

**On `recus/index`:** Must use `withCount('depenses')` — never lazy-load count in a loop.

```php
// ✅ Correct
$recus = auth()->user()->recus()->withCount('depenses')->latest()->get();

// ❌ N+1 — never do this in a view
@foreach($recus as $recu)
    {{ $recu->depenses->count() }}
@endforeach
```

**On `depenses/index`:** Must use `with('recu')` to avoid loading receipt per expense row.

```php
// ✅ Correct
Depense::whereHas('recu', ...)->with('recu')->get();
```

### Page Must Never Freeze

The `store` action in `RecuController` must always return before the AI call completes. The Job must be dispatched, not run inline.

```php
// ✅ Correct — dispatches and redirects immediately
ExtractExpensesFromReceipt::dispatch($recu);
return redirect()->route('recus.show', $recu)
    ->with('success', 'Receipt being processed...');

// ❌ Wrong — blocks the request
$job = new ExtractExpensesFromReceipt($recu);
$job->handle(app(AiManager::class)); // synchronous, freezes page
```

---

## 14. What the Agent Generated vs. What Was Modified

| File | Agent generated | Human modified |
|---|---|---|
| `app/Jobs/ExtractExpensesFromReceipt.php` | Full class structure, try/catch, dispatch | System prompt text, JSON schema enum values |
| `app/Http/Requests/StoreRecuRequest.php` | Full class | Adjusted `min` length to 20 based on real Darija receipts |
| `app/Models/Recu.php` | Full model with `$casts`, `$fillable`, relationships | Verified cast types matched migration enum values |
| `app/Models/Depense.php` | Full model | Confirmed `decimal:2` cast for `prix_unitaire` |
| `app/Http/Controllers/RecuController.php` | Full CRUD | Added `withCount` for N+1 fix, added `auth()->id()` scope check |
| `resources/views/recus/index.blade.php` | Full table with badges | Status badge colors and Darija-friendly labels |
| `database/migrations/` | All migrations | Verified enum values match PHP enums exactly |
| `tests/Feature/ExtractionTest.php` | Full test with fake | Verified fake response matches actual JSON contract |
| `config/ai.php` | Base structure | Added `model` key and environment variable reference |
| AI system prompt (in Job) | ❌ Human only | Written entirely by hand — critical quality gate |

---

## 15. Architectural Decisions Log

### Decision 1: Queue instead of synchronous AI call

**Problem:** Groq API calls take 3–15 seconds. Running in a controller blocks the HTTP process and freezes the browser.  
**Decision:** Dispatch `ExtractExpensesFromReceipt` as a queued job. Controller returns a redirect immediately.  
**Trade-off:** User must refresh or poll to see the result. Acceptable for this use case — status is visible on the receipt detail page.  
**Alternative considered:** SSE / WebSockets for real-time status update. Rejected as over-engineered for a 5-day solo project.

---

### Decision 2: Structured output instead of raw `json_decode`

**Problem:** LLMs produce unreliable JSON when using `chat()`. Markdown fences, extra text, and schema violations cause `json_decode` failures that crash the job silently.  
**Decision:** Use `laravel/ai` SDK's `structured()` method with an explicit JSON schema. The provider (Groq) enforces the schema at inference time.  
**Trade-off:** Slightly longer API call. Completely justified by correctness guarantee.  
**Alternative considered:** Manual regex/cleanup of raw response. Rejected — fragile and unmaintainable.

---

### Decision 3: Form Request validation before AI call

**Problem:** Every AI API call costs latency and potentially money. Invalid or empty inputs should be rejected before reaching the queue.  
**Decision:** `StoreRecuRequest` validates `texte_brut` at the HTTP layer. Only valid inputs are dispatched as jobs.  
**Trade-off:** None — this is strictly better than validating inside the controller or the job.

---

### Decision 4: Enum casts for `statut` and `categorie`

**Problem:** Storing status and category as raw strings allows invalid values to enter the database and requires string comparisons throughout the codebase.  
**Decision:** PHP 8.1+ backed enums with Eloquent casts. `StatutRecu::from('pending')` throws on invalid values. IDE autocompletion works correctly.  
**Trade-off:** Slightly more boilerplate. Completely justified by type safety.

---

### Decision 5: Store `payload_brut` as JSON

**Problem:** If a receipt's expenses are corrupted or need re-processing, the original AI response is lost.  
**Decision:** Save the full AI payload as `payload_brut` (JSON cast) on the `recus` table. This enables debugging without re-calling the API.  
**Trade-off:** Slightly more storage. Justified by debuggability.

---

*This document is the single source of truth for AI usage, architectural decisions, and development workflow on the Expense Assistant project. It must be committed on Day 1 and kept up-to-date throughout the 5-day sprint.*