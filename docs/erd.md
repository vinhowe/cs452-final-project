# LifeLab ERD (draft)

> **Drafted by Claude (Opus 5.5)** on 2026-11-19 from design decisions Vin made in conversation.
> Open questions at the bottom are Vin's to resolve.

LifeLab uses [LiveStore](https://github.com/livestorejs/livestore): an **append-only event log is the system of record**, and the SQL tables below are a **read model** that LiveStore's materializers rebuild from the log on every client (browser SQLite via WebAssembly, with FTS5) and on the server (Node). Single user, self-hosted on a tailnet, one store per user. `?` in a comment means nullable.

The central idea of the read model is a **`DOCUMENT` supertable**: every piece of markdown (journal entry, comment, issue/experiment body, experiment spec, amendment, review) is a row in `DOCUMENT`, and the specific tables share its primary key (class-table inheritance). Search, references, block IDs, revision history, and soft delete are each implemented once, on `DOCUMENT`.

## System of record: the event log

| Column | Type | Notes |
|---|---|---|
| `seq` | int | global order, assigned by the sync backend |
| `parent_seq` | int | the event this client saw last (push is rejected if stale; the client rebases) |
| `name` | text | e.g. `v1.ThreadOpened` |
| `args` | json | event payload |
| `client_id`, `session_id` | text | which device/tab |
| `created_at` | datetime | |

Event catalog (first pass, versioned names so the schema can evolve):

| Event | Payload (abridged) | Materializes into |
|---|---|---|
| `EntryCreated` | id, entryDate, body, checkinId?, threadId? | `document`, `journal_entry`, `reference` |
| `DocumentEdited` | documentId, body | `document`, `document_revision`, `reference` |
| `DocumentDeleted` | documentId | `document.deleted_at` |
| `ThreadOpened` | id, kind, title, bodyDocId, body | `thread` (number assigned in log order), `document` |
| `ThreadRetitled` / `ThreadClosed` / `ThreadReopened` | threadId, title / reason | `thread`, `thread_event` |
| `SpecCreated` | threadId, specDocId, body | `document`, `thread.spec_document_id` |
| `CommentAdded` | id, threadId, body | `document`, `comment` |
| `AmendmentAdded` | id, threadId, body, checkinId? | `document`, `amendment` |
| `AmendmentsFolded` | threadId, revisionBody, amendmentIds, reviewId? | `document_revision`, `amendment.folded_into_revision_id` |
| `AmendmentDismissed` | amendmentId | `amendment.dismissed_at` |
| `ExperimentAdopted` / `ExperimentDowngraded` | threadId, reviewId? | `thread.state`, `thread_event` |
| `CheckinStarted` | id, localDate | `checkin` |
| `CycleStarted` / `CycleEndChanged` | id, start, end | `cycle` |
| `ReviewStarted` / `ReviewCompleted` | id, cycleId, body | `document`, `review` |
| `CommitmentChecked` | reviewId, threadId, result, note? | `commitment_check` |
| `SettingsChanged` | timezone?, checkinTime?, cycleLengthDays? | `settings` |

Push subscriptions are device-local secrets for the server, so they live in a small server-side table, not in the synced log.

## Read model

```mermaid
erDiagram
    DOCUMENT {
        text id PK "client-generated (nanoid)"
        text kind "entry, comment, thread_body, spec, amendment, review"
        text body "markdown with ^block-ids"
        datetime created_at
        datetime updated_at
        datetime deleted_at "? soft delete"
    }
    DOCUMENT_REVISION {
        text id PK
        text document_id FK
        int version "unique per document"
        text body "snapshot"
        text review_id FK "? saved during a review"
        datetime created_at
    }
    REFERENCE {
        text id PK
        text source_document_id FK "where the link was written"
        text target_thread_id FK "? #12"
        text target_document_id FK "? [[date/id]]"
        text target_block_id "? ^k3f9a2"
        text kind "mention or addresses"
    }
    JOURNAL_ENTRY {
        text document_id PK, FK
        date entry_date "local date, shown in links"
        text checkin_id FK "?"
        text thread_id FK "? written about this thread"
    }
    THREAD {
        text id PK "client-generated"
        int number UK "shared #sequence, assigned in log order"
        text kind "issue or experiment"
        text title
        text state "open, adopted, closed"
        text close_reason "? free text, suggested defaults"
        text body_document_id FK, UK
        text spec_document_id FK, UK "? experiments only"
        datetime created_at
        datetime deleted_at "?"
    }
    COMMENT {
        text document_id PK, FK
        text thread_id FK
    }
    AMENDMENT {
        text document_id PK, FK
        text thread_id FK
        text checkin_id FK "?"
        text folded_into_revision_id FK "? null means pending"
        datetime dismissed_at "?"
    }
    THREAD_EVENT {
        text id PK
        text thread_id FK
        text kind "closed, reopened, adopted, downgraded, retitled"
        text detail "?"
        text review_id FK "? decided during review"
        datetime created_at
    }
    CHECKIN {
        text id PK
        date local_date "missed day means no row"
        datetime created_at
    }
    CYCLE {
        text id PK
        date start_date
        date end_date
    }
    REVIEW {
        text document_id PK, FK
        text cycle_id FK, UK
        datetime completed_at "?"
    }
    COMMITMENT_CHECK {
        text id PK
        text review_id FK
        text thread_id FK
        text result "kept, partial, slipped"
        text note "?"
    }
    SETTINGS {
        int id PK "always 1"
        text timezone "IANA, e.g. America/Denver"
        text checkin_time "floating HH:MM"
        int cycle_length_days "default for next cycle"
    }

    DOCUMENT ||--o{ DOCUMENT_REVISION : "history"
    DOCUMENT ||--o{ REFERENCE : "contains links"
    DOCUMENT |o--o{ REFERENCE : "block target of"
    THREAD |o--o{ REFERENCE : "target of"
    DOCUMENT ||--o| JOURNAL_ENTRY : "is a"
    DOCUMENT ||--o| COMMENT : "is a"
    DOCUMENT ||--o| AMENDMENT : "is a"
    DOCUMENT ||--o| REVIEW : "is a"
    DOCUMENT ||--o| THREAD : "body of"
    DOCUMENT |o--o| THREAD : "spec of"
    THREAD ||--o{ COMMENT : has
    THREAD ||--o{ AMENDMENT : "quick fixups"
    THREAD ||--o{ THREAD_EVENT : logs
    THREAD |o--o{ JOURNAL_ENTRY : "written about"
    THREAD ||--o{ COMMITMENT_CHECK : "checked in"
    DOCUMENT_REVISION |o--o{ AMENDMENT : absorbs
    CHECKIN |o--o{ JOURNAL_ENTRY : groups
    CHECKIN |o--o{ AMENDMENT : "written during"
    CYCLE ||--o| REVIEW : "reviewed by"
    REVIEW ||--o{ COMMITMENT_CHECK : includes
    REVIEW |o--o{ DOCUMENT_REVISION : "saved during"
    REVIEW |o--o{ THREAD_EVENT : "decided during"
```

FTS5: one virtual table `document_fts` over `DOCUMENT.body` searches everything at once. LiveStore ships an FTS5-enabled wa-sqlite build (`@livestore/wa-sqlite/dist/fts5/`).

## Key design choices

- **IDs are client-generated strings (nanoid), not autoincrement integers.** Events are created on whichever device you're using before the server orders them, so the database cannot hand out IDs.
- **`THREAD.number` is assigned by the materializer in log order.** Every client replays the same order, so every client computes the same numbers. A thread created on two devices at the same moment could have its number change once after sync (shown as provisional until synced); references store `thread_id`, so links survive.
- **`THREAD` holds both issues and experiments** (single-table inheritance, like a GitHub PR being an issue). A commitment is an experiment with `state = 'adopted'`.
- **`DOCUMENT` is a supertype** for everything containing markdown. A reference's source is always a real foreign key.
- **Spec revisions are `DOCUMENT_REVISION` rows of the experiment's spec document**, and every other document gets edit history for free. No revision messages for now.
- **Pending amendment** = `folded_into_revision_id IS NULL AND dismissed_at IS NULL` (and its document not deleted).
- **Blocks have no table.** Block IDs live in the markdown (`^k3f9a2`, Obsidian-style).
- **The timeline comes from the log and the read model:** state changes in `THREAD_EVENT`, plus comments, revisions, amendments, and backlinks by timestamp.
- **Cycles are stored rows**, so changing the cadence never moves past cycles. Experiments relate to cycles by date overlap.
- **"What was I doing on day X?"** can be answered from the read model (latest revision as of X plus amendments pending at X) or by replaying the log up to X.

## Invariants and where they are enforced

With event sourcing, materializers must be deterministic and cannot reject events, so rules are checked **before an event is committed** (the command layer), against the current read model.

| Rule | Enforcement |
|---|---|
| Only experiments can be adopted or have a spec | command layer; `CHECK` on the read-model table as a backstop |
| Exactly one target per reference | `CHECK` on `REFERENCE` (references are derived by parsing markdown) |
| Adopting requires zero pending amendments | the adopt flow emits `AmendmentsFolded` / `AmendmentDismissed` for every leftover, then `ExperimentAdopted`, committed together; the command refuses to adopt otherwise |
| Version numbers unique per document | assigned by the materializer; `UNIQUE(document_id, version)` |
| One review per cycle | command layer; `UNIQUE(cycle_id)` on `REVIEW` |

## Resolved (Vin's decisions)

- **Storage and sync: LiveStore** (event sourcing, reactive SQLite on each client), TypeScript throughout.
- **Typed links, GitHub-style:** a plain `#12` is a `mention`; `addresses #12` creates an `addresses` link. Adopting an experiment opens a checklist: "close these addressed issues?"
- **Spec revisions are separate from the thread body.** Editing an adopted commitment's spec is allowed. The experiment's full view shows the whole revision timeline, including amendments and when each was folded or dismissed.
- **Leftover amendments:** saving a spec edit nudges you to fold pending amendments but does not block the save. Folding and dismissing are the only ways an amendment closes.
- **Adoption requires a clean spec:** zero pending amendments.
- **No uniqueness on check-in entries.**
- **Changing the cadence mid-cycle asks the user** whether it applies to the current cycle or starts with the next one (if applying it now would end the cycle in the past, end it today).
- **Close reasons:** free text with suggested defaults (abandoned, inconclusive, superseded by #N).

## Open questions (Vin to resolve)

- Which sync backend runs on the tailnet (see the system design).
- When does a save create a `DOCUMENT_REVISION`: every `DocumentEdited` event, or coalesced edits (e.g. the editor emits one event per pause in typing)?
