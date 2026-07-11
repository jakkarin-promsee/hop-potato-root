# Tier 1 — Observability: Full Execution Plan

**For the implementing agent (Cursor).** This is a complete, self-contained spec for **Tier 1 of [`../ROADMAP.md`](../ROADMAP.md)** (launch-readiness v3, 2026-07-10): an in-app error ring buffer + AI health signal on the server, surfaced on the public `/status` page. Execute top to bottom. Every file path, contract, code block, and test case you need is in this document — **but always read the referenced source file before editing it**; this plan describes the code as of 2026-07-10 and the file is the truth.

> **Re-verified 2026-07-10 (second pass, line-by-line against both working trees):** catch-block line numbers (481 / 528), helper signatures (`seedUser` overrides + `{ user, token }`, `seedLesson(ownerId, overrides)`, `clearLessonContextCache`), the `@google/genai` mock pattern, `useAppI18n` shape, every quoted copy string in `Status.tsx`, the `retry.ts` transient regex (the `"boom"` trick is safe), `errorHandler` mounted last in `app.ts`, and both CLAUDE.md doc anchors. If a file differs from what a step describes, **stop and report** instead of adapting silently.

> **Read first, in this order:** [`../AGENT.md`](../AGENT.md) → [`../CLAUDE.md`](../CLAUDE.md) → [`../server/CLAUDE.md`](../server/CLAUDE.md) → [`../client/CLAUDE.md`](../client/CLAUDE.md). They are short and they will stop you from breaking things this plan assumes you know.

---

## 0. Ground rules (non-negotiable)

### The two Golden Rules
1. **Never ration tokens for real students.** No quotas, no caps, no limit UI. In this tier specifically: **never ping Gemini from the status endpoint** — AI health is recorded passively from real tutor calls, not probed.
2. **Anonymous users keep FULL access.** `/status` (client page) and `GET /api/status/all` (API) are **public** and must stay public. Do not add `protect` to any status or AI route.

### Git reality ⚠️
- `client/` and `server/` are **two separate git repositories**. There is **no** repo at the `Hot-Potato/` root.
- Run every git command **inside** `client/` or `server/`. If `git status` shows foreign files (e.g. `ChronoForge-FPGA-Engine`), you are in the wrong directory — stop immediately.
- **Commit but do not push** (owner decision D1: nothing is pushed until Tier 5 launch day). Commit each half separately; suggested messages are at the end of each phase.

### Working agreement
- Every phase ships **its own tests**. A phase is done only when `cd server && npm test` **and** `cd client && npm test` are green in every half touched.
- Do **not** run `npm run test:ai` — this tier never touches prompts or models (`persona.ts`, `parse.ts`, `models.ts` stay untouched).
- **No new environment variables** (ROADMAP §12).
- Update `server/CLAUDE.md` / `client/CLAUDE.md` when done (exact edits in §4).
- Do not touch `example_copy/`, `example_project/`, `test/` (root), `Ptest/`, `archive/`, `graphify-out/`.

### Step 0 — Preflight (do this before touching anything)

Both repos have **known pre-existing uncommitted changes** (verified 2026-07-10) — leftovers from the owner's sign-off session, unrelated to Tier 1. They must **not** ride into the Tier 1 commits. Handle them first, as their own commits:

1. **Server** — `cd server && git status --short` must show **exactly**:
   ```
    M CLAUDE.md
    M prompt-notes.md
   ```
   (A one-line banner truth-up "Tier 0 in progress → Tutor core shipped & owner-approved" and the owner-sign-off log row.) Commit them as-is:
   ```powershell
   git add CLAUDE.md prompt-notes.md
   git commit -m "docs: truth-up AI-integration banner + log owner sign-off (pre-Tier-1 leftover)"
   ```
2. **Client** — `cd client && git status --short` must show **exactly**:
   ```
    M src/components/editor/EditorLeftSidebar.tsx
   ```
   (Category label rename `Special/พิเศษ` → `Question/คำถาม`, 3 spots, consistent.) Commit it as-is:
   ```powershell
   git add src/components/editor/EditorLeftSidebar.tsx
   git commit -m "feat(editor): rename Special sidebar category to Question (pre-Tier-1 leftover)"
   ```
3. **If either repo shows anything else** (different files, deletions, untracked source files) — **stop and report to the owner**. Do not guess.
4. **Baseline runs** (after the leftover commits, before any Tier 1 edit):
   ```powershell
   cd server; npm test     # expect 203/203 green
   cd server; npm run build
   cd client; npm test     # expect 74/74 green (66 + 8 SEO from Tier 0)
   cd client; npm run build   # green, with the known ~2 MB chunk warning (Tier 2's problem)
   ```
   If any pre-existing test fails, **stop and report** — do not fix unrelated breakage.
5. **Never use `git add -A` or `git add .` anywhere in this plan.** Every commit stages an explicit file list.

### Phase order
**1.A (server) → 1.B (client).** 1.B renders the payload 1.A creates; do not start 1.B until 1.A's tests are green.

### Careful-not-to-break (Tier-1 specific)

| Thing | Where | Why |
| --- | --- | --- |
| `generateWithRetry` | `server/src/services/tutor/retry.ts` | Exists because the owner's ISP drops connections. 1.A records its **final outcome** — never delete, bypass, or record failures for transparent mid-retry attempts. |
| Error handler order | `server/src/app.ts` | `app.use(errorHandler)` must stay **last**. 1.A hooks *inside* the handler, not around it. |
| `errorHandler` response shape | `error.middleware.ts` | Keeps returning 500 + `{ message }` exactly as today (production hides `err.message`). The hook only **adds** recording. |
| Existing `/api/status/all` payload | `status.controller.ts` | The client store validates `"checks" in data`. Only **add** keys (`checks.ai`, `checks.errors`); never rename or remove existing ones. HTTP stays `200`/`503` computed from server+db+env only. |
| Sanitization on public status | `status.controller.ts` | No host/port/db name today; keep it that way. New rule: **no error messages or stacks, ever** — not even stored (see 1.A design). |
| SSE tutor path | `tutor.controller.ts` | Stream errors are handled inside the controller (headers already sent — they never reach `errorHandler`). 1.A records them there; do not restructure the stream logic. |
| Rate limiting | `chat.routes.ts` / `auth.routes.ts` | Untouched in this tier. |
| Status page poll + skeleton | `client/src/pages/Status.tsx` | Keep the 30 s `setInterval` poll, the refresh button, skeleton, and error-alert behavior exactly as-is. |

