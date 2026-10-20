---
name: sac-data-action
description: SAP Analytics Cloud Data Action expert. Use when building, debugging, or verifying SAC Data Actions — advanced formula steps, allocation steps, copy steps, cross-model copy steps, currency conversion steps, fact deletion steps, and embedded data action steps. Covers formula syntax, scope rules, null handling, allocations (all 5 variants), blank-output diagnosis, and output verification. Invoke with /da-help, /da-context, /da-build, /da-allocate, or /da-verify.
---

# SAC Data Action Assistant

**SAC version covered**: 2026.15 (Q3 2026)

You are an SAC Data Action expert. You are first-time right. You answer from the knowledge embedded in this skill — you do not guess, and you do not confuse SAC Advanced Formula with SQL, FOX, or HANA Script.

---

## Commands

| Command | What it does |
|---|---|
| `/da-help` | How to use this skill; how to export model data |
| `/da-context` | Phase 1 — capture model context |
| `/da-build` | Phase 2 — build a data action spec |
| `/da-allocate` | Allocation intake — variant decision tree + spec |
| `/da-verify` | Phase 3 — verify actual vs expected output |

---

## /da-help

### What to provide

**To build a new data action**: Describe what must move where, what drives the distribution, which versions hold source and target data, and whether source should zero out. Attach a CSV export of a story table showing real member IDs (see export steps below).

**To debug blank or wrong output**: Describe what you ran, what you expected, what you got. Attach a CSV of the target slice after the run.

**To ask a question**: Ask directly — no command needed.

### How to export model context as CSV

1. Open the SAC story that uses your planning model
2. Add a table widget; configure it to show all key dimensions as rows/columns
3. Set Display → **ID** for every dimension (not Description)
4. Include at least one slice of booked data (one version, one quarter, a few cost centers)
5. Right-click the table → **Export** → **CSV**
6. Attach the CSV here

Claude reads: exact dimension names, member IDs, data granularity, which members have booked data.

---

## Step Types — Overview

| Step Type | Purpose | Currency conversion? |
|---|---|---|
| Advanced Formula | Calculations, conditional logic, cross-model LINK | No |
| Copy Step | Copy data between members within same model | No |
| Cross-Model Copy Step | Copy data from one planning model to another | No |
| Currency Conversion Step | Copy across measures with different currencies | **Yes** |
| Fact Deletion Step | Delete facts, delete zeros, or set facts to zero | No |
| Allocation Step (Legacy) | Proportional distribution using driver values | No |
| Embedded Data Action Step | Reuse another data action as a step | Inherited |

> **Currency conversion caveat**: Only the Currency Conversion Step handles currency. All other step types copy raw values without conversion. Mixed-currency copies via other step types can give unexpected results.

---

## Four Rules Before Writing Any Formula

1. **Leaf level only.** Calculations run only on leaf members. Parent members are excluded unless `CONFIG.READ_CALCULATED_MEMBER_VALUES = ON` (HANA Cloud tenants only).
2. **RESULTLOOKUP does not aggregate.** Each unspecified dimension returns one row per member — not a sum. Use `AGGREGATE_DIMENSIONS` explicitly for aggregation.
3. **Unspecified dimensions iterate automatically.** Every leaf member of every dimension not in scope will be processed. Narrow with MEMBERSET.
4. **Steps run sequentially.** Step B reads the result of Step A. Data commits only after all steps succeed.

---

## Calculation Scope

```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Account] = ("Revenue", "Cost")
DATA() = RESULTLOOKUP([d/Version] = "Actual")
```

**Scope levels** (narrowest wins):
1. MEMBERSET — whole step
2. IF / FOREACH block — within the block only
3. Individual DATA() statement — single line

**ELSEIF excludes prior scope** (from 2021.02):
```
IF [d/Date] = "202501" THEN
  DATA() = 100          // Scope: Date = 202501
ELSEIF [d/Region] = "EUR" THEN
  DATA() = 200          // Scope: Date ≠ 202501 AND Region = EUR
ELSE
  DATA() = 300          // Scope: everything else
ENDIF
```

---

## CONFIG Functions

| CONFIG | Values | Effect |
|---|---|---|
| `CONFIG.GENERATE_UNBOOKED_DATA` | OFF (default) / ON | ON treats unbooked source as 0 in copies |
| `CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE` | OFF (default) / ON | ON flips sign for EXP/INC per account type. **Most common source of wrong INC+EXP results.** |
| `CONFIG.TIME_HIERARCHY` | CALENDARYEAR / FISCALYEAR | Which time hierarchy the script uses |
| `CONFIG.READ_CALCULATED_MEMBER_VALUES` | ON | Allows RESULTLOOKUP to read calculated/parent members (HANA Cloud only) |
| `CONFIG.TIME_ZONE_OFFSET` | numeric (e.g. 5.5) | Adjusts TODAY() for timezone |
| `CONFIG.HIERARCHY [ModelName]` | dimension hierarchy path | Specifies which hierarchy to use |
| `CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY` | OFF (default) / ON | ON expands scope to members not in any hierarchy (orphan members) |
| `CONFIG.MDXERRORLEVEL` | 0 / 1 / 2 | 0 = silent, 2 = error |

CONFIG must appear at the top of the script, before MEMBERSET.

---

## Member Selector Functions

### MEMBERSET
```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = "202501" TO "202512"
MEMBERSET [d/Product] = ("PRD001", "PRD002")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
```

> **Version dimension in Advanced Formulas**: Version cannot be defined in MEMBERSET — it is treated specially and controlled by the TargetVersion parameter. Use it only inside RESULTLOOKUP or LINK functions.

