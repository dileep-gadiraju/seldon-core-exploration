# Seldon Core v2 on Windows + WSL2 — Setup & Troubleshooting

End-to-end notes for getting Seldon Core v2 running on a Windows laptop via
WSL2 (Ubuntu) + a local `kind` cluster, using the upstream Ansible playbooks.
This is the exact path I used on this machine — capturing it so I (or anyone
else) can reproduce it without re-discovering every gotcha.

> Tested on: Windows 11, WSL2 (Ubuntu 24.04 "noble"), Python 3.10/3.11,
> Docker Engine 26+, `kind` v0.23+, Seldon Core v2 (branch `v2`).

---

## Contents

1. [Windows-side prep (WSL2 + Docker)](#1-windows-side-prep-wsl2--docker)
2. [Inside the Ubuntu WSL distro](#2-inside-the-ubuntu-wsl-distro)
3. [Tooling: kubectl, helm, kind, jq, gh, seldon CLI](#3-tooling-kubectl-helm-kind-jq-gh-seldon-cli)
4. [Clone Seldon Core v2 + Python venv](#4-clone-seldon-core-v2--python-venv)
5. [Run the Ansible install](#5-run-the-ansible-install)
6. [Connect a local shell / Jupyter to the cluster](#6-connect-a-local-shell--jupyter-to-the-cluster)
7. [Verification](#7-verification)
8. [Troubleshooting log](#8-troubleshooting-log)
9. [Teardown / reset](#9-teardown--reset)
10. [Quick reference cheatsheet](#10-quick-reference-cheatsheet)

---

## 1. Windows-side prep (WSL2 + Docker)

### 1.1 Enable WSL2 and install Ubuntu

From an **elevated PowerShell**:

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default-version 2
wsl --update
wsl -l -v          # confirm VERSION = 2
```

Then launch Ubuntu once from the Start menu to create your Linux user.

### 1.2 Give WSL2 enough RAM / CPU

`kind` + Strimzi (Kafka) + Prometheus + Jaeger + OpenTelemetry is **not light**.
With WSL2 defaults (~50% of host RAM, no swap cap), the kind node OOM-kills
pods. Set explicit limits in `%USERPROFILE%\.wslconfig` (i.e. `C:\Users\<you>\.wslconfig`):

```ini
[wsl2]
memory=24GB
processors=8
swap=8GB
localhostForwarding=true
```

Then in PowerShell:

```powershell
wsl --shutdown
```

(re-launching Ubuntu picks up the new config).

> The file **must** be at `C:\Users\<you>\.wslconfig` — not in your WSL home,
> and not named `wsl.conf` (that's a different, distro-local file).

### 1.3 Docker

Two options — pick one:

- **Docker Desktop for Windows** with the "Use the WSL 2 based engine" + "Enable
  integration with my default WSL distro" toggles enabled. Easiest.
- **Native `docker` inside Ubuntu/WSL** (no Docker Desktop). Works fine; just
  `sudo apt install docker.io` and add yourself to the `docker` group:
  ```bash
  sudo usermod -aG docker $USER && newgrp docker
  ```

Either way, verify from inside WSL:

```bash
docker version
docker run --rm hello-world
```

---

## 2. Inside the Ubuntu WSL distro

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential curl wget git unzip jq \
  python3 python3-venv python3-pip \
  make ca-certificates gnupg lsb-release \
  iproute2 procps
```

`iproute2` (for `ss`) and `procps` (for `pgrep`) are used by the
`seldon_setup.ipynb` helper notebook — they're usually already present but
worth pinning.

Set a sensible git identity once:

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

---

## 3. Tooling: kubectl, helm, kind, jq, gh, seldon CLI

### kubectl

```bash
KUBECTL_VER=$(curl -L -s https://dl.k8s.io/release/stable.txt)
curl -LO "https://dl.k8s.io/release/${KUBECTL_VER}/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client
```

### helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### kind

```bash
KIND_VER=v0.23.0
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/${KIND_VER}/kind-linux-amd64"
chmod +x kind && sudo mv kind /usr/local/bin/kind
kind version
```

### GitHub CLI (`gh`) — optional but recommended

```bash
sudo apt-get install -y gh
gh auth login --hostname github.com --git-protocol https --web
```

> **WSL gotcha:** `gh auth login --web` tries to open a browser via
> `xdg-open` / `wslview`, neither of which exists by default in a fresh
> Ubuntu WSL. It will print
> `Failed opening a web browser ... Please try entering the URL in your browser manually`
> and a one-time code like `XXXX-XXXX`. **Copy the code, open
> https://github.com/login/device in Windows manually, paste, authorize.**
> The CLI then unblocks.
>
> To fix this permanently, install `wslu`:
> ```bash
> sudo apt-get install -y wslu
> ```
> after which `wslview` is on PATH and `gh auth login --web` opens the
> Windows default browser automatically.

### `seldon` CLI

Built from the Seldon Core v2 source (see §4) — there's no apt package. The
Ansible playbook does not install it for you; you build it once with:

```bash
cd ~/seldon-core/operator
make build-seldon          # produces ./bin/seldon
sudo install -m 0755 bin/seldon /usr/local/bin/seldon
seldon --help
```

(needs Go ≥ 1.22 — `sudo apt-get install -y golang-go` or grab a tarball from
https://go.dev/dl/ if the apt version is too old).

---

## 4. Clone Seldon Core v2 + Python venv

```bash
cd ~
git clone --branch v2 https://github.com/SeldonIO/seldon-core.git
cd seldon-core

python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r ansible/requirements.txt
```

`ansible/requirements.txt` pins `ansible`, `kubernetes`, `docker`, etc. — the
versions the playbooks were tested against (see `ansible/README.md` for the
support matrix).

Then install the Ansible collections the playbooks depend on:

```bash
cd ansible
make install_deps_stable
# or, equivalently:
# ansible-galaxy collection install -r requirements.yml
```

---

## 5. Run the Ansible install

From `seldon-core/ansible/` with the venv active:

### One-shot (recommended for a fresh laptop)

```bash
ansible-playbook playbooks/seldon-all.yaml
```

This runs, in order:

1. `kind-cluster.yaml` — creates a `kind` cluster named `seldon`.
2. `setup-ecosystem.yaml` — installs Strimzi/Kafka, Prometheus, Grafana,
   Jaeger, OpenTelemetry, cert-manager.
3. `setup-seldon.yaml` — installs Seldon CRDs, components, servers in the
   `seldon-mesh` namespace.

Expect this to take 5–15 minutes the first time (image pulls dominate).

### Slimmer install (skip the heavy ecosystem bits)

```bash
ansible-playbook playbooks/seldon-all.yaml \
  -e install_prometheus=no \
  -e install_grafana=no \
  -e install_jaeger=no \
  -e install_opentelemetry=no
```

Kafka is still needed by Seldon's pipeline/dataflow plane — don't disable it
unless you know you don't need pipelines.

### Confirm

```bash
kubectl get ns
kubectl -n seldon-mesh get pods
kubectl -n seldon-mesh get svc
```

You should see `seldon-scheduler`, `seldon-envoy`, `seldon-modelgateway`,
`seldon-pipelinegateway`, `seldon-dataflow-engine`, `mlserver-0`, `triton-0`,
and `svc/seldon-mesh` (the inference entry point) all `Running` / `Ready`.

---

## 6. Connect a local shell / Jupyter to the cluster

The `seldon` CLI talks to two endpoints:

| Plane | Env var | Goes to | In-cluster service |
| --- | --- | --- | --- |
| Control (gRPC) | `SELDON_SCHEDULE_HOST` | `host:port` | `svc/seldon-scheduler` port `9004` |
| Data (HTTP via envoy) | `SELDON_INFER_HOST` | `host:port` **no scheme** | `svc/seldon-mesh` port `80` |

Since `kind` is private to your laptop, you reach those via `kubectl port-forward`.

The companion notebook
[`notebooks/seldon_setup.ipynb`](notebooks/seldon_setup.ipynb) automates the
whole dance (kills stale forwards, starts fresh ones with `nohup`, exports the
env vars, verifies end-to-end). To do it by hand:

```bash
# Terminal 1 — scheduler (control plane)
kubectl -n seldon-mesh port-forward svc/seldon-scheduler 9104:9004

# Terminal 2 — inference mesh (data plane)
kubectl -n seldon-mesh port-forward svc/seldon-mesh 9100:80

# Terminal 3 — your working shell
export SELDON_SCHEDULE_HOST=0.0.0.0:9104
export SELDON_INFER_HOST=0.0.0.0:9100      # host:port, NO http:// prefix
seldon model list
```

> ⚠️ **Biggest single gotcha in this whole stack:** `SELDON_INFER_HOST` must be
> `host:port` only. The `seldon` CLI prepends `http://` itself. If you set
> `SELDON_INFER_HOST=http://0.0.0.0:9100` you'll get malformed URLs like
> `http://http://0.0.0.0:9100/v2/...` and confusing 4xx/5xx errors with no
> obvious cause.

---

## 7. Verification

```bash
# Control plane
seldon model list                               # empty table = success

# Data plane (envoy is up even with no models loaded)
curl -sS -o /dev/null -w '%{http_code}\n' \
  http://0.0.0.0:9100/v2/health/live            # expect 200

# End-to-end (only if you've loaded a model, e.g. iris)
seldon model load -f k8s/samples/models/sklearn-iris-gs.yaml
seldon model status iris -w ModelAvailable
seldon model infer iris -i 5 \
  '{"inputs":[{"name":"predict","shape":[1,4],"datatype":"FP32","data":[[1,2,3,4]]}]}'
```

---

## 8. Troubleshooting log

Concrete things that bit me on this laptop, with the fix that worked.

### 8.1 `gh: command not found`
`gh` is not preinstalled on Ubuntu 24.04 WSL.
**Fix:** `sudo apt-get install -y gh`. The apt version is older (2.45) and
doesn't have `--accept-visibility-change-consequences` on `gh repo edit`;
use the REST API for visibility changes:
```bash
gh api -X PATCH repos/<owner>/<repo> -f visibility=public
```

### 8.2 `gh auth login --web` says "Failed opening a web browser"
No `xdg-open` / `wslview` on a vanilla WSL Ubuntu.
**Fix:** copy the one-time code printed by `gh`, manually open
`https://github.com/login/device` in your Windows browser, paste, authorize.
Or `sudo apt-get install -y wslu` for a permanent fix.

### 8.3 `seldon model list` hangs / returns `connection refused`
The scheduler port-forward isn't up, or it died silently after the kernel went
to sleep. Check:
```bash
ss -tlnp | grep 9104                            # is anything listening?
pgrep -af 'kubectl.*port-forward.*seldon-scheduler'
tail -n 20 /tmp/pf-seldon-scheduler.log
```
**Fix:** re-run the port-forward (or just re-execute the `seldon_setup.ipynb`
notebook — it kills stale forwards first).

### 8.4 `seldon model infer` returns a weird URL error
You almost certainly set `SELDON_INFER_HOST=http://...`. The CLI prepends the
scheme itself.
**Fix:** `export SELDON_INFER_HOST=0.0.0.0:9100` (no scheme).

### 8.5 `port 9100 in use` after a notebook re-run
Leftover `kubectl port-forward` from a previous Jupyter kernel.
**Fix:**
```bash
pkill -f 'kubectl.*port-forward.*(seldon-mesh|seldon-scheduler)'
```
The notebook does this automatically via its cleanup cell.

### 8.6 Kind cluster pods stuck `Pending` / `OOMKilled`
WSL2 ran out of memory. Default WSL caps memory at ~50% of host RAM but with
all of Strimzi + Kafka + Prometheus + Jaeger + OTel + 2 model servers the
node burns ~12–16 GB.
**Fix:** raise the limit in `C:\Users\<you>\.wslconfig` (see §1.2) and run
`wsl --shutdown` from PowerShell.

### 8.7 `ansible-playbook: command not found` after closing the shell
You forgot to re-activate the venv.
**Fix:** `source ~/seldon-core/.venv/bin/activate`.

### 8.8 Helm chart errors about missing CRDs (cert-manager, Strimzi, Prometheus)
You re-ran `setup-ecosystem.yaml` too fast and the CRDs from the operators
weren't registered yet.
**Fix:** wait a few seconds and re-run. The playbooks are idempotent.

### 8.9 `docker: permission denied while trying to connect to the Docker daemon socket`
Your user isn't in the `docker` group (native docker case).
**Fix:**
```bash
sudo usermod -aG docker $USER
newgrp docker        # or fully log out / wsl --shutdown
```

### 8.10 `kind create cluster` hangs forever on "Ensuring node image"
Slow first-time pull of `kindest/node`. Not actually broken — give it 5+ minutes
on a fresh laptop. If it still fails, `docker pull kindest/node:v1.30.0`
manually to see the real error.

### 8.11 `kubectl` works in one shell but not another
WSL doesn't share env vars between shells. `KUBECONFIG` defaults to
`~/.kube/config`, which is fine — but if you ever exported a non-default
`KUBECONFIG`, put it in `~/.bashrc`.

### 8.12 GitHub push fails with `could not read Username`
No credential helper / no `gh` / no SSH key configured.
**Fix:** the cleanest path on WSL is `gh auth login` (it also configures git
credential helper). SSH works too — generate a key with `ssh-keygen -t ed25519`
and add the `.pub` to https://github.com/settings/keys.

### 8.13 Clock skew inside WSL (TLS errors, expired certs)
WSL2's clock drifts after the host suspends.
**Fix:** `sudo hwclock -s` or just `wsl --shutdown` and relaunch.

---

## 9. Teardown / reset

```bash
# Drop just the kind cluster (keeps your tooling)
kind delete cluster --name seldon

# Also clear Ansible's local cache
rm -rf ~/.cache/seldon/

# Full nuke (re-do steps 4–5 to reinstall)
docker system prune -af --volumes
```

---

## 10. Quick reference cheatsheet

```bash
# Reactivate the project env
cd ~/seldon-core && source .venv/bin/activate

# Reinstall (idempotent)
cd ansible && ansible-playbook playbooks/seldon-all.yaml

# Bring the laptop back online after a reboot:
#   1) ensure docker is up
docker ps >/dev/null || sudo service docker start
#   2) ensure kind cluster is up (it persists across reboots)
kubectl cluster-info --request-timeout=5s
#   3) (re)start port-forwards via the notebook
jupyter lab notebooks/seldon_setup.ipynb

# Useful one-liners
kubectl -n seldon-mesh get pods -w
kubectl -n seldon-mesh logs deploy/seldon-scheduler -f
kubectl -n seldon-mesh logs statefulset/mlserver -f
seldon model list
seldon pipeline list
seldon experiment list
```

---

## See also

- Upstream Ansible docs: [`seldon-core/ansible/README.md`](https://github.com/SeldonIO/seldon-core/blob/v2/ansible/README.md)
- Upstream getting-started: https://docs.seldon.io/projects/seldon-core/en/v2/
- `kind` on WSL2 notes: https://kind.sigs.k8s.io/docs/user/using-wsl2/
