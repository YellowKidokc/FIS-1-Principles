# File Intelligence System: Architectural Principles

> **Status:** conceptual architecture and decision record. This document deliberately
> defines the system before selecting an implementation or workflow engine.

The proposed policy/assurance layer that applies these principles to the three
workbooks is documented in [POLICY_ASSURANCE_ARCHITECTURE.md](POLICY_ASSURANCE_ARCHITECTURE.md).
It is an architectural crosswalk only; no detectors or mutation paths are implemented.

## Executive decision

**BUILD A SIMPLIFIED VERSION.**

FIS should be a **typed capability registry plus a small, inspectable workflow
language**, not a collection of uniformly shaped scripts and not a taxonomy encoded
into permanent IDs. Its durable unit is a versioned semantic contract. Implementations
are replaceable adapters. Workflows are immutable, decomposable graphs of capability
invocations. Policy controls whether a graph may run; an event log records what did.

The central design choice is to keep four concerns orthogonal:

| Plane | Question | Durable object |
|---|---|---|
| Semantics | What can the system do? | Capability contract |
| Execution | How is it performed here? | Implementation binding |
| Orchestration | In what reasoning/process is it used? | Workflow graph |
| Governance | May it run, and what happened? | Policy decision and receipt |

Trying to compress all four into a code such as `5GA04` creates an attractive but
brittle illusion of order. FIS should instead offer a stable symbolic identity, a
mutable semantic address for orientation, and facets for cross-cutting discovery.

---

## 1. Problem restatement

The real problem is **semantic interoperability at registry scale**: independently
implemented operations must be discoverable, type-compatible, safely composable,
replaceable, explainable, and auditable across many user interfaces and runtimes.

This is a real problem, but most of its mechanisms are established ideas: Unix
filters, typed functions, package registries, dataflow/DAG engines, policy engines,
and event provenance. FIS should not rebuild their runtimes. Its useful contribution
is a file-intelligence vocabulary and a human-readable contract/profile joining those
ideas. The danger is ontology work becoming a substitute for working capabilities.

FIS is therefore best understood as a **control plane**, not one giant application:

```text
Registry (meaning) ──compile──> Plan (typed graph) ──execute──> Receipts (facts)
       │                         ▲       │                       │
       └── implementations ─────┘       └── policy decisions ──┘
```

## 2. Architectural invariants

1. **One capability, one externally observable responsibility.** Atomicity is judged
   by contract and effects, not lines of code or process boundaries.
2. **Identity denotes meaning, never an implementation.** Changing Python to Rust
   does not change identity; changing promised semantics does.
3. **Inputs, outputs, errors, and effects are explicit and machine-checkable.** No
   hidden filesystem, network, clock, model, or database access.
4. **Data compatibility, not taxonomy adjacency, governs composition.** A capability
   may follow another when ports/types and declared constraints agree.
5. **Every composite remains recursively inspectable to atomic invocations.** A
   published composite may be called as one node but never becomes falsely atomic.
6. **Planning is separate from execution.** A resolved plan pins contract,
   implementation, configuration, model/data, and policy versions before execution.
7. **Effects are least-privilege and policy-mediated.** Mutation is not authorized
   merely because a workflow mentions it.
8. **Meaning is immutable within a major contract version.** Breaking semantic,
   schema, or effect changes create a new major version; old versions remain
   decodable.
9. **Every invocation emits a receipt, including skips, denials, retries, and
   failures.** Receipts form a causally linked run graph, not mutable log prose.
10. **Human labels and taxonomy placement may evolve without breaking identity.**
11. **The registry is the source of declared truth; receipts are the source of
    execution truth.** Documentation is rendered from both rather than duplicated.
12. **The core stays domain-neutral.** NLP, SQL, shell, models, and file operations
    use the same ports/effects protocol; domain-specific details live in schemas and
    facets.

## 3. Canonical ontology

### 3.1 Do not use a single tree as the ontology

`Acquire → Identify → Extract → ...` mixes lifecycle phase, intent, mechanism, and
effect. For example, OCR both extracts text and transforms pixels; hashing can
identify, relate, or verify; writing a sidecar both records and outputs. A capability
can legitimately occupy several branches. A single tree forces arbitrary placement.

