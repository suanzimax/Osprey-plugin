---
name: multi-subsystem-pv-knowledge
description: >
  Use when someone has multiple standard-format CSVs of EPICS PVs from different
  subsystems (laser, target, vacuum, magnet, RF, diagnostics, ...) and needs to
  convert them into OSPREY channel-database inputs that are merged across IOC
  boundaries. Triggers include: "ingest these PV CSVs", "merge the subsystem
  CSVs into OSPREY", "build a channel database from my subsystem lists",
  "convert these CSVs into OSPREY semantic input", "process PV CSVs into an
  OSPREY knowledge base", "merge distributed PV lists", "I have multiple IOC
  PV files", "give me a channel database for these subsystems", "把这些子系统
  CSV 合成 OSPREY 知识库", "把这些 PV 表合成 OSPREY 通道库". Do NOT use for a
  single CSV (run `osprey channel-finder build-database` instead), nor for
  reading live PVs from IOCs.
allowed-tools: Read, Bash, Write, Edit, Glob, Grep, Agent
---

# Multi-subsystem PV knowledge ingest

Take N CSVs (one per subsystem — laser / target / magnet / RF / vacuum /
diagnostics / ...) and produce ONE OSPREY knowledge-base input set that an
operator can ask natural-language questions about. The PVs cross many IOCs
and many IP subnets; **that information does not enter the data model.**
EPICS CA looks up a PV by name, so `address` is the canonical identifier.
Do not invent IOC, host, or subnet columns.

## Five gates, in order

Skipping or reordering any gate is how bad data silently lands in the
deployment.

```
GATE 1  INVENTORY    list CSVs, count rows, lock down the column set
GATE 2  NORMALISE    every row to the canonical schema (see references/csv-contract.md)
GATE 3  MERGE        cross-CSV dedup + family roll-up, produce one table
GATE 4  LIMITS       derive channel_limits.json — only :SP rows are writable
GATE 5  VALIDATE     osprey channel-finder validate, refuse to proceed on failure
```

Every gate is a stop: do **not** start the next gate until the current one
passes. Do not ask the user for permission to skip a gate. A row that does
not pass Normalise is rejected (and the rejection is logged), not silently
fixed.

### Gate 1 — Inventory

For every CSV the user supplies:

1. Confirm the file exists and is readable.
2. Read the header line. The expected header is `address,description,family_name,instances,sub_channel`. Headers with extra columns are accepted (extras are ignored); headers missing a canonical column are **refused** — ask the user.
3. Count rows (subtract blank / `#` comment lines). Record per-CSV totals in `inventory.md`.
4. Spot-check the first 3 rows and the last 3 rows. Anything that looks like an EPICS PV name (contains `:`, all-caps segments separated by `:` or `.`) is fine; anything that looks like an IP address or hostname is a smell — flag it.

Output of Gate 1: `inventory.md` with one section per CSV.

### Gate 2 — Normalise

For every row across every CSV, write a normalised row to a per-CSV file
under `build/<subsystem>.normalised.csv`. Rules:

- `address` — trim whitespace. Reject (do not fix) any row whose address is empty, contains whitespace, contains a `/`, or does not contain a `:`. EPICS PV names are colon-separated uppercase tokens.
- `description` — trim whitespace. Reject empty descriptions. If the description is non-English, leave it; we do not translate, the LLM step will read it.
- `family_name` — keep as-is, or infer if blank (see below).
- `instances` — parse on the spot. A bare integer is `"1".."N"`. A `-` separated pair is `"A".."B"`. Anything else is a rejection.
- `sub_channel` — trim, keep blank if absent.

#### Inferring `family_name` when the CSV left it blank

If a CSV row has no `family_name`, infer one from the address prefix:

- Take the 2nd colon-delimited token if there is one, else the first token before the first `:`.
- Strip leading digits and trailing `_`-separated qualifiers.
- PascalCase it.
- Reuse the same family across rows whose prefix matches the same regex.

