# Venue Handover Inspector

An offline browser workbench for a community venue team to record room conditions before an event, inspect again afterward, and hand over a printable account of what changed.

## Product requirements

**Audience and problem.** A staff member opens a fictional community hall before a booking and another closes it later. When room lists change or photos live separately from the checklist, it is hard to tell whether an item was removed from the checklist, was never inspected, or was actually found in a different condition.

**Outcome.** Start a handover from a room template, record readiness and counts, and mark inaccessible items as unseen with a reason. Finalize the baseline locally, then open a follow-up against a specific template revision. Review changes by stable item ID, see linked evidence and unseen reasons, and print the completed handover. Reopening the app shows the same closed snapshots and comparison.

**First durable milestone.** Open a venue fixture in Chrome, record one room’s conditions, close the server and resume the same draft. When each row has an observation, finalize the baseline and verify its read-only snapshot after a fresh restart.

| User | Needs |
| --- | --- |
| Opening volunteer | Start from an agreed checklist, record a count or condition, and leave a draft for later. |
| Closing volunteer | Inspect the requested checklist revision and see which baseline observations can be compared. |
| Venue coordinator | Review change and coverage categories, see photo evidence in context, and print a concise handover. |

**Release scope.** One fictional venue, one local data directory, a before/after pair per handover, a local template editor with stable item IDs, and a printable report. Include a list of handovers, a room-by-room checklist, a template revision selector, and a comparison view with category counts and an evidence preview. A draft remains editable and includes explicit save feedback. Finalizing locks the current local snapshot and records its local timestamp. It does not identify a person or serve as a digital signature.

Limit evidence to bundled original synthetic PNGs representing staged room details. No venue photos, contacts, booking records, uploads, or camera permissions are needed. Incoming or outgoing bookings, cloud sharing, remote teams, legal condition reports, and compliance assessments are outside the release scope.

**Accessibility.** Use headings, field labels, required indicators in text, visible focus, native controls where possible, and named section links. All edit, finalize, comparison, evidence, and print-preview paths are reachable by keyboard. Distinguish observation and comparison categories using text, independent of color. Make save and validation feedback perceivable without moving focus unexpectedly. Give evidence images meaningful text alternatives and retain readable notes when images are unavailable.

**Proposed decisions to confirm.** Begin with room and equipment count checks and three condition values. Confirm whether future users need richer condition categories or people to share a workspace. The first release uses fixture data and modelled staff roles.

## Technical design

### Runtime and boundaries

Use Python 3 standard-library HTTP, JSON, hashing and SQLite modules, a local database file, and bundled HTML, CSS and JavaScript. A documented command initializes a new local data directory with the fictional fixtures and prints a loopback address for Chrome. On a normal restart, open the existing directory and preserve its contents. Refuse accidental reinitialization and explain how to choose an empty demo path. Keep all fonts, assets, scripts and reports local. Check Python, SQLite and Chrome availability before implementation.

Serve requests from 127.0.0.1 and use the same origin for application requests. Use typed JSON for mutations, bounded UTF-8 body and field lengths, expected-origin validation, and read-only GET routes. Use parameterized queries. Render labels and notes as text, including printable views. Limit templates to 12 rooms and 100 items, observations to 500 characters per note, and evidence to a whitelisted fixture identifier per observed item. Static image bytes come only from a bundled registry. A malformed request leaves persistent records unchanged.

### Data model and invariants

| Record | Stored data and relationship |
| --- | --- |
| Room and item identities | Stable opaque IDs. A room can have multiple items. Labels and order are stored in template versions. Retire an ID without reuse. |
| Template version | Parent template version ID if any, ordered rooms and items, display labels, and creation sequence. Publishing inserts an immutable full snapshot, even if only one label changes. |
| Handover | Generated ID and title, one baseline round, and zero or one follow-up round. A follow-up belongs to exactly one finalized baseline. |
| Round | Baseline or follow-up, pinned template version, draft revision, state, and finalization sequence when closed. Any later template edit leaves the round unchanged. |
| Observation | One row per pinned item ID and round once entered, with condition, optional count or unseen reason, note, and optional evidence key. |
| Evidence registry | Bundled synthetic image key, digest, alternative text, and path in the packaged asset directory. A final snapshot copies the key, digest and alternative text for each selected image. |
| Final snapshot | Immutable round metadata, template and item display metadata, finalized observations and monotonic finalize sequence. Store local time for display. Timeline order uses sequence to tolerate clock changes. |
| Command record | Unique command key, canonical request digest, and result. Write it atomically with data for successful mutation replay. |

