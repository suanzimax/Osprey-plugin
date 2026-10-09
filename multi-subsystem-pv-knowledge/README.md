# multi-subsystem-pv-knowledge

A Claude Code skill that ingests **N CSVs (one per subsystem)** of EPICS PVs
and emits a single OSPREY-compatible channel database plus its
`channel_limits.json`. The point of the skill is to turn a repetitive
multi-IOC, multi-PV consolidation workflow into a deterministic, auditable
pipeline that an operator can re-run on edited input.

This README is the human-facing usage document.
[`SKILL.md`](SKILL.md) is the agent-facing instruction manual.
[`references/csv-contract.md`](references/csv-contract.md) is the formal
column-by-column contract for input CSVs.

---

## What problem it solves

You have:

- **Many subsystems** (laser, target, vacuum, magnet, RF, diagnostics, ...)
- **Distributed across many IOCs and many IPs**
- **Each subsystem ships its PVs in its own CSV**
- The CSVs are similar — same column shape, same EPICS PV convention — but
  the rows are not unified, the families are not classified the same way,
  and the description column has wildly different verbosity.

You need:

- ONE `channel_database.json` that captures every PV across every IOC.
- A correct `channel_limits.json` where only `:SP` rows are writable and
  every readback is closed.
- A path that any operator can re-run after a CSV is updated.

`multi-subsystem-pv-knowledge` produces those three artefacts.

---

## What it does not do

- **Not** a connection to live IOCs. It does not read `caget` / `camonitor`
  / ChannelFinder. CSVs only.
- **Not** TTL / Neo4j / graph generation. After this skill produces
  `channel_database.json`, run `osprey knowledge build-ttl` separately
  if you want a knowledge graph.
- **Not** `osprey build`. That step happens after validation, in your
  own deployment repo.
- **Not** a single-CSV fast path. If you have **one** CSV, run
  `osprey channel-finder build-database` directly — this skill is
  over-engineered for that.

---

## Inputs the user must supply

One or more CSVs in the working directory, each row with the standard
header:

```csv
address,description,family_name,instances,sub_channel
```

Read [`references/csv-contract.md`](references/csv-contract.md) for the
column-by-column rules. The most-cited constraints in practice:

- `address` is the **full EPICS PV name**. Do not include IOC or host.
  EPICS CA does cross-subnet lookup by name.
- `description` must be a natural-language, one-sentence description —
  this is what the LLM step and the in-context / Cypher retrieval step
  both read. Quality here dominates every downstream semantic match.
- `family_name` is optional. If absent, the agent infers it from the
  second colon-token of the address (e.g. `MAG:QF:01:...` →
  family `MagQF`).
- `instances` is optional. `8` means `1..8`, `4-11` means `4..11`. If
  absent, single-instance default.

An optional `epics_pv_limits.csv` (columns `address,min,max`) overrides
the default `:SP` bounds per address.

---

## How to invoke it

### In a Claude Code session, after the plugin is installed

```bash
claude plugin marketplace add als-apg/osprey --sparse .claude-plugin plugins
claude plugin install osprey@osprey
```

Place the CSVs in a working directory. Start Claude Code in that
directory. The skill is triggered by phrases such as:

- "Ingest these subsystem CSVs into OSPREY"
- "Merge the laser / target / vacuum CSVs into one channel database"
- "把这些子系统 CSV 合成 OSPREY 知识库"
- "Build a channel database for these PV lists"
- "Process these CSV exports into OSPREY semantic input"

You can also invoke it directly:

```
/osprey:multi-subsystem-pv-knowledge
```

### Direct CLI invocation (no agent)

Most of the pipeline is a sequence of `osprey` commands. If you prefer
to run it by hand, the equivalent shell sequence is:

```bash
# Gate 1 + 2: agent normalises. Or run the validator script below
#   for a one-shot CSV lint.

# Gate 3: deterministic channel database build
osprey channel-finder build-database \
    --csv build/all.normalised.csv \
    --output data/processed/channel_database.nolabels.json

# Gate 3.5: LLM naming, scoped to standalone rows only
# (see SKILL.md for how to derive build/standalone.csv)

osprey channel-finder build-database \
    --csv build/standalone.csv \
    --use-llm \
    --output data/processed/standalone.labels.json

# Gate 4: limits projection — implement in your tooling.
# See `references/csv-contract.md` §4 for the writable-row rule.

# Gate 5: validate
osprey channel-finder validate \
    --database data/processed/channel_database.json \
    --verbose

osprey channel-finder preview \
    --database data/processed/channel_database.json \
    --depth 3
```

