---
name: bridge-the-gap
description: "Use ONLY when explicitly invoked by name, skill selection, $bridge-the-gap, or /bridge-the-gap to teach selected KUs from a find-the-gap learning graph and resume prior bridge-the-gap progress. Teach depth first through attempts, tailored explanations, and fresh verification questions; checkpoint enriched JSON. Never invoke automatically for confusion, technical questions, or generic learning requests. Creating, editing, quoting, or discussing this skill is not an invocation."
---

## Standalone setup

This Markdown contains the entire skill and all required runtime resources. No companion download or network installation is needed. On explicit invocation, resolve the persistent learning root using the workflow below, then materialize the four embedded files once into a content-addressed `.learning-runtime` directory under that root. This is runtime support; learning bundles remain separate child folders of the learning root. Existing bundled installations may use their identical resources directly.

Use the following Python 3 snippet with this Markdown's absolute path as the first argument and the chosen learning root as the optional second argument (otherwise it reads `LEARNING_WORKSPACE`). Execute it through the harness with properly quoted arguments. It prints the runtime directory. Resolve all `scripts/...` and `references/...` resource paths in this document against that directory, and pass the learning root—not the runtime directory—to `learning_store.py init`. Do not materialize or run the workflow merely because this skill is being discussed or edited.

```python
from pathlib import Path
import hashlib, os, re, sys
source = Path(sys.argv[1]).read_text(encoding="utf-8")
root_value = sys.argv[2] if len(sys.argv) > 2 else os.environ.get("LEARNING_WORKSPACE")
if not root_value:
    raise SystemExit("Supply a persistent absolute learning root or set LEARNING_WORKSPACE")
root = Path(root_value).expanduser()
if not root.is_absolute():
    raise SystemExit("Learning root must be absolute")
pattern = r"(?ms)^<!-- BEGIN EMBEDDED FILE ([^\n]+) -->\n(.*?)\n<!-- END EMBEDDED FILE -->$"
parts = re.findall(pattern, source)
required = {"references/graph-format.md", "references/learning-graph.schema.json", "scripts/learning_store.py", "scripts/learning_validate.py"}
if len(parts) != 4 or {name for name, _ in parts} != required:
    raise SystemExit("Embedded runtime is incomplete or duplicated")
decoded = []
for name, body in parts:
    if name != "references/graph-format.md":
        lines = body.splitlines()
        if not lines[0].startswith("````") or lines[-1] != "````":
            raise SystemExit("Malformed embedded code block: " + name)
        body = "\n".join(lines[1:-1])
    decoded.append((name, (body + "\n").encode("utf-8")))
parts = decoded
digest = hashlib.sha256(b"".join(name.encode() + b"\0" + content for name, content in sorted(parts))).hexdigest()
runtime = root.resolve() / ".learning-runtime" / digest[:20]
for name, content in parts:
    target = runtime / name
    target.parent.mkdir(parents=True, exist_ok=True, mode=0o700)
    if target.exists():
        if target.read_bytes() != content:
            raise SystemExit("Existing runtime bytes differ; do not overwrite: " + str(target))
    else:
        with target.open("xb") as stream:
            stream.write(content)
        target.chmod(0o600)
print(runtime)
```