Use a **faceted ontology**. Give each capability one primary intent for quick browsing,
then independent facets. The browse tree is a projection over these facets, not
identity.

### 3.2 Primary intent vocabulary

The following verbs partition the promised result rather than prescribing a linear
pipeline:

| Intent | Promise | Examples |
|---|---|---|
| `sense` | Observe or acquire facts without interpreting them | scan directory, read metadata |
| `derive` | Deterministically derive a representation or fact | hash, parse, extract text |
| `infer` | Estimate uncertain meaning | classify domain, summarize, embed |
| `relate` | Establish relationships among entities | duplicates, similarity, citations |
| `propose` | Produce a candidate decision or change | filename, destination, keeper |
| `effect` | Change state outside the invocation | move, convert, write, delete |
| `assure` | Test a claim, invariant, or outcome | validate, collision check, verify move |
| `publish` | Make information available to another boundary | index, export, notify |

`record` is not a normal capability phase: the runtime records every invocation.
Domain-specific record writing (for example, a user-visible sidecar) remains an
`effect` or `publish` capability.

### 3.3 Independent facets

Every entry can be filtered by:

- **subject:** `file`, `document`, `claim`, `image`, `audio`, `video`, `collection`,
  `index`, `receipt`;
- **representation:** MIME type/schema, not informal “Excel” buckets;
- **technique:** `filesystem`, `cryptographic`, `parser`, `nlp`, `llm`, `sql`, etc.;
- **effect class:** `pure`, `read`, `recommend`, `write`, `destructive`;
- **assurance:** deterministic/probabilistic, reversibility, idempotency,
  transactionality;
- **cost/requirements:** latency class, compute, network, secrets, model, platform;
- **domain:** finance, legal, science, personal, or any controlled domain vocabulary.

Families are saved queries over facets. Thus “all supported document text extractors”
can expand into atomic implementations without creating an indivisible mega-operation.

## 4. Addressing system

### 4.1 Options considered

| Scheme | Orientation | Stability | Verdict |
|---|---:|---:|---|
| Serial/UUID only | Poor | Excellent | Use internally, never alone |
| Encoded hierarchical ID (`5GA04`) | Good initially | Poor under reclassification | Reject as canonical identity |
| Hierarchical path + UUID | Good | Good | Viable, but paths still imply one parent |
| Namespaced symbolic ID | Very good | Good with governance | Use as public identity |
| Facets only | Excellent search | Poor exact reference | Use for discovery, not identity |
| **Symbolic ID + opaque key + facets** | **Excellent** | **Excellent** | **Adopt** |

### 4.2 Three-part identity

Example:

```text
Reference:  fis.infer.claim.maturity@2
Registry key: cap_01J7M6Y8QMR2Z6R4B4D7M7Q8N1
Browse address: INFER / CLAIM / CLASSIFICATION / MATURITY
```

- The **reference** is a governed, memorable semantic name plus contract major.
- The **registry key** is immutable, opaque, and never reused.
- The **browse address** is generated from current facets and may change freely.

Aliases redirect renamed symbolic references. They never silently redirect across an
incompatible major version. A capability with two natural homes has two browse paths
but one key. Inserting taxonomy levels changes projections, not references. Materially
changing meaning creates `@3`; changing implementation only changes a binding version.

For compact displays, the registry may issue a non-semantic checksum-backed short code
such as `C7-K4P9`. It is a lookup convenience, not a miniature ontology. The decrypter
should never pretend a five-character code can communicate a many-dimensional concept.

## 5. Universal capability schema

### 5.1 Mandatory contract

```yaml
kind: Capability
key: cap_01J7M6Y8QMR2Z6R4B4D7M7Q8N1
ref: fis.infer.claim.maturity@2
title: Classify claim maturity
summary: Assigns a maturity label and calibrated confidence to one claim.
intent: infer
ports:
  inputs:  {claim: fis.schema.claim@1}
  outputs: {classification: fis.schema.claim-maturity@2}
errors: [invalid_input, unavailable, no_result]
effects:
  class: read
  resources: [model_service]
assurance:
  determinism: probabilistic
  idempotency: conditional
contract_version: 2.1.0
```

The mandatory minimum is: identity, title/summary, intent, typed ports, declared error
surface, effects, assurance, and contract version. “No effects” must be stated rather
than inferred.