### BASEMEMBER — all leaf members under a parent
```
BASEMEMBER([d/Date].[h/YQM], "2025")       // Returns all months in 2025
BASEMEMBER([d/Date].[h/YQM], "2025.Q1")    // Returns Jan, Feb, Mar 2025
```
**Always use BASEMEMBER when referencing a time parent.** Writing to a parent directly does nothing — data is stored at leaf level.

### ELIMMEMBER — exclude members from a set
```
ELIMMEMBER([d/Product].[LEVEL00].Members, [d/Product].[Total])
ELIMMEMBER([d/Entity], [d/Entity].[p/Consolidation], "Y")  // Exclude by attribute
```

### AGGREGATE_DIMENSIONS / AGGREGATE_WRITETO
```
AGGREGATE_DIMENSIONS [d/Entity]
AGGREGATE_WRITETO [d/Entity] = "#"
DATA([d/Date] = "202501") = RESULTLOOKUP([d/Date] = "202412") + 20
```
Without AGGREGATE_DIMENSIONS: adds 20 to each entity. With it: sums all entities first, then adds 20, writes to "#".

---

## Write-Back Functions

### DATA — primary write (overwrites)
```
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Price") * RESULTLOOKUP([d/Account] = "Qty")
DATA()    // writes to all cells in current scope
```

### DATA.APPEND — accumulate instead of overwrite
```
// Sum two source accounts into one target without clearing target between lines
DATA.APPEND([d/Account] = "TotalCost") = RESULTLOOKUP([d/Account] = "DirectCost")
DATA.APPEND([d/Account] = "TotalCost") = RESULTLOOKUP([d/Account] = "IndirectCost")
```
> **Re-run gotcha**: DATA.APPEND accumulates on every run. Add `DELETE()` before the loop to reset the target, or results will compound.

### CARRYFORWARD — built-in opening/closing balance carry-forward
```
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
CARRYFORWARD([d/Movement] = "CLOSING", [d/Movement] = "OPENING")
```
Iterates chronologically through all periods in scope. Each period's OPENING = prior period's CLOSING. First period in scope is left unchanged.

### DELETE
```
MEMBERSET [d/Product] = "Discontinued_SKU"
DELETE()
```

### WRITETO — cross-dimensional write
```
WRITETO([d/Entity] = "Consolidated", RESULTLOOKUP() * 1.1)
```

---

## RESULTLOOKUP — Reading Data

```
RESULTLOOKUP()                                          // Current scope
RESULTLOOKUP([d/Account] = "Revenue")                  // Specific member
RESULTLOOKUP([d/Version] = "Actual")                   // Different version
RESULTLOOKUP([d/Date] = PREVIOUS(1, "MONTH"))          // Time offset
RESULTLOOKUP([d/Date] = PREVIOUS(12, "MONTH"))         // Same month prior year
```

RESULTLOOKUP does **not** aggregate. Each unspecified dimension returns separate values per member.

---

## Unbooked (Null) Value Behaviour — Critical Rules

SAC treats unbooked cells as **null**, not zero. Arithmetic with null is not the same as arithmetic with 0.

| Formula | RESULTLOOKUP(a) = null, (b) = positive | Result |
|---|---|---|
| `DATA(c) = RL(a) + RL(b)` | null + positive | **null — no write** |
| `DATA(c) = RL(a) - RL(b)` | null - positive | **null — no write** |
| `DATA(c) = RL(a)` | null | written as-is |

| Formula | RESULTLOOKUP(a) = null, (b) = 0 | Result |
|---|---|---|
| `DATA(c) = RL(a) + RL(b)` | null + 0 | **0 — a zero IS written** |
| `DATA(c) = RL(a) - RL(b)` | null - 0 | **null — no write** |

| Formula | RESULTLOOKUP(a) = 0, (b) = positive | Result |
|---|---|---|
| `DATA(c) = RL(a) + RL(b)` | 0 + positive | positive |
| `DATA(c) = RL(a) * RL(b)` | 0 × positive | 0 |

> **Most common "formula ran but nothing appeared" cause**: Source cell is unbooked (null), not zero. null + anything = null.

**`null` is a valid keyword** in IF conditions:
```
IF RESULTLOOKUP([d/Account] = "Revenue") = null THEN
  DATA([d/Account] = "Margin") = 0
ENDIF

// Safe division
IF RESULTLOOKUP([d/Account] = "Headcount") <> null THEN
  IF RESULTLOOKUP([d/Account] = "Headcount") <> 0 THEN
    DATA([d/Account] = "CostPerHead") = RESULTLOOKUP([d/Account] = "TotalCost")
                                      / RESULTLOOKUP([d/Account] = "Headcount")
  ENDIF
ENDIF
```

**Structural alternatives to null checks**:
- `FOREACH.BOOKED` — skips unbooked members entirely
- `CONFIG.GENERATE_UNBOOKED_DATA = ON` — treats unbooked source as 0 in copies

---

## Time Functions

| Function | Syntax | Description |
|---|---|---|
| `PREVIOUS` | `PREVIOUS(n, "MONTH")` | n periods before current |
| `NEXT` | `NEXT(n, "MONTH")` | n periods after current |
| `PREYEARLAST` | `PREYEARLAST([d/Date])` | Last period of prior year |
| `FIRST` | `FIRST([d/Date])` | First period of the year |
| `LAST` | `LAST([d/Date])` | Last period of the year |
| `PERIOD` | `PERIOD("20250304")` | Specific date member |
| `TODAY` | `TODAY()` | Current date (adjusted by TIME_ZONE_OFFSET) |
| `DAY` | `DAY([d/Date])` | Day number within month |
| `DAYSINMONTH` | `DAYSINMONTH([d/Date])` | Days in the month |
| `DAYSINYEAR` | `DAYSINYEAR([d/Date])` | Days in the calendar year |
| `WEEK` | `WEEK([d/Date])` | ISO week number |
| `DATEDIFF` | `DATEDIFF([d/Date], %End%, "MONTH")` | Difference between two dates. Units: DAY, WEEK, MONTH, QUARTER, YEAR. Optional modes: CalendarDiff, FLOOR, Ceiling |
| `DATERATIO` | `DATERATIO([d/Date], %Start%, %End%)` | Overlapping days ÷ total period days. Returns 0–1. Use for partial-period cost proration. |

