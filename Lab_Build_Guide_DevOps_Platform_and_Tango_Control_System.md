# Lab Build Guide — On-Prem DevOps Platform + Tango Control System (from scratch)

**Purpose:** Rebuild, in a lab, the two flagship projects from the Master CV so they can be
demonstrated end-to-end:

- **Project #6 — On-Prem DevOps Platform** (IaC · self-hosted GitOps · multi-format packaging · observability)
- **Project #4 — Tango Controls system** for a 6–20 MeV electron-LINAC beamline (PyTango device servers,
  Sardana, Taurus, HDB++)

…and then wire them together so the Tango software is **built, packaged, and deployed through the DevOps
platform** — the integration story that makes both projects one coherent capability.

> **Honest scoping note.** A software lab has no real beamline hardware (magnets, Ce:YAG cameras, Faraday
> cups, vacuum pumps). This guide therefore builds the **complete software architecture against simulated
> hardware** — simulator device servers and Sardana's built-in dummy controllers. That is exactly how these
> systems are developed and tested before hardware exists, so it is faithful to the real workflow, not a shortcut.

---

## 0. Architecture at a glance

```
                        ┌─────────────────────────────────────────────────────────┐
                        │                     DevOps PLATFORM (#6)                  │
                        │                                                           │
   dev laptop ──git──►  │  GitLab (SCM + Container Registry)                        │
                        │     │                                                     │
                        │     ├─► GitLab CI runners ──► build .deb / conda / pip    │
                        │     │                         build OCI (Podman)          │
                        │     │                         build Apptainer (.sif)      │
                        │     │                              │                      │
                        │     │                              ▼                      │
                        │     │        signed apt repo (reprepro) + conda channel   │
                        │     │        + OCI images in GitLab registry              │
                        │     │                              │                      │
                        │     ▼                              │                      │
                        │  Argo CD ◄── watches Git ──────────┘  (GitOps)            │
                        │     │  reconciles                                         │
                        │     ▼                                                     │
                        │  kubeadm Kubernetes cluster  ◄── Prometheus / Grafana /   │
                        │     │                              Alertmanager (obs.)    │
                        └─────┼─────────────────────────────────────────────────────┘
                              │ deploys
                              ▼
                        ┌─────────────────────────────────────────────────────────┐
                        │                   TANGO CONTROL SYSTEM (#4)               │
                        │                                                           │
                        │  Tango DB (MariaDB) ◄─ central device registry            │
                        │  Device servers (PyTango, simulated hardware):            │
                        │    • Magnet power supplies (dipole/quad/corrector)        │
                        │    • Ce:YAG beam-profile camera (synthetic image)         │
                        │    • Faraday cup / beam-current transformer               │
                        │    • UHV pumping station / vacuum gauge                   │
                        │    • Motion (via Sardana dummy controllers)               │
                        │  Sardana (Pool + MacroServer) · Taurus operator GUIs      │
                        │  HDB++ archiving (TimescaleDB) · Alarm/interlock monitor  │
                        └─────────────────────────────────────────────────────────┘
```

### Two viable topologies

| | **Topology A — single workstation + VMs** (recommended for lab) | **Topology B — 3–4 physical mini-PCs** |
|---|---|---|
| Hardware | 1 host, ≥32 GB RAM, 8+ cores, ≥500 GB SSD | 3–4 nodes, 16 GB RAM each |
| K8s | kubeadm across 3 VMs (1 control-plane + 2 workers) | kubeadm across physical nodes |
| Pros | cheap, snapshot/rollback, reproducible | closest to production, real network |
| Cons | resource pressure when everything runs | more cabling/setup |

The commands below assume **Topology A** on **Ubuntu 22.04 LTS** guests. Everything is open-source, no cloud.

