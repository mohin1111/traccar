# Annotation Layer

This directory is a **read-only knowledge layer** for the source tree. It mirrors the structure of `src/main/java/org/traccar/` so each source file has a sibling `<filename>.md` describing it in LLM-friendly terms.

## Why a parallel tree?

- **Upstream-sync clean.** Upstream Traccar ships monthly releases. If we commented inside the source files, every upstream pull would conflict in every commented file. The annotation layer lives outside the upstream paths, so `git pull upstream master` merges without friction.
- **Higher signal than line comments.** Annotations focus on the *why*, the *where it fits*, the *cross-references*, and the *gotchas* — what an LLM (or new contributor) needs and what good code can't self-document.
- **Cheap to regenerate.** When a file is heavily refactored upstream, re-read it and rewrite the annotation. No painful merge.

## Scope

The server is ~1,500 Java files. This annotation layer covers the **critical-path packages** — the ones Transport OS will read, extend, or depend on (see parent [`CLAUDE.md`](../../CLAUDE.md)):

**Per-file annotations (L1):**
- root files (`org/traccar/*.java`) — boot, DI, Netty pipeline, protocol base classes
- `api/` — REST resources + WebSocket + security
- `handler/` — the position-processing pipeline + event detectors
- `storage/` — the custom JDBC ORM
- `session/` — device session lifecycle + cache + motion/overspeed state
- `model/` — the domain entities
- `protocol/` — base classes + the `Its*` files only (AIS-140 / Indian VLT)

**Package-level annotations only (L2):**
- `config/`, `database/`, `broadcast/`, `command/`, `forward/`, `geocoder/`, `geofence/`, `geolocation/`, `helper/`, `mail/`, `mapmatcher/`, `media/`, `notification/`, `notificators/`, `reports/`, `schedule/`, `sms/`, `speedlimit/`, `web/`

**Not annotated:** the 261 non-`Its` protocol decoders in `protocol/`. They all follow the same pattern — see `protocol/PACKAGE.md` for the pattern, and the `Its*` annotations for a worked example.

## Structure

```
.claude/annotations/
├── README.md
└── src/main/java/org/traccar/
    ├── <RootFile>.java.md
    ├── PACKAGE.md             ← root-package overview (boot + pipeline)
    ├── api/
    │   ├── PACKAGE.md
    │   ├── resource/PACKAGE.md
    │   ├── security/PACKAGE.md
    │   └── <File>.java.md
    ├── handler/PACKAGE.md  (+ events/, network/ PACKAGE.md)
    ├── storage/PACKAGE.md  (+ query/ PACKAGE.md)
    ├── session/PACKAGE.md  (+ cache/, state/ PACKAGE.md)
    ├── model/PACKAGE.md
    ├── protocol/PACKAGE.md
    └── <supporting-package>/PACKAGE.md
```

## Per-file annotation format (L1)

```
# <filename>

**Role:** 1-2 sentence one-liner.
**Fits in:** where in the system / who calls this.
**Read next:** [[other-file]] — cross-references (use [[name]] = annotation filename without .md/path).

## Public API
- `ClassName` / `methodName()` (lines X-Y) — what it exports

## Key flows
Stepwise narrative of non-obvious flows.

## Gotchas / non-obvious
Invariants, threading, reflection tricks, Guice wiring, surprising patterns.

## Line index
Jump-list of important lines.
```

## Per-package annotation format (L2)

Each `PACKAGE.md` covers directory purpose, file index table, dependency graph, multi-file flows, conventions, and "how to add X" recipes.

## Conventions

- Cross-references use `[[name]]` matching the annotation filename without `.md`/path.
- Line refs use `<file>:<line>` notation.
- Keep it terse — readable in under 90 seconds.
- Descriptive, not prescriptive — design decisions belong in the parent `CLAUDE.md`.

## Maintenance

When upstream changes a file significantly, regenerate the annotation rather than patching. Stale annotations are worse than none.

## Top-of-module pointers

- Parent module CLAUDE.md: [`../../CLAUDE.md`](../../CLAUDE.md)
- Top-level traccar index: [`../../../CLAUDE.md`](../../../CLAUDE.md)