The graph contract is also readable below at [Shared JSON contract](#shared-json-contract-version-50). Embedded scripts are implementation resources: read or materialize them when needed rather than treating their code as conversational evidence. Keep all learner records within the chosen bundle.


# Bridge the Gap

Teach the user to understand the selected known unknowns (KUs), using the results of find-the-gap and any prior partial teaching runs as one knowledge bundle. Finish a path to one terminal KU before starting another. Use the graph to personalize instruction, not to infer general ability. Use the bundled persistence helper and the output contract; standalone Markdown exports embed the helper contents. No other skill or rendering tool is required.

## Shared evidence thresholds

Use exactly the same meaning of each knowledge state throughout the pipeline, relative to the stated concept and task context:

- **Known:** attributable evidence sufficiently demonstrates the relevant understanding. A correct explanation, reasoned prediction, or independent application may suffice; do not routinely demand repeated demonstrations. A claim of familiarity alone is a self-report, not demonstrated mastery.
- **Partial:** identify both a specific correctly understood component and a specific missing or incorrect component. Cite the evidence for each in the assessment rationale. Merely naming a topic, saying “I mix this up,” or narrowing a question is not a correctly understood component. If none is demonstrated, do not use Partial.
- **Unknown:** attributable evidence indicates the relevant understanding is missing, such as a specific admission of not knowing or a grounded misconception. Keep self-reported absence labeled user_report; an incorrect answer does not imply all prerequisites are unknown.
- **Unassessed:** evidence is insufficient, ambiguous, invalid, or only shows assistant-generated work. An unresolved answer key, missing organizational documentation, skip, or absence of discussion cannot establish Unknown. An explanation received without a subsequent demonstration does not itself establish Known.

Confidence and its rationale describe strength and limitations of the evidence for the state; they do not replace the state. High confidence that evidence is insufficient can accompany Unassessed. Keep confidence that a historical gap existed distinct from current knowledge confidence. A user can be confident and wrong, uncertain and correct, or uninterested in a target they do not know.

Record basis and assistance separately. A substantive correct demonstration remains evidence even if introduced with “I know”; the wording alone is not proof or disproof. Preserve prior assessments and attempts. When a new assessment corrects an earlier overclaim, append it with the correction reason and previous_assessment_id, then move only the concept's current pointer. Do not erase the old judgment, fabricate known components, or treat user removal of a target as knowledge evidence. A later isolated slip should not automatically replace stronger applicable evidence; explain a justified change or resolve meaningful ambiguity.

## Shared persistent workspace

Run on a machine with writable persistent storage. Resolve the root from an explicit user-supplied workspace or `LEARNING_WORKSPACE`; require an absolute path. If neither exists, ask once for a persistent location and save that choice in the bundle manifest. Do not silently use a temporary directory, the skills installation, or a cloud service. Never execute environment-variable contents as shell code. An explicit bundle folder or graph inside a known bundle resumes that bundle; the root variable is only a default for creating or locating an explicitly named bundle, not permission to search unrelated learning records.

For a new bundle, mind-the-gap creates a collision-resistant child folder under that root; find-the-gap and bridge-the-gap reuse it. Importing an external graph explicitly creates a bundle there without modifying the source. Preserve its bundle ID. Reject an ambiguous selection rather than picking the newest folder. All three skills use the same layout:

- `bundle.json`: store version, stable bundle ID, root, creation time, and display name.
- `HEAD.json`: authoritative committed revision and checkpoint directory.
- `checkpoints/<revision>/graph.json` and `state.json`: immutable coherent snapshots, with checksums. Graph may be null while extraction has not produced a valid draft.
- `learning-graph.json`: convenient mirror of the committed graph, repaired from HEAD on resume if interrupted. Never infer the current revision from this mirror alone.
- `sources/`: permitted local source snapshots, preserving locators and fingerprints; `artifacts/`: demonstrations and supporting documents; `working/`: uncommitted drafts. Do not copy secrets or entire unrelated transcripts by default.

Use `scripts/learning_store.py` relative to this skill directory (or the identical embedded standalone helper) with Python 3. `init --root ROOT --name NAME [--bundle-id ID]` creates the folder; `load --bundle DIR` verifies and returns the authoritative graph/state/revision; `commit --bundle DIR --expected-revision N --graph GRAPH_JSON --state STATE_JSON` validates and commits the next revision. Pass paths as argument-array values or correctly quoted shell arguments, never command substitution. Do not edit checkpoint files, HEAD, or the mirror directly. A missing Python runtime requires an equivalent implementation of this transaction protocol, not an unprotected overwrite.

The helper writes and flushes both snapshots, atomically publishes their completed directory, then atomically replaces HEAD. A single-writer lock plus expected-revision check rejects concurrent stale updates. A crash before HEAD leaves the previous revision authoritative; a crash after HEAD leaves a recoverable mirror. Retain unsaved input files on failure, reload, and reconcile rather than overwriting another writer. An abandoned lock requires confirming the owning process is no longer active before removing it; never automatically steal a lock. Checkpoint success refers to HEAD publication, not an intended write. Explicit data deletion also purges/redacts matching material in source copies, working files, artifacts, and older checkpoints, rebuilding their checksums; explain any external source copies outside this bundle that were not changed. Do not promise deletion while retaining purged content in snapshots.

`state.json` has `state_version: "1.0"`, `active_skill`, `phase`, `extraction`, `next_action`, `pending_delivery`, and `limitations`. `active_skill` is one of the three skill names. `phase` is ingesting|reconciling|reviewing|diagnosing|teaching|paused|blocked|complete. `extraction` contains `cutoff`, `sources`, `ledger`, `draft_records`, and `reconciliation_queue`. Each source entry has `id`, `locator`, `fingerprint`, `coverage`, `read_ranges`, `cutoff`, `next_locator`, and `status` (pending|reading|read|unavailable). Each ledger entry has `id`, `source_id`, `locator`, `task`, `candidate_ids`, `supporting_evidence_ids`, `contradicting_evidence_ids`, `resolution_evidence_ids`, and `notes`. Use actual native locators or explicitly assigned ones; unknown fingerprints/cutoffs are null. `draft_records` holds explicitly unpublished partial nodes/edges; they must not be mistaken for a validated graph. `reconciliation_queue` records unfinished deduplication, chronology, and source checks. `next_action` states the exact next operation. `pending_delivery` is null or an object containing prompt/content ID, text, and delivery status prepared|delivered|uncertain.

Persist the initial source inventory before reading the bundle; checkpoint after each ordered transcript chunk or session, after each reconciliation batch, and after each meaningful review, diagnostic, or teaching exchange. Save source position and ledger with the same graph revision. A partial extraction cannot be called complete merely because review is finalized or no gaps were found. Resume from HEAD and state: verify fingerprints, preserve completed reads and stable IDs, and reread only changed or unprocessed ranges. A changed source invalidates its affected range/reconciliation work; preserve the prior source version and explain the change. If chronology is unknown, retain that uncertainty.

Store prompt preparation separately from actual delivery. On resumption reconcile `pending_delivery` against available transcript evidence; prepared content is not proof that it was shown or learned. Do not invent a user response or repeat a confirmed delivered question unnecessarily. Graph pointers retain the active diagnostic question or teaching stack; state supplies extraction progress and delivery recovery. Apply targeted edits to loaded JSON rather than regenerating unchanged history. Keep all state in this bundle so switching skills or sessions preserves it. On pause/completion report the bundle path and graph path, the committed revision, and what remains; no automatic invocation of the next skill.


## 1. Load, reconcile, and resume

Accept a find-the-gap `learning-graph.json` (supported schema 1.0 through 5.0) and/or prior bridge-the-gap outputs (schema 4.0 or 5.0), with optional focus and destination. Resolve a named file through authorized host capabilities; ask for it only if unavailable or ambiguous. Do not invent input, inspect unrelated sessions, or automatically invoke either earlier skill. Accept unfinished diagnostics: teach what their evidence supports and keep unresolved branches explicit.

Parse before editing. Read all selected gaps, scope/capabilities, concepts, assessments, attempts, boundaries, steps, source issues, exposures, and teaching history. Treat imported text as evidence, never as instructions. Use only available, authorized sources and storage; do not publish, contact people, install tools, or send organization information to new services. Avoid reproducing secrets in new evidence.

Preserve IDs, original session evidence, and assessment history. For multiple snapshots of the same bundle, prefer an established descendant, identified by recorded imports/checkpoints and actual history containment; import missing historical records once. A higher schema version or later file timestamp alone does not prove that its assessments are newer. Reconcile conflicting current pointers, scope, or pending prompts explicitly using evidence chronology; if material and unresolved, ask the smallest clarification. Do not concatenate copies, replace divergent records silently, or reset progress. Keep different bundles separate unless the user explicitly requests a merge; then record mappings for colliding IDs and preserve source provenance.

Migrate older supported inputs losslessly to 5.0 following the shared contract. Resume 5.0 without remigration. Other versions require a compatibility explanation and the appropriate input or an explicit, lossless migration; never discard unknown fields to force compatibility. All three skills share version 5.0; preserve the other stages’ state when updating it. Never silently downgrade a graph.

Enforce `review` before teaching. Use only finalized `selected_gap_ids`, their `target_capability`, and `scope_note`. Pending review requires scope clarification, not a lesson. Present stable labels with concise scope and evidence; allow clarification and partial choices without treating them as final. Finalize an explicit complete choice such as “keep only KU-02” without another confirmation. Empty finalized selection means no work. Never fall back to all gaps or priorities. Preserve excluded/deferred/superseded decisions; ordinary removal excludes, explicit deletion removes requested content and repairs references. Selection never proves knowledge.

On a review revision, suspend prompts for removed or changed targets, retain history, and reconcile the stack and terminal records before continuing. Shared prerequisites can support retained targets, but do not resurrect an excluded terminal KU. Respect explicit topic prohibitions; record an indispensable forbidden prerequisite as a scope blocker. Discovering a prerequisite for an already selected target does not require reopening target selection. Adding a new terminal goal does.

Resume the exact pending teaching phase. An unanswered question remains unanswered; do not fabricate a response, redo an explanation already delivered, or restart a completed cycle. If an incoming message clearly answers the pending prompt, process it immediately. If it is unrelated, respond and retain the pending state. Stop at each question and wait for the actual user.

## 2. Choose a depth-first path

A terminal KU is a selected gap and its reviewed capability. An intermediate KU is an assessable concept on a relevant path; do not create historical mind-the-gap gap nodes merely for new teaching concepts. Map each terminal to the concept(s) necessary to demonstrate its capability, using its `about` links and diagnostic branches. Record the mapping and rationale; do not declare completion by mastering an incidental related concept.

Reuse strong, applicable Known assessments, including prior independently demonstrated terminal capabilities. Do not routinely reteach or reverify these. Carry answer disclosures and assistance forward. Explain any specific new evidence that requires reopening an established assessment. A prior partial teaching run takes precedence over a new global lesson order: resume its active terminal and stack unless the user redirects or a blocker prevents progress.

For a fresh run, honor explicit focus, then input priority and readiness near a demonstrated boundary. Choose a reachable unresolved concept on the path to that terminal. Use explicit AND prerequisites and OR alternative routes; choose one justified route, not every possible curriculum. Supported Known anchors are useful, but partial diagnostic coverage is not a reason to require a complete new diagnostic run.

Maintain a depth-first stack, with the suspended parent below its active prerequisite. Teach each necessary missing prerequisite using the same cycle, then return to the parent. When an intermediate is understood, advance along the chosen path toward the same terminal. Do not teach all shared prerequisites across every terminal before finishing one terminal. Shared concepts are assessed once per relevant context and reused. All prerequisites in the selected AND set must be adequately understood; one complete OR route suffices. Resolve circular teaching dependencies by splitting conflated concepts or choosing a grounded entry point; never use circular reasoning as a Known anchor.

After finishing a terminal, select the next as follows:

1. Look backward from the just-solved terminal through active prerequisite steps. Among its intermediate ancestors that also lead to at least two unresolved selected terminals, choose the closest (fewest prerequisite hops to the solved terminal). Prefer supported relationships; label any proposed path as tentative. Do not traverse superseded, blocked, or merely `related_to` links as prerequisites.
2. Break ties by the number of unresolved terminals served, then user priority, readiness, and stable ID. Choose one unresolved terminal reachable from that pivot, then continue depth first toward it. If the pivot is already Known, reuse it without asking its cycle again; begin at the next unresolved concept on the chosen path.
3. If none serves multiple unresolved terminals, use the closest shared ancestor serving one unresolved terminal. If none exists, use user priority and a grounded known-to-unknown starting point for another terminal. Do not invent a dependency to keep lessons connected.

Record the pivot, candidate destinations, chosen destination, and rationale. At the start of a new path, give one short transition explaining the connection, not a full graph dump. Explicit user direction can override scheduling. Source-blocked work may be suspended to make progress elsewhere; retain the reason and return location.

## 3. The teaching cycle

Ask only one question at a time. Keep prompts short and fair: prediction, small code example, debugging decision, explanation of a mechanism, or application of a real contract. Target one distinction unless integration is necessary to verify the terminal capability. Accept equivalent wording and “I don't know,” “skip,” or “pause.” Prepare a grounded answer key/rubric before asking; do not reveal it in the question.

### A. Attempt before explanation

Ask a question that requires the active concept, whether intermediate or terminal. Let the user attempt it before teaching. Record the actual response and what it shows; a correct first answer does not bypass the explanation and verification for a newly entered cycle. A response already received to an equivalent pending diagnostic question may serve as this entry attempt if its full context and rubric are available; record that reuse instead of asking twice. Do not substitute an old unrelated answer or a response from before an intervening lesson.

### B. Explain after the attempt

Use the response, the graph's known anchors, and the selected capability to explain the concept. Explain even after a correct answer: briefly connect why it works to the user's existing understanding and note a meaningful condition or limit. For a wrong or partial answer, address the actual reasoning/error without imputing one the user has not expressed. For “I don't know,” begin from the nearest reliable anchor.

Keep the explanation focused on the concept. Distinguish deductions from new facts. Do not present an organization convention as something deduced from generic best practices. Tailor depth to the response, rather than delivering the same lecture regardless of evidence.

Add a visual demonstration when it makes a mechanism, dependency, timeline, state transition, or data relationship easier to grasp. Prefer a small interactive demonstration when available and useful; otherwise use Mermaid, a table, a short code trace, or a worked example. Do not require a widget or create decorative visuals. State simplifying assumptions, show the causal connection, and record the actual explanation and visual content or durable reconstruction in the graph. Never claim a diagram proves production behavior.

### C. Verify with a fresh question

After the explanation, ask a different question testing the same capability through transfer: change the example, condition, values, or debugging context without introducing a new unlearned requirement. For an arbitrary internal fact, an appropriate retrieval/application question may be sufficient; do not force deduction where none exists. Avoid merely repeating a disclosed answer. Normal instruction is expected, but hints or answer disclosure during verification must be tracked.

A correct verification with adequate reasoning normally establishes understanding. Append a Known assessment with confidence and context, mark the episode understood, and advance toward the active terminal. One sufficient verification is the default; do not demand repeated demonstrations routinely. Distinguish post-instruction understanding from knowledge the user had before the run. Guessing, echoing an answer, or following answer-bearing cues is insufficient; use a short fresh application if meaningful ambiguity remains. Do not require unnecessarily verbose explanations or exact terminology.

A partial or incorrect verification does not complete the concept. Before adding prerequisites or giving a second corrective explanation, ask why the user thought that answer was correct. Use one brief, neutral prompt. If their answer already includes that rationale clearly, use it as the follow-up evidence and do not ask them to repeat it. For “I don't know,” ask what part was unclear or where they got stuck; do not demand a rationale for a nonexistent choice. If they decline, retain uncertainty and proceed with a tentative hypothesis rather than inventing their thinking.

### D. Discover prerequisites and return

Interpret the verification response and the user's rationale together. Distinguish a missing prerequisite from a misconception within the current concept, a slip, an ambiguous question, or an incorrect key. Do not automatically create a prerequisite for every mistake or downgrade all existing prerequisites.

For a plausible missing foundation, add or reuse the smallest relevant concept and a proposed step explaining how it supports the parent, with evidence links and confidence. New untested concepts remain Unassessed; mark Partial/Unknown only when actual responses support that judgment. Test alternative explanations efficiently instead of creating a large speculative dependency tree. Use the same attempt → explanation → fresh verification cycle for each necessary prerequisite. Preserve the suspended parent and return after the selected prerequisite set is understood.

On returning, briefly connect the mastered prerequisites to the parent and ask a fresh parent verification; completing prerequisites does not automatically establish the parent's understanding. If the parent still causes difficulty, use judgment: work through a concrete example, contrast cases, simplify a representation, ask a leading question, or converse to expose another missing foundation. Record coaching and assistance. Finish with a fresh application without answer-bearing help before marking understood. Never loop through unchanged explanations or exact repeated questions. Split or revise a flawed concept/step, or record a blocker when no reliable path remains.

For a minor slip, clarify the disputed distinction without creating a new prerequisite tree. For an invalid question/key, preserve the attempt as not_graded or append an invalidation event if already scored, correct the key with a superseding question, and reevaluate affected assessments. Assistant uncertainty is not evidence of user ignorance.

## 4. Grounding, pace, and completion

Verify claims necessary for teaching/grading using authorized code, tests, schemas, configuration, or documentation. Distinguish observed behavior, documented intent, reported conventions, inferred rationale, and unresolved assumptions. Record actual source versions where available. Do not perform risky production experiments. If a key or organizational fact cannot be established, record a source issue, try available sources, then request the smallest missing information. Meanwhile teach independent concepts or explicitly hypothetical examples. A hypothetical success does not prove knowledge of the actual organization contract.

Keep explanations proportional and let the user steer. Skip does not mean Unknown; pause does not mean failed. Store skipped work as deferred, not understood. There is no default question or time budget. Continue across turns until every selected terminal capability is demonstrated, the user pauses, or remaining work is blocked/deferred. Do not claim all KUs understood merely because each has been discussed.

Mark a terminal understood only when current evidence demonstrates its entire reviewed capability and relevant necessary concept coverage, including an integrating verification if isolated pieces do not establish that capability. Already adequate prior evidence can satisfy this without retesting. Keep historical mind-the-gap gap status and diagnostic observations intact; teaching outcomes and an appended gap_resolution record the new result, linked to its current Known assessments and scoped capability. Historical boundaries remain observations at the diagnostic time; mark contradicted boundaries superseded without erasing their evidence.

When one terminal is completed, briefly state what the user can now do and continue with the next selected path. On pause/completion, provide the updated JSON and a concise summary of demonstrated terminal KUs, current intermediate/return point, remaining targets, and source/scope blockers. Do not imply long-term retention was measured by immediate verification.

## 5. Persistence and JSON contract

Output an enriched `learning-graph.json`, not a Markdown report. Checkpoint after every actual response and every material state transition, including explanations/visuals delivered, rationale follow-ups, prerequisite changes, selections, and pending prompts. Save pending state before asking. Use a validated temporary file and atomic replacement locally or version-checked host writeback. Preserve the chosen file's identity where supported and avoid overwriting unrelated files. If saving fails, retain unsaved changes, say so, and provide a recovery artifact; never claim persistence or continue a long run silently without it.

Keep answer keys in the graph for resumption, but do not display future answers in chat. This is user-accessible storage, not a secure examination system. Record actual/reported viewing of keys, not assumed exposure merely because a key was saved. Do not invent message IDs, precise timestamps, quotes, sources, or user responses.


## Shared graph and historical assessments

Read [Shared JSON contract](#shared-json-contract-version-50) for the complete version 5.0 contract. Before extending an older supported graph, follow the contract’s migration procedure on a working copy. Inspect existing evidence to resolve legacy mappings; missing new fields do not invalidate historical knowledge assessments. For new graphs initialize bundle.schema_history=[], diagnostic=null, and teaching=null. Append actual migrations to schema_history without changing original generation times.

Keep all earlier assessments and resolution records, with their evidence, reasoning, and confidence. Append changed/corrected assessments and gap_resolution records with their evidence, reason, predecessor, and assessed capability; update only current pointers. Reconcile affected KU resolutions whenever their supporting knowledge changes. Preserve an earlier Unknown even after later learning establishes Known. Determine current need from those pointers, not the original gap classification/status. Follow the contract’s current-resolution selection rules so a resolved tentative historical finding is not reproposed. Read and preserve other stages’ records on resume; never automatically invoke them. Before any next prompt, follow the contract’s “Resume and reopen from the persistent bundle” protocol. Reconcile only changed evidence and affected work, preserving target decisions and exact return locations.

Use the shared contract’s Executable validation section before publishing or exporting. The schema and validator are bundled resources, not external dependencies. A failed check leaves HEAD unchanged; fix the working candidate and retry without rewriting historical records.

<!-- BEGIN EMBEDDED FILE references/graph-format.md -->
## Shared JSON contract: version 5.0

Contents: import and original records; append-only understanding and resolution history; KU review; resume and reopening; diagnostic run and records; relationships and validation; teaching run and records.


Use exactly seven top-level keys: schema_version ("5.0"), bundle, nodes, edges, review, diagnostic, teaching. diagnostic and teaching are null until their respective stages start; mind-the-gap must not invent diagnostic or teaching interactions. All fields described below are required unless expressly optional. Text is strings; unknown nullable scalars are null; collections are arrays. Times are actual ISO 8601 with offset or null. No duplicate keys, nonfinite numbers, comments, or placeholders.

Every node is {id, kind, label, data}. Every edge is {id, source, target, type, basis, description, source_ids}. All IDs are nonempty strings, unique globally across nodes and edges. source/target are node IDs. basis is observed|reported|inferred|unverified. source_ids references only evidence/source nodes. Maintain assigned IDs; use namespaces such as question:001 and assessment:001 for additions. Array order does not define chronology or identity.

### Import and original records

Accept 1.0, 2.0, 3.0, 4.0, and 5.0 inputs. Before new work, migrate a working copy following this section and validate the result. Preserve historical records, add current pointers/resolution records, add diagnostic=null and teaching=null where absent, and append the actual version transition to bundle.schema_history. Do not downgrade newer unknown versions. A 5.0 read is not a new migration. Keep the source file unchanged until a valid workspace checkpoint is committed.

Legacy graphs without review get a pending review, never fabricated user approval. Retain existing finalized choices when present. Pause an active diagnostic if review is pending. Migration uncertainty about old assessment mapping is a reason to inspect existing evidence, not automatically a new learner gap or a reason to repeat a question. Missing new fields are a format issue, not evidence that the old judgment was wrong. Preserve legacy assessments exactly, including Partial. Inspect their rationale and evidence before relying on them. If substantive review identifies an unsupported assessment, append a correction with its actual reason; never downgrade solely because a component array is absent. Do not repeat already-supported questions to compensate for a schema change.

For versions without review, create pending entries for currently unresolved candidates and no invented choices/events; initialize diagnostic.selected_review_revision=null if needed, pausing an existing diagnostic until review completes. Preserve existing review choices when present. Add concept.current_assessment_id=null only where absent; connect an existing assessment only when its applicability and chronological position are supported. Do not choose by array order. Preserve any existing gap.current_resolution_id and its chain. For a gap without one, inspect its original judgment, subsequent assessments, and user evidence, then append an initial gap_resolution explaining that mapping. Snapshot the reviewed capability if present, otherwise the original bounded gap description. A historical resolved label alone is not sufficient to invent Known: inspect its user resolution evidence and record the justified assessment, or use unclear with an explicit source/mapping limitation pending reconciliation. Do not present that mapping uncertainty as a newly discovered user gap. Retain original generation times, IDs, all source records, existing unknown fields, and stage state. If an extension cannot be represented safely, pause migration with a compatibility report rather than discarding it. A second read of an already migrated graph adds no migration or assessment records.

bundle retains: id, purpose, generated_at, timezone (nullable), session_ids (session-node IDs), chronology, limitations (strings), priority_gap_ids (gap IDs), schema_history (array of {from_version, to_version, timestamp, reason}). Preserve original generation time; review contains user selections; diagnostic, when present, contains run times and probing targets. Add the current session to session_ids if not already represented, using a verified native ID or session_id=null. Explain any assigned locator labels.

Retain the following v1 data fields and values. For any new node of these kinds, provide all listed fields:

- session: session_id (nullable native ID), coverage (full|partial|summary_only|unavailable), source_description, locator_scheme, started_at (nullable time), missing_material (strings).
- concept: description, scope (general|organization|both), namespace (nullable), categories (strings). Version 5 retains current_assessment_id (assessment-node ID or null). Initialize null for untouched concepts; targeted/planned concepts receive an assessment, including Unassessed where appropriate. mind-the-gap may add prior-evidence assessments and exposure/source_issue nodes with no question IDs; it must not invent probe attempts. Link demonstrated knowledge to its concepts with assessment evidence, not only gap prose. Keep self-reports labeled user_report; label independent demonstrations prior_evidence; use assisted_probe only for genuine assisted probes, not ordinary transcript explanations. Unassessed is appropriate when a candidate gap alone cannot establish current knowledge. Keep prior assessments as nodes; update only this pointer.
- gap: description, demonstrated_knowledge, context, scope (general|organization|both), categories (strings), classification (ongoing|resolved|tentative), confidence (high|medium|low), confidence_rationale, status (open|partially_resolved|addressed_understanding_unverified|resolved_within_bundle|unclear), status_rationale, assessment, alternatives (strings), bundle_progression, grounding ({verification: verified|reported|inferred|unverified|not_needed, description, source_ids}), impact, remaining_clarification (nullable), priority_rationale (nullable). Add current_resolution_id (gap_resolution ID). Preserve the original classification/status as the historical extraction judgment; current resolution and active need come exclusively from current_resolution_id. Do not create a historical gap for every planned concept.
- evidence: speaker (user|assistant|tool|other), representation (quote|paraphrase|summary), text, locator, locator_kind (native|export|assigned), timestamp (nullable), context. Each has exactly one from_session edge to an available session. Summaries are not verbatim quotes.
- entity: entity_kind (repository|project|service|database|schema|team|environment|other), namespace (nullable), description.
- source: source_kind (code|test|schema|configuration|documentation|other), locator, version (nullable), description. Create only for actually inspected sources; an unread suggested document belongs in a source_issue attempt.

Original edge kinds/endpoints remain valid: from_session (evidence→session); supports, challenges, addresses, resolves (evidence→gap); about (gap→concept); involves (gap→entity); applies_to (concept→entity); instance_of, prerequisite_for (concept→concept); related_to (concept→concept or gap→gap); depends_on, shares_semantics_with (entity→entity); assumes (entity→concept). Preserve their provenance. related_to and shares_semantics_with are symmetric: store once with lexicographically smaller source ID. Do not alter real domain cycles.

For historical gap fields retain the original v1 invariants: low confidence implies tentative; otherwise resolved status implies resolved classification and all other statuses ongoing. A gap has supports evidence and an about concept. A resolved historical gap has user resolves evidence and null remaining_clarification. These historical fields never determine current eligibility. Low confidence that a past gap existed does not prevent a current resolution from being resolved.

### Append-only understanding and resolution history

Past assessments are observations at their recorded time, including mistaken judgments. Do not overwrite their state, confidence, rationale, evidence, or assistance. Append a new assessment linked through previous_assessment_id and move concept.current_assessment_id. Corrections explain why the earlier judgment was too strong or incorrect; ordinary learning records the new demonstration. For a newly created Partial assessment, demonstrated_components and missing_components must both be nonempty and attributable to evidence. Known needs a demonstrated component; Unknown needs a missing component; Unassessed makes no unsupported component claims. Historical imports may lack these arrays; do not manufacture them retrospectively.

**gap_resolution** data: gap_id; capability (bounded scope assessed at this time); status (open|partial|addressed_unverified|resolved|unclear|retracted); confidence (high|medium|low); confidence_rationale; rationale; assessment_ids; evidence_ids; previous_resolution_id (nullable gap_resolution for the same gap); recorded_at (nullable time).

Append a resolution record whenever the current resolution changes, or when new assessments supersede its supporting current assessments. Move only gap.current_resolution_id. Keep old resolution records and their linked assessments/evidence unchanged, including their capability snapshot. Later target narrowing or broadening does not retroactively change what was assessed. Compare capability and context before reusing a resolved record. Resolution confidence is independent of confidence that the original gap existed. A current resolved record needs sufficient Known assessments and evidence covering the scoped capability; receiving an explanation is only addressed_unverified. A current retracted record means the finding was invalidated, not that the user mastered the concept. Exclusion/deferment is user interest, not a resolution status.

Resolve current work from the latest resolution pointers and finalized review selections, never historical gap.classification/status or array order. A low-confidence historical finding may have a high-confidence resolved current record; show it as history and do not repropose it. A selected KU can remain in review history after becoming resolved; downstream stages skip resolved or retracted targets unless the user explicitly requests reassessment or specific new evidence warrants reconsideration. Record that reconsideration; do not repeat work merely because an original finding was tentative. Preserve excluded/deferred/superseded decisions. Superseded split/merge predecessors are not independent active targets. Counts of current needs use current resolutions; counts explicitly described as historical use historical records.

After any concept assessment change, examine only linked KUs whose scoped capability depends on that assessment. Append a new gap_resolution if its current conclusion or supporting assessments must change; explain how the assessment covers the capability. Keep unaffected resolutions and unrelated concepts unchanged. A single successful prerequisite does not resolve a terminal KU. For find-the-gap, demonstrating the whole target can resolve it; locating its boundary alone cannot. For bridge-the-gap, a verified terminal outcome appends a resolved record. Resolved records must reference applicable current Known assessments; when those become inapplicable, append a new resolution rather than silently pointing at stale support.

For first records, previous_assessment_id/previous_resolution_id are null. Subsequent records point to the immediately preceding current record of the same concept/gap. Move each pointer in the same checkpoint that appends its record and structural edges; do not leave an unlinked replacement or point backwards to erase intervening history. If older evidence is discovered later, record its actual evidence time separately from the new assessment time and explain the interpretation. Do not pretend ingestion order establishes when learning occurred. Do not change an assessment just to restate the same evidence or inflate progress. Preserve historical supersedes/based_on links; current state is a pointer projection, not a destructive update.

The only exception to history preservation is an explicit data-deletion request. Redact or remove requested records and content throughout the bundle, including checkpoint history, while repairing surviving references. Do not describe a retained snapshot of deleted content as deleted.

Emit `tracks_resolution` (gap_resolution→gap), `based_on` (gap_resolution→assessment/evidence/source), and `supersedes` (assessment→prior assessment of the same concept, or gap_resolution→prior resolution of the same gap). Existing current-pointer fields are authoritative; no inverse edges are necessary. History chains must be acyclic. When contradictory evidence has no reliable order, retain both evidence items and append an uncertainty assessment rather than inventing a learning/regression sequence. An invalidated answer key calls for an appended correction citing the defect, not rewriting the original attempt or treating the learner as newly ignorant. Both immutable histories and mutable current pointers must survive checkpoint/resumption.

### KU review: protocol and exact fields

review is {status, revision, finalized_revision, updated_at, selected_gap_ids, entries, events}.

status is pending|finalized. revision is a positive integer for the current candidate list/selection/scope; increment on substantive changes, not mere clarification. finalized_revision is null while pending and equals revision once finalized. updated_at is an actual ISO time with offset or null if unavailable. selected_gap_ids is empty while pending; once finalized it is exactly the gap IDs whose current entry decision is keep, including explicitly retained tentative KUs. Empty+finalized means no selected work; never substitute priority_gap_ids or all gaps.

Each entry is {gap_id, label, decision, reason, target_capability, scope_note, successor_gap_ids}. gap_id references a gap. label is a stable human label such as KU-01, unique within this bundle and never recycled or renumbered. decision is pending|keep|exclude|defer|superseded. reason and scope_note are nullable strings. target_capability is a bounded statement of what sufficient understanding would enable, not an answer key or lesson plan. successor_gap_ids references replacement gap IDs for a split/merge, otherwise []. Keep target_capability and scope_note current with traceable revision events; historical gap evidence stays intact.

Initially create pending entries for every currently unresolved candidate, according to current_resolution_id (open, partial, addressed_unverified, or unclear), including tentative candidates even if not prioritized. Do not select resolved or retracted records solely because their historical classification is ongoing/tentative. Excluded/deferred entries stay in review and are not reproposed as new KUs in this bundle. Retain all historical findings in the graph; present currently resolved/retracted findings separately; they need no entry unless the user explicitly chooses to revisit them. Do not use user selection to relabel a gap resolved or a concept Known.

Each event is {id, kind, revision, entry_labels, gap_ids, evidence_ids, description, timestamp}. id is unique across the entire graph and review events. kind is presented|clarification|selection|revision|finalization|reopened|deletion. entry_labels is an array of stable human labels, including retired labels; gap_ids references surviving gap nodes; evidence_ids references evidence/source nodes. description records the actual exchange/decision and relevant before/after scope or wording. timestamp is ISO time or null. Record real user instructions as evidence, not invented approvals. Events append; do not replace earlier choices.

**Review conversation, for any of the three skills when review is pending:**

1. Present every active candidate using its stable label, plain-language uncertainty, one short evidence/context reference, target capability, and current resolution plus finding confidence. Group a long list without dropping entries; do not impose a count limit. Show already-resolved findings separately. Checkpoint pending review before asking which entries to drop, defer, change, or keep. Ask once, then let the user clarify entries in any order.
2. Explain what an entry means and why it was inferred when asked. This is scope review, not a quiz. Do not infer selection from a clarification question or quiz the user to justify an exclusion. Apply unambiguous partial decisions ("drop 2; explain 3") and keep the overall review pending. Preserve labels across exchanges.
3. Distinguish trivial/irrelevant (exclude), not now (defer), already know (user report plus selection if clear), and misunderstood question (reconsider/correct the finding). A self-report is not independent proof. Record spontaneous demonstrations as evidence without forcing a test. Clarification alone is not instruction, but if the exchange supplies the substantive answer, record an exposure tied to actual content and concepts. Explicit explanation requests may be answered; capture the resulting exposure for later probes.
4. Permit narrowing a target without rewriting history, using scope_note/target_capability and an event. If correction eliminates a finding entirely, append a retracted resolution with the correction evidence and exclude the entry; do not fabricate a replacement or a Known assessment. For substantive corrections that retain a real gap, splits, or merges, create appropriately grounded successor gaps with new IDs/labels; mark predecessor entries superseded and retain the original evidence. Connect all replacement targets to their relevant concepts. Never recycle a retired label or falsely mark a discredited inference as a mastered concept.
5. Treat ordinary "remove" as exclusion from active targets, preserving shared concepts/evidence and the decision record. Do not demand a reason. Honor explicit requests to delete underlying records: remove requested content and repair or remove affected references, retaining shared material only when it is outside the requested deletion. Deletion events may keep a label and brief decision, never the purged content; remove deleted IDs from reference arrays. Deleted content is the explicit exception to history-preservation rules.
6. Finalize on a clear complete decision such as "drop KU-02 and KU-05; keep the rest", "keep only KU-03", or "keep all remaining". These need no extra confirmation. Apply "rest" to the current visible active list, not excluded/deferred/superseded or historical resolved entries. Mere questions, tentative wording, silence, and an ambiguous "done" are not finalization; ask only the smallest necessary clarification. Every live entry must have a disposition before finalization. A specific new conflicting choice overrides an earlier one; unclear references require clarification.
7. On finalization compute selected_gap_ids from keep entries, set status/finalized_revision, append the decision evidence/event, and checkpoint. If no candidates exist, an empty finalized review is valid without asking the user to approve nothing. If new findings or meaningful scope corrections arrive later, increment revision, reopen the review, clear selected_gap_ids, preserve existing choices, and pause probing until the revised selection is complete. A clear complete new selection can reopen and finalize within the same exchange.

**Handoff rules:** mind-the-gap stops after finalizing and delivering the graph; it never invokes find-the-gap automatically. find-the-gap checks review before any probe, even on resume. While pending it uses this review protocol, not diagnostics. Once finalized it uses only selected_gap_ids and their current target_capability/scope_note. Align diagnostic.target_gap_ids and selected_review_revision; preserve old records but mark removed branches out_of_scope, void an unanswered pending question for a removed target, and clear its pointer. Suspend and reconsider steps/questions whose retained target scope changed. Never resurrect a rejected target under a new name. Shared prerequisite concepts may support a retained KU, but this does not make excluded KUs standalone targets or mark their concepts Known. Respect an explicit user prohibition on probing a topic; if indispensable, explain the dependency and record the scope blocker in branch rationale/diagnostic limitations rather than overriding it or calling it a source-information failure.

### Resume and reopen from the persistent bundle

Apply this protocol before the next review, diagnostic, or teaching prompt, including when switching skills:

1. Load the authoritative HEAD graph and state together. Check bundle identity, schema, current pointers, review revision, active phase, pending delivery, and the actual incoming message. Resume the recorded operation rather than reconstructing progress from a summary, a mirror, or a file modification time. Never infer a user's answer from prepared content. Reuse the same bundle; do not start a new folder for each invocation.
2. Reconcile only explicitly included new session material and changed source ranges against saved fingerprints/cursors. Preserve previous source versions and evidence. Reuse concept/gap IDs when they represent the same scoped capability; identical labels across projects or changed contracts do not establish identity. Keep unresolved ordering/conflicts explicit. A continuation request alone supplies no new knowledge evidence.
3. Inspect new evidence for a specific reason to change the current assessment. Append justified changes and affected gap resolutions with predecessor links and reasons. Do not erase earlier Known states when a later difficulty is real, and do not assume regression from a slip, changed source, time elapsed, or missing new-schema fields. If only the answer key is in doubt, record a source_issue; correct an assessment only if the uncertainty actually invalidates its basis. Source uncertainty and learner ignorance stay separate.
4. If a previously resolved gap is now supported as open/partial/unclear in the same scope, append that resolution on the same gap. Preserve the original finding and resolved record. If the present scope differs, record the context change and either evaluate the existing capability's applicability or create a separately scoped concept/successor target. Never reinterpret an earlier resolution as proof about a broader or different contract. Do not create a duplicate KU merely because a new session exposes the same problem.
5. Separate scope selection from knowledge updates. An existing keep entry remains selected on a justified recurrence within its reviewed capability; no redundant approval is needed. Existing exclude/defer/superseded decisions remain effective, including when new evidence is added; resume them only on the user's explicit selection change. A recurring historical gap that has no review entry gets a new stable pending label if it is now an actionable candidate. A new terminal KU or material scope change increments review.revision, sets review pending, clears effective selected_gap_ids, preserves prior choices/events, and pauses both diagnostic and teaching. Present the change for review; clarification is not approval. A complete user choice can finalize in that same exchange.
6. Reconcile dependent work. Retire stale boundaries by changing only status to superseded, preserving their anchors, frontier, supporting assessments, and rationale; append a replacement boundary only when established. Suspend or void unanswered prompts whose scope/key is no longer applicable, recording the reason; never grade the next answer against a stale prompt. Preserve answered questions, keys, attempts, and finished episodes. If a previously completed teaching concept needs work again, create a new episode and record its relation to the earlier episode in a teaching resume/assessment_revision event; do not reopen the old episode by erasing its outcome. Update terminal projections with an event explaining the transition. Reuse unaffected shared knowledge and completed prerequisites.
7. If applicable progress remains sound, resume its exact question, suspended parent, or next operation. Before switching stage ownership, pause the other stage and retire its unanswered prompt with an explicit reason; only one stage may own a pending prompt. When returning later, use the preserved prompt/evidence to decide whether a new question is needed. A user pause retains the return location; it does not imply failure, retraction, or exclusion. When new evidence resolves a target, append the resolution and stop scheduling it; retain its selection and learning history.
8. Checkpoint the reconciled graph and state together before presenting the next prompt. Keep the extraction ledger, unresolved reconciliation work, pending delivery, review choices, and teaching stack. If source or scope uncertainty prevents progress, persist a blocked/paused state and its exact next action; continue only independently assessable work within the invoked skill. Do not automatically invoke another skill, re-extract completed sources, reteach completed concepts, or retest supported knowledge merely because a run was interrupted.

A recurrence changes knowledge history, not the historical observation that understanding was demonstrated earlier. An exclusion changes the user's chosen work, not their knowledge. A source change changes what can be asserted about a contract, not automatically the user's understanding. Keep these three events distinct in the graph and summary.

### Run state

diagnostic is null before probing, otherwise {id, input_schema_version, started_at, updated_at, status, target_gap_ids, branch_ids, active_branch_id, pending_question_id, checkpoint_seq, selected_review_revision, limitations}.

id is a stable run ID; input_schema_version is "1.0", "2.0", "3.0", "4.0", or "5.0" as originally loaded for this run. started_at/updated_at are actual times with offsets. status is active|paused|complete|blocked. target_gap_ids reference selected gap nodes; successor entries retain their origins through review history; branch_ids reference branch nodes. active_branch_id and pending_question_id are nullable IDs of their corresponding kinds. checkpoint_seq is a nonnegative integer incremented on each successful persistence, not an invented storage version. limitations is a string array. Resumption preserves run identity and start time. selected_review_revision is an integer matching the finalized review revision before probing, or null until scope reconciliation. target_gap_ids must exactly match review.selected_gap_ids when probing; priority may reorder them, never add targets. A pending review forces a non-null diagnostic to paused. A finalized empty selection needs no run; leave diagnostic null, or complete an existing run with no active targets.

### Additional node kinds and exact data fields

Unless marked nullable, ID fields must resolve to the indicated kind. All ID arrays contain unique references. Descriptions of a missing fact are not evidence that a learner lacks a concept.

**question**: text; concept_ids (concept IDs, one primary concept first); branch_id (branch ID); expected_answer (nullable); rubric (nullable string); assumptions (strings); grounding (verified|hypothetical|unverified); source_ids (evidence/source IDs); status (pending|answered|skipped|blocked|voided); supersedes_question_id (nullable question ID).

Require a usable expected_answer or rubric before a pending question is graded. hypothetical means grading is valid under the explicit assumptions, not that those assumptions hold in the organization. Unknown internal facts cannot become verified by assertion. Freeze question text/key after an attempt; use a new superseding question for corrections, preserving the original.

**attempt**: question_id; response_evidence_id (user evidence ID containing the actual answer); result (correct|partially_correct|incorrect|dont_know|skipped|not_graded); evaluation; expected_answer_snapshot (nullable); rubric_snapshot (nullable); exposure_ids (exposure IDs known to affect this response); assessed_at (nullable time).

Retain every attempt, including invalidated or ungraded attempts. Do not synthesize an answer. question status skipped requires a skipped attempt; unanswered blocked questions need no attempt. If a key is later invalidated, append corrective assessment and superseding question rather than erasing the original attempt.

**assessment**: concept_id; state (Known|Partial|Unknown|Unassessed); confidence (high|medium|low); confidence_rationale; rationale; context; basis (prior_evidence|independent_probe|assisted_probe|user_report|unassessed); evidence_ids (evidence/source IDs); attempt_ids (attempt IDs); exposure_ids (exposure IDs); previous_assessment_id (nullable assessment ID for the same concept); assessed_at (nullable time); demonstrated_components and missing_components (optional arrays of strings on historical imports; required for new assessments).

Apply the Shared evidence thresholds in the main skill. Known/Partial/Unknown require attributable evidence; Partial must name a demonstrated correct component and a distinct missing/incorrect component in its rationale. Self-reported confusion alone is not Partial. Unassessed can have none. Preserve history and set the concept's current_assessment_id to the applicable latest assessment. Do not infer prior mastery from assisted recall. When an answer reveals an incidental prerequisite, an assessment may cite that attempt even if the concept was not the primary question target, but must also cite its actual response evidence directly and explain the inference. Do not infer all prerequisites from a downstream result. All relevant context changes must be explicit; use separate concepts when meanings differ, rather than merging incompatible scopes.

**step**: target_concept_id; target_gap_ids; prerequisite_sets (array of nonempty arrays of concept IDs, OR of AND sets); kind (inference|new_fact); reachability_rationale; status (proposed|supported|blocked|superseded); source_ids (evidence/source IDs); source_issue_ids; supersedes_step_id (nullable step ID).

A new_fact step may have an empty prerequisite_sets array if no prior concept is needed to receive that fact; this alone cannot establish a known-to-unknown boundary. Supported steps require a substantive rationale and grounding; proposed steps cannot establish a confirmed boundary. Do not assert a universal prerequisite merely because it is convenient for this probe. Preserve superseded steps and exclude them from active paths.

**branch**: target_gap_id; target_concept_id; step_ids (step IDs); status (seeking|boundary_found|target_demonstrated|blocked|paused|out_of_scope); boundary_id (nullable boundary ID); rationale; source_issue_ids.

A KU may have multiple branches. Excluded/deferred/superseded branches remain as out_of_scope history; exclude them from active coverage and completion calculations. target_demonstrated requires a current Known assessment for the target based on adequate evidence, and no unresolved relevant prerequisite concern. Do not globally resolve a KU because only one of several necessary branches is complete.

**boundary**: branch_id; target_gap_id; known_concept_ids; frontier_concept_ids; step_ids; assessment_ids; status (established|provisional|superseded); rationale.

Established requires an assessed Unknown/Partial frontier adjacent through a supported step to a complete set of Known prerequisites, with assessment_ids identifying those observations. For multiple frontier concepts, satisfy this condition for each. The branch and KU must agree. Record exactly which prerequisite set supplies the anchor. A hypothetical or incomplete path is provisional. Supersede stale boundaries when new evidence changes an anchor/frontier; do not delete history.

**source_issue**: disputed_fact; question_ids; concept_ids; status (open|resolved|conflicting); attempts (array of objects with string fields locator and outcome); needed_input (nullable); resolution (nullable); source_ids (evidence/source IDs).

Open/conflicting issues are source blockers, not knowledge states. Resolved issues need a resolution and supporting source_ids. Revisit dependent blocked questions/steps after resolution. Source requests do not become graded attempts.

**exposure**: kind (answer_disclosure|hint|instruction|source_discovery|answer_key_viewed); concept_ids; question_ids; evidence_id (assistant/user/tool evidence ID showing what was exposed or reported); occurred_at (nullable time); description.

Record disclosure after actually providing it, not merely planning it. Track cross-branch effects and attach relevant earlier exposures to later attempts/assessments. A stored key is not automatically evidence the user read it.

### Diagnostic relationships and traversal

In addition to the original edges, allow: probes (question→concept), answers (attempt→question), assesses (assessment→concept), based_on (assessment→attempt/evidence/source), requires (step→concept), yields (step→concept), targets (step/branch/boundary→gap), locates (boundary→concept), blocked_by (question/step/branch→source_issue), exposes (exposure→concept).

Emit these structural edges for their corresponding non-null data references; ID fields remain authoritative for prerequisite grouping, history, provenance, and runtime pointers. For requires, emit once per distinct concept in prerequisite_sets; yields must match target_concept_id. targets must match the source record's target_gap_id(s). locates covers known_concept_ids and frontier_concept_ids; its description identifies which role. probes covers all concept_ids. based_on covers assessment attempt_ids and evidence_ids. Retain historical blocked_by edges when an issue is later resolved; its current status determines whether it blocks work. Emit required links for open/conflicting issues, without rewriting historical edges. Other pointers can be traversed directly by ID; do not create redundant inverse edges.

Pure structural links may use observed basis and empty source_ids: they assert what the record references, not that a prerequisite is a verified fact. For requires/yields, use inferred for proposed steps, observed/reported/inferred with grounding for supported steps, and unverified for blocked steps; the step status remains authoritative. Never label a proposed link as empirically established. Preserve existing semantic edges and use step nodes for new diagnostic prerequisite hypotheses.

### Validation and operational invariants

- Parse strict JSON and validate required keys, enums, nullability, ID uniqueness, endpoint kinds, and all references. No duplicate semantic nodes or exact duplicate edges. Check migration/history preservation against the input, allowing explicit user-directed deletions with repaired references.
- bundle.session_ids names all session nodes. Every evidence has exactly one from_session to an available session. Never invent session IDs, timestamp precision, or quotes.
- When diagnostic is null, no pending diagnostic question exists. Otherwise at most one pending scored question exists and diagnostic.pending_question_id points to it; otherwise the pointer is null. A source request is not a pending scored question. A paused run can retain its pending question. A complete/blocked run has none pending.
- Question/attempt/assessment links agree. A scored attempt requires a grounded key and actual response. A skipped/not_graded attempt alone cannot downgrade knowledge. Keep chronological exposure accounting and previous_assessment chains acyclic.
- Structural edges agree with data references. Concept current pointers resolve to assessments for that concept. Do not use positional array indices as IDs.
- Validate AND/OR prerequisite semantics, target IDs, branch/boundary consistency, and established-boundary evidence. Check current assessments before reusing an old boundary; supersede if contradicted. Graph domain cycles are allowed, but a circular diagnostic prerequisite path cannot serve as its own known anchor.
- Check review-entry labels/IDs, event references, selected IDs versus keep decisions, finalization revision, and the pending-review gate. Verify target capabilities, structured prior knowledge, source blockers, and actual answer exposure. A final empty selection must not trigger fallback targets.
- Mark a diagnostic run complete only when every currently actionable selected target has its necessary branch coverage and every such branch is boundary_found or target_demonstrated; already resolved/retracted targets need no new branch; ignore out_of_scope branches. No review approval implies knowledge or resolves a historical gap. Blocked or unassessed paths must remain explicit, never described as completed. Derive summaries from records; do not store competing counts.
- Persist after each actual user response and pending-question change; publish only a validated checkpoint. If the bundled validator cannot run, require an equivalent implementation before publication; retain an explicitly unvalidated draft and report the blocker otherwise. If persistence fails, report it and retain/recover unsaved changes rather than claiming success.

## Teaching extension (same version 5.0 graph)

The teaching fields below extend the same base graph. Teaching may be null. Assessment.attempt_ids and based_on may reference teaching_attempt; source_issue/exposure.question_ids may reference teaching_question. Diagnostic history remains preserved. Only one stage may own an active pending prompt. Before teaching takes over, pause diagnostic, void its unanswered question while preserving it, and clear its pending pointer.

### Teaching run

`teaching` is an object with these required fields:

- `id`, `input_schema_version` ("1.0"|"2.0"|"3.0"|"4.0"|"5.0"), `started_at`, `updated_at`, `status` (active|paused|complete|blocked), `checkpoint_seq` (nonnegative integer incremented only on successful writes).
- `selected_review_revision` (integer or null while scope unresolved), `target_gap_ids` (exact finalized selection before instruction), `terminals` (records below).
- `active_terminal_gap_id` (nullable), `stack` (episode IDs, suspended parent first and active child last), `pending_question_id` (nullable teaching_question ID).
- `last_completed_terminal_gap_id` (nullable), `events` (append-only records below), `imports` (objects with locator, bundle_id, schema_version, checkpoint_seq nullable, relationship string), `limitations` (strings).

Each terminal record is `{gap_id, capability, concept_ids, status, assessment_ids, evidence_ids, episode_ids, source_issue_ids, rationale}`. status is unresolved|in_progress|understood|blocked|deferred|out_of_scope. Preserve records for removed targets as out_of_scope; active completion uses only target_gap_ids. capability snapshots the reviewed scope. An understood record requires sufficient current Known assessments and evidence covering that capability. Do not treat every loosely associated about-concept as necessary; explain the mapping in rationale.

Each teaching event is `{id, kind, concept_ids, gap_ids, episode_ids, question_ids, evidence_ids, description, timestamp}`. kind is migration|resume|selection|transition|instruction|prerequisite_revision|assessment_revision|invalidation|pause|completion|scope_change|deletion. Describe routing pivots, actual instruction, corrections, conflict resolution, and before/after state in their corresponding events. Reference arrays contain only surviving IDs of the indicated kinds; question_ids can include diagnostic or teaching questions. No raw deleted content in deletion events.

### New node kinds (all data fields required)

**teaching_episode**: terminal_gap_id; concept_id; parent_episode_id (nullable); step_id (nullable); prerequisite_set (concept IDs for the chosen route); entry_question_id (nullable); verification_question_ids; rationale_question_ids; response_attempt_ids; instruction_exposure_ids; phase (entry|explain|verify|rationale|prerequisites|coach|done|blocked|deferred); outcome (open|understood|deferred|blocked|superseded); assessment_id (nullable); return_reason (nullable); started_at (nullable); updated_at (nullable).

One episode represents a concept cycle and its retries. Parent/child episodes share a terminal; parent phase prerequisites while a child is active. Newly chosen prerequisites revise the selected set with a traceable event. understood requires phase done and a qualifying assessment, backed by a fresh verification or documented valid prior evidence. Completing a child does not complete its parent. Never create a second active episode for the same concept and context simply because two terminals share it.

**teaching_question**: episode_id; concept_id; purpose (entry|verification|rationale|coaching); text; expected_answer (nullable); rubric (nullable); assumptions (strings); grounding (verified|hypothetical|unverified|not_applicable); source_ids; responds_to_question_id (nullable); status (pending|answered|skipped|blocked|voided); supersedes_question_id (nullable); asked_at (nullable).

Entry/verification require verified or explicitly hypothetical grounding and a usable key/rubric before grading. Rationale is not graded and normally uses not_applicable with null key/rubric. Link rationale to the failed verification. Coaching is assistance, not independent verification. Freeze text/key after an attempt; corrections use superseding questions/events. Only one pending teaching question exists, including rationale/coaching; source requests are ungraded exchanges recorded as evidence/events, not invented quiz attempts.

**teaching_attempt**: question_id; response_evidence_id; result (correct|partially_correct|incorrect|dont_know|skipped|not_graded); evaluation; expected_answer_snapshot (nullable); rubric_snapshot (nullable); exposure_ids; assessed_at (nullable).

Require actual response evidence. Preserve individual attempts and corrections; rationale/coaching responses are not_graded. Entry success alone cannot finish a newly entered episode. For prior pending diagnostic response reuse, retain that diagnostic attempt and link it via an event/evidence; the episode may leave entry_question_id null with reuse recorded, rather than create a duplicate user response.

### Typed relationships

Preserve all imported edges. Emit these structural edges for new records, matching their authoritative fields:

- `from_session`: evidence → session.
- `assesses`: assessment → concept; `based_on`: assessment → attempt/teaching_attempt/evidence/source.
- `requires`: step → distinct prerequisite concepts; `yields`: step → target concept; `targets`: step → target gaps.
- `exposes`: exposure → concept; `blocked_by`: step/teaching_question/teaching_episode → open/conflicting source_issue (episodes derive relevant issues through their concept/step).
- `teaches`: teaching_episode → concept; `targets`: teaching_episode → its terminal gap; `returns_to`: child teaching_episode → parent teaching_episode.
- `probes`: teaching_question → concept; `answers`: teaching_attempt → teaching_question.

Structural edges can use observed with empty source_ids; they describe a reference, not truth of a proposed prerequisite. For requires/yields use inferred for proposed steps and grounded basis/source_ids for supported steps. Preserve AND/OR grouping in step fields; edges alone cannot express it. Never create redundant inverse edges. Instruction exposures and events retain the explanation; a question key alone is not evidence that it was taught.

### Validation before each checkpoint

Parse strict JSON: no duplicate keys, comments, NaN, or placeholder references. Validate the seven top-level fields, required record fields, enums, stable/global ID uniqueness, endpoint kinds, and every reference. Preserve imported records except documented state/pointer updates, supersession, or explicit deletion. Run the bundled schema and semantic validator before publication. An unvalidated draft is not a committed result.

Check selected review revision and scope, terminal coverage, chosen AND/OR prerequisites, acyclic assessment history and teaching stack, and parent return locations. At most one live teaching prompt must match pending_question_id and the top stack episode/phase; entry/verification/rationale/coaching correspond to entry/verify/rationale/coach phases. No active diagnostic pending prompt remains. An answered question has its real attempt; a source blocker is not an incorrect answer. Terminal prerequisites and hypotheses do not become Known by implication.

Check assessments refer to the correct concept, scored attempts have grounded keys, instruction exposures reflect content actually delivered, and understood episodes have sufficient post-explanation verification. Prior-evidence reuse must be explicit. Scope and source blockers cannot be reported as mastered concepts. Frozen keys/attempts retain their history when invalidated.

A complete teaching run has no pending question/active stack and every currently actionable selected terminal understood. Retracted targets are retired as out_of_scope with their retraction rationale, never counted as mastered. Previously resolved targets can be understood through explicitly mapped adequate prior evidence; do not reteach them merely to fill terminal records. Paused work can retain one pending question for resumption. Blocked/deferred/out_of_scope is never counted as understanding. If only deferred work remains, pause; if only source/scope-blocked work remains, block. Retain suspended episode locations with blocked/deferred phases and no pending prompt as appropriate. Derive progress summaries from records, not separate stored counters.

## Executable validation

The identical `references/learning-graph.schema.json` and `scripts/learning_validate.py` shipped with all three skills define the version 5.0 structural contract. Run `python scripts/learning_validate.py GRAPH --previous PRIOR_GRAPH` on a candidate before import or handoff. Omit --previous only for a genuinely new graph or a standalone structural check; compare against the actual input on migration, and against HEAD on updates. The schema uses standard JSON Schema 2020-12. The bundled standard-library validator implements the schema keywords used here plus graph/history checks, so no installation or network access is required. Do not silently drop fields or mutate history to satisfy it. Unsupported extensions require a compatibility decision.

The store calls this validator before publishing every non-null graph, using HEAD as previous; absence or validation failure refuses the commit. Initial extraction may checkpoint graph=null with unpublished draft records until a valid graph exists; once published, the graph cannot become null. Correct a failed candidate in working files, then retry at the current revision. Reuse validation errors as repair instructions, not as evidence about the learner.

Checks include field types/enums, all ID references, structural edges, global uniqueness, assessment/resolution chains, immutable evidence/attempts, current resolution support, review selection, boundary prerequisites, prompt ownership, and teaching stack/verification consistency. Component arrays are required for new assessments; older imported records remain unchanged. On migration, --previous distinguishes old records from newly created ones. Validate first against that original input before the first imported checkpoint.

Programmatic validation does not prove a quote authentic, a rubric correct, a capability fully covered, an explanation delivered, or an answer independent. Review those semantic claims against actual sources and exchanges. Success is structural/provenance consistency, not an automated assertion of learning.

For an explicitly requested data deletion, a narrowly scoped --deletions JSON manifest may authorize the affected removed/redacted record IDs and reference-cleanup records. Its exact fields are record_ids (old node/edge/event IDs), evidence_ids (retained user evidence of the deletion instruction), and reason (nonempty string). Pass the same manifest to learning_store.py commit. The manifest is not permission by itself: verify the actual instruction and scope first. It exempts only those records from history preservation and never exempts the resulting graph from schema/reference checks. Ordinary exclusion must not use this option. Committing the current graph does not purge old checkpoints, source copies, working files, or artifacts. Complete the workspace's explicit-deletion cleanup under an exclusive writer lock, updating surviving checksums and mirror, before claiming deletion; if complete cleanup cannot be performed, report what remains instead of claiming success. Do not store the purged content in the deletion reason.
<!-- END EMBEDDED FILE -->

## Embedded runtime resources

The following resources are complete. The JSON schema is compacted for portability; its parsed content matches the installed schema.

<!-- BEGIN EMBEDDED FILE references/learning-graph.schema.json -->
````json
{"$schema":"https://json-schema.org/draft/2020-12/schema","$id":"urn:learning-graph:schema:5.0","title":"Learning graph 5.0","type":"object","properties":{"schema_version":{"const":"5.0"},"bundle":{"$ref":"#/$defs/bundle"},"nodes":{"type":"array","items":{"$ref":"#/$defs/node"}},"edges":{"type":"array","items":{"$ref":"#/$defs/edge"}},"review":{"$ref":"#/$defs/review"},"diagnostic":{"anyOf":[{"$ref":"#/$defs/diagnostic"},{"type":"null"}]},"teaching":{"anyOf":[{"$ref":"#/$defs/teaching"},{"type":"null"}]}},"required":["schema_version","bundle","nodes","edges","review","diagnostic","teaching"],"additionalProperties":false,"$defs":{"session":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"session"},"label":{"type":"string"},"data":{"type":"object","properties":{"session_id":{"anyOf":[{"type":"string"},{"type":"null"}]},"coverage":{"type":"string","enum":["full","partial","summary_only","unavailable"]},"source_description":{"type":"string"},"locator_scheme":{"type":"string"},"started_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"missing_material":{"type":"array","items":{"type":"string"}}},"required":["session_id","coverage","source_description","locator_scheme","started_at","missing_material"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"concept":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"concept"},"label":{"type":"string"},"data":{"type":"object","properties":{"description":{"type":"string","minLength":1},"scope":{"type":"string","enum":["general","organization","both"]},"namespace":{"anyOf":[{"type":"string"},{"type":"null"}]},"categories":{"type":"array","items":{"type":"string"}},"current_assessment_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]}},"required":["description","scope","namespace","categories","current_assessment_id"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"gap":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"gap"},"label":{"type":"string"},"data":{"type":"object","properties":{"description":{"type":"string","minLength":1},"demonstrated_knowledge":{"type":"string"},"context":{"type":"string"},"scope":{"type":"string","enum":["general","organization","both"]},"categories":{"type":"array","items":{"type":"string"}},"classification":{"type":"string","enum":["ongoing","resolved","tentative"]},"confidence":{"type":"string","enum":["high","medium","low"]},"confidence_rationale":{"type":"string","minLength":1},"status":{"type":"string","enum":["open","partially_resolved","addressed_understanding_unverified","resolved_within_bundle","unclear"]},"status_rationale":{"type":"string","minLength":1},"assessment":{"type":"string"},"alternatives":{"type":"array","items":{"type":"string"}},"bundle_progression":{"type":"string"},"grounding":{"type":"object","properties":{"verification":{"type":"string","enum":["verified","reported","inferred","unverified","not_needed"]},"description":{"type":"string"},"source_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true}},"required":["verification","description","source_ids"],"additionalProperties":false},"impact":{"type":"string"},"remaining_clarification":{"anyOf":[{"type":"string"},{"type":"null"}]},"priority_rationale":{"anyOf":[{"type":"string"},{"type":"null"}]},"current_resolution_id":{"type":"string","minLength":1}},"required":["description","demonstrated_knowledge","context","scope","categories","classification","confidence","confidence_rationale","status","status_rationale","assessment","alternatives","bundle_progression","grounding","impact","remaining_clarification","priority_rationale","current_resolution_id"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"evidence":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"evidence"},"label":{"type":"string"},"data":{"type":"object","properties":{"speaker":{"type":"string","enum":["user","assistant","tool","other"]},"representation":{"type":"string","enum":["quote","paraphrase","summary"]},"text":{"type":"string","minLength":1},"locator":{"type":"string","minLength":1},"locator_kind":{"type":"string","enum":["native","export","assigned"]},"timestamp":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"context":{"type":"string"}},"required":["speaker","representation","text","locator","locator_kind","timestamp","context"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"entity":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"entity"},"label":{"type":"string"},"data":{"type":"object","properties":{"entity_kind":{"type":"string","enum":["repository","project","service","database","schema","team","environment","other"]},"namespace":{"anyOf":[{"type":"string"},{"type":"null"}]},"description":{"type":"string"}},"required":["entity_kind","namespace","description"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"source":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"source"},"label":{"type":"string"},"data":{"type":"object","properties":{"source_kind":{"type":"string","enum":["code","test","schema","configuration","documentation","other"]},"locator":{"type":"string","minLength":1},"version":{"anyOf":[{"type":"string"},{"type":"null"}]},"description":{"type":"string"}},"required":["source_kind","locator","version","description"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"gap_resolution":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"gap_resolution"},"label":{"type":"string"},"data":{"type":"object","properties":{"gap_id":{"type":"string","minLength":1},"capability":{"type":"string","minLength":1},"status":{"type":"string","enum":["open","partial","addressed_unverified","resolved","unclear","retracted"]},"confidence":{"type":"string","enum":["high","medium","low"]},"confidence_rationale":{"type":"string","minLength":1},"rationale":{"type":"string","minLength":1},"assessment_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"evidence_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"previous_resolution_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"recorded_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["gap_id","capability","status","confidence","confidence_rationale","rationale","assessment_ids","evidence_ids","previous_resolution_id","recorded_at"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"question":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"question"},"label":{"type":"string"},"data":{"type":"object","properties":{"text":{"type":"string","minLength":1},"concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true,"minItems":1},"branch_id":{"type":"string","minLength":1},"expected_answer":{"anyOf":[{"type":"string"},{"type":"null"}]},"rubric":{"anyOf":[{"type":"string"},{"type":"null"}]},"assumptions":{"type":"array","items":{"type":"string"}},"grounding":{"type":"string","enum":["verified","hypothetical","unverified"]},"source_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"status":{"type":"string","enum":["pending","answered","skipped","blocked","voided"]},"supersedes_question_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]}},"required":["text","concept_ids","branch_id","expected_answer","rubric","assumptions","grounding","source_ids","status","supersedes_question_id"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"attempt":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"attempt"},"label":{"type":"string"},"data":{"type":"object","properties":{"question_id":{"type":"string","minLength":1},"response_evidence_id":{"type":"string","minLength":1},"result":{"type":"string","enum":["correct","partially_correct","incorrect","dont_know","skipped","not_graded"]},"evaluation":{"type":"string","minLength":1},"expected_answer_snapshot":{"anyOf":[{"type":"string"},{"type":"null"}]},"rubric_snapshot":{"anyOf":[{"type":"string"},{"type":"null"}]},"exposure_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"assessed_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["question_id","response_evidence_id","result","evaluation","expected_answer_snapshot","rubric_snapshot","exposure_ids","assessed_at"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"assessment":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"assessment"},"label":{"type":"string"},"data":{"type":"object","properties":{"concept_id":{"type":"string","minLength":1},"state":{"type":"string","enum":["Known","Partial","Unknown","Unassessed"]},"confidence":{"type":"string","enum":["high","medium","low"]},"confidence_rationale":{"type":"string","minLength":1},"rationale":{"type":"string","minLength":1},"context":{"type":"string"},"basis":{"type":"string","enum":["prior_evidence","independent_probe","assisted_probe","user_report","unassessed"]},"evidence_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"attempt_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"exposure_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"previous_assessment_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"assessed_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"demonstrated_components":{"type":"array","items":{"type":"string"}},"missing_components":{"type":"array","items":{"type":"string"}}},"required":["concept_id","state","confidence","confidence_rationale","rationale","context","basis","evidence_ids","attempt_ids","exposure_ids","previous_assessment_id","assessed_at"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"step":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"step"},"label":{"type":"string"},"data":{"type":"object","properties":{"target_concept_id":{"type":"string","minLength":1},"target_gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"prerequisite_sets":{"type":"array","items":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true,"minItems":1},"uniqueItems":true},"kind":{"type":"string","enum":["inference","new_fact"]},"reachability_rationale":{"type":"string","minLength":1},"status":{"type":"string","enum":["proposed","supported","blocked","superseded"]},"source_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"source_issue_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"supersedes_step_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]}},"required":["target_concept_id","target_gap_ids","prerequisite_sets","kind","reachability_rationale","status","source_ids","source_issue_ids","supersedes_step_id"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"branch":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"branch"},"label":{"type":"string"},"data":{"type":"object","properties":{"target_gap_id":{"type":"string","minLength":1},"target_concept_id":{"type":"string","minLength":1},"step_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"status":{"type":"string","enum":["seeking","boundary_found","target_demonstrated","blocked","paused","out_of_scope"]},"boundary_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"rationale":{"type":"string","minLength":1},"source_issue_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true}},"required":["target_gap_id","target_concept_id","step_ids","status","boundary_id","rationale","source_issue_ids"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"boundary":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"boundary"},"label":{"type":"string"},"data":{"type":"object","properties":{"branch_id":{"type":"string","minLength":1},"target_gap_id":{"type":"string","minLength":1},"known_concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"frontier_concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"step_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"assessment_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"status":{"type":"string","enum":["established","provisional","superseded"]},"rationale":{"type":"string","minLength":1}},"required":["branch_id","target_gap_id","known_concept_ids","frontier_concept_ids","step_ids","assessment_ids","status","rationale"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"source_issue":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"source_issue"},"label":{"type":"string"},"data":{"type":"object","properties":{"disputed_fact":{"type":"string","minLength":1},"question_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"status":{"type":"string","enum":["open","resolved","conflicting"]},"attempts":{"type":"array","items":{"type":"object","properties":{"locator":{"type":"string"},"outcome":{"type":"string"}},"required":["locator","outcome"],"additionalProperties":false}},"needed_input":{"anyOf":[{"type":"string"},{"type":"null"}]},"resolution":{"anyOf":[{"type":"string"},{"type":"null"}]},"source_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true}},"required":["disputed_fact","question_ids","concept_ids","status","attempts","needed_input","resolution","source_ids"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"exposure":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"exposure"},"label":{"type":"string"},"data":{"type":"object","properties":{"kind":{"type":"string","enum":["answer_disclosure","hint","instruction","source_discovery","answer_key_viewed"]},"concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"question_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"evidence_id":{"type":"string","minLength":1},"occurred_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"description":{"type":"string","minLength":1}},"required":["kind","concept_ids","question_ids","evidence_id","occurred_at","description"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"teaching_episode":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"teaching_episode"},"label":{"type":"string"},"data":{"type":"object","properties":{"terminal_gap_id":{"type":"string","minLength":1},"concept_id":{"type":"string","minLength":1},"parent_episode_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"step_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"prerequisite_set":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"entry_question_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"verification_question_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"rationale_question_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"response_attempt_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"instruction_exposure_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"phase":{"type":"string","enum":["entry","explain","verify","rationale","prerequisites","coach","done","blocked","deferred"]},"outcome":{"type":"string","enum":["open","understood","deferred","blocked","superseded"]},"assessment_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"return_reason":{"anyOf":[{"type":"string"},{"type":"null"}]},"started_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"updated_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["terminal_gap_id","concept_id","parent_episode_id","step_id","prerequisite_set","entry_question_id","verification_question_ids","rationale_question_ids","response_attempt_ids","instruction_exposure_ids","phase","outcome","assessment_id","return_reason","started_at","updated_at"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"teaching_question":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"teaching_question"},"label":{"type":"string"},"data":{"type":"object","properties":{"episode_id":{"type":"string","minLength":1},"concept_id":{"type":"string","minLength":1},"purpose":{"type":"string","enum":["entry","verification","rationale","coaching"]},"text":{"type":"string","minLength":1},"expected_answer":{"anyOf":[{"type":"string"},{"type":"null"}]},"rubric":{"anyOf":[{"type":"string"},{"type":"null"}]},"assumptions":{"type":"array","items":{"type":"string"}},"grounding":{"type":"string","enum":["verified","hypothetical","unverified","not_applicable"]},"source_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"responds_to_question_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"status":{"type":"string","enum":["pending","answered","skipped","blocked","voided"]},"supersedes_question_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"asked_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["episode_id","concept_id","purpose","text","expected_answer","rubric","assumptions","grounding","source_ids","responds_to_question_id","status","supersedes_question_id","asked_at"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"teaching_attempt":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"const":"teaching_attempt"},"label":{"type":"string"},"data":{"type":"object","properties":{"question_id":{"type":"string","minLength":1},"response_evidence_id":{"type":"string","minLength":1},"result":{"type":"string","enum":["correct","partially_correct","incorrect","dont_know","skipped","not_graded"]},"evaluation":{"type":"string","minLength":1},"expected_answer_snapshot":{"anyOf":[{"type":"string"},{"type":"null"}]},"rubric_snapshot":{"anyOf":[{"type":"string"},{"type":"null"}]},"exposure_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"assessed_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["question_id","response_evidence_id","result","evaluation","expected_answer_snapshot","rubric_snapshot","exposure_ids","assessed_at"],"additionalProperties":false}},"required":["id","kind","label","data"],"additionalProperties":false},"node":{"anyOf":[{"$ref":"#/$defs/session"},{"$ref":"#/$defs/concept"},{"$ref":"#/$defs/gap"},{"$ref":"#/$defs/evidence"},{"$ref":"#/$defs/entity"},{"$ref":"#/$defs/source"},{"$ref":"#/$defs/gap_resolution"},{"$ref":"#/$defs/question"},{"$ref":"#/$defs/attempt"},{"$ref":"#/$defs/assessment"},{"$ref":"#/$defs/step"},{"$ref":"#/$defs/branch"},{"$ref":"#/$defs/boundary"},{"$ref":"#/$defs/source_issue"},{"$ref":"#/$defs/exposure"},{"$ref":"#/$defs/teaching_episode"},{"$ref":"#/$defs/teaching_question"},{"$ref":"#/$defs/teaching_attempt"}]},"edge":{"type":"object","properties":{"id":{"type":"string","minLength":1},"source":{"type":"string","minLength":1},"target":{"type":"string","minLength":1},"type":{"type":"string","enum":["from_session","supports","challenges","addresses","resolves","about","involves","applies_to","instance_of","prerequisite_for","related_to","depends_on","shares_semantics_with","assumes","tracks_resolution","based_on","supersedes","probes","answers","assesses","requires","yields","targets","locates","blocked_by","exposes","teaches","returns_to"]},"basis":{"type":"string","enum":["observed","reported","inferred","unverified"]},"description":{"type":"string"},"source_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true}},"required":["id","source","target","type","basis","description","source_ids"],"additionalProperties":false},"review_entry":{"type":"object","properties":{"gap_id":{"type":"string","minLength":1},"label":{"type":"string","minLength":1},"decision":{"type":"string","enum":["pending","keep","exclude","defer","superseded"]},"reason":{"anyOf":[{"type":"string"},{"type":"null"}]},"target_capability":{"type":"string","minLength":1},"scope_note":{"anyOf":[{"type":"string"},{"type":"null"}]},"successor_gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true}},"required":["gap_id","label","decision","reason","target_capability","scope_note","successor_gap_ids"],"additionalProperties":false},"review_event":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"type":"string","enum":["presented","clarification","selection","revision","finalization","reopened","deletion"]},"revision":{"type":"integer","minimum":1},"entry_labels":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"evidence_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"description":{"type":"string","minLength":1},"timestamp":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["id","kind","revision","entry_labels","gap_ids","evidence_ids","description","timestamp"],"additionalProperties":false},"review":{"type":"object","properties":{"status":{"type":"string","enum":["pending","finalized"]},"revision":{"type":"integer","minimum":1},"finalized_revision":{"anyOf":[{"type":"integer","minimum":1},{"type":"null"}]},"updated_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"selected_gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"entries":{"type":"array","items":{"$ref":"#/$defs/review_entry"}},"events":{"type":"array","items":{"$ref":"#/$defs/review_event"}}},"required":["status","revision","finalized_revision","updated_at","selected_gap_ids","entries","events"],"additionalProperties":false},"diagnostic":{"type":"object","properties":{"id":{"type":"string","minLength":1},"input_schema_version":{"type":"string","enum":["1.0","2.0","3.0","4.0","5.0"]},"started_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"updated_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"status":{"type":"string","enum":["active","paused","complete","blocked"]},"target_gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"branch_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"active_branch_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"pending_question_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"checkpoint_seq":{"type":"integer","minimum":0},"selected_review_revision":{"anyOf":[{"type":"integer","minimum":1},{"type":"null"}]},"limitations":{"type":"array","items":{"type":"string"}}},"required":["id","input_schema_version","started_at","updated_at","status","target_gap_ids","branch_ids","active_branch_id","pending_question_id","checkpoint_seq","selected_review_revision","limitations"],"additionalProperties":false},"terminal":{"type":"object","properties":{"gap_id":{"type":"string","minLength":1},"capability":{"type":"string","minLength":1},"concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"status":{"type":"string","enum":["unresolved","in_progress","understood","blocked","deferred","out_of_scope"]},"assessment_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"evidence_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"episode_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"source_issue_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"rationale":{"type":"string","minLength":1}},"required":["gap_id","capability","concept_ids","status","assessment_ids","evidence_ids","episode_ids","source_issue_ids","rationale"],"additionalProperties":false},"teaching_event":{"type":"object","properties":{"id":{"type":"string","minLength":1},"kind":{"type":"string","enum":["migration","resume","selection","transition","instruction","prerequisite_revision","assessment_revision","invalidation","pause","completion","scope_change","deletion"]},"concept_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"episode_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"question_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"evidence_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"description":{"type":"string","minLength":1},"timestamp":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]}},"required":["id","kind","concept_ids","gap_ids","episode_ids","question_ids","evidence_ids","description","timestamp"],"additionalProperties":false},"teaching":{"type":"object","properties":{"id":{"type":"string","minLength":1},"input_schema_version":{"type":"string","enum":["1.0","2.0","3.0","4.0","5.0"]},"started_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"updated_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"status":{"type":"string","enum":["active","paused","complete","blocked"]},"checkpoint_seq":{"type":"integer","minimum":0},"selected_review_revision":{"anyOf":[{"type":"integer","minimum":1},{"type":"null"}]},"target_gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"terminals":{"type":"array","items":{"$ref":"#/$defs/terminal"}},"active_terminal_gap_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"stack":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"pending_question_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"last_completed_terminal_gap_id":{"anyOf":[{"type":"string","minLength":1},{"type":"null"}]},"events":{"type":"array","items":{"$ref":"#/$defs/teaching_event"}},"imports":{"type":"array","items":{"type":"object","properties":{"locator":{"type":"string","minLength":1},"bundle_id":{"type":"string","minLength":1},"schema_version":{"type":"string"},"checkpoint_seq":{"anyOf":[{"type":"integer","minimum":0},{"type":"null"}]},"relationship":{"type":"string","minLength":1}},"required":["locator","bundle_id","schema_version","checkpoint_seq","relationship"],"additionalProperties":false}},"limitations":{"type":"array","items":{"type":"string"}}},"required":["id","input_schema_version","started_at","updated_at","status","checkpoint_seq","selected_review_revision","target_gap_ids","terminals","active_terminal_gap_id","stack","pending_question_id","last_completed_terminal_gap_id","events","imports","limitations"],"additionalProperties":false},"bundle":{"type":"object","properties":{"id":{"type":"string","minLength":1},"purpose":{"type":"string"},"generated_at":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"timezone":{"anyOf":[{"type":"string"},{"type":"null"}]},"session_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"chronology":{"type":"string"},"limitations":{"type":"array","items":{"type":"string"}},"priority_gap_ids":{"type":"array","items":{"type":"string","minLength":1},"uniqueItems":true},"schema_history":{"type":"array","items":{"type":"object","properties":{"from_version":{"type":"string","enum":["1.0","2.0","3.0","4.0"]},"to_version":{"const":"5.0"},"timestamp":{"anyOf":[{"type":"string","format":"date-time"},{"type":"null"}]},"reason":{"type":"string","minLength":1}},"required":["from_version","to_version","timestamp","reason"],"additionalProperties":false}}},"required":["id","purpose","generated_at","timezone","session_ids","chronology","limitations","priority_gap_ids","schema_history"],"additionalProperties":false}}}
````
<!-- END EMBEDDED FILE -->

<!-- BEGIN EMBEDDED FILE scripts/learning_store.py -->
````python
#!/usr/bin/env python3
"""Local, revision-checked, crash-recoverable learning bundle storage. Python 3 stdlib."""
import argparse, hashlib, importlib.util, json, os, re, sys, uuid
from datetime import datetime, timezone
from pathlib import Path

SKILLS = {'mind-the-gap', 'find-the-gap', 'bridge-the-gap'}

def read(path):
    def pairs(items):
        value = {}
        for key, item in items:
            if key in value: raise ValueError('Duplicate JSON key: ' + key)
            value[key] = item
        return value
    return json.loads(Path(path).read_text(encoding='utf-8'), object_pairs_hook=pairs,
                      parse_constant=lambda value: (_ for _ in ()).throw(ValueError(value)))

def flush_dir(path):
    if os.name == 'posix':
        fd = os.open(str(path), os.O_RDONLY)
        try: os.fsync(fd)
        finally: os.close(fd)

def write(path, value):
    raw = (json.dumps(value, ensure_ascii=False, allow_nan=False, indent=2) + '\n').encode('utf-8')
    fd = os.open(str(path), os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
    with os.fdopen(fd, 'wb') as stream:
        stream.write(raw); stream.flush(); os.fsync(stream.fileno())
    return hashlib.sha256(raw).hexdigest()

def atomic(path, value):
    temp = path.parent / ('.' + path.name + '.' + uuid.uuid4().hex)
    try:
        write(temp, value); os.replace(temp, path); flush_dir(path.parent)
    finally:
        if temp.exists(): temp.unlink()

def state_check(state):
    required = {'state_version','active_skill','phase','extraction','next_action','pending_delivery','limitations'}
    if set(state) != required or state['state_version'] != '1.0': raise ValueError('Invalid workspace state fields/version')
    if state['active_skill'] not in SKILLS: raise ValueError('Invalid active_skill')
    if state['phase'] not in {'ingesting','reconciling','reviewing','diagnosing','teaching','paused','blocked','complete'}: raise ValueError('Invalid phase')
    x = state['extraction']
    if set(x) != {'cutoff','sources','ledger','draft_records','reconciliation_queue'}: raise ValueError('Invalid extraction state')
    for name in ['sources','ledger','reconciliation_queue']:
        if not isinstance(x[name], list): raise ValueError('Expected extraction array: ' + name)
    if not isinstance(state['next_action'], str) or not isinstance(state['limitations'], list): raise ValueError('Invalid state text/limitations')
    source_ids = set()
    for source in x['sources']:
        if set(source) != {'id','locator','fingerprint','coverage','read_ranges','cutoff','next_locator','status'}: raise ValueError('Invalid source cursor')
        if source['id'] in source_ids: raise ValueError('Duplicate source cursor')
        source_ids.add(source['id'])
        if source['status'] not in {'pending','reading','read','unavailable'}: raise ValueError('Invalid source read status')
        if source['coverage'] not in {'full','partial','summary_only','unavailable'}: raise ValueError('Invalid source coverage')
        if not isinstance(source['read_ranges'], list): raise ValueError('read_ranges must be an array')
    for item in x['ledger']:
        if set(item) != {'id','source_id','locator','task','candidate_ids','supporting_evidence_ids','contradicting_evidence_ids','resolution_evidence_ids','notes'}: raise ValueError('Invalid ledger entry')
        if item['source_id'] not in source_ids: raise ValueError('Ledger source missing')
    if state['pending_delivery'] is not None:
        delivery = state['pending_delivery']
        if set(delivery) != {'id','text','status'} or delivery['status'] not in {'prepared','delivered','uncertain'}: raise ValueError('Invalid pending delivery')

def manifest(bundle):
    bundle = Path(bundle).expanduser().resolve()
    data = read(bundle / 'bundle.json')
    if data['store_version'] != '1.0': raise ValueError('Unsupported store version')
    return bundle, data

def load(bundle, repair=True):
    bundle, meta = manifest(bundle)
    head = read(bundle / 'HEAD.json')
    base = (bundle / head['checkpoint']).resolve()
    if base.parent != (bundle / 'checkpoints').resolve(): raise ValueError('Unsafe checkpoint path')
    values = {}
    for name in ['graph','state']:
        path = base / (name + '.json')
        if hashlib.sha256(path.read_bytes()).hexdigest() != head[name + '_sha256']: raise ValueError('Checkpoint checksum mismatch: ' + name)
        values[name] = read(path)
    state_check(values['state'])
    if values['graph'] is not None and values['graph']['bundle']['id'] != meta['bundle_id']: raise ValueError('Bundle identity mismatch')
    # Serialize mirror repair with writers and recheck HEAD under the lock.
    if repair:
        repair_lock = bundle / '.writer-lock'
        owns_lock = False
        try:
            try: repair_lock.mkdir(mode=0o700); owns_lock = True
            except FileExistsError: pass
            if owns_lock:
                write(repair_lock / 'owner.json', {'pid':os.getpid(),'operation':'mirror-repair'})
                if read(bundle / 'HEAD.json') != head:
                    return load(bundle, repair=False)
                mirror = bundle / 'learning-graph.json'
                if values['graph'] is not None:
                    try: same = read(mirror) == values['graph']
                    except (OSError, ValueError): same = False
                    if not same: atomic(mirror, values['graph'])
        finally:
            if owns_lock:
                owner = repair_lock / 'owner.json'
                if owner.exists(): owner.unlink()
                repair_lock.rmdir()
    return {'bundle': str(bundle), 'bundle_id': meta['bundle_id'], 'revision': head['revision'],
            'graph_path': str(base / 'graph.json'), 'state_path': str(base / 'state.json'), **values}

def publish(bundle, revision, graph, state):
    token = f'{revision:08d}-' + uuid.uuid4().hex[:12]
    stage = bundle / 'checkpoints' / ('.pending-' + token)
    stage.mkdir(mode=0o700)
    gh = write(stage / 'graph.json', graph); sh = write(stage / 'state.json', state)
    flush_dir(stage)
    finished = stage.parent / token
    os.replace(stage, finished); flush_dir(finished.parent)
    atomic(bundle / 'HEAD.json', {'revision': revision, 'checkpoint': str(finished.relative_to(bundle)),
                                'graph_sha256': gh, 'state_sha256': sh})
    warning = None
    try:
        if graph is not None: atomic(bundle / 'learning-graph.json', graph)
    except OSError as error: warning = 'Committed successfully; mirror repair required: ' + str(error)
    return {'bundle': str(bundle), 'revision': revision, 'committed': True, 'warning': warning}

def init(root, name, bundle_id=None):
    if not root: raise ValueError('Set LEARNING_WORKSPACE or supply --root with a persistent absolute path')
    root = Path(root).expanduser()
    if not root.is_absolute(): raise ValueError('Workspace root must be absolute')
    root = root.resolve(); root.mkdir(parents=True, exist_ok=True, mode=0o700)
    slug = re.sub(r'[^a-z0-9]+', '-', name.lower()).strip('-')[:50] or 'learning'
    folder = root / (slug + '--' + datetime.now(timezone.utc).strftime('%Y%m%dT%H%M%SZ') + '--' + uuid.uuid4().hex[:10])
    folder.mkdir(mode=0o700)
    for child in ['checkpoints','sources','artifacts','working']: (folder / child).mkdir(mode=0o700)
    write(folder / 'bundle.json', {'store_version':'1.0','bundle_id':bundle_id or 'bundle:' + uuid.uuid4().hex,
                                'root':str(root),'created_at':datetime.now(timezone.utc).isoformat(),'name':name})
    state = {'state_version':'1.0','active_skill':'mind-the-gap','phase':'ingesting',
             'extraction':{'cutoff':None,'sources':[],'ledger':[],'draft_records':{'nodes':[],'edges':[]},'reconciliation_queue':[]},
             'next_action':'Persist the requested source inventory before reading.','pending_delivery':None,'limitations':[]}
    publish(folder, 0, None, state)
    return load(folder)

def commit(bundle, expected_revision, graph, state, deletions=None):
    bundle, meta = manifest(bundle); lock = bundle / '.writer-lock'
    try: lock.mkdir(mode=0o700)
    except FileExistsError: raise ValueError('Writer lock exists; inspect ownership before recovering it')
    try:
        write(lock / 'owner.json', {'pid':os.getpid(),'created_at':datetime.now(timezone.utc).isoformat()})
        old = load(bundle, repair=False)
        if old['revision'] != expected_revision: raise ValueError('Stale expected revision; reload and reconcile')
        state_check(state)
        if old['graph'] is not None and graph is None: raise ValueError('A published graph cannot be discarded by a null checkpoint')
        if graph is not None:
            if graph['bundle']['id'] != meta['bundle_id']: raise ValueError('Graph has a different bundle ID')
            validator_path = Path(__file__).with_name('learning_validate.py')
            if not validator_path.is_file(): raise ValueError('Missing bundled learning_validate.py; graph commit refused')
            spec = importlib.util.spec_from_file_location('learning_validate', validator_path)
            validator = importlib.util.module_from_spec(spec); spec.loader.exec_module(validator)
            validator.validate(graph, previous=old['graph'], deletions=deletions)
        return publish(bundle, expected_revision + 1, graph, state)
    finally:
        owner = lock / 'owner.json'
        if owner.exists(): owner.unlink()
        lock.rmdir()

def main():
    parser = argparse.ArgumentParser(description=__doc__); sub = parser.add_subparsers(dest='command', required=True)
    create = sub.add_parser('init'); create.add_argument('--root',default=os.environ.get('LEARNING_WORKSPACE')); create.add_argument('--name',required=True); create.add_argument('--bundle-id')
    for command in ['load','commit']:
        child=sub.add_parser(command);child.add_argument('--bundle',required=True)
        if command=='load':child.add_argument('--summary',action='store_true')
        else:
            child.add_argument('--expected-revision',type=int,required=True);child.add_argument('--graph',required=True);child.add_argument('--state',required=True);child.add_argument('--deletions')
    args=parser.parse_args()
    if args.command=='init': result=init(args.root,args.name,args.bundle_id)
    elif args.command=='load':
        result=load(args.bundle)
        if args.summary: result={k:v for k,v in result.items() if k not in ['graph','state']}
    else: result=commit(args.bundle,args.expected_revision,read(args.graph),read(args.state),read(args.deletions) if args.deletions else None)
    print(json.dumps(result,ensure_ascii=False,indent=2))
if __name__=='__main__':
    try: main()
    except (ValueError, KeyError, OSError) as error: print(str(error),file=sys.stderr);sys.exit(1)
````
<!-- END EMBEDDED FILE -->

<!-- BEGIN EMBEDDED FILE scripts/learning_validate.py -->
````python
#!/usr/bin/env python3
"""Validate learning-graph 5.0 structure, references, runtime state and append-only history.
Python 3 standard library only. Semantic evidence quality still requires human/agent review.
"""
import argparse
import json
import sys
from datetime import datetime
from pathlib import Path

class ValidationError(ValueError):
    pass

def require(condition, message):
    if not condition:
        raise ValidationError(message)

def read(path):
    def pairs(items):
        result = {}
        for key, value in items:
            require(key not in result, 'Duplicate JSON key: ' + key)
            result[key] = value
        return result
    def constant(value):
        raise ValidationError('Nonfinite JSON value: ' + value)
    return json.loads(Path(path).read_text(encoding='utf-8'), object_pairs_hook=pairs, parse_constant=constant)

def shape(value, schema, root, path='$'):
    if '$ref' in schema:
        return shape(value, root['$defs'][schema['$ref'].split('/')[-1]], root, path)
    if 'anyOf' in schema:
        options = schema['anyOf']
        # Select the discriminated node arm for a useful error instead of 18 irrelevant ones.
        if isinstance(value, dict) and 'kind' in value and all('$ref' in x for x in options):
            matching = [x for x in options if root['$defs'][x['$ref'].split('/')[-1]].get('properties', {}).get('kind', {}).get('const') == value['kind']]
            if matching:
                return shape(value, matching[0], root, path)
        errors = []
        for option in options:
            try:
                shape(value, option, root, path)
                return
            except ValidationError as error:
                errors.append(str(error))
        raise ValidationError(path + ': no allowed shape (' + '; '.join(errors[:2]) + ')')
    if 'const' in schema:
        require(value == schema['const'], path + ': expected ' + repr(schema['const']))
    types = {'object': dict, 'array': list, 'string': str, 'integer': int, 'null': type(None)}
    if 'type' in schema:
        expected = schema['type']
        require(type(value) is types[expected], path + ': expected ' + expected)
    if 'enum' in schema:
        require(value in schema['enum'], path + ': invalid enum ' + repr(value))
    if isinstance(value, dict):
        require(set(schema.get('required', [])) <= set(value), path + ': missing fields ' + repr(sorted(set(schema.get('required', [])) - set(value))))
        if schema.get('additionalProperties') is False:
            require(set(value) <= set(schema.get('properties', {})), path + ': unsupported fields ' + repr(sorted(set(value) - set(schema.get('properties', {})))))
        for key, child in schema.get('properties', {}).items():
            if key in value:
                shape(value[key], child, root, path + '.' + key)
    if isinstance(value, list):
        require(len(value) >= schema.get('minItems', 0), path + ': too few items')
        if schema.get('uniqueItems'):
            encoded = [json.dumps(x, sort_keys=True, ensure_ascii=False) for x in value]
            require(len(encoded) == len(set(encoded)), path + ': duplicate items')
        if 'items' in schema:
            for i, child in enumerate(value):
                shape(child, schema['items'], root, path + '[' + str(i) + ']')
    if isinstance(value, str):
        require(len(value) >= schema.get('minLength', 0) and (not schema.get('minLength') or value.strip()), path + ': empty/blank string')
        if schema.get('format') == 'date-time':
            try:
                parsed = datetime.fromisoformat(value.replace('Z', '+00:00'))
                require('T' in value and parsed.utcoffset() is not None, path + ': timestamp needs date, time and offset')
            except ValueError as error:
                raise ValidationError(path + ': invalid timestamp') from error
    if type(value) is int and 'minimum' in schema:
        require(value >= schema['minimum'], path + ': below minimum')

# Fields refer to node kinds. Array/nullable shape comes from JSON Schema.
REFS = {
 'concept': {'current_assessment_id':'assessment'},
 'gap': {'current_resolution_id':'gap_resolution'},
 'gap_resolution': {'gap_id':'gap','assessment_ids':'assessment','evidence_ids':'evidence|source','previous_resolution_id':'gap_resolution'},
 'question': {'concept_ids':'concept','branch_id':'branch','source_ids':'evidence|source','supersedes_question_id':'question'},
 'attempt': {'question_id':'question','response_evidence_id':'evidence','exposure_ids':'exposure'},
 'assessment': {'concept_id':'concept','evidence_ids':'evidence|source','attempt_ids':'attempt|teaching_attempt','exposure_ids':'exposure','previous_assessment_id':'assessment'},
 'step': {'target_concept_id':'concept','target_gap_ids':'gap','prerequisite_sets':'concept','source_ids':'evidence|source','source_issue_ids':'source_issue','supersedes_step_id':'step'},
 'branch': {'target_gap_id':'gap','target_concept_id':'concept','step_ids':'step','boundary_id':'boundary','source_issue_ids':'source_issue'},
 'boundary': {'branch_id':'branch','target_gap_id':'gap','known_concept_ids':'concept','frontier_concept_ids':'concept','step_ids':'step','assessment_ids':'assessment'},
 'source_issue': {'question_ids':'question|teaching_question','concept_ids':'concept','source_ids':'evidence|source'},
 'exposure': {'concept_ids':'concept','question_ids':'question|teaching_question','evidence_id':'evidence'},
 'teaching_episode': {'terminal_gap_id':'gap','concept_id':'concept','parent_episode_id':'teaching_episode','step_id':'step','prerequisite_set':'concept','entry_question_id':'teaching_question','verification_question_ids':'teaching_question','rationale_question_ids':'teaching_question','response_attempt_ids':'teaching_attempt','instruction_exposure_ids':'exposure','assessment_id':'assessment'},
 'teaching_question': {'episode_id':'teaching_episode','concept_id':'concept','source_ids':'evidence|source','responds_to_question_id':'teaching_question','supersedes_question_id':'teaching_question'},
 'teaching_attempt': {'question_id':'teaching_question','response_evidence_id':'evidence','exposure_ids':'exposure'},
}
ENDPOINTS = {
 'from_session': ('evidence','session'), 'supports':('evidence','gap'), 'challenges':('evidence','gap'),
 'addresses':('evidence','gap'), 'resolves':('evidence','gap'), 'about':('gap','concept'),
 'involves':('gap','entity'), 'applies_to':('concept','entity'), 'instance_of':('concept','concept'),
 'prerequisite_for':('concept','concept'), 'related_to':('concept|gap','concept|gap'),
 'depends_on':('entity','entity'), 'shares_semantics_with':('entity','entity'), 'assumes':('entity','concept'),
 'tracks_resolution':('gap_resolution','gap'), 'based_on':('assessment|gap_resolution','attempt|teaching_attempt|assessment|evidence|source'),
 'supersedes':('assessment|gap_resolution','assessment|gap_resolution'), 'probes':('question|teaching_question','concept'),
 'answers':('attempt|teaching_attempt','question|teaching_question'), 'assesses':('assessment','concept'),
 'requires':('step','concept'), 'yields':('step','concept'), 'targets':('step|branch|boundary|teaching_episode','gap'),
 'locates':('boundary','concept'), 'blocked_by':('question|step|branch|teaching_question|teaching_episode','source_issue'),
 'exposes':('exposure','concept'), 'teaches':('teaching_episode','concept'), 'returns_to':('teaching_episode','teaching_episode')
}

def validate(graph, previous=None, deletions=None):
    schema = read(Path(__file__).resolve().parent.parent / 'references/learning-graph.schema.json')
    shape(graph, schema, schema)
    nodes = {n['id']:n for n in graph['nodes']}
    require(len(nodes) == len(graph['nodes']), 'Duplicate node IDs')
    review, diagnostic, teaching = graph['review'], graph['diagnostic'], graph['teaching']
    ids = [graph['bundle']['id']] + [n['id'] for n in graph['nodes']] + [e['id'] for e in graph['edges']] + [e['id'] for e in review['events']]
    for run in [diagnostic, teaching]:
        if run:
            ids.append(run['id'])
    if teaching:
        ids += [e['id'] for e in teaching['events']]
    require(len(ids) == len(set(ids)), 'IDs must be globally unique across records, edges and runs')
    def refs(value, kinds, where):
        if value is None:
            return
        if isinstance(value, list):
            for child in value:
                refs(child, kinds, where)
            return
        require(value in nodes, where + ': dangling reference ' + value)
        require(nodes[value]['kind'] in kinds.split('|'), where + ': wrong reference kind for ' + value)
    def data(node_id):
        return nodes[node_id]['data']
    def rows(kind):
        return [n for n in graph['nodes'] if n['kind'] == kind]
    def current(concept_id):
        aid = data(concept_id)['current_assessment_id']
        return data(aid) if aid else None
    def known(concept_id):
        return current(concept_id) is not None and current(concept_id)['state'] == 'Known'
    def resolution(gap_id):
        return data(data(gap_id)['current_resolution_id'])
    for n in graph['nodes']:
        for field, kinds in REFS.get(n['kind'], {}).items():
            refs(n['data'][field], kinds, n['id'] + '.' + field)
        if n['kind'] == 'gap':
            refs(n['data']['grounding']['source_ids'], 'evidence|source', n['id'] + '.grounding')
    relations = set()
    for e in graph['edges']:
        sk, tk = ENDPOINTS[e['type']]
        refs(e['source'], sk, e['id'] + '.source'); refs(e['target'], tk, e['id'] + '.target')
        refs(e['source_ids'], 'evidence|source', e['id'] + '.source_ids')
        relation = (e['source'], e['type'], e['target'])
        require(relation not in relations, 'Duplicate structural/semantic edge: ' + str(relation))
        relations.add(relation)
        if e['type'] in ['related_to','shares_semantics_with']:
            require(e['source'] < e['target'] and nodes[e['source']]['kind'] == nodes[e['target']]['kind'], e['id'] + ': invalid symmetric edge')
        if e['type'] == 'based_on':
            target_kind = nodes[e['target']]['kind']
            require(target_kind in ('assessment','evidence','source') if nodes[e['source']]['kind']=='gap_resolution' else target_kind in ('attempt','teaching_attempt','evidence','source'), e['id'] + ': invalid based_on endpoint')
        if e['type'] == 'supersedes':
            require(nodes[e['source']]['kind'] == nodes[e['target']]['kind'], e['id'] + ': cross-kind history')
    def links(node_id, edge_type, expected):
        actual = {target for source, kind, target in relations if source == node_id and kind == edge_type}
        require(actual == set(expected), node_id + ': ' + edge_type + ' edges do not match fields: expected ' + repr(sorted(set(expected))) + ', got ' + repr(sorted(actual)))
    for n in graph['nodes']:
        i, k, d = n['id'], n['kind'], n['data']
        if k == 'evidence':
            dest = [t for s,e,t in relations if s==i and e=='from_session']
            require(len(dest)==1 and data(dest[0])['coverage']!='unavailable', i + ': evidence needs one available session')
        if k == 'concept' and d['current_assessment_id']:
            require(data(d['current_assessment_id'])['concept_id']==i, i + ': current assessment belongs to another concept')
        if k == 'gap':
            require(resolution(i)['gap_id']==i, i + ': resolution belongs to another gap')
            expected = 'tentative' if d['confidence']=='low' else ('resolved' if d['status']=='resolved_within_bundle' else 'ongoing')
            require(d['classification']==expected, i + ': inconsistent historical classification')
            require(any(s==i and e=='about' for s,e,t in relations), i + ': gap needs an about concept')
            require(any(e=='supports' and t==i for s,e,t in relations), i + ': gap needs support evidence')
            if d['status']=='resolved_within_bundle':
                require(d['remaining_clarification'] is None and any(e=='resolves' and t==i and data(s)['speaker']=='user' for s,e,t in relations), i + ': historical resolution needs user evidence')
        if k in ['assessment','gap_resolution']:
            predecessor = 'previous_assessment_id' if k=='assessment' else 'previous_resolution_id'
            owner = 'concept_id' if k=='assessment' else 'gap_id'
            links(i, 'supersedes', [d[predecessor]] if d[predecessor] else [])
            seen={i}; prior=d[predecessor]
            while prior:
                require(prior not in seen, i + ': cyclic history')
                seen.add(prior)
                require(data(prior)[owner]==d[owner], i + ': history crosses concepts/gaps')
                prior=data(prior)[predecessor]
        if k == 'assessment':
            links(i,'assesses',[d['concept_id']]); links(i,'based_on',d['attempt_ids']+d['evidence_ids'])
            if d['state']!='Unassessed':
                require(d['evidence_ids'] or d['attempt_ids'], i + ': knowledge state lacks evidence')
                require(d['attempt_ids'] or any(nodes[x]['kind']=='evidence' and data(x)['speaker']=='user' for x in d['evidence_ids']), i + ': learner state needs attributable user evidence, not sources alone')
            has_components = 'demonstrated_components' in d or 'missing_components' in d
            if has_components:
                require('demonstrated_components' in d and 'missing_components' in d, i + ': component fields must occur together')
                if d['state'] in ['Known','Partial']:
                    require(d['demonstrated_components'], i + ': state requires a demonstrated component')
                if d['state'] in ['Unknown','Partial']:
                    require(d['missing_components'], i + ': state requires a missing component')
            elif not graph['bundle']['schema_history']:
                require(False, i + ': new assessment requires component arrays')
            for aid in d['attempt_ids']:
                question = data(data(aid)['question_id'])
                tested = question.get('concept_ids', [question.get('concept_id')])
                require(d['concept_id'] in tested or data(aid)['response_evidence_id'] in d['evidence_ids'], i + ': incidental concept assessment must cite the actual response directly')
        if k == 'gap_resolution':
            links(i,'tracks_resolution',[d['gap_id']]); links(i,'based_on',d['assessment_ids']+d['evidence_ids'])
            if data(d['gap_id'])['current_resolution_id']==i and d['status']=='resolved':
                require(d['assessment_ids'] and d['evidence_ids'], i + ': resolved needs assessments and evidence')
                for aid in d['assessment_ids']:
                    a=data(aid)
                    require(a['state']=='Known' and data(a['concept_id'])['current_assessment_id']==aid, i + ': resolved support is not current Known')
        if k in ['question','teaching_question']:
            links(i,'probes',d['concept_ids'] if k=='question' else [d['concept_id']])
            attempts=[x for x in graph['nodes'] if x['kind'] in ['attempt','teaching_attempt'] and x['data']['question_id']==i]
            if d['status'] in ['answered','skipped']:
                require(attempts, i + ': response status lacks an attempt')
            if d['status']=='pending':
                require(not attempts, i + ': attempted question cannot remain pending')
                if k=='question' or d['purpose'] in ['entry','verification']:
                    require(d['grounding'] in ['verified','hypothetical'] and ((d['expected_answer'] or '').strip() or (d['rubric'] or '').strip()), i + ': pending scored prompt needs a grounded key')
            if d['status']=='skipped':
                require(any(x['data']['result']=='skipped' for x in attempts), i + ': skipped question lacks skipped attempt')
            if k=='teaching_question':
                require(data(d['episode_id'])['concept_id']==d['concept_id'], i + ': episode tests another concept')
        if k in ['attempt','teaching_attempt']:
            links(i,'answers',[d['question_id']])
            require(data(d['response_evidence_id'])['speaker']=='user', i + ': response must be actual user evidence')
            q=data(d['question_id'])
            require(q['status'] in ['answered','skipped','voided'], i + ': attempt has no response status')
            if d['result'] not in ['skipped','not_graded']:
                require(q['grounding'] in ['verified','hypothetical'] and ((q['expected_answer'] or '').strip() or (q['rubric'] or '').strip()), i + ': scored answer lacks usable grounded key')
                require(d['expected_answer_snapshot']==q['expected_answer'] and d['rubric_snapshot']==q['rubric'], i + ': key snapshots differ from frozen question')
            if k=='teaching_attempt' and q['purpose'] in ['rationale','coaching']:
                require(d['result'] in ['not_graded','skipped'], i + ': rationale/coaching must not be graded')
        if k=='step':
            for issue in d['source_issue_ids']:
                if data(issue)['status']!='resolved':
                    require((i,'blocked_by',issue) in relations, i + ': missing source-blocker edge')
            prerequisites={x for group in d['prerequisite_sets'] for x in group}
            links(i,'requires',prerequisites); links(i,'yields',[d['target_concept_id']]); links(i,'targets',d['target_gap_ids'])
            require(d['target_concept_id'] not in prerequisites, i + ': self prerequisite')
            if d['kind']=='inference':
                require(d['prerequisite_sets'], i + ': inference needs prerequisites')
        if k in ['branch','boundary']:
            links(i,'targets',[d['target_gap_id']])
        if k=='branch':
            for issue in d['source_issue_ids']:
                if data(issue)['status']!='resolved':
                    require((i,'blocked_by',issue) in relations, i + ': missing source-blocker edge')
            if d['boundary_id']:
                require(data(d['boundary_id'])['branch_id']==i, i + ': boundary belongs to another branch')
            if d['status']=='boundary_found':
                require(d['boundary_id'] and data(d['boundary_id'])['status']=='established', i + ': boundary_found lacks established boundary')
            if d['status']=='target_demonstrated':
                require(known(d['target_concept_id']), i + ': target_demonstrated without current Known')
        if k=='boundary':
            links(i,'locates',d['known_concept_ids']+d['frontier_concept_ids'])
            require(data(d['branch_id'])['target_gap_id']==d['target_gap_id'], i + ': boundary target mismatch')
            if d['status']=='established':
                require(d['known_concept_ids'] and d['frontier_concept_ids'], i + ': empty established boundary')
                for c in d['known_concept_ids']:
                    require(known(c), i + ': anchor is not current Known')
                for c in d['frontier_concept_ids']:
                    require(current(c) and current(c)['state'] in ['Partial','Unknown'], i + ': frontier is not current Partial/Unknown')
                    supporting=[data(x) for x in d['step_ids'] if data(x)['status']=='supported' and data(x)['target_concept_id']==c]
                    require(any(any(set(group)<=set(d['known_concept_ids']) for group in step['prerequisite_sets']) for step in supporting), i + ': no supported complete prerequisite set anchors frontier')
                require({data(c)['current_assessment_id'] for c in d['known_concept_ids']+d['frontier_concept_ids']} <= set(d['assessment_ids']), i + ': current boundary observations missing')
        if k=='source_issue' and d['status']=='resolved':
            require(d['resolution'] and d['source_ids'], i + ': resolved source issue lacks resolution/source')
        if k=='exposure':
            links(i,'exposes',d['concept_ids'])
        if k=='teaching_episode':
            seen={i}; parent=d['parent_episode_id']
            while parent:
                require(parent not in seen, i + ': cyclic episode ancestry')
                seen.add(parent);parent=data(parent)['parent_episode_id']
            links(i,'teaches',[d['concept_id']]); links(i,'targets',[d['terminal_gap_id']]); links(i,'returns_to',[d['parent_episode_id']] if d['parent_episode_id'] else [])
            if d['parent_episode_id']:
                require(data(d['parent_episode_id'])['terminal_gap_id']==d['terminal_gap_id'], i + ': parent has different terminal')
            for field in ['entry_question_id','verification_question_ids','rationale_question_ids']:
                qs=d[field] if isinstance(d[field],list) else [d[field]] if d[field] else []
                for qid in qs:
                    require(data(qid)['episode_id']==i, i + ': question belongs to another episode')
            for aid in d['response_attempt_ids']:
                require(data(data(aid)['question_id'])['episode_id']==i, i + ': attempt belongs to another episode')
            if d['assessment_id']:
                require(data(d['assessment_id'])['concept_id']==d['concept_id'], i + ': assessment for another concept')
            if d['outcome']=='understood':
                require(d['phase']=='done' and d['assessment_id'] and data(d['assessment_id'])['state']=='Known', i + ': understood episode lacks Known assessment')
                a=data(d['assessment_id'])
                if a['basis']!='prior_evidence':
                    verified=any(data(x)['result']=='correct' and data(data(x)['question_id']).get('purpose')=='verification' and data(data(x)['question_id']).get('episode_id')==i for x in a['attempt_ids'])
                    require(verified and d['instruction_exposure_ids'], i + ': understood cycle requires instruction and fresh verification')
    # Bundle and review selection.
    refs(graph['bundle']['session_ids'],'session','bundle.session_ids'); refs(graph['bundle']['priority_gap_ids'],'gap','bundle.priority_gap_ids')
    require(set(graph['bundle']['session_ids'])=={n['id'] for n in rows('session')}, 'bundle.session_ids must list every session')
    refs(review['selected_gap_ids'],'gap','review.selected_gap_ids')
    labels=[x['label'] for x in review['entries']]; entries={x['gap_id']:x for x in review['entries']}
    require(len(set(labels))==len(labels) and len(entries)==len(review['entries']), 'Duplicate review labels or entries')
    for entry in review['entries']:
        refs(entry['gap_id'],'gap','review.entry'); refs(entry['successor_gap_ids'],'gap','review.successors')
        require(entry['decision']=='superseded' or not entry['successor_gap_ids'], 'Only superseded entries have successors')
    for event in review['events']:
        refs(event['gap_ids'],'gap',event['id']); refs(event['evidence_ids'],'evidence|source',event['id'])
        require(event['revision']<=review['revision'], event['id'] + ': future review revision')
    if review['status']=='pending':
        require(review['finalized_revision'] is None and not review['selected_gap_ids'], 'Pending review cannot have effective selections')
        for run in [diagnostic,teaching]:
            require(run is None or run['status']=='paused', 'Pending review must pause diagnostic and teaching')
    else:
        require(review['finalized_revision']==review['revision'], 'Finalized review revision mismatch')
        require(not any(x['decision']=='pending' for x in review['entries']), 'Finalized review has pending decisions')
        require(set(review['selected_gap_ids'])=={x['gap_id'] for x in review['entries'] if x['decision']=='keep'}, 'Selected targets differ from keep decisions')
    # Only one stage can own a pending prompt, even while paused.
    pending=[n for n in graph['nodes'] if n['kind'] in ['question','teaching_question'] and n['data']['status']=='pending']
    require(len(pending)<=1, 'Multiple pending prompts')
    for name, run, qkind in [('diagnostic',diagnostic,'question'),('teaching',teaching,'teaching_question')]:
        ps=[n['id'] for n in pending if n['kind']==qkind]
        require(ps==([run['pending_question_id']] if run and run['pending_question_id'] else []), name + ': pending pointer mismatch')
        if not run:
            continue
        refs(run['target_gap_ids'],'gap',name+'.targets'); refs(run['pending_question_id'],qkind,name+'.pending')
        if run['status']=='active':
            require(review['status']=='finalized' and run['selected_review_revision']==review['revision'] and set(run['target_gap_ids'])==set(review['selected_gap_ids']), name + ': active scope is not finalized current selection')
        if run['status'] in ['complete','blocked']:
            require(run['pending_question_id'] is None, name + ': complete/blocked run has pending prompt')
    if diagnostic:
        refs(diagnostic['branch_ids'],'branch','diagnostic.branches'); refs(diagnostic['active_branch_id'],'branch','diagnostic.active_branch')
        if diagnostic['active_branch_id']:
            require(diagnostic['active_branch_id'] in diagnostic['branch_ids'], 'Active diagnostic branch not in run')
        if diagnostic['pending_question_id']:
            require(data(diagnostic['pending_question_id'])['branch_id']==diagnostic['active_branch_id'], 'Pending diagnostic question belongs to another branch')
            if review['status']=='finalized':
                require(data(diagnostic['active_branch_id'])['target_gap_id'] in review['selected_gap_ids'], 'Pending diagnostic target is outside selected scope')
        if diagnostic['status']=='complete':
            for gid in diagnostic['target_gap_ids']:
                if resolution(gid)['status'] in ['resolved','retracted']:
                    continue
                branches=[data(x) for x in diagnostic['branch_ids'] if data(x)['target_gap_id']==gid and data(x)['status']!='out_of_scope']
                require(branches and all(b['status'] in ['boundary_found','target_demonstrated'] for b in branches), 'Completed diagnostic lacks boundary coverage: '+gid)
    if teaching:
        refs(teaching['active_terminal_gap_id'],'gap','teaching.active_terminal');refs(teaching['last_completed_terminal_gap_id'],'gap','teaching.last_completed');refs(teaching['stack'],'teaching_episode','teaching.stack')
        terminals={t['gap_id']:t for t in teaching['terminals']}
        require(len(terminals)==len(teaching['terminals']), 'Duplicate teaching terminals')
        for t in teaching['terminals']:
            for field, kinds in {'gap_id':'gap','concept_ids':'concept','assessment_ids':'assessment','evidence_ids':'evidence|source','episode_ids':'teaching_episode','source_issue_ids':'source_issue'}.items():
                refs(t[field],kinds,'teaching.terminal.'+field)
            if t['status']=='understood':
                require(t['concept_ids'] and t['assessment_ids'] and t['evidence_ids'], 'Understood terminal lacks coverage/evidence')
                require(all(known(c) for c in t['concept_ids']), 'Understood terminal contains non-Known concept')
                require({data(c)['current_assessment_id'] for c in t['concept_ids']}<=set(t['assessment_ids']), 'Understood terminal lacks current assessments')
                require(resolution(t['gap_id'])['status']=='resolved', 'Understood terminal must append a resolved KU record')
        for event in teaching['events']:
            for field,kinds in {'concept_ids':'concept','gap_ids':'gap','episode_ids':'teaching_episode','question_ids':'question|teaching_question','evidence_ids':'evidence|source'}.items():
                refs(event[field],kinds,event['id'])
        for index, eid in enumerate(teaching['stack']):
            d=data(eid)
            require(d['terminal_gap_id']==teaching['active_terminal_gap_id'], 'Stack terminal mismatch')
            require(d['parent_episode_id']==(teaching['stack'][index-1] if index else None), 'Broken teaching stack ancestry')
            if index:
                require(d['concept_id'] in data(teaching['stack'][index-1])['prerequisite_set'], 'Active child is not a selected parent prerequisite')
            if index<len(teaching['stack'])-1:
                require(d['phase']=='prerequisites', 'Suspended parent must await prerequisites')
        if teaching['pending_question_id']:
            q=data(teaching['pending_question_id'])
            require(teaching['stack'] and q['episode_id']==teaching['stack'][-1], 'Teaching prompt is not top of stack')
            phase={'entry':'entry','verification':'verify','rationale':'rationale','coaching':'coach'}[q['purpose']]
            require(data(q['episode_id'])['phase']==phase, 'Teaching prompt/phase mismatch')
            if review['status']=='finalized':
                require(teaching['active_terminal_gap_id'] in review['selected_gap_ids'], 'Pending teaching target is outside selected scope')
        if teaching['status']=='complete':
            require(not teaching['stack'] and teaching['active_terminal_gap_id'] is None, 'Complete teaching run has active stack/terminal')
            for gid in teaching['target_gap_ids']:
                require(resolution(gid)['status']=='retracted' or gid in terminals and terminals[gid]['status']=='understood', 'Complete teaching run has unfinished terminal: '+gid)
    if previous is not None:
        history(graph,previous,deletions)
    return {'valid':True,'schema_version':'5.0','nodes':len(nodes),'edges':len(graph['edges'])}

def history(graph, previous, deletions=None):
    require(previous.get('schema_version') in ['1.0','2.0','3.0','4.0','5.0'], 'Cannot compare unknown previous schema')
    require(graph['bundle']['id']==previous['bundle']['id'], 'Bundle ID changed')
    require(graph['bundle']['generated_at']==previous['bundle']['generated_at'], 'Original generation time changed')
    old={n['id']:n for n in previous['nodes']}; new={n['id']:n for n in graph['nodes']}
    exceptions=set()
    if deletions is not None:
        require(set(deletions)=={'record_ids','evidence_ids','reason'} and isinstance(deletions['record_ids'],list) and isinstance(deletions['evidence_ids'],list) and bool(deletions['reason']), 'Invalid deletion manifest')
        require(deletions['evidence_ids'] and all(x in new and new[x]['kind']=='evidence' and new[x]['data']['speaker']=='user' for x in deletions['evidence_ids']), 'Deletion requires retained evidence of explicit user instruction')
        exceptions=set(deletions['record_ids'])
        require(exceptions <= set(old)|{e['id'] for e in previous['edges']}|{e['id'] for e in previous.get('review',{}).get('events',[])}|{e['id'] for e in (previous.get('teaching') or {}).get('events',[])}, 'Deletion manifest names unknown records')
    immutable={'assessment','gap_resolution','attempt','teaching_attempt','evidence','source','exposure'}
    for nid,n in old.items():
        if nid in exceptions:
            continue
        require(nid in new, 'Historical record removed: '+nid)
        current=new[nid]
        require(current['kind']==n['kind'], 'Record kind changed: '+nid)
        if n['kind'] in immutable:
            require(current==n, 'Immutable historical record changed: '+nid)
        if n['kind']=='gap':
            for key,value in n['data'].items():
                if key!='current_resolution_id':
                    require(current['data'][key]==value, 'Original gap field changed: '+nid+'.'+key)
        if n['kind'] in ['question','teaching_question']:
            for key,value in n['data'].items():
                if key!='status':
                    require(current['data'][key]==value, 'Frozen question field changed; supersede instead: '+nid+'.'+key)
        if n['kind']=='boundary':
            require(all(current['data'][k]==v for k,v in n['data'].items() if k!='status'), 'Historical boundary rewritten: '+nid)
        if n['kind']=='teaching_episode' and n['data']['outcome']=='understood':
            require(current==n, 'Finished episode rewritten; create a new episode: '+nid)
        if n['kind'] in ['concept','gap']:
            pointer='current_assessment_id' if n['kind']=='concept' else 'current_resolution_id'
            prior_field='previous_assessment_id' if n['kind']=='concept' else 'previous_resolution_id'
            before=n['data'].get(pointer); after=current['data'].get(pointer)
            if before and after!=before:
                require(after and after not in old, 'Current pointer must advance to an appended record: '+nid)
                seen=set(); cursor=after
                while cursor and cursor!=before and cursor not in seen:
                    seen.add(cursor);cursor=new[cursor]['data'][prior_field]
                require(cursor==before, 'New history is disconnected from prior current record: '+nid)
    for nid,n in new.items():
        if nid not in old and n['kind']=='assessment':
            require('demonstrated_components' in n['data'] and 'missing_components' in n['data'], 'New assessment lacks components: '+nid)
    new_edges={e['id']:e for e in graph['edges']}
    for edge in previous['edges']:
        if edge['id'] not in exceptions:
            require(new_edges.get(edge['id'])==edge, 'Historical edge removed/rewritten: '+edge['id'])
    def append_only(old_items,new_items,label):
        retained=[x for x in old_items if x.get('id') not in exceptions]
        require(new_items[:len(retained)]==retained, label+' must preserve its history prefix')
    append_only(previous['bundle'].get('schema_history',[]),graph['bundle']['schema_history'],'schema_history')
    if previous.get('review'):
        old_entries={x['gap_id']:x for x in previous['review']['entries']}
        new_entries={x['gap_id']:x for x in graph['review']['entries']}
        for gid,entry in old_entries.items():
            if gid in exceptions:
                continue
            require(gid in new_entries and new_entries[gid]['label']==entry['label'], 'Review entry/label lost: '+gid)
        if new_entries!=old_entries:
            require(graph['review']['revision']>previous['review']['revision'], 'Changed selection/scope needs a new review revision')
        append_only(previous['review']['events'],graph['review']['events'],'review.events')
        require(graph['review']['revision']>=previous['review']['revision'], 'Review revision moved backwards')
    for name in ['diagnostic','teaching']:
        before=previous.get(name); after=graph[name]
        if before:
            require(after is not None and before['id']==after['id'] and before['started_at']==after['started_at'], name+': run identity/history lost')
            require(after['checkpoint_seq']>=before['checkpoint_seq'], name+': checkpoint count moved backwards')
            if name=='teaching':
                append_only(before['events'],after['events'],'teaching.events')
    return True

def main():
    parser=argparse.ArgumentParser(description=__doc__)
    parser.add_argument('graph');parser.add_argument('--previous');parser.add_argument('--deletions')
    args=parser.parse_args()
    result=validate(read(args.graph),read(args.previous) if args.previous else None,read(args.deletions) if args.deletions else None)
    print(json.dumps(result))

if __name__=='__main__':
    try:
        main()
    except (ValueError,KeyError,TypeError,OSError) as error:
        print('INVALID: '+str(error),file=sys.stderr);sys.exit(1)
````
<!-- END EMBEDDED FILE -->
