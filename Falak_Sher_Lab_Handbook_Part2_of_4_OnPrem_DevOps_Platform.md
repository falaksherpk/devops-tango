# Building a Reproducible On-Prem DevOps Platform and a Tango Controls System

## PART 2 of 4 — The On-Prem DevOps Platform

*Part 2 builds the platform bottom-up: five VMs defined as code, a self-hosted GitLab with CI runners, a kubeadm Kubernetes cluster reconciled from Git by Argo CD, one component packaged in five formats and published to your own GPG-signed apt repository and conda channel, and a Prometheus/Grafana/Alertmanager observability stack. Every chapter follows the Part 1 pattern — THEORY, then LAB, then a Checkpoint you can demo in isolation — and assumes exactly the state Part 1 left you in: a host with a hypervisor, Vagrant, Ansible, an SSH key, and a Git workspace at `~/lab/devops-platform`.*

---

## How Part 2 is organised

| Chapter | Builds | Pillar (Part 1, Ch. 5) |
|---|---|---|
| **1** | The five-VM lab, provisioned by Vagrant and configured by Ansible | Infrastructure as Code |
| **2** | Self-hosted GitLab: Git hosting, merge requests, container registry | (the source of truth everything else needs) |
| **3** | CI runners and your first pipelines | CI/CD |
| **4** | Containers from zero: images, Podman, the OCI model | (prerequisite for Ch. 5, 7) |
| **5** | A real Kubernetes cluster with kubeadm | (the GitOps target) |
| **6** | GitOps with Argo CD | GitOps |
| **7** | One component in five package formats | Packaging |
| **8** | The signed apt repository and conda channel | Packaging (distribution side) |
| **9** | Prometheus, Grafana, Alertmanager, and a runbook | Observability |
| **10** | Part 2 review: self-assessment, glossary additions, demo script | — |

Estimated effort: **2–3 weeks** working steadily, dominated by Chapters 5, 7, and 8. Chapters must be done in order — each consumes the previous one's checkpoint.

**A reading rule for this part.** Part 2 contains a lot of configuration files. Never paste one without reading its annotation first: the files are short *because* every line is doing work, and the interview value of this lab is being able to explain any line you shipped. Where a file is shown "(essentials)", it is complete enough to work but trimmed of repetition; your Git history is the full record.

**Conventions reminder** (full table in Part 1): `$` = your normal user; commands inside a `vagrant ssh <vm>` block run *inside that VM*; **Path A** = conda-forge, **Path B** = apt/`.deb`; do not pass a failed **Checkpoint**.

---

# PART 2 — THE ON-PREM DEVOPS PLATFORM

---

## Chapter 1 — Infrastructure as Code: the five-VM lab with Vagrant + Ansible

Part 1 ended with a 90-second single-VM taste of Vagrant. This chapter scales that to the real thing: the five hosts of the architecture diagram, created by one file and configured by another, both in Git — so that from here on, *the lab itself is a artifact you can destroy and regenerate at will*. That property is not a convenience; it is the safety net that lets you experiment aggressively in every later chapter.

### THEORY 1.1 — Provisioning vs configuration management, precisely

Part 1 introduced the split; now sharpen it, because the two tools have different failure modes and you must know which layer a problem lives in:

- **Provisioning** creates the machine and its virtual hardware — CPU count, RAM, disks, network interfaces, base OS image. **Vagrant** does this by driving the hypervisor from a single `Vagrantfile`. If a VM doesn't exist, has the wrong IP, or won't boot — that's the provisioning layer.
- **Configuration management** takes an existing, booted, SSH-reachable machine and installs/configures software on it. **Ansible** does this by connecting over SSH and applying **playbooks**. If a VM exists but a package is missing, a service isn't running, or a config file is wrong — that's the configuration layer.

**Callout — Why Vagrant here, and what replaces it elsewhere.** In production you might build golden VM images with Packer, provision cloud instances with Terraform, or deploy from templates in vSphere. Vagrant is the right *teaching* tool because it makes the entire lab one `vagrant up` and one `vagrant destroy` — perfect for practising reproducibility — and because Ansible, the half that transfers everywhere unchanged, does all the interesting work anyway.

### THEORY 1.2 — Ansible's model: inventory, playbooks, roles, modules

Ansible has four nouns; once they click, every Ansible file in this book reads like prose.

**1. Inventory — *which machines exist*.** A plain text file listing hosts and grouping them. Groups matter because configuration is assigned to groups, not individual machines: "all `kube` nodes get containerd" scales from three nodes to three hundred without editing anything but the inventory.

**2. Playbook — *what should be true where*.** A YAML file mapping groups of hosts to lists of roles or tasks. Our top-level playbook is `site.yml` — by convention, the playbook that describes the *entire site*.

**3. Role — *a reusable bundle of configuration*.** A directory with a conventional layout: `tasks/` (what to do), `handlers/` (things to do only when notified, like restarting a service after its config changed), `templates/` (config files with variables), `defaults/` (variables). Roles are the unit of reuse and the unit of thinking: "the `gitlab` role" *is* our GitLab installation, reviewable as a diff.

**4. Module — *one idempotent operation*.** Tasks call modules: `apt` (ensure a package state), `systemd` (ensure a service state), `copy`/`template` (ensure file contents), `user`, `ufw`, and hundreds more. Modules are where idempotence lives: `apt: name=chrony state=present` checks first and does nothing if chrony is already installed.

**The idempotence discipline.** Prefer modules over raw `shell:` commands, precisely because modules are idempotent and shell commands usually are not. Sometimes shell is unavoidable (vendor install scripts, `kubeadm init`); the pattern then is to guard it with `args: { creates: /path/that/exists/afterwards }` — Ansible skips the task if that file already exists, restoring idempotence by hand. You will see this pattern throughout Part 2; every `creates:` line is a deliberate idempotence guard, not decoration.

**How a run works, mechanically.** `ansible-playbook site.yml` reads the inventory, SSHes to each targeted host in parallel (using the key from Part 1, Lab 1.4), gathers **facts** (OS, IPs, memory — you saw them with `ansible localhost -m setup`), then executes tasks in order, printing one line per task per host: `ok` (already correct — nothing done), `changed` (corrected something), or `failed`. The summary line at the end — `ok=41 changed=3 failed=0` — is how you read the health of a run, and `changed=0` on a second run is the experimental proof of idempotence.

### THEORY 1.3 — The role layout for this platform

The roles mirror the architecture one-to-one — deliberately, so that the repository *is* the documentation:

```
~/lab/devops-platform/
├── Vagrantfile
├── versions.md
└── ansible/
    ├── inventory.ini
    ├── site.yml
    └── roles/
        ├── common/        # baseline every host gets: packages, NTP, hosts file, firewall
        ├── gitlab/        # GitLab CE + container registry            (Ch. 2)
        ├── runner/        # GitLab CI runner                          (Ch. 3)
        ├── k8s-common/    # containerd + kube tools, all cluster nodes (Ch. 5)
        ├── k8s-control/   # kubeadm init + CNI                        (Ch. 5)
        ├── k8s-worker/    # kubeadm join                              (Ch. 5)
        ├── repo/          # signed apt repo + conda channel           (Ch. 8)
        └── observability/ # node_exporter etc.                       (Ch. 9)
```

You create `common` now; each later chapter adds its role. By Part 4, `git log` on this repository is a build diary of the whole platform.

### LAB 1.1 — The Vagrantfile

In `~/lab/devops-platform` (the Git repo from Part 1, Lab 7.3), create the fleet definition. Read the annotations; this file is Part 1's Chapter 6 table translated into code:

```ruby
# ~/lab/devops-platform/Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"          # base image for every VM (pinned in versions.md)

  nodes = {                                     # the Chapter-6 topology, as data
    "svc"   => { ip: "192.168.56.10", mem: 6144, cpu: 2 },
    "cp"    => { ip: "192.168.56.20", mem: 4096, cpu: 2 },
    "w1"    => { ip: "192.168.56.21", mem: 4096, cpu: 2 },
    "w2"    => { ip: "192.168.56.22", mem: 4096, cpu: 2 },
    "tango" => { ip: "192.168.56.30", mem: 6144, cpu: 2 },
  }

  nodes.each do |name, o|                       # stamp out one VM per entry
    config.vm.define name do |n|
      n.vm.hostname = name
      n.vm.network "private_network", ip: o[:ip]   # the host-only 192.168.56.0/24 net
      n.vm.provider "virtualbox" do |vb|           # (or "libvirt")
        vb.memory = o[:mem]
        vb.cpus   = o[:cpu]
      end
    end
  end

  config.vm.provision "ansible" do |a|          # after boot, hand every VM to Ansible
    a.playbook       = "ansible/site.yml"
    a.inventory_path = "ansible/inventory.ini"
    a.limit          = "all"
  end
end
```