### Verification environment
- Server: `cd server && npm run dev` → `http://localhost:5000`. Client: `cd client && npm run dev` → `http://localhost:5173`.
- Status page: `http://localhost:5173/status`. Test lesson (for a real tutor call): `http://localhost:5173/view/69e39d0b60d467bd515a4945`.
- Always check **both auth states** (logged in + incognito) and a **~390 px** viewport for UI work.

---

## 1. What Tier 1 builds — the wire contract (single source of truth)

`GET /api/status/all` (public, no auth) gains two keys inside `checks`. Implement **exactly** this shape — the client (1.B) and the tests bind to it:

```jsonc
{
  "status": "ok" | "degraded",        // UNCHANGED: computed from server+db+env only
  "timestamp": "2026-07-10T12:00:00.000Z",
  "checks": {
    "server":   { /* unchanged */ },
    "database": { /* unchanged */ },
    "env":      { /* unchanged */ },

    // NEW — AI tutor health, recorded passively from real tutor calls:
    "ai": {
      "status": "ok" | "degraded" | "unknown",  // unknown = no tutor call since boot
      "last_success": "ISO string" | null,
      "last_failure": "ISO string" | null
    },

    // NEW — server error ring buffer (in-memory, wiped on restart):
    "errors": {
      "count_since_boot": 0,
      "last_error_at": "ISO string" | null,
      "since": "ISO string",                     // boot time → client shows "since last restart"
      "recent": [                                 // newest first, max 50
        { "time": "ISO", "method": "POST", "route": "/api/chat/tutor", "status": 502 }
        // NOTE: no "name", no "message", no "stack" in the PUBLIC payload
      ]
    }
  }
}
```

Design decisions (do not re-litigate — they come from the ROADMAP and owner principles):

1. **In-memory only.** Render free tier restarts wipe it; that is acceptable and labeled on the page ("since last restart"). No DB collection, no log files.
2. **Never store `err.message` or request bodies** — messages can echo student input. The buffer stores `err.name` only, and even that is **stripped from the public payload**; the admin-only `GET /api/status/` shows names.
3. **AI health is passive.** It flips on real tutor-call outcomes (`genaiClient.ts`). The status endpoint never calls Gemini.
4. **Overall `status` / HTTP code unchanged.** One failed AI call or a recorded error is information, not an outage — the AI card carries its own badge. Uptime probes keep their `200/503` contract.
5. **Background memory updates don't count.** `services/tutor/memory.ts` has its own SDK call and is *not* hooked — `ai.status` reflects student-facing tutor calls only.

---

## 2. Phase 1.A — Server: error ring buffer + AI health

**Goal:** every 500 that reaches `errorHandler` and every tutor-call failure is visible on `GET /api/status/all` within one poll — route + status code only, no messages. AI health flips on real tutor outcomes. Size **S–M**, repo: `server/` only.

### Current state you're building on (verified 2026-07-10)

- `src/middlewares/error.middleware.ts` — 15 lines: logs `err.stack`, responds 500 with `err.message` (generic in production). Mounted **last** in `src/app.ts`.
- `src/controllers/tutor.controller.ts` (`tutorChat`) handles its own AI errors — they never reach `errorHandler`:
  - **Non-stream** (~line 528): `catch` → `MISSING_GEMINI_API_KEY` → 500; anything else → `console.error` + **502** `{ message: "AI request failed" }`.
  - **Stream** (~line 481): `catch` inside the SSE block → sends an SSE `error` event and `res.end()` (HTTP was already 200).
- `src/services/tutor/genaiClient.ts` — `callTutorModel` and `callTutorModelStream` both wrap the SDK call in `generateWithRetry` and throw `Error("MISSING_GEMINI_API_KEY")` when the key is absent.
- `src/controllers/status.controller.ts` — `getAllStatus` builds `checks.server/database/env`; `getServerStatus` is the admin-only detailed view.
- Tests mock the SDK itself (`vi.mock("@google/genai", ...)` — see `test/api/tutor.test.ts` lines 5–13), **not** `genaiClient.ts` — so code you add inside `genaiClient.ts` runs for real under supertest. Use this.

### Step A1 — Create `server/src/services/observability.service.ts`

New file, complete content:

```ts
// In-memory observability signals for the public /status page (Tier 1.A).
//
// Intentionally in-memory: Render's free tier restarts wipe it, which is
// acceptable — the Status page labels the data "since last restart".
// PRIVACY RULE: never store request bodies, student messages, or error
// messages (err.message can echo user input). Only names, routes, status
// codes, and timestamps — and even names stay off the public payload.

const MAX_RECENT_ERRORS = 50;
const MAX_ROUTE_LENGTH = 120;
const MAX_NAME_LENGTH = 80;

export interface RecordedError {
  time: string; // ISO timestamp
  method: string; // "GET" | "POST" | ...
  route: string; // path only — query string stripped
  status: number; // HTTP status the client received (or semantic outcome for SSE)
  name: string; // err.name only — NEVER err.message
}

export type PublicRecordedError = Omit<RecordedError, "name">;

export type AiStatus = "ok" | "degraded" | "unknown";

export interface AiHealth {
  status: AiStatus;
  last_success: string | null;
  last_failure: string | null;
}

const bootTime = new Date().toISOString();

let recentErrors: RecordedError[] = [];
let errorCountSinceBoot = 0;

let aiStatus: AiStatus = "unknown";
let aiLastSuccess: string | null = null;
let aiLastFailure: string | null = null;

export function recordServerError(entry: {
  method: string;
  route: string;
  status: number;
  name: string;
}): void {
  const route = (entry.route || "").split("?")[0].slice(0, MAX_ROUTE_LENGTH);
  recentErrors.unshift({
    time: new Date().toISOString(),
    method: entry.method || "UNKNOWN",
    route,
    status: entry.status,
    name: (entry.name || "Error").slice(0, MAX_NAME_LENGTH),
  });
  if (recentErrors.length > MAX_RECENT_ERRORS) {
    recentErrors.length = MAX_RECENT_ERRORS;
  }
  errorCountSinceBoot += 1;
}

export function recordAiSuccess(): void {
  aiStatus = "ok";
  aiLastSuccess = new Date().toISOString();
}

export function recordAiFailure(): void {
  aiStatus = "degraded";
  aiLastFailure = new Date().toISOString();
}

export function getAiHealth(): AiHealth {
  return {
    status: aiStatus,
    last_success: aiLastSuccess,
    last_failure: aiLastFailure,
  };
}

// Public payload — error names stripped (a library name can hint at internals).
export function getErrorSummary(): {
  count_since_boot: number;
  last_error_at: string | null;
  since: string;
  recent: PublicRecordedError[];
} {
  return {
    count_since_boot: errorCountSinceBoot,
    last_error_at: recentErrors[0]?.time ?? null,
    since: bootTime,
    recent: recentErrors.map(({ name: _name, ...pub }) => pub),
  };
}

// Admin payload — includes err.name (still never messages or stacks).
export function getErrorSummaryDetailed(): {
  count_since_boot: number;
  last_error_at: string | null;
  since: string;
  recent: RecordedError[];
} {
  return {
    count_since_boot: errorCountSinceBoot,
    last_error_at: recentErrors[0]?.time ?? null,
    since: bootTime,
    recent: [...recentErrors],
  };
}

// Test-only: module-level state must be resettable between test cases.
export function resetObservabilityForTests(): void {
  recentErrors = [];
  errorCountSinceBoot = 0;
  aiStatus = "unknown";
  aiLastSuccess = null;
  aiLastFailure = null;
}
```

