# Building a Reproducible On-Prem DevOps Platform and a Tango Controls System
### A From-Scratch Lab Handbook — Theory and Practice

*Recreating a self-hosted GitOps + packaging platform and a distributed beamline control system, entirely with open-source tools, on your own hardware.*

---

> **Installment 1 of 2.** This file contains the front matter, **Part I — Foundations & Theory**, and
> **Part II — The On-Prem DevOps Platform**. Part III (the Tango control system) and Part IV (integration,
> operations, and the build schedule) follow in the second installment, in the same style. Once the whole book is
> reviewed it will be converted to Word/PDF.

---

## How to use this book

This is a **dual-track handbook**. Each chapter is split into two kinds of section:

- **Theory** sections explain *why* a technology exists and how it works, pitched so a newcomer to DevOps or
  control systems can follow without prior exposure. If you already know a topic, skim these.
- **Lab** sections are hands-on, command-by-command, and assume competence with Linux, a shell, and an editor.
  They are written to be typed and run in order.

The goal is not just to make the two projects *work once*, but to make you able to **rebuild them from a clean
machine and explain every layer** — the difference between having done something and understanding it.

### Conventions

| Element | Meaning |
|---|---|
| **THEORY** heading | conceptual explanation; safe to skim if familiar |
| **LAB** heading | hands-on steps to run |
| `monospace` | commands, filenames, code |
| **Checkpoint** | a concrete, testable "this now works" milestone |
| **Callout — Why** | the reasoning behind a choice |
| **Callout — Pitfall** | a common failure and how to avoid it |
| **Path A / Path B** | the two install routes: **A = conda-forge**, **B = apt/`.deb`** |

### The two install paths (read this once)

Wherever the conda-forge and Debian/apt routes genuinely differ, the book shows both:

- **Path A — conda-forge.** Fastest, most portable, fewest packaging quirks. Everything installs into a conda
  environment; great for development and for reproducing the lab quickly on any Linux host. This is the path to
  choose if you just want it working.
- **Path B — apt / `.deb`.** Closest to how a real Debian-based beamline host is actually operated: system
  packages, systemd units, native `.deb` artifacts served from a signed apt repository. Slower to set up, but it is
  the path that teaches the packaging skills at the centre of the DevOps role.

Where a step is identical on both paths, it is shown once. Where they diverge, you'll see **Path A** and **Path B**
side by side. You can build the whole book on one path; the labs note the few places where a later chapter assumes
you did.

---

## What you will build

Two systems, and the bridge between them.

**Project #6 — The On-Prem DevOps Platform.** A reproducible, fully automated, entirely open-source platform that
develops, packages, and delivers Linux-based scientific control-system software with no cloud dependency. Its
layers: infrastructure-as-code host provisioning (Vagrant + Ansible), a self-hosted GitLab with CI runners,
multi-format software packaging (native Debian `.deb`, Conda, Pip, plus Podman/OCI and Apptainer images) published
to a self-hosted GPG-signed apt repository and conda channel, GitOps delivery (Argo CD reconciling a kubeadm
Kubernetes cluster from Git), and an observability stack (Prometheus, Grafana, Alertmanager).

**Project #4 — The Tango Controls System.** A distributed control system for a 6–20 MeV electron-LINAC beamline,
built on the Tango Controls framework: PyTango device servers for magnet power supplies, a Ce:YAG beam-profile
camera, a Faraday cup / beam-current transformer, a UHV vacuum station, and motion; Sardana for scan orchestration;
Taurus operator GUIs; HDB++ for archiving; and alarm/interlock monitoring — all developed against **simulated
hardware**, exactly as real controls software is developed before beam.

**The bridge.** The Tango software is built, tested, packaged, and deployed **through** the DevOps platform, so a
single code change flows automatically: commit → CI tests and builds → signed artifacts in the registry →
deployment updates → the change is visible to operators. That loop is the point of the whole book.

### A note on honesty and scope

A software lab has no real magnets, cameras, or vacuum pumps. Every "device" in Part III is a **simulator** that
returns physically-plausible values. This is not a shortcut or a fudge: control-system software is *always*
developed and tested against simulators first, because beam time is scarce and hardware is dangerous. Building the
simulators is itself part of the real skill set. Where the lab diverges from a hardware deployment, the text says so.

---

# PART I — FOUNDATIONS & THEORY

*Read Part I once for the mental model; you will refer back to it constantly.*

## Chapter 1 — Scientific control systems and the beamline problem

### THEORY — What a control system actually is

A **control system** is the software and hardware layer that lets humans (and higher-level programs) observe and
command a physical machine. At a particle accelerator or a photon-science beamline, that machine is enormous and
distributed: hundreds of magnets, power supplies, vacuum pumps, cameras, motors, temperature sensors, and safety
interlocks, spread over hundreds of metres, each with its own electronics.

The control system's job is to turn all of that heterogeneous hardware into a **uniform, networked interface** so
that an operator in a control room — or an automated scan, or a physicist's Python script — can do things like
"set dipole magnet 3 to 120 amps", "read the beam current", or "move the sample stage to 10 mm", without caring
what brand of power supply or motor controller is underneath.

Three properties make this hard, and they shape every design decision in this book:

1. **It is distributed.** No single computer talks to all the hardware. The system is many cooperating processes on
   many hosts, communicating over a network. A failure in one subsystem must not take down the rest.