### 5.2 Strongly recommended

- preconditions and postconditions;
- data classification/privacy constraints;
- cost, latency, and resource hints;
- confidence/calibration semantics for inference;
- examples and counterexamples (“does not do”);
- compatibility/deprecation information;
- owner, maturity status, and support boundary;
- default receipt redaction rules;
- one or more implementation bindings with artifact digest, supported platforms, and
  health status.

### 5.3 Optional

- synonyms, localization, citations, UI hints, icons;
- common predecessor/successor suggestions (never hard dependencies);
- domain and technique facets;
- compensation capability and preview capability;
- conformance fixtures, benchmarks, service-level objectives;
- memberships in curated families and example workflows.

Risk is **derived**, not one vague manually assigned label. Policy evaluates effect
class, data sensitivity, reversibility, scope/cardinality, environment, actor, and
confidence. The same `move` capability may be low risk in a temporary directory and
high risk over a shared archive.

## 6. Universal execution anatomy

The 00–12 script layout standardizes source-file appearance, not execution semantics.
It also bakes `NLP_ROUTE` into a supposedly universal design. Repeating imports,
configuration, logging, handoff, archive, and `main` hundreds of times guarantees
drift.

Replace it with a runtime protocol:

```text
resolve → authorize → validate → invoke → validate result → commit/compensate → emit receipt
```

| Existing section | Disposition |
|---|---|
| `00_IMPORTS` | Implementation detail; no architectural section |
| `01_CONSTANTS` | Package with implementation; contract values live in registry |
| `02_CONFIG` | Runtime injects typed, resolved configuration |
| `03_LOGGING` | Shared structured telemetry/receipt library |
| `04_INGEST` | Typed input adapter at system boundaries |
| `05_VALIDATE` | Generated/shared schema validation plus custom predicates |
| `06_NLP_ROUTE` | Remove; ordinary workflow routing or implementation selection |
| `07_PROCESS` | Keep: the only capability-specific operation |
| `08_ARTIFACTS` | Shared artifact store, declared through output ports |
| `09_WORKFLOW` | Remove from atomic capability; orchestrator owns composition |
| `10_HANDOFF` | Output ports and orchestrator edges |
| `11_ARCHIVE` | Separate effect/publish capability or retention policy |
| `12_MAIN` | Thin generated adapter for a chosen runtime |

An implementation needs only an adapter resembling
`invoke(context, typed_inputs) -> typed_outputs | typed_error`. The context provides
explicit clients for permitted effects, cancellation, secrets, artifacts, and receipt
events. CLI, Python, container, SQL, API, LLM, and MCP bindings conform through
adapter-specific manifests and common conformance tests.

## 7. Composition algebra

### 7.1 Minimal core

Use five graph constructs:

1. **`call`** — invoke an atomic capability or a published composite.
2. **`pipe`** — bind typed outputs/values to later inputs; ordinary edges imply
   dependency and sequence.
3. **`choose`** — select zero or one branch from a total, pure expression.
4. **`map`** — apply a subgraph to a finite or streamed collection with bounded
   concurrency.
5. **`wait`** — suspend until an explicit signal, time, or approval decision.

A workflow is a typed directed graph built from those nodes. Parallelism needs no
special construct: independent ready nodes run concurrently. A join is a node with
multiple inbound bindings. Filtering is `choose` inside `map`. Stop is a terminal
outcome. Triggering is outside the graph and creates a run with inputs.

### 7.2 Failure policy is annotation, not control-flow proliferation

Each `call` or subgraph may declare:

```yaml
timeout: 30s
retry: {max_attempts: 3, backoff: exponential, on: [unavailable]}
on_error:
  unavailable: {fallback: local_binding}
  invalid_input: {emit: rejected}
compensate_with: fis.effect.file.restore@1
```

Retries require declared retry-safe/idempotent behavior or an idempotency key.
Fallback must preserve the same output contract and record the selected binding.
Compensation is a new forward action, not magical rollback. True rollback exists only
inside a declared transactional boundary supported by the underlying resource.
Timeout requests cancellation; receipts distinguish cancelled, timed out, and
eventually completed external work.