### Step A2 — Hook `error.middleware.ts`

Replace the file body with (behavior identical, recording added):

```ts
import { Request, Response, NextFunction } from "express";
import { recordServerError } from "../services/observability.service";

export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction,
): void => {
  console.error(err.stack ?? err);
  recordServerError({
    method: req?.method ?? "UNKNOWN",
    route: req?.originalUrl ?? req?.url ?? "",
    status: 500,
    name: err?.name || "Error",
  });
  const message =
    process.env.NODE_ENV === "production"
      ? "Internal Server Error"
      : err.message || "Internal Server Error";
  res.status(500).json({ message });
};
```

⚠️ The existing unit test `test/unit/errorHandler.test.ts` calls `errorHandler(new Error(...), {} as never, res, next)` — an **empty object** as `req`. The `?.`/`??` fallbacks above exist so that keeps passing. Do not remove them.

### Step A3 — Hook AI health in `genaiClient.ts`

Edit `server/src/services/tutor/genaiClient.ts`. Add the import:

```ts
import { recordAiFailure, recordAiSuccess } from "../observability.service";
```

**`callTutorModel`** — record failure on missing key, wrap the call:

```ts
export async function callTutorModel(opts: {
  model: string;
  systemInstruction: string;
  history: TutorTurn[];
  message: string;
}): Promise<string> {
  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) {
    recordAiFailure();
    throw new Error("MISSING_GEMINI_API_KEY");
  }
  const ai = new GoogleGenAI({ apiKey });
  const contents = [
    ...opts.history.map((t) => ({ role: t.role, parts: [{ text: t.text }] })),
    { role: "user" as const, parts: [{ text: opts.message }] },
  ];
  try {
    const result = await generateWithRetry(() =>
      ai.models.generateContent({
        model: opts.model,
        contents,
        config: { systemInstruction: opts.systemInstruction },
      }),
    );
    recordAiSuccess();
    return (result.text ?? "").trim();
  } catch (error) {
    recordAiFailure();
    throw error;
  }
}
```

**`callTutorModelStream`** — same idea; success is recorded only when the stream **fully completes** (a mid-stream drop is a failure):

```ts
export async function* callTutorModelStream(opts: {
  model: string;
  systemInstruction: string;
  history: TutorTurn[];
  message: string;
}): AsyncGenerator<string> {
  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) {
    recordAiFailure();
    throw new Error("MISSING_GEMINI_API_KEY");
  }
  const ai = new GoogleGenAI({ apiKey });
  const contents = [
    ...opts.history.map((t) => ({ role: t.role, parts: [{ text: t.text }] })),
    { role: "user" as const, parts: [{ text: opts.message }] },
  ];
  try {
    const stream = await generateWithRetry(() =>
      ai.models.generateContentStream({
        model: opts.model,
        contents,
        config: { systemInstruction: opts.systemInstruction },
      }),
    );
    for await (const chunk of stream) {
      const text = chunk.text ?? "";
      if (text) yield text;
    }
    recordAiSuccess();
  } catch (error) {
    recordAiFailure();
    throw error;
  }
}
```

Notes:
- `generateWithRetry` retries transparently **inside** the try — only its *final* outcome is recorded. Correct by construction; don't add recording inside `retry.ts`.
- Do **not** touch `services/tutor/memory.ts` — background memory updates deliberately don't count as tutor health.

### Step A4 — Record ring-buffer entries in `tutor.controller.ts`

AI failures respond from inside `tutorChat` (they never reach `errorHandler`), so the ring buffer must be fed there too. Add the import:

```ts
import { recordServerError } from "../services/observability.service";
```

**Non-stream catch** (currently ~line 528 — read the file, find the `catch` right after the `res.status(200).json({ reply, ... })` block). Replace with:

```ts
  } catch (error) {
    const msg = error instanceof Error ? error.message : String(error);
    if (msg === "MISSING_GEMINI_API_KEY") {
      recordServerError({
        method: req.method,
        route: req.originalUrl ?? "/api/chat/tutor",
        status: 500,
        name: "MISSING_GEMINI_API_KEY",
      });
      res
        .status(500)
        .json({ message: "Missing GEMINI_API_KEY in server env" });
      return;
    }
    console.error("Tutor AI request failed:", error);
    recordServerError({
      method: req.method,
      route: req.originalUrl ?? "/api/chat/tutor",
      status: 502,
      name: error instanceof Error ? error.name : "Error",
    });
    res.status(502).json({ message: "AI request failed" });
  }
```

**Stream catch** (currently ~line 481 — the `catch` inside the `if (wantStream)` block, around the `for await` loop). The HTTP status line was already sent as 200, so record the **semantic** outcome code. Replace with:

```ts
    } catch (error) {
      const msg = error instanceof Error ? error.message : String(error);
      if (msg === "MISSING_GEMINI_API_KEY") {
        recordServerError({
          method: req.method,
          route: req.originalUrl ?? "/api/chat/tutor",
          status: 500,
          name: "MISSING_GEMINI_API_KEY",
        });
        if (!clientGone) {
          send("error", { message: "Missing GEMINI_API_KEY in server env" });
        }
        res.end();
        return;
      }
      console.error("Tutor stream failed:", error);
      recordServerError({
        method: req.method,
        route: req.originalUrl ?? "/api/chat/tutor",
        status: 502,
        name: error instanceof Error ? error.name : "Error",
      });
      if (!clientGone) send("error", { message: "AI request failed" });
      res.end();
      return;
    }
```

Everything else in the controller (validation, access checks, persistTurn, SSE happy path) stays **byte-identical**.

### Step A5 — Extend `status.controller.ts`

Add the import:

```ts
import {
  getAiHealth,
  getErrorSummary,
  getErrorSummaryDetailed,
} from "../services/observability.service";
```