2. **It is heterogeneous.** A magnet power supply, a camera, and a vacuum gauge speak completely different
   electrical and software protocols. The control system must hide that behind a common abstraction.
3. **It is safety-critical and continuous.** Beam can damage equipment or people. Interlocks must be respected,
   state must be observable at all times, and history must be archived for diagnostics and compliance.

### THEORY — Two frameworks: EPICS and Tango

The scientific-facility world converged on two open-source control frameworks, and you will hear both named
constantly at places like DESY:

- **EPICS** (Experimental Physics and Industrial Control System) models the world as a flat namespace of
  **Process Variables (PVs)** — named values like `BEAM:CURRENT` — served by **IOCs** (Input/Output Controllers)
  and accessed over the Channel Access or pvAccess protocols. It is dominant in the US accelerator community and
  extremely mature.
- **Tango Controls** models the world as **devices** — objects with **attributes** (readable/writable values),
  **commands** (actions), and **properties** (configuration) — registered in a central database. It is
  object-oriented, strong in Europe (ESRF, DESY, ELETTRA, SOLEIL, MAX IV, ALBA), and is the framework Project #4
  is built on.

They solve the same problem with different shapes: EPICS gives you *values*; Tango gives you *objects that own
values*. This book uses Tango, but the DevOps platform is framework-agnostic — it would package and deliver EPICS
IOCs just as happily.

**Callout — Why photon science cares about this.** A synchrotron like PETRA IV produces X-ray beams for dozens of
experiments simultaneously. Each beamline is its own small machine that must be controlled, coordinated, and
archived. The control group's software is what makes those beamlines usable — and it must be delivered to many
hosts, reproducibly, which is precisely where DevOps enters.

### THEORY — Why a control group needs DevOps

Historically, control-system software was hand-installed on each host by an expert. That does not scale to a
facility with hundreds of hosts and dozens of beamlines, and it is not reproducible: when a host dies, rebuilding
it is archaeology. The modern answer is to treat control-system software like any other serious software product:

- keep it in version control,
- build it through automated pipelines that run tests,
- package it into versioned, signed artifacts,
- deploy those artifacts to hosts declaratively,
- and monitor the running result.

That is the entire thesis of this book, and the reason Projects #4 and #6 belong together.

**Checkpoint (conceptual):** you can explain, in one sentence each, what a device is in Tango, why control systems
are distributed, and why a control group benefits from CI/CD and packaging.

---

## Chapter 2 — What DevOps means for scientific software

### THEORY — The five pillars

"DevOps" is an overloaded word. For our purposes it decomposes into five concrete capabilities, each of which
becomes a part of Project #6.

**1. Infrastructure as Code (IaC).** Instead of clicking through installers to configure a server, you *describe*
the desired server in text files and run a tool that makes reality match the description. Two flavours appear here:
**provisioning** (creating the machine — Vagrant) and **configuration management** (installing and configuring
software on it — Ansible). The payoff is reproducibility: the machine is defined by files in Git, so it can be
rebuilt identically and audited.

The key property to internalise is **idempotence**: running the same configuration twice must leave the system in
the same correct state, not apply changes twice. Good IaC tools are declarative and idempotent by design.

**2. Continuous Integration / Continuous Delivery (CI/CD).** CI means: every time code changes, an automated system
checks it out, builds it, and runs the tests — so integration problems surface immediately, not at release. CD
extends that to automatically producing deployable artifacts (and, in its fullest form, deploying them). In this
book, **GitLab CI** runs the pipelines and **runners** execute the jobs.

**3. Software packaging.** A pile of source files is not deployable; a **package** is. Packaging turns your code
into a versioned, dependency-declaring, installable artifact. Scientific software commonly needs several formats at
once — Debian `.deb` for system installs, Conda for scientific Python environments, Pip/wheel for Python libraries,
and container images (OCI via Podman, or Apptainer for HPC) for isolation. Project #6 builds all of them, because a
facility's users live in all of those worlds.

**4. GitOps.** A specific, powerful way to do delivery: the desired state of your running systems is stored **in
Git**, and an agent continuously **reconciles** the real system to match. If reality drifts, the agent corrects it;
to change production, you change Git. **Argo CD** is the agent here, and a **Kubernetes** cluster is what it
reconciles. The benefits are auditability (every change is a commit) and self-healing (drift is corrected).

**5. Observability.** You cannot operate what you cannot see. Observability is the practice of emitting **metrics**
(numeric time series), collecting them (**Prometheus**), visualising them (**Grafana**), and alerting on them
(**Alertmanager**). The discipline of choosing *what* to watch is captured by the **Golden Signals** — latency,
traffic, errors, and saturation — which we will implement.

### THEORY — Why "on-prem" and "open-source" matter here

Two constraints in Project #6 are deliberate and worth understanding, because they are typical of scientific
facilities:

- **On-prem, no cloud.** Facility networks are often isolated for security and data-sovereignty reasons; control
  systems may be on physically separate networks with no internet route. So every component — Git host, registry,
  package repo — is self-hosted. This is a feature, not a limitation, and it is exactly the skill a facility values.
- **Fully open-source.** No proprietary licences, no vendor lock-in, and the ability to inspect and patch anything.
  It also means the platform can itself contribute back to the open-source scientific-IT ecosystem.