Approval is `wait`, followed by `choose`; policy may require it but approval itself
does not grant permissions. Loops are `map` for data and an explicitly bounded
`repeat` macro (`choose + back-edge`) for convergence. Unbounded cycles are rejected.

### 7.3 One grammar for rules and workflows

Fields, operators, and values form a **pure typed expression language**. Capabilities
are actions. `when/then` is syntax sugar for a trigger plus `choose`; guardrails are
policy, not workflow branches:

```yaml
trigger: {event: file.discovered, bind: {file: event.file}}
graph:
  - choose:
      when: all:
        - {in: [{field: file.media_type}, {set: DOCUMENT_TYPES}]}
        - {gt: [{field: file.age_days}, 180]}
      then:
        - call: fis.derive.file.sha256@1
        - call: fis.relate.file.exact-duplicates@1
        - choose:
            when: {field: duplicate.found}
            then: {use: quarantine_duplicate_safely@3}
policy: fis.policy.archive-mutation@4
```

Operators cannot invoke effects. Expressions cannot smuggle arbitrary code. The type
checker verifies field paths and operator types before a workflow is publishable.

## 8. Hierarchical reuse

Reduce five overlapping nouns to three:

| Model term | Meaning | Executable? |
|---|---|---:|
| **Capability** | Atomic semantic contract | Yes |
| **Family** | Dynamic saved query or curated set of capabilities | No |
| **Workflow** | Versioned typed graph; may invoke workflows recursively | Yes |

“Pill,” “station,” and “action” map to capability. “Bundle” maps either to a family
(selection/catalog convenience) or a workflow (execute all); ambiguity is prohibited.
“Routine” is simply a smaller reusable workflow. “Full workflow” is a workflow used at
a business boundary, not a different kind of object.

A workflow invocation may look like one node in its caller, but the resolved plan pins
and expands its internal graph. Expansion detects recursion, exposes effects and cost,
and generates a Merkle-style plan digest. A family used for execution must first be
resolved to a versioned explicit member list, preventing future registry additions
from silently changing behavior.

## 9. Decrypter / explorer

The explorer is a registry UI/API, search engine, and graph viewer—not merely
`DECODE(code)`:

- `resolve(ref)` returns immutable key, contract versions, aliases, and status.
- `decode(ref|key|short_code)` renders plain English, browse paths, ports, effects,
  assurance, implementations, examples, and change history.
- `search(text, facets, port_type)` combines synonyms/full text, semantic retrieval,
  facets, and exact schema compatibility; it returns explained candidates, never a
  silent best guess.
- `encode("classify claim maturity")` is search with disambiguation. It yields exact,
  compatible, and related candidates with matched terms/facets.
- `compatible(output_port)` uses schema assignability and constraints, not a manually
  curated successor list.
- `neighbors(ref)` shows derives-from, alternative-to, broader/narrower, required-by,
  and commonly-adjacent edges, each labeled with its source.
- `impact(ref@version)` lists workflows, policies, families, implementations, and
  schemas affected by change or retirement.
- `lineage(run_id)` reconstructs upstream/downstream invocation, artifact, policy,
  approval, and causal receipt graphs.

The UI should offer four synchronized views: **plain-language card**, **faceted
catalog**, **typed graph**, and **raw manifest**. AI-assisted reverse lookup may suggest
candidates but cannot mint, alias, or select a destructive capability without normal
registry and policy controls.

## 10. Demonstration

### 10.1 Heterogeneous capability catalog