Condition values are **Ready**, **Needs attention**, and **Unseen**. Ready requires a whole count from 1 to 250. Needs attention allows 0 through 250 and requires a note and one synthetic image. Unseen requires a reason, has no count and cannot have an image. A missing observation means unentered draft work. It cannot stand for Unseen in a final snapshot. Reject wrong-template item IDs, invalid counts, and unknown evidence keys. Track a metadata label change separately from changes in condition or count.

### State transitions and recovery

1. Create the baseline round as a draft with a current published template version. Save observations using the pinned item set. Show remaining unentered rows and all validation errors before finalize.
2. Finalize the baseline only once. In a single SQLite transaction, validate a complete round, insert its final snapshot and close the draft. The baseline then stays readable. Revision metadata and rows in published snapshots are immutable through the application.
3. Create a follow-up draft only after baseline finalization. The coordinator selects the baseline template version or a published successor. The new round has empty observations even for items already seen. The original round remains visible for reference, including any Unseen record.
4. Finalize the follow-up with the same completeness and evidence checks. Only a finalized baseline and follow-up produce a final comparison and print view. Draft progress can be viewed without final totals.
5. After restart, list the saved handovers and reopen their latest committed state. A request that never committed leaves the previous revision. If a response is lost after commit, retry with the same command key and same canonical payload to recover the prior result. Reusing that key with a different request yields a conflict.

Carry a base revision on edits and finalization. In a short write transaction, look up the command key, verify the revision before applying the mutation, then update rows, increment the revision and save the result together. Serialize eligibility checks and finalization in that transaction. A competing edit to the same round gets a stale-revision response and a reload action. A competing follow-up creation gets the existing round link or a conflict without creating a second round. Report database contention, unavailable fixture assets and invalid round state as distinct recoverable results. Show print diagnostics if locally missing asset bytes prevent evidence from being rendered. The structured snapshot remains readable in that case.

### Comparison contract

Join the two finalized template item sets by stable item ID. Return categories in fixed order Added, Removed, Changed, Unseen, Unchanged. Sort rows by item ID within each category. Show the pinned room and item labels for each applicable side, including the reason when either side marks Unseen.

1. **Added** applies when the item exists only in the follow-up template. **Removed** applies when it exists only in the baseline template. The membership category wins even if the one observation is Unseen. Display the observation that is present.
2. **Unseen** applies when the item exists in both templates and either observation is Unseen. Emit each affected side and reason. Never infer a loss from an unobserved item.
3. **Changed** applies to comparable observations when condition or count differs. Report individual old and new fields.
4. **Unchanged** applies to all remaining comparable observations. Changes to labels, notes or linked assets remain visible as metadata or evidence history. They do not on their own make an item Changed.

Each union item appears exactly once. Count categories by rows and verify their sum equals the union size. The structured comparison has named category keys, ordered row lists, both snapshot references, and before/after observations and evidence descriptors where applicable. Generate it from saved snapshots on demand. The report repeats the same categories and order, includes a textual evidence descriptor and image for applicable rows, and uses print CSS to keep headings and evidence readable. Printing does not edit either round.

### Interface and failure responses

Provide documented local routes to list handovers and templates, read a round, save a typed observation, finalize a round, create the follow-up, and look up and print a final comparison. A concise same-origin JSON error includes a stable code and an affected field or state. Distinguish unentered requirements, unknown evidence, stale revision, already finalized, missing baseline, and missing assets. The navigation preserves the active room and keeps a newly rejected field in view. A separate warning and explicit retry action communicates temporary database contention. Keyboard inspection and print checks use bundled fixture data in Chrome.

## Original verification fixtures

The following values are expected results derived from a synthetic fixture. Application and browser verification are planned work.