### Prerequisites (skills you already have, listed for completeness)
- Comfortable Linux admin (RHCE/RHCSA level), SSH, systemd, networking, TLS basics.
- Git fundamentals; basic Python; basic YAML; basic containers.
- A hypervisor on the host: **VirtualBox** (simplest with Vagrant) or **libvirt/KVM** (lighter, preferred on Linux hosts).

### Base tooling on the host
```bash
# On the host workstation (Ubuntu example)
sudo apt update
sudo apt install -y git curl gnupg2 make build-essential
# Vagrant + a provider
sudo apt install -y virtualbox                # or: qemu-kvm libvirt-daemon-system + vagrant-libvirt plugin
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y vagrant
# Ansible
sudo apt install -y ansible
```

---

# PART A — The On-Prem DevOps Platform (Master CV #6)

## A1. Infrastructure & configuration as code (Vagrant + Ansible)

Goal: idempotent, rebuildable VMs defined entirely in code. Rebuilding the lab must be `vagrant destroy && vagrant up`.

**A1.1 Repo layout**
```
devops-platform/
├── Vagrantfile
├── ansible/
│   ├── inventory.ini
│   ├── site.yml
│   └── roles/
│       ├── common/         # hardening, packages, users, NTP
│       ├── gitlab/
│       ├── runner/
│       ├── k8s-common/
│       ├── k8s-control/
│       ├── k8s-worker/
│       └── repo/           # apt/conda package registry host
└── README.md
```

**A1.2 Vagrantfile** (control-plane + 2 workers + a "services" VM for GitLab/registry)
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  nodes = {
    "svc"  => { ip: "192.168.56.10", mem: 6144, cpu: 2 },  # GitLab, registry, obs
    "cp"   => { ip: "192.168.56.11", mem: 4096, cpu: 2 },  # k8s control-plane
    "w1"   => { ip: "192.168.56.12", mem: 4096, cpu: 2 },  # k8s worker
    "w2"   => { ip: "192.168.56.13", mem: 4096, cpu: 2 },  # k8s worker
  }
  nodes.each do |name, opts|
    config.vm.define name do |n|
      n.vm.hostname = name
      n.vm.network "private_network", ip: opts[:ip]
      n.vm.provider "virtualbox" do |vb|
        vb.memory = opts[:mem]; vb.cpus = opts[:cpu]
      end
    end
  end
  # Provision all nodes once they are up
  config.vm.provision "ansible" do |a|
    a.playbook = "ansible/site.yml"
    a.inventory_path = "ansible/inventory.ini"
    a.limit = "all"
  end
end
```

**A1.3 `common` role** — the idempotent hardening baseline: create an admin user + SSH key, `apt` unattended
security updates, `chrony` NTP, `ufw` firewall, timezone, and a MOTD. Keep every task idempotent (use Ansible
modules, not `shell`, wherever possible). Bring the cluster up:
```bash
cd devops-platform
vagrant up            # first build: 15–30 min depending on host
vagrant provision     # re-run Ansible any time; must be safely repeatable
```

**Deliverable checkpoint A1:** `vagrant destroy -f && vagrant up` reproduces four hardened, identical hosts.

---

## A2. Self-hosted GitLab (SCM + container registry)

Runs on `svc` (192.168.56.10). Use the Omnibus package driven by an Ansible role.

**A2.1 Install (role `gitlab`, executed by Ansible; shown here as raw commands)**
```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://gitlab.lab" apt-get install -y gitlab-ce
# Add 192.168.56.10 gitlab.lab to /etc/hosts on every node + your workstation
sudo gitlab-ctl reconfigure
sudo gitlab-rake "gitlab:password:reset[root]"   # set the root password
```

**A2.2 Enable the built-in container registry** in `/etc/gitlab/gitlab.rb`:
```ruby
registry_external_url 'http://gitlab.lab:5050'
```
then `sudo gitlab-ctl reconfigure`. For a lab, plain HTTP is fine; for TLS use a local CA (see Appendix).

**A2.3 Create the group/projects** in the UI: a group `beamline`, projects `tango-devices`, `platform-gitops`.

**Deliverable checkpoint A2:** you can `git push` to GitLab and `podman login gitlab.lab:5050`.

---

## A3. GitLab CI runners

**A3.1 Install and register a runner on `svc`** (docker executor so jobs run in clean containers):
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner
sudo apt install -y podman docker.io    # docker executor needs a container engine
sudo gitlab-runner register \
  --non-interactive --url "http://gitlab.lab/" \
  --registration-token "<PROJECT_OR_GROUP_TOKEN>" \
  --executor "docker" --docker-image "ubuntu:22.04" \
  --description "lab-runner" --tag-list "lab,build"
```