| # | Canonical reference | Intent | Input → output | Effect/assurance |
|---:|---|---|---|---|
| 1 | `fis.sense.collection.scan@1` | sense | location → file refs | read, deterministic |
| 2 | `fis.sense.file.stat@1` | sense | file ref → metadata | read, deterministic |
| 3 | `fis.derive.file.media-type@1` | derive | byte sample → media type | pure, deterministic |
| 4 | `fis.derive.file.sha256@1` | derive | byte stream → digest | read, deterministic |
| 5 | `fis.derive.document.text.pdf@2` | derive | PDF → text document | read, deterministic |
| 6 | `fis.derive.document.text.docx@1` | derive | DOCX → text document | read, deterministic |
| 7 | `fis.derive.document.text.markdown@1` | derive | Markdown → text document | read, deterministic |
| 8 | `fis.derive.image.text.ocr@2` | derive | image → text document | read, probabilistic |
| 9 | `fis.derive.audio.transcript@1` | derive | audio → transcript | read, probabilistic |
| 10 | `fis.derive.text.sentences@1` | derive | text → sentences | pure, model-dependent |
| 11 | `fis.infer.text.claims@2` | infer | sentences → claims | read, probabilistic |
| 12 | `fis.infer.claim.type@1` | infer | claim → class/confidence | read, probabilistic |
| 13 | `fis.infer.claim.maturity@2` | infer | claim → maturity/confidence | read, probabilistic |
| 14 | `fis.infer.document.domain@2` | infer | text → domains/confidence | read, probabilistic |
| 15 | `fis.infer.text.embedding@3` | infer | text → vector | external read, model-dependent |
| 16 | `fis.relate.file.exact-duplicates@1` | relate | digests → duplicate sets | pure, deterministic |
| 17 | `fis.relate.document.similarity@2` | relate | vectors → scored edges | pure, deterministic given vectors |
| 18 | `fis.propose.file.canonical-keeper@1` | propose | duplicate set + metadata → ranking | pure, policy-dependent |
| 19 | `fis.propose.file.name@2` | propose | document facts → candidate name | read, probabilistic |
| 20 | `fis.propose.file.destination@2` | propose | file facts + routing rules → path | pure, policy-dependent |
| 21 | `fis.assure.path.collision-free@1` | assure | proposed path → verdict | read, time-sensitive |
| 22 | `fis.effect.file.move@2` | effect | file + destination → moved file | write, reversible conditionally |
| 23 | `fis.effect.file.quarantine@1` | effect | file + quarantine root → record | write, reversible conditionally |
| 24 | `fis.effect.sidecar.write@1` | effect | file + metadata → sidecar | write, idempotent by key |
| 25 | `fis.publish.search-index.upsert@2` | publish | document record → index result | external write, idempotent by key |
| 26 | `fis.assure.file.digest@1` | assure | file + expected digest → verdict | read, deterministic |
| 27 | `fis.effect.image.thumbnail@1` | effect | image + dimensions → artifact | write, deterministic |
| 28 | `fis.publish.receipts.export@1` | publish | receipt query → signed export | external write, deterministic |

File-format support is a family query such as `intent=derive AND output<=TextDocument
AND input.media_type IN supported_document_types`; entries 5–8 remain atomic.

### 10.2 Workflow A — simple: fingerprint one file

Human view:

```text
READ METADATA → CALCULATE SHA-256 → VERIFY DIGEST (optional expected value)
```

Canonical machine representation:

```yaml
kind: Workflow
ref: fis.workflow.file.fingerprint@1
inputs: {file: fis.schema.file-ref@1, expected: fis.schema.sha256@1?}
nodes:
  stat:   {call: fis.sense.file.stat@1, with: {file: $inputs.file}}
  hash:   {call: fis.derive.file.sha256@1, with: {file: $inputs.file}}
  verify:
    choose:
      when: {present: $inputs.expected}
      then: {call: fis.assure.file.digest@1,
             with: {file: $inputs.file, expected: $inputs.expected}}
outputs: {metadata: $nodes.stat.metadata, digest: $nodes.hash.digest,
          verdict: $nodes.verify.verdict?}
```

`stat` and `hash` may execute concurrently because neither depends on the other.

### 10.3 Workflow B — moderate: route heterogeneous documents safely

Human view:

```text
SCAN
→ FOR EACH FILE: DETECT MEDIA TYPE
→ CHOOSE PDF / DOCX / MARKDOWN / OCR TEXT EXTRACTOR
→ CLASSIFY DOMAIN
→ PROPOSE NAME + DESTINATION (parallel)
→ COLLISION CHECK
→ PREVIEW
→ WAIT FOR APPROVAL
→ MOVE
→ VERIFY DIGEST
→ WRITE SIDECAR + UPDATE INDEX (parallel)
```

Canonical machine representation (abridged but executable in the algebra):

