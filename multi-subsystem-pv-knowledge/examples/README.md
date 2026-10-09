# Examples — kicking the tyres before plugging in real data

These are six sample CSVs that mimic a real accelerator's PV landscape. They
exist so you can run the skill on **real-shaped** data without yet plugging
in your facility's actual export — once you are happy with the produced
artefacts (`channel_database.json`, `channel_limits.json`), swap the CSV
files for yours and re-run.

## Files

| File | What it covers |
|---|---|
| `01_magnet.csv` | Storage ring quadrupoles (QF/QD), sextupoles (SF/SD), bends, correctors |
| `02_vacuum.csv` | Cold-cathode / hot-cathode / ion pumps, gate valves |
| `03_rf.csv` | Main RF cavities (storage ring) and booster cavities |
| `04_diag.csv` | BPMs, DCCT (beam current), BLM (loss monitors), BCM (loss rate) |
| `05_laser.csv` | UV / IR lasers, laser diodes |
| `06_target.csv` | Target chamber vacuum, target positioning, target motors |
| `epics_pv_limits.csv` | Per-PV write bounds for `:SP` rows; merged in by Gate 4 |

All rows are normal EPICS PV names of the form `SECTION:SYSTEM:DEVICE:ID:FIELD`.
Sections: SR (storage ring), BR (booster), HEBT (high-energy beam transport),
LASER (laser system). All NAMES are conventional and would resolve through
EPICS CA across whatever IOC hosts them; you do not need to wire each PV to
an IOC host to ingest them.

## How to run the skill on these

```bash
# 1. Set up a working directory
mkdir -p ~/epics-ingest && cd ~/epics-ingest

# 2. Copy the example CSVs in
cp /path/to/osprey/plugins/osprey/skills/multi-subsystem-pv-knowledge/examples/*.csv .

# 3. (Optional) install the plugin if not already
claude plugin marketplace add /path/to/osprey --sparse .claude-plugin plugins
claude plugin install osprey@osprey

# 4. Start Claude Code here and ask, for example:
#    "把当前目录下的 6 个 CSV 文件合并成 OSPREY 知识库"
#    "Ingest these subsystem CSVs into OSPREY and emit channel_database.json"

# 5. After the skill finishes, the artefacts are at:
#    build/inventory.md
#    build/all.normalised.csv
#    build/merge.log
#    data/processed/channel_database.json
#    data/processed/channel_database.nolabels.json
#    data/processed/standalone.labels.json
#    data/limits/channel_limits.json
```

## Sanity-check the output

After the run, look at a few things by hand:

```bash
# Channel count and write-safety:
python3 -c "
import json
db = json.load(open('data/processed/channel_database.json'))
lim = json.load(open('data/limits/channel_limits.json'))
print('Templates:', sum(1 for c in db['channels'] if c.get('template')))
print('Standalones:', sum(1 for c in db['channels'] if not c.get('template')))
print('Writable channels in limits:', sum(1 for k, v in lim.items() if not k.startswith('_') and v.get('writable')))
# It is normal for the writable count to be roughly = the number of :SP rows.
"
```

Approximate expected numbers from these example CSVs:

- ~10 templates (MagQF, MagQD, MagSF, MagSD, MagBEND, MagCOR, VacCCG, VacTCG, VacIP, VacVALVE, RFCAV-SR, RFCAV-BR, BPM, DCCT, BLM, UVLaser, IRLaser, Diode, TrgtChm, TrgtPos, TrgtMtr, HEBTBeam)
- ~6 standalones (LASER:IR group with 1 instance × standalone-style fields; HEBT:BEAM:CURRENT:RB; HEBT:BEAM:PROFILE:RB; VAC:RING:STATUS:RB — depending on family inference)
- ~30 writable rows in `channel_limits.json` (the `:SP` rows that have entries in `epics_pv_limits.csv`)

If you see wildly different numbers, the family-inference in Gate 2 went
off-rail — re-read `references/csv-contract.md` §"Variant B".

## What to change for real data

Once you have your own facility's CSVs:

1. **Replace the files.** Same header, same column order. You do not need to
   keep the `examples/` filenames; the skill inventories whatever CSVs are
   in the working directory.
2. **Family classification may differ.** If your team uses the family column
   reliably (Variant A), you are ahead. If not, the inference rule will
   group by the second colon-token (`MAG:QF:01:...` → family `MagQF`).
3. **Limits.** Replace `epics_pv_limits.csv` with yours. Same three columns.
   If you don't have per-row limits, delete the file — every `:SP` row
   becomes `writable: true, confirm: true` with no min/max.
4. **Run the skill again.** Idempotent — overwrites previous outputs cleanly.

## What you should NOT change

The header line. The five columns. Their order. If your CSV has different
columns, the skill will refuse it on Gate 1 — that is by design, not by
inconvenience, because the upstream
`src/osprey/services/channel_finder/tools/build_database.py` only parses
these five. Change the contract here, in this folder, in conversation with
the skill; do not change the data.
