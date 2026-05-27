# seldon-core-exploration

Personal exploration notes, helper notebooks, and small utilities for working
with [Seldon Core v2](https://github.com/SeldonIO/seldon-core) on a local
Kubernetes cluster (typically `kind`).

This repo is intentionally lightweight: it does **not** vendor or fork
Seldon Core itself. It assumes you already have a working Seldon Core v2
install in a cluster and want a reproducible way to drive it from your laptop.

---

## Contents

| Path | What it does |
| --- | --- |
| [`notebooks/seldon_setup.ipynb`](notebooks/seldon_setup.ipynb) | (Re)connects a local shell / Jupyter kernel to a Seldon Core v2 deployment: discovers services, (re)starts `kubectl port-forward`s for the scheduler and the inference mesh, exports the env vars the `seldon` CLI expects, and verifies the control plane + data plane end-to-end. |

More notebooks and notes will be added over time.

---

## Prerequisites

- A reachable Kubernetes cluster with Seldon Core v2 installed
  (default namespace assumed: `seldon-mesh`).
- Local tools on `PATH`:
  - `kubectl` — configured against the target cluster
  - [`seldon`](https://github.com/SeldonIO/seldon-core/tree/v2/operator/cmd/seldon) CLI
  - `curl`
- Python 3.10+ with `jupyter` (only for running the notebooks).

The `seldon_setup` notebook validates all of the above in its first cell, so
you'll get a clear error if something is missing.

---

## Quick start

```bash
git clone https://github.com/dileep-gadiraju/seldon-core-exploration.git
cd seldon-core-exploration

# Optional: isolated Python env for the notebooks
python -m venv .venv
source .venv/bin/activate
pip install jupyter

jupyter lab notebooks/seldon_setup.ipynb
```

Then run the cells top-to-bottom. By default the notebook forwards:

| Component | Local port | Service | Cluster port |
| --- | --- | --- | --- |
| Scheduler (gRPC, control plane) | `9104` | `svc/seldon-scheduler` | `9004` |
| Inference mesh (HTTP, data plane via envoy) | `9100` | `svc/seldon-mesh` | `80` |

Adjust the constants in the **Configuration** cell if your cluster layout
differs.

After the notebook completes successfully, the `seldon` CLI is usable from
any shell that inherits the exported env vars:

```bash
export SELDON_SCHEDULE_HOST=0.0.0.0:9104
export SELDON_INFER_HOST=0.0.0.0:9100   # host:port, no scheme — the CLI adds http://

seldon model list
```

---

## Notes & gotchas

- `SELDON_INFER_HOST` **must not** include a `http://` prefix — the `seldon`
  CLI prepends the scheme itself; including it leads to malformed URLs like
  `http://http://...`.
- The notebook uses `nohup ... &` to background the port-forwards so they
  survive a kernel restart. Re-running the notebook is safe: stale
  port-forwards are killed first.
- A teardown cell at the bottom of the notebook stops the port-forwards.

---

## License

MIT — see [`LICENSE`](LICENSE).