```yaml
kind: Workflow
ref: fis.workflow.document.safe-route@3
nodes:
  files: {call: fis.sense.collection.scan@1, with: {location: $inputs.source}}
  each:
    map:
      over: $nodes.files.files
      concurrency: 8
      body:
        - {id: before, call: fis.derive.file.sha256@1}
        - {id: type, call: fis.derive.file.media-type@1}
        - id: text
          choose:
            when:
              cases:
                application/pdf: fis.derive.document.text.pdf@2
                application/vnd.openxmlformats-officedocument.wordprocessingml.document: fis.derive.document.text.docx@1
                text/markdown: fis.derive.document.text.markdown@1
              default: fis.derive.image.text.ocr@2
        - {id: domain, call: fis.infer.document.domain@2}
        - parallel_bindings:
            name: {call: fis.propose.file.name@2}
            destination: {call: fis.propose.file.destination@2}
        - {id: collision, call: fis.assure.path.collision-free@1}
        - {id: approval, wait: {signal: human.approval, timeout: 7d}}
        - choose:
            when: {all: [$collision.clear, {$approval: approved}]}
            then:
              - {id: move, call: fis.effect.file.move@2, compensate_with: fis.effect.file.restore@1}
              - {id: verify, call: fis.assure.file.digest@1, with: {expected: $before.digest}}
              - parallel_bindings:
                  sidecar: {call: fis.effect.sidecar.write@1}
                  index: {call: fis.publish.search-index.upsert@2}
            else: {emit: not_moved}
policy: fis.policy.managed-file-write@3
```

`parallel_bindings` is presentation sugar: the canonical graph simply contains two
independent nodes.

### 10.4 Workflow C — complex: governed corpus consolidation

Human view:

```text
ON SCHEDULE, SNAPSHOT REGISTRY/POLICY VERSIONS
→ SCAN MANY SOURCES
→ FOR EACH FILE (bounded parallelism): STAT + HASH + TYPE
→ GROUP EXACT DUPLICATES
→ FOR EACH UNIQUE DOCUMENT: TYPE-SPECIFIC EXTRACT/TRANSCRIBE/OCR
→ SEGMENT → EXTRACT CLAIMS
→ FOR EACH CLAIM: TYPE + MATURITY (parallel)
→ DOMAIN + EMBEDDING (parallel)
→ BUILD SIMILARITY GRAPH
→ FOR EACH DUPLICATE SET: PROPOSE KEEPER
→ PROPOSE NAMES AND DESTINATIONS
→ COLLISION, QUOTA, PRIVACY, AND DIGEST ASSURANCE
→ CREATE SIGNED PREVIEW
→ WAIT FOR TWO-PERSON APPROVAL
→ QUARANTINE LOSERS AND MOVE KEEPERS WITH COMPENSATION
→ VERIFY ALL POSTCONDITIONS
→ WRITE SIDECARS, UPSERT INDEX, EXPORT RECEIPT MANIFEST
→ IF ANY ASSURANCE FAILS: STOP PUBLISHING, COMPENSATE ELIGIBLE MOVES, ESCALATE
```

Canonical machine representation (the subworkflows remain expandable):

```yaml
kind: Workflow
ref: fis.workflow.corpus.governed-consolidation@1
trigger: {schedule: "0 2 * * *"}
inputs:
  sources: fis.schema.location-list@1
  policy: fis.policy.corpus-consolidation@1
nodes:
  inventory:
    use: fis.workflow.collection.inventory@2
    with: {sources: $inputs.sources}
  duplicates:
    call: fis.relate.file.exact-duplicates@1
    with: {digests: $nodes.inventory.digests}
  intelligence:
    map:
      over: $nodes.duplicates.unique_files
      concurrency: 16
      body: {use: fis.workflow.document.intelligence@4}
      on_error: {unsupported_media: {emit: manual_review}}
  similarity:
    call: fis.relate.document.similarity@2
    with: {vectors: $nodes.intelligence.embeddings}
  proposed:
    use: fis.workflow.corpus.propose-layout@2
    with:
      inventory: $nodes.inventory
      duplicates: $nodes.duplicates
      intelligence: $nodes.intelligence
      similarity: $nodes.similarity
  assured:
    use: fis.workflow.corpus.preflight@3
    with: {proposal: $nodes.proposed}
  preview:
    use: fis.workflow.change-set.signed-preview@1
    when: {$nodes.assured.verdict: pass}
  approval:
    wait:
      signal: approval.quorum
      constraints: {distinct_approvers: 2, expires: 72h}
  apply:
    choose:
      when: {all: [{$nodes.assured.verdict: pass}, {$nodes.approval: approved}]}
      then:
        use: fis.workflow.corpus.apply-change-set@2
        with: {change_set: $nodes.preview.change_set}
        timeout: 6h
        compensate_with: fis.workflow.corpus.compensate-change-set@2
      else: {emit: no_change}
  postflight:
    use: fis.workflow.corpus.verify-change-set@2
    when: {$nodes.apply.status: completed}
  publish:
    choose:
      when: {$nodes.postflight.verdict: pass}
      then:
        - {call: fis.publish.search-index.upsert-batch@1}
        - {call: fis.publish.receipts.export@1}
      else:
        - {use: fis.workflow.corpus.compensate-change-set@2}
        - {emit: operator_escalation}
policy: $inputs.policy
resolution:
  pin: [contracts, implementations, schemas, models, workflows, policy]
  digest: sha256:<computed-at-plan-time>
```

