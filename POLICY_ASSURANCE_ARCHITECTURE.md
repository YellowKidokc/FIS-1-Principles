# FIS Policy and Assurance Layer

> **Status:** proposed architectural crosswalk, not an implementation. The three supplied
> workbooks have been reviewed together. This proposal should remain provisional until
> workbook owners confirm terminology, thresholds, and the intended meaning of “Bowel
> Symptom Program.” No detector or file mutation is implemented here.

## 1. Decision

Promote `folder_symptom_registry.xlsx` from a diagnostic catalog into one input to a
versioned **policy bundle**, but do not turn every symptom into a guardrail. A detector
produces evidence; a rule interprets that evidence in the context of a target profile,
a proposed action, and an evaluation stage; policy combines the resulting decisions;
a workflow enforces the decision; and a receipt records facts.

This preserves the separation already adopted in the FIS architecture:

| FIS plane | Policy-layer object | Responsibility |
|---|---|---|
| Capability | detector, snapshotter, comparer, mover, compensator | Typed operation with declared effects |
| Workflow | assurance envelope | Orders scans, planning, approval, execution, verification, and compensation |
| Policy | rule and versioned policy bundle | Decides whether a resolved plan may advance |
| Receipt | snapshot, finding, decision, action result, comparison | Immutable evidence and causal history |

A symptom ID such as `I01` remains a human/domain identifier. It must not double as a
capability identity or encode policy behavior. `detect_program_root()` is an unbound
workbook implementation hint; a governed detector reference such as
`fis.assure.folder.program-root@1` is the semantic contract.

## 2. Workbook crosswalk

| Workbook source | Keep | Change before execution use |
|---|---|---|
| `folder_symptom_registry.xlsx` / Symptom Registry | 40 descriptions, evidence ideas, recommended responses, severity, risks, examples | Add applicability, stages, decision defaults, detector contracts, evidence schema, confidence, unknown/error behavior, and postconditions |
| `folder_symptom_registry.xlsx` / Detection Functions | Function-to-symptom mapping and rough complexity | Replace uniform `folder_path -> SymptomResult[]` with typed inputs/outputs; record detector version and evidence; implement and test separately |
| `folder_symptom_registry.xlsx` / Severity Scale | Human triage urgency | Do not infer execution decisions from severity alone |
| `FIS_Rule_Automation_Grammar.xlsx` / Fields and Operators | Starting vocabulary for pure selection expressions | Add snapshot and profile fields; freeze types and null/unknown semantics |
| Grammar / Actions Master and Action Conditions | Action risk defaults, prerequisites, preview and approval expectations | Express as versioned action policies; add freshness, exact outcomes, atomicity/compensation, and postconditions |
| Grammar / Guardrails | Twelve strong safety intentions | Split broad prose into composable rules with explicit scope and evidence |
| Grammar / Workflow Templates | Human-readable playbooks | Compile them into inspectable graphs that always invoke the envelope |
| Grammar / Ledger Schema | Useful action-ledger seed | Replace row-oriented mutable status with causally linked, append-only receipts and artifact digests |
| `FIS SYSTEM.xlsx` | Embedded grammar plus inventory/move user experience as migration evidence | Treat its macro inventory and bulk move sheets as an **untrusted legacy adapter**, never an execution path; it lacks the required baseline/freshness/postflight envelope |

The two grammar-bearing workbooks substantially overlap. They should not become two
sources of policy truth. Select one canonical, exported manifest set; retain workbook
sheets as authoring/import views. The policy compiler must reject conflicting duplicate
IDs rather than choose whichever workbook was loaded last.

## 3. Health symptoms versus guardrails

Separate three concepts:

1. **Observation** — detector evidence, e.g. “three case-folded names collide.”
2. **Health interpretation** — corpus/folder quality, e.g. “naming collision: high.”
3. **Execution decision** — whether this exact plan worsens or violates an invariant.

A finding is not inherently blocking. Its rule supplies a context-sensitive decision:

| Decision | Meaning | Workflow effect |
|---|---|---|
| `BLOCK` | A required invariant is false or cannot be established safely | No approval or execution; a newly stale preview must be rebuilt |
| `REQUIRE_APPROVAL` | Execution may proceed only with an unexpired, scoped human decision | Create approval request bound to the exact plan digest |
| `WARN` | Material risk is visible but does not violate this action's invariants | Show prominently; receipt must record acknowledgement policy |
| `RECORD` | Health/provenance fact that does not affect this execution | Preserve for reports and later workflows |