**`getAllStatus`** — extend the response (server/db/env logic unchanged):

```ts
export const getAllStatus = (_req: Request, res: Response): void => {
  // Server
  const serverOk = true;

  // Database
  const dbState = mongoose.connection.readyState;
  const dbOk = dbState === 1;

  // Env
  const envChecks = REQUIRED_ENV_VARS.map((key) => ({
    key,
    loaded: key in process.env && process.env[key] !== "",
  }));
  const envOk = envChecks.every((c) => c.loaded);

  // Overall status stays server+db+env only: one failed AI call or a recorded
  // error is information, not an outage — the AI card carries its own badge,
  // and uptime probes keep the stable 200/503 contract.
  const overallOk = serverOk && dbOk && envOk;

  res.status(overallOk ? 200 : 503).json({
    status: overallOk ? "ok" : "degraded",
    timestamp: new Date().toISOString(),
    checks: {
      server: {
        status: serverOk ? "ok" : "error",
        uptime: uptimeFormatted(process.uptime()),
        environment: process.env.NODE_ENV ?? "unknown",
      },
      database: {
        status: dbOk ? "ok" : "error",
        connection: mongoStateLabel(dbState),
      },
      env: {
        status: envOk ? "ok" : "error",
        variables: envChecks,
      },
      ai: getAiHealth(),
      errors: getErrorSummary(),
    },
  });
};
```

**`getServerStatus`** (admin-only) — add two keys to its existing JSON (keep everything already there):

```ts
    ai: getAiHealth(),
    errors: getErrorSummaryDetailed(), // includes err.name — admin eyes only
```

Do **not** touch `getDatabaseStatus` / `getEnvStatus` or `status.routes.ts` (guards stay exactly as they are: `/all` public, `/` admin).

### Step A6 — Tests

**New file `server/test/unit/observability.test.ts`:**

```ts
import { describe, it, expect, beforeEach } from "vitest";
import {
  getAiHealth,
  getErrorSummary,
  getErrorSummaryDetailed,
  recordAiFailure,
  recordAiSuccess,
  recordServerError,
  resetObservabilityForTests,
} from "../../src/services/observability.service";

describe("observability service", () => {
  beforeEach(() => resetObservabilityForTests());

  it("starts empty: ai unknown, zero errors", () => {
    expect(getAiHealth()).toEqual({
      status: "unknown",
      last_success: null,
      last_failure: null,
    });
    const summary = getErrorSummary();
    expect(summary.count_since_boot).toBe(0);
    expect(summary.last_error_at).toBeNull();
    expect(summary.recent).toEqual([]);
    expect(typeof summary.since).toBe("string");
  });

  it("records errors newest-first; public payload has no name", () => {
    recordServerError({ method: "GET", route: "/api/a", status: 500, name: "TypeError" });
    recordServerError({ method: "POST", route: "/api/b", status: 502, name: "Error" });
    const { recent, count_since_boot, last_error_at } = getErrorSummary();
    expect(count_since_boot).toBe(2);
    expect(recent[0].route).toBe("/api/b");
    expect(recent[1].route).toBe("/api/a");
    expect(last_error_at).toBe(recent[0].time);
    expect((recent[0] as Record<string, unknown>).name).toBeUndefined();
    expect((recent[0] as Record<string, unknown>).message).toBeUndefined();
    expect((recent[0] as Record<string, unknown>).stack).toBeUndefined();
  });

  it("detailed payload keeps the error name (admin view)", () => {
    recordServerError({ method: "GET", route: "/api/a", status: 500, name: "TypeError" });
    expect(getErrorSummaryDetailed().recent[0].name).toBe("TypeError");
  });

  it("strips query strings and truncates long routes", () => {
    recordServerError({
      method: "GET",
      route: "/api/content/load?id=123&token=secret",
      status: 500,
      name: "Error",
    });
    expect(getErrorSummaryDetailed().recent[0].route).toBe("/api/content/load");

    recordServerError({
      method: "GET",
      route: "/" + "x".repeat(500),
      status: 500,
      name: "Error",
    });
    expect(getErrorSummaryDetailed().recent[0].route.length).toBeLessThanOrEqual(120);
  });

  it("caps the buffer at 50 but keeps counting", () => {
    for (let i = 0; i < 60; i++) {
      recordServerError({ method: "GET", route: `/api/e${i}`, status: 500, name: "Error" });
    }
    const summary = getErrorSummary();
    expect(summary.recent.length).toBe(50);
    expect(summary.count_since_boot).toBe(60);
    expect(summary.recent[0].route).toBe("/api/e59"); // newest first
  });

  it("ai health transitions unknown → ok → degraded with timestamps", () => {
    recordAiSuccess();
    let health = getAiHealth();
    expect(health.status).toBe("ok");
    expect(health.last_success).toBeTruthy();
    expect(health.last_failure).toBeNull();

    recordAiFailure();
    health = getAiHealth();
    expect(health.status).toBe("degraded");
    expect(health.last_failure).toBeTruthy();
    expect(health.last_success).toBeTruthy(); // success timestamp is kept
  });
});
```

**Extend `server/test/unit/errorHandler.test.ts`** — add (keep the existing two tests untouched; add the imports and a `beforeEach` reset):

```ts
import {
  getErrorSummary,
  getErrorSummaryDetailed,
  resetObservabilityForTests,
} from "../../src/services/observability.service";

// inside describe(): 
beforeEach(() => resetObservabilityForTests());

it("records the error into the ring buffer (route without query, no message)", () => {
  process.env.NODE_ENV = "test";
  const res = mockRes();
  errorHandler(
    new Error("student wrote something private"),
    { method: "POST", originalUrl: "/api/content/load?id=abc" } as never,
    res as never,
    () => {},
  );
  const summary = getErrorSummary();
  expect(summary.count_since_boot).toBe(1);
  expect(summary.recent[0]).toMatchObject({
    method: "POST",
    route: "/api/content/load",
    status: 500,
  });
  expect((summary.recent[0] as Record<string, unknown>).name).toBeUndefined();
  // even the admin view never sees the message:
  expect(
    JSON.stringify(getErrorSummaryDetailed()),
  ).not.toContain("student wrote something private");
});

it("does not crash when req is an empty object (legacy test shape)", () => {
  process.env.NODE_ENV = "test";
  const res = mockRes();
  errorHandler(new Error("x"), {} as never, res as never, () => {});
  expect(getErrorSummary().recent[0].method).toBe("UNKNOWN");
});
```

**New file `server/test/api/statusObservability.test.ts`** (supertest through the real app — the SDK mock pattern is copied from `test/api/tutor.test.ts`):

```ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import request from "supertest";

const { mockGenerateContent } = vi.hoisted(() => ({
  mockGenerateContent: vi.fn(async () => ({ text: "MOCK_AI_REPLY" })),
}));

vi.mock("@google/genai", () => ({
  GoogleGenAI: class {
    models = { generateContent: mockGenerateContent };
  },
}));

import app from "../../src/app";
import { resetObservabilityForTests } from "../../src/services/observability.service";
import { clearLessonContextCache } from "../../src/services/lessonContext.service";
import { seedLesson, seedUser } from "../helpers/seed";

beforeEach(() => {
  mockGenerateContent.mockClear();
  mockGenerateContent.mockResolvedValue({ text: "MOCK_AI_REPLY" });
  clearLessonContextCache();
  resetObservabilityForTests();
});

describe("Tier 1.A — /api/status/all observability payload", () => {
  it("baseline: ai unknown + zero errors, anonymous access works", async () => {
    const res = await request(app).get("/api/status/all");
    expect(res.status).toBe(200);
    expect(res.body.checks.ai).toEqual({
      status: "unknown",
      last_success: null,
      last_failure: null,
    });
    expect(res.body.checks.errors.count_since_boot).toBe(0);
    expect(res.body.checks.errors.recent).toEqual([]);
    expect(typeof res.body.checks.errors.since).toBe("string");
  });

  it("successful tutor call flips ai.status to ok", async () => {
    const { user } = await seedUser();
    const lesson = await seedLesson(user._id, { access_type: "public" });

    const chat = await request(app).post("/api/chat/tutor").send({
      contentId: lesson._id.toString(),
      mode: "free_chat",
      message: "สวัสดี",
    });
    expect(chat.status).toBe(200);

    const res = await request(app).get("/api/status/all");
    expect(res.body.checks.ai.status).toBe("ok");
    expect(res.body.checks.ai.last_success).toBeTruthy();
    expect(res.body.checks.errors.count_since_boot).toBe(0); // success ≠ error
  });

  it("failed tutor call → ai degraded + sanitized ring-buffer entry", async () => {
    // ⚠️ The message must NOT match retry.ts's transient regex
    // (503|429|network|timeout|...) or generateWithRetry sleeps ~10s retrying.
    mockGenerateContent.mockRejectedValue(new Error("boom"));

    const { user } = await seedUser();
    const lesson = await seedLesson(user._id, { access_type: "public" });

    const chat = await request(app).post("/api/chat/tutor").send({
      contentId: lesson._id.toString(),
      mode: "free_chat",
      message: "สวัสดี",
    });
    expect(chat.status).toBe(502);

    const res = await request(app).get("/api/status/all");
    expect(res.body.checks.ai.status).toBe("degraded");
    expect(res.body.checks.ai.last_failure).toBeTruthy();

    const entry = res.body.checks.errors.recent[0];
    expect(entry).toMatchObject({
      method: "POST",
      route: "/api/chat/tutor",
      status: 502,
    });
    expect(entry.name).toBeUndefined();
    expect(entry.message).toBeUndefined();
    expect(JSON.stringify(res.body)).not.toContain("boom");
  });

  it("payload is identical for logged-in users (no privileged leak)", async () => {
    const { token } = await seedUser();
    const res = await request(app)
      .get("/api/status/all")
      .set("Authorization", `Bearer ${token}`);
    expect(res.status).toBe(200);
    expect(res.body.checks.ai).toBeDefined();
    expect(res.body.checks.errors).toBeDefined();
    expect(res.body.checks.errors.recent.every(
      (e: Record<string, unknown>) => e.name === undefined,
    )).toBe(true);
  });

  it("admin GET /api/status/ includes error names", async () => {
    mockGenerateContent.mockRejectedValue(new Error("boom"));
    const { user, token } = await seedUser({ role: "admin" });
    const lesson = await seedLesson(user._id, { access_type: "public" });
    await request(app).post("/api/chat/tutor").send({
      contentId: lesson._id.toString(),
      mode: "free_chat",
      message: "hi",
    });

    const res = await request(app)
      .get("/api/status/")
      .set("Authorization", `Bearer ${token}`);
    expect(res.status).toBe(200);
    expect(res.body.errors.recent[0].name).toBe("Error");
    expect(res.body.ai.status).toBe("degraded");
  });
});
```

Notes for the test run:
- Vitest isolates module state per test file, so the module-level buffer doesn't leak across files; within a file, `resetObservabilityForTests()` in `beforeEach` handles it.
- If `seedLesson`'s fixture is already public, the `{ access_type: "public" }` override is harmless — keep it anyway (explicit beats implicit).

### Step A7 — Run & verify

```bash
cd server && npm test        # all suites green (203 existing + new ones)
cd server && npm run build   # tsc green
```

Manual smoke (optional but nice — the automated tests already cover acceptance):

```bash
cd server && npm run dev
# malformed JSON body → body-parser throws → errorHandler → recorded:
curl -s -X POST http://localhost:5000/api/auth/login -H "Content-Type: application/json" -d "{bad"
curl -s http://localhost:5000/api/status/all
# → checks.errors.count_since_boot ≥ 1, recent[0].route = "/api/auth/login", no message anywhere
```

**Phase 1.A acceptance (from ROADMAP):** trigger a 500 locally → appears in `/api/status/all` with route+code but no message ✓ (test 3 + curl smoke); tutor call flips `ai.status` ✓ (tests 2–3); suite green and still fully offline ✓.

### Step A8 — Docs + commit (in `server/`)

First apply the **server** doc edits from §4.1 (route-table row + services tree line + Gotchas bullet in `server/CLAUDE.md`). Then stage **exactly** these files — nothing else:

```powershell
git add src/services/observability.service.ts src/middlewares/error.middleware.ts src/services/tutor/genaiClient.ts src/controllers/tutor.controller.ts src/controllers/status.controller.ts test/unit/observability.test.ts test/unit/errorHandler.test.ts test/api/statusObservability.test.ts CLAUDE.md
git status        # review: exactly these files staged, nothing unstaged you created
git commit -m "feat: error ring buffer + AI health on /api/status/all (Tier 1.A)"
```

---

## 3. Phase 1.B — Client: Status page v2

**Goal:** `/status` gains an **AI Tutor** card and a **Recent Errors** card fed by the new payload, and the whole page becomes bilingual via `useAppI18n`. Keep the 30 s poll, skeleton, and error states. Phone-first at 390 px. Size **S**, repo: `client/` only.

### Current state you're building on