This graph exposes ordering and governance while delegating detail to expandable,
version-pinned subgraphs. The planner rejects a missing compensation binding, unsafe
retry, incompatible port, unbounded map, unresolved family, or effect not authorized
by policy.

## 11. Failure analysis

| Failure mode | How it appears | Countermeasure / stop rule |
|---|---|---|
| Bureaucracy | Manifest takes longer than implementation | Progressive schema; mandatory core only; generate forms/docs/tests |
| Taxonomy hell | Meetings debate a capability's one “correct” branch | Facets and multiple projections; taxonomy council governs vocabulary, not identity |
| False atomicity | Tiny wrappers make every library call a capability | Register only independently useful, observable, governable contracts |
| Mega-capabilities | “Process all files” hides decisions | Effect/type/cost budgets; require graph expansion for composites |
| Brittle hierarchy | Rename breaks workflows | Stable symbolic refs + opaque keys; browse paths are mutable metadata |
| Dependency chaos | Suggested predecessors become hard coupling | Compose by typed ports; distinguish runtime dependency from catalog relation |
| Version explosion | Every implementation release forks semantics | Separate contract, implementation, model, schema, and workflow versions |
| Metadata rot | Claims disagree with runtime | Conformance tests, signed artifacts, telemetry, ownership, deprecation automation |
| Effect laundering | Composite advertised read-only contains writes | Planner derives transitive effects from expanded graph |
| Retry damage | Non-idempotent move runs twice | Idempotency keys; reject retry policy unless declared and tested safe |
| Fake rollback | External API cannot undo work | Explicit compensation and residual-risk receipts; never promise global ACID |
| Receipt leakage | Inputs/logs expose confidential content | Content-addressed/redacted references, retention policy, access control |
| Receipt overhead | Per-item events dominate large runs | Batched/streamed storage, sampling only for telemetry—not mandatory causal receipts |
| Planner overhead | Resolution slows trivial calls | Cache signed plans; allow direct invocation while preserving authorization/receipt |
| Poor developer experience | Authors hand-write YAML and adapters | SDKs, schema inference proposals, local runner, linting, generated docs/scaffolds |
| Non-reproducible AI | Same model name produces new answers | Pin provider/model/prompt/config where possible; record uncertainty, never claim bitwise reproducibility |
| Dynamic family drift | “All types” changes tomorrow | Resolve family membership into the immutable plan |
| Distributed inconsistency | Move succeeds but index update fails | Saga-style compensation/reconciliation and explicit partial-success states |

Most importantly, **do not build a custom general-purpose orchestrator first**. Stop
and adopt an existing engine when durable scheduling, distributed retries, queues, or
high availability become requirements. FIS owns contracts, compilation, policy profile,
and exploration; an engine may own execution.

## 12. Existing analogues

- **Unix pipes and filters:** validate small composable tools, but byte streams lack
  FIS's typed semantic ports, declared effects, registry, and provenance.
- **Functional composition and typed intermediate representations:** provide the
  strongest conceptual basis—values flow through typed functions and a compiler checks
  a graph. Real-world effects and probabilistic outputs require an effect system and
  richer receipts.
- **DAG/dataflow engines** (for example Airflow, Dagster, Prefect, Temporal, Argo):
  already solve portions of scheduling, retry, state, and observability. FIS should
  compile to one rather than duplicate it. Some support cycles/events differently,
  so the portable profile must remain conservative.