Three things to notice. First, the topology is a Ruby hash — adding a sixth host is one line. Second, every VM automatically gets *two* network interfaces: Vagrant's default NAT (outbound internet for downloads) plus our `private_network` (the lab net) — exactly the pattern Part 1, Chapter 6.4 described. Third, the `provision "ansible"` block is the handoff: Vagrant's job ends at "booted and SSH-able"; Ansible owns everything after.

**Reduced lab (16 GB hosts):** delete the `w2` line and reduce `svc` to 5120 — the labs note the two places this matters (Ch. 5 verification and one scheduling exercise).

### LAB 1.2 — Inventory and site playbook

```ini
# ansible/inventory.ini
[services]
svc   ansible_host=192.168.56.10

[control_plane]
cp    ansible_host=192.168.56.20

[workers]
w1    ansible_host=192.168.56.21
w2    ansible_host=192.168.56.22

[tango]
tango ansible_host=192.168.56.30

[kube:children]        ; a group made of other groups:
control_plane          ; "kube" = every Kubernetes node,
workers                ; regardless of role
```

```yaml
# ansible/site.yml — the whole site, one page
- hosts: all            # every machine, no exceptions,
  become: true          # as root (become = sudo),
  roles: [common]       # gets the baseline

- hosts: services
  become: true
  roles: [gitlab, runner, repo, observability]   # added chapter by chapter

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

Until the later roles exist, comment their names out — Ansible fails on missing roles. Uncomment each as its chapter creates it; the `site.yml` diff per chapter is then a one-word change, which is exactly how infrastructure growth *should* look in review.

### LAB 1.3 — The `common` role: an idempotent baseline

Every host — whatever its job — needs the same hygiene: correct time (clustered systems misbehave badly on clock skew, and Kubernetes certificates are time-sensitive), baseline packages, name resolution for the lab, and a default-deny firewall. That is the `common` role:

```yaml
# ansible/roles/common/tasks/main.yml
- name: Set timezone
  community.general.timezone: { name: UTC }        # servers run UTC; humans convert

- name: Install baseline packages
  apt:
    name: [chrony, ufw, curl, gnupg2, ca-certificates, htop, vim]
    state: present
    update_cache: true                             # apt update first

- name: Enable time sync
  systemd: { name: chrony, enabled: true, state: started }

- name: Populate /etc/hosts for the lab           # Part 1, Ch. 6.4's name plan, enforced everywhere
  blockinfile:
    path: /etc/hosts
    block: |
      192.168.56.10 svc gitlab.lab registry.lab repo.lab grafana.lab prometheus.lab
      192.168.56.20 cp cp.lab
      192.168.56.21 w1 w1.lab
      192.168.56.22 w2 w2.lab
      192.168.56.30 tango tango.lab

- name: Allow SSH through the firewall
  community.general.ufw: { rule: allow, port: '22', proto: tcp }

- name: Enable the firewall (default deny incoming)
  community.general.ufw: { state: enabled, policy: deny, direction: incoming }
```

Note `blockinfile`: it manages a *marked block* inside an existing file — re-running replaces the block rather than appending duplicates. That is idempotence applied to file fragments, and it's the right module whenever you own part of a file that something else also owns.

**Callout — Pitfall (the firewall trap).** `ufw` now default-denies incoming traffic. Every service added later — GitLab on 80/5050, the Kubernetes API on 6443, Prometheus on 9090 — will be *silently unreachable* until its role opens its ports. The discipline used throughout this book: **firewall rules live in the role of the service that needs them**, never in a central list that drifts out of sync. When something is unreachable in later chapters, "did the role open the port?" is always your first check.

### LAB 1.4 — Bring the lab up, and prove idempotence

```bash
cd ~/lab/devops-platform
git add -A && git commit -m "iac: five-VM topology, inventory, common role"

vagrant up               # first run: downloads the box once, builds 5 VMs, runs Ansible
                         # expect 15–30+ minutes; read the Ansible output as it goes
```

When it finishes, tour your infrastructure:

```bash
vagrant status                          # five machines, all "running"
vagrant ssh svc                         # you're on the services host
  cat /etc/hosts                        # the block your role wrote
  systemctl status chrony               # running and enabled (Part 1, Lab 1.3 skills)
  sudo ufw status                       # active; 22/tcp allowed
  ping -c 2 tango.lab                   # name resolution + lab network, in one test
  exit
```

Now the two experiments that make IaC real rather than rhetorical:

**Experiment 1 — idempotence.** Re-apply everything:

```bash
vagrant provision 2>&1 | tail -n 8      # look at the PLAY RECAP lines
```

Every host should report `changed=0` (or nearly — the first re-run may legitimately fix one or two things). The playbook *described* a state; the state already holds; nothing happens. This is the property that makes it safe to run `site.yml` against three hundred production hosts at 2 p.m. on a Tuesday.

**Experiment 2 — drift correction.** Sabotage a host, then let code fix it:

```bash
vagrant ssh w1 -c "sudo systemctl stop chrony && sudo systemctl disable chrony"
vagrant provision --provision-with ansible 2>&1 | grep -A1 "time sync"
# the chrony task reports "changed" on w1 only — drift detected and corrected
```

**Checkpoint 1:** `vagrant destroy -f && vagrant up` reproduces five hardened, identical, name-resolving hosts with no manual step; a second `vagrant provision` reports (almost) nothing changed; and a hand-broken service is repaired by re-running the playbook. Commit everything. **This is the substrate every later chapter stands on.**

**Try it.** Time a full `destroy`/`up` cycle and write the number in `versions.md`. In interviews, "my entire five-host platform rebuilds from Git in N minutes" is a concrete, memorable claim — make N a number you have measured.

---
## Chapter 2 — Self-hosted GitLab: the single source of truth

### THEORY 2.1 — What a forge is, and why we self-host one

A **forge** is the collaboration server built around Git hosting: repositories with access control, the merge-request/review workflow from Part 1, Chapter 3, issue tracking, a CI engine, and artifact storage. GitHub is the famous hosted one; **GitLab** is the one you can fully run yourself, which is why facilities standardise on it (DESY, CERN, ESRF, and most national labs operate large self-hosted GitLab instances).

Why self-host, rather than use gitlab.com? Recall Part 1, Theory 5.6: facility networks are isolated by design. The forge is the **single source of truth** for code, configuration, pipelines, *and* (from Chapter 6) the deployed state of production — so it must live inside the trust boundary, keep working air-gapped, and be operated, backed up, and upgraded by the control group itself. Operating a forge is therefore part of the job, not an inconvenience on the way to it.

One bundled component deserves special billing: the **container registry**. It is where the OCI images your pipelines build (Chapters 4 and 7) are stored and pulled from — the image counterpart of an apt repository. Co-locating source control, CI, and registry in one product keeps the whole software supply chain coherent and auditable: the same authentication, the same backup, one audit trail from commit to artifact.

### THEORY 2.2 — What installing GitLab actually involves

GitLab CE (Community Edition — the open-source core) ships as an **omnibus package**: one large `.deb` bundling GitLab plus its entire runtime (PostgreSQL, Redis, nginx, Sidekiq…), configured through a single file, `/etc/gitlab/gitlab.rb`, and applied with `gitlab-ctl reconfigure`. Notice the shape: *a declarative config file plus a reconcile command* — the idempotence idea again, implemented inside one product. Our Ansible role manages `gitlab.rb` and triggers reconfigure only when the file changes, using Ansible's **handler** mechanism (a task that runs at most once per play, only if notified).

**Callout — Why plain HTTP in the lab.** Production GitLab runs HTTPS with real certificates. In an isolated lab, TLS means running your own certificate authority and distributing its root to every host and container runtime — a worthwhile exercise, but a distraction from this chapter's point. The lab uses HTTP and notes the two places it costs us something (Podman needs the registry listed as "insecure"; browsers warn). Part 4's appendix sketches the local-CA upgrade; be ready to *say* in an interview that you know HTTP is a lab simplification.

### LAB 2.1 — The `gitlab` role

```yaml
# ansible/roles/gitlab/tasks/main.yml (essentials)
- name: Add GitLab CE apt repository            # vendor script adds repo + key —
  shell: curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | bash
  args: { creates: /etc/apt/sources.list.d/gitlab_gitlab-ce.list }   # idempotence guard

- name: Install GitLab CE
  apt: { name: gitlab-ce, state: present, update_cache: true }
  environment: { EXTERNAL_URL: "http://gitlab.lab" }    # first-run URL

- name: Configure external URLs in gitlab.rb
  blockinfile:
    path: /etc/gitlab/gitlab.rb
    block: |
      external_url 'http://gitlab.lab'
      registry_external_url 'http://gitlab.lab:5050'    # enables the container registry
  notify: reconfigure gitlab                            # handler runs ONLY on change

- name: Open GitLab ports (rule lives with the service — Ch. 1 discipline)
  community.general.ufw: { rule: allow, port: "{{ item }}", proto: tcp }
  loop: ['80', '5050', '22']