---

## Mathematical Functions

| Function | Syntax | Description |
|---|---|---|
| `ABS` | `ABS(<number>)` | Absolute value |
| `LOG` | `LOG(<number>)` | Natural log (base e) |
| `LOG10` | `LOG10(<number>)` | Base-10 log |
| `MOD` | `MOD(<number>, <divisor>)` | Remainder after integer division |
| `POWER` | `POWER(<base>, <exp>)` | base ^ exp |
| `SQRT` | `SQRT(<number>)` | Square root |

## Conversion Functions

| Function | Syntax | Description |
|---|---|---|
| `ROUND` | `ROUND(<n>, <decimals>)` | Round to nearest |
| `FLOOR` | `FLOOR(<n>, <decimals>)` | Round down (toward −∞) |
| `CEIL` | `CEIL(<n>, <decimals>)` | Round up (toward +∞) |
| `TRUNC` | `TRUNC(<n>, <decimals>)` | Truncate toward 0 |
| `FLOAT` | `FLOAT(<number>)` | Convert to floating-point |
| `INT` | `INT(<number>)` | Integer portion only |

`<decimals>` is optional, defaults to 0. Negative values round to tens/hundreds (e.g. -2 = nearest 100).
> **FLOOR vs TRUNC for negatives**: `FLOOR(-2.7)` = -3. `TRUNC(-2.7)` = -2.

All math/conversion functions accept: numeric literal, variable (`@x`), parameter (`%x%`), `RESULTLOOKUP()`, or `ATTRIBUTE()`.

---

## FOREACH and FOR Loops

```
// Basic FOREACH
FOREACH [d/Employee]
  DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = "Sales") * 0.1
ENDFOR

// FOREACH.BOOKED — skip members with no data
FOREACH.BOOKED [d/Product]
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue")
                                - RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// FOREACH with ordering — critical for cumulative calculations
FOREACH [d/Date] ASC
  DATA([d/Account] = "Cumulative") = RESULTLOOKUP([d/Account] = "Cumulative", [d/Date] = PREVIOUS())
                                   + RESULTLOOKUP([d/Account] = "Monthly")
ENDFOR

// FOR counter loop
INTEGER @i
FOR @i = 1 TO 12
  // body
ENDFOR

// FOR with STEP
INTEGER @m
FOR @m = 1 TO 12 STEP 3    // 1, 4, 7, 10
ENDFOR
FOR @m = 10 TO 1 STEP -1   // countdown
ENDFOR

// BREAK
FOREACH [d/Product]
  IF RESULTLOOKUP([d/Account] = "Revenue") > 200 THEN
    BREAK
  ENDIF
ENDFOR
```

> **FOREACH not supported for Measures in models with measures.** Use DATA([d/Measures] = ...) instead.

---

## Variables

```
FLOAT @GrowthRate
INTEGER @Counter
VARIABLEMEMBER #totalSales OF [d/Account]

@GrowthRate = RESULTLOOKUP([d/Account] = "Rate") / 100
DATA([d/Account] = "Forecast") = RESULTLOOKUP([d/Account] = "Actual") * (1 + @GrowthRate)
```

> **Version dimension**: Not supported in VARIABLEMEMBER. Measures ARE supported: `VARIABLEMEMBER #total OF [d/Measures]`

---

## LINK and MODEL Block — Cross-Model

```
MODEL [DriverModel]
  MEMBERSET [d/Date] = "202501" TO "202512"
  HIERARCHY = [d/Entity].[h/Regional]
ENDMODEL

DATA([d/Account] = "BudgetCost") = LINK([DriverModel], [d/Version] = "Actual", [d/Measures] = "Cost")

// Null check on LINK
IF LINK([DriverModel], [d/Version] = "Actual") = null THEN
  DATA() = 0
ENDIF
```

**LINK prerequisites** — linked model must have:
- Same currency conversion setting (enabled or disabled)
- Same date dimension granularity
- Same fiscal year setting (or both off)
- When source model has measures: must specify a single measure in LINK statement

**Classic Account → Model with Measures migration**: Cross-model copy steps cannot copy between different model types. Use LINK in Advanced Formula instead:
```
DATA([d/Measures]="Quantity") = LINK([AccountModel], [d/Version]="public.Forecast", [d/Account]="Quantity")
```
Add `CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE=ON` to preserve negative values on accounts with auto sign flipping.

---

## Advanced Formulas Step — UI Panel Fields

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off |
| Header | Mode toggle | **Visual** / **Script** — switches between drag-and-drop canvas and script editor |
| Context | Add Filters button | Adds dimension filters to scope the formula |
| Variable | Add Variables button | Declares script variables (FLOAT, MEMBER, etc.) |
| Canvas (Visual mode) | Begin → [formula blocks] → End | Drag blocks between Begin and End nodes; blocks include Calendar Year, Signflip (On/Off), Unbooked (On/Off) |
| Canvas controls | Zoom In / Zoom Out / Fit to Screen / Minimap toggle | Standard canvas navigation |

> In **Script mode**, the canvas is replaced by a code editor. All CONFIG, MEMBERSET, DATA(), RESULTLOOKUP() etc. are written directly. The Context (filters) and Variable sections remain visible as helpers.

## Advanced Formulas — Visual Tool vs Script Editor

| | Visual Tool | Script Editor |
|---|---|---|
| **Target user** | Business users | IT / modeling specialists |
| **Coding required** | No | Yes |
| **How created** | Drag and drop | Write scripts |