`severity` answers **how serious is the underlying condition?** `decision` answers
**what may this run do now?** They are independent. Auto-fixability is also independent:
a low auto-fixable issue can be blocked by boundary policy, while a critical condition
can be safely recorded during a read-only scan.

Decision precedence is `BLOCK > REQUIRE_APPROVAL > WARN > RECORD`. Precedence never
allows a broader `WARN` to weaken a narrower `BLOCK`. Policy exceptions must be
separate, signed, expiring grants naming rule ID, plan digest, scope, actor, reason,
and allowed decision transition. Universal invariants such as changed source hashes,
out-of-bound destinations, unresolved collisions, and silent overwrites are
non-overridable for the initial release.

### Critical registry crosswalk

| Symptom | Health interpretation | Contextual policy behavior |
|---|---|---|
| I01 Program-root danger | Critical infrastructure finding | `BLOCK` broad move/rename/delete/archive within a program root; `REQUIRE_APPROVAL` for an explicitly modeled whole-root migration; `RECORD` for scan/report |
| I03 Config leak | Critical security finding | `BLOCK` publish or external copy; `BLOCK` plans that expose secrets; protect/redact evidence; ordinary internal movement requires security policy review |
| S06 Naming collision | High structural finding | `BLOCK` when the proposed destination collides under destination filesystem semantics; otherwise `WARN` existing ambiguity |
| S08 Phantom reference | Medium health finding | Record existing broken links; `BLOCK` if a move/rename would create additional resolvable broken references, otherwise `WARN` when reference parsing is incomplete |
| I05 Repository/non-repository mix | High infrastructure finding | `BLOCK` loose-file moves that alter tracked repository content or strand untracked dependencies; require Git-aware planning and clean/explicit status |
| I06 Mirror drift | High infrastructure finding | `BLOCK` sync/reconciliation or a move presented as authoritative until source-of-truth is chosen; `WARN` unrelated local copies |
| N04 Path-length danger | Medium portability finding | `BLOCK` a destination exceeding a declared platform limit; `WARN` when it only exceeds a conservative portability threshold |
| R07 Law coverage hole | High corpus-health finding | `RECORD` for filesystem actions; `WARN` or `REQUIRE_APPROVAL` in corpus publication/research-completeness review |
| R08 Claim without evidence | High content-health finding | `RECORD` for move/copy; `BLOCK` or `REQUIRE_APPROVAL` for publication according to editorial policy |

Unknown detector results fail closed only where evidence is mandatory. For example, an
unavailable collision detector blocks rename, while an unavailable law-coverage query
records an incomplete health scan during a local move.

## 4. Policy composition

Policy selection is additive:

```text
selected policy = universal
                + every classified folder profile (including mixed/unknown)
                + action policy
                + boundary/data-classification policy
                + organization/user policy that only tightens defaults
```

Classification may be multi-label with evidence and confidence. A Git repository that
also contains documents is not forced into one bucket: it receives `git_repository`,
`document_collection`, and `unknown_mixed` where ambiguity remains. The most
conservative applicable rule wins.

Initial folder profiles are `program_root`, `git_repository`, `obsidian_vault`,
`authored_website`, `media_collection`, `document_collection`, `archive`,
`working_folder`, `generated_output`, and `unknown_mixed`. Low-confidence or conflicting
classification always adds `unknown_mixed`; it never removes a detected high-risk
profile.

Universal rules require source existence and matching identity/hash; a destination
inside the approved boundary; collision-free, no-overwrite semantics; adequate space;
one expected outcome per selected object; and a writable receipt store. Folder rules
add profile-aware dependencies (Git state, links, attachments, plugins, sidecars,
configuration, build outputs). Action rules add move/rename/delete/copy/publish
semantics. Rules cannot execute changes; they only return decisions and evidence.

## 5. Required execution envelope

Every effectful FIS workflow expands to these gates:

1. **Classify target.** Produce profile labels, confidence, classifier version, and evidence.
2. **Capture baseline snapshot.** Persist it immutably before planning.
3. **Select rubrics.** Resolve and pin universal, profile, action, and boundary policies.
4. **Run preflight.** Evaluate rules; stop on blocks or required evidence failures.
5. **Compile exact change plan.** Enumerate every intended effect and expected outcome.
6. **Freshness rescan.** Re-read concurrency-sensitive facts immediately before approval/execution.
7. **Stop on drift.** Mark the plan `STALE`; never “refresh” it silently.
8. **Obtain approval.** Bind actor approval to plan, baseline, policy, and preview digests with expiry.
9. **Execute.** Use exclusive/no-clobber operations and an idempotency key; append results per item.
10. **Run postflight.** Observe declared invariants, not merely the original symptoms.
11. **Compare expected/observed.** Produce a structured diff and verdict.
12. **Receipt or compensation/escalation.** Pass emits a completion receipt; mismatch stops downstream effects and invokes only declared eligible compensation.

Approval occurs after the freshness scan so that the approver sees a current preview.
For long approval windows, repeat freshness immediately after approval and before the
first write. Any drift invalidates approval because its plan digest no longer describes
the world.

## 6. Immutable baseline snapshot

The snapshot is a content-addressed artifact with a canonical serialization and digest.
It contains enough information to re-evaluate the exact plan without copying sensitive
file content into the receipt:

- snapshot ID, schema version, capture timestamps, scanner versions, host/platform,
  filesystem identity/capabilities, and normalized path/case/unicode semantics;
- approved roots and resolved real paths, mount/device/volume IDs, symlink policy,
  boundary/data-classification labels, and destination free-space observation;
- deterministic tree inventory: relative and canonical path, object type, stable file
  identity where available, size, mtime/ctime precision, mode/ACL summary, link target,
  and cryptographic content digest for every selected or dependency-relevant file;
- directory membership/tree digest, including relevant empty directories and excluded
  entries represented by explicit exclusion rules;
- target profile labels, evidence, confidence, and classifier contract version;
- repository identity, HEAD/ref/index/worktree/submodule status, tracked/untracked/
  ignored sets relevant to scope, and configured worktree boundaries;
- dependency/config markers and digests, lockfiles, sidecar pairs, resolvable incoming
  and outgoing references, and detector evidence required by selected rules;
- mirror endpoints and comparison digest where mirror policy applies;
- policy bundle, rule, capability, schema, implementation, configuration, and threshold
  versions; actor and runtime authority; and receipt/quarantine locations plus
  writability probes;
- redaction map and references to encrypted evidence for secrets. Secret values are
  never copied into ordinary snapshots or logs.

The artifact store must prevent mutation (content addressing plus write-once retention
or equivalent). A receipt references its digest. A new observation creates a new
snapshot; it never updates the old one.

## 7. Freshness and concurrent change detection

Freshness is a purpose-built, fast re-evaluation of all facts on which the plan relies:

- re-resolve roots and destinations to defeat symlink/junction swaps;
- verify source existence, stable identity, size/metadata, and SHA-256 against baseline;
- recompute affected directory membership/tree digests to detect additions, removals,
  and renames;
- re-check every destination for exact, case-folded, and Unicode-normalized collision;
- re-check free space and writable receipt/quarantine locations;
- re-read Git HEAD, index and relevant worktree status; dependency/lock/config marker
  digests; reference graph inputs; mirror state; mounts and filesystem capabilities;
- ensure selected policy/implementation/configuration versions and execution authority
  remain pinned and available.

The output is `FRESH` or `STALE`, never “close enough.” It includes typed drift entries
(`source_content_changed`, `tree_member_added`, `destination_appeared`, `git_head_changed`,
`dependency_changed`, `mount_changed`, etc.), baseline and fresh values/digests, scope,
and timestamp. Metadata-only differences may be ignored only when a rule explicitly
excluded that field from the plan dependency set. A stale run has no effects and emits
a terminal stale-plan receipt pointing to a newly requested planning run.

## 8. Proposed versioned rule schema

The canonical form should be JSON/YAML validated by a published schema. This complete
example expresses I01 without binding its semantics to one Python function:

```yaml
api_version: fis.policy/v1alpha1
kind: AssuranceRule
metadata:
  key: rule_01_program_root_move
  ref: fis.policy.folder.program-root-protection@1
  version: 1.0.0
  title: Protect program roots from structural mutation
  source_symptoms: [I01]
  owner: fis-core
  status: proposed
spec:
  purpose: guardrail                 # guardrail | health
  severity: critical                 # triage, not execution decision
  applicability:
    folder_types: [program_root, git_repository]
    actions: [move, rename, delete, archive]
    stages: [preflight, freshness, postflight]
    boundaries: [local, shared, external]
  detector:
    ref: fis.assure.folder.program-root@1
    inputs: [target_snapshot, action_plan]
    evidence_schema: fis.schema.evidence.program-root@1
    on_unknown: BLOCK
    on_error: BLOCK
  required_evidence:
    - package_markers
    - dependency_directories
    - repository_status
  condition:
    any:
      - {field: evidence.program_root, equals: true}
      - {field: target.profiles, contains: git_repository}
  decisions:
    - when: {field: action.scope, equals: broad_internal_mutation}
      decision: BLOCK
      reason_code: program_root_internal_mutation
    - when: {field: action.scope, equals: whole_root_relocation}
      decision: REQUIRE_APPROVAL
      reason_code: governed_root_relocation
    - otherwise: RECORD
  approval:
    roles: [repository_owner]
    quorum: 1
    expires_in: PT30M
    binds: [baseline_digest, plan_digest, policy_bundle_digest]
  postconditions:
    - id: program_root_unchanged
      comparator: tree_digest_equals
      expected: plan.expected.program_root_tree_digest
      observed: postflight.program_root_tree_digest
  remediation:
    recommendation: Protect the root; plan a whole-project migration separately.
    compensation_ref: null
  receipt:
    evidence_redaction: metadata_only
    retention_class: assurance
```

Schema rules:

- `metadata.ref` major version denotes stable meaning; breaking meaning/applicability/
  evidence changes require a new major. Threshold or wording changes use normal semantic
  versioning and produce a new policy-bundle digest.
- Applicability sets are explicit and closed. `*` is prohibited for destructive action
  rules unless reviewed as universal.
- Every required detector declares `on_unknown` and `on_error`.
- Conditions use the grammar's pure, typed fields/operators only; no arbitrary code or effects.
- Decisions include stable reason codes. Default fallthrough is `RECORD`, never implicit allow.
- Postconditions name a comparator and typed expected/observed sources.
- A `health` rule cannot emit `BLOCK` unless a separately reviewed guardrail mapping is present.
- Detector output records confidence, coverage, evidence pointers, start/end time, and
  implementation digest. Policy—not detector code—maps evidence to a decision.

A `PolicyBundle` pins exact rule versions and selection logic. A resolved plan embeds
the bundle digest and explicit selected rule list; future registry edits cannot alter
an approved run.

## 9. Exact plans and postflight comparison

Each plan item has an immutable `item_id`, action, source identity/path/digest,
destination canonical path, overwrite policy (`deny` initially), expected source
outcome, expected destination outcome/digest, reference/sidecar effects, compensation
eligibility, and ordering dependencies. The plan also states global cardinalities,
allowed incidental writes (normally receipt artifacts only), empty-folder outcomes,
and invariants that must remain unchanged.

Postflight takes the plan—not the detector list—as its specification. It observes only
after all execution results are durably recorded and reports per-invariant values:

```yaml
comparison:
  expected:
    moved: 31
    destination_hash_matches: 31
    overwrites: 0
    missing: 0
    new_broken_references: 0
    source_folder_empty: true
    unexpected_changes: 0
  observed:
    moved: 31
    destination_hash_matches: 31
    overwrites: 0
    missing: 0
    new_broken_references: 0
    source_folder_empty: true
    unexpected_changes: 0
  differences: []
  verdict: PASS
```

Every comparison includes comparator/version, evidence pointers, and whether the
invariant is required or advisory. A required mismatch yields `POSTFLIGHT_FAILED`,
stops later nodes (publication, cleanup, notifications claiming success), preserves
all evidence, and begins a compensation decision.

## 10. Partial success, compensation, quarantine, and residual risk

Use explicit run and item states rather than one mutable `status` cell:

- Run outcomes: `DENIED`, `STALE`, `APPROVAL_EXPIRED`, `SUCCEEDED`,
  `PARTIALLY_SUCCEEDED`, `POSTFLIGHT_FAILED`, `COMPENSATED`,
  `COMPENSATION_PARTIAL`, `ESCALATED`.
- Item outcomes: `PLANNED`, `SKIPPED`, `APPLIED`, `VERIFIED`, `FAILED`,
  `COMPENSATION_ELIGIBLE`, `COMPENSATED`, `COMPENSATION_FAILED`, `RESIDUAL`.