```

```yaml
# ansible/roles/gitlab/handlers/main.yml
- name: reconfigure gitlab
  command: gitlab-ctl reconfigure          # GitLab's own reconcile step (takes a few minutes)
```

Enable the role (uncomment `gitlab` in `site.yml`), commit, and apply:

```bash
git add -A && git commit -m "gitlab: CE + registry on svc"
vagrant provision --provision-with ansible     # GitLab install takes several minutes; svc needs its 6 GB
```

### LAB 2.2 — First login and lab DNS for your workstation

```bash
# Set the root password (prompts for a new one):
vagrant ssh svc -c "sudo gitlab-rake 'gitlab:password:reset[root]'"
```

Your *workstation* also needs the lab names (the VMs got them from the `common` role; your browser didn't):

```bash
echo "192.168.56.10 gitlab.lab registry.lab repo.lab grafana.lab prometheus.lab" | sudo tee -a /etc/hosts
```

Browse to `http://gitlab.lab`, log in as `root`. You are looking at your own private GitHub-equivalent, running on hardware you control, reachable by nothing outside your machine.

### LAB 2.3 — Groups, projects, keys, and the first push

Structure before content — in the UI:

1. Create a **group** `beamline` (a namespace for related projects, with shared permissions).
2. Inside it, create two blank projects: **`tango-devices`** (the control-system code — Part 3's payload) and **`platform-gitops`** (the Kubernetes manifests Argo CD will watch — Chapter 6's source of truth). The names are load-bearing: one repo holds *what the software is*, the other *what should be running*.
3. Add your SSH public key (Part 1, Lab 1.4): profile → SSH Keys → paste `~/.ssh/id_ed25519.pub`.

Prove the loop from your workstation:

```bash
cd ~/lab
git clone git@gitlab.lab:beamline/tango-devices.git
cd tango-devices
echo "# tango-devices — beamline control software" > README.md
git add . && git commit -m "init" && git push
```

Refresh the project page: your commit is there — served by your VM.

**Checkpoint 2:** you can push over SSH to `gitlab.lab`, both projects exist under `beamline`, and `http://gitlab.lab:5050` answers (the registry endpoint; logging into it comes with Podman in Chapter 4). Snapshot moment: this is a good point for `vagrant snapshot save post-gitlab` if you want cheap rollback while learning.

**Try it.** Open a merge request against your own README (branch, edit, push, MR, merge in the UI) purely to see the review surface you'll gate with CI in the next chapter.

---

## Chapter 3 — CI runners and your first pipelines

### THEORY 3.1 — The division of labour: GitLab decides, runners execute

GitLab's CI engine decides *what* should run — it reads `.gitlab-ci.yml` from the repository on every push and constructs a **pipeline**. But GitLab itself executes nothing: **runners** — separate agent processes, registered to the server — poll for jobs, execute them, and stream results back. This split matters operationally: runners are where the CPU is spent, where build tools live, and what you scale when pipelines queue. Facilities run fleets of them; your lab runs one, on `svc`.

Each runner has an **executor**, which defines the environment a job runs in:

- the **docker executor** (which drives Podman or Docker) starts a **fresh container per job** from an image the job names — clean, reproducible, nothing leaks between jobs. This is the professional default and what the lab uses.
- the **shell executor** runs jobs directly in the runner host's shell — simple, but every job inherits the residue of previous ones. Useful for special cases (we'll meet one in Chapter 7 when building `.deb`s); a smell everywhere else.

### THEORY 3.2 — The pipeline model: stages, jobs, artifacts, and the gate

A pipeline is a directed sequence of **stages** (e.g. `test → build → publish`); each stage contains **jobs** that run in parallel; a stage starts only if every job in the previous stage succeeded. Jobs can save **artifacts** — files that outlive the job (a built wheel, a `.deb`) and can be downloaded or consumed by later jobs.

Two configuration facts to internalise now, because they explain everything in later pipelines:

- **`tags`** route jobs to runners: a job tagged `lab` runs only on runners registered with that tag. This is how heterogeneous fleets work (GPU runners, packaging runners, HPC runners).
- **The gate is structural, not procedural.** Because `build` cannot start until `test` passes, and because GitLab can be configured to refuse merging an MR whose pipeline is red, the rule *"no artifact is published unless tests pass"* is enforced by the shape of the system — not by anyone's diligence. When you say that sentence in an interview, this chapter is what backs it.

### LAB 3.1 — The `runner` role