**Script-only functions** (not available in visual tool):
`CARRYFORWARD`, `ATTRIBUTE`, `ELIMMEMBER`, `MOD`, `POWER`, `ROUND`, `FLOOR`, `CEILING`, `TRUNC`, `DAY`, `MONTH`, `YEAR`, `PERIOD`, `DAYSINMONTH`, `DAYSINYEAR`, `DATERATIO`, `CONFIG.HIERARCHY`, `CONFIG.TIME_ZONE_OFFSET`, `ELSEIF`, `ELSE`, some `MEMBERSET` forms, `AGGREGATE_DIMENSIONS`, `AGGREGATE_WRITETO`, `MODEL`

**Rule**: Use the script editor for any complex business scenario. The visual tool is not a replacement — use it only for simple cases.

---

## Cell Value Filter Rules (IF RESULTLOOKUP() > x)

**Case 1 — Filter and calculation use same dimension member**: Condition narrows the MEMBERSET scope. Members not passing the condition are dropped.

**Case 2 — Filter dimension ≠ calculation dimension**: SAC finds members matching the filter condition, then intersects with the MEMBERSET for the DATA() write. Can produce unexpected scope — always verify with tracing.

---

## ATTRIBUTE Function

```
IF ATTRIBUTE([d/Product].[p/Factor]) > 2000 THEN
  DATA() = RESULTLOOKUP() * 1.5
ENDIF

IF [d/Account].[p/ACCTYPE] = ("AST", "LEQ") THEN
  // Runs only for Asset and Liability accounts
ENDIF
```

---

## SisterMember

```
MEMBERSET [d/Entity].[p/Calculation] = "Y"
DATA([d/Entity] = [d/Entity].[p/SisterMember]) = RESULTLOOKUP()
```

---

## Script Structure Template

```
// 1. CONFIG (top)
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON

// 2. Variables / VARIABLEMEMBER
FLOAT @Rate

// 3. MEMBERSET (scope narrowing)
MEMBERSET [d/Version] = "Budget"          // NOT valid — version handled by TargetVersion param
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")

// 4. Logic
@Rate = RESULTLOOKUP([d/Account] = "GrowthRate") / 100
DATA([d/Account] = "Forecast") = RESULTLOOKUP([d/Account] = "Actual", [d/Version] = "Actual")
                                * (1 + @Rate)
```

---

## Copy Step — Complete Reference

### Overview
A copy step copies data between members within the **same model**. Currency conversion is not supported — all values are copied as-is.

Use cases:
- Copy current-period forecast data to future periods
- Copy data from one product/region to another
- Copy between plan versions using SourceVersion + TargetVersion parameters

### Section 1 — Filters (source scope)

- Filters narrow which source data is read before copying.
- **Version dimension**: Available **only in Filters** — never in Copy Rules. This is by SAP design.
  - To copy from Forecast → Budget: add a Version filter here, select source version (or use a `SourceVersion` parameter).
  - If no version filter is added, data copies **within the target version only** (the `TargetVersion` parameter controls the write target).
- **Account dimension with multiple hierarchies**: The hierarchy you pick for the account filter must match the hierarchy used in Copy Rules for source members on that account dimension.
- Parameters are supported in all filter fields.

> **Critical rule**: Version is ONLY in Filters. It is intentionally absent from Copy Rules across all models — this is not a model configuration issue.

### Section 2 — Aggregate To

- Override the default disaggregation behaviour for specific dimensions.
- **Default behaviour (no Aggregate To)**: Copied data follows the same distribution as the source. Equivalent to "Paste underlying values."
- **With Aggregate To**: All data for that dimension aggregates into the single leaf member you specify.
  - Example: aggregate Customer dimension to "#" (Unassigned) to remove customer distribution for privacy.
- **Constraints**:
  - Must choose a **single leaf member** (not a parent)
  - Dimensions added here **cannot** also appear in Filters or Copy Rules for the same step
- Parameters are supported.

### Section 3 — Copy Rules

- Define source → target member mappings. Optional: if Filters + Aggregate To fully describe the copy, no rules are needed.
  - Example — copy 2025 Forecast to target version: only need a version Filter and date Filter. No copy rules needed.
  - Example — copy 2025 Forecast to both 2025 and 2026: copy rule on Date dimension required.
- **Version in Copy Rules**: Not available (see Section 1 — this is by design).
- **Date dimension**: Non-leaf members at the same hierarchy level are valid targets (e.g., Q1 → Q2, or 2025 → 2026). This is an exception to the leaf-only rule.
- **Measures**: Keep value range and decimal precision in mind. When using measure parameters, specify data types where possible.
- **Account dimension with multiple hierarchies**: Source members in Copy Rules must use the same hierarchy as the account filter. Different hierarchies are allowed for target members.
- Multiple target members per rule: value is copied to each target independently.
- Multiple source → same target: data is aggregated into the single target.
- Parameters are supported in From and To columns.

### Section 4 — Options

**Write Mode**

| Mode | Behaviour | Default? |
|---|---|---|
| Append | Copied data is added to existing target values | **Yes — default** |
| Overwrite | Existing target data is replaced by copied values | No — must set explicitly |

> **Write Mode = Append is the default.** For a version copy scenario (clearing a version before writing), set Write Mode = **Overwrite** explicitly, or you will be adding on top of prior data.

**Auto-Generation Mode** (how copy rules apply to time dimensions)

| Mode | Behaviour |
|---|---|
| Detect Automatically | SAC chooses the most suitable mode |
| Day Granularity | Based on number of days between members; accounts for leap years |
| Week Granularity | Based on number of days between members; weeks always = 7 days (no leap year adjustment) |
| Month/Year Granularity | Based on number of months between members |

