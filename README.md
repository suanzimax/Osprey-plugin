# Osprey-plugin

A collection of Claude Code skills for the [Osprey framework](https://als-apg.github.io/osprey) — the agentic control-room interface for accelerators and beamlines (EPICS-based).

## Currently shipping

| Skill | What it does |
|---|---|
| [`multi-subsystem-pv-knowledge`](plugins/osprey/skills/multi-subsystem-pv-knowledge/) | Ingest N CSVs (one per subsystem: laser / target / vacuum / magnet / RF / diagnostics / ...) of EPICS PVs and emit a single OSPREY channel database plus its `channel_limits.json`. Bridges EPICS PV naming into OSPREY semantic knowledge-base inputs. |

## How to use

Each skill ships as a Claude Code slash-command and a Claude Code agent skill, both wired into the [`osprey`](https://github.com/als-apg/osprey) plugin. After cloning this repo locally:

```bash
claude plugin marketplace add .claude-plugin plugins
claude plugin install osprey@osprey
```

… then invoke the relevant skill from the Claude Code command palette.

## License

BSD 3-Clause — same as the parent OSPREY repository.