Registration requires a **token** that proves the runner may join your GitLab: in the UI, group `beamline` → Build → Runners → copy the registration token, and put it in `ansible/group_vars/services.yml` as `runner_token: "<value>"` (add this file to `.gitignore` — tokens are secrets and don't belong in Git; note the discipline, it recurs).

```yaml
# ansible/roles/runner/tasks/main.yml (essentials)
- name: Add runner apt repository
  shell: curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash
  args: { creates: /etc/apt/sources.list.d/runner_gitlab-runner.list }

- name: Install runner and a container engine
  apt: { name: [gitlab-runner, docker.io], state: present, update_cache: true }

- name: Register the runner (once)
  command: >
    gitlab-runner register --non-interactive
    --url "http://gitlab.lab/"
    --registration-token "{{ runner_token }}"
    --executor docker
    --docker-image ubuntu:22.04
    --description lab-runner
    --tag-list "lab,build"
  args: { creates: /etc/gitlab-runner/config.toml }    # idempotence guard again
```

Enable the role in `site.yml`, commit, `vagrant provision`. The runner should appear green under group Runners in the UI.

### LAB 3.2 — First pipeline: prove the plumbing

In your `tango-devices` clone:

```yaml
# .gitlab-ci.yml
stages: [test]

smoke:
  stage: test
  tags: [lab]              # route to our runner
  image: ubuntu:22.04      # the container this job runs in
  script:
    - echo "runner works"
    - uname -a
    - cat /etc/os-release | head -2
```

```bash
git add .gitlab-ci.yml && git commit -m "ci: smoke pipeline" && git push
```

Watch it in the UI: **Build → Pipelines** → the running job → live log. What you are seeing, spelled out: your push hit *your* GitLab, which instructed *your* runner, which pulled the `ubuntu:22.04` image and started a fresh container, ran your script inside it, streamed the log back, and reported green. Every pipeline for the rest of the book — tests, five-format package builds, image pushes — is this same mechanism with a longer script.

### LAB 3.3 — A real gate: tests that can fail

Make the pipeline mean something. Add a trivial Python package skeleton and a test:

```bash
mkdir -p src/beamline_hello tests
printf 'def add(a, b):\n    return a + b\n' > src/beamline_hello/__init__.py
printf 'from beamline_hello import add\n\ndef test_add():\n    assert add(2, 3) == 5\n' > tests/test_add.py
```

```yaml
# .gitlab-ci.yml (replace)
stages: [test]

unit-tests:
  stage: test
  tags: [lab]
  image: python:3.11
  script:
    - pip install pytest
    - PYTHONPATH=src pytest -q
```

Push — green. Now, on a **branch**, break the function (`return a - b`), push, and open a merge request: the MR shows a red pipeline, and the **Merge** button reflects it. Fix the bug on the branch, push, watch it turn green, merge. You have just walked the full quality gate: *broken code was physically visible as unmergeable*. Finally, in the project's Settings → Merge requests, enable "Pipelines must succeed" so the gate is policy, not etiquette.

**Checkpoint 3:** a green pipeline runs in a clean container on your self-hosted runner; a failing test blocks a merge request; and you can explain the GitLab/runner/executor division of labour without notes.

---

## Chapter 4 — Containers from zero: images, Podman, and the OCI model

Part 1 deferred containers with a promise; this chapter keeps it. You cannot understand Kubernetes (Chapter 5), OCI packaging (Chapter 7), or what your CI runner has been doing, without a precise model of what a container is — and "a lightweight VM" is not it.

### THEORY 4.1 — What a container actually is

A **container is a process** (or a small tree of processes) running on the host's own kernel, but wrapped in kernel-enforced isolation so that it *appears* to have a machine to itself:

- **namespaces** give it a private view of the system: its own process table (PID 1 inside is just some process outside), its own network stack, its own hostname, and — crucially — its own **root filesystem**;
- **cgroups** limit how much CPU, memory, and I/O it may consume;
- that private root filesystem comes from an **image**.

Contrast with a VM (Part 1, Ch. 2): a VM boots its own kernel on virtual hardware — heavyweight, minutes to boot, strong isolation. A container shares the host kernel — starts in milliseconds, costs almost nothing, isolation is good but kernel-shared. That is why the lab's *substrate* is VMs (they emulate facility hosts) while its *payload* runs in containers (they are the deployable unit).

An **image** is a frozen, layered filesystem plus metadata (which command to run, environment, exposed ports). Images are built from a recipe — a **Containerfile** (Docker calls it Dockerfile; same format) — and are **immutable and content-addressed**: an image tag like `python:3.11-slim` names a specific stack of layers, and the same image runs identically on your laptop, the CI runner, and a beamline host. *That* is the property everyone is buying: **the environment ships with the software.** "Works on my machine" stops being a sentence anyone says.

**OCI** (Open Container Initiative) is the standards body: the image format and runtime behaviour are specified, so images built by any tool run under any compliant runtime. Which brings us to tools:

| Tool | What it is | Why the lab uses it where |
|---|---|---|
| **Docker** | the original engine; a root daemon (`dockerd`) does everything | installed on the runner (the docker executor drives it) |
| **Podman** | daemonless, near-drop-in Docker replacement; can run rootless | your image-building tool — no root daemon is a real security posture difference facilities care about |
| **Apptainer** | HPC-oriented runtime; single-file images, never a root daemon | Chapter 7, for the compute-cluster format |
| **containerd** | the minimal runtime engines embed | what Kubernetes uses under the hood (Chapter 5) |

A **registry** is to images what an apt repository is to `.deb`s: a named, versioned, HTTP-served store that runtimes push to and pull from. Yours is GitLab's, at `registry` — er, at `gitlab.lab:5050` (Chapter 2 set it up; now you finally use it).

### LAB 4.1 — Run, inspect, and demystify

Work on `svc` (`vagrant ssh svc`), installing Podman first:

```bash
sudo apt install -y podman

# Run a container: pull an image and execute a process inside it
podman run --rm ubuntu:22.04 cat /etc/os-release | head -2
# --rm: delete the container (not the image) when the process exits

# It's just a process — prove it. In terminal 1:
podman run --rm -it ubuntu:22.04 sleep 300
# In terminal 2 (another vagrant ssh svc):
ps aux | grep "sleep 300"      # visible in the HOST's process table, as a normal process
podman ps                       # and in podman's view of running containers

# A private filesystem, a shared kernel:
podman run --rm ubuntu:22.04 ls /        # a whole Ubuntu root tree...
podman run --rm ubuntu:22.04 uname -r    # ...but the HOST's kernel version
```

That last pair of commands is the entire theory section in two lines: private userland, shared kernel.

### LAB 4.2 — Build an image and push it to your registry

```bash
mkdir -p ~/hello-image && cd ~/hello-image
cat > Containerfile <<'EOF'
FROM python:3.11-slim                    # start from this base image (a layer stack)
RUN pip install --no-cache-dir cowsay    # each RUN adds a layer
COPY greet.py /app/greet.py              # copy a file from the build context in
ENTRYPOINT ["python", "/app/greet.py"]   # what running this image executes
EOF
cat > greet.py <<'EOF'
import cowsay
cowsay.cow("built, shipped, and pulled — the same bits everywhere")
EOF

podman build -t hello:0.1 .
podman run --rm hello:0.1
podman images                            # see the layers' combined size
```

Now push it to *your* registry. HTTP (not HTTPS) means Podman must be told the registry is "insecure" — the lab-simplification cost flagged in Chapter 2:

```bash
sudo tee /etc/containers/registries.conf.d/lab.conf <<'EOF'
[[registry]]
location = "gitlab.lab:5050"
insecure = true
EOF

podman login gitlab.lab:5050             # your GitLab credentials
podman tag hello:0.1 gitlab.lab:5050/beamline/tango-devices/hello:0.1
podman push gitlab.lab:5050/beamline/tango-devices/hello:0.1
```

In the GitLab UI: project `tango-devices` → Deploy → Container Registry — your image, listed with its tag. Complete the loop by deleting it locally and pulling it back:

```bash
podman rmi hello:0.1 gitlab.lab:5050/beamline/tango-devices/hello:0.1
podman run --rm gitlab.lab:5050/beamline/tango-devices/hello:0.1   # pulled from the registry, runs
```

**Checkpoint 4:** you can explain container-vs-VM in two sentences, build an image from a Containerfile, and push/pull it against your own registry. Chapter 5 assumes all three.

**Try it.** `podman inspect hello:0.1 | less` — find the layer digests and the `Entrypoint`. Then change one line of `greet.py`, rebuild, and observe which layers were reused from cache and which rebuilt: layering is also a build-speed strategy, which will matter in CI.

---
## Chapter 5 — Kubernetes with kubeadm: a real cluster, the explicit way

### THEORY 5.1 — The problem Kubernetes solves

You can now run containers on one host with Podman. Scale the question: run *dozens of services*, as containers, across a *pool of hosts*, such that they restart when they crash, move when a host dies, find each other by name, and can be updated without downtime. Doing that by hand — deciding placement, writing systemd units per host, updating them all consistently — is exactly the kind of undifferentiated toil automation exists to kill.

**Kubernetes** (k8s) is the orchestrator that does it. Its contract is the declarative one you already know from Ansible and will meet again in Argo CD: you submit **manifests** describing desired state — "2 replicas of image X, exposing port Y" — and controllers **reconcile** the cluster toward that state, continuously, forever. Kill a container: it returns. Kill a node: its workloads reappear elsewhere. Kubernetes is not "a place to run containers"; it is a *reconciliation engine whose desired state happens to describe containers*. Hold that framing and Chapter 6 will feel inevitable.

### THEORY 5.2 — The moving parts you must know

A cluster splits into a brain and muscle:

**Control plane** (our `cp` VM) — the brain:
- **kube-apiserver** — the front door; *everything* (kubectl, nodes, controllers, Argo CD) talks to the cluster only through this API, on port 6443;
- **etcd** — the datastore where all cluster state (every manifest, every status) lives;
- **scheduler** — decides which node each new pod should run on;
- **controller-manager** — the reconciliation loops themselves.

**Worker nodes** (`w1`, `w2`) — the muscle:
- **kubelet** — the per-node agent: watches the API for pods assigned to its node and makes them real;
- **containerd** — the container runtime kubelet drives (Chapter 4's table pays off);
- **kube-proxy** — per-node network plumbing for Services.

**Objects** you'll declare as YAML:
- a **Pod** — the atom: one or more containers scheduled together, sharing an IP;
- a **Deployment** — "keep N replicas of this pod template running", with rolling updates;
- a **Service** — a stable name and virtual IP in front of a changing set of pods;
- a **Namespace** — a folder for objects (`argocd`, `monitoring`, `beamline`).

One more piece is not built in: the **CNI** (Container Network Interface) plugin, which gives every pod an IP address and routes pod-to-pod traffic *across* nodes. Nothing works until a CNI is installed; the lab uses **Flannel**, the simplest mainstream choice, with its conventional pod network `10.244.0.0/16`.

**Callout — Why kubeadm and not k3s (reprise, now with detail).** **kubeadm** is the official bootstrapper for a real multi-node cluster: it generates the certificate authority and certificates, starts the control-plane components, and hands you an explicit `join` command for workers — every architectural seam visible. One-command distributions (k3s, minikube, kind) weld those seams shut for convenience. Learn on kubeadm and you *understand* clusters; then k3s (a genuinely excellent single-binary distribution, worth an afternoon as a sidebar) becomes a convenience, not a mystery.

### THEORY 5.3 — The three prerequisites everyone gets wrong

Nearly every failed kubeadm install traces to one of three host-preparation facts. Read them now so the role below makes sense:

1. **Swap must be off.** The kubelet refuses to run with swap enabled (its memory accounting assumes none). Disable now *and* in `/etc/fstab` (or it returns at reboot).
2. **One cgroup driver.** cgroups (Ch. 4) have two management interfaces; kubelet and containerd must agree on `systemd`. containerd's default config says otherwise — one `sed` fixes it, and forgetting it produces nodes that join then flap.
3. **The pod CIDR must match the CNI.** `kubeadm init --pod-network-cidr=10.244.0.0/16` and Flannel's built-in `10.244.0.0/16` must agree, or pods get IPs no route serves.

Additionally two kernel switches: the `br_netfilter` and `overlay` modules loaded, and sysctls allowing bridged traffic through iptables and IP forwarding — cluster networking is built on both.

### LAB 5.1 — The `k8s-common` role (every cluster node)

```yaml
# ansible/roles/k8s-common/tasks/main.yml (essentials)
- name: Install containerd
  apt: { name: containerd, state: present, update_cache: true }

- name: Default containerd config with systemd cgroups     # prerequisite 2
  shell: |
    mkdir -p /etc/containerd
    containerd config default > /etc/containerd/config.toml
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
  args: { creates: /etc/containerd/config.toml }
  notify: restart containerd

- name: Disable swap now and in fstab                       # prerequisite 1
  shell: swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab

- name: Kernel modules and sysctls for cluster networking
  copy:
    dest: "{{ item.path }}"
    content: "{{ item.content }}"
  loop:
    - { path: /etc/modules-load.d/k8s.conf, content: "overlay\nbr_netfilter\n" }
    - { path: /etc/sysctl.d/k8s.conf,       content: "net.bridge.bridge-nf-call-iptables=1\nnet.ipv4.ip_forward=1\n" }

- name: Load modules and apply sysctls
  shell: modprobe overlay; modprobe br_netfilter; sysctl --system

- name: Add Kubernetes apt repo (v1.30) and install the tools
  shell: |
    mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
    echo "deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" > /etc/apt/sources.list.d/k8s.list
    apt-get update && apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl        # cluster upgrades are deliberate acts, not apt-upgrade side effects
  args: { creates: /usr/bin/kubeadm }

- name: Open cluster ports (rules live with the service)
  community.general.ufw: { rule: allow, port: "{{ item.port }}", proto: "{{ item.proto }}" }
  loop:
    - { port: '6443',  proto: tcp }   # API server
    - { port: '10250', proto: tcp }   # kubelet
    - { port: '8472',  proto: udp }   # Flannel VXLAN
```

Recognise the moves: the signed-third-party-repo dance from Part 1, Lab 7.2 (key → keyring → sources.list.d → install), `creates:` guards, and firewall-rules-with-the-service. Note also `apt-mark hold` — pinning made operational.

### LAB 5.2 — Control plane (`k8s-control`) and workers (`k8s-worker`)

```yaml
# ansible/roles/k8s-control/tasks/main.yml (essentials)
- name: kubeadm init                     # the big moment: CA, certs, control-plane up
  shell: >
    kubeadm init
    --apiserver-advertise-address=192.168.56.20     # bind to the LAB network, not NAT
    --pod-network-cidr=10.244.0.0/16                # prerequisite 3: must match Flannel
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

- name: Generate a join command for the workers
  shell: kubeadm token create --print-join-command
  register: join_cmd

- name: Stash the join command where workers can read it
  copy: { content: "{{ join_cmd.stdout }}", dest: /vagrant/join.sh, mode: '0755' }
  # /vagrant is the shared project folder every VM mounts — a lab convenience
```

```yaml
# ansible/roles/k8s-worker/tasks/main.yml (essentials)
- name: Join the cluster
  shell: "bash /vagrant/join.sh"
  args: { creates: /etc/kubernetes/kubelet.conf }
```

The `--apiserver-advertise-address` flag deserves a pause: every VM has *two* interfaces (NAT + lab network, Ch. 1), and kubeadm's default guess is the NAT one — which every node has under the *same* IP, producing a cluster that can't talk to itself. Pinning the lab address is the fix, and "which interface did it bind?" is a debugging question you'll now recognise for life.

Enable both roles in `site.yml`, commit, provision, then verify:

```bash
vagrant ssh cp
kubectl get nodes -o wide          # cp, w1, w2 — wait for STATUS "Ready" (CNI needs a minute)
kubectl get pods -A                # the control plane itself, running as pods; flannel on every node
```

### LAB 5.3 — First workload: watch reconciliation happen

```bash
# still on cp
kubectl create namespace beamline
kubectl -n beamline create deployment hello --image=nginx:alpine --replicas=2
kubectl -n beamline get pods -o wide         # two pods, spread across w1/w2

# Reconciliation, live — in one terminal:
kubectl -n beamline get pods -w              # -w = watch
# in another (vagrant ssh cp again): kill a pod by name:
kubectl -n beamline delete pod <one-of-them>
# watch the replacement appear within seconds: desired state 2, so 2 there shall be

# Expose and reach it:
kubectl -n beamline expose deployment hello --port=80
kubectl -n beamline run tmp --image=busybox:1.36 -it --rm --restart=Never -- wget -qO- http://hello
# ^ a throwaway pod resolves the Service BY NAME and fetches the nginx page

kubectl -n beamline delete deployment hello --cascade=foreground
kubectl -n beamline delete service hello
```

**Callout — Pitfall.** Nodes stuck `NotReady` → the CNI didn't come up: `kubectl -n kube-flannel get pods` and check the pod CIDR matches `10.244.0.0/16`. On VirtualBox private networks Flannel occasionally picks the NAT interface; pin it by adding `--iface=eth1` (the lab NIC) to the flannel container args in the DaemonSet. Pods `Pending` forever → describe them (`kubectl describe pod …`): usually insufficient RAM on the reduced lab — this is the promised place where dropping `w2` shows up.

**Checkpoint 5:** a three-node cluster (two on the reduced lab), all `Ready`, that reschedules killed pods within seconds and resolves Services by name — built *entirely by roles in Git*, meaning `vagrant destroy -f && vagrant up` now regenerates a working Kubernetes cluster from nothing. Confirm that claim once before proceeding; it is Part 2's most impressive single trick.

---

## Chapter 6 — GitOps with Argo CD: reconciliation all the way to Git

### THEORY 6.1 — Push vs pull, and why pull wins

You could deploy to the cluster from CI: a final pipeline job runs `kubectl apply`. Many shops do. This is **push-based** deployment, and it has structural problems: CI must hold cluster admin credentials (a juicy secret in the build system); nothing detects drift between deploys (a hand-edited cluster stays edited); and the deployed state is whatever the *history of pipeline runs* happened to produce — reconstructable only forensically.

**GitOps** inverts the arrow. An agent — **Argo CD** — runs *inside* the cluster and continuously **pulls** desired state from a Git repository, diffs it against reality, and reconciles. The consequences, now in operational language:

- **Git is the single source of truth.** To change production you commit — through a reviewed MR. There is no out-of-band `kubectl` in the process at all.
- **Drift self-heals.** Hand-edit or hand-delete something Argo CD manages and it reverts within its sync interval. (You will do this experimentally in the lab, because seeing it converts skeptics.)
- **Audit and rollback are `git log` and `git revert`.** The deployment history *is* version history.
- **Credentials stay home.** The cluster reaches out to Git read-only; CI never holds cluster keys.
- **Disaster recovery is repointing.** New cluster + same repo = same production.

Note the pattern completing itself: Ansible reconciles hosts to playbooks; Kubernetes reconciles pods to manifests; Argo CD reconciles manifests to Git. One idea, three scales — the sentence Part 1's Checkpoint 5 asked you to rehearse is now entirely built.

### THEORY 6.2 — Argo CD's one noun: the Application

Argo CD configuration is itself Kubernetes objects (of course it is). The one that matters is the **Application**: *watch path P of repo R at revision V, and keep namespace N of cluster C equal to it*. Its `syncPolicy` chooses the temperament: manual (a human clicks Sync — good for cautious production), or `automated` with `selfHeal` (drift reverts) and `prune` (objects deleted from Git get deleted from the cluster). The lab runs fully automated, because feeling the full loop is the point.

### LAB 6.1 — Install Argo CD

```bash
vagrant ssh cp
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server     # wait for it

# initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# expose the UI to your workstation (leave running in this terminal):
kubectl -n argocd port-forward --address 0.0.0.0 svc/argocd-server 8080:443
```

Browse `https://192.168.56.20:8080` (accept the self-signed-certificate warning), log in as `admin`.

### LAB 6.2 — The GitOps repo and the Application

Populate `platform-gitops` (created in Chapter 2 — its moment has come) from your workstation:

```bash
cd ~/lab && git clone git@gitlab.lab:beamline/platform-gitops.git && cd platform-gitops
mkdir -p manifests
cat > manifests/hello.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: hello, namespace: beamline }
spec:
  replicas: 2
  selector: { matchLabels: { app: hello } }
  template:
    metadata: { labels: { app: hello } }
    spec:
      containers:
        - name: hello
          image: nginx:alpine
          ports: [ { containerPort: 80 } ]
EOF
git add -A && git commit -m "manifests: hello deployment" && git push
```

Register the Application (on `cp`, save as `app.yaml` and `kubectl apply -f app.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: beamline-apps, namespace: argocd }
spec:
  project: default
  source:
    repoURL: http://gitlab.lab/beamline/platform-gitops.git   # HTTP clone URL; public repo or add a read token
    targetRevision: main
    path: manifests
  destination: { server: https://kubernetes.default.svc, namespace: beamline }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ CreateNamespace=true ]
```

(If the repo is private, the quickest lab route is Settings → General → Visibility → Public inside GitLab; the production route is a read-only deploy token registered in Argo CD — do that version if you have ten spare minutes, it's one `argocd repo add`.)

In the Argo CD UI the Application appears, syncs, and turns green; `kubectl -n beamline get pods` shows the deployment — which you never applied by hand.

### LAB 6.3 — The three demonstrations

These three experiments *are* GitOps; run all of them:

**1. Deploy by commit.** Edit `manifests/hello.yaml`: `replicas: 3`. Commit, push. Within ~3 minutes (or instantly with the UI's Refresh) a third pod exists. You changed production by changing Git.

**2. Self-healing.** Sabotage: `kubectl -n beamline delete deployment hello`. Watch the Argo CD UI notice (OutOfSync) and restore it. Reality drifted; Git won.

**3. Rollback.** `git revert HEAD && git push` in the gitops repo — production walks back to 2 replicas, and the revert itself is an audited commit.

**Checkpoint 6:** editing YAML in Git changes the cluster with no manual `kubectl`; deleting a managed object triggers automatic restoration; a `git revert` is a rollback. From this point on in the handbook, *nothing is ever `kubectl apply`'d by hand again* except Argo CD's own bootstrap — everything cluster-bound goes through `platform-gitops`.

**Try it.** In the UI, click into the Application's resource tree and its diff view. Being able to *show* a live diff between Git and cluster is a demo that lands well in interviews.

---
## Chapter 7 — Multi-format software packaging: one component, five artifacts

This chapter is the heart of the target role — the DESY posting's first bullet is "development and maintenance of CI/CD-based build and distribution systems for scientific software such as Debian, Conda, Pip, or Docker packages" — and it is where the lab's learning-per-hour peaks. You will take one small Python component and produce **five** installable artifacts from it, understanding at each step what the format adds and who consumes it.

### THEORY 7.1 — Why one component needs many formats

Recall the Part 1 table and now make it concrete with the lab's own future components:

- The **HDB++ archiver** must run on a beamline *host*, start at boot under systemd, and be upgraded with the OS → native **`.deb`**, installed from the facility apt repository.
- A **beam-analysis library** lives in scientists' Python environments alongside numpy and a compiled FFT library — dependencies apt knows nothing about → **Conda**, from a facility channel.
- A pure-Python **utility library** other code imports → **Pip/wheel**, the format Python tooling natively consumes.
- The **simulator device servers** deploy on the Kubernetes cluster → **OCI image**, pulled from the registry.
- The same analysis code must also run on a shared **HPC cluster** where users have no root and Docker daemons are banned → **Apptainer `.sif`**, a single file a user can copy and run.

Same source; five consumers; five formats. The professional skill is not knowing one toolchain — it is knowing **which format answers which deployment question**, and wiring CI to produce them all from one commit (which Part 4 does; this chapter builds each by hand first, because you cannot debug in CI what you have never run locally).

### THEORY 7.2 — Anatomy of each format, one paragraph each

**Wheel (`.whl`).** A zip of your Python package plus metadata (`pyproject.toml` is the modern source of that metadata: name, version, dependencies, entry points). `pip` installs it into whatever Python environment is active. Entry points matter: `[project.scripts]` turns a Python function into a shell command upon install — how CLI tools ship.

**Conda package (`.conda`).** Built from a **recipe** (`meta.yaml`): metadata plus a build script, with dependencies resolved from conda **channels** — and, crucially, those dependencies may be *non-Python* (compilers, C libraries, CUDA). This is why scientific Python standardised on it, and why **conda-forge** — the community channel with thousands of maintained recipes, including the entire Tango stack — is Path A of this whole handbook.

**Debian package (`.deb`).** The richest format: files + metadata + dependency relationships against the *system's* package universe + **maintainer scripts** (run at install/remove) + the ability to ship systemd units, create users, and register config files. Built from a `debian/` directory whose complexity is tamed by **debhelper** (automation of the dozens of policy steps) and **dh-python/pybuild** (the Python-specific knowledge). The result installs with `apt`, integrates with everything Part 1 Chapter 1 taught, and is how a beamline *host* consumes software.

**OCI image.** Chapter 4 covered it: the environment ships with the software; the registry distributes it; Kubernetes runs it.

**Apptainer (`.sif`).** A container image flattened into one file, built from a definition file that can bootstrap *from* an OCI image (so the OCI build is reused). Runs without any daemon and without root — the properties HPC centres require. Common at facilities for exactly the analysis-on-the-compute-cluster case.

### LAB 7.1 — The sample component

On your workstation, inside the `tango-devices` clone (this code is the stand-in that Part 4 swaps for real device servers):

```
beamline-utils/
├── pyproject.toml
├── src/beamline_utils/__init__.py
└── src/beamline_utils/cli.py
```

```toml
# beamline-utils/pyproject.toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "beamline-utils"
version = "0.1.0"
description = "Lab stand-in for a control-system component"
requires-python = ">=3.10"

[project.scripts]
beamline-info = "beamline_utils.cli:main"     # installs a shell command
```

```python
# beamline-utils/src/beamline_utils/cli.py
def main():
    print("beamline-utils 0.1.0 — OK")
```

(`__init__.py` may be empty.) Commit it.

### LAB 7.2 — Format 1: Pip / wheel

```bash
cd beamline-utils
python3 -m pip install --upgrade build
python3 -m build                    # → dist/beamline_utils-0.1.0-py3-none-any.whl (+ .tar.gz sdist)

# prove it installs and delivers its entry point, in a throwaway venv:
python3 -m venv /tmp/v && /tmp/v/bin/pip install dist/*.whl
/tmp/v/bin/beamline-info            # "beamline-utils 0.1.0 — OK"
```

### LAB 7.3 — Format 2 (Path A): Conda

With miniforge installed (one installer from conda-forge; if you don't have it, `wget` the Miniforge3 installer and run it):

```bash
conda install -y conda-build
mkdir -p conda-recipe
cat > conda-recipe/meta.yaml <<'EOF'
package: { name: beamline-utils, version: "0.1.0" }
source: { path: .. }
build:
  script: {{ PYTHON }} -m pip install . -vv
  entry_points: [ "beamline-info = beamline_utils.cli:main" ]
requirements:
  host: [ python, pip, setuptools ]   # needed AT BUILD time
  run:  [ python ]                    # needed AT RUN time — the dependency declaration
EOF
conda build conda-recipe/             # → a .conda artifact (path: conda build ... --output)
```

The `host:`/`run:` split is the concept to retain: conda distinguishes build-time from run-time dependencies explicitly, which is what lets it ship compiled software portably.

### LAB 7.4 — Format 3 (Path B): the native Debian `.deb`

The centrepiece. Do this one on `svc` or a lab VM (Debian tooling belongs on a Debian-family host, and building on the platform is more honest anyway):

```bash
sudo apt install -y devscripts debhelper dh-python python3-all pybuild-plugin-pyproject dh-make
cd beamline-utils
dh_make --python -y -p beamline-utils_0.1.0     # scaffolds the debian/ directory
```

`debian/` is the package's control room. Two files matter today:

```bash
# debian/control — the metadata (edit to:)
#   Source: beamline-utils
#   Build-Depends: debhelper-compat (= 13), dh-python, python3-all, python3-setuptools, pybuild-plugin-pyproject
#   Package: python3-beamline-utils
#   Depends: ${python3:Depends}, ${misc:Depends}     ← substituted automatically at build

# debian/rules — the build program (replace entirely with:)
cat > debian/rules <<'EOF'
#!/usr/bin/make -f
%:
	dh $@ --with python3 --buildsystem=pybuild
EOF
chmod +x debian/rules
```

Pause on `debian/rules`: it is three lines because `dh` (debhelper) sequences the ~40 standardised steps of a policy-correct build, and `pybuild` teaches it Python. Before these tools, each step was hand-written per package; this is what "debhelper/dh-python" on a CV actually names. Build and inspect:

```bash
dpkg-buildpackage -us -uc -b        # -us -uc: unsigned for now (Ch. 8 signs); -b: binary only
ls ../*.deb                          # ../python3-beamline-utils_0.1.0-1_all.deb
dpkg-deb -I ../python3-beamline-utils_*.deb    # metadata — find your Depends line
dpkg-deb -c ../python3-beamline-utils_*.deb    # contents — where files will land
sudo apt install ../python3-beamline-utils_*.deb && beamline-info
```

**Callout — Pitfall.** `dpkg-buildpackage` failing with "no build system detected" → the `pybuild-plugin-pyproject` package is missing (it teaches pybuild about `pyproject.toml`-only projects). Lintian warnings about the changelog are normal for a lab package; read one or two — lintian is Debian's built-in reviewer, and skimming its complaints is a packaging education in itself.

### LAB 7.5 — Format 4: OCI image, into your registry

Chapter 4 made this routine; note the pattern of installing *the wheel you already built* rather than copying source — build once, reuse everywhere:

```dockerfile
# beamline-utils/Containerfile
FROM python:3.11-slim
COPY dist/*.whl /tmp/
RUN pip install --no-cache-dir /tmp/*.whl
ENTRYPOINT ["beamline-info"]
```

```bash
podman build -t gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0 .
podman login gitlab.lab:5050
podman push gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0
podman run --rm gitlab.lab:5050/beamline/tango-devices/beamline-utils:0.1.0
```

### LAB 7.6 — Format 5: Apptainer `.sif`

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
apptainer run beamline-utils.sif        # runs as YOU, no daemon, no root — the HPC point
ls -lh beamline-utils.sif               # one file; scp-able to any cluster
```

**Checkpoint 7:** from one source tree you hold a wheel, a `.conda`, a `.deb`, an OCI image *in your registry*, and a `.sif` — and for each you can answer, in one sentence, "who installs this, and why this format for them?" Commit everything including `debian/` and the recipes: *packaging definitions are code*.

---

## Chapter 8 — The signed distribution registry: reprepro and a conda channel

### THEORY 8.1 — Distribution is half of packaging

Chapter 7's artifacts sit in directories. That is not delivery. Delivery means a clean host can say `apt install beamline-utils` or `conda install -c <channel> beamline-utils` and receive the right version, dependency-resolved, **with cryptographic proof of origin** — the consumer experience you've enjoyed since Part 1, Lab 1.2, now provided *by you*. Two mechanisms:

- A **repository/channel** is an HTTP-served directory *plus indexes* — metadata files listing every package, version, and dependency, which is what lets clients resolve by name instead of URL. For apt, the index format is elaborate and managed by **reprepro** (a repository manager: you feed it `.deb`s, it maintains a policy-correct pool and index and signs the result). For conda, `conda-index` generates `repodata.json` per platform subdirectory.
- **GPG signing** closes the trust loop. You generate a key pair; reprepro signs the repository's `Release` index with the private key; clients install your *public* key and apt verifies every update against it. On a facility network this is the control preventing a tampered or rogue package from ever reaching a beamline host — supply-chain security in its most concrete form. You performed the client half for HashiCorp and Kubernetes upstream; today you are the publisher.

### LAB 8.1 — A signed apt repository with reprepro (the `repo` role, shown as commands)

On `svc` (fold these into `ansible/roles/repo/` as you go — by now translating commands into idempotent tasks is a skill you have):

```bash
sudo apt install -y reprepro nginx

# 1. The signing identity (no passphrase for lab automation; production uses one + an agent):
gpg --quick-generate-key "Lab Repo <repo@lab>" default default never
KEY=$(gpg --list-keys --with-colons repo@lab | awk -F: '/^fpr:/{print $10; exit}')

# 2. Repository definition — reprepro's one config file:
mkdir -p ~/apt-repo/conf
cat > ~/apt-repo/conf/distributions <<EOF
Origin: lab
Label: lab
Codename: jammy                      # matches the Ubuntu 22.04 clients
Architectures: amd64 source
Components: main
Description: Lab control-system apt repo
SignWith: $KEY                       # ← the signing hookup
EOF

# 3. Ingest the Chapter-7 package (reprepro pools it, indexes it, signs the Release):
reprepro -b ~/apt-repo includedeb jammy ~/python3-beamline-utils_0.1.0-1_all.deb
reprepro -b ~/apt-repo list jammy    # confirm

# 4. Serve over HTTP and export the public key for clients:
sudo ln -s ~/apt-repo /var/www/html/apt
gpg --armor --export repo@lab | sudo tee /var/www/html/apt/lab.gpg.asc >/dev/null
```

Now the consumer side, on a different clean VM (`vagrant ssh tango` — its first job!). Watch how exactly this mirrors adding the HashiCorp repo in Part 1:

```bash
curl -fsSL http://repo.lab/apt/lab.gpg.asc | sudo gpg --dearmor -o /usr/share/keyrings/lab.gpg
echo "deb [signed-by=/usr/share/keyrings/lab.gpg] http://repo.lab/apt jammy main" \
  | sudo tee /etc/apt/sources.list.d/lab.list
sudo apt update                      # verifies YOUR signature; a tampered repo fails here
sudo apt install -y python3-beamline-utils
beamline-info                        # your component, delivered like any system package
```

**Try it (the security demo).** Corrupt one byte of the served `.deb` in the pool (`printf 'X' | sudo dd of=<pool-file> bs=1 seek=100 conv=notrunc`), then `apt install --reinstall` on the client: apt *refuses* — hash mismatch against the signed index. Undo by re-running `includedeb`. Thirty seconds, and you have witnessed the entire point of signing.

### LAB 8.2 — The conda channel

```bash
mkdir -p ~/conda-channel/linux-64
cp "$(conda build conda-recipe/ --output)" ~/conda-channel/linux-64/
conda install -y conda-index
python -m conda_index ~/conda-channel          # writes repodata.json indexes
sudo ln -s ~/conda-channel /var/www/html/conda
```

Client (any host with conda/miniforge):

```bash
conda install -y -c http://repo.lab/conda beamline-utils
beamline-info
```

(Conda-side signing/verification is a weaker, less standardised story than apt's — worth knowing and saying honestly; the channel's trust in practice comes from serving it inside the trusted network and, in stricter setups, TLS + checksummed lockfiles.)

**Checkpoint 8:** a clean lab host installs your component **by name** from your signed apt repository — signature verified, tampering demonstrably rejected — and from your conda channel. The producer-to-consumer chain is closed with no cloud anywhere in it. This checkpoint, told as a story, is the strongest three minutes in a demo of this platform.

---

## Chapter 9 — Observability: Prometheus, Grafana, Alertmanager

### THEORY 9.1 — The mechanics: exporters, scraping, time series, rules

Part 1 gave the philosophy (metrics, Golden Signals); now the machinery. **Prometheus** works by **pull**: every observed component exposes an HTTP endpoint, `/metrics`, listing current values in a plain-text format —

```
node_memory_MemAvailable_bytes 8.234567e+09
node_cpu_seconds_total{cpu="0",mode="idle"} 123456.78
```

— and Prometheus **scrapes** all such endpoints every N seconds, storing each value as a **time series** identified by its name plus **labels** (the `{…}` key-values, which let one metric name cover many CPUs, hosts, or endpoints). Components that don't speak `/metrics` natively get an **exporter** sidekick: `node_exporter` translates a Linux host's vitals; databases, GitLab, and Tango (Part 4) all have exporter stories.

Queries and alerts are written in **PromQL**. You need a reading knowledge of three shapes today:

```promql
node_memory_MemAvailable_bytes                          # current value, per host (per label set)
rate(node_cpu_seconds_total{mode!="idle"}[5m])          # per-second rate over a 5-min window
1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes   # arithmetic between series
```

An **alerting rule** is a PromQL expression plus a `for:` duration (the anti-flapping delay) and severity labels. Prometheus evaluates rules continuously and hands *firing* alerts to **Alertmanager**, whose separate job is human-facing routing: grouping related alerts, deduplicating, silencing during maintenance, and delivering to email/webhooks/chat. **Grafana** is the third leg: dashboards querying Prometheus, for humans watching trends rather than waiting for pages.

In Kubernetes-land the whole stack ships as one bundle, **kube-prometheus-stack**, installed with **Helm** — a package manager for Kubernetes applications (a "chart" is a parameterised bundle of manifests; think apt for the cluster, a tool worth meeting once even though our GitOps flow can also render charts via Argo CD).

### LAB 9.1 — Install the stack

```bash
vagrant ssh cp
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
kubectl create namespace monitoring
helm install kps prometheus-community/kube-prometheus-stack -n monitoring
kubectl -n monitoring rollout status deploy/kps-grafana

# expose Grafana to your workstation (default login: admin / prom-operator):
kubectl -n monitoring port-forward --address 0.0.0.0 svc/kps-grafana 3000:80
```

Browse `http://192.168.56.20:3000`. Out of the box the bundle already scrapes the cluster itself — explore the prebuilt Kubernetes dashboards and find your `beamline` namespace's pods.

### LAB 9.2 — Watch the hosts too: node_exporter everywhere

Cluster metrics aren't enough; `svc` and `tango` are the hosts most likely to hurt (GitLab is heavy; the archiver will fill disks). Add to `ansible/roles/observability/tasks/main.yml`:

```yaml
- name: Install node_exporter
  apt: { name: prometheus-node-exporter, state: present, update_cache: true }
- name: Enable node_exporter
  systemd: { name: prometheus-node-exporter, enabled: true, state: started }
- name: Open its port
  community.general.ufw: { rule: allow, port: '9100', proto: tcp }
```

Apply it to **all** hosts (move `observability` into the `hosts: all` play in `site.yml`), provision, and verify a raw scrape target by hand — always look at one `/metrics` in your life:

```bash
curl -s http://tango.lab:9100/metrics | grep node_memory_MemAvailable
```

Then teach Prometheus about the VMs with an *additional scrape config* (the kube-prometheus-stack accepts one via Helm values; the shape is):

```yaml
# values-extra-scrapes.yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: lab-hosts
        static_configs:
          - targets: ["192.168.56.10:9100","192.168.56.30:9100"]
```

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring -f values-extra-scrapes.yaml
```

In Grafana, import dashboard **1860** (the standard node-exporter board) and watch your five hosts breathe.

### LAB 9.3 — A Golden-Signals alert, fired on purpose

A saturation rule, delivered as a `PrometheusRule` object — and delivered **through GitOps**: put it in `platform-gitops/manifests/`, let Argo CD apply it, because monitoring configuration is production configuration:

```yaml
# platform-gitops/manifests/golden-signals-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: golden-signals
  namespace: monitoring
  labels: { release: kps }          # so the stack's Prometheus picks it up
spec:
  groups:
    - name: golden-signals
      rules:
        - alert: HostHighMemory
          expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
          for: 2m
          labels: { severity: warning }
          annotations:
            summary: "Host {{ $labels.instance }} memory > 90%"
            runbook: "http://gitlab.lab/beamline/platform-gitops/-/blob/main/docs/runbook.md#high-memory"
```

Commit, push, watch Argo CD sync it. Then cause the condition:

```bash
vagrant ssh tango -c "sudo apt install -y stress && stress --vm 4 --vm-bytes 1024M --timeout 240"
```

Within ~2–3 minutes: Prometheus (port-forward 9090, Alerts page) shows `HostHighMemory` pending → firing, and Alertmanager receives it. Let the stress end and watch it resolve.

### LAB 9.4 — The runbook (a deliverable, not an afterthought)

An alert without a documented response is half a system. Create `platform-gitops/docs/runbook.md` — the one-pager an on-call responder at 3 a.m. follows — as a symptom → checks → fixes table. Seed it with entries you can already write from experience: *high memory* (this lab), *service down* (`systemctl status`/`journalctl -u`, Part 1 Lab 1.3), *node NotReady* (Ch. 5's pitfall), *disk filling* (find growth with `du -xh --max-depth=1 /var | sort -h`), *Argo CD OutOfSync* (diff view, then decide: revert Git or accept). Every alert you ever add gets a `runbook:` annotation pointing at its section — the link you already planted above.

**Callout — Why the runbook counts.** It is the artifact that proves you think operationally, it directly answers the posting's "ensuring reliable operation" language, and — practically — writing symptoms-to-fixes down is the fastest way to consolidate everything Part 2 taught you.

**Checkpoint 9:** Grafana shows live host and cluster dashboards; a deliberately induced condition fires an alert that lands in Alertmanager and resolves afterwards; the alert links to a runbook section that exists; and the rule itself arrived via GitOps.

---

## Chapter 10 — End of Part 2: review, self-assessment, and the demo script

### What you now have

A reproducible, fully open-source, on-prem platform, every layer defined in Git:

| Layer | Implementation | Chapter |
|---|---|---|
| Hosts as code | Vagrant (5 VMs) + Ansible roles, idempotence proven | 1 |
| Source of truth | self-hosted GitLab CE + container registry | 2 |
| CI with a quality gate | GitLab CI on a self-hosted docker-executor runner; red pipelines block merges | 3 |
| Container fluency | Podman builds; images pushed/pulled against your registry | 4 |
| Orchestration | 3-node kubeadm cluster (containerd, Flannel), rebuildable from Git | 5 |
| GitOps delivery | Argo CD auto-sync + self-heal + prune from `platform-gitops` | 6 |
| Packaging | one component as wheel, conda, `.deb`, OCI, `.sif` | 7 |
| Distribution | GPG-signed reprepro apt repo + indexed conda channel, tamper-rejection demonstrated | 8 |
| Observability | kube-prometheus-stack + node_exporter fleet + Golden-Signals alert + runbook | 9 |

### The 10-minute demo script (rehearse it)

1. `vagrant status` — "five hosts, all from this Vagrantfile; the whole lab rebuilds from Git in *N* minutes."
2. Push a failing test to a branch → show the red MR, unmergeable. Fix → green → merge. "The gate is structural."
3. `apt install python3-beamline-utils` on a clean host from *your* signed repo; mention the tamper demo.
4. Bump `replicas` in `platform-gitops` → Argo CD rolls it. Delete the deployment by hand → it returns. "`git log` is my deployment history."
5. Grafana: hosts and cluster live; show the alert with its runbook link. "And when it pages, the response is written down."

### Self-assessment (answer without notes; section references in brackets)

1. A VM exists but nginx isn't installed on it — which layer failed, provisioning or configuration, and which tool owns it? *(1.1)*
2. What are Ansible's four nouns, and where does idempotence actually live? *(1.2)*
3. What does `args: { creates: … }` do and why do our shell tasks carry it? *(1.2)*
4. Why do firewall rules live inside each service's role? *(1.3)*
5. What two experiments prove IaC is working, and what output demonstrates each? *(1.4)*
6. Give three reasons a facility self-hosts its forge rather than using gitlab.com. *(2.1)*
7. What is a handler, and why does `gitlab-ctl reconfigure` run as one? *(2.2, Lab 2.1)*
8. GitLab vs runner vs executor — who decides, who executes, who defines the environment? *(3.1)*
9. Why is the docker executor preferred over shell, and when might shell still be right? *(3.1)*
10. Explain mechanically how "no artifact ships without passing tests" is enforced by structure. *(3.2, Lab 3.3)*
11. Container vs VM in two sentences; which two kernel mechanisms make containers possible? *(4.1)*
12. Why does the lab use Podman for building but containerd in the cluster? *(4.1, 5.2)*
13. Name the control-plane components and each one's job. *(5.2)*
14. The three kubeadm prerequisites everyone gets wrong — and the symptom each produces when missed. *(5.3)*
15. Why did `kubeadm init` need `--apiserver-advertise-address` in this lab specifically? *(Lab 5.2)*
16. Push vs pull deployment: three structural advantages of pull. *(6.1)*
17. What do `selfHeal` and `prune` each do in an Argo CD Application? *(6.2)*
18. For each of the five formats: who consumes it and why that format for them? *(7.1)*
19. What do debhelper and dh-python each contribute to a three-line `debian/rules`? *(Lab 7.4)*
20. Walk the trust chain from your GPG key to a client's successful `apt update`. What breaks if one served byte changes? *(8.1, Try-it)*
21. How does Prometheus obtain metrics from a Linux host that has no idea Prometheus exists? *(9.1, Lab 9.2)*
22. Name the Golden Signals; which one was your lab alert, and what does `for: 2m` prevent? *(9.1, Lab 9.3)*
23. Why was the alert rule delivered through `platform-gitops` rather than `kubectl apply`? *(Lab 9.3, 6.3)*
24. Ansible, Kubernetes, Argo CD: state the "one idea, three scales" in your own words. *(6.1)*

### Glossary additions (extends the Part 1 glossary)

| Term | Definition |
|---|---|
| **Application (Argo CD)** | the object binding a Git repo/path/revision to a cluster/namespace, with a sync policy |
| **artifact (CI)** | a file a job preserves beyond its run (a wheel, a `.deb`) for download or later jobs |
| **cgroups / namespaces** | kernel mechanisms for resource limits / private views — together, what a container is made of |
| **channel (conda)** | an indexed HTTP directory of conda packages; conda-forge is the community one; Ch. 8 built yours |
| **CNI / Flannel** | the pod-networking plugin interface / the simple implementation this lab uses (VXLAN, 10.244.0.0/16) |
| **containerd** | the minimal container runtime kubelet drives |
| **control plane / worker** | the cluster's brain (API server, etcd, scheduler, controllers) / where pods actually run |
| **Deployment / Pod / Service / Namespace** | keep-N-replicas controller / the scheduling atom / stable name+IP in front of pods / object folder |
| **executor (runner)** | the job environment strategy: docker (fresh container per job) or shell (on the host) |
| **exporter / node_exporter** | a `/metrics` translator for software that lacks one / the standard Linux-host exporter on :9100 |
| **fact (Ansible)** | discovered host information available to tasks and templates |
| **forge** | the Git-hosting collaboration server: repos, MRs, CI, registry — here, self-hosted GitLab CE |
| **handler (Ansible)** | a task run at most once per play, only when notified — e.g. restart-on-config-change |
| **Helm / chart** | Kubernetes's application package manager / a parameterised bundle of manifests |
| **kubeadm join / token** | the command (with time-limited token) that enrols a worker into the cluster |
| **kubectl** | the CLI for the Kubernetes API |
| **omnibus package** | GitLab's all-in-one `.deb` configured via `gitlab.rb` + `gitlab-ctl reconfigure` |
| **pipeline / stage / job / tag** | the CI run / its ordered phases / parallel units within a phase / the label routing jobs to runners |
| **PromQL** | Prometheus's query language for time series and alert expressions |
| **PrometheusRule** | the Kubernetes object carrying alerting rules into the kube-prometheus-stack |
| **registry (container)** | the versioned HTTP store for OCI images — GitLab's, at gitlab.lab:5050 |
| **reprepro** | the apt-repository manager: ingests `.deb`s, maintains pool + indexes, signs the Release file |
| **rolling update** | a Deployment replacing pods gradually so the service never fully stops |
| **scrape / target** | Prometheus pulling `/metrics` / one endpoint it pulls from |
| **`Vagrantfile` / box** | the fleet definition file / the pre-built base image VMs are stamped from |

### Where Part 3 begins

The platform is complete and idle — a delivery machine with nothing of substance to deliver. **Part 3 — The Tango Controls System** builds the payload on the `tango` VM: the Tango database, your first PyTango device server (a magnet power supply with a real state machine and a simulator), then the full simulated beamline — camera, Faraday cup, vacuum, interlocks — orchestrated by Sardana, faced by Taurus GUIs, and archived by HDB++. Part 4 then feeds it all through the machine you just built.

---

*End of Part 2 of 4.*