**Copy Factor** — numeric multiplier applied to all copied values. Default = 1 (copy as-is). Set to 0.5 to halve values, 2 to double, etc. Supports parameters.

### Version copy — correct configuration pattern

```
Step type: Copy Step

Filters:
  Version = %SourceVersion%   ← required; controls read scope
  [optional: Date, CostCenter, etc.]

Aggregate To: [leave empty unless needed]

Copy Rules: [add only if member remapping is needed]
  (not Version — Version is never in Copy Rules)

Options:
  Write Mode = Overwrite       ← must set explicitly for version copy
  Auto-Generation Mode = Detect Automatically

Parameters:
  SourceVersion   member  [source version to copy from]
  TargetVersion   member  [write target — handled by SAC planning engine]
```

### UI Panel Fields (what you see in the editor)

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label shown in the step list |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off — inactive steps are skipped at runtime |
| Context | Data Source | Read-only — always the data action's default model |
| Context | Filters | Add Filter button; dimension picker dropdown; Version filter here only |
| Aggregate To | — | Add Dimension button; pick one leaf member per dimension |
| Copy Rules | — | Per-dimension section; From/To member pickers |
| Options | Write Mode | Append (default) / Overwrite |
| Options | Auto-Generation Mode | Detect Automatically / Day / Week / Month-Year |
| Options | Copy Factor | Numeric input; default = 1 |

### Copy Step gotchas

| # | Symptom | Cause | Fix |
|---|---|---|---|
| CS1 | Version not available in Copy Rules dropdown | By SAP design — Version is Filters-only | Use Version Filter + SourceVersion parameter instead |
| CS2 | Data keeps accumulating on re-run | Write Mode = Append (default) | Set Write Mode = Overwrite |
| CS3 | Dimension cannot be added to both Filters and Aggregate To | Mutual exclusion rule | Move to one section only |
| CS4 | Aggregate To rejects parent member | Must be a single leaf member | Choose a leaf member or "#" |
| CS5 | Copy rule Date validation warning | Non-leaf date targets must be at the same hierarchy level as source | Use same hierarchy level (e.g. Q1→Q2 works; Q1→2025 does not) |
| CS6 | Account hierarchy mismatch error | Filter hierarchy ≠ Copy Rule source hierarchy for same account | Match hierarchies |
| CS7 | Wrong values on partial-year copy | Auto-Generation Mode mismatch | Try Day Granularity or Month/Year explicitly |
| CS8 | Nothing copied but no error | No booked data in source scope; or Aggregate To leaf member has no data | Verify source data exists in filter scope |

---

## Fact Deletion Step — Complete Reference

### UI Panel Fields (what you see in the editor)

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off |
| Context | Data Source | Read-only — data action's default model |
| Context | Filters | Add Filter button; dimension filters (Version, Date, etc.); parameters supported |
| Options | Delete Mode | Dropdown: Delete Facts / Delete Zeros from Facts / Set Facts to Zero |

### Overview
Removes or zeroes out fact records in the data action's default model. Use this as a clean-up step before a copy or formula step to avoid accumulation issues, or to purge stale planning data.

### Configuration sections

**Context**:
- Data Source: the data action's default model (read-only, cannot change)
- Filters: narrow which records are affected — add dimension filters (Version, Date, CostCenter, etc.). Parameters are supported.
  - Version filter: same pattern as other step types — filter to set which version's facts are affected.

**Options**:

**Delete Mode**

| Mode | Behaviour |
|---|---|
| Delete Facts | Permanently removes the fact records from the model |
| Delete Zeros from Facts | Removes only records where the value = 0; non-zero records are untouched |
| Set Facts to Zero | Overwrites fact values with 0 rather than deleting the record; the cell exists but holds 0 |

> **Delete Facts vs Set Facts to Zero**: Delete Facts removes the record entirely (cell becomes unbooked/null). Set Facts to Zero writes an explicit 0 (cell is booked with value 0). This matters for formulas that check `= null` vs `= 0`.

### Common use patterns

- **Pre-copy clean-up**: Add a Fact Deletion Step (Delete Facts, version filter = TargetVersion) before a Copy Step set to Append, so you get a clean target without needing Overwrite mode.
- **Remove planning data for a closed period**: Filter to the specific date range + version and delete.
- **Zero out a budget version**: Use Set Facts to Zero to preserve the version structure while clearing values.

---

## Cross-Model Copy Step — Complete Reference

### UI Panel Fields (what you see in the editor)

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off |
| Context | Source Model | Free-text input + value-help button. User picks the source model. |
| Context | Target Model | Read-only — always the data action's default model |
| Context | Filters (Add Filter) | Dimension filter picker; Version filter required; parameters supported |
| Mapping | [Dimension rows] | One row per target dimension; shows "Drop Source Dimension or Select default value…". Drag source dimensions from the source panel onto target rows |
| Mapping | Source Dimensions panel | Left panel listing all source model dimensions; has a filter textbox for search |
| Options | Write Mode | Append / Overwrite (no Copy Factor unlike regular Copy Step) |

> Account and Date are auto-mapped and cannot be unmapped. Hidden accounts are NOT auto-mapped.

### Overview
Copies data from one planning model (source) to another (target). The **target model** is always the data action's default model.

- Currency conversion is **not supported** — all values are copied as-is.
- Can only copy between the **same model type** (classic account ↔ classic account, or measures ↔ measures). To copy between different types, use LINK in Advanced Formulas.
- Data booked on non-leaf members is ignored (warning message shown; dismiss to continue).
- Parameters are supported throughout.

### Configuration sections

**Source Model**: Pick the model whose data you want to copy.

**Filter section**:
- Version filter is **mandatory** — select the source version or use a parameter.
- The TargetVersion parameter controls the write target (same pattern as regular Copy Step).
- Additional filters (Date, CostCenter, etc.) are optional.
- If dimension has a parent-child hierarchy: source members in copy rules should use the same hierarchy as the filter.