- **Build systems** (Make, Bazel, Nix): contribute dependency graphs, content hashes,
  caching, hermeticity, and reproducible resolution. File intelligence adds mutation,
  approvals, inference, and business policy.
- **Package/plugin registries:** contribute immutable identities, namespaces,
  semantic versions, dependency resolution, signing, and deprecation. A capability
  contract is closer to a package interface than a source filename.
- **OpenAPI, JSON Schema, Protocol Buffers, and CloudEvents:** supply interface,
  validation, compatibility, and event-envelope mechanisms. They do not themselves
  define workflow meaning.
- **Command pattern and capability-based security:** separate a request from its
  handler and constrain authority. FIS should use capability-style grants but avoid
  confusing a catalog “capability” with an unforgeable security token.
- **ETL and enterprise integration patterns:** already cover routing, splitting,
  aggregation, filtering, idempotent consumers, and dead-letter handling. FIS's small
  algebra should reuse those semantics.
- **Policy-as-code** (such as OPA): separates authorization/guardrails from process
  graphs. Embedding guardrails as ordinary `if` statements would be weaker and harder
  to audit.
- **W3C PROV and OpenTelemetry:** offer vocabulary for entities, activities, agents,
  traces, and causal links. FIS receipts should profile established models instead of
  inventing an isolated ledger format.
- **MIME/media-type registries and content negotiation:** demonstrate why file-type
  support should be data-driven and expandable, not a monolithic “all formats” action.
- **Microservices:** share independently replaceable implementations, but making every
  atomic operation a network service would add latency and operational burden. FIS
  capability granularity is semantic, not deployment granularity.

The combination is not proven novel. Its potentially valuable emphasis is a
human-oriented, file-intelligence-specific semantic registry that unifies typed
composition, effects/policy, and provenance across local tools, models, remote
services, and non-programmer interfaces.

## 13. Recommendation and delivery sequence

### BUILD A SIMPLIFIED VERSION

Build the semantic control plane, but borrow execution machinery. Validate the idea
with 20–30 capabilities and two end-to-end workflows before designing a “periodic
table” or supporting thousands.

1. **Specify the kernel:** capability/schema/workflow/receipt JSON Schemas, compatibility
   rules, effect vocabulary, and five composition nodes.
2. **Create one registry:** Git-backed manifests are sufficient initially; add a search
   index as a projection, not a second source of truth.
3. **Wrap existing code thinly:** adapters plus conformance fixtures; do not rewrite
   algorithms or preserve the 00–12 boilerplate.
4. **Implement plan/lint/explain first:** resolve aliases and families, type-check,
   expand composites, derive transitive effects, pin versions, and render a plan.
5. **Execute locally:** a small reference runner proves semantics; use an established
   orchestration engine once durability/distribution is required.
6. **Add policy before destructive effects:** preview, explicit authorization,
   idempotency, compensation, and causal receipts are release gates for move/delete.
7. **Measure governance cost:** reject the architecture if registering a modest pure
   capability cannot be done in minutes, or if users cannot explain a plan from its
   rendered graph without reading implementation code.

### Mapping the original vocabulary

| Original term | Adopted interpretation |
|---|---|
| Pill / station / action | Atomic capability contract plus one or more bindings |
| Field / operator / value / condition | Typed, pure expression language |
| Trigger | Run creation event outside the workflow graph |
| Guardrail | Independently versioned policy evaluated at plan/run time |
| Family | Saved/curated registry query; never implicitly executable |
| Bundle | Remove; spell it as family or workflow |
| Routine | Reusable workflow/subgraph |
| Workflow | Versioned typed graph of calls and control nodes |
| Station Script Standard | Replace with adapter protocol and shared runtime |
| Handoff | Typed edge/port binding |
| Archive | Retention policy or explicit effect capability |
| Decrypter | Registry resolver, explorer, compatibility graph, and lineage UI |

The decisive simplification is this: **identity is a stable name, orientation is a
faceted view, execution is a binding, composition is a typed graph, permission is
policy, and history is a receipt.** Keeping those separate gives FIS room to grow
without asking one identifier—or one Python template—to carry the whole architecture.