**A3.2 First pipeline** — add `.gitlab-ci.yml` to a project to prove it works:
```yaml
stages: [test]
smoke:
  stage: test
  tags: [lab]
  script:
    - echo "runner works"; uname -a
```

**Deliverable checkpoint A3:** a green pipeline appears in GitLab.

---

## A4. Kubernetes cluster with kubeadm

Control-plane on `cp`, workers `w1`/`w2`. Do this in the `k8s-common/control/worker` roles; commands shown raw.

**A4.1 Every node — container runtime + kube tools**
```bash
# containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
# kernel prereqs
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
# kubeadm/kubelet/kubectl (pin a version, e.g. v1.30)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
echo "deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/k8s.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**A4.2 Control-plane init (`cp`)**
```bash
sudo kubeadm init --apiserver-advertise-address=192.168.56.11 --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube && sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
# CNI (Flannel matches the pod CIDR above)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**A4.3 Join workers** — run the `kubeadm join …` line printed by `init` on `w1` and `w2` (or `kubeadm token create --print-join-command`). Verify:
```bash
kubectl get nodes -o wide     # all three Ready
```

**Deliverable checkpoint A4:** 3-node cluster, all `Ready`, CNI healthy.

---

## A5. Argo CD (GitOps delivery)

**A5.1 Install**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
# initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443    # then open https://localhost:8080
```

**A5.2 Register the GitOps repo** (`platform-gitops`) and create an Application that syncs a folder of manifests.
Argo CD now **reconciles the cluster from Git** — the core GitOps loop. Commit a manifest change, watch Argo apply it.

**Deliverable checkpoint A5:** editing YAML in Git changes the cluster with no `kubectl apply`.

---

## A6. Multi-format packaging & a signed distribution registry

This is the DevOps heart of the CV claim: build one control-system component **five ways** and publish it.

**A6.1 Sample component** — a tiny Python package `beamline-utils` (a stand-in you will later swap for the Tango
device server). Give it `pyproject.toml`, one module, and a console entry point.

**A6.2 Pip / wheel**
```bash
python3 -m pip install --upgrade build
python3 -m build           # produces dist/*.whl and *.tar.gz
```

**A6.3 Native Debian `.deb` (debhelper + dh-python)**
```bash
sudo apt install -y devscripts debhelper dh-python python3-all pybuild-plugin-pyproject
# in the package root:
dh_make --python -y -p beamline-utils_0.1.0   # scaffolds debian/
# edit debian/control (Build-Depends: dh-python, python3-all; Depends: ${python3:Depends})
# edit debian/rules to: %:\n\tdh $@ --with python3 --buildsystem=pybuild
dpkg-buildpackage -us -uc -b    # produces ../beamline-utils_0.1.0_all.deb
```

**A6.4 Conda package**
```bash
# install miniforge, then:
conda install -y conda-build
# write conda-recipe/meta.yaml (name, version, requirements, entry_points)
conda build conda-recipe/            # produces a .conda/.tar.bz2 artifact
```

**A6.5 OCI image (Podman) → GitLab registry**
```bash
# Containerfile installs the wheel and sets an entrypoint
podman build -t gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0 .
podman login gitlab.lab:5050
podman push gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0
```

**A6.6 Apptainer image (.sif)** — the HPC/beamline-friendly single-file format
```bash
sudo apt install -y apptainer
# beamline-utils.def:  Bootstrap: docker / From: python:3.11-slim / %post pip install the wheel
apptainer build beamline-utils.sif beamline-utils.def
apptainer run beamline-utils.sif --help
```

**A6.7 Self-hosted, GPG-signed apt repo (reprepro)** on `svc`
```bash
sudo apt install -y reprepro
gpg --quick-generate-key "Lab Repo <repo@lab>" default default never   # signing key
mkdir -p ~/apt-repo/conf
cat > ~/apt-repo/conf/distributions <<'EOF'
Origin: lab
Label: lab
Codename: jammy
Architectures: amd64 source
Components: main
Description: Lab control-system apt repo
SignWith: <KEY_FINGERPRINT>
EOF
reprepro -b ~/apt-repo includedeb jammy ../beamline-utils_0.1.0_all.deb
# serve it (nginx) and add on clients:
#   deb [signed-by=/usr/share/keyrings/lab.gpg] http://svc.lab/apt jammy main
```

**A6.8 Self-hosted conda channel**
```bash
mkdir -p ~/conda-channel/linux-64
cp <built-conda-pkg> ~/conda-channel/linux-64/
conda index ~/conda-channel        # creates repodata; serve via nginx; use with -c http://svc.lab/conda
```

**A6.9 Wire it all into CI** — `.gitlab-ci.yml` stages `test → build → publish` that produce the wheel, `.deb`,
conda pkg, OCI image and `.sif`, then push to the registry/repos. Gate `publish` behind passing tests + a
manual/code-review approval to model the "no artifact reaches a host without passing the gate" rule.

**Deliverable checkpoint A6:** a tagged commit yields all five artifacts in your registries, reproducibly.

---

## A7. Observability (Prometheus · Grafana · Alertmanager)

**A7.1 Install the kube-prometheus-stack via Helm** (covers cluster + gives Grafana/Alertmanager)
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
kubectl create namespace monitoring
helm install kps prometheus-community/kube-prometheus-stack -n monitoring
kubectl -n monitoring port-forward svc/kps-grafana 3000:80    # admin / prom-operator
```