Receipts are append-only events linked by `run_id`, `item_id`, `caused_by`, previous
receipt digest, baseline/plan/policy/approval digests, actor, timestamp, and evidence
artifacts. “Rollback” is not promised. Compensation is a new forward action with its
own authorization and receipt. It may fail or be unsafe after external observation.

Quarantine is a first-class governed destination with boundary, access, capacity,
retention/hold, original path and digest, reason, restore capability, and manifest.
Nothing is deleted merely because quarantine succeeded. A quarantine receipt proves
both destination integrity and the exact source outcome.

Residual risk entries contain `risk_id`, affected items, condition, severity,
evidence, attempted compensation, current location/hash, exposure, reversibility,
owner, required next action, deadline, and acknowledgement. A run cannot claim success
while required residual risks remain; it ends `ESCALATED` or `COMPENSATION_PARTIAL`.

## 11. Worked example: moving a mixed folder safely

### Scenario

`/work/inbox` contains 31 ordinary documents intended for `/archive/2026/intake`, plus:

- `client-site/` with `.git`, `package.json`, tracked assets, and two loose PDFs (I01/I05);
- `.env` containing a credential (I03);
- `Report.md` and `report.md` (S06 on a case-insensitive destination);
- Markdown links to `old-note.md`, already missing, and one link to a moved image (S08);
- a mirror record whose NAS copy differs for one document (I06);
- one proposed destination path of 214 characters (N04);
- a corpus database reporting an under-covered law and a claim-heavy note with weak
  evidence (R07/R08).

The requested action is “move this mixed folder.” FIS must turn that phrase into an
exact, reviewable set; it must not recursively move everything.

### A. Classification and baseline

The classifier emits `working_folder`, `document_collection`, `git_repository`,
`authored_website`, and `unknown_mixed` with evidence. The baseline captures 39 total
objects, hashes, memberships, links, sidecars, Git HEAD/index/worktree status, package
and secret markers, mirror comparison, destination semantics/free space, and receipt/
quarantine writability. Selected policies are universal + unknown/mixed + Git/program
root + authored website + document collection + move + internal-boundary.

### B. Preflight decisions

| Finding | Decision | Plan consequence |
|---|---|---|
| I01 program root | `BLOCK` for recursive root move | Exclude `client-site/` as a protected subtree; a separate whole-project migration is required |
| I05 loose PDFs inside repo | `REQUIRE_APPROVAL` only after dependency/reference proof | Do not silently extract them; present two separately planned items or leave in place |
| I03 `.env` | `BLOCK` from archive boundary if that boundary is less protected | Leave protected in place; emit redacted security escalation |
| S06 case collision | `BLOCK` both original destinations | Require explicit disambiguated names and regenerate plan |
| S08 existing missing note | `WARN` existing debt | Record baseline count; moving image must rewrite/retain its link or is blocked from increasing count |
| I06 drifted mirror file | `BLOCK` that item | Choose source of truth in a separate reconciliation decision |
| N04 214-character path | `WARN` under hard destination limit, or `BLOCK` if declared platform limit is exceeded | Preview shortened destination if policy requires portability |
| R07 law hole | `RECORD` | Include in corpus-health report; no move impact |
| R08 evidence gap | `RECORD` | Preserve finding; it would become approval/blocking in publish workflow |

After resolving the two naming collisions with approved candidate names, the compiler
produces a 31-item plan: 28 direct document moves, two explicitly approved loose-PDF
moves proven untracked and dependency-free, and one image move paired with an atomic
link update. The program subtree, secret, mirror-drift item, and unrelated objects all
have explicit `leave_unchanged` outcomes. There are no implicit selections.

Expected global outcomes are 31 source removals, 31 destination files with matching
hashes, one authorized Markdown content change with its own before/after digest, zero
overwrites, zero new broken references, zero unexpected changes, protected subtree
unchanged, secret unchanged/unexposed, and the source folder **not** empty because
protected/unresolved content remains. This corrects the common but unsafe assumption
that every move should empty its source.

### C. Freshness, approval, and execution

Immediately before approval, FIS re-resolves paths, hashes selected and dependency
files, compares memberships, destinations, Git state, link inputs, mirror marker,
space, mounts, and receipt target. If another process adds `report-2026.md`, changes a
selected hash, checks out another Git commit, or creates a destination, the plan is
`STALE` with no writes.

