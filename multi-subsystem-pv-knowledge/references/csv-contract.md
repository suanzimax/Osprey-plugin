# CSV input contract — multi-subsystem PV ingest

This document is the **canonical column schema** the agent validates every
input CSV against in Gate 1 (`Inventory`) and Gate 2 (`Normalise`).

It is referenced from `SKILL.md` and must be re-read whenever OSPREY's
`channel-finder` build-database changes its accepted input layout (check
`src/osprey/services/channel_finder/tools/build_database.py` to verify).

## 1. The standard header

```
address,description,family_name,instances,sub_channel
```

Five columns. Order is fixed. Extra columns are ignored. Missing canonical
columns are rejected.

### Field semantics

| Column | Type | Required | Empty allowed | Notes |
|---|---|---|---|---|
| `address` | string | yes | **no — rejected if blank** | Full EPICS PV name. EPICS CA lookups by name; this is the canonical identifier. IOC/host information is **not** a column; do not invent one. |
| `description` | string | yes | **no — rejected if blank** | One-sentence natural-language description. The seed of every semantic lookup downstream. |
| `family_name` | string | no | yes (inferred from address prefix — see SKILL.md §"Inferring `family_name`") | Same name across rows that should expand into a templated family. Operators usually fill this; the column exists exactly so we do not have to guess. |
| `instances` | string | no | yes (default `1`) | A bare integer `N` means `1..N`. A `-`-separated pair `A-B` means `A..B` inclusive. Anything else is a malformed row. |
| `sub_channel` | string | no | yes | Sub-address within the family. Expandable into `{suffix}` of the family's address_pattern. |

A leading `#` is a comment and the row is skipped. Empty data rows are
skipped. Whitespace around cell values is trimmed.

### Pre-validation: what an address must look like

Before accepting a row in Gate 2 the agent checks:

- not blank
- no whitespace anywhere
- no `/` (path-like values are not PVs)
- contains at least one `:` (a single-token PV with no separator is allowed but suspicious)

These rules catch typos like `MAG QF:01:CURRENT:SP` (whitespace) and
`/srv/epics/ioc.db` (path) before they enter the merged CSV.

### Description quality bar

The pipeline reads `description` literally when running Cypher or in-context
retrieval, so its quality determines semantic match quality. Adopt this
five-element pattern:

```
<device name>, <quantity/role>, <system role>, [<unit>], [<position/location>]
```

Examples of well-formed descriptions:

```
Quadrupole magnet QF01 horizontal focus strength setpoint, in the storage ring, amperes.
BPM 05 measured horizontal beam position, ring diagnostics, millimetres, straight section 3.
RF cavity 01 input power, RF system, kilowatts, station 1.
```

Avoid:

- "Magnetic field" — quantity but no device; useless downstream.
- "PV for QF01 current setpoint" — third-person meta-narration; redundant.
- Chinese-only descriptions — the LLM step reads them, but in-context and
  Cypher substring matching is best in English. Bilingual is fine.

## 2. Two ways a row can come in

The standard column list above is the supported shape. In practice,
upstream teams hand you one of two variants.

### Variant A — pre-classified rows (operator already templated)

```csv
address,description,family_name,instances,sub_channel
MAG:QF:01:CURRENT:SP,聚焦四极磁铁 01 励磁电流设定值,MagQF,1-32,CURRENT:SP
MAG:QF:01:CURRENT:RB,聚焦四极磁铁 01 励磁电流读出值,MagQF,1-32,CURRENT:RB
MAG:QF:02:CURRENT:SP,聚焦四极磁铁 02 励磁电流设定值,MagQF,1-32,CURRENT:SP
```

The agent's job in Gate 2 is *pass-through*: validate the address,
trim, write the normalised row out unchanged.

### Variant B — raw rows with no classification