**Birch Hall fixture.** The baseline template has six items across the Workshop and Studio rooms. The follow-up removes VH-I003, adds VH-I005, and renames the label for VH-I006 with the same stable ID. It changes no other item identity. Before and after templates each contain six IDs, five of which are shared. The Needs attention observation for VH-I002 uses a generated image with original synthetic pixels. Its evidence key is IMG-CASE-01 with alternative text “Fictional case with a striped corner marking.”

| ID | Item | Baseline | Follow-up | Expected category |
| --- | --- | --- | --- | --- |
| VH-I001 | Stacking chairs | Ready, 4 | Ready, 3 | Changed |
| VH-I002 | Storage case | Ready, 1 | Needs attention, 1, note and IMG-CASE-01 | Changed |
| VH-I003 | Notice board | Ready, 1 | Not in template | Removed |
| VH-I004 | Timer | Ready, 1 | Unseen, shelf locked | Unseen |
| VH-I005 | Roll-up sign | Not in template | Ready, 1 | Added |
| VH-I006 | Cable set | Ready, 2 | Ready, 2, label renamed to Cable kit | Unchanged |
| VH-I007 | Shelf mats | Unseen, label unreadable | Ready, 2 | Unseen |

Expected output order is Added [VH-I005], Removed [VH-I003], Changed [VH-I001, VH-I002], Unseen [VH-I004, VH-I007], Unchanged [VH-I006]. Counts are 1, 1, 2, 2, 1 for seven total rows. VH-I001 shows a count change only. VH-I002 shows a condition change only and includes the expected synthetic image descriptor. VH-I004 points to the follow-up unseen reason and VH-I007 to the baseline reason. The label rename for VH-I006 remains visible without creating a change row.

**Membership edge.** Change VH-I005 to Unseen with the reason “not accessible.” It still appears under Added, with that observation visible. Category totals stay the same. In a second variation, give the follow-up snapshot a new note for VH-I006. It stays Unchanged. Change VH-I002 to an unentered observation before follow-up finalize and expect a blocking list containing VH-I002 and no final comparison.

**Durability and evidence checks.** Save the baseline observation for VH-I001 at draft revision 1 in one tab. Reject the competing save at revision 1 in another tab after the first succeeds. After server restart the saved Ready count is 4. A retry of the successful command returns its prior result without a second revision increment. A save with an unknown synthetic evidence key fails. The rejected observation remains unentered and prevents finalization. After completing both rounds, restart with the same fixture assets and verify that the normalized comparison and expected evidence digest match. In a disposable copy, an unavailable synthetic asset surfaces an evidence diagnostic while the snapshot remains readable.

## Epic · Inspect and hand over a fictional venue offline

Deliver a repeatable before/after inspection, exact item-level comparison, and printable handover with original synthetic evidence. The planned implementation and verification effort is **14–22 hours**. Complete VH-01 first. VH-02 and VH-03 can then proceed in parallel against the pinned template and observation designs. Complete the follow-up and comparison in VH-04, the integrated print handover in VH-05, and the independent offline verification in VH-06. The first durable milestone is a baseline draft that survives restart and can become an immutable snapshot. The completed epic adds follow-up and exact coverage semantics, evidence, and end-to-end verification.

### VH-01 · Set up the local venue and versioned room templates

**User story VH-S01.** As a venue coordinator, I want a baseline checklist with stable room and item IDs so that changing a label or selecting a later template does not change what an earlier checklist asked.

**Estimate.** 2–3 hours. **Dependencies.** None.

- **VH-01-A.** Initialize a new fixture directory, open the local dashboard, and read the expected six baseline IDs. A second initialization against the same directory fails and leaves the database unchanged.
- **VH-01-B.** Publish the follow-up template as an immutable revision. It keeps shared IDs, removes VH-I003, adds VH-I005, and records the rename for VH-I006. Reopen both versions and verify their original labels and memberships.
- **VH-01-C.** Reject a duplicate ID or blank required label without publishing a revision. Verify that a newly added item uses an independent ID even when its label matches a retired one.

### VH-02 · Save observations and finalize a baseline

**User story VH-S02.** As an opening volunteer, I want to leave and return to a draft and then lock the exact observations I finished.

**Estimate.** 3–4 hours. **Dependencies.** VH-01.