The skill is the agent version of the same flow: it inventories,
normalises, dedups, builds, narrows the LLM step, projects limits, and
runs the validator for you.

---

## Outputs and where they land

After a successful run the skill leaves behind:

| File | Meaning |
|---|---|
| `build/inventory.md` | Per-CSV header, row counts, sample rows, any anomalies |
| `build/<subsystem>.normalised.csv` | Per-CSV canonicalised rows |
| `build/all.normalised.csv` | Concatenation across subsystems, ready for `build-database` |
| `build/merge.log` | Cross-CSV dedup decisions (kept for audit) |
| `data/processed/channel_database.nolabels.json` | Deterministic build, before LLM |
| `data/processed/standalone.labels.json` | LLM-named slice (still in template build shape) |
| `data/processed/channel_database.json` | The merged OSPREY channel database |
| `data/limits/channel_limits.json` | `:SP` rows writable, everything else refused |

These file paths are the same paths the OSPREY web build pipeline reads
from. After running this skill, `osprey build` inside your deployment
repo will pick the new `channel_database.json` up automatically.

---

## Re-runs and edits

The pipeline is idempotent at the **file** level: every run overwrites
its outputs from the inputs. Re-runs are safe.

- **Adding a new subsystem CSV**: drop it in the working directory and
  invoke the skill again. The new rows are merged in; existing rows
  remain.
- **Editing a description**: edit the source CSV. Re-run the skill. The
  new description will flow through Gate 2 (normalise) → Gate 3 (LLM
  rename only if standalone) → Gate 5 (validate).
- **Removing a subsystem**: delete the CSV. Re-run. The skill
  inventories whatever CSVs remain.
- **Recomputing LLM labels**: not currently part of the deterministic
  pipeline. To force a re-label, delete
  `data/processed/standalone.labels.json` and re-run. The skill will
  rebuild it from the rows currently classified as standalone.

---

## What the user reviews

The skill is auditable end-to-end. After a run, look at:

1. **`build/inventory.md`** — every CSV that was loaded, how many rows,
   what was sampled.
2. **`build/merge.log`** — every cross-CSV duplicate address and how it
   was resolved.
3. **`data/limits/channel_limits.json`** — confirm only `:SP` rows
   have `writable: true`. This is the only line where misclassification
   can leak a write to a readback.
4. **`osprey channel-finder preview`** output — does the family tree
   look right? Are the standalone rows the ones you expected?

If any of those is wrong, **fix the source CSV**, not the JSON
artefacts. The JSON is derived; re-running regenerates it. Hand-edits
get reverted on the next run.

---

## How the skill is structured

```
plugins/osprey/skills/multi-subsystem-pv-knowledge/
├── SKILL.md                       # Agent-facing instruction manual
├── README.md                      # This file (you are here)
└── references/
    └── csv-contract.md            # The CSV column-by-column contract
```

- **`SKILL.md`** is loaded into the agent when it triggers. It is concise,
  decisive, and addresse every failure mode an LLM assistant might fall
  into ("just run build-database per CSV separately", "let the LLM name
  everything", "the user can fix the JSON after", etc.).
- **`csv-contract.md`** is heavy reference — 100+ lines, only loaded on
  first use or after a contract change. It explains every column, every
  rejection rule, every variant of input the agent may see.

---

## Extending the skill

If your facility needs something this skill does not cover — your PV
naming does not include a `:`, your readback convention is not `:RB`,
your limits come from a machine-learning model rather than a sidecar CSV
— the place to extend it is `csv-contract.md`. Document the facility's
convention there and tell the agent to read it.

If the upstream `osprey channel-finder build-database` itself changes,
re-validate this skill by re-reading
`src/osprey/services/channel_finder/tools/build_database.py` against the
contract above. The contract must follow the code, not lead it.

---

## Versioning

This skill lives with the Osprey plugin it ships from. When OSPREY
version bumps (CalVer `vYYYY.M.P`), re-check the skill against the new
`build-database.py` and against the new `channel_limits.json` shape.
The CSV contract is stable across releases by design — only the
internal column-by-column treatment has changed between releases.

---

## License

Same license as the parent OSPREY repository: BSD 3-Clause.
