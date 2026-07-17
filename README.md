# Pinscope

Pinscope reviews schematics the way a good senior engineer does: with the datasheets open.

<!-- ── demo screenshot / GIF ──────────────────────────────────────────
<p align="center">
  <img src="frontend/public/report.png" width="760" alt="A Pinscope review, findings cited to datasheet pages">
</p>
─────────────────────────────────────────────────────────────────────── -->

Give it a netlist, a BOM, and your datasheet PDFs. It builds a graph of your design, reads each IC's datasheet, and checks the circuit around every part against what the manufacturer actually specifies — reference application, pin functions, absolute maximums, recommended operating conditions. Every finding points at the datasheet page that backs it up, so you can judge the call yourself instead of trusting a black box.

The reason it exists: ERC passes boards that don't work. Your EDA tool has no idea that the CH340E you powered from 5 V drives its TXD at 4.5 V into an MCU pin that maxes out at 3.6 V, or that the net you labeled `UART5_TX` lands on a pin whose alternate-function table only offers `UART5_RX`, or that the LDO's bypass pin you left floating costs you an order of magnitude in output noise. None of that is an electrical *rule* violation. All of it is in the datasheet, and nobody has time to re-read 400 pages per part on every revision.

## How it works

<p align="center">
  <img src="docs/how-it-works.svg" width="920" alt="Pipeline: the netlist and BOM are parsed into a design graph; datasheet PDFs are extracted into pin tables and specs; a per-IC review reads both and files findings cited to datasheet pages; the derating table and BOM roll-up are computed straight from the graph, no model involved.">
</p>

1. **Parse** the BOM (CSV/XLSX) and netlist (PADS-PCB `.asc` or EDIF 2.0.0 `.edn` — exportable from KiCad, Altium, OrCAD, Allegro, Xpedition, EasyEDA, Eagle) into a queryable bipartite graph of components and nets.
2. **Extract** pin tables and specs from the PDFs. Large datasheets are trimmed to the relevant pages first, and every extraction is cached in a shared library, so a given part number is only ever processed once.
3. **Review** each IC in isolation. The model gets the trimmed datasheet plus that IC's circuit neighborhood, can query the graph (`find_connected_components`, `get_net_for_pin`, `get_pintable`) and pull pages from a *connected* part's datasheet when a finding spans an interface. It files findings with severity, reasoning, and page citations.
4. **Compute** the deterministic parts deterministically — BOM roll-up and a capacitor voltage-derating table come straight from the graph, no model involved.

A post-pass normalizes findings conservatively: it can merge duplicates and downgrade severity, never upgrade. If the reviewer hedged, the report hedges.

It's a reviewer, not an oracle. It misses things, and it will occasionally question a choice you made on purpose — that's what the citations are for.

## Try it on the bundled design

`simple_project/` is a small MSPM0G3507 board with a CH340E USB-UART bridge and an SPX3819 LDO. Run it through and Pinscope flags, among other things, the LDO's bypass pin left unconnected (~300 µV<sub>RMS</sub> output noise instead of ~40) and the 5 V-powered CH340E driving the 3.3 V MCU directly — each with the page reference to check its work.

You need Python 3.12+, Node 20+, and an [Anthropic API key](https://console.anthropic.com/):

```bash
pip install -r backend/requirements.txt
cp backend/.env.example .env                 # set ANTHROPIC_API_KEY
python3 scripts/upload_skills.py --update    # one-time: registers the extraction prompts under your account
python3 -m uvicorn backend.main:app --reload
cd frontend && npm install && npm run dev
```

Open http://localhost:3000, create a project, and feed it the netlist and BOM from `simple_project/` plus datasheet PDFs for the ICs — grab those from the manufacturers, or set the optional DigiKey API keys and let it fetch them. Everything runs locally against your own key; projects and the extraction library live in `data/`. Architecture notes are in [CLAUDE.md](CLAUDE.md).

## Hosted version

This repo is the product minus accounts and billing. If you'd rather not run it yourself, [pinscope.ai](https://pinscope.ai) is the same code, hosted, with team workspaces and a shared parts library that's already warm.

## License

AGPL-3.0. For commercial licensing, write to dev@faradworks.com.