**Mapping area**:
For each target dimension, either map a source dimension or set a default value.

- **Map** a source dimension when you want to distribute data across different members of the target dimension. Drag source dimension onto target dimension to map.
- **Set default value** when no appropriate source dimension exists, or when you want to collapse all data to a single leaf member.
  - Example: mapping currency-enabled source to currency-enabled target — set Currency dimension default to source model's currency.
- Account and Date dimensions are auto-mapped and cannot be unmapped.
  - Hidden accounts are NOT auto-mapped.
  - To use a different date dimension or a leaf date as default, delete the auto-mapping.

**Auto-Creation Strategy** (per dimension mapping):

| Strategy | Behaviour |
|---|---|
| No Auto-Creation | Manual copy rules only |
| Identical Names | Auto-maps members with identical names |
| Identical Names (Including Ancestors) | Maps if source member name (or any ancestor name) = target member name |
| First day of time period | Maps using first day of source period as reference (cross-granularity date mapping) |
| Last day of time period | Maps using last day of source period as reference |

> For date dimensions with different granularities (e.g. month → day), use First or Last day of time period.
> When granularity and time settings are identical, First and Last day give the same result.

**Completion Options** (for unmapped source members):

| Option | Behaviour |
|---|---|
| Manual | Must create copy rules for every listed source member |
| Fixed Value | Specify a default target for unmapped members; partial copy rules still allowed |
| Ignore | Unmapped source members are filtered out |

**Copy Rules** (within a dimension mapping):
- Source member: does NOT need to be a leaf member
- Target member: must be a leaf member (calculated members not allowed as targets)
- Multiple targets: source value is copied to each target
- Multiple sources → same target: values are aggregated
- Account dimension with multiple hierarchies: source members must use same hierarchy as source context filter
- Date dimension: incomplete mapping is allowed — unmatched time periods are silently ignored

**Write Mode** (same as regular Copy Step):
- Append (default)
- Overwrite

---

## Currency Conversion Step — Complete Reference

### UI Panel Fields (what you see in the editor)

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label shown in the step list |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off |
| Context | Data Source | Read-only — data action's default model |
| Context | Filters | Add Filter button; Version filter + any other dimension filters; parameters supported |
| Conversion Rules | From | Source measure (value-help picker) — not conversion/calculated measures |
| Conversion Rules | To | Target measure (value-help picker) — not conversion/calculated measures |
| Conversion Rules | + Add Conversion Rule | Adds a new From→To row |
| Options | Write Mode | Append / Overwrite |

> Each conversion rule row has a delete button. Multiple rules allowed (one per source→target measure pair).

### Prerequisites
- Data action must be based on a **model with measures** that has currency conversion enabled.
- Does NOT apply to classic account models.

### Overview
Copies data between base measures with different currencies, applying conversion rates from the model's currency table. Each conversion rule: source measure → convert → target measure.

Parameters are supported throughout (filters, source/target measures).

### Configuration

**Context (Filters)**:
- Filter dimensions to scope the source data.
- Version filter: same pattern as other step types — filter to set source version; TargetVersion controls write target.

**Conversion Rules** (one per source→target measure pair):
- Source measure (From) and target measure (To).
- Conversion and calculated measures are not available as source or target.
- Multiple rules allowed — can copy, edit, duplicate, delete rules.

**Conversion Settings per rule**:

| Setting | Options | Description |
|---|---|---|
| Date | Booking Date | Use the rate that applies to each value's booking date |
| | Booking Date + 1 / Booking Date − 1 | Use rate offset by one period from the booking date |
| | Fixed Date | Use rates from a specific fixed date for all values |
| Category | Dynamic | Use rates for the target version's category/rate version |
| | Fixed category (e.g. Actuals) | Use rates for the specified category |
| | Specific | Use rates for a specific rate version in the currency table |

> Currency table must have rates for all currencies, dates, categories, and rate versions involved.

**Write Mode**: Append or Overwrite (same as other step types).

---

## Allocation Step (Legacy) — /da-allocate

### UI Panel Fields (what you see in the editor)

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off |
| Source Context | Data Source | Read-only — data action's default model |
| Source Context | Filters | Add Filter button; Version + dimension filters; parameters supported. Account dimension cannot be filtered here — filter at the base allocation step |
| Source Context | Write Mode → Keep Source | Checkbox. OFF = source values are zeroed after allocation; ON = source values are preserved |
| Driver Context | Reference Dimension | Date (Year, Quarter, Month) pre-populated; Add Reference Dimension for Audit etc. Each entry means allocation uses per-period/per-member driver weights |
| Target Context | Booking Account | "Select a Booking Account" picker (value-help). Empty = results land in source account. Set for Variant 3 |
| Target Context | Write Mode → Overwrite Target | Checkbox. Must be ON to avoid double-counting on re-run |
| Allocation Rules | Source Dimension | Column: pick source dimension; + adds a new dimension column |
| Allocation Rules | Driver Dimension | Column: pick driver dimension; + adds a new driver column |
| Allocation Rules | Target Dimension | Column: pick target dimension; + adds a new target column |
| Allocation Rules | Toolbar | Copy, Paste ▾, Delete, Show All filter, Search |

### Prerequisites
Set up the base legacy allocation step (in Allocations) before adding it to a data action. Only steps based on the data action's default model appear in the selection list.

### Configuration
- Add optional dimension filters (member-based or parameter-based).
  - **Account dimension cannot be filtered here** — apply account filters in the base allocation step instead.
  - When you pick a base allocation step that has an account filter, that filter appears in the list automatically.
- Select the allocation step from the list.

### Intake questions

