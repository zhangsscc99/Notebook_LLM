# OpenClaw Memory System Explanation

## Summary

In OpenClaw, memory is not hidden model state. It is an explicit, file-backed memory system.

At a high level it works like this:

1. Important information is written to disk.
2. Memory files are chunked and indexed.
3. `memory_search` and `memory_get` retrieve relevant pieces later.
4. The system is plugin-based, and the default plugin is `memory-core`.

---

## What memory stores

According to the docs:

- `MEMORY.md` — long-term memory
- `memory/YYYY-MM-DD.md` — daily notes
- `DREAMS.md` — optional dreaming / review output

Relevant source:

- `docs/concepts/memory.md:9-24`

The docs also state that there is no hidden memory state: only what is written to disk is remembered.

Relevant source:

- `docs/concepts/memory.md:9-11`

---

## How memory fits into the architecture

Memory is implemented as a plugin slot, not as a hardcoded built-in subsystem.

### Entry point

- `src/plugins/memory-runtime.ts:9-19`
- `src/plugins/memory-runtime.ts:56-75`

This code reads `plugins.slots.memory`, resolves the active memory plugin, and asks that plugin for a runtime.

### Memory plugin interface

- `src/plugins/memory-state.ts:97-110`

A memory plugin provides at least these capabilities:

- `getMemorySearchManager(...)`
- `resolveMemoryBackendConfig(...)`
- `closeAllMemorySearchManagers()`

So the core system talks to a stable interface and does not need to know whether the backend is SQLite, QMD, or something else.

---

## Default implementation: `memory-core`

The default implementation is the `memory-core` plugin.

Relevant source:

- `extensions/memory-core/index.ts:170-220`

This plugin registers:

- memory capability
- `memory_search`
- `memory_get`
- `openclaw memory ...` CLI commands
- memory flush plan
- public artifacts

So `memory-core` is the default memory engine for the system.

---

## Core runtime object: `MemorySearchManager`

The main abstraction is `MemorySearchManager`.

Relevant source:

- `packages/memory-host-sdk/src/host/types.ts:85-110`

Its main methods are:

- `search(...)`
- `readFile(...)`
- `status()`
- `sync(...)`
- `probeEmbeddingAvailability()`
- `probeVectorAvailability()`
- `close()`

You can think of this as the unified controller for memory indexing, search, reading, and health checks.

---

## Memory backends

Backend resolution happens here:

- `packages/memory-host-sdk/src/host/backend-config.ts:363-452`

The main modes are:

- `builtin`
- `qmd`

### Builtin

This is the built-in SQLite-based backend.

### QMD

This is a more advanced backend that supports collections, extra paths, sessions, and update policies.

---

## How memory files are discovered

File discovery is implemented here:

- `packages/memory-host-sdk/src/host/internal.ts:92-100`
- `packages/memory-host-sdk/src/host/internal.ts:139-216`

The system recognizes:

- `MEMORY.md`
- `DREAMS.md`
- files under `memory/`
- configured `extraPaths`

It also:

- skips symlinks
- deduplicates by realpath
- skips repair helper directories

So memory file collection is intentionally constrained and safety-aware.

---

## How files become searchable memory

This happens in two main stages.

### 1. Chunking

Relevant source:

- `packages/memory-host-sdk/src/host/internal.ts:362-465`

Markdown content is split into chunks, each with:

- `startLine`
- `endLine`
- `text`
- `hash`
- `embeddingInput`

This means search is done over chunks, not whole files.

### 2. Indexing

Relevant source:

- `packages/memory-host-sdk/src/host/memory-schema.ts:12-89`

Important database tables include:

- `files`
- `chunks`
- `embedding_cache`
- FTS virtual table

So the builtin backend stores file metadata, chunk text, cached embeddings, and a full-text search index.

---

## How search works

The most important search logic is here:

- `extensions/memory-core/src/memory/manager.ts:302-491`

The flow is roughly:

1. If the index is missing or empty, try `sync(...)`.
2. Normalize and preflight the query.
3. Initialize embedding provider if needed.
4. Run search.
5. Return results.

### Two retrieval paths

#### Keyword / FTS path

- `extensions/memory-core/src/memory/manager.ts:368-431`

If no embedding provider is available, the system falls back to FTS-only search.

#### Hybrid retrieval path

- `extensions/memory-core/src/memory/manager.ts:434-490`

If embeddings are available, it performs:

- keyword search
- vector search
- hybrid merge

The docs describe the same architecture:

- `docs/concepts/memory-search.md:58-80`

Meaning:

- vector search handles semantic similarity
- BM25 / FTS handles exact term matching

---

## How vector search works

Vector search implementation is here:

- `extensions/memory-core/src/memory/manager-search.ts:121-215`

If `sqlite-vec` is available, it uses native vector KNN search.
If not, it falls back to reading stored embeddings from the `chunks` table and computing cosine similarity in process.

So vector search gracefully degrades instead of completely failing.

---

## Why `memory_get` is relatively safe

File reading logic is here:

- `packages/memory-host-sdk/src/host/read-file.ts:48-133`

This code enforces several restrictions:

- the path must be inside allowed memory locations
- workspace reads are limited to memory-related paths
- `extraPaths` are validated too
- arbitrary path traversal is blocked
- reads are bounded by line and character limits

Continuation/truncation behavior is implemented here:

- `packages/memory-host-sdk/src/host/read-file-shared.ts:3-114`

So `memory_get` is a controlled excerpt reader, not an unrestricted file reader.

---

## How memory enters model context at startup

Startup memory context is handled here:

- `src/auto-reply/reply/startup-context.ts:7-15`
- `src/auto-reply/reply/startup-context.ts:17-30`
- `src/auto-reply/reply/startup-context.ts:91-111`
- `src/auto-reply/reply/startup-context.ts:202-220`

It limits:

- max bytes per file
- max characters per file
- total injected characters
- how many recent daily memory files are included

This is important because memory can be large on disk, but only a bounded subset is injected into the model prompt.

---

## Automatic memory flush before compaction

Before compaction, OpenClaw can run a memory flush pass.

Relevant source:

- `extensions/memory-core/src/flush-plan.ts:10-23`
- `extensions/memory-core/src/flush-plan.ts:95-139`

This flush plan tells the system to:

- write durable information into `memory/YYYY-MM-DD.md`
- append instead of overwrite
- avoid editing read-only files like `MEMORY.md`, `DREAMS.md`, and `AGENTS.md`

Execution path:

- `src/auto-reply/reply/agent-runner-memory.ts:696-820`

So the design is:

> before compressing the active context, save important information to disk so it is not lost.

---

## One-sentence summary

OpenClaw memory is a plugin-based system that stores memory in files, chunks and indexes those files, searches them with hybrid retrieval, reads them back safely on demand, and can automatically flush useful context to disk before compaction.

---

## Good files to read next

Recommended reading order:

1. `docs/concepts/memory.md`
2. `extensions/memory-core/index.ts`
3. `src/plugins/memory-runtime.ts`
4. `packages/memory-host-sdk/src/host/types.ts`
5. `packages/memory-host-sdk/src/host/internal.ts`
6. `packages/memory-host-sdk/src/host/memory-schema.ts`
7. `extensions/memory-core/src/memory/manager.ts`
8. `extensions/memory-core/src/memory/manager-search.ts`
9. `packages/memory-host-sdk/src/host/read-file.ts`
10. `extensions/memory-core/src/flush-plan.ts`