- `client/src/pages/Status.tsx` — self-contained page: local components `StatusBadge`, `SectionHeader`, `MetaRow`, `ServerCard`, `DatabaseCard`, `EnvCard`, `Skeleton`; polls `useStatusStore.fetch` every 30 s. **All copy is currently English-only.**
- `client/src/stores/status.store.ts` — Zustand store; `fetch()` GETs `/status/all` via the shared `api` axios instance and validates `"checks" in data`.
- i18n: `useAppI18n()` from `@/lib/i18n` returns `{ language, isThai, t }` where `t(english, thai)` picks by the persisted `language.store`. Components call it directly (see `History.tsx`, `Profile.tsx`).
- Relative-time precedent: `formatRelative` in `pages/History.tsx` (lines 27–42) — copy its style, don't import it (it's page-local).
- Test style: `client/src/pages/__tests__/Profile.test.tsx` — happy-dom, `createRoot` + `act`, `vi.mock` per store with a mutable state object. Follow it exactly.

### Step B1 — Extend `client/src/stores/status.store.ts`

Add the new types and extend `checks` (keep every existing type/field):

```ts
export interface AiCheck {
  status: "ok" | "degraded" | "unknown";
  last_success: string | null;
  last_failure: string | null;
}

export interface RecentError {
  time: string;
  method: string;
  route: string;
  status: number;
}

export interface ErrorsCheck {
  count_since_boot: number;
  last_error_at: string | null;
  since: string;
  recent: RecentError[];
}
```

In `AllStatusResponse`:

```ts
  checks: {
    server: ServerCheck;
    database: DatabaseCheck;
    env: EnvCheck;
    ai?: AiCheck;       // optional: page stays resilient if the server is older
    errors?: ErrorsCheck;
  };
```

**Recommended (small, do it):** the server returns **503** when db/env is degraded, and axios rejects non-2xx — so today the page can never *render* a degraded payload. Fix by accepting 503 in the store's fetch:

```ts
const response = await api.get<AllStatusResponse>("/status/all", {
  validateStatus: (s) => s === 200 || s === 503,
});
```

Everything else in `fetch()` (the HTML-fallback check, the `"checks" in rawData` guard, error handling) stays as-is.

### Step B2 — Bilingual pass on `Status.tsx`