- **VH-02-A.** Save an observation, close Chrome and restart the server. Verify the count and draft revision after reopening. A second tab with the old revision receives a conflict and can reload the committed values.
- **VH-02-B.** A missing row blocks finalization. Recording Unseen with a reason clears that row’s missing requirement. An out-of-range count or an item from a different template is rejected without changing the revision.
- **VH-02-C.** Finalize a complete baseline with Ready and Unseen observations. Retrying its successful command leaves a single snapshot. Further edits to the finalized round and later template changes do not alter that snapshot.

### VH-03 · Attach and verify bundled evidence

**User story VH-S03.** As an inspection volunteer, I want to link a detail image to a fictional item so I can revisit the condition finding without leaving the checklist.

**Estimate.** 2–4 hours. **Dependencies.** VH-01. Build against the observation contract agreed with VH-02.

- **VH-03-A.** Select IMG-CASE-01 for VH-I002 while recording Needs attention and show the image and exact alternative text in the keyboard-accessible preview. Saving an unknown image key or an attachment on Unseen fails.
- **VH-03-B.** Finalization requires the Needs attention note and asset. The locked record retains the selected image key and digest. A missing or mismatching file in a disposable test data set produces a diagnostic while retaining the original snapshot fields.
- **VH-03-C.** Verify the packaged synthetic evidence is generated or original, can load on the loopback page with external traffic blocked, and is tracked by the expected digest in the fixture manifest.

### VH-04 · Inspect again and compare finalized snapshots

**User story VH-S04.** As a closing volunteer, I want a follow-up against a chosen template revision and a comparison that distinguishes coverage gaps from actual condition or count changes.

**Estimate.** 3–4 hours. **Dependencies.** VH-02 and VH-03.

- **VH-04-A.** Reject follow-up creation before the baseline is locked. After finalization, choose the successor template and verify its pinned membership and empty draft observations. Repeated requests leave exactly one follow-up.
- **VH-04-B.** Finalize both Birch Hall rounds. Verify exact category order, item IDs, field deltas and counts 1, 1, 2, 2, 1. Verify the number of rows equals seven and the evidence descriptor survives in the normalized result.
- **VH-04-C.** Check the unseen membership and note-only variants. Finalization still rejects an unentered observation. After restart compare again and get the same normalized report data.

### VH-05 · Build an accessible handover summary and print view

**User story VH-S05.** As a venue coordinator, I want one readable handover report with enough detail to follow up on a count, condition or coverage finding.

**Estimate.** 2–4 hours. **Dependencies.** VH-04.

- **VH-05-A.** Navigate by keyboard from handover list to completed comparison, category and evidence preview. Verify labels, visible focus, status text and the exact before and after reasons for both Unseen items.
- **VH-05-B.** Print the final Birch Hall report through Chrome’s print preview. Verify both template references, the five totals, the two changed field summaries, and the case image and caption. Print preview keeps data readable across pages.
- **VH-05-C.** Keep final printing unavailable for an incomplete round with a clear progress message. A missing bundled image renders its caption and diagnostic in the report and leaves the snapshot readable.

### VH-06 · Verify the end-to-end handover in a clean offline workspace

**User story VH-S06.** As a maintainer, I want reproducible evidence that another person can validate the workflow from empty workspace to printed report.

**Estimate.** 2–3 hours. **Dependencies.** VH-05.

- **VH-06-A.** Initialize a disposable Linux demo directory, open Chrome with external traffic blocked, and record the prepared baseline and follow-up. Finalize, restart and verify the hand-derived IDs, counts and printed output.
- **VH-06-B.** Run deterministic checks for mixed comparison, two-tab retry or stale revision, label changes, invalid evidence and missing snapshots. Record the actual outcomes against the acceptance IDs and distinguish any unrun checks.
- **VH-06-C.** Document the local run command, fixtures, evidence bounds and manual keyboard checks. Confirm no contacts, real venue images, external accounts or network dependencies are needed.

## Reference notes

- [SQLite transaction documentation](https://sqlite.org/lang_transaction.html)
- [W3C keyboard accessibility guidance](https://www.w3.org/WAI/WCAG22/Understanding/keyboard.html)
