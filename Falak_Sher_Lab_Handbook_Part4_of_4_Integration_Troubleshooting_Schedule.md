# Building a Reproducible On-Prem DevOps Platform and a Tango Controls System

## PART 4 of 4 — Integration, Troubleshooting, and the Realistic Schedule

*Part 4 is where the book keeps its promise. Parts 2 and 3 each ended demonstrable but separate: a delivery platform shipping a toy package, and a control system started by a shell script. This part fuses them — the device servers get real packaging (systemd-carrying `.deb`s and OCI images), the `tango-devices` pipeline grows the full test → build → publish chain, `platform-gitops` learns the beamline's manifests, and the thesis is then demonstrated live: **one commit** changing the magnet's ramp behaviour flows through review, tests, signed artifacts, and Argo CD until an operator's trend plot draws a different slope, untouched by human hands. Around that spine: monitoring the control system itself, a debugging methodology and symptom-indexed troubleshooting compendium spanning every layer, the realistic build schedule with its honest bottlenecks, and the rehearsed twenty-minute demo that turns the whole lab into an interview answer.*

---

## How Part 4 is organised

| Chapter | Builds / delivers | Depends on |
|---|---|---|
| **1** | The integration architecture: *what deploys where, and why* — the placement decisions | Parts 2–3 complete |
| **2** | Real packaging for the device servers: a systemd-carrying `.deb`, a conda package, an OCI image | Part 2 Ch. 7, Part 3 Ch. 4–5 |
| **3** | The full CI pipeline: test → build → sign → publish, with secrets handled properly | Part 2 Ch. 3 + 8, Ch. 2 here |
| **4** | GitOps for the beamline: manifests in `platform-gitops`, host delivery via apt + Ansible | Part 2 Ch. 6 + 8 |
| **5** | **The end-to-end demonstration**: one commit → operator-visible change, narrated | Ch. 1–4 |
| **6** | Monitoring the control system itself: golden signals for device servers | Part 2 Ch. 9 |
| **7** | Debugging methodology + the troubleshooting compendium (platform / control system / integration) | everything |
| **8** | The realistic schedule (lean vs full), the risk register, and the final demo script | — |
| **Appendices** | A: HTTPS with a local CA · B: k3s as the fast alternative · C: physical (Topology B) notes | — |

Estimated effort: **1–1.5 weeks** — mostly Chapters 2–5, and mostly *plumbing you already understand*, which is precisely the point: integration should feel like assembly, not invention. If any step here requires a new mechanism, something in Parts 2–3 was skipped; go back rather than improvising forward.

**Conventions reminder** (full table in Part 1): **THEORY** → **LAB** → **Checkpoint**; `$` = your normal user; **Path A** = conda-forge, **Path B** = apt/`.deb`; commands run where stated; do not pass a failed **Checkpoint**.

---

# PART 4 — INTEGRATION, TROUBLESHOOTING, AND SCHEDULE

---

## Chapter 1 — The integration architecture: what deploys where, and why

Before wiring anything, decide — deliberately, defensibly — *where each piece of the control system should run and in which package format it should arrive*. This chapter is short but disproportionately valuable: placement decisions are exactly the "develop deployment concepts" language of the target role, and the reasoning matters more than the conclusion.

### THEORY 1.1 — The placement question, stated honestly

The platform offers two delivery lanes, built in Part 2:

- **Lane 1 — host-installed:** a signed `.deb` from the lab apt repository, installed by Ansible, run as a **systemd service** on a specific host. Properties: pinned to a machine, survives independently of the cluster, integrates with the OS (boot order, journald, local devices), upgraded deliberately via apt.
- **Lane 2 — cluster-hosted:** an **OCI image** from the registry, run as a Kubernetes **Deployment**, reconciled from Git by Argo CD. Properties: declarative, self-healing, rolling updates, placement decided by the scheduler, horizontal by nature.

Which lane for which component? The honest decision rule for control systems:

**Hardware-adjacency and statefulness pull toward the host lane.** A real device server that talks to a PCIe timing card, a serial multiplexer, or a camera-link frame grabber is *physically bound* to the machine with that hardware — scheduling flexibility is worthless and hostile there. Databases (the Tango DB's MariaDB, the HDB++ store) want stable storage and deliberate lifecycle. These belong on the `tango` host as `.deb` + systemd, delivered by apt under Ansible.

**Statelessness and fleet-shape pull toward the cluster lane.** Our *simulators* touch no hardware — they are pure computation behind a Tango interface, the textbook cluster workload. So are web-ish services, exporters, and anything you want self-healing and rolling-updatable. These belong on Kubernetes, delivered by Argo CD.

**The lab's placement, and its facility translation:**

| Component | Lane | Why |
|---|---|---|
| MariaDB + `DataBaseds` + HDB++ subscribers | host (`tango`), `.deb`/system pkgs + systemd | stateful, foundational, everything bootstraps against it |
| Sardana Pool/MacroServer, Taurus | host (`tango`) / workstation | operator-session tools, tied to the operator's seat |
| **Simulator device servers** (magnet, motor, camera, cup, gauge, interlock) | **Kubernetes**, OCI via Argo CD | stateless computation; the perfect demonstration workload |
| beamline-utils and future libraries | wheel/conda for humans, baked into images for services | consumed, not run |

**Callout — The honesty box: Tango on Kubernetes, in real life.** Running device servers on Kubernetes is a real and growing facility practice (notably for simulation, gateways, and stateless services — SKA's control system is the famous all-in example), but it is *not* uncomplicated: Tango clients connect **directly** to device servers after the DB lookup (Part 1, Ch. 4.4), so the server must advertise an address clients can actually reach — which collides with pod-network address translation. The lab's clean solution is **`hostNetwork: true`** on the device-server pods: the pod uses its node's network identity, Tango's direct connections just work, and you must be able to *explain* that trade-off (you give up network-namespace isolation and must manage port collisions — acceptable for a handful of servers, a design decision at scale, where node selectors, stable ports, and Tango's `ORBendPoint` configuration become the real engineering). Saying exactly this — pattern, trade-off, when it stops scaling — is worth more in an interview than pretending the problem doesn't exist.

### THEORY 1.2 — The finished data flow (extend Part 1's diagram; draw it once)

```
 developer ──MR + review──► GitLab (tango-devices) ──CI──► pytest (DeviceTestContext)
                                                             │ green only
                                    ┌────────────────────────┼─────────────────────┐
                                    ▼                        ▼                     ▼
                              .deb (signed) ──►  apt repo   conda pkg ──► channel  OCI image ──► registry
                                    │                                              │
                        Ansible/apt upgrade                          platform-gitops (image tag bump, MR)
                                    ▼                                              ▼
                          tango host services                    Argo CD ──reconciles──► K8s: simulator pods
                       (DataBaseds, HDB++, Sardana)                                        (hostNetwork)
                                    └────────────── one Tango system ──────────────┘
                                                        │ events
                                        Taurus panels · HDB++ → Grafana · alarms → Alertmanager
                                        Prometheus scrapes hosts + cluster + device servers (Ch. 6)
```

**Checkpoint 1 (conceptual):** for each of the seven component rows above, state the lane and the one-sentence reason; explain `hostNetwork` for Tango pods — what it buys, what it costs, and when it stops being the right answer.

---

## Chapter 2 — Packaging the device servers for real

Part 2, Chapter 7 packaged a toy in five formats by hand. Now the real payload gets the three formats its placement demands — and two of them carry something the toy never did: **operational integration** (a systemd unit inside the `.deb`; registration and configuration inside the image's entrypoint). This is the difference between "packaged" and "operable".

### THEORY 2.1 — What a *service-carrying* `.deb` adds

The toy `.deb` delivered files. A service `.deb` delivers a *running responsibility*. Debian's machinery for that, layered on what you know:

- **`debian/<pkg>.service`** — a systemd unit file shipped in the package; **`dh_installsystemd`** (invoked automatically by the `dh` sequence) installs it, enables it, starts it on install, restarts it on upgrade, and stops it on removal. The unit file is Part 1, Theory 1.6's anatomy, now written by you for your own software.
- **`debian/postinst` / `prerm`** — maintainer scripts for the moments around install/remove. Ours stays nearly empty (dh generates the service handling), but knowing where bespoke steps *would* go — creating a system user, seeding a config — is part of the literacy.
- **Configuration vs code**: the unit file reads environment from `/etc/default/<pkg>` (a Debian convention) so `TANGO_HOST` and instance names are *host configuration*, not baked into the package.

### LAB 2.1 — Restructure `tango-devices` for packaging

One repository, one Python distribution, all six device classes, console entry points per server — the shape CI will build from:

```
tango-devices/
├── pyproject.toml
├── src/beamline_devices/        # magnet_ps.py, motor_sim.py, ... (Part 3)
├── tests/                       # Part 3, Ch. 9
├── debian/                      # this chapter
├── conda-recipe/meta.yaml       # this chapter
├── Containerfile                # this chapter
└── .gitlab-ci.yml               # Chapter 3
```

```toml
# pyproject.toml (essentials)
[project]
name = "beamline-devices"
version = "1.0.0"
requires-python = ">=3.10"
dependencies = ["numpy"]          # pytango arrives via .deb Depends / conda run / image base

[project.scripts]                 # each device server becomes a real command:
MagnetPowerSupply = "beamline_devices.magnet_ps:main"
MotorSim          = "beamline_devices.motor_sim:main"
CeYAGCamera       = "beamline_devices.ceyag_camera:main"
FaradayCup        = "beamline_devices.faraday_cup:main"
VacuumGauge       = "beamline_devices.vacuum_gauge:main"
Interlock         = "beamline_devices.interlock:main"
beamline-register = "beamline_devices.register_devices:main"
```

(Each module gains a two-line `main()` wrapping `run_server()`; `register_devices.py` gains one wrapping its loop. Ten minutes of mechanical edits; commit them.)

### LAB 2.2 — The `.deb`, now with a service inside

`debian/` as in Part 2, Lab 7.4, plus the new pieces:

```
# debian/control (essentials)
Source: beamline-devices
Build-Depends: debhelper-compat (= 13), dh-python, python3-all,
               python3-setuptools, pybuild-plugin-pyproject
Package: python3-beamline-devices
Depends: ${python3:Depends}, ${misc:Depends}, python3-tango, python3-numpy
Description: LINAC lab device servers (simulated)
```

```ini
# debian/python3-beamline-devices.magnet-ps@.service
# An instance-TEMPLATE unit: one file serves q1, q2, ... as magnet-ps@<instance>
[Unit]
Description=Magnet PS device server (instance %i)
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/default/beamline-devices    # TANGO_HOST lives here, per host
ExecStart=/usr/bin/MagnetPowerSupply %i          # %i = the instance name after '@'
Restart=on-failure
RestartSec=3
User=tango                                        # a service user, not root

[Install]
WantedBy=multi-user.target
```

```makefile
# debian/rules — one flag longer than Part 2's:
#!/usr/bin/make -f
%:
	dh $@ --with python3 --buildsystem=pybuild
override_dh_installsystemd:
	dh_installsystemd --name=magnet-ps@ --no-start   # template units: enable per-instance, don't auto-start blindly
```

Note the two professional touches: a **template unit** (`@.service`) so *one* packaged file runs any number of named instances — `systemctl start magnet-ps@lab` — which is exactly how a facility runs `q1`, `q2`, `d1` from one package; and `--no-start`, because *which* instances run on a host is Ansible's decision (host configuration), not the package's. Build and prove the integration:

```bash
dpkg-buildpackage -us -uc -b
sudo apt install ../python3-beamline-devices_1.0.0-1_all.deb
echo 'TANGO_HOST=tango.lab:10000' | sudo tee /etc/default/beamline-devices
sudo systemctl start magnet-ps@lab && systemctl status magnet-ps@lab
journalctl -u magnet-ps@lab -n 20        # your device server, under journald, like any service
```

Part 1's Lab 1.3 skills, Part 2's packaging, Part 3's device — one artifact.

### LAB 2.3 — Conda recipe and the OCI image

The conda recipe is Part 2, Lab 7.3 with real dependencies:

```yaml
# conda-recipe/meta.yaml (essentials)
package: { name: beamline-devices, version: "1.0.0" }
source: { path: .. }
build:
  script: {{ PYTHON }} -m pip install . -vv
requirements:
  host: [ python, pip, setuptools ]
  run:  [ python, pytango, numpy ]
```

The image bakes the wheel onto a base that already carries PyTango (conda-forge inside the image is the pragmatic route), and its entrypoint does something worth reading twice — it makes the container **self-registering**:

```dockerfile
# Containerfile
FROM condaforge/miniforge3:latest
RUN conda install -y -c conda-forge python=3.11 pytango numpy && conda clean -afy
COPY dist/*.whl /tmp/
RUN pip install --no-cache-dir /tmp/*.whl
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
# entrypoint.sh — $1 = server class, $2 = instance name
#!/usr/bin/env bash
set -euo pipefail
beamline-register                 # idempotent (Part 3, Lab 3.2) — safe on every start
exec "$1" "$2"                    # e.g.:  MagnetPowerSupply lab
```

Idempotent registration in the entrypoint means a pod needs *nothing* pre-arranged in the Tango DB — first start creates its entries, every later start finds them. Build against the wheel (`python3 -m build` first), tag it `1.0.0`, push to `gitlab.lab:5050/beamline/tango-devices/beamline-devices:1.0.0`, and smoke-test with Podman:

```bash
podman run --rm --net=host -e TANGO_HOST=tango.lab:10000 \
  gitlab.lab:5050/beamline/tango-devices/beamline-devices:1.0.0 MagnetPowerSupply lab
```

(`--net=host` is Podman's rehearsal of the `hostNetwork: true` decision from Chapter 1 — same reasoning, same effect.)

**Checkpoint 2:** the `.deb` installs a working template systemd service that journald logs; the conda package builds; the image self-registers and serves `linac/magnet/q1` when run with host networking; all three definitions are committed. You now hold the real payload in the three formats its placement demands — built by hand once, so CI (next chapter) automates something you understand.

---
## Chapter 3 — The full pipeline: test → build → sign → publish

### THEORY 3.1 — What changes when a pipeline publishes

Part 2's pipelines tested and built. A *publishing* pipeline crosses a line: its output lands where consumers trust it. Three new concerns arrive with that step, and each has a standard answer:

1. **Secrets.** Publishing needs credentials — the GPG signing key, registry login. These must never live in the repository (the gitignored-secrets reflex from Part 2). GitLab's answer is **CI/CD variables**: values stored in project/group settings, injected into jobs as environment variables, maskable in logs and restrictable to **protected branches** — so a feature branch's pipeline *cannot* sign or publish even if compromised. The registry needs no stored secret at all: GitLab injects short-lived `CI_REGISTRY_USER`/`CI_REGISTRY_PASSWORD` into every job automatically.
2. **Placement of the publish step.** Building can happen anywhere; *publishing into the apt repo* must happen where reprepro and its key live (`svc`). The clean lab mechanism: a **second runner on `svc` with the shell executor, tagged `publish`** — the legitimate shell-executor use case Part 2, Theory 3.1 promised — and only publish jobs, only on `main`, carry that tag.
3. **When to publish.** Never from branches. The rule `only: [main]` (or its modern `rules:` equivalent) plus protected-branch variables means: feature branches test and build; **only reviewed, merged code signs and ships**. The quality gate, extended to its last inch.

Versioning discipline completes the picture: the version lives in `pyproject.toml` (single source), a release is a **Git tag** (`v1.0.1`), and the image tag equals the release version — so "what is running?" always has a Git-answerable answer. Bumping is a reviewed commit like any other.

### LAB 3.1 — Prepare the publish lane

On `svc` (fold into the `runner` role): register a second runner with `--executor shell --tag-list publish`, and give the `gitlab-runner` user what publishing needs — membership able to run `reprepro` against `~/apt-repo` and read the GPG key (lab-pragmatic: run reprepro as a small `sudo`-permitted wrapper or share the keyring with the runner user; note in `versions.md` which you chose — in production this is an HSM/signing-service conversation, and saying so is the right interview move). In GitLab: Settings → CI/CD → Variables — nothing needed for the registry; add `CONDA_CHANNEL_DIR=/var/www/html/conda` for tidiness.

### LAB 3.2 — The pipeline, in full

```yaml
# tango-devices/.gitlab-ci.yml
stages: [test, build, publish]

# ---------- STAGE: test (every push, every branch) ----------
device-tests:
  stage: test
  tags: [lab]
  image: condaforge/miniforge3:latest
  script:
    - conda create -n ci -c conda-forge python=3.11 pytango pytest -y
    - conda run -n ci pip install -e .
    - conda run -n ci pytest tests/ -v

# ---------- STAGE: build (every push; artifacts kept) ----------
build-wheel:
  stage: build
  tags: [lab]
  image: python:3.11
  script:
    - pip install build && python -m build
  artifacts: { paths: ["dist/*.whl"] }

build-deb:
  stage: build
  tags: [lab]
  image: ubuntu:22.04
  script:
    - apt-get update && apt-get install -y devscripts debhelper dh-python
        python3-all python3-setuptools pybuild-plugin-pyproject
    - dpkg-buildpackage -us -uc -b
    - mv ../*.deb .                      # artifacts must live inside the workspace
  artifacts: { paths: ["*.deb"] }

build-conda:
  stage: build
  tags: [lab]
  image: condaforge/miniforge3:latest
  script:
    - conda install -y conda-build
    - conda build conda-recipe/ -c conda-forge --output-folder conda-out/
  artifacts: { paths: ["conda-out/"] }

build-image:
  stage: build
  tags: [lab]
  image: quay.io/podman/stable            # podman-in-CI: rootless build
  needs: [build-wheel]                    # consumes the wheel artifact
  script:
    - podman build -t $CI_REGISTRY_IMAGE/beamline-devices:$CI_COMMIT_SHORT_SHA .
    - podman login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY --tls-verify=false
    - podman push --tls-verify=false $CI_REGISTRY_IMAGE/beamline-devices:$CI_COMMIT_SHORT_SHA
    # on tagged releases, also push the version tag:
    - if [ -n "$CI_COMMIT_TAG" ]; then
        podman tag $CI_REGISTRY_IMAGE/beamline-devices:$CI_COMMIT_SHORT_SHA
                   $CI_REGISTRY_IMAGE/beamline-devices:${CI_COMMIT_TAG#v};
        podman push --tls-verify=false $CI_REGISTRY_IMAGE/beamline-devices:${CI_COMMIT_TAG#v};
      fi

# ---------- STAGE: publish (main only; the svc shell runner) ----------
publish-apt:
  stage: publish
  tags: [publish]                        # routes to svc's shell runner
  rules: [{ if: '$CI_COMMIT_BRANCH == "main"' }]
  script:
    - reprepro -b ~/apt-repo includedeb jammy *.deb    # reprepro signs with the repo key

publish-conda:
  stage: publish
  tags: [publish]
  rules: [{ if: '$CI_COMMIT_BRANCH == "main"' }]
  script:
    - cp conda-out/noarch/*.conda ${CONDA_CHANNEL_DIR}/noarch/ 2>/dev/null ||
      cp conda-out/linux-64/*.conda ${CONDA_CHANNEL_DIR}/linux-64/
    - python -m conda_index ${CONDA_CHANNEL_DIR}
```

Read the shape before pushing it: **every branch** runs tests and builds (so an MR proves the artifacts *can* be made); **only `main`** reaches the publish stage and the signing key; the **image build consumes the wheel artifact** (`needs:`) rather than rebuilding — one source of binary truth per pipeline; and `--tls-verify=false` is the Chapter-2-of-Part-2 HTTP simplification surfacing again, with Appendix A as its cure. Push, watch all six jobs go green, then verify like a consumer: `apt update && apt policy python3-beamline-devices` on `tango` shows the CI-published version; the registry UI shows the SHA-tagged image.

**Callout — Pitfall.** `build-deb` red with "artifacts not found": `dpkg-buildpackage` writes to the *parent* directory — the `mv ../*.deb .` line is load-bearing. Publish jobs stuck `pending`: the `publish`-tagged runner isn't registered/online (`gitlab-runner list` on `svc`). Conda publish copying nothing: check whether your build landed in `noarch` vs `linux-64` — pure-Python recipes can go either way depending on recipe flags.

**Checkpoint 3:** a push to a branch runs test + all four builds, green, publishing nothing; a merge to `main` additionally publishes — the `.deb` installable from `repo.lab`, the conda package from the channel, the image pulled by SHA from the registry. Tag `v1.0.0` and confirm the version-tagged image appears. The pipeline is now the only path artifacts take; retire the by-hand builds.

---

## Chapter 4 — GitOps for the beamline

### THEORY 4.1 — Two lanes, one Git discipline

Chapter 1 placed components in two lanes; both now get their delivery wired, and the point to hold onto is that **both lanes are Git-driven** — they differ only in agent:

- **Cluster lane:** manifests in `platform-gitops`, reconciled by Argo CD (Part 2, Ch. 6 verbatim — new payload, same machinery). Deploying a new version *is* a one-line image-tag bump, made as a reviewed MR.
- **Host lane:** the `tango` role in the platform repo declares which packages (from your repo) and which service instances a host runs; Ansible reconciles. Upgrading the archiver host *is* `apt upgrade` under Ansible control — also a Git-visible act, because the role and the pinned version live in the repository.

If someone asks "how do you deploy?", the complete answer is now one sentence: *"Everything is a reviewed commit — manifests for the cluster, roles for the hosts; agents reconcile; nothing is ever hand-applied."*

### LAB 4.1 — The beamline manifests

In `platform-gitops`, one manifest per simulator (shown for the magnet; the other five are the same shape — and yes, write all six, the repetition is the fluency):

```yaml
# platform-gitops/manifests/beamline/magnet-ps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: magnet-ps-lab
  namespace: beamline
spec:
  replicas: 1                          # a device server is an identity, not a farm — see callout
  selector: { matchLabels: { app: magnet-ps-lab } }
  template:
    metadata: { labels: { app: magnet-ps-lab } }
    spec:
      hostNetwork: true                # the Chapter 1 decision, enacted
      containers:
        - name: magnet-ps
          image: gitlab.lab:5050/beamline/tango-devices/beamline-devices:1.0.0
          args: ["MagnetPowerSupply", "lab"]      # entrypoint: register, then exec
          env:
            - name: TANGO_HOST
              value: "tango.lab:10000"
          resources:
            requests: { cpu: 50m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }
```

**Callout — Why `replicas: 1`, always, for a device server.** A Deployment usually earns its keep by scaling replicas; a Tango device server must not — two pods both exporting `linac/magnet/q1` is a split-brain identity error, not high availability. What the Deployment buys here is *supervision* (restart on crash, reschedule on node death — the systemd `Restart=` idea, cluster-grade), not scale. Knowing which Kubernetes reflexes to *suppress* for control systems is exactly the judgment the role wants.

Before committing, do the migration hygiene: stop the shell-script servers from Part 3, Lab 5.5 on `tango` (the cluster owns the simulators now; the DB, HDB++, and Sardana stay host-side per Chapter 1). Then commit, push, and let Argo CD do what it does — `kubectl -n beamline get pods` fills with six simulators, each self-registering on start. Run the Part 3 smoke test unchanged: the Taurus panel, spock's `ascan`, HDB++ — none of them can tell anything moved. *That* is the device-model uniformity promise, now demonstrated across a deployment-architecture change.

### LAB 4.2 — The host lane, closed properly

Extend the `tango` role: the lab apt source + key (Part 2, Lab 8.1's client steps, as tasks), `python3-beamline-devices` pinned `state: present` with a version variable, `/etc/default/beamline-devices` via `template:`, and the *choice of instances* as data:

```yaml
# roles/tango/defaults/main.yml
beamline_devices_version: "1.0.0-1"
host_device_instances: []          # the simulators moved to the cluster; the hook remains

# roles/tango/tasks/main.yml (added)
- name: Enable configured device-server instances
  systemd: { name: "magnet-ps@{{ item }}", enabled: true, state: started }
  loop: "{{ host_device_instances }}"
```

`vagrant provision` — and now a *hardware-bound* device server, the day one exists, is one line in a host var file and a reviewed MR away. The lane is proven even while it carries only the databases and Sardana.

**Checkpoint 4:** all six simulators run as Argo CD-managed pods, self-registered, indistinguishable to every Part 3 client; the shell script is deleted from the repo (with a commit message saying why); the `tango` role installs from your signed repo and can enable template-unit instances from data. Both lanes answer to Git.

---

## Chapter 5 — The end-to-end demonstration: one commit, narrated

### THEORY 5.1 — Why this chapter is the point of the whole book

Everything before this was capability; this is the *claim*. The pitch at the centre of the CV and cover letter is not "I installed GitLab" or "I wrote device servers" — it is: **the same pipeline that ships any control-system component ships this one**: review, tests, signed multi-format artifacts, GitOps deployment, observability, one loop, no hands. Interviewers probe exactly here — *"walk me through what happens when you change a line of code"* — and the difference between a described loop and a **performed** loop is audible within one sentence. So: perform it, time it, record it.

### LAB 5.1 — The run

**The change.** Something operator-visible and physically meaningful: the magnet's default slew. In `magnet_ps.py`:

```python
    ramp_rate = device_property(dtype=float, default_value=5.0)   # before
    ramp_rate = device_property(dtype=float, default_value=8.0)   # after
```

*(Keep the test suite honest: Part 3's tests inject `ramp_rate` explicitly, so they still pass — a default changed, a contract didn't. If a test had pinned the default, this MR would rightly force that conversation. Say that in the narration; it's the gate thinking.)*

**The loop, step by step — narrate as you go:**

1. **Branch + MR.** `git switch -c feat/faster-default-ramp`, commit with a message that explains *why* ("operations requested snappier default slew for lab magnets; hardware limit is 10 A/s"), push, open the MR.
2. **The gate.** CI runs: device tests, four builds. Green. (Optional but excellent theatre: first push a version with a sign error in the ramp, show the red MR and the blocked merge button, then push the fix — thirty extra seconds that *show* the gate rather than assert it.)
3. **Review + merge.** Merge to `main`. The publish stage fires: new `.deb` into the signed repo, conda into the channel, image pushed. Bump `version = "1.0.1"` in this MR (or as its sibling), tag `v1.0.1` → the registry now holds `beamline-devices:1.0.1`.
4. **The deployment commit.** In `platform-gitops`, one line: `image: ...beamline-devices:1.0.0` → `:1.0.1`. MR, review, merge. This commit *is* the deployment record — author, reason, diff, timestamp.
5. **Reconciliation.** Argo CD notices, rolls `magnet-ps-lab`; the new pod's entrypoint re-registers (idempotently, finding everything already there) and exports `linac/magnet/q1` from version 1.0.1. Watch it in the Argo CD UI — the rolling replacement is visible, and worth showing.
6. **The operator's view.** The Taurus panel (Part 3, Ch. 7). `Reset()`, then write `setpoint = 50`. The trend plot climbs at **8 A/s — ~6 seconds to target instead of ten.** Before/after, on one screen, caused by a reviewed commit and nothing else.

**The clock.** Time the whole run, commit-to-visible-change. In the lab it lands in the **10–20 minute** range, dominated by CI. Write the number in `versions.md` next to the rebuild time from Part 2 — you now own two concrete, measured claims: *the platform rebuilds from Git in N minutes; a code change reaches operators in M.*

**The recording.** Capture steps 2–6 as the definitive screen recording (~4 minutes edited): red-then-green MR, publish jobs, the one-line GitOps diff, Argo CD rolling, the slope changing. Paired with Part 3's panel recording, this is the interview artifact.

### LAB 5.2 — Prove the loop's other half: rollback

A delivery story is half a story without retreat. In `platform-gitops`: `git revert HEAD && git push` — Argo CD rolls the pod back to `1.0.1 → 1.0.0`; the panel ramps at 5 A/s again; the revert is itself an audited commit. Total elapsed: about ninety seconds, no artifacts rebuilt (that's the point — rollback re-selects a *published, immutable* version, it doesn't re-create one). Then roll forward again the same way.

**Checkpoint 5:** the loop has been run at least twice end-to-end and once in reverse; the elapsed time is measured and recorded; the screen recording exists and has been watched critically once ("would this convince me?"). **This checkpoint is the book's go/no-go milestone** — Chapter 8's schedule is built around reaching it.

---

## Chapter 6 — Monitoring the control system itself

Part 1's architecture narration promised "Prometheus scrapes every layer — hosts, cluster, *and the control system itself*." Hosts and cluster were Part 2; this short chapter keeps the third promise, and doing so closes a conceptual loop: the Golden Signals apply to device servers exactly as to web services, once you say what they mean here.

### THEORY 6.1 — Golden Signals, translated to a beamline

| Signal | Web-service meaning | Device-server meaning |
|---|---|---|
| **Latency** | request duration | attribute read/command round-trip time — a slow device server stalls GUIs and scans |
| **Traffic** | requests/second | attribute reads + events emitted per second — the archiver's appetite made visible |
| **Errors** | 5xx rate | `DevFailed` exceptions per minute; devices in `FAULT`/`ALARM` state |
| **Saturation** | CPU/queue depth | the process's CPU/memory (a runaway simulator tick), event-queue backlogs, DB connections |

Two implementation routes, in order of effort: **(a) black-box now** — a tiny exporter that *probes* devices from outside (state + a timed read per device), which is genuinely useful and one lab long; **(b) white-box later** — instrument the device classes themselves with `prometheus_client` counters/histograms inside read/write/command handlers, the production-grade route and a natural "what I'd build next".

### LAB 6.1 — A 40-line Tango exporter (and everything it rides on)

```python
# src/beamline_devices/tango_exporter.py
"""Black-box Tango probe → Prometheus metrics on :9105."""
import time
import tango
from prometheus_client import Gauge, start_http_server

DEVICES = ["linac/magnet/q1", "linac/motion/fc-actuator", "linac/camera/ceyag1",
           "linac/beam/faradaycup1", "linac/vacuum/gauge1", "linac/safety/interlock1"]

up      = Gauge("tango_device_up", "1 if reachable", ["device"])
state   = Gauge("tango_device_state", "DevState enum value", ["device"])
latency = Gauge("tango_device_read_seconds", "state() round-trip", ["device"])

def probe():
    for name in DEVICES:
        try:
            t0 = time.time()
            s = tango.DeviceProxy(name).state()
            latency.labels(name).set(time.time() - t0)
            state.labels(name).set(int(s))
            up.labels(name).set(1)
        except tango.DevFailed:
            up.labels(name).set(0)

def main():
    start_http_server(9105)
    while True:
        probe(); time.sleep(5)

if __name__ == "__main__":
    main()
```

Now watch how much existing machinery this 40-line file rides for free — this paragraph *is* the platform's value proposition: it ships in the **same package** (one `pyproject.toml` entry point, `tango-exporter`), through the **same pipeline** (nothing to add), deploys as a **seventh manifest** in `platform-gitops` (ordinary pod network is fine — it's a client), gets scraped via the **additional-scrape-config** pattern from Part 2, Lab 9.2 (or a `ServiceMonitor` if you route it through a Service), and alerts through the **same Alertmanager**:

```yaml
# added to platform-gitops/manifests/golden-signals-rules.yaml
- alert: TangoDeviceDown
  expr: tango_device_up == 0
  for: 2m
  labels: { severity: critical }
  annotations:
    summary: "Tango device {{ $labels.device }} unreachable"
    runbook: "http://gitlab.lab/beamline/platform-gitops/-/blob/main/docs/runbook.md#tango-device-down"

- alert: TangoDeviceFault
  expr: tango_device_state == 8       # DevState.FAULT
  for: 1m
  labels: { severity: warning }
```

Fire it for real: kill the magnet pod's deployment via a gitops commit (or just `kubectl scale --replicas=0` and let the demo double as a self-heal reminder), watch `TangoDeviceDown` reach Alertmanager, add the runbook section it points at. Grafana gets one small "Beamline health" row: `tango_device_up` as a status grid, read latency as a graph.

**Checkpoint 6:** every device's reachability, state, and probe latency are live in Grafana; killing a device server pages within ~2 minutes with a runbook link; the exporter reached production through the standard loop — which you can now say about *anything*.

---
## Chapter 7 — Debugging methodology and the troubleshooting compendium

### THEORY 7.1 — How to debug a layered system (the method behind the tables)

The compendium below is symptom-indexed, but tables don't debug — a method does. Three habits, each earned somewhere in this book:

**1. Locate the layer first.** The stack has clean seams: *host → service/process → network → framework (Tango/K8s/GitLab) → application code*. Every symptom belongs to a layer, and each layer has a canonical first question — is the host up and resourced (`free -h`, `df -h`)? is the process running and what do its logs say (`systemctl status`, `journalctl -u`, `kubectl logs`)? can A reach B:port (`ping`, `ss -tlnp`, is ufw's rule there)? does the framework know the object (`tango_admin --check-device`, `kubectl describe`, runner list)? only *then* read your own code. Ninety percent of "the code is broken" turns out to live two layers down.

**2. Half-split the pipeline.** For end-to-end failures ("my commit didn't reach the operator"), don't walk the loop start to finish — cut it in the middle: *is the artifact in the registry/repo?* If yes, the problem is deployment-side (gitops repo, Argo CD, pod); if no, it's build-side (pipeline, runner, packaging). One question eliminates half the system; recurse.

**3. Reproduce before you fix, and record after.** A fix you can't reproduce the failure for is a superstition. And every diagnosis that took you more than ten minutes earns a row in the runbook — that discipline is why `docs/runbook.md` exists and why the compendium below could be written at all.

**Callout — Why a compendium instead of scattering these into each chapter.** During an outage or a failing demo you reach for a *symptom-indexed table*, not a memory of chapter order. This chapter is the CV's "documented host-troubleshooting runbook" grown to cover the whole system — and Checkpoint 7 makes you *use* it, because a runbook that has never been exercised is a hypothesis.

### The compendium

*Each entry: symptom → most likely cause → check first. Layer-sorted within each table.*

#### Platform (Part 2)

| Symptom | Likely cause | Check first |
|---|---|---|
| `vagrant up` fails on VT-x/AMD-V | virtualization disabled in firmware, or VirtualBox/KVM contending | `egrep -c '(vmx\|svm)' /proc/cpuinfo`; unload the other hypervisor's modules |
| Ansible: host unreachable | SSH/key/inventory IP mismatch | `ansible <host> -m ping`; `vagrant ssh <host>` directly |
| A service is unreachable that "should" work | ufw default-deny; the role never opened the port | `sudo ufw status` on the target — the Part 2, Ch. 1 trap, forever |
| GitLab answers 502 just after provisioning | still reconfiguring, or `svc` RAM-starved | wait 2–3 min; `gitlab-ctl status`; `free -h` (GitLab wants its 6 GB) |
| CI job `pending` forever | no online runner matches the job's tags | `gitlab-runner list` on `svc`; compare `tags:` in the job vs runner registration |
| Job fails pulling its image | registry unreachable from executor, or TLS/insecure config missing | `podman pull` the image by hand on `svc`; check `registries.conf.d` insecure entry |
| `dpkg-buildpackage`: missing build-dep | `debian/control` Build-Depends incomplete | install the named package, re-run; keep control in sync as imports grow |
| conda build green locally, red in CI | channel flags differ from local `~/.condarc` | make `-c conda-forge` (and any others) explicit in the CI command |
| `reprepro includedeb` rejects the package | wrong distribution codename, or key/keyring mismatch | codename in `conf/distributions` vs command; `gpg --list-keys` as the runner user |
| Client `apt update` fails on your repo | key not installed, or signed-by path typo | re-run the two client setup lines; `apt-key` era leftovers conflict — use keyrings only |
| kubeadm init hangs / kubelet flapping | swap on, or cgroup driver mismatch | `swapon --show` (must be empty); `SystemdCgroup = true` in containerd config |
| Nodes `NotReady` | CNI not up, or pod CIDR mismatch, or flannel picked the NAT NIC | `kubectl -n kube-flannel get pods`; CIDR = 10.244.0.0/16; pin `--iface=eth1` |
| Pod `Pending` forever | insufficient node resources (reduced lab!), or no schedulable node | `kubectl describe pod` → Events section first, always |
| Pod `ImagePullBackOff` | registry auth/TLS from *nodes*, or tag typo | `kubectl describe pod`; nodes need the insecure-registry containerd config too |
| Argo CD `OutOfSync` forever | immutable field changed, or app watching wrong path/revision | `argocd app diff`; some changes need delete+recreate — a known K8s sharp edge |
| Argo CD can't reach the repo | private repo without credentials, or name resolution in-cluster | repo URL reachable from a debug pod; deploy token registered |
| Prometheus target absent/down | scrape config missing, port closed, exporter dead | curl the `/metrics` URL from where Prometheus runs; then the ufw question again |
| Grafana panel blank though data exists | wrong datasource UID, or time range | edit panel → run query raw; check dashboard's datasource binding |
| Alert never fires though condition true | `for:` window not yet elapsed, or rule not loaded (label selector) | Prometheus UI → Alerts (pending?); rule has `release: kps` label |

#### Control system (Part 3)

| Symptom | Likely cause | Check first |
|---|---|---|
| No client can find any device | `TANGO_HOST` unset/wrong — the number-one bug | `echo $TANGO_HOST`; is `DataBaseds` listening on that host:port (`ss -tlnp`) |
| `tango_admin --check-device` times out | server process not running, or crashed at start | is the process alive; run it in the **foreground** and read the traceback |
| Device exists but state `UNKNOWN`/errors | exception inside `init_device()` | foreground run again — `init_device` tracebacks are otherwise swallowed into logs |
| Attribute write silently ignored | attribute declared `READ`, not `READ_WRITE` | the `AttrWriteType` in the class — check before suspecting anything deeper |
| Write rejected unexpectedly | `min_value`/`max_value` doing their job | `attribute_query(<attr>)` shows the configured limits — maybe they're right |
| Alarm never triggers though value is wild | only `min/max_value` set, not `min/max_alarm` | they are *distinct* — limits gate writes; alarms judge reads. Set the `_alarm` pair |
| Device never leaves `MOVING` | background task crashed mid-ramp (exception in the coroutine) | foreground logs; wrap the task body's core in try/except-log during diagnosis |
| Events not reaching subscribers | `set_change_event` not declared, or event ports blocked between hosts | the `init_device` declaration; ufw range from Part 3, Lab 2.3 on *both* ends |
| Properties ignored after a change | device never re-initialised | `Init()` on the proxy — properties load in `init_device`, not continuously |
| Sardana `ascan` hangs on point 1 | controller's `StateOne` never reports done | poll the underlying device's `state()` yourself — does it actually reach `ON`? |
| spock can't find motors/counters | Pool elements not defined (fresh Pool) | replay the `defctrl`/`defelem` setup script — Pool config is DB state, rebuild it |
| Taurus widget shows `---` | model name typo, or device down | `tango_admin --check-device <name>` *before* suspecting Taurus |
| HDB++: attribute has no history | added but never started, or subscriber down | `AttributeStart` issued? is `hdbpp-es` alive? then look for rows in SQL directly |
| `DeviceTestContext` test hangs | ramp task never resolves within the wait | deadline-poll (not fixed sleep); longer-term: the injectable clock, as flagged |

#### Integration (Part 4)

| Symptom | Likely cause | Check first |
|---|---|---|
| Publish jobs `pending` on `main` | the `publish`-tagged shell runner offline | `gitlab-runner list` on `svc` |
| `.deb` published but client sees old version | apt cache, or reprepro replaced vs added confusion | `apt update` then `apt policy <pkg>`; `reprepro list jammy` on `svc` |
| Pod runs but device unreachable via Tango | pod not on `hostNetwork`, so its advertised address is unroutable | the `hostNetwork: true` line — Chapter 1's whole discussion, in one symptom |
| Pod restarts loop: registration errors | Tango DB down, or `TANGO_HOST` env missing in manifest | `kubectl logs` — the entrypoint prints registration output first |
| Two pods fighting over one device | replicas > 1, or an old shell-script server still running on `tango` | `kubectl get deploy` replicas; `ps aux | grep -i magnet` on the tango host |
| Image tag bumped, nothing rolled | Argo CD synced but the tag didn't change (typo), or app OutOfSync elsewhere | `argocd app diff`; the UI's resource tree shows the live image tag |
| Rollback "didn't work" | reverted the code repo instead of the gitops repo | rollback = revert in **platform-gitops**; the code repo revert only matters at next release |
| Exporter shows all devices down | exporter's own `TANGO_HOST`/network, not the devices | can the exporter pod resolve and reach `tango.lab:10000`? probe from a debug pod |
| Demo loop "slow" | CI queue on the single runner — builds serialised | a second `lab` runner is one Ansible loop away; also `needs:` already parallelises |

**Checkpoint 7:** pick any **three rows** across the tables, deliberately reproduce each symptom in the lab (unset `TANGO_HOST`; set `max_value` but not `max_alarm` and watch the alarm not fire; scale a simulator to 2 replicas and observe the identity fight), confirm the "check first" step surfaces the cause — then add one row of your *own*, from a real diagnosis this build gave you, with a commit to `docs/runbook.md`.

---

## Chapter 8 — The realistic schedule, the risk register, and the final demo

### THEORY 8.1 — Where the time actually goes

Honest bottlenecks, in descending order of hours consumed, from this build:

1. **The simulator device servers with believable physics** (Part 3, Ch. 4–5) — not because any one is hard, but because six devices × state machines × events × testing is genuine engineering volume;
2. **The five-format packaging chain, especially Debian** (Part 2, Ch. 7; realised in Part 4, Ch. 2–3) — debhelper's learning curve front-loads; it amortises fast;
3. **kubeadm cluster networking** (Part 2, Ch. 5) — the three prerequisites plus the dual-NIC traps; budget for one frustrating evening and be pleased if you don't spend it;
4. **HDB++ schema and archiving wiring** (Part 3, Ch. 8) — version/property alignment across CM, ES, and schema is fiddly;
5. everything else moves *faster* than intuition suggests, because it is configuration of mature, well-documented tools — GitLab, Argo CD, Prometheus install in minutes and mostly just work.

Two scheduling principles fall out. **Front-load the bottlenecks you can parallelise around** (start the kubeadm evening early in a week, with GitLab work as the fallback task). **Protect the integration week** — Chapter 5's loop is the go/no-go milestone, and everything after it is enhancement; everything before it is prerequisite.

### The lean build — interview-ready in ~3.5–4 weeks

*(part-time evenings + weekends pace; halve it for full-time)*

| Week | Focus | Hard deliverable (demo-able in isolation) |
|---|---|---|
| **1** | Part 1 fully; Part 2 Ch. 1–3 | five hosts from `vagrant up`; a pushed commit runs a CI job on your own runner |
| **2** | Part 2 Ch. 4–6 + Ch. 7–8 *lean* (Debian format only) + Ch. 9 basics | a `.deb` built, **signed, published, and installed by name** on a clean host; Argo CD self-heals a deleted deployment; Grafana shows hosts |
| **3** | Part 3 Ch. 1–5 *lean* (magnet + vacuum + interlock; skip camera/cup if pressed) + Ch. 7 + Ch. 9 | live Taurus screen: a ramping magnet and a tripping alarm; device tests green in CI |
| **3.5–4** | Part 4 Ch. 2–5 (one format lane is fine: `.deb` + image); troubleshooting notes; **the recording** | **the commit-to-operator loop, run, timed, and recorded** |

The lean path **deliberately defers** — and you should be able to recite what and why: conda/pip/Apptainer formats (Debian alone proves the chain; the others are additive), Sardana and HDB++ (Taurus + built-in alarms carry the demo; scans and archiving deepen it), the cross-device alarm→Alertmanager routing (per-attribute alarms suffice initially), and Chapter 6's exporter. Deferred ≠ dropped: each is a named "what I'd build next", which is itself a good interview answer.

### The full build — ~6–8 weeks

Adds, atop lean: all five packaging formats and both signed repos (Part 2, Ch. 7–8 complete); the full six-device beamline with the camera image in Taurus; Sardana scans (Part 3, Ch. 6); HDB++ → Grafana and the alarm → Alertmanager route (Ch. 8); the exporter and beamline-health dashboard (Part 4, Ch. 6); the complete test suite across all devices; and both appendix upgrades if appetite remains (TLS being the more valuable).

**Callout — Why two numbers instead of one.** The lean estimate is what it takes to walk into an interview with a *genuine, performed, recorded* artifact rather than a description of one. The full estimate is the honest continuation — deeper Sardana/HDB++ competency is directly useful *on the job*, not just for getting it. Commit to the lean path with a start date; hold the full path as the answer to "what would you build next?"

### THEORY 8.2 — The risk register (know your soft spots before someone finds them)

A build this size has known weak points; naming them *first* is a strength move. The five an informed interviewer might probe, with the honest answer for each:

| Probe | Honest answer |
|---|---|
| "It's all HTTP?" | Lab simplification, consciously taken; Appendix A is the local-CA upgrade path; production is TLS + real PKI, and the insecure-registry flags are exactly where it surfaces. |
| "Your interlock is software." | Correct — and deliberately framed as supervision only; real protection is hardware SIS/PLC beneath the control system. (Part 3, Ch. 5's honesty box, verbatim.) |
| "Tests sleep on wall-clock physics." | Known; deadline-polling mitigates; the injectable-clock refactor is the named fix. |
| "Single runner, single DB, no HA anywhere." | Lab scale by design; the architecture points (more runners = one loop; DB backup/replication; multi-CP kubeadm) are understood and would be the first production deltas. |
| "Secrets handling is thin." | Gitignored vars + protected CI variables in the lab; production wants a vault and a signing service/HSM — named, not hand-waved. |

### The final demo: twenty minutes, three acts

Assemble the three recorded/rehearsed pieces into one arc — this is the artifact the whole handbook was for:

- **Act I — The platform (Part 2's script, compressed to 6 min):** rebuildability claim with your measured number; the red-then-green MR; `apt install` from your signed repo with the tamper line.
- **Act II — The payload (Part 3's script, 6 min):** the whiteboard device model; the ramping Taurus panel; the leak trip with the hardware-interlock honesty sentence; the `ascan` peak.
- **Act III — The loop (Part 4, 6 min):** the one-commit demonstration, before/after slope, the 90-second rollback, and the closing sentence: *"every layer you've seen — hosts, platform, control system, monitoring — rebuilds from the Git repositories I've shown you, and nothing reaches an operator except through the gate."*
- **+2 min held for the risk-register questions you now hope they ask.**

**Checkpoint 8:** a calendar block exists for Week 1 with a specific start date; Chapter 5's loop is written down as the go/no-go milestone; the three-act demo has been rehearsed aloud at least once end-to-end, with the clock running.

### CV-claim traceability (every claim, backed by a checkpoint)

The handbook's final quality gate: every technical claim on the CV maps to something you have *performed*, with the chapter that proves it. Walk this table once before any interview; if a row feels thin, that chapter's checkpoint is your revision plan.

| CV / cover-letter claim | Where it became true | The 20-second proof you can offer |
|---|---|---|
| "Infrastructure as Code — idempotent, rebuildable environments (Vagrant, Ansible)" | P2 Ch. 1 | the `changed=0` second run; the measured destroy/rebuild time in `versions.md` |
| "Self-hosted GitLab as single source of truth" | P2 Ch. 2 | group/projects tour; SSH push; "the forge lives inside the trust boundary" |
| "GitLab CI on on-prem runners, gated by code review and tests" | P2 Ch. 3; P3 Ch. 9 | the red-MR-blocks-merge ritual, performed for platform *and* device code |
| "Multi-format packaging: `.deb` (debhelper/dh-python), Conda, Pip, OCI, Apptainer" | P2 Ch. 7; P4 Ch. 2 | one source → five artifacts; the template systemd unit inside the `.deb` |
| "Self-hosted, GPG-signed apt/Conda registry" | P2 Ch. 8 | `apt install` by name on a clean host; the one-byte tamper rejection |
| "Argo CD reconciling an on-prem Kubernetes (kubeadm) cluster from Git" | P2 Ch. 5–6; P4 Ch. 4 | delete a managed deployment, watch it return; `git revert` as rollback |
| "Prometheus/Grafana/Alertmanager, Golden-Signals alerting, documented runbook" | P2 Ch. 9; P4 Ch. 6 | the stress-fired alert with its runbook link; the Tango exporter row |
| "Distributed Tango/PyTango device-server codebase for a 6–20 MeV LINAC beamline" | P3 Ch. 3–5 | the whiteboard model + the ramping panel + the interlock trip |
| "Sardana-orchestrated motion / scans" | P3 Ch. 6 | `ascan` tracing the physical peak near 35 A |
| "Operator GUIs (Taurus), HDB++ archiving, alarm handling" | P3 Ch. 7–8 | the panel recording; the Grafana beam-history dashboard; alarm → Alertmanager |
| "Shipped the codebase through the DevOps platform for reproducible builds" | **P4 Ch. 5** | the timed, recorded commit-to-operator loop — the book's thesis, performed |

### Self-assessment (answer without notes; section references in brackets)

1. State the placement rule: what pulls a component toward the host lane, and what toward the cluster lane? *(1.1)*
2. Why does a Tango device server on Kubernetes need `hostNetwork: true`, what does it cost, and when does the pattern stop scaling? *(1.1)*
3. What does a *service-carrying* `.deb` add over a file-carrying one — name the three Debian mechanisms involved. *(2.1)*
4. Why is the systemd unit a *template* (`@.service`), and why `--no-start` at install? *(Lab 2.2)*
5. What does the OCI image's entrypoint do before exec-ing the server, and what property of the registration script makes that safe? *(Lab 2.3)*
6. Name the three concerns that arrive when a pipeline starts *publishing*, and the standard answer to each. *(3.1)*
7. Why does the publish stage run on a shell-executor runner on `svc`, and why is that the legitimate shell-executor case? *(3.1)*
8. How do protected branches + `rules:` enforce "only reviewed code signs and ships"? *(3.1, Lab 3.2)*
9. Why does `build-image` declare `needs: [build-wheel]` instead of rebuilding the wheel? *(Lab 3.2)*
10. Describe both delivery lanes in one sentence each, and the sentence that unifies them. *(4.1)*
11. Why must a device-server Deployment keep `replicas: 1`, and what is the Deployment buying if not scale? *(Lab 4.1)*
12. After migrating the simulators to the cluster, why could no Part 3 client tell the difference — and which promise did that demonstrate? *(Lab 4.1)*
13. Walk the six steps of the commit-to-operator loop from memory, naming the repository each step touches. *(Lab 5.1)*
14. Why does rollback take ninety seconds and rebuild nothing? *(Lab 5.2)*
15. In the rollback, which repository gets the `git revert` — and what happens if you revert the other one? *(Lab 5.2, compendium)*
16. Translate the four Golden Signals into device-server terms. *(6.1)*
17. Black-box vs white-box monitoring of device servers: what does each mean, and which did the lab build first, why? *(6.1)*
18. List everything the 40-line exporter reused from the existing platform. *(Lab 6.1)*
19. Recite the layer ladder of debugging habit 1 and the canonical first question at each rung. *(7.1)*
20. Apply half-splitting to "my commit didn't reach the operator" — what is the one question that halves the system? *(7.1)*
21. Pick three compendium rows you reproduced for Checkpoint 7 — symptom, cause, first check, from memory. *(Ch. 7)*
22. Rank the build's four honest bottlenecks and say what makes each expensive. *(8.1)*
23. What does the lean path defer, and why is "deferred, and here's why" a better interview position than silently omitting? *(8.1)*
24. Answer any two risk-register probes in your own words — TLS, interlock, tests, HA, or secrets. *(8.2)*

### Glossary additions (extends Parts 1–3; completes the handbook glossary)

| Term | Definition |
|---|---|
| **black-box / white-box monitoring** | probing a system from outside (reachability, timed reads) vs instrumenting its internals (counters in handlers) |
| **CI/CD variables (GitLab)** | server-stored secrets injected into jobs as env vars; maskable, restrictable to protected branches |
| **deployment record** | the gitops commit that changed an image tag — author, reason, diff, timestamp; the auditable "what shipped when" |
| **`dh_installsystemd`** | the debhelper step that installs, enables, and lifecycle-manages a unit file shipped in a `.deb` |
| **half-splitting** | debugging a pipeline by testing its midpoint, eliminating half the system per question |
| **`hostNetwork: true`** | pod uses its node's network identity — the lab's answer to Tango's direct client↔server connections on K8s |
| **lane (delivery)** | this book's term for a delivery path: host lane (`.deb` + Ansible + systemd) vs cluster lane (OCI + Argo CD) |
| **`/etc/default/<pkg>`** | the Debian convention for a service's environment file — host configuration, outside the package |
| **maintainer scripts** | `postinst`/`prerm` etc.: hooks a `.deb` runs at install/remove moments |
| **protected branch** | a branch (e.g. `main`) with restricted push/merge rights and access to protected CI variables |
| **risk register** | the named list of a system's known weaknesses with honest positions on each — offense, not confession |
| **self-registering container** | an image whose entrypoint idempotently creates its own Tango DB entries before exec-ing the server |
| **template unit (`name@.service`)** | one systemd unit file serving many named instances (`magnet-ps@q1`, `@q2`) |
| **traceability (CV)** | the claim → chapter → performable-proof mapping that turns a résumé line into a demonstration |

---

## Appendix A — HTTPS with a local CA (the promised upgrade path)

The lab ran HTTP everywhere for focus; here is the shape of the fix, as a half-day project. **1.** Generate a lab certificate authority (`openssl req -x509 -newkey` for a CA key + self-signed root cert, 10-year validity; or use `mkcert`/`step-ca` for comfort). **2.** Issue a server certificate for `gitlab.lab` with subjectAltNames covering `gitlab.lab`, `registry.lab`, `repo.lab` (one cert, three names). **3.** GitLab: point `external_url 'https://gitlab.lab'` and `registry_external_url 'https://gitlab.lab:5050'` at the cert/key in `gitlab.rb`, reconfigure. **4.** Distribute trust — the real lesson of the exercise: the CA root must be installed on *every* consumer: workstation browser, each VM's `/usr/local/share/ca-certificates/` + `update-ca-certificates` (an Ansible task in `common` — of course), Podman's `/etc/containers/certs.d/gitlab.lab:5050/ca.crt`, and containerd's config on every cluster node. **5.** Delete every `insecure`/`--tls-verify=false` crutch and watch everything still work — that deletion diff is the proof. The distribution step is why facilities run real internal PKI; having done the miniature earns you that sentence.

## Appendix B — k3s: the one-evening alternative cluster

**k3s** is a CNCF-certified single-binary Kubernetes: `curl -sfL https://get.k3s.io | sh -` on a server node, one more command with a join token on agents, batteries (CNI, ingress, storage) included. Everything from Part 2 Ch. 6+ — Argo CD, the manifests, Helm, the monitoring stack — runs on it unchanged, which is itself the lesson: *Kubernetes is an API, and the distribution is an implementation detail.* Use it when you want a second, disposable cluster for experiments, or on hardware too small for kubeadm's footprint. Keep kubeadm as the primary story: "I built the explicit cluster once, so k3s is a convenience, not a mystery" is exactly the right ordering to have lived.

## Appendix C — Topology B: the same lab on physical machines

Everything transfers to real mini-PCs on a real switch with three substitutions: **provisioning** — manual OS installs (or PXE, if you want that project) replace Vagrant; the `Vagrantfile`'s role is taken by a documented install checklist + the same Ansible inventory with real IPs; **networking** — the host-only net becomes a physical subnet; `/etc/hosts` management via the `common` role is unchanged and now genuinely useful; **snapshots** — you lose them; compensate with stricter Ansible discipline and, ideally, one sacrificial machine you're willing to reinstall. Ansible, GitLab, Kubernetes, Tango, and every pipeline are *identical* — which is the whole IaC argument, demonstrated by migration.

---

## End of Part 4 — and of the handbook

The four parts together: **Part 1** — foundations from zero (Linux, virtualization, Git, control systems, the DevOps mental model) and the prepared lab host. **Part 2** — the on-prem platform: IaC, self-hosted GitLab + CI, containers, kubeadm, Argo CD, five-format packaging, signed distribution, observability. **Part 3** — the Tango system: the device model, six credible simulators, Sardana, Taurus, HDB++, alarms, and tests in the gate. **Part 4** — the fusion: real packaging with operational integration, the full publish pipeline, two Git-driven delivery lanes, the performed commit-to-operator loop with rollback, control-system monitoring, the debugging method and compendium, and the schedule that gets a person from a clean machine to a recorded, twenty-minute, three-act demonstration.

The claim the handbook set out to make demonstrable is now one sentence, and it is true: *a reviewed commit is the only way anything — infrastructure, platform, or beamline software — changes in this system, and every layer of it rebuilds from Git.*

---

## About this handbook

This handbook was developed as preparation for a DevOps Engineer application (control-system software delivery for photon-science beamlines). It is a personal lab-build reference, not an official product of any employer or institution named within it.

---

*End of Part 4 of 4.*