**A7.2 Host metrics** — run `node_exporter` on the VMs (or a DaemonSet) and scrape them.

**A7.3 Golden-Signals alerting** — add a PrometheusRule for latency, traffic, errors, saturation; route to
Alertmanager (email/webhook). Import a node dashboard into Grafana. Write the one-page **host-troubleshooting
runbook** referenced on your CV (symptom → check → fix table).

**Deliverable checkpoint A7:** Grafana dashboards live; a deliberately-triggered alert fires in Alertmanager.

> **Part A done.** You now have the reproducible DevOps platform: IaC, GitLab + CI, kubeadm + Argo CD GitOps,
> five-format signed packaging, and observability.

---

# PART B — The Tango Controls System (Master CV #4)

Runs on a dedicated `tango` host (add a fifth VM, or reuse `svc` if resources allow). Simulated hardware throughout.

## B1. Tango core (database + tools)

**B1.1 Install** (apt route on Ubuntu; conda-forge is an equally valid alternative)
```bash
sudo apt install -y mariadb-server
sudo apt install -y tango-db tango-starter tango-test tango-common
# During tango-db setup you'll be asked for the TANGO_HOST (e.g. tango:10000)
export TANGO_HOST=tango:10000     # put in /etc/profile.d/tango.sh
```
Install the Java tools for inspection/scaffolding: **Jive** (browse the DB), **Astor** (start/stop servers), **Pogo** (code generator).
```bash
sudo apt install -y jive astor pogo || pip install pytango   # tools may also come via conda-forge
```