Example: `MAG:QF:01:CURRENT:SP` and `MAG:QF:02:CURRENT:RB` both become family `MagQF`.

Two rows that share a family must agree on `address_pattern`. If the
prefix-token rule produces two families for what is clearly the same
device type, defer to the user's column when present; otherwise ask.

#### Inferring `instances` when the CSV left it blank

Walk the rest of the file. If the addresses are `...:01:...`, `...:02:...`,
`...:03:...`, the family has instances `1-3` (or whatever the range is).
If the numbering uses `01`, `02`, ..., derive the leading-zero count from the
minimum width across the rows. Do **not** invent instances for a single row.

### Gate 3 — Merge

Concatenate the per-CSV normalised files into `build/all.normalised.csv`.
Then dedup:

- **Exact-address collision** — keep the row with the longest, most specific
  description. If descriptions tie, keep both but prefix the duplicate's
  description with `[DUP] <subsystem-of-other>: ` and log it.
- **Family collision on same address pattern** — keep one family; merge
  sub_channels. If two families expand to overlapping addresses, warn
  loudly — this is almost always a CSV bug.

Then run the deterministic channel-database build (no LLM yet):

```bash
osprey channel-finder build-database \
    --csv build/all.normalised.csv \
    --output data/processed/channel_database.json
```

Then run the LLM **only** for channels the deterministic expander left as
standalone — see Gate 3.5 below.

#### Gate 3.5 — LLM naming, narrowly scoped

LLM naming is helpful for standalone channels and harmful noise for
templated families. To confine it:

1. Run `osprey channel-finder build-database --csv build/all.normalised.csv --output data/processed/channel_database.nolabels.json` (no `--use-llm`).
2. Look at the JSON. Identify the rows whose `template` is `false` AND whose `description` is shorter than 12 words (heuristic for "LLM could add value").
3. Build a tiny CSV with **only** those rows, run `osprey channel-finder build-database --use-llm --csv small.csv --output data/processed/standalone.labels.json`.
4. Splice the `channel` and enriched `description` from `standalone.labels.json` into `channel_database.nolabels.json` by matching on `address`. Save the merged result as `data/processed/channel_database.json`.

Do **not** run `--use-llm` on the entire dataset. The LLM does not get to
rename a templated channel whose name is already a `PascalCase{instance}`.
A consistent channel naming across the whole facility is the whole point
of using family templates.

### Gate 4 — Limits

Derive `data/limits/channel_limits.json` from `all.normalised.csv`:

- Every row whose `address` ends in `:SP` is potentially writable. The
  rule from the upstream `channel_limits.json` comment applies: a channel
  is writable if and only if it is a setpoint. For each `:SP` row, set
  `writable: true`, `confirm: true`. Without explicit min/max, leave them
  out — the framework will refuse writes that have no bounds.
- Every other row (`:RB`, `:X`, `:Y`, `:STATUS`, anything else) is
  `writable: false`. **This is non-negotiable.** A misclassified RB is
  how an LLM writes to a readback.
- Default `writable: true` and `confirm: true` for the few channels the
  user explicitly marked writable but which are not setpoints — only if
  the user said so; otherwise stay strict.

If the user supplies min/max values for individual channels (a separate
sidecar CSV, `epics_pv_limits.csv` with columns `address, min, max`),
merge them in by address.

### Gate 5 — Validate

```bash
osprey channel-finder validate --database data/processed/channel_database.json --verbose
osprey channel-finder preview    --database data/processed/channel_database.json --depth 3
```

If validate returns non-zero:

- Read every error the validator emits. Most are family-pattern mismatches
  or duplicate addresses — fix the **source** CSV, not the JSON.
- Do not paper over with Edit calls on the JSON. The JSON is a derived
  artifact; regenerate it.

If preview shows a tree whose family grouping looks wrong (e.g. four
siblings under one family that should each be their own), go back to
Gate 2 and re-examine the family inference.