**Callout — Pitfall.** Newcomers often assume "DevOps = the cloud". It does not. Every concept here (IaC, CI/CD,
GitOps, observability) is fully realisable on bare metal in an air-gapped room. That is the whole point of this
build.

**Checkpoint (conceptual):** you can name the five pillars and, for each, the specific tool this book uses and the
problem it solves.

---

## Chapter 3 — The architecture we are building

### THEORY — The whole picture

Before touching a keyboard, hold the end state in your head. The lab is a small cluster of Linux hosts. On them:

- a **services host** runs GitLab (source control + container registry), the signed apt repo and conda channel, and
  the observability stack;
- a small **Kubernetes cluster** (one control-plane + two workers, built with kubeadm) is the deployment target,
  reconciled from Git by Argo CD;
- a **Tango host** runs the control system: the Tango database, the simulated device servers, Sardana, Taurus,
  HDB++.

Code flows left to right: a developer pushes to GitLab; CI runners build multi-format packages and push them to the
registry and repos; Argo CD (or Ansible) deploys them; Prometheus watches everything.

```
 developer ──git push──► GitLab ──triggers──► CI runners ──build──► .deb / conda / pip / OCI / .sif
                           │                                              │
                           │                                              ▼
                           │                              signed apt repo · conda channel · registry
                           │                                              │
                           ▼                                              ▼
                        Argo CD ──reconciles──► kubeadm K8s ◄──deploy── device servers / services
                           ▲                        │
                           └────── Git (desired state)                    ▲
                                                                          │
   Prometheus / Grafana / Alertmanager ──────── scrape & alert ──────────┘
                                                                          │
                                              Tango DB · device servers · Sardana · Taurus · HDB++
```

### THEORY — Host topology and sizing

The book assumes **Topology A: one workstation running several VMs**, because it is cheap, snapshot-able, and fully
reproducible with `vagrant destroy && vagrant up`. A capable single host (≈32 GB RAM, 8+ cores, ≈500 GB SSD) is
enough. Topology B (several physical mini-PCs) is mentioned where it differs but is not required.

| VM | Role | RAM | vCPU |
|---|---|---|---|
| `svc` | GitLab, registry, apt/conda repos, observability | 6 GB | 2 |
| `cp` | Kubernetes control-plane | 4 GB | 2 |
| `w1`, `w2` | Kubernetes workers | 4 GB each | 2 each |
| `tango` | Tango DB, device servers, Sardana, Taurus, HDB++ | 4–6 GB | 2 |

**Callout — Why VMs and not just containers.** Building the platform on VMs teaches host-level provisioning and
lets you practise kubeadm on "real" machines. Containers appear *inside* the platform (as CI jobs, as OCI images,
as pods) — but the substrate is deliberately host-shaped, matching a facility.

**Checkpoint (conceptual):** you can draw the data flow from `git push` to a deployed, monitored device server
without looking at the diagram.

---

## Chapter 4 — Preparing the lab foundation

### THEORY — What the host needs and why

You need three things on the physical host: a **hypervisor** (to run VMs), **Vagrant** (to define and create those
VMs from code), and **Ansible** (to configure them). Vagrant and Ansible are the IaC pair from Chapter 2:
Vagrant *provisions*, Ansible *configures*. Everything the VMs need beyond that will be installed *by* Ansible, so
the host stays clean.

### LAB — Install host tooling

Shown for an Ubuntu host; adapt package names for Fedora/RHEL. A **libvirt/KVM** provider is lighter on Linux than
VirtualBox; both are given.

```bash
sudo apt update
sudo apt install -y git curl gnupg2 make build-essential

# Hypervisor — choose ONE:
# (VirtualBox — simplest with Vagrant)
sudo apt install -y virtualbox
# (KVM/libvirt — lighter; also install the vagrant-libvirt plugin later)
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst

# Vagrant (HashiCorp apt repo)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y vagrant

# If using libvirt:
sudo apt install -y libvirt-dev
vagrant plugin install vagrant-libvirt

# Ansible
sudo apt install -y ansible
ansible --version    # confirm >= 2.14
```

**Callout — Pitfall.** On a host that is *itself* a VM, nested virtualisation must be enabled or the guest VMs
won't boot. On bare metal, enable VT-x/AMD-V in firmware. Check with `egrep -c '(vmx|svm)' /proc/cpuinfo` (non-zero
is good).

### LAB — Create the project workspace and pin versions

Reproducibility starts with pinning. Create a `versions.md` and a Git repo now.

```bash
mkdir -p ~/lab/devops-platform && cd ~/lab/devops-platform
git init
cat > versions.md <<'EOF'
# Pinned versions (edit as you go, keep the lab reproducible)
Ubuntu guest:      22.04 LTS (bento/ubuntu-22.04)
Kubernetes:        v1.30.x
containerd:        distro package
GitLab CE:         latest omnibus at build time (record exact version here)
Argo CD:           stable manifest at build time
Tango / PyTango:   record exact versions in Part III
EOF
git add . && git commit -m "chore: lab skeleton and version pins"
```

**Checkpoint 4:** `vagrant --version`, `ansible --version`, and your chosen hypervisor all run, nested virt is
available, and you have a Git repo with a `versions.md`.

---

# PART II — THE ON-PREM DEVOPS PLATFORM (Project #6)