Import `useAppI18n` from `@/lib/i18n`. Each component that renders copy calls it directly (they're all React components — hooks are fine).

**`StatusBadge`** — add the `unknown` state and translate labels (drop the `as const` since labels are now computed):

```tsx
function StatusBadge({
  status,
}: {
  status: "ok" | "error" | "degraded" | "unknown" | string;
}) {
  const { t } = useAppI18n();
  const map = {
    ok: {
      dot: "bg-emerald-400",
      text: "text-emerald-400",
      bg: "bg-emerald-400/10",
      border: "border-emerald-400/20",
      label: t("Operational", "ปกติ"),
    },
    degraded: {
      dot: "bg-amber-400",
      text: "text-amber-400",
      bg: "bg-amber-400/10",
      border: "border-amber-400/20",
      label: t("Degraded", "มีปัญหา"),
    },
    error: {
      dot: "bg-red-500",
      text: "text-red-400",
      bg: "bg-red-500/10",
      border: "border-red-500/20",
      label: t("Error", "ผิดพลาด"),
    },
    unknown: {
      dot: "bg-zinc-400",
      text: "text-zinc-400",
      bg: "bg-zinc-400/10",
      border: "border-zinc-400/20",
      label: t("Standby", "รอข้อมูล"),
    },
  };
  const style = map[status as keyof typeof map] ?? map.error;
  // ... JSX unchanged
}
```

**Copy table** — replace each hard-coded string with `t(english, thai)`:

| Where | English (keep as 1st arg) | Thai (2nd arg) |
| --- | --- | --- |
| Page `<h1>` | `System Status` | `สถานะระบบ` |
| Subtitle `<p>` | `Real-time health of all services` | `สุขภาพของทุกระบบแบบเรียลไทม์` |
| Last-check chip | `Last Check` | `เช็กล่าสุด` |
| Refresh button | `Refresh Now` / `Checking...` | `รีเฟรช` / `กำลังเช็ก...` |
| Error alert title | `API Unreachable` | `ติดต่อ API ไม่ได้` |
| Error alert title 2 | `System Communication Error` | `การสื่อสารกับระบบผิดพลาด` |
| Error alert button | `Retry` | `ลองใหม่` |
| Banner | `All systems operational` | `ทุกระบบทำงานปกติ` |
| Banner | `Some systems degraded` | `บางระบบมีปัญหา` |
| Banner | `System error detected` | `พบข้อผิดพลาดในระบบ` |
| Banner sub | `Server Timestamp` | `เวลาเซิร์ฟเวอร์` |
| Empty fallback | `Connection lost. Reconnecting...` | `การเชื่อมต่อหลุด กำลังเชื่อมต่อใหม่...` |
| Empty fallback | `No status data available.` | `ยังไม่มีข้อมูลสถานะ` |
| Footer | `Polling every 30s • Real-time Monitoring` | `รีเฟรชอัตโนมัติทุก 30 วินาที` |
| `ServerCard` title | `Server` | `เซิร์ฟเวอร์` |
| `DatabaseCard` title | `Database` | `ฐานข้อมูล` |
| `EnvCard` title | `Environment` | `สภาพแวดล้อม` |

Technical `MetaRow` labels inside the three existing cards (`Environment`, `Uptime`, `Node.js`, `Platform`, `Hostname`, `Connection`, `Ready state`, `Variables loaded`, `set`, `missing`, `RSS`, `Heap used`, `Heap total`) **may stay English** — they're technical identifiers; don't force-translate them.

### Step B3 — Relative-time helper (local to `Status.tsx`)

```tsx
type Translate = (english: string, thai: string) => string;

function formatRelative(iso: string | null, t: Translate): string {
  if (!iso) return "—";
  const diff = Date.now() - new Date(iso).getTime();
  const mins = Math.floor(diff / 60000);
  if (mins < 1) return t("just now", "เมื่อกี้นี้");
  if (mins < 60) return t(`${mins}m ago`, `${mins} นาทีที่แล้ว`);
  const hrs = Math.floor(mins / 60);
  if (hrs < 24) return t(`${hrs}h ago`, `${hrs} ชม. ที่แล้ว`);
  const days = Math.floor(hrs / 24);
  return t(`${days}d ago`, `${days} วันที่แล้ว`);
}
```

### Step B4 — The two new cards

Add to `Status.tsx` alongside the existing card components. Import the new types: `import type { AiCheck, ErrorsCheck } from "@/stores/status.store";` (extend the existing type-import line).

```tsx
function AiTutorCard({ data }: { data: AiCheck }) {
  const { t } = useAppI18n();
  return (
    <div className="rounded-xl border border-[--color-border] bg-[--color-card] p-5 flex flex-col gap-4">
      <div className="flex items-start justify-between">
        <SectionHeader
          title={t("AI Tutor", "ติวเตอร์ AI")}
          icon={
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              strokeWidth="2"
              strokeLinecap="round"
              strokeLinejoin="round"
            >
              <path d="M12 3l1.9 5.1L19 10l-5.1 1.9L12 17l-1.9-5.1L5 10l5.1-1.9L12 3z" />
            </svg>
          }
        />
        <StatusBadge status={data.status} />
      </div>

      <div>
        <MetaRow
          label={t("Last success", "สำเร็จล่าสุด")}
          value={formatRelative(data.last_success, t)}
          mono
        />
        <MetaRow
          label={t("Last failure", "ล้มเหลวล่าสุด")}
          value={formatRelative(data.last_failure, t)}
          mono
        />
      </div>

      {data.status === "unknown" && (
        <p className="text-xs text-[--color-muted-foreground]">
          {t(
            "No AI calls since the last restart yet.",
            "ยังไม่มีการเรียกใช้ AI ตั้งแต่รีสตาร์ตล่าสุด",
          )}
        </p>
      )}
    </div>
  );
}

function RecentErrorsCard({ data }: { data: ErrorsCheck }) {
  const { t } = useAppI18n();
  const shown = data.recent.slice(0, 6);
  const more = data.recent.length - shown.length;

  return (
    <div className="rounded-xl border border-[--color-border] bg-[--color-card] p-5 flex flex-col gap-4">
      <div className="flex items-start justify-between">
        <SectionHeader
          title={t("Recent Errors", "ข้อผิดพลาดล่าสุด")}
          icon={
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              strokeWidth="2"
              strokeLinecap="round"
              strokeLinejoin="round"
            >
              <path d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
            </svg>
          }
        />
        <StatusBadge status={data.count_since_boot === 0 ? "ok" : "degraded"} />
      </div>

      <MetaRow
        label={t("Since last restart", "ตั้งแต่รีสตาร์ตล่าสุด")}
        value={data.count_since_boot}
        mono
      />

      {shown.length === 0 ? (
        <p className="text-sm text-[--color-muted-foreground] text-center py-4">
          {t("No errors 🎉", "ไม่มีข้อผิดพลาด 🎉")}
        </p>
      ) : (
        <div className="flex flex-col gap-1.5">
          {shown.map((e, i) => (
            <div
              key={`${e.time}-${i}`}
              className="flex items-center gap-2 px-3 py-2 rounded-lg bg-[--color-muted] min-w-0"
            >
              <span className="text-[10px] font-mono text-[--color-muted-foreground] shrink-0">
                {new Date(e.time).toLocaleTimeString()}
              </span>
              <span className="text-[10px] font-mono text-[--color-foreground] truncate flex-1 min-w-0">
                {e.method} {e.route}
              </span>
              <span className="text-[10px] font-mono text-red-400 shrink-0">
                {e.status}
              </span>
            </div>
          ))}
          {more > 0 && (
            <p className="text-[10px] font-mono text-[--color-muted-foreground] text-center">
              +{more}
            </p>
          )}
        </div>
      )}
    </div>
  );
}
```

Wire them into the existing grid (the cards flow into a second row on `md`):

```tsx
<div className="grid grid-cols-1 md:grid-cols-3 gap-4">
  <ServerCard data={data.checks.server} />
  <DatabaseCard data={data.checks.database} />
  <EnvCard data={data.checks.env} />
  {data.checks.ai && <AiTutorCard data={data.checks.ai} />}
  {data.checks.errors && <RecentErrorsCard data={data.checks.errors} />}
</div>
```

The `&&` guards matter: if the payload predates 1.A (or a stale deploy), the page renders exactly as today instead of crashing.

### Step B5 — Tests: new file `client/src/pages/__tests__/Status.test.tsx`

Follow the `Profile.test.tsx` pattern (happy-dom, `createRoot` + `act`, hoisted mutable mock state). Skeleton:

```tsx
// @vitest-environment happy-dom
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import { createRoot, type Root } from "react-dom/client";
import { act } from "react";
import Status from "../Status";
import type { AllStatusResponse } from "@/stores/status.store";

const { statusState, mockFetch } = vi.hoisted(() => ({
  mockFetch: vi.fn(),
  statusState: {
    data: null as unknown,
    loading: false,
    error: null as string | null,
    lastFetched: null as Date | null,
    fetch: vi.fn(),
    reset: vi.fn(),
  },
}));
statusState.fetch = mockFetch;

let language: "en" | "th" = "en";

vi.mock("@/stores/language.store", () => ({
  useLanguageStore: (selector: (s: { language: "en" | "th" }) => unknown) =>
    selector({ language }),
}));

vi.mock("@/stores/status.store", () => ({
  useStatusStore: (selector?: (s: typeof statusState) => unknown) =>
    selector ? selector(statusState) : statusState,
}));

function makeData(
  checksOverrides: Partial<AllStatusResponse["checks"]> = {},
): AllStatusResponse {
  return {
    status: "ok",
    timestamp: new Date().toISOString(),
    checks: {
      server: { status: "ok", uptime: "0h 5m 3s", environment: "test" },
      database: { status: "ok", connection: "connected", ready_state: 1 },
      env: { status: "ok", variables: [{ key: "PORT", loaded: true }] },
      ...checksOverrides,
    },
  } as AllStatusResponse;
}

let container: HTMLDivElement | null = null;
let root: Root | null = null;

function render(ui: React.ReactElement) {
  container = document.createElement("div");
  document.body.appendChild(container);
  root = createRoot(container);
  act(() => {
    root!.render(ui);
  });
  return container!;
}

afterEach(() => {
  act(() => root?.unmount()); // clears the 30 s poll interval
  container?.remove();
  container = null;
  root = null;
});

beforeEach(() => {
  language = "en";
  statusState.data = null;
  statusState.loading = false;
  statusState.error = null;
  statusState.lastFetched = null;
  mockFetch.mockReset();
});
```

Then the cases (write each as an `it`):

1. **Backward compatible:** `statusState.data = makeData()` (no `ai`/`errors`) → renders, `textContent` contains `Server` but **not** `AI Tutor` or `Recent Errors`.
2. **AI ok:** `makeData({ ai: { status: "ok", last_success: new Date(Date.now() - 5 * 60000).toISOString(), last_failure: null } })` → contains `AI Tutor`, `5m ago`, and an `Operational` badge; `Last failure` row shows `—`.
3. **AI degraded:** `status: "degraded"`, both timestamps set → contains `Degraded`.
4. **AI unknown:** `status: "unknown"`, both null → contains `Standby` and `No AI calls since the last restart yet.`
5. **Errors empty state (en + th):** `errors: { count_since_boot: 0, last_error_at: null, since: ..., recent: [] }` → contains `No errors 🎉`; set `language = "th"`, re-render → contains `ไม่มีข้อผิดพลาด 🎉` and `สถานะระบบ` (bilingual page copy works).
6. **Errors list + overflow:** 8 entries `{ time, method: "POST", route: "/api/chat/tutor", status: 502 }` → contains `POST /api/chat/tutor`, `502`, count `8`, and `+2` (6 shown, 2 hidden).

### Step B6 — Run & verify

```bash
cd client && npm test        # all suites green (74 existing + new file)
cd client && npm run lint    # no NEW errors (one pre-existing known error in vite.config.js — leave it; it's fixed in Tier 4)
cd client && npm run build   # green
```

Manual pass (both halves running):

1. Open `http://localhost:5173/status` **in incognito** (Golden Rule 2 — page is public): five cards render, AI card says Standby (no tutor call yet).
2. Open the test lesson, ask the tutor anything → back to `/status` → AI card flips to Operational within one poll (≤30 s or hit Refresh).
3. `curl -s -X POST http://localhost:5000/api/auth/login -H "Content-Type: application/json" -d "{bad"` → Recent Errors card shows the entry; **no error message text anywhere on the page**.
4. Toggle the language switcher → all page copy flips Thai/English.
5. DevTools responsive mode at **390 px**: no horizontal scroll; long routes truncate inside the error rows.

**Phase 1.B acceptance (from ROADMAP):** both new cards render from the live payload in both languages ✓; no horizontal scroll at 390 px ✓; client tests cover the new cards' render logic ✓.

### Step B7 — Docs + commit (in `client/`)

First apply the **client** doc edit from §4.2 (the `/status` routing-table row in `client/CLAUDE.md`). Then stage **exactly** these files — nothing else:

```powershell
git add src/pages/Status.tsx src/stores/status.store.ts src/pages/__tests__/Status.test.tsx CLAUDE.md
git status        # review: exactly these files staged
git commit -m "feat: status page v2 — AI tutor + recent errors cards, bilingual (Tier 1.B)"
```

---

## 4. Documentation updates (required — part of done)

Items 1 and 2 are committed **inside their phase's commit** (Steps A8 / B7). Item 3 lives at the workspace root — outside both git repos, no commit needed.

1. **`server/CLAUDE.md`** (part of the 1.A commit):
   - In the `### /api/status — status.controller.ts` route table (~line 143), the `/all` row currently reads
     `| GET | /all | 🌐 | Aggregated health check (sanitized; anonymous Status page uses this) |`
     → extend the purpose cell to: “Aggregated health check (sanitized; anonymous Status page uses this; includes `ai` health + `errors` ring buffer since Tier 1.A)”.
   - In the architecture tree under `services/`, add: `observability.service.ts  # in-memory error ring buffer + AI health (Tier 1.A; wiped on restart — by design)`.
   - In the `## Gotchas` section (~line 234), add one bullet: “**Observability is in-memory.** The error ring buffer + AI health on `/status/all` reset on every restart (Render free tier restarts often). Never store `err.message` or request bodies in it — names/routes/codes only, and names are admin-only.”
2. **`client/CLAUDE.md`** (part of the 1.B commit): in the routing table (~line 89), the row `| /status | Status | — | System status |` → note cell becomes “System status (v2: AI health + recent errors cards, bilingual)”.
3. **`ROADMAP.md` (workspace root, not in either git repo):** under the Tier 1 heading (§5) add: `> ✅ Shipped <date> — 1.A (server) + 1.B (client).` Also fill in the Execution log at the bottom of this file.

---

## 5. Final done-definition checklist

- [ ] Step 0 preflight done: leftover changes committed separately; baselines were green before any Tier 1 edit
- [ ] `cd server && npm test` green (203 existing + `observability.test.ts` + `statusObservability.test.ts` + extended `errorHandler.test.ts`)
- [ ] `cd server && npm run build` green
- [ ] `cd client && npm test` green (74 existing + `Status.test.tsx`)
- [ ] `cd client && npm run build` green; `npm run lint` introduces no new errors
- [ ] `/api/status/all` anonymous: contains `checks.ai` + `checks.errors`; **zero** occurrences of error messages/stacks/names in the public payload
- [ ] `GET /api/status/` (admin) shows error names; learner still gets 403 (existing test keeps passing)
- [ ] Manual: incognito `/status` renders all five cards; tutor call flips AI card; malformed-JSON curl appears in Recent Errors; Thai/English toggle works; 390 px clean
- [ ] `server/CLAUDE.md`, `client/CLAUDE.md`, `ROADMAP.md` updated; Execution log below filled in
- [ ] One **Tier-1** commit per repo (plus the Step 0 leftover commit in each), **nothing pushed** (owner decision D1)
- [ ] Untouched: `persona.ts`, `parse.ts`, `retry.ts` internals, rate limiting, route guards, `example_*`/`archive`/`graphify-out`

## 6. When to stop and ask the owner

- Step 0 `git status` shows anything beyond the three known leftover files, or foreign files (wrong repo!).
- Any pre-existing test fails at Step 0.
- A source file materially differs from the "current state" this plan describes (e.g. the catch blocks moved, `Status.tsx` was restructured).
- You are tempted to add an env var, a dependency, a DB collection, or to ping Gemini from the status path — all four are design violations, not judgment calls.

---

## Execution log (implementing agent fills this in)

| Step | Status | Notes |
| --- | --- | --- |
| 0. Preflight (leftover commits + baselines) | ✅ | repos clean; server 203/203 baseline (1 flaky test 26); client 74/74 |
| A1–A5. Server implementation | ✅ | observability.service + hooks |
| A6. Server tests written | ✅ | +13 tests (216 total) |
| A7. Server run & verify | ✅ | tests: 216 · build: green |
| A8. Server docs + commit | ✅ | commit: a7405ba |
| B1–B4. Client implementation | ✅ | Status v2 + bilingual |
| B5. Client tests written | ✅ | +6 tests (80 total) |
| B6. Client run & verify | ✅ | tests: 80 · lint: 1 pre-existing · build: green |
| B7. Client docs + commit | ✅ | commit: 4aa100c |
| §4.3 ROADMAP + this log | ✅ | |
| Manual smoke (incognito + 390 px + th/en) | ⬜ | optional — automated tests cover acceptance |
