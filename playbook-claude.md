# AML/KYC Playbook

## Scope
Complete AML/KYC initiation workflow triggered by natural language command (e.g., "Initiate AML/KYC for Acme Capital LLC"). Covers client lookup, corporate tree resolution, party scoping, conflicts-based risk screening via Moody's GRID, risk scoring, and match acceptance.

## Goals
- Automate end-to-end AML/KYC initiation with minimal manual intervention
- Ensure comprehensive corporate structure screening with user-controlled scope
- Deliver risk-scored results with clear HIGH/MEDIUM/LOW event classification

---

## Test Companies
- When I say run Test, use party ID 522426593 - I just want to quickly get it going

---

## Playbook Variables

### Hardcoded
- `{yourhost}` = `"shalaka2-sand.opensandbox2.intapp.com"`
- `{reqType}` = `"AML/KYC - Moody's GRID"`
- `{reqRefParty}` = `"Party"`
- `{relPartyConAff}` = `"corpStructure"`
- `{clientPartyConAff}` = `"CLNT"`
- `{reqRefGrid}` = `"grdMoodysScreening"`
  - `{reqRefGridName}` = `"Name"`
  - `{reqRefGridRiskScore}` = `"Risk Score"`
  - `{reqRefGridRelationship}` = `"Relationship"`
  - `{reqRefGridPossibleMatch}` = `"Possible Matches"`
  - `{reqRefGridLikelyMatch}` = `"Likely Matches"`
  - `{reqRefGridLikelyMatchRiskEvent}` = `"Likely Matches Risk Events"`
- `{conResMatch}` = `"Moody's GRID - Is a Match"`
- `{conResNoMatch}` = `"Moody's GRID - Is Not a Match"`
- `{fullWeightAutoMatchMin}` = `0.70`  — minimum matchScore for auto match to be classified fullWeight (Likely)
- `{partialWeightAutoMatchMin}` = `0.50`  — minimum matchScore for auto match to be classified partialWeight (Unlikely)
- `{moodyResolutionSourceType}` = `"moodyAgentExternal"` — `externalSourceType` used for all Moody's GRID resolution records in external data