*Part II builds the platform bottom-up. Each chapter has theory then labs, and ends with a checkpoint you can
demo in isolation.*

## Chapter 5 — Infrastructure as Code: Vagrant + Ansible

### THEORY — Provisioning vs. configuration, and idempotence

Two distinct jobs are easy to confuse:

- **Provisioning** creates the machine and its virtual hardware — CPU, RAM, disks, network interfaces. Vagrant does
  this by talking to the hypervisor, driven by a single `Vagrantfile`.
- **Configuration management** takes an existing machine and installs/configures software on it. Ansible does this
  by connecting over SSH and applying **playbooks** made of **roles** and **tasks**.

Ansible's superpower is **idempotence**: a task like "ensure package `chrony` is installed" does nothing if it's
already installed. This means you can run the whole playbook repeatedly and safely — the basis of "configuration as
code". Prefer Ansible **modules** (`apt`, `copy`, `user`, `systemd`) over raw `shell` commands precisely because
modules are idempotent and `shell` usually isn't.

**Callout — Why Vagrant for a lab.** In production you might build VM images with Packer or provision cloud
instances with Terraform. Vagrant is the right teaching tool because it makes the *entire* lab a single
`vagrant up`, and tears it down just as fast — ideal for practising reproducibility.

### THEORY — How the pieces fit

The `Vagrantfile` defines the VMs and then hands each one to Ansible for configuration. Ansible reads an
**inventory** (which hosts exist and their groups) and a **site playbook** (`site.yml`) that maps groups to roles.
Roles are reusable bundles of tasks, templates, and variables. Our role layout mirrors the platform:

```
ansible/
├── inventory.ini
├── site.yml
└── roles/
    ├── common/       # baseline: users, SSH, updates, NTP, firewall, hosts file
    ├── gitlab/       # GitLab CE + registry
    ├── runner/       # GitLab CI runner
    ├── k8s-common/   # containerd + kube tools (all cluster nodes)
    ├── k8s-control/  # kubeadm init + CNI
    ├── k8s-worker/   # kubeadm join
    ├── repo/         # apt (reprepro) + conda channel host
    └── observability/# node_exporter and friends
```

### LAB — The Vagrantfile

```ruby
# ~/lab/devops-platform/Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  nodes = {
    "svc"   => { ip: "192.168.56.10", mem: 6144, cpu: 2 },
    "cp"    => { ip: "192.168.56.11", mem: 4096, cpu: 2 },
    "w1"    => { ip: "192.168.56.12", mem: 4096, cpu: 2 },
    "w2"    => { ip: "192.168.56.13", mem: 4096, cpu: 2 },
    "tango" => { ip: "192.168.56.20", mem: 6144, cpu: 2 },
  }
  nodes.each do |name, o|
    config.vm.define name do |n|
      n.vm.hostname = name
      n.vm.network "private_network", ip: o[:ip]
      n.vm.provider "virtualbox" do |vb|   # or "libvirt"
        vb.memory = o[:mem]; vb.cpus = o[:cpu]
      end
    end
  end
  config.vm.provision "ansible" do |a|
    a.playbook = "ansible/site.yml"
    a.inventory_path = "ansible/inventory.ini"
    a.limit = "all"
  end
end
```

### LAB — Inventory and site playbook

```ini
# ansible/inventory.ini
[services]
svc ansible_host=192.168.56.10
[control_plane]
cp ansible_host=192.168.56.11
[workers]
w1 ansible_host=192.168.56.12
w2 ansible_host=192.168.56.13
[tango]
tango ansible_host=192.168.56.20
[kube:children]
control_plane
workers
```

```yaml
# ansible/site.yml
- hosts: all
  become: true
  roles: [common]

- hosts: services
  become: true
  roles: [gitlab, runner, repo, observability]

- hosts: kube
  become: true
  roles: [k8s-common]

- hosts: control_plane
  become: true
  roles: [k8s-control]

- hosts: workers
  become: true
  roles: [k8s-worker]
```

### LAB — The `common` role (idempotent baseline)

```yaml
# ansible/roles/common/tasks/main.yml
- name: Set timezone
  community.general.timezone: { name: UTC }

- name: Install baseline packages
  apt:
    name: [chrony, ufw, curl, gnupg2, ca-certificates, htop, vim]
    state: present
    update_cache: true

- name: Enable time sync
  systemd: { name: chrony, enabled: true, state: started }

- name: Populate /etc/hosts for the lab
  blockinfile:
    path: /etc/hosts
    block: |
      192.168.56.10 svc gitlab.lab
      192.168.56.11 cp
      192.168.56.12 w1
      192.168.56.13 w2
      192.168.56.20 tango

- name: Allow SSH through the firewall
  community.general.ufw: { rule: allow, port: '22', proto: tcp }

- name: Enable the firewall (default deny incoming)
  community.general.ufw: { state: enabled, policy: deny, direction: incoming }
```

**Callout — Pitfall.** If `ufw` default-denies before you open the ports a later service needs (GitLab 80/5050,
Kubernetes 6443, etc.), that service will be unreachable. Each role below opens its own ports; keep firewall rules
next to the service that needs them.

### LAB — Bring it up