**B1.2 Verify** with the bundled `TangoTest`:
```bash
Starter &                 # or via Astor
tango_admin --add-server TangoTest/test TangoTest test/tango/1
TangoTest test &
# In Jive you should see test/tango/1 with attributes
```

**Deliverable checkpoint B1:** Tango DB up, `TANGO_HOST` set, TangoTest device visible in Jive.

## B2. PyTango + first simulated device server (scaffold with Pogo)

**B2.1 Environment**
```bash
python3 -m venv ~/tango-venv && source ~/tango-venv/bin/activate
pip install pytango numpy
```

**B2.2 Scaffold a device class in Pogo** (GUI) or hand-write the high-level API. Pattern for a **magnet power
supply** with a `Current` attribute that ramps toward a setpoint (physics simulation):
```python
# magnet_ps.py  — simulated dipole/quad/corrector power supply
from tango import AttrWriteType, DevState
from tango.server import Device, attribute, command, device_property
import threading, time

class MagnetPS(Device):
    MaxCurrent = device_property(dtype=float, default_value=200.0)

    def init_device(self):
        super().init_device()
        self._current = 0.0
        self._setpoint = 0.0
        self.set_state(DevState.ON)
        threading.Thread(target=self._ramp, daemon=True).start()

    def _ramp(self):
        while True:                      # simulate a real supply slewing to setpoint
            step = 0.5
            if abs(self._current - self._setpoint) > step:
                self._current += step if self._setpoint > self._current else -step
            time.sleep(0.1)

    current = attribute(label="Current", dtype=float, unit="A")
    def read_current(self): return self._current

    setpoint = attribute(label="Setpoint", dtype=float,
                         access=AttrWriteType.READ_WRITE, unit="A")
    def read_setpoint(self): return self._setpoint
    def write_setpoint(self, v): self._setpoint = min(v, self.MaxCurrent)

    @command
    def Reset(self): self._setpoint = 0.0

if __name__ == "__main__":
    MagnetPS.run_server()
```

**B2.3 Register + run**
```bash
tango_admin --add-server MagnetPS/dipole1 MagnetPS beam/magnet/dipole1
python magnet_ps.py dipole1 &
# In Jive: set 'setpoint' on beam/magnet/dipole1 and watch 'current' ramp
```

**Deliverable checkpoint B2:** a running, writable, self-ramping device server you built.

## B3. The full device-server set (all simulated)

Repeat the B2 pattern for each subsystem from the CV. Suggested classes and their simulated behaviour:

| Device class | Simulated behaviour |
|---|---|
| `MagnetPS` (dipole, quad, corrector, scan) | current ramps to setpoint; interlock flag |
| `CeYAGCamera` | returns a synthetic 2-D Gaussian beam image (numpy), with centroid & sigma attributes |
| `FaradayCup` / `BCT` | returns simulated beam current with noise, gated by a "beam on" flag |
| `VacuumPump` / `Gauge` | pressure decays toward a base pressure when "pumping"; trips interlock above threshold |
| `PLCInterlock` | aggregates subsystem states into a single `Permit` boolean (models safety interlock monitoring) |

Keep each device server in its own module under `tango-devices/` with a shared `pyproject.toml`.

**Deliverable checkpoint B3:** 5–6 device servers registered and inspectable in Jive.

## B4. Motion & orchestration (Sardana) + operator GUIs (Taurus)

**B4.1 Install**
```bash
pip install sardana taurus taurus_pyqtgraph pyqt5
```

