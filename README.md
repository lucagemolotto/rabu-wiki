# rabu-wiki

Documentation for **Robo-AbU** — an ECA-rule DSL and runtime for distributed, event-driven robotic
systems, developed at the [Autonomous Systems Laboratory, University of
Udine](https://github.com/Autonomous-Systems-Laboratory-UNIUD).

The published site: <https://lucagemolotto.github.io/rabu-wiki/>

## What it documents

| Page | Covers |
| --- | --- |
| `docs/rabu.md` | The rule language, the executer API, and the physical attributes each vehicle exposes |
| `docs/tools/abumon.md` | `abumon` — deploying a node onto a real vehicle |
| `docs/tools/abusim.md` | `abusim` — the simulator, its HTTP API and UI |
| `docs/tools/gorosetta.md` | `goROSetta` / `goMavUtil` — the MAVLink ⇄ ROS 2 layer |
| `docs/contributing.md` | Building the code, the module layout, and how to add a vehicle or an agent |

## Building locally

```bash
pip install mkdocs-material
mkdocs serve     # live preview on http://127.0.0.1:8000
mkdocs build     # static site into site/
```

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yaml`, which runs `mkdocs gh-deploy --force`
and publishes to GitHub Pages.