```bash
cd ~/lab/devops-platform
vagrant up            # first run downloads the box + builds 5 VMs (15–30+ min)
# Re-apply configuration any time (idempotent):
vagrant provision
# Prove idempotence — a second run should report few or no "changed" tasks:
vagrant provision | tail -n 5
```

**Checkpoint 5:** `vagrant destroy -f && vagrant up` reproduces five hardened, identical hosts, and re-running
`vagrant provision` changes little or nothing (idempotence proven).

---

## Chapter 6 — Self-hosted GitLab (source control + container registry)

### THEORY — Why a single self-hosted forge

GitLab bundles, in one self-hostable product, everything the platform's "single source of truth" needs: Git
hosting, merge requests and code review, a CI engine, and a **container registry**. Self-hosting it means the
entire software supply chain lives inside the lab network — essential for an air-gapped facility, and the concrete
meaning of "self-hosted GitLab as single source of truth" from the CV.

The **container registry** matters as much as the Git hosting: it is where the OCI images your pipelines build are
stored and pulled from. Co-locating SCM, CI, and registry keeps the supply chain coherent and auditable.

### LAB — Install GitLab CE (role `gitlab`, shown raw)

```yaml
# ansible/roles/gitlab/tasks/main.yml (essentials)
- name: Add GitLab CE apt repo
  shell: curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | bash
  args: { creates: /etc/apt/sources.list.d/gitlab_gitlab-ce.list }

- name: Install GitLab CE
  apt: { name: gitlab-ce, state: present, update_cache: true }
  environment: { EXTERNAL_URL: "http://gitlab.lab" }

- name: Configure registry in gitlab.rb
  blockinfile:
    path: /etc/gitlab/gitlab.rb
    block: |
      external_url 'http://gitlab.lab'
      registry_external_url 'http://gitlab.lab:5050'
  notify: reconfigure gitlab

- name: Open GitLab ports
  community.general.ufw: { rule: allow, port: "{{ item }}", proto: tcp }
  loop: ['80', '5050']
```
```yaml
# ansible/roles/gitlab/handlers/main.yml
- name: reconfigure gitlab
  command: gitlab-ctl reconfigure
```

Then set the root password and log in:
```bash
vagrant ssh svc -c "sudo gitlab-rake 'gitlab:password:reset[root]'"
# Add '192.168.56.10 gitlab.lab' to your workstation's /etc/hosts, then browse http://gitlab.lab
```

**Callout — Why plain HTTP in the lab.** TLS everywhere is correct for production but adds a local-CA yak-shave.
The lab uses HTTP to stay focused; Appendix A shows how to add a local CA and switch to HTTPS when you want the
full experience (and to make Podman push without `--tls-verify=false`).

### LAB — Create the projects

In the UI create a group `beamline` and two projects: `tango-devices` (the control-system code) and
`platform-gitops` (the Kubernetes manifests Argo CD will watch). Add your SSH key and confirm a push works:
```bash
git clone git@gitlab.lab:beamline/tango-devices.git
cd tango-devices && echo "# tango-devices" > README.md
git add . && git commit -m "init" && git push
```

**Checkpoint 6:** you can push over SSH to GitLab and `podman login gitlab.lab:5050` succeeds.

---

## Chapter 7 — CI runners and your first pipelines

### THEORY — Runners, executors, and the pipeline model

GitLab decides *what* to run (the pipeline, defined in `.gitlab-ci.yml`); a **runner** is a separate agent that
actually *executes* the jobs. Runners have **executors** that determine the job environment: the **docker/podman**
executor runs each job in a fresh container (clean, reproducible — preferred), while the **shell** executor runs
directly on the runner host (simple, but state leaks between jobs).

A pipeline is a sequence of **stages** (e.g. `test → build → publish`); each stage contains **jobs** that run in
parallel; a later stage starts only if the previous one passed. This structure is what enforces "no artifact is
published unless tests pass".

### LAB — Install and register a runner

```yaml
# ansible/roles/runner/tasks/main.yml (essentials)
- name: Add runner apt repo
  shell: curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash
  args: { creates: /etc/apt/sources.list.d/runner_gitlab-runner.list }
- name: Install runner + a container engine
  apt: { name: [gitlab-runner, podman, docker.io], state: present, update_cache: true }
- name: Register runner
  command: >
    gitlab-runner register --non-interactive --url "http://gitlab.lab/"
    --registration-token "{{ runner_token }}" --executor docker
    --docker-image ubuntu:22.04 --description lab-runner --tag-list "lab,build"
  args: { creates: /etc/gitlab-runner/config.toml }
```

### LAB — First pipeline

```yaml
# .gitlab-ci.yml in the tango-devices project
stages: [test]
smoke:
  stage: test
  tags: [lab]
  image: ubuntu:22.04
  script:
    - echo "runner works"; uname -a
```
Push, then watch the pipeline go green in **CI/CD → Pipelines**.

**Checkpoint 7:** a green pipeline runs on your self-hosted runner in a clean container.

---

## Chapter 8 — Kubernetes with kubeadm

### THEORY — What Kubernetes is, minimally

Kubernetes (k8s) is a system for running containers across a set of machines while keeping a **declared** state
true: you tell it "I want 2 replicas of this image exposed on this port", and it schedules the containers, restarts
them if they die, and reschedules them if a node fails. The pieces you must know:

- **Control-plane** — the brain: the API server (everything talks to it), the scheduler, and etcd (the datastore).
- **Nodes/workers** — where your containers (grouped into **pods**) actually run, via the **kubelet** agent and a
  container runtime (**containerd**).
- **CNI** — the networking plugin (Flannel/Calico) that gives pods IP addresses and lets them talk across nodes.
- **Objects** — you declare **Deployments** (desired pods), **Services** (stable network endpoints), etc., as YAML.

**kubeadm** is the official tool that bootstraps a real cluster from scratch — as opposed to single-node toys like
minikube. Using it teaches the actual moving parts, which is why the CV says "kubeadm".

**Callout — Why kubeadm and not k3s here.** k3s is a lightweight, single-binary distribution — excellent, and a
fine secondary skill. kubeadm is the reference path and exposes the control-plane/worker split explicitly, so it is
the better teacher. (k3s is noted as a faster alternative in Appendix B.)

### THEORY — The three prerequisites people get wrong

Almost every failed kubeadm install trips on one of these: **swap must be off**; the **cgroup driver** must be
`systemd` for both containerd and kubelet; and the **pod network CIDR** passed to `kubeadm init` must match what the
CNI expects. The lab below sets all three correctly.

### LAB — `k8s-common` role (every cluster node)

```yaml
# ansible/roles/k8s-common/tasks/main.yml (essentials)
- name: Install containerd
  apt: { name: containerd, state: present, update_cache: true }
- name: Default containerd config with systemd cgroups
  shell: |
    mkdir -p /etc/containerd
    containerd config default > /etc/containerd/config.toml
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
  args: { creates: /etc/containerd/config.toml }
  notify: restart containerd
- name: Disable swap now and in fstab
  shell: swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab
- name: Kernel modules and sysctl for k8s
  copy:
    dest: "{{ item.path }}"
    content: "{{ item.content }}"
  loop:
    - { path: /etc/modules-load.d/k8s.conf, content: "overlay\nbr_netfilter\n" }
    - { path: /etc/sysctl.d/k8s.conf, content: "net.bridge.bridge-nf-call-iptables=1\nnet.ipv4.ip_forward=1\n" }
- name: Load modules and apply sysctl
  shell: modprobe overlay; modprobe br_netfilter; sysctl --system
- name: Add Kubernetes apt repo (v1.30) and install tools
  shell: |
    mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
    echo "deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" > /etc/apt/sources.list.d/k8s.list
    apt-get update && apt-get install -y kubelet kubeadm kubectl && apt-mark hold kubelet kubeadm kubectl
  args: { creates: /usr/bin/kubeadm }
- name: Open cluster ports
  community.general.ufw: { rule: allow, port: "{{ item }}", proto: tcp }
  loop: ['6443','10250','8472']   # api, kubelet, flannel vxlan (udp 8472 in prod)
```

### LAB — control-plane and workers

```yaml
# ansible/roles/k8s-control/tasks/main.yml (essentials)
- name: kubeadm init
  shell: kubeadm init --apiserver-advertise-address=192.168.56.11 --pod-network-cidr=10.244.0.0/16
  args: { creates: /etc/kubernetes/admin.conf }
- name: Set up kubectl for the vagrant user
  shell: |
    mkdir -p /home/vagrant/.kube
    cp /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown -R vagrant:vagrant /home/vagrant/.kube
  args: { creates: /home/vagrant/.kube/config }
- name: Install Flannel CNI
  become_user: vagrant
  shell: kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
- name: Generate a join command for workers
  shell: kubeadm token create --print-join-command
  register: join_cmd
- name: Stash join command for the worker role
  copy: { content: "{{ join_cmd.stdout }}", dest: /vagrant/join.sh, mode: '0755' }
```
```yaml
# ansible/roles/k8s-worker/tasks/main.yml (essentials)
- name: Join the cluster
  shell: "bash /vagrant/join.sh"
  args: { creates: /etc/kubernetes/kubelet.conf }
```

Verify from `cp`:
```bash
vagrant ssh cp -c "kubectl get nodes -o wide"   # cp, w1, w2 all Ready
```

**Callout — Pitfall.** If nodes stay `NotReady`, the CNI didn't come up: check `kubectl -n kube-flannel get pods`
and confirm the pod CIDR matches `10.244.0.0/16`. VirtualBox private networks sometimes need the Flannel interface
pinned to the `192.168.56.0/24` NIC via the DaemonSet args.

**Checkpoint 8:** three-node cluster, all `Ready`, and a test pod (`kubectl run tmp --image=busybox -it --rm --
sh`) can reach the network.

---

## Chapter 9 — GitOps with Argo CD

### THEORY — Reconciliation, not deployment

Traditional deployment is **imperative**: a pipeline runs `kubectl apply` and *pushes* changes into the cluster.
GitOps inverts this. Argo CD runs *inside* the cluster and continuously **pulls** the desired state from a Git repo,
comparing it to reality and reconciling any difference. Consequences:

- **Git is the single source of truth.** To change production, you commit; there is no out-of-band `kubectl`.
- **Drift is self-healing.** If someone hand-edits the cluster, Argo CD notices and reverts to what Git says.
- **Everything is audited.** Every production change is a reviewable commit with an author and a diff.

This is why the CV phrase is "Argo CD reconciling an on-prem Kubernetes cluster from Git" — reconciliation is the
key idea.