When fresh, the preview shows all 31 moves, the one link edit, every unchanged object,
warnings, compensation eligibility, and expected residuals. Approval binds to the
baseline, plan, and policy digests for 30 minutes. FIS performs a second minimal
freshness check after approval, then executes no-clobber operations in dependency-safe
order. Each destination is verified before its source outcome is accepted. Receipts
are appended per operation.

### D. Postflight and failure path

Postflight verifies item hashes and paths, source absence for moved items, unchanged
hashes/tree digests for protected objects, authorized link after-hash, no new broken
references, Git state invariant, no extra destination files, no overwrite, and the
expected non-empty source membership.

If 30 moves verify but one destination hash differs, the run becomes
`POSTFLIGHT_FAILED`, downstream publication stops, and evidence is frozen. Only moves
whose source can be restored without collision and whose destination identity still
matches are compensation-eligible. Compensation restores those items with new
receipts. The mismatched item remains quarantined or in place according to its actual
state; a residual-risk record names its two observed copies/hashes, exposure,
operator, and required reconciliation. The final outcome is `COMPENSATION_PARTIAL` or
`ESCALATED`, never “rolled back successfully” unless every declared compensation
postcondition passes.

## 12. Smallest implementation sequence (future work; not performed here)

1. **Freeze exports, not rules.** Export the 40 symptoms and overlapping grammar sheets
   into reviewable manifests; assign immutable keys; validate duplicate IDs and source
   lineage. Confirm the “Bowel Symptom Program” naming/scope with workbook owners.
2. **Define schemas first.** Publish schemas for `Evidence`, `Finding`, `AssuranceRule`,
   `PolicyBundle`, `BaselineSnapshot`, `ExactPlan`, `Approval`, `Comparison`, and
   append-only `Receipt`; add fixtures and a policy linter.
3. **Build read-only kernel capabilities.** Implement deterministic inventory,
   real-path/boundary resolution, SHA-256, tree digest, space/collision checks, Git
   state, and receipt-store probe. These support the envelope before any mutation.
4. **Implement the first critical detector slice.** I01, I03, S06, N04, and I05 cover
   project roots, exposure, collision, portability, and repository state. Each gets
   unit fixtures, false-positive cases, evidence redaction, and `unknown/error` tests.
5. **Implement plan/explain/freshness without execute.** Generate exact plans and stale
   diffs for a no-op simulator. Make workbook `ACT_MOVE` preview data an import-only
   adapter, never authority.
6. **Add a safe copy pilot.** Copy is easier to compensate than move. Enforce no-clobber,
   destination hash verification, postflight comparison, approvals, and receipts.
7. **Add governed move as copy-verify-remove.** Limit to one local filesystem/profile,
   small cardinality, and no program roots; implement compensation and fault-injection tests.
8. **Expand dependency assurance.** S08 link graphs and sidecars, I06 mirror comparison,
   then profile-specific Obsidian/HTML/media rules.
9. **Add research-health detectors last for movement.** R07/R08 feed corpus and publish
   policies; they should not delay safe filesystem primitives.
10. **Gate release.** No delete/publish/sync until freshness race tests, power/interruption
    simulations, compensation tests, secret-redaction tests, and receipt verification pass.

The first usable milestone is therefore not “40 implemented functions.” It is one
small, explainable policy bundle around scan → plan → freshness → approved copy/move →
compare → receipt, with five high-value detectors and adversarial tests. The remaining
35 detectors can then be added without changing the safety kernel.

## 13. Decisions still requiring owner confirmation

- Whether “Bowel Symptom Program” is the intended name of a workbook/domain or a typo;
  none of the supplied filenames or visible sheet titles uses that name.
- Which grammar workbook is canonical, since `FIS SYSTEM.xlsx` embeds the grammar while
  `FIS_Rule_Automation_Grammar.xlsx` also exists independently.
- Destination platform path limits and Unicode/case semantics; thresholds such as 200
  characters must be policy configuration, not universal truth.
- Boundary classifications, secret-handling authority, approval roles/quorum/expiry,
  snapshot retention, and quarantine retention/restore ownership.
- Whether a whole-program-root migration is ever permitted, and if so which build/test/
  deployment postconditions its profile requires.
- Corpus definitions and thresholds for R07/R08, and the publication states in which
  they change from record/warn to approval/block.

Until these are confirmed, `v1alpha1` is intentionally provisional and must not be
used to authorize effects.