```csv
address,description,family_name,instances,sub_channel
MAG:QF:01:CURRENT:SP,聚焦四极磁铁 01 励磁电流设定值,空白,空白,空白
MAG:QF:01:CURRENT:RB,聚焦四极磁铁 01 励磁电流读出值,空白,空白,空白
MAG:QF:02:CURRENT:SP,聚焦四极磁铁 02 励磁电流设定值,空白,空白,空白
```

The agent must:

1. Read every row in the CSV once into memory.
2. Group rows by the regex pattern `(PREFIX):(FAMILY):(REST)` — the
   middle token becomes the candidate family. In the example, the family
   is `QF` (between the `MAG:` prefix and the device instance).
3. Detect the instance range by looking at the numeric token immediately
   after `QF:`.
4. Detect sub-channels by looking at the last `:`-separated token
   (here, `CURRENT:SP` / `CURRENT:RB`).
5. Produce a normalised row set where `family_name=MagQF`, `instances=1-32`,
   `sub_channel=CURRENT:SP` etc.

A variant-B file with only one row per family stays **standalone**
— Gate 2 does not invent `instances > 1`.

## 3. Limits sidecar (optional)

A user-supplied `epics_pv_limits.csv` with three columns:

```csv
address,min,max
MAG:QF:01:CURRENT:SP,0.0,300.0
MAG:QF:02:CURRENT:SP,0.0,300.0
```

… is loaded in Gate 4 and merged by address into `channel_limits.json`.
Rows in this file override the default rule "every `:SP` is writable but
has no min/max". A `min`/`max` of `null` means "writable, no bound".

## 4. Sub-channel conventions OSPREY understands

The convention is `SignalType[Direction]` and is enforced by what the
agent does **not** do (no special-casing of MR / RB / SP / etc.):

| Suffix | Direction | Writes allowed? |
|---|---|---|
| `:SP` | setpoint | yes |
| `:RB` / `:X` / `:Y` / `:VAL` / `:FVAL` | readback | **no** |
| `:STATUS[:...]` | status bit / enum | **no** |
| `:OFFSET` / `:CALC` / `:DES` / `:EGU` / `:HOPR` / `:LOPR` / `:HIHI` / `:LOLO` / `:HIGH` / `:LOW` | metadata | **no** |
| `:MDEL` / `:ADEL` / `:SCAN` / `:PREC` / `:DTYP` | monitor metadata | **no** |
| Anything ending in `:SP` | setpoint | yes (the only rule — suffix alone decides) |
| Anything else | unknown | **no** — refuse rather than guess |

The determinant is the **suffix**, not device intuition. An agent that
"knows" `:CMD` is a setpoint for a particular device is **wrong**: if
the row's `address` does not literally end in `:SP`, Gate 4 marks it
not-writable.

## 5. What the agent must do when a row fails validation

It **rejects** the row, logs the rejection with the source CSV name, and
continues. A row count at the end of Gate 2 must equal:

```
sum(accepted rows across CSVs) + sum(rejected rows across CSVs) = sum(input rows)
```

The `build/inventory.md` carries both numbers. If the rejected count is
non-zero, Gate 2 reports it to the user **before** Gate 3 starts. The
agent does **not** fix bad addresses by guessing at correct PVs.

## 6. Reading the upstream source when this contract is unclear

| Question | Look at |
|---|---|
| What columns does `build-database` actually parse? | `src/osprey/services/channel_finder/tools/build_database.py` — `load_csv()`. |
| What rules govern `family_name` / `instances` / `sub_channel`? | The same file, `create_template()` and the four helpers around it. |
| How are SP/RB channels categorised in the write-safety layer? | `src/osprey/services/channel_finder/` (limits projection comments) and the shipped `data/channel_limits.json`. |
| What does `--use-llm` actually do? | `src/osprey/services/channel_finder/tools/llm_channel_namer.py`. |
| What is the precise semantic schema going to OSPREY's graph? | `FIELD_ANATOMY.md` in the knowledge-template package; re-read after every Osprey release. |

If the contract here disagrees with the code or the comments in
`channel_limits.json`, the **code wins** — file an issue, do not invent
an exception.