### LAB — Install Argo CD and connect the GitOps repo

```bash
vagrant ssh cp
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
# admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# reach the UI from your workstation:
kubectl -n argocd port-forward --address 0.0.0.0 svc/argocd-server 8080:443
```

Create an Application that watches `platform-gitops`:
```yaml
# apply this manifest (or create it in the UI)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: beamline-apps, namespace: argocd }
spec:
  project: default
  source:
    repoURL: http://gitlab.lab/beamline/platform-gitops.git
    targetRevision: main
    path: manifests
  destination: { server: https://kubernetes.default.svc, namespace: beamline }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```

Commit a simple manifest (e.g. an nginx Deployment) into `platform-gitops/manifests/`, and watch Argo CD sync it.
Then hand-delete the Deployment and watch it come back — self-healing in action.

**Checkpoint 9:** editing YAML in Git changes the cluster with no manual `kubectl apply`, and deleting a managed
object triggers automatic restoration.

---

## Chapter 10 — Multi-format software packaging

### THEORY — Why one component, many formats

A "package" is a versioned, dependency-declaring, installable unit. Scientific facilities need several formats
because their users live in different worlds:

- **Pip/wheel** — the native Python format; for libraries other Python code imports.
- **Conda** — resolves *non-Python* dependencies too (compilers, libraries), which is why scientific Python leans on
  it; delivered via **channels**.
- **Debian `.deb`** — the system format for Debian/Ubuntu hosts; integrates with `apt` and `systemd`; how a real
  beamline *host* installs software.
- **OCI images (Podman)** — a whole userland bundled for container runtimes and Kubernetes.
- **Apptainer `.sif`** — a single-file container format designed for HPC/multi-user compute where Docker daemons
  aren't allowed; common at facilities.

Project #6 builds all five from the same source so the component can be consumed anywhere. This chapter uses a tiny
stand-in package, `beamline-utils`; in Part IV you'll point the exact same machinery at the real Tango device
servers.

### LAB — The sample component

```
beamline-utils/
├── pyproject.toml
├── src/beamline_utils/__init__.py
└── src/beamline_utils/cli.py
```
```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"
[project]
name = "beamline-utils"
version = "0.1.0"
description = "Lab stand-in for a control-system component"
requires-python = ">=3.10"
[project.scripts]
beamline-info = "beamline_utils.cli:main"
```
```python
# src/beamline_utils/cli.py
def main():
    print("beamline-utils 0.1.0 — OK")
```

### LAB — Path-agnostic: Pip / wheel

```bash
python3 -m pip install --upgrade build
python3 -m build          # -> dist/beamline_utils-0.1.0-py3-none-any.whl and .tar.gz
```

### LAB — Path A (conda-forge): Conda package

```bash
# with miniforge installed:
conda install -y conda-build
mkdir -p conda-recipe
cat > conda-recipe/meta.yaml <<'EOF'
package: { name: beamline-utils, version: "0.1.0" }
source: { path: .. }
build:
  script: {{ PYTHON }} -m pip install . -vv
  entry_points: [ "beamline-info = beamline_utils.cli:main" ]
requirements:
  host: [ python, pip, setuptools ]
  run:  [ python ]
EOF
conda build conda-recipe/     # -> a .conda artifact under the conda-bld directory
```

### LAB — Path B (apt): native Debian `.deb` with debhelper + dh-python

```bash
sudo apt install -y devscripts debhelper dh-python python3-all pybuild-plugin-pyproject
dh_make --python -y -p beamline-utils_0.1.0    # scaffolds debian/
# Edit debian/control:
#   Build-Depends: debhelper-compat (= 13), dh-python, python3-all, python3-setuptools
#   Depends: ${python3:Depends}, ${misc:Depends}
# Edit debian/rules to a single pybuild line:
cat > debian/rules <<'EOF'
#!/usr/bin/make -f
%:
	dh $@ --with python3 --buildsystem=pybuild
EOF
chmod +x debian/rules
dpkg-buildpackage -us -uc -b    # -> ../beamline-utils_0.1.0_all.deb
dpkg-deb -I ../beamline-utils_0.1.0_all.deb   # inspect metadata
```

**Callout — Why debhelper/dh-python specifically.** `debhelper` automates the dozens of fiddly steps of a correct
`.deb`; `dh-python` teaches it how to lay out Python modules to Debian policy. Together they turn packaging from
arcane to a three-line `debian/rules` — the exact toolchain named on the CV.

### LAB — OCI image (Podman) → GitLab registry

```dockerfile
# Containerfile
FROM python:3.11-slim
COPY dist/*.whl /tmp/
RUN pip install /tmp/*.whl
ENTRYPOINT ["beamline-info"]
```
```bash
podman build -t gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0 .
podman login gitlab.lab:5050
podman push gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0
```

### LAB — Apptainer `.sif`

```bash
sudo apt install -y apptainer
cat > beamline-utils.def <<'EOF'
Bootstrap: docker
From: python:3.11-slim
%files
    dist/*.whl /tmp/
%post
    pip install /tmp/*.whl
%runscript
    exec beamline-info "$@"
EOF
apptainer build beamline-utils.sif beamline-utils.def
apptainer run beamline-utils.sif
```

**Checkpoint 10:** from one source tree you have produced a wheel, a conda package, a `.deb`, an OCI image in the
registry, and a `.sif` — the same component in five formats.

