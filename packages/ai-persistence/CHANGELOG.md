# @tanstack/ai-persistence

## 0.1.0

### Minor Changes

- [#984](https://github.com/TanStack/ai/pull/984) [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a) - Add per-store typer helpers: `defineMessageStore`, `defineRunStore`,
  `defineInterruptStore`, `defineMetadataStore`.

  Each takes a store implementation and returns it typed against the contract, so
  you get autocomplete and checking on the object literal inline — no separate
  `: MessageStore` return annotation. They compose into `defineAIPersistence`,
  which already infers **exact presence**: a store you define is a defined,
  non-optional, autocompleted key on `persistence.stores`, and accessing a store
  you did not define is a compile error.

  ```ts
  import {
    defineAIPersistence,
    defineMessageStore,
    defineRunStore,
  } from '@tanstack/ai-persistence'

  const persistence = defineAIPersistence({
    stores: {
      messages: defineMessageStore({ loadThread, saveThread }),
      runs: defineRunStore({ createOrResume, update, get, findActiveRun }),
    },
  })

  persistence.stores.runs // RunStore (defined)
  persistence.stores.interrupts // compile error — not provided
  ```

- [#984](https://github.com/TanStack/ai/pull/984) [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a) - Server-authoritative reconnect is now automatic and keyed on the thread, not the run.

  A chat's durable identity is its **thread**; run ids are ephemeral (a single turn
  can span several runs via interrupts or tool continuations), so basing reconnect
  on a client-cached run id goes stale the moment a turn rolls to a new run. This
  moves the whole reconnect story onto the stable thread id, resolved by the server.
  - **`RunStore.findActiveRun(threadId)`** — new optional, feature-detected store
    method returning the most recent `'running'` run for a thread. Implemented by
    the in-memory reference backend and covered by the conformance testkit, so any
    adapter that provides it is held to the same invariants (most-recent-running
    wins, thread-scoped, null when idle).
  - **`reconstructChat` now returns `{ messages, activeRun, interrupts }`** (was a
    bare message array): the stored transcript as UI messages, a cursor to an
    in-flight run if one exists, and any pending human-in-the-loop interrupts (tool
    approvals / waits) plus the run they paused. It reads the active run before the
    transcript so observing "no active run" guarantees the transcript is final
    (closing a finish-window race).
  - **`@tanstack/ai-client` hydrates itself on mount.** In server-authoritative
    mode (`persistence: true`) the client caches no transcript and no run
    pointer: on mount `useChat`/`ChatClient` calls the connection's new
    `hydrate(threadId)` (a JSON GET against the same endpoint), paints the returned
    transcript, and — if a run is in flight — tails it via the existing `joinRun`
    durability replay. A reload and the same thread opened on another device are the
    identical, server-resolved path. No loader, no `initialMessages`, no
    `initialResumeSnapshot`, no app-side fetching required.
  - **Interrupts reconstruct from the server too.** A paused approval (a tool with
    `needsApproval`) is restored from `reconstructChat`'s `interrupts` exactly as a
    persisted resume snapshot would be, so a reload — or another device — re-prompts
    the same approve/reject decision and resumes the run it paused. Previously the
    pending interrupt was only recoverable from client storage, so a fresh client
    showed the paused tool call with no way to resolve it.

  Apps keep the single GET endpoint they already have (durability replay when a
  resume cursor is present, else `reconstructChat`); everything else is handled by
  the hook.

- [#984](https://github.com/TanStack/ai/pull/984) [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a) - Move multi-instance **locks** to `@tanstack/ai` under a dedicated `@tanstack/ai/locks` subpath, and nest persistence agent skills like `ai-core`.
  - **`LockStore` / `InMemoryLockStore` / `LocksCapability` / `getLocks` / `provideLocks` / `withLocks`** live in `@tanstack/ai/locks` (not the main `@tanstack/ai` barrel, and not `@tanstack/ai-persistence`).
  - `@tanstack/ai-sandbox` consumes the core `LocksCapability` token (no local lock re-export).
  - The locks agent skill moves with the code: `ai-core/locks` in `@tanstack/ai`, not `ai-persistence/locks`.
  - Agent skills under `@tanstack/ai-persistence` nest as `skills/ai-persistence/{stores,server,build-*-adapter}/`.
  - Docs: locks guide under advanced middleware.

- [#984](https://github.com/TanStack/ai/pull/984) [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a) - Add server-side persistence for `chat()`: durable thread messages, run records, and interrupts.

  `withPersistence(persistence)` is a chat middleware that stores the conversation transcript, tracks each run's status, and records interrupt state so a paused run (tool approval, client-tool execution, generic interrupt) survives a server restart.

  `@tanstack/ai-persistence` ships the **contract**, not a backend for your database:
  - The four store interfaces — `MessageStore`, `RunStore`, `InterruptStore`, `MetadataStore` — with the invariants the middleware depends on (full-replace `saveThread`, idempotent `createOrResume`, insert-if-absent interrupt `create`, `requestedAt`-ascending listings).
  - The `withPersistence` / `withGenerationPersistence` middleware, plus `composePersistence` to assemble stores that live in different systems.
  - `memoryPersistence()`, an in-process reference backend for dev and tests.
  - `LockStore` / `withLocks` / `InMemoryLockStore` for cross-worker coordination — deliberately **not** a state store, and not composable through `composePersistence`.
  - A shared conformance testkit at `@tanstack/ai-persistence/testkit`. `runPersistenceConformance` exercises every method of every store you provide and fails loudly on a store that is missing without being declared in `skip`.

  Implement the stores against whatever database you already run and hand the result to `withPersistence` — the core never inspects your tables, so the schema stays yours. The [Build Your Own Adapter](https://tanstack.com/ai/latest/docs/persistence/build-your-own-adapter) guide walks through a complete `node:sqlite` backend end to end, and the package ships Agent Skills with worked Drizzle, Prisma, and Cloudflare D1 recipes (`npx @tanstack/intent@latest install`). `examples/ts-react-chat` runs on a self-contained `node:sqlite` adapter built this way and verified by the conformance testkit.

  Resume reconstruction is delegated to the chat engine: persistence records interrupts and gates new input on a thread with pending interrupts, while the engine rebuilds the resume tool state from the resume batch and the interrupt bindings carried in the (server-loaded) message history.

  `reconstructChat(persistence, request)` is a server helper that returns a thread's stored messages as a JSON `Response`, so a server-authoritative client can hydrate its transcript on load from a one-line `GET` handler.

- [#1004](https://github.com/TanStack/ai/pull/1004) [`1120f0f`](https://github.com/TanStack/ai/commit/1120f0f8824262b4fd1d3788e606793158d6ac3c) - `RunStore.findActiveRun` is now **required**. It was optional and
  feature-detected (`store.findActiveRun?.(threadId)`), which meant an adapter that
  had not implemented it was indistinguishable from one reporting "nothing is
  running": `reconstructChat` returned `activeRun: null`, and a client reloading
  mid-generation silently never reconnected to the run still producing. That is a
  production failure the type system was in a position to catch.

  Adapters that already implement `findActiveRun` need no change. Adapters that do
  not will now get a compile error; implement it as "most recent `'running'` run
  for the thread, `null` if none" — in SQL,
  `WHERE thread_id = ? AND status = 'running' ORDER BY started_at DESC LIMIT 1`.
  A backend that genuinely has no run lifecycle should declare
  `ChatTranscriptStores` and omit `runs` entirely rather than stub the method.

  The store-contract evolution policy changes to match: new store methods are
  added as required, and capability tiers are expressed at the store level, not by
  optional methods. The conformance testkit no longer skips its `findActiveRun`
  assertions when the method is absent.

- [#984](https://github.com/TanStack/ai/pull/984) [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a) - Make a mid-stream reload resume the same conversation cleanly.
  - `withPersistence` now persists the pending turn at the start of a run (so a
    reload during generation still shows the user's message), stamps each
    assistant turn with its stream `messageId`, and accepts
    `withPersistence(persistence, { snapshotStreaming: true })` to also persist the
    in-progress reply on a throttled interval (`snapshotIntervalMs`, default
    `1000`) for partial-output durability.
  - `ModelMessage` gains an optional `id`; `modelMessagesToUIMessages` preserves
    it, so a hydrated message keeps the same identity as its live stream.
  - On reload, the chat client rebuilds an in-flight assistant turn from the
    delivery log (replaying from the start and applying the buffered backlog in one
    batch) instead of reconciling against the persisted partial, so the reload
    shows one clean bubble that catches up and continues rather than a frozen or
    duplicated partial.

### Patch Changes

- Updated dependencies [[`3301398`](https://github.com/TanStack/ai/commit/330139878958fc5c5c167a69347c884fa35b792a), [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a), [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a), [`3301398`](https://github.com/TanStack/ai/commit/330139878958fc5c5c167a69347c884fa35b792a), [`3301398`](https://github.com/TanStack/ai/commit/330139878958fc5c5c167a69347c884fa35b792a), [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a), [`478a4da`](https://github.com/TanStack/ai/commit/478a4da3756e0de09548f2902da3b45748c27b52), [`347b61b`](https://github.com/TanStack/ai/commit/347b61bc788bb816bbd12287c1a426ca7def00f4), [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a), [`7c7aa09`](https://github.com/TanStack/ai/commit/7c7aa09a7402b45e6285ebc78a606131aec3e288), [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a), [`4ce7600`](https://github.com/TanStack/ai/commit/4ce7600d5b543d4b7e3bd6d63cdf5ecf91cdeeaa), [`4ab149f`](https://github.com/TanStack/ai/commit/4ab149fd46a1cf55691266cdd118fdc9999c0b2a)]:
  - @tanstack/ai@0.43.0