**B4.2 Create a Sardana Pool + MacroServer** and use the **built-in dummy motor/counter controllers** (perfect,
since there's no real motion hardware):
```bash
Pool poolLINAC &
MacroServer msLINAC &
# via spock (Sardana CLI):
spock
> defctrl DummyMotorController motctrl
> defelem mot01 motctrl 1        # a simulated motor
> defctrl DummyCounterTimerController ctctrl
> defelem ct01 ctctrl 1
> mv mot01 10                     # move the simulated motor
> ascan mot01 0 10 10 0.1        # a step scan producing simulated data
```

**B4.3 Taurus GUI** — build an operator panel that shows the magnet current, camera image, vacuum pressure, and
a scan plot:
```bash
taurus form beam/magnet/dipole1/current beam/vacuum/gauge1/pressure &
taurus newpanel   # or assemble a .py TaurusGui with TaurusForm + image + trend widgets
```

**Deliverable checkpoint B4:** a Taurus operator screen driving simulated devices + a working Sardana scan.

## B5. HDB++ archiving

**B5.1 Backend + servers** (TimescaleDB/PostgreSQL backend is the modern default)
```bash
sudo apt install -y postgresql
sudo apt install -y hdbpp-cm hdbpp-es libhdbpp-timescale   # names vary by distro/conda
# create the hdb schema in postgres (schema SQL ships with the packages)
```

**B5.2 Configure** the Configuration Manager (`hdbpp-cm`) and an Event Subscriber (`hdbpp-es`), then subscribe a
few attributes (e.g. `beam/magnet/dipole1/current`) for archiving. Verify rows accumulate in the DB and view them
in a Taurus trend or Grafana (PostgreSQL datasource).

**Deliverable checkpoint B5:** attribute history is being archived and can be plotted.

## B6. Alarms & interlock monitoring

Implement alarm handling either via the **PyAlarm** device server or your own `PLCInterlock` aggregator: raise an
alarm/state change when vacuum pressure exceeds a threshold or a magnet reports a fault, and surface it on the
Taurus GUI. This models the "safety-interlock monitoring and alarm handling (complementing hardware protection)"
line from the CV.

**Deliverable checkpoint B6:** tripping a simulated fault changes device state and raises a visible alarm.

## B7. Engineering practice — tests, version control, CI

- Put `tango-devices/` under Git in the GitLab `tango-devices` project.
- Add **unit tests** with `pytest` + `DeviceTestContext` (PyTango's in-process test harness — no live DB needed):
```python
from tango.test_context import DeviceTestContext
from magnet_ps import MagnetPS
def test_setpoint_clamps():
    with DeviceTestContext(MagnetPS, properties={"MaxCurrent": 100.0}) as dev:
        dev.setpoint = 999
        assert dev.setpoint <= 100.0
```
- Add linting (`ruff`/`flake8`) and a `.gitlab-ci.yml` `test` stage.

**Deliverable checkpoint B7:** green CI on the Tango repo, tests running in the runner.

---

# PART C — Integration (the story that ties #4 and #6 together)

## C1. Package the device servers through the platform
Point the A6 packaging pipeline at the **real** `tango-devices` repo: build the device servers as a wheel, a
`.deb`, a conda package, and an OCI image; push to the signed apt repo / conda channel / GitLab registry.

## C2. CI builds → registry, gated
The `tango-devices` pipeline runs `pytest` → builds artifacts → (manual approval) → publishes. This is the
"reproducible, repeatable control-system builds across lab hosts" claim, now literally true in your lab.

## C3. Deploy via GitOps (or Ansible)
Two equally valid demonstrations:
- **Kubernetes/Argo CD:** wrap a device server OCI image in a Deployment manifest in `platform-gitops`; Argo CD
  syncs it to the cluster. (Use this to show GitOps; note the Tango DB dependency must be reachable.)
- **Bare-host/Ansible:** an Ansible role installs the `.deb` from your apt repo onto the `tango` host and enables
  a systemd unit for the device server — closer to how beamline hosts actually run.

**Final deliverable:** commit a code change to a device server → CI tests + builds + publishes → deployment updates
automatically → the change is visible in Jive/Taurus. That single loop demonstrates both projects and their union.

---

# PART D — Time estimate (6–8 h/day, 5 days/week)

Estimates are **working days** for someone at your level (strong Linux/Ansible, some k8s, prior exposure to these
tools), building methodically **with debugging included**. Ranges reflect low/high depending on how much is
first-time vs. reused.

### Part A — DevOps platform
| Phase | Task | Days |
|---|---|---|
| A1 | Vagrant + Ansible IaC base (roles, hardening, multi-VM) | 2–3 |
| A2 | Self-hosted GitLab + registry | 1–2 |
| A3 | CI runners | 0.5–1 |
| A4 | kubeadm cluster (containerd, CNI, join) | 2–3 |
| A5 | Argo CD + first GitOps app | 1 |
| A6 | Five-format packaging + signed apt/conda repos + CI wiring | 3–4 |
| A7 | Prometheus/Grafana/Alertmanager + runbook | 1–2 |
| **A subtotal** | | **10.5–16** |

### Part B — Tango control system
| Phase | Task | Days |
|---|---|---|
| B1 | Tango core (DB, tools, TangoTest) | 1–2 |
| B2 | PyTango env + first device server via Pogo | 2–3 |
| B3 | Full simulated device-server set (5–6 subsystems) | 5–8 |
| B4 | Sardana (dummy controllers) + Taurus GUIs | 2–3 |
| B5 | HDB++ archiving (TimescaleDB) | 1–2 |
| B6 | Alarms / interlock monitoring | 1–2 |
| B7 | Tests + Git + CI | 2–3 |
| **B subtotal** | | **14–23** |

### Part C — Integration
| Phase | Task | Days |
|---|---|---|
| C1–C3 | Package device servers, gated CI, GitOps/Ansible deploy | 2–3 |
| **C subtotal** | | **2–3** |

### Totals
| Scenario | Working days | Calendar (5-day weeks) |
|---|---|---|
| **Fast** (lots reused, few surprises) | ~27 | **~5.5 weeks** |
| **Typical** (realistic debugging) | ~35 | **~7 weeks** |
| **Thorough** (first-time on several tools, polished demos) | ~42 | **~8.5 weeks** |

**Planning figure: ≈ 6–8 weeks** of focused lab work at 6–8 h/day. A sensible single number to quote is **~7 weeks
(~35 working days)**.

### What moves the number
- **Faster:** reuse community Ansible roles/Helm charts; use conda-forge for the whole Tango stack (avoids apt
  packaging quirks); start with 2 device servers, not 6; skip Apptainer if not needed.
- **Slower:** writing realistic hardware simulators; TLS/local-CA everywhere; kubeadm networking issues; HDB++
  schema/versioning friction; making the Taurus GUIs genuinely polished.

### Suggested build order (each phase independently demoable)
1. A1 → A2 → A3 (you can already show IaC + CI in ~4 days)
2. B1 → B2 → B3 (a working simulated control system, demoable without k8s)
3. A4 → A5 (cluster + GitOps)
4. A6 → C (packaging + the integration loop — the headline demo)
5. A7, B4–B6 (observability, GUIs, archiving, alarms — depth and polish)

---

## Appendix — versions, tips, pitfalls
- **Pin versions** (Ubuntu 22.04, k8s v1.30.x, a fixed Tango/PyTango release) so the lab is reproducible; record
  them in a `versions.md`.
- **conda-forge Tango** is often the least painful install path if apt packaging fights you:
  `conda create -n tango -c conda-forge tango pytango sardana taurus`.
- **Snapshot** each VM after a green checkpoint so you can roll back instead of rebuilding.
- **kubeadm gotchas:** swap must be off; cgroup driver must be `systemd` on both containerd and kubelet; pod CIDR
  must match the CNI.
- **Tango gotcha:** every shell that runs a device server or tool needs `TANGO_HOST` exported.
- **Don't over-scope simulators:** a device server that returns physically-plausible numbers is enough to prove the
  architecture; you're demonstrating controls software, not modelling beam dynamics.

*Everything above is open-source and reproducible; rebuilding from a clean host should be a matter of
`vagrant up` + running the pipelines.*