---

## Chapter 11 — The signed distribution registry

### THEORY — Why signing and repositories matter

Building artifacts isn't enough; hosts must be able to **find and trust** them. Two mechanisms:

- A **repository** is an indexed, HTTP-served collection of packages that clients (`apt`, `conda`) can install from
  by name and version — no manual file copying.
- **GPG signing** lets clients cryptographically verify that a package came from you and wasn't tampered with. On a
  facility network this is how you prevent a poisoned package from reaching a beamline host. "GPG-signed apt/conda
  registry" on the CV is exactly this.

### LAB — A signed apt repository with reprepro (on `svc`)

```bash
sudo apt install -y reprepro nginx
gpg --quick-generate-key "Lab Repo <repo@lab>" default default never
KEY=$(gpg --list-keys --with-colons repo@lab | awk -F: '/^fpr:/{print $10; exit}')
mkdir -p ~/apt-repo/conf
cat > ~/apt-repo/conf/distributions <<EOF
Origin: lab
Label: lab
Codename: jammy
Architectures: amd64 source
Components: main
Description: Lab control-system apt repo
SignWith: $KEY
EOF
reprepro -b ~/apt-repo includedeb jammy ~/beamline-utils_0.1.0_all.deb
# Serve it:
sudo ln -s ~/apt-repo /var/www/html/apt
# Export the public key for clients:
gpg --armor --export repo@lab | sudo tee /var/www/html/apt/lab.gpg.asc
```
On a client host:
```bash
curl -fsSL http://svc.lab/apt/lab.gpg.asc | sudo gpg --dearmor -o /usr/share/keyrings/lab.gpg
echo "deb [signed-by=/usr/share/keyrings/lab.gpg] http://svc.lab/apt jammy main" | sudo tee /etc/apt/sources.list.d/lab.list
sudo apt update && sudo apt install -y beamline-utils && beamline-info
```

### LAB — A self-hosted conda channel

```bash
mkdir -p ~/conda-channel/linux-64
cp $(conda build conda-recipe/ --output) ~/conda-channel/linux-64/
conda install -y conda-index && python -m conda_index ~/conda-channel
sudo ln -s ~/conda-channel /var/www/html/conda
# client:
conda install -c http://svc.lab/conda beamline-utils
```

**Checkpoint 11:** a clean host installs your component *by name* from the signed apt repo and from the conda
channel — no manual file copying, signature verified.

---

## Chapter 12 — Observability: Prometheus, Grafana, Alertmanager

### THEORY — Metrics, scraping, and the Golden Signals

**Metrics** are numeric time series (e.g. CPU %, request count). **Prometheus** collects them by **scraping** HTTP
`/metrics` endpoints that each component exposes (an **exporter** like `node_exporter` provides host metrics).
**Grafana** queries Prometheus to draw dashboards. **Alertmanager** receives alerts that Prometheus fires from rules
and routes them (email, webhook), handling grouping and silencing.

The discipline of *what* to watch is the **Golden Signals**: **latency** (how long requests take), **traffic** (how
much demand), **errors** (failure rate), and **saturation** (how full the system is). Alerting on these four catches
most real problems without drowning you in noise.

### LAB — Install the kube-prometheus-stack (via Helm)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
kubectl create namespace monitoring
helm install kps prometheus-community/kube-prometheus-stack -n monitoring
kubectl -n monitoring rollout status deploy/kps-grafana
kubectl -n monitoring port-forward --address 0.0.0.0 svc/kps-grafana 3000:80   # admin / prom-operator
```

### LAB — Host metrics + a Golden-Signals alert

Run `node_exporter` on the VMs (a systemd unit via an Ansible task, or a DaemonSet) and add a scrape target. Then a
minimal saturation rule:
```yaml
# a PrometheusRule (saturation example)
groups:
- name: golden-signals
  rules:
  - alert: HostHighMemory
    expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
    for: 2m
    labels: { severity: warning }
    annotations: { summary: "Host {{ $labels.instance }} memory > 90%" }
```
Import a node dashboard into Grafana (dashboard ID 1860 is a common node-exporter board). Finally write the
one-page **host-troubleshooting runbook** (symptom → check → fix) that the CV references — a table living in
`platform-gitops/docs/runbook.md`.

**Callout — Why a runbook counts.** Observability without a documented response is half a system. The runbook is
what turns an alert into a fixed problem at 3 a.m., and it's a deliverable in its own right.

**Checkpoint 12:** Grafana shows live host and cluster dashboards, and a deliberately-induced condition (e.g. `stress
--vm`) fires an alert that lands in Alertmanager.

> **Part II complete.** You now have the full reproducible DevOps platform: IaC (Vagrant + Ansible), self-hosted
> GitLab + CI, a kubeadm cluster reconciled by Argo CD, five-format signed packaging, and observability. Part III
> builds the Tango control system that this platform will deliver; Part IV wires them into a single commit-to-
> operator loop and gives the realistic build schedule.

---

*End of Installment 1. Part III (Tango Controls: concepts, install on both paths, PyTango device servers, the
simulated beamline, Sardana, Taurus, HDB++, alarms, testing) and Part IV (integration, operations, troubleshooting
compendium, and the week-by-week schedule) continue in Installment 2, followed by Word/PDF conversion.*