| # | Question | Why |
|---|---|---|
| 1 | Should source zero out, or retain value? | Keep Source checkbox |
| 2 | Results in same account as source, or a separate booking account? | Booking Account (Variant 3) |
| 3 | Driver data in same version/year as target, or different? | Calculated Account driver (Variant 4) |
| 4 | Multiple audit trail entries that need independent handling? | Overwrite Dimension / Reference Dimension |
| 5 | Two cost centers allocating to each other (reciprocal)? | Number of Repetitions > 1 |
| 6 | Will this be re-run in the same planning cycle? | Overwrite Target must be ON |
| 7 | Each audit trail needs its own driver weights, or shared pool? | Reference Dimension (Variant 5) |

### Variant decision tree

```
Q: Driver data in a different version or year?
  YES → Variant 4 (Calculated Account) + LOOKUP formula in the driver member
  NO  → continue

Q: Each audit trail needs independent driver weights?
  YES → Variant 5 (Reference Dimension = Audit)
  NO  → continue

Q: Results to a separate booking account?
  YES → Variant 3 (Booking Account)
  NO  → continue

Q: Restrict to one specific audit trail only?
  YES → Variant 2 (Overwrite Dimension = Audit)
  NO  → Variant 1 (Basic Proportional)

Variants 2–5 can be combined (e.g. Variant 3 + 5).
Reciprocal CC → any variant + Number of Repetitions ≥ 7.
```

### Variant quick reference

**Variant 1 — Basic Proportional**
- Rule columns: Entity (Source) | Account (Driver) | Entity (Target)
- Math: Target = Source × (Target driver ÷ Sum of all target drivers)

**Variant 2 — Overwrite Dimension**
- Set Overwrite Dimension = Audit at step level
- Rule columns: Entity (Source) | Audit (Overwrite) | Account (Driver) | Entity (Target) | Audit (Target)
- Use when: only one audit trail's data should be sent out

**Variant 3 — Booking Account**
- Set Booking Account = target account name in Target Context
- Rule columns: Entity (Source) | Account (Driver) | Entity (Target) — no audit column
- Use when: results should land in a separate named account, not the source account

**Variant 4 — Calculated Account Driver**
- Driver column points to a calculated account member
- Calculated member formula:
  ```
  LOOKUP([HeadCount], [d/Version]="public.Actual" AND [d/Date]=Previous("Year",1))
  ```
- Use when: driver data is in a different version or year than the allocation

**Variant 5 — Reference Dimension**
- Add Audit to Driver Context → Reference Dimension (alongside default Date)
- Rule columns: Entity (Source) | Account (Driver) | Entity (Target) — no audit column
- Each audit trail's pool is allocated using only that audit trail's driver values

**Default Reference Dimension**: Date (Year, Quarter, Month) is always pre-populated. Each period is allocated using its own period's driver weights.

### Spec template

```
Allocation Step: [Name]
Variant: [1–5]

SOURCE CONTEXT
  Version:       [e.g. public.Plan]
  Account:       [e.g. All Expense]
  Entity:        [e.g. CC_HR]
  Keep Source:   [ON / OFF]

DRIVER CONTEXT
  Reference Dim: Date (default) [+ Audit if Variant 5]
  [Variant 4: Calculated account name + LOOKUP formula]

TARGET CONTEXT
  Booking Acct:  [account / empty]
  Overwrite:     [ON / OFF]

RULE
  # | Source   | [Overwrite Dim] | Driver    | Target         | [Target Audit]
  1 | CC_HR    | [Input by User] | Headcount | All Portfolios | [Allocs]

REPETITIONS: [1 / 7+ for reciprocal]

EXPECTED OUTPUT
  - Source CC:  [zeros out / retains value]
  - Version:    results in public.Allocs audit trail
  - Story:      add public.Allocs to Version filter to see results
```

### Pre-build checklist

- [ ] Source version has booked data in the specified scope
- [ ] Driver account has booked data for target members in same scope
- [ ] If Variant 4: LOOKUP formula has correct version string including "public." prefix
- [ ] Overwrite Target is ON
- [ ] Date appears in Reference Dimension field
- [ ] If reciprocal: Repetitions ≥ 7
- [ ] Story filter will include public.Allocs after run

---

## Allocation Gotchas

| # | Symptom | Cause | Fix |
|---|---|---|---|
| G1 | Source not zeroing out | Keep Source is checked | Uncheck Keep Source |
| G2 | Double-counting on re-run | Overwrite Target not checked | Check Overwrite Target |
| G3 | All targets same % regardless of period | Date not in Reference Dimension | Verify Date appears in Driver Context → Reference Dimension |
| G4 | Wrong audit trail being allocated | No Overwrite Dimension set | Set Overwrite Dimension = Audit; add audit columns to rule |
| G5 | Driver returns 0 — nothing allocated | Driver account has no data; LOOKUP version wrong; target is parent node | Check driver data; verify "public." prefix in LOOKUP; use leaf target members |
| G6 | Reciprocal leaves large residuals | Repetitions = 1 | Set Repetitions to 7–10 |
| G7 | Results not visible in story | Story Version filter doesn't include public.Allocs | Add public.Allocs to story Version filter |
| G8 | Allocated amounts mixed with source costs | Booking Account not set | Set Booking Account in Target Context |
| G9 | Audit trail weights mixing | Audit not in Reference Dimension | Add Audit to Driver Context → Reference Dimension |
| G10 | Driver uses wrong version or year | LOOKUP formula hard-codes year/version | Use `Previous("Year",1)`; verify "public." prefix |
| G11 | Wrong tool | Trying to do conditional logic in Allocation step | Use Advanced Formula for IF/ELSEIF logic; use Allocation step for proportional distribution |

---

## Allocation Step vs Advanced Formula — When to Use Which

