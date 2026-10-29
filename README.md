# 🤖 Featured Project: FRIDAY (Personalized Telegram AI Assistant)

<div align="center">

[![TypeScript](https://img.shields.io/badge/TypeScript-5.8-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Node.js](https://img.shields.io/badge/Node.js-22+-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Google Gemini](https://img.shields.io/badge/Google_Gemini-3.5_Flash-4285F4?logo=google-gemini&logoColor=white)](https://aistudio.google.com/)
[![Supabase](https://img.shields.io/badge/Supabase-pgvector-3ECF8E?logo=supabase&logoColor=white)](https://supabase.com/)
[![Tavily](https://img.shields.io/badge/Tavily-Live_Search-FF6F00)](https://tavily.com/)
[![grammY](https://img.shields.io/badge/Telegram_Bot-grammY-24A1DE?logo=telegram&logoColor=white)](https://grammy.dev/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Tests](https://img.shields.io/badge/Vitest-184_Passing-6E9F18?logo=vitest&logoColor=white)](https://vitest.dev/)

**An autonomous, single-owner AI executive assistant operating over Telegram, engineered with long-term semantic vector memory, real-time web research, multi-provider outage failover, and proactive scheduled briefings.**


---

</div>

## 📌 Executive Summary

**FRIDAY** is an intelligent assistant designed to act as a 24/7 autonomous executive partner. Built from scratch with **TypeScript ESM** and **Node.js 22**, it bridges cutting-edge LLMs with persistent PostgreSQL vector storage, proactive background cron scheduling, and Telegram's interactive messaging interface.

- **Author & Architect:** [Tanmay Chavhan (@tanmayChava)](https://github.com/tanmayChava)
- **Repository:** [NexusInno1/Friday](https://github.com/NexusInno1/Friday)
- **Test Suite:** **184 tests across 26 test suites (100% pass rate)**

---

## ⚡ System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Telegram Private Chat                           │
│                      (Strict Owner Auth Guard)                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     grammY Runtime Middleware Engine                   │
│         (Dual-mode: Zero-tunnel Polling ⇄ High-throughput Webhook)     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Tiered Failover │      │ Long-Term Memory │      │ Proactive Engine │
│  LLM Engine      │      │ (pgvector HNSW)  │      │ (Croner Runner)  │
│  - Gemini 3.5    │      │ - Cosine Recall  │      │ - Morning Digest │
│  - Groq Llama 3  │      │ - Auto Extract   │      │ - Minute Alerts  │
│  - OpenAI GPT-4o │      │ - Superseding    │      │ - Lease Lock     │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

---

## 🔬 Core Engineering & Code Walkthrough

### 1. Tiered Multi-Provider Outage Failover (`src/services/agent.service.ts`)
To achieve zero downtime against API outages and rate limits, FRIDAY cascades automatically:
**Google Gemini 3.5 Flash ➔ Groq (Llama 3.3 70B) ➔ OpenAI (GPT-4o)**.

```typescript
// Automatic tier cascade upon retriable upstream failures
try {
  const res = await runGeneration(this.model);
  responseText = res.text;
  tokensUsed = res.tokensUsed;
} catch (primaryErr) {
  if (this.fallbacks.length > 0 && isRetriableModelError(primaryErr)) {
    logger.warn("model_failover", {
      fromProvider: env().DEFAULT_LLM_PROVIDER,
      fromModel: env().GEMINI_MODEL,
      reason: primaryErr instanceof Error ? primaryErr.message : String(primaryErr),
    });

    for (const fallback of this.fallbacks) {
      try {
        let res: { text: string; tokensUsed?: number };

        // Fallback with simpler schema resilience if tool parser rejects complex schema
        if (fallback.provider.toLowerCase().includes("groq")) {
          try {
            res = await runGeneration(fallback.model);
          } catch {
            const simplifiedTools = createSimplifiedTools({ ...turnState });
            res = await runGeneration(fallback.model, simplifiedTools);
          }
        } else {
          res = await runGeneration(fallback.model);
        }

        responseText = `${res.text}\n\n_(⚡ Served via ${fallback.provider} fallback)_`;
        tokensUsed = res.tokensUsed;
        break;
      } catch (fallbackErr) {
        logger.warn("model_fallback_failed", { provider: fallback.provider });
      }
    }
  }
}
```

---

### 2. Semantic Memory & Vector Cosine Recall (`src/actions/actions.ts`)
Stores facts and life preferences with 768-dimensional embeddings generated via `gemini-embedding-2` and recalls them using PostgreSQL `pgvector` HNSW index queries:

```typescript
/**
 * Computes 768-dimensional dense vector embeddings
 */
export async function generateEmbedding(text: string): Promise<number[]> {
  const { GEMINI_API_KEY, EMBEDDING_MODEL: configuredModel } = env();
  const activeModel = configuredModel || "gemini-embedding-2";

  const response = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/${activeModel}:embedContent?key=${GEMINI_API_KEY}`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: `models/${activeModel}`,
        content: { parts: [{ text }] },
        outputDimensionality: 768,
      }),
    }
  );

  const data = await response.json();
  return data.embedding.values;
}

/**
 * Recalls memories using vector cosine distance RPC in Supabase pgvector
 */
export async function recallMemoriesAction(
  query: string,
  limit = 5,
  threshold = 0.65
): Promise<Memory[]> {
  const queryEmbedding = await generateEmbedding(query);
  const matched = await store.matchMemories(
    userId,
    queryEmbedding,
    threshold,
    limit
  );
  return matched;
}
```

---

### 3. Distributed Lease-Locked Scheduler (`src/db/supabase-datastore.ts`)
Prevents double-delivery of minute-by-minute reminders across distributed instances using atomic conditional leases:

```typescript
/**
 * Claims due reminders with atomic timestamp lease locks
 */
async claimDueReminders(nowIso: string, leaseDurationMs: number): Promise<Reminder[]> {
  const leaseUntilIso = new Date(Date.now() + leaseDurationMs).toISOString();

  // Atomically filter candidates that are due AND unleased or lease-expired
  const { data: candidates } = await this.db
    .from("reminders")
    .select()
    .lte("trigger_at", nowIso)
    .eq("is_completed", false)
    .eq("is_cancelled", false)
    .is("sent_at", null)
    .or(`lease_until.is.null,lease_until.lt.${nowIso}`);

  if (!candidates?.length) return [];

  // Claim candidate lease in an atomic update
  const claimed = await Promise.all(
    candidates.map(async (reminder) => {
      const { data } = await this.db
        .from("reminders")
        .update({
          lease_until: leaseUntilIso,
          delivery_attempts: (reminder.delivery_attempts ?? 0) + 1,
        })
        .eq("id", reminder.id)
        .eq("is_completed", false)
        .select()
        .single();
      return data;
    })
  );

  return claimed.filter(Boolean);
}
```

---

### 4. Zero-Trust Single-Owner Auth Guard (`src/bot/middlewares/auth.ts`)
Ensures no unauthorized user or group injection can query or view executive memories:

```typescript
/**
 * Strict owner whitelist & chat-scope isolation
 */
export async function authMiddleware(ctx: Context, next: NextFunction): Promise<void> {
  const allowedUserId = env().TELEGRAM_ALLOWED_USER_ID;
  const userId = ctx.from?.id;

  // 1. Drop unauthorized users silently (zero footprint)
  if (!userId || userId !== allowedUserId) {
    console.warn(`[auth] Blocked unauthorized access attempt from ID: ${userId ?? "unknown"}`);
    return;
  }

  // 2. Reject group chats, supergroups, and channels
  if (ctx.chat && ctx.chat.type !== "private") {
    console.warn(`[auth] Blocked request from non-private chat type: ${ctx.chat.type}`);
    return;
  }

  await next();
}
```

---

## 💬 Supported Telegram Commands

| Command | Action |
| :--- | :--- |
| `/start` | Initialize assistant and display operational status |
| `/briefing [HH:MM]` | Configure proactive daily morning intelligence report |
| `/reminders` | Interactive reminder list with inline **Snooze** and **Cancel** buttons |
| `/remember <fact>` | Explicitly save a long-term memory or life preference |
| `/memories [query]` | Inspect and delete stored vector memories with similarity search |
| `/search <query>` | Trigger Tavily AI real-time web search with source citations |
| `/sessions` | Switch conversation threads or isolate project topics |
| `/history [query]` | Hybrid search over historical chat turns and past decisions |
| `/status` | View live telemetry, token costs, model latency, and database health |

---

## 🧪 Verification & Quality Bar

```bash
# Typecheck TypeScript codebase (Zero errors)
npm run typecheck

# Execute full Vitest suite (184 unit & integration tests)
npm test
```

```text
 ✓ src/services/agent.test.ts (13 tests)
 ✓ src/actions/actions.test.ts (16 tests)
 ✓ src/services/scheduler.test.ts (11 tests)
 ✓ src/db/supabase-datastore.test.ts (3 tests)
 ✓ src/bot/middlewares/auth.test.ts (6 tests)
 ...
 Test Files  26 passed (26)
      Tests  184 passed (184)
   Duration  15.56s
```

---

## 🛠️ Tech Stack Matrix

| Dimension | Technology |
| :--- | :--- |
| **Language & Runtime** | TypeScript 5.8, Node.js 22 (ESM, strict) |
| **Bot Framework** | [grammY](https://grammy.dev) |
| **Primary AI Model** | Google Gemini 3.5 Flash |
| **Failover Models** | Groq (`llama-3.3-70b-versatile`), OpenAI (`gpt-4o`) |
| **Vector Database** | Supabase (PostgreSQL 15+, `pgvector`, HNSW) |
| **Embeddings** | Google `gemini-embedding-2` (768 dimensions) |
| **Web Research** | Tavily Search API |
| **Task Scheduler** | Croner |
| **Testing** | Vitest (184 unit & integration tests) |

---

<div align="center">

**Crafted by [Tanmay Chavhan (@tanmayChava)](https://github.com/tanmayChava)**  
*Project Repository: [NexusInno1/Friday](https://github.com/NexusInno1/Friday)*

</div>