Only declare the run complete after Gate 5 is green.

## Outputs

| File | Purpose |
|---|---|
| `build/<subsystem>.normalised.csv` | Per-CSV normalised rows (Gate 2 output) |
| `build/all.normalised.csv` | Concatenation across subsystems (Gate 3 input) |
| `build/inventory.md` | Per-CSV header, row count, sample rows (Gate 1 output) |
| `build/merge.log` | Dedup log — every collision and how it was resolved |
| `data/processed/channel_database.json` | The actual OSPREY channel DB |
| `data/processed/channel_database.nolabels.json` | Deterministic build before LLM (kept for diffing) |
| `data/processed/standalone.labels.json` | The LLM-named slice |
| `data/limits/channel_limits.json` | `:SP` rows writable, everything else not |

A future step (graph / Neo4j) is **out of scope** for this skill. If the
user wants a TTL, point them at `osprey knowledge build-ttl` with the
database this skill produced as the input — do not run it from here.

## Failure modes this skill prevents

| Excuse an agent might rationalise | Why it is wrong |
|---|---|
| "Just run `build-database` on each CSV separately" | Produces N isolated databases, never merges. The whole point is one merged DB. |
| "I'll let the LLM name everything" | Inconsistent naming across subsystems. Family templates exist precisely to keep names stable — confine LLM to standalone. |
| "I'll add an IOC column for traceability" | `address` is the canonical identifier. Adding IOC shifts people to filter by host, which EPICS CA makes obsolete. |
| "RB is a special channel, mark it writable" | A readback is by definition not a setpoint. Misclassifying it is how an LLM later writes to a sensor. |
| "The user can fix the JSON after I write it" | The JSON is generated. Fix the source CSV. Hand-editing the JSON reproduces the same bug on the next build. |
| "Skip validate, the JSON looks fine to me" | "Looks fine" is not the validator's verdict. The validator catches family-pattern errors the eye misses. |
| "Run `--use-llm` once on the whole file" | Burns tokens on rows the deterministic expander already named. Costs money and pollutes consistency. |
| "I'll merge by hand in a spreadsheet" | Loses provenance, can not be reproduced, and Gate 5 validation cannot detect a hand merge. |
| "Family inference: just use the device prefix as family name, even if every row in the CSV already has its own family_name" | Family inference is the fallback for CSV rows that left family_name blank. If the column is filled, trust it — the operator's classification usually beats the prefix heuristic. |
| "It's only N PVs, do not bother with a manifest" | N grows. Without `inventory.md` you cannot answer "did the optics CSV make it in?" at audit time. |

## Quick reference

```bash
# Gate 1
ls -1 *.csv | xargs -I {} wc -l {}

# Gate 2
for f in *.csv; do
    python3 -c "
import csv, sys
with open(sys.argv[1]) as fh:
    r = csv.DictReader(fh)
    for row in r:
        a = row.get('address','').strip()
        if not a or ' ' in a or '/' in a or ':' not in a:
            print(f'{sys.argv[1]}: BAD address {a!r}', file=sys.stderr)
" "$f"
done

# Gate 3 (deterministic — no LLM)
osprey channel-finder build-database \
    --csv build/all.normalised.csv \
    --output data/processed/channel_database.nolabels.json

# Gate 4
python3 -c "
import csv, json
out = {'defaults': {'writable': False, 'confirm': True},
       'BR:MAG:DIPOLE:01:CURRENT:SP': {'writable': True, 'confirm': True}}  # illustrative
# real impl: derive from CSV by address suffix
"

# Gate 5
osprey channel-finder validate --database data/processed/channel_database.json --verbose
osprey channel-finder preview   --database data/processed/channel_database.json --depth 3
```

The canonical column contract for the CSVs (header layout, semver of
admissible values, what to do when a row is malformed) lives in
`references/csv-contract.md`. Read it on first use and after every
upstream change to the standard header.