| Need | Use |
|---|---|
| Proportional distribution based on a driver | Allocation Step |
| Multiple senders/receivers | Allocation Step |
| Reciprocal cost centers | Allocation Step (Repetitions) |
| Cross-version driver | Allocation Step (Variant 4) |
| Conditional logic (IF/ELSEIF) | Advanced Formula |
| Write to multiple target accounts | Advanced Formula |
| Complex iteration with FOREACH | Advanced Formula |

**Hard rule**: Allocation step writes to one Booking Account per step. Multiple conditional target accounts → Advanced Formula.

---

## Embedded Data Action Step

### UI Panel Fields (what you see in the editor)

| Section | Field | Notes |
|---|---|---|
| Header | Name | Step label |
| Header | Description | Free-text, optional |
| Header | Step Active toggle | On/Off |
| — | Data Action Name *(required)* | Dropdown + browse icon. Lists data actions that share the same default model. After selection, parameter mapping rows appear |

> The panel is intentionally minimal — there is no Context or Options section. All configuration lives in the embedded data action itself. Only parameter values are set here.

### Overview
Runs another data action as a step within the current one. The embedded data action is a **reference** — it is not modified, and can still be run independently.

Both data actions must be based on the **same default model**.

**Use cases**:
- Reuse a common calculation step across multiple data actions
- Run the same step multiple times in one data action with different parameters
  - Example: revenue calculation step parameterised by ProductType, embedded 3× with different product types

**Nesting rules**:
- Embedding a data action within multiple different data actions ✓
- Embedding a data action multiple times in one data action ✓
- Nesting multiple levels of embedded data actions ✓
- Circular embedding (a data action embedded within itself, directly or indirectly) ✗

> **Multi Action alternative**: For running data actions on different versions or models, or when you need to publish data between steps, use a Multi Action instead.

### Configuration

1. Select the data action to embed (must share the same default model).
2. Set parameter values for the embedded data action's dynamic parameters:
   - **Member parameters**: set a fixed member, or map to a parameter from the container data action.
   - **Number parameters**: type a number, or choose a number parameter from the container data action.
   - **TargetVersion**: not needed — the embedded data action runs on the same version as the container.
   - If a parameter has a default value in the embedded data action, the default can be applied with one click.
3. Validate the step before saving.

---

## Tracing (Debugging)

1. Open the data action script editor
2. Click in the **left margin** next to a line to add a tracepoint
3. Click **Open Tracing Panel** in the toolbar
4. Run the data action — execution completes fully; tracepoints don't pause it
5. Select a tracepoint to see data state at that point + changes since prior tracepoint

**Limits**: 20 tracepoints per data action; 10 traced iterations within FOREACH.

```
// TRACE() — quick log output
TRACE(RESULTLOOKUP([d/Account] = "Revenue"), "Revenue before allocation")
```

---

## Blank Output — Diagnostic Checklist

When the formula runs without error but nothing appears:

1. **Source is null, not zero** — null + anything = null. Check that source cells have booked data, not just a 0.
2. **Wrong version filter** — source version correct? Has booked data in that scope?
3. **Scope too narrow** — MEMBERSET excludes the cells you think it's writing to?
4. **Scope too broad, writing to parent** — using a parent member instead of BASEMEMBER?
5. **Date granularity mismatch** — formula uses "202501" but model stores at Quarter level?
6. **Sign flip on INC/EXP** — result written but appears zero because INC and EXP cancelled? Add CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON.
7. **Target version locked** — target is in a locked state?
8. **RESULTLOOKUP returns one row per member** — missing AGGREGATE_DIMENSIONS for aggregation?
9. **DATA.APPEND target was never reset** — stale prior-run data masking the result?
10. **Copy Step: Write Mode = Append** — re-run accumulating on top of prior data? Set Write Mode = Overwrite.

Use **Data Action Tracing** (click in left margin of script editor to add tracepoints) to inspect the data state at any point in execution.

---

## Common Script Patterns

### Carry-Forward (month-to-month)
```
MEMBERSET [d/Account] = "Headcount"
MEMBERSET [d/Movement] = ("CLOSING", "OPENING")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
CARRYFORWARD([d/Movement] = "CLOSING", [d/Movement] = "OPENING")
```

### Year-over-Year Growth
```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Account] = "Revenue"
DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(12, "MONTH"), [d/Version] = "Actual") * (1 + %GrowthRate%)
```

### Partial-Period Proration
```
// Apportion cost based on how many days of the month fall within a contract
MEMBERSET [d/Account] = "ContractCost"
DATA() = RESULTLOOKUP([d/Account] = "FullMonthCost")
       * DATERATIO([d/Date], %ContractStart%, %ContractEnd%)
```

### Gross Margin (mixed account types)
```
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON
MEMBERSET [d/Account] = "GrossMargin"
DATA() = RESULTLOOKUP([d/Account] = "NetRevenue")
       + RESULTLOOKUP([d/Account] = "CostOfGoodsSold")
// With flag ON: Revenue (900) + COGS (−700) = 200 ✓
// With flag OFF: Revenue (900) + COGS (700) = 1600 ✗
```

---

## Output Format for Every Spec

Every data action spec must include:
- Step type (Advanced Formula / Allocation / Copy / Cross-Model Copy / Currency Conversion / Embedded)
- Source scope (version, dimensions, member filters)
- Target scope (version, dimensions)
- Advanced formula script (if applicable)
- Parameters (name, type, default)
- Pre-run checklist
- Expected output description

Always use member IDs in MDX format: `[Dimension].[Hierarchy].&[MemberID]`

If context is insufficient to safely generate a spec, ask for the missing information — do not guess dimension names or member IDs.

---

## Maintainer
Cognivion SAC Practice | SAC 2026.15 | github.com/anilkumarkn100/sac-da-knowledge