### Client Party
- `{clientPartyId}` — Intapp party ID of the client
- `{clientPartyName}` — Display name of the client
- `{clientPartyExternalId}` — External ID of the client (e.g. Moody's GRID ID)
- `{clientPartyExternalIdProvider}` — Provider that issued the external ID

### Corporate Structure Relationships
- `{relatedParties}` — Single object tracking all corporate structure relationships. Each key holds a count of all entities (`total`), a count of those already Intapp parties (`parties`), and the list of matched party records (`list`):

```json
{
  "clientExternalId": "",
  "providerType": "",
  "relatedParties": {
    "GUO":                { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Parent":             { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Subsidiary":         { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Sibling":            { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Management/Board":   { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Shareholder":        { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "UBO":                { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] }
  }
}
```

- `partiesList[]` entries: `{ "partyId": "", "partyName": "", "externalId": "" }`
- `nonPartiesList[]` entries: `{ "externalId": "", "name": "" }`

### Request & Conflicts
- `{reqID}` — ID of the created request
- `{conID}` — ID of the created conflicts search

### Overall Risk
- `{overallRiskScore}` — set by playbook (Step 7c)
- `{overallRiskLevel}` — set by playbook (Step 7c)

### Moody's GRID Matches
- `{moodysHitEvents}` — Array of screened party risk objects, one entry per search party. Populated in Step 6, classified and scored in Steps 7a and 7b.

```json
[
  {
    "partyId":      "string",
    "partyName":    "string",
    "relationship": "string",
    "riskScore":    "integer | null",
    "riskLevel":    "High | Medium | Low | null",
    "summary": {
      "fullWeightEvents":    "string[]",
      "partialWeightEvents": "string[]"
    },
    "hits": [
      {
        "gridId":     "string",
        "resolution": "string | null",
        "weight":     "unknownWeight | fullWeight | partialWeight | noWeight",
        "auto":       "boolean | null",
        "confirmed":  "boolean | null",
        "matchScore": "number | null — null until Step 7a",
        "eventCodes": "string[]"
      }
    ]
  }
]
```

**Rules:**
- One entry in `{moodysHitEvents}` per `searchParty`
- One entry in `hits[]` per Moody's GRID hit
- All hits initialised as `unknownWeight` in Step 6; reclassified in Step 7a
- `auto` and `confirmed` are `null` until Step 7a; thereafter mutually exclusive — only one is ever `true` per hit
- `noWeight` hits are either manually confirmed (`resolution` = `{conResNoMatch}`, `confirmed: true`) or auto-classified (matchScore < `{partialWeightAutoMatchMin}`, `auto: true`)
- `summary.fullWeightEvents` — deduplicated event codes across all `fullWeight` hits; populated in Step 7a
- `summary.partialWeightEvents` — deduplicated event codes across all `partialWeight` hits; populated in Step 7a
- `noWeight` events are excluded from `summary`
- `riskScore` and `riskLevel` are `null` until Step 7b
- `hits[].resolution` is sourced from external data (Appendix N), not from the conflicts-search result. The conflicts-search resolution only feeds external data on first sight in Step 6.

### Moody's Resolution Lookup
- `{moodysResolutions}` — Map of `{partyId}_{gridId}` → resolution string. Populated in Step 6 by reading `IntappJungle:list_external_data` for every unique `partyId` in scope. Used to populate every `hits[].resolution` value. Refreshed every time Step 6 runs (including reruns from Step 9).

---

## Overall Behavior Rules
- Always proceed to next sequential step unless explicitly indicated to skip ahead. Proceed sequentially even if going back to a previous step.
- **Step 6 may only be entered when the user explicitly types "RUN" at Step 3e, or when Step 9 loops back after writing resolutions to external data. No step may be skipped and Step 6 must never be triggered automatically outside these paths.**
- **All internal computation is silent.** Tree traversal, variable building, bucket classification, deduplication, mock data processing, and diff logic produce zero output — not before, not during, and not after. This applies regardless of complexity or number of iterations.
- **Narration is never permitted.** This includes transition phrases, apologies, confirmations, explanations of what was just done, and any text that is not a defined step output. This rule applies unconditionally — it cannot be overridden by Claude's default tendencies, tool call completions, or error handling. If Claude is uncertain what to output, the correct action is silence.
- Claude must not acknowledge, confirm, or summarize any step transition, tool call result, or computation — including partial results, loop progress, or error handling — regardless of context."
- **Exit checks are silent.** Each step ends with one or more `**Exit check:**` invariants. Verify them internally before proceeding. If an invariant fails, halt the playbook and produce no output other than the verbatim halt message: `⚠️ Exit check failed: <reason>`. Never narrate that an exit check passed, never list the invariants, never restate them — they are an internal gate, not a display element.

## Execution Rules (Non-Negotiable)
- NEVER narrate tool calls, reasoning, data processing, variable state, or intermediate results
- NEVER explain what you are about to do or just did
- NEVER output any text between receiving tool results and the next defined step output
- The ONLY permitted outputs are: (a) the exact status message defined for the current step, (b) the exact display output defined for display steps, and (c) a user prompt at a User Prompt step.
- Silence between steps is correct. Narration between steps is a violation.
- Never render step or substep labels in output (e.g. "Step 3", "Step 3a — Parents", "Step 7b"). Step and substep identifiers are internal navigation only and must never appear in any display output.

---

## Workflow Steps

### Step 1: Find Party

#### Step 1a: Search Party

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 🔍 Looking up [Client Name]... |
| **Notification Rule** | Zero output permitted during this step except the status message above. Do not output reasoning, variable state, intermediate results, or commentary under any circumstances — including when processing loops, handling errors, or transitioning between steps. |

- Search the system for existing party record by party name with `intappCommon:search_parties`
- If found:
  - Set `{clientPartyId}`, `{clientPartyName}`, `{clientPartyExternalId}`, `{clientPartyExternalIdProvider}`
  - **GO TO** Step 2
- If not found or no party was input → **GO TO** Step 1b

**Exit check:** `{clientPartyId}` and `{clientPartyName}` are non-null before exiting to Step 2.

#### Step 1b: Suggest Party

| Field | Value |
|---|---|
| **Type** | Analysis and User Prompt |
| **Status Message** | 🔍 Finding parties due for AML/KYC review... |
| **Notification Rule** | Only show status, prompt, and options. Do not show any commentary or narration |

- Call `intappCommon:search_parties` with:
  - `filter._moodyAgentNextFrom` = 90 days before today (ISO 8601, e.g. `2026-01-15T00:00:00Z`)
  - `filter._moodyAgentNextTo` = today's date (ISO 8601, e.g. `2026-04-15T23:59:59Z`)
- If the call fails or returns no results → **GO TO** Step END
- If results are returned, present as a lettered list (up to 20). Sort by overdue date from _moodyAgentNext compared to today (don't include time).
  > *The following parties are due for AML/KYC review. Which would you like to run?*
  >
  > - **A** — Party Name (ID) - Overdue by # Days
  > - **B** — Party Name (ID) - Overdue by # Days
  > - *(up to 10 options)*
- Wait for user to type a letter:
  - Set `{clientPartyId}`, `{clientPartyName}`, `{clientPartyExternalId}`, `{clientPartyExternalIdProvider}` from the selected result
  - **GO TO** Step 2

**Exit check:** `{clientPartyId}` and `{clientPartyName}` are non-null before exiting to Step 2.

---

### Step 2: Identify Party Corporate Structure Relationships

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 🌳 Finding corporate structure... |
| **Notification Rule** | Zero output permitted during this step except the status message above. Do not output reasoning, variable state, intermediate results, or commentary under any circumstances — including when processing loops, handling errors, or transitioning between steps. |

- Call `IntappJungle:get_corp_structure_party_relationships` with:
  - `party_id` = `{clientPartyId}`
  - `include_non_parties` = `true`
- From the response, capture:
  - `{clientExternalId}` — the client's external ID as returned in the corp structure response (used for node matching within trees)
  - `{relatedParties}` — per-bucket counts and `partiesList` of matched Intapp parties
- Note: `{clientPartyExternalIdProvider}` was captured in Step 1 and is the authoritative provider name used throughout the playbook. The `providerType` field on the response is informational only — do not use it downstream.
- If no corporate trees found → **GO TO** Step END
- **GO TO** Step 3

**Exit check:** `{clientPartyId}` does not appear as an entry in any bucket's `partiesList` or `nonPartiesList`. The client is the subject of the screen, never a relationship to itself.

---

### Step 3: Review Corporate Structure Relationships

---

#### Step 3a: Parent

**Build `{parentTreeHierarchy}`**
- Use `relatedParties.Parent.displayHierarchy` as the tree structure.
- For each node in the hierarchy, enrich with `name`, `partyId`, and `type` by matching `externalId` against `partiesList` (type = `"party"`) and `nonPartiesList` (type = `"non-party"`).
- The client node is identified where `externalId` = `{clientExternalId}` — set type = `"client"`. The client appears in the hierarchy for spatial context only and is **never** counted in any bucket count, never added to any `searchTerms`, and never added to any grid row downstream.
- If a node's `externalId` cannot be matched to either list, skip it.

**IF** `relatedParties.Parent.total` = 0:
- DISPLAY: `"No Corporate Tree Parents available"`
- ACTION: GO TO Step 3b

**IF** `relatedParties.Parent.total` > 0 AND `relatedParties.Parent.nonParties` = 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{parentTreeHierarchy}`
    label: `Corporate tree parents are all included in screening`
    NOTE: The Appendix M tree render is mandatory and must appear in full before the next action. Do not replace it with a count summary.
- ACTION: GO TO Step 3b

**IF** `relatedParties.Parent.total` > 0 AND `relatedParties.Parent.nonParties` > 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{parentTreeHierarchy}`
    label: `Corporate tree parents`
    NOTE: The Appendix M tree render is mandatory and must appear in full before the next action. Do not replace it with a count summary.
  `Type ADD to add {N} parents to screening`
  `Type SKIP to continue`
- ACTION:
  - WAIT for user input
    - `ADD` → create all non-party parents in a single bulk `intappCommon:create_party` call (pass the full array as `body`) → GO TO Step 3b
    - `SKIP` → GO TO Step 3b

**Exit check:** `relatedParties.Parent.total` = `relatedParties.Parent.parties` + `relatedParties.Parent.nonParties`. The client `partyId` is not in `partiesList` and the client `externalId` is not in `nonPartiesList` for this bucket.

---

#### Step 3b: Subsidiary

**Build `{subsidiaryTreeHierarchy}`**
- Use `relatedParties.Subsidiary.displayHierarchy` as the tree structure.
- For each node in the hierarchy, enrich with `name`, `partyId`, and `type` by matching `externalId` against `partiesList` (type = `"party"`) and `nonPartiesList` (type = `"non-party"`).
- The client node is identified where `externalId` = `{clientExternalId}` — set type = `"client"`. The client appears in the hierarchy for spatial context only and is **never** counted in any bucket count, never added to any `searchTerms`, and never added to any grid row downstream.
- If a node's `externalId` cannot be matched to either list, skip it.

**IF** `relatedParties.Subsidiary.total` = 0:
- DISPLAY: `"No Corporate Tree Subsidiaries available"`
- ACTION: GO TO Step 3c

**IF** `relatedParties.Subsidiary.total` > 0 AND `relatedParties.Subsidiary.nonParties` = 0:
- DISPLAY: `"All Corporate Tree Subsidiaries included in screening"`
- ACTION: GO TO Step 3c

**IF** `relatedParties.Subsidiary.total` > 0 AND `relatedParties.Subsidiary.nonParties` > 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{subsidiaryTreeHierarchy}`
    label: `Corporate tree subsidiaries`
  `Type ADD to add {N} subsidiaries to screening`
  `Type SKIP to continue`
- ACTION:
  - WAIT for user input
    - `ADD` → create all non-party subsidiaries in a single bulk `intappCommon:create_party` call (pass the full array as `body`) → GO TO Step 3c
    - `SKIP` → GO TO Step 3c

**Exit check:** `relatedParties.Subsidiary.total` = `relatedParties.Subsidiary.parties` + `relatedParties.Subsidiary.nonParties`. The client `partyId` is not in `partiesList` and the client `externalId` is not in `nonPartiesList` for this bucket.

---

#### Step 3c: Management & Board

**Build `{boardTreeHierarchy}`**
- Root node: `{ name: {clientPartyName}, externalId: {clientExternalId}, partyId: {clientPartyId}, type: "client", children: [] }`
- For each entry in `relatedParties.Management/Board.partiesList`: create a node `{ name, externalId, partyId, type: "party", children: [] }` and attach as direct child of root
- For each entry in `relatedParties.Management/Board.nonPartiesList`: create a node `{ name, externalId, partyId: null, type: "non-party", children: [] }` and attach as direct child of root
- The client root appears for spatial context only and is **never** counted in the bucket count, never added to `searchTerms`, and never added to any grid row downstream.
- Board members are always flat — they have no children

**IF** `relatedParties.Management/Board.total` = 0:
- DISPLAY: `"No Management & Board members available"`
- ACTION: GO TO Step 3d

**IF** `relatedParties.Management/Board.total` > 0 AND `relatedParties.Management/Board.nonParties` = 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{boardTreeHierarchy}`
    label: `Management & Board`
    NOTE: The Appendix M tree render is mandatory and must appear in full before the next action. Do not replace it with a count summary.
- ACTION: GO TO Step 3d

**IF** `relatedParties.Management/Board.total` > 0 AND `relatedParties.Management/Board.nonParties` > 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{boardTreeHierarchy}`
    label: `Management & Board`
    NOTE: The Appendix M tree render is mandatory and must appear in full before the next action. Do not replace it with a count summary.
  `Type ADD to add {N} board members to screening`
  `Type SKIP to continue`
- ACTION:
  - WAIT for user input
    - `ADD` → create all non-party board members in a single bulk `intappCommon:create_party` call (pass the full array as `body`) → GO TO Step 3d
    - `SKIP` → GO TO Step 3d

**Exit check:** `relatedParties.Management/Board.total` = `relatedParties.Management/Board.parties` + `relatedParties.Management/Board.nonParties`. The client `partyId` is not in `partiesList` and the client `externalId` is not in `nonPartiesList` for this bucket.

---

#### Step 3d: UBO

**Build `{uboTreeHierarchy}`**
- Use `relatedParties.UBO.displayHierarchy` as the tree structure.
- For each node in the hierarchy, enrich with `name`, `partyId`, and `type` by matching `externalId` against `partiesList` (type = `"party"`) and `nonPartiesList` (type = `"non-party"`).
- The client node is identified where `externalId` = `{clientExternalId}` — set type = `"client"`. The client appears in the hierarchy for spatial context only and is **never** counted in any bucket count, never added to any `searchTerms`, and never added to any grid row downstream.
- If a node's `externalId` cannot be matched to either list, skip it.

**IF** `relatedParties.UBO.total` = 0:
- DISPLAY: `"No Beneficial Owners available"`
- ACTION: GO TO Step 3e

**IF** `relatedParties.UBO.total` > 0 AND `relatedParties.UBO.nonParties` = 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{uboTreeHierarchy}`
    label: `Beneficial Owners`
    NOTE: The Appendix M tree render is mandatory and must appear in full before the next action. Do not replace it with a count summary.
- ACTION: GO TO Step 3e

**IF** `relatedParties.UBO.total` > 0 AND `relatedParties.UBO.nonParties` > 0:
- DISPLAY:
  Render Appendix M — Hierarchy display
    treeHierarchy: `{uboTreeHierarchy}`
    label: `Beneficial Owners`
    NOTE: The Appendix M tree render is mandatory and must appear in full before the next action. Do not replace it with a count summary.
  `Type ADD to add {N} beneficial owners to screening`
  `Type SKIP to continue`
- ACTION:
  - WAIT for user input
    - `ADD` → create all non-party beneficial owners in a single bulk `intappCommon:create_party` call (pass the full array as `body`) → GO TO Step 3e
    - `SKIP` → GO TO Step 3e

**Exit check:** `relatedParties.UBO.total` = `relatedParties.UBO.parties` + `relatedParties.UBO.nonParties`. The client `partyId` is not in `partiesList` and the client `externalId` is not in `nonPartiesList` for this bucket.

---

#### Step 3e: Proceed

| Field | Value |
|---|---|
| **Type** | Prompt |
| **Status Message** | None. |
| **Notification Rule** | Only show display and prompt. Do not show any commentary or narration. |

- If any parties were created across Steps 3a–3d → re-call `IntappJungle:get_corp_structure_party_relationships` with `include_non_parties` = `false`, update `{relatedParties}`
- DISPLAY:

*Type `RUN` to proceed to Moody's GRID screening.*

WAIT for user input:
- `RUN` → GO TO Step 4

---


### Step 4: Merge Relationships

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | *(none — processing step)* |
| **Notification Rule** |  Do not show commentary or narration. |

- Build the table from `{relatedParties}`: iterate each bucket (GUO, Parent, Subsidiary, Sibling, Management/Board, Shareholder, UBO), collect all entries from each `partiesList`, group by `partyId`, and merge relationship labels for any party that appears in multiple buckets.
  - See **Appendix F** for `{relatedParties}` full schema.

**GO TO** Step 5

**Exit check:** `{clientPartyId}` does not appear as an entry in `{relatedPartiesTable[]}`. No entry has the relationship label `"Client"`.

---

Step 5: Create Request and Conflicts Screen

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 🛡️ Creating request and Moody's GRID screen... |
| **Notification Rule** | Status message only. Do not show commentary or narration. |

- Create a request with `IntappApp:create_request`
  - See Appendix D for API body/XML details
  - Set `{reqID}` from response
- Create a conflicts search linked to the request with `IntappApp:create_conflict_search`
  - See Appendix E for API body details
  - Link to request with `{reqID}`
  - Set `{conID}` from response
- Run conflicts search with `IntappApp:run_conflict_search` using `{conID}`

**Exit check:** `{reqID}` and `{conID}` are both non-null.

---

### Step 6: Fetch Moody's GRID Hits

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 🛡️ Screening parties on Moody's GRID... |
| **Notification Rule** | Zero output permitted during this step except the status message above. |

Step 6 has four phases. All operate silently — no output between them.

#### Phase A — Fetch conflicts hits

- Call `IntappApp:get_conflicts_search` with `{conID}` and `properties=resultsPlusMoodysGRID`
- From the response, extract all entries in `results[]` where `type` = `"Moody's GRID"`
- For each hit, capture: `resultId`, `resolution`, `searchTerm`, and `moodysGRID.eventCategories[]`
- If `moodysGRIDSkipped` is present and > 0, note that enrichment was capped at 25 hits

#### Phase B — Build `{moodysHitEvents}[]` skeleton

For each search party, initialise one entry in `{moodysHitEvents}[]` with:

- `partyId`, `partyName`, `relationship` from `{relatedParties}`
- `hits[]` — one object per Moody's GRID hit containing:
  - `gridId` — the `resultId`
  - `resolution` = `null` — populated in Phase D from `{moodysResolutions}`
  - `eventCodes[]` — taken directly from `moodysGRID.eventCategories[]`
  - `weight` = `"unknownWeight"` — placeholder; recategorized in Step 7a
  - `auto` = `null` — set in Step 7a
  - `confirmed` = `null` — set in Step 7a
  - `matchScore` = `null` — set in Step 7a
- `riskScore`, `riskLevel`, `summary` — left empty; populated in Steps 7b and 7c

#### Phase C — Sync conflicts-search resolutions to external data

Build a single bulk write of all conflicts-search resolutions seen in Phase A, then call `IntappJungle:post_external_data_bulk` once.

- For every hit captured in Phase A where the conflicts-search `resolution` is **exactly** `{conResMatch}` or `{conResNoMatch}`:
  - Build one external data record per **Appendix N** schema, with `remoteId` = `"{partyId}_{gridId}"`
- Any other `resolution` value — empty string, `null`, `"Cleared"`, or any string other than the two allowed values — is **skipped**. Do not write it to external data.
- If at least one record was built, call `IntappJungle:post_external_data_bulk` once with all records in a single `items[]` array.
- This call is upsert: existing records (matched by `remoteId` + `externalSourceType`) are overwritten with the latest conflicts-search resolution. New records are created.
- If no hits had an allowed resolution, skip the call entirely.

> The conflicts-search `resolution` field is only used as input to this sync. Once written to external data, all downstream reads use external data as the source of truth.

#### Phase D — Read external data and populate hit resolutions

- Verify `IntappJungle:list_external_data` is loaded. If not, call `tool_search` to load it. If still unavailable after one retry, halt — do not proceed to Step 7 with an empty `{moodysResolutions}`.
- Initialise `{moodysResolutions}` as an empty map.
- For each unique `partyId` in `{moodysHitEvents}[]`, call `IntappJungle:list_external_data` with:
  - `search_externalSourceType` = `{moodyResolutionSourceType}`
  - `search_name` = `{partyId}`
  - `search_rowsToTake` = `200`
- For each record returned, set `{moodysResolutions}[record.remoteId]` = `record.moodyAgentExternal._moodyAgentHitResolution`.
- For each hit in every `{moodysHitEvents}[].hits[]`:
  - Look up `{moodysResolutions}["{partyId}_{gridId}"]`
  - If found, set `hit.resolution` to that value
  - If not found, leave `hit.resolution` = `null`

**GO TO** Step 7

**Exit check:** `IntappJungle:list_external_data` was called for every unique `partyId` in `{moodysHitEvents}[]`.

---

### Step 7: Analyze Moody's GRID Hits

#### Step 7a: Categorize Weight

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 🛡️ Analyzing Moody's GRID screening results for matches... |
| **Notification Rule** | Zero output permitted during this step except the status message above. |

- Use the already-fetched `{moodysHitEvents}[]` data from Step 6 — no additional API calls required. `hits[].resolution` is sourced from external data via Step 6 Phase D.
- For each screened party, classify each hit individually:

  - If `resolution` = `{conResMatch}` or `{conResNoMatch}` → classify immediately per the table below
  - If `resolution` is null / empty / any other value → determine `isLikely` and `matchScore` via name matching:

    Normalise both the hit's `moodysGRID.name` and the hit's `searchTerm` before comparing: lowercase, strip punctuation, collapse whitespace, remove legal suffixes (LLC, OOO, AO, JSC, GmbH, Inc, Ltd, Corp, Co, and common equivalents).

    Compute `matchScore` as the token overlap ratio: shared tokens ÷ total unique tokens across both normalised strings. Round to 2 decimal places. Store `matchScore` on the hit object.

    Set `isLikely = true` if ANY condition is met:
    - Normalised GRID name exactly matches normalised search term
    - Normalised GRID name contains the full normalised search term as a substring
    - Normalised search term contains the full normalised GRID name as a substring
    - `matchScore` ≥ `{fullWeightAutoMatchMin}`

    Otherwise `isLikely = false`. Evaluate every unresolved hit individually — do not use total hit count to bypass this logic.

- Classify each hit and set `weight`, `auto`, `confirmed`, and `matchScore` fields:

| Resolution | matchScore | weight | auto | confirmed |
|---|---|---|---|---|
| `{conResMatch}` | null | `fullWeight` | false | true |
| `{conResNoMatch}` | null | `noWeight` | false | true |
| null / other | ≥ `{fullWeightAutoMatchMin}` | `fullWeight` | true | false |
| null / other | ≥ `{partialWeightAutoMatchMin}` and < `{fullWeightAutoMatchMin}` | `partialWeight` | true | false |
| null / other | < `{partialWeightAutoMatchMin}` | `noWeight` | true | false |

- **Auto-classification rule for `noWeight`:** Hits classified as `noWeight` fall into two subtypes:
  - **Manually confirmed** (`resolution` = `{conResNoMatch}`): `confirmed` = `true`, `auto` = `false`
  - **Auto-classified** (matchScore < `{partialWeightAutoMatchMin}`): `auto` = `true`, `confirmed` = `false`
  In both cases, event codes are excluded from `summary` arrays and never contribute to risk score.

- For each party, update its entry in `{moodysHitEvents}[]`:
  - Set `weight`, `auto`, `confirmed`, and `matchScore` on each hit object
  - Derive `summary.fullWeightEvents` — deduplicated event codes across all `fullWeight` hits
  - Derive `summary.partialWeightEvents` — deduplicated event codes across all `partialWeight` hits

Total hits per party must equal: fullWeight hits (auto + confirmed) + partialWeight hits (auto only) + noWeight hits (auto + confirmed)

**Exit check:** Every hit has `weight ∈ {fullWeight, partialWeight, noWeight}` (no `unknownWeight` remaining) and non-null mutually-exclusive `auto` / `confirmed` (exactly one is `true` per hit).

---

#### Step 7b: Party Risk Score

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 📊 Calculating client and corporate structure relationship risk scores... |
| **Notification Rule** | Zero output permitted during this step except the status message above. |

For each entry in `{moodysHitEvents}[]`, apply the scoring rules defined in **Appendix B** in full, in order. Set `riskScore` and `riskLevel` on each entry.

**Exit check:** Every entry in `{moodysHitEvents}[]` has non-null `riskScore` and `riskLevel`.

---

#### Step 7c: Overall Risk Score

| Field | Value |
|---|---|
| **Type** | Processing |
| **Status Message** | 📊 Calculating overall risk score... |
| **Notification Rule** | Zero output permitted during this step except the status message above. |

Apply the overall scoring rules defined in **Appendix C** using `{moodysHitEvents}[]`. Set `{overallRiskScore}` and `{overallRiskLevel}`.

**Exit check:** `{overallRiskScore}` and `{overallRiskLevel}` are both non-null.

---

### Step 8: Display Moody's GRID Screen Results and Risk

| Field | Value |
|---|---|
| **Type** | Display |
| **Status Message** | *(none — display output is the communication)* |
| **Notification Rule** | Return ONLY the display specified below. Do not add commentary, preamble, or narration of any kind. |

#### Column Assignment (Silent)

For every party in `{moodysHitEvents}[]`, silently assign each hit to a display column based solely on `weight`:
- `fullWeight` → **Strong Match**
- `partialWeight` → **Moderate Match**
- `noWeight` → **Weak Match**

For errored hits where `moodysGRID` is null due to `moodysError`: assign weight based on matchScore as normal and note `[enrichment error — event codes unavailable]`.

> ⚠️ Column placement is determined solely by `weight`. Event code severity (including Critical codes such as WLT, SNX, PEP) affects `riskScore` only — it never promotes a hit to a higher-weight column. A `partialWeight` hit with WLT goes in **Moderate Match**, always.

Verify that the total assigned hits per party equals `hits[].length` for that party before rendering. If counts do not match, correct before proceeding.

**Pre-render check:** No party row in the table has `relationship` = `"Client"` unless its `partyId` = `{clientPartyId}`. If false, fix upstream in Step 3 / Step 4 before rendering.

#### Render

**Table structure:** Two header rows. The first row has empty cells under Review, Name (ID), Risk Score, and Relationship, and a single spanning header **Risk Events** over the three match columns. The second row contains all seven column labels.

| | | | | Risk Events | | |
|---|---|---|---|---|---|---|
| **Review** | **Name (ID)** | **Risk Score** | **Relationship** | **Strong Match** | **Moderate Match** | **Weak Match** |

**Column definitions:**
- **Review** — a single letter (A, B, C, …) assigned to every party in the table in render order. Used by the Step 8 prompt to select a party to review.
- **Name (ID)** — party name and ID. Prefix client row with 🔷, related party rows with 🔹
- **Risk Score** — `{riskLevel}` (`{riskScore}`) from Step 7b. If riskScore = 0, display `-` with no score value or icon.
- **Relationship** — `Client` for the client row; relationship label for all others
- **Strong Match** — all hits where `weight = fullWeight`
- **Moderate Match** — all hits where `weight = partialWeight`
- **Weak Match** — all hits where `weight = noWeight`

Per cell:
1. **Bold summary line** — deduplicated event codes across all hits assigned to that column only
2. One blank line
3. Indented italic hit counts, with no blank line between them but one blank line separating them from the summary line above:

   **{event codes}**

   &nbsp;&nbsp;*{n} Auto-Matched Hits*
   &nbsp;&nbsp;*{n} Matched Hits*

   Omit a line if its count is 0.
4. Render `—` when no hits of that weight exist for that party

**Sort order:** The client row is **always first** in the table regardless of hit count or score, with `relationship = "Client"`. Render it even if the client has zero hits — show `—` in all three Risk Event columns and `-` in Risk Score in that case. Related parties follow, sorted by `riskScore` descending. Letters are assigned in this rendered order: client = A, next party = B, and so on.

> [Request Name (ID)](https://{yourhost}/app/request.aspx?rid={reqID})
> ## Overall Risk Score: {overallRiskLevel} — {overallRiskScore}

| | | | | Risk Events | | |
|---|---|---|---|---|---|---|
| **Review** | **Name (ID)** | **Risk Score** | **Relationship** | **Strong Match** | **Moderate Match** | **Weak Match** |
| **A** | 🔷 `{clientPartyName}` (`{clientPartyId}`) | {riskLevel} {riskScore} | Client | **{fullWeightSummary}**<br><br>&nbsp;&nbsp;*{n} Moody's profiles* | **{partialWeightSummary}**<br><br>&nbsp;&nbsp;*{n} Moody's profiles* | **{noWeightSummary}**<br><br>&nbsp;&nbsp;*{n} Moody's profiles* |
| **B** | 🔹 `{partyName}` (`{partyId}`) | {riskLevel} {riskScore} | {relationship} | — | **{partialWeightSummary}**<br><br>&nbsp;&nbsp;*{n} Moody's profiles* | — |

#### Prompt

After the table, display:

*Enter a letter (A, B, C, …) to review and manually resolve that party's hits.*
*Enter `COMPLETE` to finalise AML/KYC.*

WAIT for user input:

- If user enters a letter → resolve it to the corresponding `partyId` from the table → **GO TO** Step 9 with that `partyId`
- If user enters `COMPLETE` → **GO TO** Step COMPLETE

---

### Step 9: Review and Resolve Party Hits

| Field | Value |
|---|---|
| **Type** | Display → User Prompt |
| **Status Message** | *(none — display output is the communication)* |
| **Notification Rule** | Return ONLY the Appendix J render and the prompt below. No commentary, no preamble, no narration of any kind. |

Entered from Step 8 with a single `{partyId}` selected by the user. Step 9 only displays the selected party's hits and collects resolution updates — it does **not** mutate any playbook variables. The external data write happens first, then control returns to Step 6 which re-reads external data and rebuilds `{moodysHitEvents}[]` from scratch.

---

#### Display

Render **Appendix J** for the selected `{partyId}` with one change: add a leftmost **Hit** column containing a letter (A, B, C, …) assigned to every hit in render order.

**Hit letter assignment rule:** Hits are ordered Strong Match first (all `fullWeight`), then Moderate Match (`partialWeight`), then Weak Match (`noWeight`). Within each group, sort by `gridId` ascending for stability. Letter A goes to the first hit, B to the second, and so on.

Augment each row with a **Current Resolution** column showing the hit's existing resolution if one is already stored in external data (i.e. `hits[].resolution` is non-null). Display `—` if no prior resolution exists.

**Table structure:**

**{partyName} (`{partyId}`) — Score: {riskScore} {riskLevel}**

| Hit | Moody's GRID Hit | Name Match Score | Match Type | Current Resolution | Risk Events (Risk Score Contribution) | Total Contribution to Risk Score |
|---|---|---|---|---|---|---|
| **A** | {gridName} (`{gridId}`) | {matchScore}% | Strong / Moderate / Weak | {resolution or —} | • {CODE} — {Label} +{value}<br>• {CODE} — {Label} +{value} | +{total} |
| **B** | {gridName} (`{gridId}`) | {matchScore}% | Strong / Moderate / Weak | {resolution or —} | • {CODE} — {Label} +{value} | +{total} |

**Total: {riskScore} {riskLevel}**

---

#### Prompt

After the table, display:

*Enter resolutions using hit letters:*
- *`Match: A, C` — mark those hits as `{conResMatch}` and recalculate risk score*
- *`No Match: B, D` — mark those hits as `{conResNoMatch}` and recalculate risk score*
- *Both can be entered on separate lines in the same response.*
- *`Go Back` — return to the risk score display without changes*

---

#### Action — user entered `Go Back`

- **GO TO** Step 8 (re-render the existing results table; no recalc, no external data write, no Step 6 loop)

#### Action — user entered any combination of `Match:` / `No Match:` lines

| Field | Value |
|---|---|
| **Status Message** | 💾 Saving resolutions to external data... |

- Parse both lines into two letter lists. Each letter must map back to a hit in the rendered table for `{partyId}`. Reject any letter not present in the table.
- Build one external data record per named hit per **Appendix N** schema:
  - `remoteId` = `"{partyId}_{gridId}"`
  - `name` = `description` = `{partyId}`
  - `moodyAgentExternal._moodyAgentHitPartyId` = `{partyId}`
  - `moodyAgentExternal._moodyAgentHitEntityId` = `{gridId}`
  - `moodyAgentExternal._moodyAgentHitResolution` = `{conResMatch}` for every letter listed under `Match:`, `{conResNoMatch}` for every letter listed under `No Match:`
- Call `IntappJungle:post_external_data_bulk` **once** with all records combined into a single `items[]` array. This is upsert — existing records for the same `remoteId` are overwritten.
- Do **not** update `{moodysHitEvents}` or any other playbook variable. Those are reset by Step 6.
- **GO TO** Step 6

> Step 6 Phase D re-reads external data and repopulates every `hits[].resolution`. Step 7a re-classifies. Steps 7b, 7c, 8 re-render from scratch. The user lands back on the updated Step 8 table with fresh scores reflecting the new resolutions.

---

### Step COMPLETE

| Field | Value |
|---|---|
| **Type** | Processing → Display |
| **Status Message** | 💾 Updating party profiles with AML/KYC results... |
| **Notification Rule** | Zero output permitted except the status message and the final completion table. No commentary between operations, no transition phrases, no confirmation of tool calls. |

Build a single `updates[]` array — one entry per unique `partyId` in `{moodysHitEvents}[]` (including the client) — then make **one call** to `IntappJungle:append_party_table_rows`. The tool fetches each party, appends the supplied `<Row>` XML to the named table field, and PATCHes only the changed fields. All party updates run in parallel internally — no manual fetch, parse, or merge required, and unrelated party fields are untouched.

---

#### Build the `updates[]` array

For each unique `partyId` in `{moodysHitEvents}[]`:

- Build a `_moodyAgentRuns` `<Row>` XML string using the field mappings below
- For the **client party only**, additionally:
  - Build a `_moodyAgentRunsClient` `<Row>` XML string
  - Include `_moodyAgentNext` in `fieldUpdates`

#### Field Mappings — `_moodyAgentRuns` Row (all parties)

| Cell Key | Value |
|---|---|
| `dateComplete` | Today's date/time, ISO 8601 (e.g. `2026-04-24T18:50:44Z`) |
| `reqID` | `{reqID}` |
| `riskScore` | `"{moodysHitEvents}[i].riskLevel ({moodysHitEvents}[i].riskScore)"` — use `"None"` if `riskScore` = 0 or null |
| `riskEvents` | Comma-separated deduplicated codes across `summary.fullWeightEvents` + `summary.partialWeightEvents`, or `"None"` |
| `confirmedRiskEvents` | Comma-separated `eventCodes` across hits where `hits[].confirmed = true`, deduplicated, or `"None"` |
| `confirmedRiskId` | Comma-separated `gridIds` where `hits[].confirmed = true`, or empty |
| `strongRiskEvents` | Comma-separated `eventCodes` across hits where `hits[].weight = 'fullWeight'`, deduplicated, or `"None"` |
| `strongRiskId` | Comma-separated `gridIds` where `hits[].weight = 'fullWeight'`, or empty |
| `modRiskEvents` | Comma-separated `eventCodes` across hits where `hits[].weight = 'partialWeight'`, deduplicated, or `"None"` |
| `modRiskId` | Comma-separated `gridIds` where `hits[].weight = 'partialWeight'`, or empty |
| `ranForClient` | `"{clientPartyName} ({clientPartyId})"` — same value for every party in this loop |
| `relationship` | Relationship label(s) for this party, comma-separated. `"Client"` if client party |

#### Field Mappings — `_moodyAgentRunsClient` Row (client party only)

| Cell Key | Value |
|---|---|
| `dateComplete` | Today's date/time, ISO 8601 (e.g. `2026-04-24T18:50:44Z`) |
| `reqID` | `{reqID}` |
| `riskScoreOverall` | `"{overallRiskLevel} ({overallRiskScore})"` — use `"None"` if `overallRiskScore` = 0 or null |

---

#### Row XML Format

Each `<Row>` is a string of the form:

```xml
<Row>
  <Cell><Key>columnName</Key><Value>value</Value></Cell>
  <Cell><Key>columnName</Key><Value>value</Value></Cell>
</Row>
```

XML-escape any values containing `&`, `<`, `>`, `"`, or `'`.

See **Appendix K** for full XML cell rules.

---

#### Single Call

```json
{
  "updates": [
    {
      "partyId": "{clientPartyId}",
      "tableAppends": {
        "_moodyAgentRuns": "<Row>...client _moodyAgentRuns row...</Row>",
        "_moodyAgentRunsClient": "<Row>...client _moodyAgentRunsClient row...</Row>"
      },
      "fieldUpdates": {
        "_moodyAgentNext": "<6 months from today at midnight UTC, e.g. 2026-10-24T00:00:00Z>"
      }
    },
    {
      "partyId": "{relatedParty1.partyId}",
      "tableAppends": {
        "_moodyAgentRuns": "<Row>...related party _moodyAgentRuns row...</Row>"
      }
    },
    {
      "partyId": "{relatedParty2.partyId}",
      "tableAppends": {
        "_moodyAgentRuns": "<Row>...related party _moodyAgentRuns row...</Row>"
      }
    }
  ]
}
```

**Pre-call check:** `updates[]` contains exactly one entry per unique `partyId` in `{moodysHitEvents}[]`. The client entry includes both `_moodyAgentRuns` and `_moodyAgentRunsClient` table appends plus the `_moodyAgentNext` field update. All other entries include only `_moodyAgentRuns`.

---

#### Response Handling

The tool returns `{ updated, failed, errors? }`. Use this to populate the per-party status in the final display:

- `updated` → ✅ for those `partyId`s
- `failed` / `errors[]` → ❌ with the corresponding error message

Do not abort on a partial failure. Render every party in the final table with its actual outcome.

---

#### Display

After the call returns, display:

---

**✅ AML/KYC Complete — `{clientPartyName}` (`{clientPartyId}`)**

| Party | Status |
|---|---|
| 🔷 `{clientPartyName}` (`{clientPartyId}`) | ✅ Saved Risk Score - Next Review Scheduled in 6 months `{nextReviewDate}` / ❌ `{error}` |
| 🔹 `{partyName}` (`{partyId}`) | ✅ Saved Risk Score / ❌ `{error}` |

---

Playbook completed.
---

### Step END

| Field | Value |
|---|---|
| **Type** | Display |
| **Status Message** | *(none)* |
| **Notification Rule** | Zero output permitted except the defined display output. |

Playbook ended but not completed.

---

## Appendices

### Appendix A: Display Preferences
- **Risk Scores:** 🔴 High, 🟠 Medium, 🔘 Low, None
  - None if risk score is 0
- **Client Indicator:** 🔷 next to client name
- **Related Party Indicator:** 🔹 next to party name
- **New Party Indicator:** 🆕 before 🔹 when `isNew` = `true`

---

### Appendix B: Party-Level Risk Score Calculation

Evaluate in order, stop at first match:

1. If any Critical code appears in `summary.fullWeightEvents` → score = 100, 🔴 High
2. Otherwise, sum all unique event code values across `summary.fullWeightEvents` (100%) and `summary.partialWeightEvents` (50%, rounded down)

| Tier | Codes | Full Weight | Partial Weight |
|---|---|---|---|
| Critical | BLK, DEN, FOF, FOS, IRC, MLA, ORG, PEP, SNX, TER, WLT | 100 → instant 🔴 High | +10 |
| Valuable | BRB, BUS, CFT, CYB, DTF, ENV, FRD, FUG, GAM, HUM, IMP, KID, LMD, LNS, MOR, MSB, MUR, OBS, PRJ, REG, RES, SEC, SEX, SMG, SPY, TAX, TFT, TRF, VCY | +5 | +2 |
| Investigative | ARS, AST, BUR, CON, DPS, FOR, IGN, PSP, ROB | +3 | +1 |
| Probative | ABU, CPR, IPR, MIS, NSC | +1 | +0 |

| Risk Level | Score |
|---|---|
| 🔴 High | ≥ 67 |
| 🟠 Medium | 40 – 66 |
| 🔘 Low | 1 – 39 |
| None | 0 |

`noWeight` event codes do not contribute to score.

---

### Appendix C: Overall Risk Score Calculation

Start with the client party's `riskScore` as the baseline. Then for each related party, add a contribution based on their `riskLevel`:

| Related Party Risk Level | Points Added |
|---|---|
| 🔴 High | +10 |
| 🟠 Medium | +5 |
| 🔘 Low | +1 |
| None | +0 |

Overall score = `client riskScore` + sum of all related party contributions, capped at 100.

| Overall Risk | Score |
|---|---|
| 🔴 High | ≥ 67 |
| 🟠 Medium | 34 – 66 |
| 🔘 Low | 1 – 33 |
| None | 0 |

---

### Appendix D: `IntappApp:create_request` API Details

#### API Body
```json
{
  "workflowDefinitionName": "AML/KYC - Moody's GRID",
  "name": "AML KYC - Moody's GRID for {clientPartyName}",
  "answers": [
    {
      "questionName": "{reqRefParty}",
      "partyLookupAnswer": {
        "partyId": "{clientPartyId}",
        "entityName": "{clientPartyName}",
        "companyDetails": {
          "companyName": "{clientPartyName}",
          "primaryExternalDataProvider": "{clientPartyExternalIdProvider}",
          "externalProviderIds": { "{clientPartyExternalIdProvider}": "{clientPartyExternalId}" }
        }
      }
    },
    {
      "questionName": "{reqRefGrid}",
      "dataTableSimpleXmlAnswer": "See Grid Question XML below"
    }
  ]
}
```

#### Grid Question XML

*(one `<GridRow>` per entry across all `partiesList[]` buckets in `{relatedParties}`, deduplicated by `partyId`, with `<RowId>` incrementing from 0. The client `partyId` is **never** included in this grid — it is the subject of the request, not a relationship.)*

```xml
<Grid>
  <GridRow>
    <RowId>0</RowId>
    <Question>
      <Name>{reqRefGridName}</Name>
      <Value>
        <PartyId>{relatedPartiesTable[0].partyId}</PartyId>
        <EntityName>{relatedPartiesTable[0].partyName}</EntityName>
      </Value>
    </Question>
    <Question>
      <Name>{reqRefGridRelationship}</Name>
      <Value>{relatedPartiesTable[0].relationships[0]}</Value>
    </Question>
  </GridRow>
  <GridRow>
    <RowId>1</RowId>
    <Question>
      <Name>{reqRefGridName}</Name>
      <Value>
        <PartyId>{relatedPartiesTable[1].partyId}</PartyId>
        <EntityName>{relatedPartiesTable[1].partyName}</EntityName>
      </Value>
    </Question>
    <Question>
      <Name>{reqRefGridRelationship}</Name>
      <Value>{relatedPartiesTable[1].relationships[0]}</Value>
    </Question>
  </GridRow>
</Grid>
```

> Note: This `<Grid><GridRow><Question>` schema is used **only** for the `dataTableSimpleXmlAnswer` field on `create_request`. It is unrelated to the `<Row><Cell>` schema used by `IntappJungle:append_party_table_rows` in Step COMPLETE — see Appendix K for that format.

---

### Appendix E: `IntappApp:create_conflict_search` API Details

#### API Body

*(one entry per unique party across all `partiesList[]` buckets in `{relatedParties}`, plus the client as first entry)*
- Client entry: `{ "affiliationType": "{clientPartyConAff}", "name": "{clientPartyName}", "partyId": "{clientPartyId}" }`
- Related party entries: `{ "affiliationType": "{relPartyConAff}", "name": "{partyName}", "partyId": "{partyId}" }` — deduplicated by `partyId`. The client `partyId` must **not** appear as a related party entry — it is only included once via the client entry above.

```json
{
  "name": "AML KYC - Moody's GRID Screen for {clientPartyName} for Request {reqId}",
  "requestId": "{reqID}",
  "searchTerms": [
    { "affiliationType": "{clientPartyConAff}", "name": "{clientPartyName}", "partyId": "{clientPartyId}" },
    { "affiliationType": "{relPartyConAff}", "name": "{relatedPartiesTable[0].partyName}", "partyId": "{relatedPartiesTable[0].partyId}" },
    { "affiliationType": "{relPartyConAff}", "name": "{relatedPartiesTable[1].partyName}", "partyId": "{relatedPartiesTable[1].partyId}" }
  ]
}
```

---

### Appendix F: Multi-Source Relationship Rule

Relationship buckets are **non-exclusive** — a single entity can and should be recorded in multiple buckets simultaneously. Evaluate all bucket conditions for every entity regardless of source; do not stop at the first match.

**Example:** The GUO is always also a Parent. If it additionally appears in the shareholders list, it holds three relationships at once: `["GUO", "Parent", "Shareholder"]`. It appears as a single row in `{relatedPartiesTable[]}` with all three labels — never as duplicate rows, and never counted more than once toward any bucket's `total`.

**Client exclusion rule:** The client `partyId` is the subject of the screen, not a relationship to itself. It must never appear in any bucket's `partiesList`, `nonPartiesList`, or in `{relatedPartiesTable[]}`. If the underlying corp structure response includes the client as a node in any tree (which is normal for hierarchy context), the merge logic must filter it out before populating `{relatedPartiesTable[]}`.

#### `IntappJungle:get_corp_structure_party_relationships` tool relatedParties Schema

```json
{
  "clientExternalId": "",
  "providerType": "",
  "relatedParties": {
    "GUO":                { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Parent":             { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Subsidiary":         { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Sibling":            { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Management/Board":   { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "Shareholder":        { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] },
    "UBO":                { "total": 0, "parties": 0, "partiesList": [], "nonPartiesList": [] }
  }
}
```

- `partiesList[]` entries: `{ "partyId": "", "partyName": "", "externalId": "" }`
- `nonPartiesList[]` entries: `{ "externalId": "", "name": "" }`

---

### Appendix G: `get_conflicts_search` Response Guide (`properties=resultsPlusMoodysGRID`)

#### Key Fields

| Field | Description |
|---|---|
| `searchId` | The conflict search ID |
| `moodysGRIDHitCount` | Number of Moody's GRID hits enriched |
| `moodysGRIDSkipped` | Hits skipped due to 25-hit limit (only present if > 0) |
| `results[]` | All conflict hits — non-Moody's hits pass through unchanged |
| `results[].moodysGRID` | Present on Moody's GRID hits only |
| `results[].moodysGRID.name` | Entity name from Moody's GRID |
| `results[].moodysGRID.eventCategories[]` | Unique category codes from all events (e.g. `SNX`, `PEP`) |
| `results[].moodysError` | Present if enrichment failed for that hit |
| `results[].resolution` | Conflict resolution status — used in Step 6 Phase C only (sync to external data). All downstream reads use external data. |

#### Sample Response

```json
{
  "searchId": "15221",
  "moodysGRIDHitCount": 1,
  "results": [
    {
      "name": "Gazprom Teploenergo",
      "resultId": "281209988",
      "type": "Moody's GRID",
      "resolution": "",
      "status": "Active",
      "searchTerm": "GAZPROM TEPLOENERGO, AO",
      "moodysGRID": {
        "name": "AO Gazprom Teploenergo",
        "eventCategories": ["SNX", "REG"]
      }
    },
    {
      "name": "Some Other Party",
      "resultId": "99001",
      "type": "Internal",
      "resolution": "Cleared"
    }
  ]
}
```

#### Notes
- Non-Moody's hits are returned as-is with no `moodysGRID` field
- If enrichment fails for a hit, `moodysGRID` will be `null` and `moodysError` will describe the failure
- Enrichment is capped at **25 Moody's hits** per call, processed in batches of 5

---

### Appendix H: Risk Label & Event Code Reference

| Code | Label |
|---|---|
| ABU | Abuse (Domestic, Elder, Child) |
| ARS | Arson |
| AST | Assault, Battery |
| BLK | Firm Specific Blocked List |
| BRB | Bribery, Graft, Kickbacks, Political Corruption |
| BUR | Burglary |
| BUS | Business Crimes (Antitrust, Bankruptcy, Price Fixing) |
| CFT | Counterfeiting, Forgery |
| CON | Conspiracy (no specific crime named) |
| CPR | Copyright Infringement (Intellectual Property, Electronic Piracy) |
| CYB | Computer Related, Cyber Crime |
| DEN | Denied Entity |
| DPS | Possession of Drugs or Drug Paraphernalia |
| DTF | Trafficking or Distribution of Drug |
| ENV | Environmental Crimes (Poaching, Illegal Logging, Animal Cruelty) |
| FOF | Former OFAC List |
| FOR | Forfeiture |
| FOS | Former Sanctions |
| FRD | Fraud, Scams, Swindles |
| FUG | Fugitive, Escape |
| GAM | Illegal Gambling |
| HUM | Human Rights, Genocide, War Crimes |
| IGN | Possession or Sale of Guns, Weapons and Explosives |
| IMP | Identity Theft, Impersonation |
| IPR | Illegal Prostitution |
| IRC | Iran Connect |
| KID | Kidnapping, Abduction, Held Against Will |
| LMD | Legal Marijuana Dispensaries |
| LNS | Loan Sharking, Usury, Predatory Lending |
| MIS | Misconduct |
| MLA | Money Laundering |
| MOR | Mortgage Related |
| MSB | Money Services Business |
| MUR | Murder, Manslaughter (Committed, Planned or Attempted) |
| NSC | Nonspecific Crime |
| OBS | Obscenity Related, Child Pornography |
| ORG | Organized Crime, Criminal Association, Racketeering |
| PEP | Politically Exposed Person |
| PRJ | Perjury, Obstruction of Justice, False Filings, False Statements |
| PSP | Possession of Stolen Property |
| REG | Regulatory Action |
| RES | Real Estate Actions |
| ROB | Robbery (Stealing by Threat, Use of Force) |
| SEC | SEC Violations (Insider Trading, Securities Fraud) |
| SEX | Sex Offenses (Rape, Sodomy, Sexual Abuse, Pedophilia) |
| SMG | Smuggling (Does not include Drugs, Money, People or Guns) |
| SNX | Sanctions Connect |
| SPY | Spying (Treason, Espionage) |
| TAX | Tax Related Offenses |
| TER | Terrorist Related |
| TFT | Theft (Larceny, Misappropriation, Embezzlement, Extortion) |
| TRF | People Trafficking, Organ Trafficking |
| VCY | Virtual Currency |
| WLT | Watchlisted Entities |

---

### Appendix I: Party Risk Score Reasoning

*Displayed only when explicitly requested. Provides a per-hit breakdown of every event code contribution that produced a party's final score.*

---

### Appendix J: Risk Score Reasoning Table

**{partyName} (`{partyId}`) — Score: {riskScore} {riskLevel}**

| Moody's GRID Hit | Name Match Score | Match Type | Risk Events (Risk Score Contribution) | Total Contribution to Risk Score |
|---|---|---|---|---|
| {gridName} (`{resultId}`) | {matchScore}% | Strong / Moderate / Weak | • {CODE} — {Label} +{value}<br>• {CODE} — {Label} +{value} | +{total} |
| {gridName} (`{resultId}`) | {matchScore}% | Strong / Moderate / Weak | • {CODE} — {Label} +{value}<br>• {CODE} — {Label} +0 | +{total} |

**Total: {riskScore} {riskLevel}**

---

### Appendix K: Patching DataTable Fields on a Party

Two DataTable custom fields are written during Step COMPLETE via a single `IntappJungle:append_party_table_rows` call. The tool fetches each party, appends the supplied `<Row>` XML to the named table field, and PATCHes only the changed fields. All party updates run in parallel internally — no manual fetch, parse, or merge required, and unrelated party fields are untouched.

---

#### `_moodyAgentRuns` — Written to all parties (client and related)

Records each party's own AML/KYC risk score from each run.

| Cell Key | Display Name | Type |
|---|---|---|
| `dateComplete` | Date Completed | Date — ISO 8601 with time, e.g. `2026-04-24T18:50:44Z` |
| `reqID` | Request ID | String |
| `riskScore` | Risk Score | String — e.g. `"Medium (28)"` or `"None"` |
| `riskEvents` | Risk Events | String — comma-separated event codes or `"None"` |
| `confirmedRiskEvents` | Risk Events - Confirmed | String |
| `confirmedRiskId` | Risk Ids - Confirmed | String |
| `strongRiskEvents` | Risk Events - Strong Match | String |
| `strongRiskId` | Risk Ids - Strong Match | String |
| `modRiskEvents` | Risk Events - Moderate Match | String |
| `modRiskId` | Risk Ids - Moderate Match | String |
| `ranForClient` | Ran for Client | String — `"{clientPartyName} ({clientPartyId})"` |
| `relationship` | Relationship to Client | String — e.g. `"GUO, Parent"` or `"Client"` |

> `ranForClient` and `relationship` must be added to the table schema in the Intapp form designer before they can be written via the API.

---

#### `_moodyAgentRunsClient` — Written to client party only

Records the overall aggregate risk score from each run — the risk across the full corporate structure screened for that client, not the client's own party-level score.

| Cell Key | Display Name | Type |
|---|---|---|
| `dateComplete` | Date Completed | Date — ISO 8601 with time, e.g. `2026-04-24T18:50:44Z` |
| `reqID` | Request ID | String |
| `riskScoreOverall` | Overall Risk Score | String — e.g. `"High (85)"` or `"None"` |

---

#### Row XML Format

Each row is a string of the form:

```xml
<Row>
  <Cell><Key>columnName</Key><Value>value</Value></Cell>
  <Cell><Key>columnName</Key><Value>value</Value></Cell>
</Row>
```

#### XML Cell Rules

- All `<Value>` content must be XML-escaped: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`, `'` → `&apos;`
- For empty / missing values, use `<Value></Value>` rather than omitting the `<Cell>` — keeps row schema consistent across runs
- Both date and string cells use plain text inside `<Value>` — no typed object wrappers (no `stringValue` / `dateValue`)
- Date cells use ISO 8601 with time, e.g. `2026-04-24T18:50:44Z`

---

#### Additional Field Updates (Client Only)

`_moodyAgentNext` is set on the client party via `fieldUpdates` in the same call — value is 6 months from today at midnight UTC (e.g. `2026-10-24T00:00:00Z`). It is a regular party field, not a DataTable, and accepts ISO 8601 directly.

---

#### Call Pattern

Single call to `IntappJungle:append_party_table_rows` with `updates[]`:

- **Client entry** — `tableAppends` includes both `_moodyAgentRuns` and `_moodyAgentRunsClient`; `fieldUpdates` includes `_moodyAgentNext`
- **All other entries** — `tableAppends` includes only `_moodyAgentRuns`

Tool returns `{ updated, failed, errors? }`. Use this to populate the per-party status in the Step COMPLETE display. Do not abort on partial failure.

---

### Appendix M: Hierarchy Display

Renders `{treeHierarchy}` as a recursively nested indented tree. Accepts any pre-built hierarchy — no knowledge of relationship type or data source is required.

---

#### Input

| Field | Description |
|---|---|
| `{treeHierarchy}` | Pre-built nested node tree passed in by the calling step |
| `{label}` | Caller-supplied label string (e.g. `Corporate tree parents`, `Beneficial owners`) |

---

#### Node Schema

```json
{
  "externalId": "string",
  "name": "string",
  "partyId": "string | null",
  "type": "client | party | non-party",
  "children": []
}
```

Children are nested inline — each node carries its own `children[]` array. Render the structure as-is.

---

#### Indicators

| Symbol | Type | Meaning |
|---|---|---|
| 🔷 | `client` | The client party. Shown for spatial context only — never holds the bucket's relationship label, never counted in the bucket's totals, never added to downstream `searchTerms` or grid rows. |
| ✅ | `party` | Exists as an Intapp party |
| ❌ | `non-party` | No Intapp party record |

The `{label}` describes only the ✅ and ❌ nodes in the tree — it does **not** describe the 🔷 node. For example, in a tree labelled `Corporate tree parents`, the 🔷 node is the client, not a parent; only the ✅ and ❌ nodes are parents.

---

#### Empty State

If `{treeHierarchy}` is null, empty, or all root nodes have no children and no type, render:

```
No {label} found.
```

Stop. Do not render a tree or summary line.

---

#### Rendering

Traverse '{treeHierarchy}' recursively depth-first. Indent each level 5 spaces relative to its parent. Always wrap the entire tree output in a fenced code block (```) so indentation is preserved in rendering.

```
{indicator}  {name} ({externalId})
     {indicator}  {name} ({externalId})
     {indicator}  {name} ({externalId})
          {indicator}  {name} ({externalId})
               {indicator}  {name} ({externalId})
```

---

#### Summary Line

Render immediately below the tree. Use `{label}` as the entity noun:

```
🔷 Client
✅ {N} {label} included in screening
❌ {N} {label} not included in screening
```

- Omit the ✅ line if there are no `party` nodes
- Omit the ❌ line if there are no `non-party` nodes

---

### Appendix N: External Data — Moody's GRID Resolutions

Moody's GRID hit resolutions are stored in external data and are the **source of truth** for `{moodysHitEvents}[].hits[].resolution` throughout the playbook. The conflicts-search `resolution` field is only used as input to seed external data on first sight in Step 6 Phase C — every subsequent read uses external data.

This enables:
- **Durability across runs** — resolutions persist beyond any single conflicts search and survive re-screening.
- **User-driven updates** — Step 9 lets the user review a single party's hits and bulk-update their resolutions in external data, then loops back to Step 6 so risk scoring recalculates from the refreshed data.

---

#### Record Schema

Every record uses the following exact shape:

```json
{
  "externalSourceType": "moodyAgentExternal",
  "remoteId": "{partyId}_{gridId}",
  "name": "{partyId}",
  "description": "{partyId}",
  "indexedData": null,
  "moodyAgentExternal._moodyAgentHitPartyId": "{partyId}",
  "moodyAgentExternal._moodyAgentHitEntityId": "{gridId}",
  "moodyAgentExternal._moodyAgentHitResolution": "{resolution}"
}
```

- `externalSourceType` is always `"moodyAgentExternal"` (= `{moodyResolutionSourceType}`).
- `remoteId` is `"{partyId}_{gridId}"` — uniqueness is per `(party, hit)` pair. The same Moody's GRID hit screened against two different parties produces two distinct records.
- `name` and `description` are both set to `{partyId}` so that read-by-party uses `search_name` for fast filtering.
- `{resolution}` is one of `{conResMatch}`, `{conResNoMatch}`, or any future resolution string supplied by the user.

---

#### Write Pattern

All writes go through `IntappJungle:post_external_data_bulk` — the bulk endpoint is upsert (matched by `remoteId` + `externalSourceType`).

**Step 6 Phase C** — bulk sync of all conflicts-search resolutions seen in this run. One call with all records as `items[]`.

**Step 9** — bulk update of one party's hits from user input (any combination of `Match:` and `No Match:` letter lists). One bulk call with all named hits combined into `items[]`. Never loop with individual posts.

Existing records are overwritten by their latest written version (last-write-wins).

---

#### Read Pattern

Reads happen in Step 6 Phase D, once per unique `partyId` in scope:

```
list_external_data(
  search_externalSourceType = "moodyAgentExternal",
  search_name               = "{partyId}",
  search_rowsToTake         = 200
)
```

For each returned record, populate `{moodysResolutions}[record.remoteId]` = `record.moodyAgentExternal._moodyAgentHitResolution`. Then for each hit in `{moodysHitEvents}[]`, look up by `"{partyId}_{gridId}"`.

This scales with party count, not hit count — typically 5–20 calls per run regardless of how many hits each party has.

---

#### Lifecycle Summary

| When | What happens |
|---|---|
| Step 6 Phase A | Conflicts search hits fetched. Their `resolution` field captured but **not yet** written to `{moodysHitEvents}`. |
| Step 6 Phase B | `{moodysHitEvents}[]` skeleton built. `hits[].resolution` initialised to `null`. |
| Step 6 Phase C | Conflicts-search resolutions matching `{conResMatch}` or `{conResNoMatch}` (and only those two values) written to external data via one bulk upsert call. All other resolution values are skipped. |
| Step 6 Phase D | External data read per partyId. `{moodysResolutions}` map built. Every `hits[].resolution` populated from the map (or left `null` if no record exists). |
| Step 7a | Classification reads `hits[].resolution` (already external-data-sourced). |
| Step 9 (Review & Resolve) | Bulk-upsert write of one party's named hit resolutions to external data, then loops back to Step 6 to re-read and re-score. |
