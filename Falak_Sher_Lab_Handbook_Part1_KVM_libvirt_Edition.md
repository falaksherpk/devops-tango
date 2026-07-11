# Building a Reproducible On-Prem DevOps Platform and a Tango Controls System

## PART 1 of 4 — Foundations from Zero: Linux, Virtualization (KVM/libvirt), Git, Control Systems, and the DevOps Mental Model

*A from-scratch lab handbook. Part 1 assumes **no prior knowledge** of Linux, virtualization, Git, DevOps, or control systems. By its end you will have a working lab foundation on your own hardware — a hypervisor host and six virtual machines built the way a real facility builds them — and the complete mental model needed for Parts 2–4.*

> **About this edition.** This is the **KVM/libvirt edition** of Part 1. An earlier draft used Vagrant + VirtualBox to create the lab VMs. This edition replaces that with the tooling a bare-metal Linux facility actually uses: **KVM/libvirt** as the hypervisor, a **golden-image-plus-clone** workflow (`virt-install`, `virt-sysprep`, `virt-clone`) instead of Vagrant boxes, and a **manual-first** philosophy — you build everything by hand first to understand it deeply, then automate it with Ansible and cloud-init in a later pass. The conceptual chapters (Linux, Git, control systems, the DevOps pillars) are unchanged, because concepts don't depend on which hypervisor runs underneath. Only the *how-you-build-the-machines* chapters (2, 6, 7) differ.

---

## How to use this book

This handbook has four parts:

| Part | Title | What you build |
|---|---|---|
| **Part 1** | Foundations from Zero *(this document)* | The mental model + a KVM/libvirt host, a golden image, and six configured VMs with a control node |
| **Part 2** | The On-Prem DevOps Platform | GitLab, CI runners, Kubernetes, Argo CD, multi-format packaging, signed repositories, observability |
| **Part 3** | The Tango Controls System | A simulated 6–20 MeV LINAC beamline: PyTango device servers, Sardana, Taurus, HDB++ |
| **Part 4** | Integration, Troubleshooting, Schedule | One commit flowing end-to-end; a troubleshooting compendium; a realistic build plan |

This is a **dual-track handbook**. Every chapter is split into two kinds of section:

- **THEORY** sections explain *why* a technology exists, how it works, and what problem it solves. In Part 1 they are written for a genuine newcomer: no Linux, no cloud, no programming-operations background is assumed. If you already know a topic, skim the theory and jump to the labs.
- **LAB** sections are hands-on, command-by-command, and meant to be typed and run in order. In Part 1 the labs also teach the *commands themselves* — what each flag means and what output to expect — because we assume you have never opened a terminal before.

The goal is not to make things *work once*. The goal is to make you able to **rebuild everything from a clean machine and explain every layer**. That is the difference between having done something and understanding it — and it is exactly what a hiring panel at a research facility probes for.

### Conventions used throughout

| Element | Meaning |
|---|---|
| **THEORY** heading | conceptual explanation; safe to skim if familiar |
| **LAB** heading | hands-on steps to run, in order |
| `monospace` | commands, filenames, code, values you type |
| **Run on `<machine>`:** | tells you *which* machine a command belongs to — critical once you have a host plus six VMs |
| **Checkpoint** | a concrete, testable "this now works" milestone — do not continue past a failed checkpoint |
| **Callout — Why** | the reasoning behind a design choice |
| **Callout — Pitfall** | a common failure and how to avoid it (many drawn from real trouble hit during this build) |
| **Try it** | a small optional experiment that deepens understanding |
| **Path A / Path B** | the two install routes used from Part 2 onward: **A = conda-forge**, **B = apt/`.deb`** |

**A note on prompts.** Because this lab has many machines (a host called `labhost` and six VMs), every command block names the machine it runs on. When you copy a command, copy only the command itself — never a shell prompt like `falak@labhost:~$`. Pasting the prompt text into the shell produces `command not found`; naming the target machine separately avoids that.

### The two install paths (read this once, matters from Part 2 on)

Wherever the conda-forge and Debian/apt routes genuinely differ, the book shows both:

- **Path A — conda-forge.** Fastest, most portable, fewest packaging quirks. Everything installs into a conda environment. Choose this if you just want the system working quickly.
- **Path B — apt / `.deb`.** Closest to how a real Debian-based beamline host is operated: system packages, systemd units, native `.deb` artifacts served from a signed apt repository. Slower, but it teaches the packaging skills at the centre of the DevOps role this handbook targets.

### What you will build (the destination)

Two systems, and the bridge between them:

**The On-Prem DevOps Platform (Part 2).** A reproducible, fully automated, entirely open-source platform that develops, packages, and delivers Linux-based scientific control-system software with **no cloud dependency**: infrastructure-as-code host provisioning (a golden image + Ansible), self-hosted GitLab with CI runners, multi-format packaging (native Debian `.deb`, Conda, Pip, plus Podman/OCI and Apptainer images) published to a self-hosted GPG-signed apt repository and conda channel, GitOps delivery (Argo CD reconciling a kubeadm Kubernetes cluster from Git), and an observability stack (Prometheus, Grafana, Alertmanager).

**The Tango Controls System (Part 3).** A distributed control system for a simulated 6–20 MeV electron-LINAC beamline, built on Tango Controls: PyTango device servers for magnet power supplies, a Ce:YAG beam-profile camera, a Faraday cup / beam-current transformer, a UHV vacuum station, and motion; Sardana for scan orchestration; Taurus operator GUIs; HDB++ archiving; alarm and interlock monitoring.

**The bridge (Part 4).** The Tango software is built, tested, packaged, and deployed **through** the DevOps platform, so a single code change flows automatically: *commit → CI tests and builds → signed artifacts → deployment → visible to operators*. That loop is the point of the whole handbook.

**A note on honesty and scope.** A software lab has no real magnets, cameras, or vacuum pumps. Every "device" in Part 3 is a **simulator** returning physically plausible values. This is not a fudge: real controls software is *always* developed against simulators first, because beam time is scarce and hardware is dangerous. Writing good simulators is itself part of the professional skill set.

### The build philosophy of this edition: manual first, then automate

This edition follows a deliberate two-pass philosophy that mirrors how a careful engineer actually learns infrastructure:

1. **First pass — by hand.** You build the golden image, clone it, and configure each VM's networking, hostname, and access **manually**, over the console and by editing files directly. No Vagrant, no cloud-init, no Ansible yet. This forces you to understand *what* is being configured and *why*, rather than hiding it behind a black box.
2. **Second pass — as code.** Once you understand every step by hand, you re-express it as automation: Ansible roles (Part 2) and cloud-init for image customization. Now the automation is transparent rather than magical, because you have already done by hand exactly what it does.

The same principle governs the whole book: *understand it manually, then automate it*. Part 1 is entirely the first pass for the lab's foundation.

### What Part 1 specifically covers

Because we assume no prior knowledge, Part 1 builds the ladder rung by rung:

1. **Chapter 1 — Linux from zero.** What an operating system, a kernel, and a distribution are; the shell; the filesystem; users and permissions; packages; processes and services; basic networking and SSH. With labs.
2. **Chapter 2 — Virtualization from zero (KVM/libvirt).** What a virtual machine is, what a hypervisor does, why labs are built on VMs, the KVM/libvirt stack, snapshots, and a hands-on first VM created with `virt-install`.
3. **Chapter 3 — Git from zero.** Why version control exists, commits, branches, merges, remotes, and the review workflow every later chapter relies on. With labs.
4. **Chapter 4 — Scientific control systems and the beamline problem.** What accelerators and beamlines are, what a control system does, EPICS vs Tango, and a worked example traced through the stack.
5. **Chapter 5 — What DevOps means for scientific software.** The five pillars — IaC, CI/CD, packaging, GitOps, observability — each explained with concrete examples.
6. **Chapter 6 — The architecture we are building.** The whole picture, the six-VM host topology, the `10.10.10.0/24` lab network, and the data flow you must be able to draw from memory.
7. **Chapter 7 — Preparing the lab foundation (LAB).** Installing KVM/libvirt on the host; building a golden image with `virt-install`; generalizing it with `virt-sysprep`; cloning six VMs with `virt-clone`; configuring static networking, hostnames, and a dedicated Ansible control node — the exact foundation Part 2 starts from.
8. **Glossary and self-assessment.**

Estimated effort for Part 1: **2–4 days** for a genuine newcomer working through every lab; a few hours if you only need the control-system and DevOps chapters.

### Hardware you need

One reasonably capable computer you are allowed to install Linux on directly (KVM/libvirt requires a Linux host):

| Resource | Minimum | Comfortable |
|---|---|---|
| RAM | 16 GB (reduced lab) | **32 GB** |
| CPU | 4 cores with virtualization extensions (VT-x / AMD-V) | 8+ cores |
| Disk | 200 GB free SSD | 500 GB+ SSD or roomy HDD |
| Host OS | A 64-bit Linux install (this edition shows Ubuntu 24.04 Desktop as the host) | Ubuntu LTS Desktop |

**Reference build.** This edition was written against a concrete machine, and the labs use its names so you have something real to map onto: a **Dell OptiPlex 9020** (quad-core i5-4570, 32 GB RAM, ~2.7 TB disk) running **Ubuntu Desktop 24.04 LTS**, acting purely as the KVM/libvirt hypervisor host — referred to throughout as **`labhost`**. Substitute your own machine's details freely; only the host's *role* matters, not its exact specs.

**Why a Linux host, not Windows.** KVM is part of the Linux kernel — it has no Windows equivalent. If your daily machine runs Windows, the clean route is to dedicate a spare machine to Linux, or dual-boot. Everything in this book runs on or inside Linux; there is no Windows path for the KVM/libvirt tooling.

---

# PART 1 — FOUNDATIONS FROM ZERO

*Read Part 1 once for the mental model and do its labs once for the muscle memory; you will refer back to it constantly.*

---

## Chapter 1 — Linux from zero: the operating system under everything

Everything in this handbook — the hypervisor, GitLab, Kubernetes, Tango, the archiver database — runs on Linux. Every scientific facility you might work at runs its control systems on Linux. So we start here, assuming you have never used it.

### THEORY 1.1 — What an operating system actually is

A computer's hardware — CPU, memory, disks, network cards — is useless raw. The **operating system (OS)** is the software layer that:

1. **manages hardware** — deciding which program gets the CPU next, which memory belongs to whom, how bytes get to the disk;
2. **provides abstractions** — programs see *files* instead of disk sectors, *processes* instead of CPU time slices, *sockets* instead of network packets;
3. **enforces boundaries** — one misbehaving program should not crash the others, and one user should not read another's files.

The core program that does all this is the **kernel**. "Linux", strictly speaking, is a kernel — started by Linus Torvalds in 1991, developed since by thousands of contributors, and open source, meaning anyone can read, modify, and redistribute its code. (This open-source kernel is also, as Chapter 2 explains, the thing that becomes a hypervisor when the KVM module is loaded — a detail that matters directly for this lab.)

A kernel alone is not usable. What you actually install is a **distribution** ("distro"): the Linux kernel plus a package manager, system services, libraries, and tooling, assembled and maintained by some organisation. Distributions cluster into families, and two families dominate scientific computing:

| Family | Members you'll meet | Package format | Package manager | Where you see it |
|---|---|---|---|---|
| **Red Hat family** | RHEL, Rocky Linux, AlmaLinux, Fedora, CentOS (historic) | `.rpm` | `dnf` / `yum` | Enterprise IT, many accelerator labs, certification world (RHCSA/RHCE) |
| **Debian family** | Debian, **Ubuntu**, Linux Mint | `.deb` | `apt` | Servers everywhere, developer desktops, **DESY beamline hosts**, this handbook's lab |

The families differ in packaging and administrative details but share the same kernel, the same shell, and the same concepts. Learn one deeply and the other is a week of adjustment. This handbook uses **Ubuntu** (Debian family) for both the host and the lab VMs because the target role centres on Debian packaging — but everything conceptual transfers to RHEL/Rocky.

**Callout — Why beamlines run Linux.** Three reasons recur at every facility: (1) *control* — source access means a lab can patch, audit, and trust the whole stack, decade after decade; (2) *automation* — everything on Linux is scriptable text, which is the precondition for the DevOps practices in Chapter 5; (3) *cost and longevity* — no per-host licences across hundreds of machines, and no vendor who can discontinue your OS. The scientific control frameworks (EPICS, Tango) are developed on and for Linux.

### THEORY 1.2 — The shell: talking to the machine in text

Windows and macOS train you to operate a computer by clicking. Linux servers have no screen attached at all; you operate them by **typing commands into a shell**. The shell is a program that reads a line of text, interprets it as a command with arguments, runs it, and shows you the output. The standard shell on Linux is **bash** (with **zsh** a common alternative); the window you type into is a **terminal**.

A command line has a simple grammar:

```
command  -short-options  --long-options  arguments
```

Example:

```bash
ls -l /home
```

- `ls` — the command ("list directory contents")
- `-l` — an option ("long format": show sizes, owners, dates)
- `/home` — the argument (which directory to list)

Three ideas make the shell powerful rather than merely quaint:

1. **Everything is text.** Commands read text and write text, so any command's output can feed any other's input.
2. **Pipes compose commands.** `command1 | command2` sends the output of the first into the second. `grep error /var/log/syslog | wc -l` means "find lines containing *error* in the system log, then count them". You build complex operations from small, single-purpose tools.
3. **Anything you can type, you can script.** A file full of commands is a program. That is the seed from which all automation in this book grows — Ansible, CI pipelines, and packaging are, at heart, disciplined descendants of shell scripting.

**Callout — Why professionals prefer the terminal.** A GUI action cannot be saved, reviewed, repeated on 200 hosts, or diffed against last month. A command can. When Chapter 5 defines Infrastructure as Code, it is really saying: *convert every click into text, so it can be versioned and replayed.* This is also why, in Chapter 7, you build VMs with the `virt-install` command rather than clicking through a wizard — the command is repeatable and documentable; the wizard clicks are not.

### THEORY 1.3 — The filesystem: one tree, everything is a file

Linux has no drive letters. There is a single directory tree rooted at `/`, and every disk, USB stick, or network share is **mounted** (grafted) somewhere onto that tree. The standard layout (the *Filesystem Hierarchy Standard*) gives every kind of file a conventional home:

| Directory | What lives there | You will touch it when… |
|---|---|---|
| `/home/<user>` | each user's personal files | always — your workspace and SSH keys live here |
| `/etc` | system-wide **configuration**, plain text files | configuring SSH, netplan networking, `/etc/hosts` |
| `/var` | **variable** data: logs (`/var/log`), databases, caches, VM disk images (`/var/lib/libvirt/images`) | reading logs, locating VM disks |
| `/usr` | installed programs and libraries | rarely directly — the package manager owns it |
| `/opt` | optional, self-contained third-party software | GitLab installs here in Part 2 |
| `/tmp` | temporary files, wiped on reboot | scratch space |
| `/root` | the administrator's home directory | rarely |
| `/proc`, `/sys` | *virtual* files exposing kernel state | checking CPU virtualization support in Chapter 7 |

Two path styles matter:

- **Absolute paths** start with `/` and mean the same thing regardless of where you are: `/etc/ssh/sshd_config`.
- **Relative paths** are interpreted from your **current working directory**: if you are in `/etc/ssh`, then `sshd_config` refers to the same file. `.` means "here", `..` means "one level up".

**Everything is a file** is a genuine Linux design principle: devices (`/dev/sda` is your first disk), kernel state (`/proc/cpuinfo` describes your CPU), even the terminal itself are presented as files you can read or write with the same tools you use on documents. It sounds odd; it makes the system radically scriptable.

### THEORY 1.4 — Users, root, and permissions

Linux was multi-user from birth, and its permission model is why it is trusted for shared, safety-adjacent infrastructure.

- Every process runs *as* some **user**; every file is *owned by* a user and a **group**.
- Each file carries three permission sets — for its **owner**, its **group**, and **everyone else** — each granting or denying **read (r)**, **write (w)**, and **execute (x)**.
- `ls -l` shows this as a string like `-rw-r--r--`: owner can read+write, group can read, others can read.

One user is special: **root** (user ID 0) may do anything — install software, edit any file, reboot. Working *as* root routinely is dangerous (a mistyped command has no safety net), so the professional pattern is:

- log in as a normal user;
- prefix individual administrative commands with **`sudo`** ("superuser do"), which runs just that one command as root after checking you're authorised.

You will type `sudo` hundreds of times in this book. Every time, it means: *this command modifies the system, not just my files.*

**Callout — Pitfall.** If a command fails with `Permission denied`, the answer is *sometimes* `sudo` — but first ask *why* it was denied. Blindly sudo-ing everything (or worse, `chmod 777`, granting everyone every permission) is how newcomers create insecure, unmaintainable systems. In a control-system context, permissions are part of the safety story. (One real example from Chapter 7: `netplan` refuses to apply a config file that is world-readable — the fix is a deliberate `chmod 600`, not disabling the check.)

### LAB 1.1 — First steps in the shell

You need a Linux machine for Chapter 1's labs. On the reference build you'll run these on `labhost` itself — but they work identically inside any of the VMs you build later, so if you prefer, do **Chapter 2 and 7 first** to create a VM, then run these inside it. The labs are written so either order works.

Open a terminal (on Ubuntu desktop: `Ctrl+Alt+T`). Type each command, press Enter, and read the output. Comments after `#` explain — you don't type them.

**Step 1 — Where am I, who am I?**

```bash
whoami          # prints your username
pwd             # Print Working Directory — where you are in the tree
ls              # list what's here
ls -l           # long form: permissions, owner, size, date
ls -la          # -a adds hidden files (names starting with .)
```

**Step 2 — Move around the tree.**

```bash
cd /            # go to the root of the tree
ls              # see the standard top-level directories from Theory 1.3
cd /var/log     # jump into the log directory (absolute path)
ls -lh          # -h = human-readable sizes (K, M, G)
cd ..           # up one level, now in /var
cd ~            # back home (~ = /home/<you>)
pwd             # confirm
```

**Step 3 — Create, inspect, move, delete.**

```bash
mkdir -p lab/playground        # make directories; -p creates parents as needed
cd lab/playground
echo "hello beamline" > note.txt    # '>' redirects output INTO a file (creates/overwrites)
cat note.txt                   # print a file's contents
echo "second line" >> note.txt # '>>' APPENDS instead of overwriting
cat note.txt
cp note.txt copy.txt           # copy
mv copy.txt renamed.txt        # move / rename
rm renamed.txt                 # delete (no recycle bin — gone is gone)
ls -l
```

**Step 4 — Look inside bigger things.**

```bash
man ls          # the manual page for ls — q to quit, arrows to scroll
less /etc/passwd    # page through a file; q quits. This file lists system users.
head -n 5 /etc/passwd    # first five lines
tail -n 5 /etc/passwd    # last five
grep root /etc/passwd    # print only lines containing "root"
```

**Step 5 — Compose with pipes.**

```bash
ls /etc | wc -l                 # how many entries are in /etc? (wc -l counts lines)
history | tail -n 20            # your last 20 commands
```

**Step 6 — Permissions and sudo.**

```bash
ls -l note.txt                  # note the -rw-rw-r-- style string and owner
chmod u+x note.txt              # add execute permission for the owner (u=user)
ls -l note.txt                  # see the x appear
sudo ls /root                   # /root is the administrator's home — denied without sudo
ls /root                        # try without sudo: Permission denied. That's the model working.
```

**Step 7 — Edit a file in the terminal.**

`nano` is the friendliest terminal editor; the lab uses it throughout (use `vim` if you already know it).

```bash
nano note.txt
# type something, then: Ctrl+O Enter to save, Ctrl+X to exit
cat note.txt
```

**Checkpoint 1.1:** without notes, you can (a) navigate to `/var/log` and back home, (b) create a directory and a file inside it, (c) explain what `sudo` does and why you don't log in as root, (d) use a pipe to count how many files are in a directory.

**Try it.** Run `cat /proc/cpuinfo | grep "model name" | head -1`. You just read *kernel state* through the file interface, filtered it, and truncated it — three tools composed. This exact file is how Chapter 7 checks whether your CPU can run virtual machines.

### THEORY 1.5 — Software installation: packages and package managers

On Windows you download an installer from a website and click through it. On Linux you almost never do that. Instead:

- Software is distributed as **packages** — archive files containing the program, plus **metadata**: name, version, and crucially a list of **dependencies** (other packages it needs).
- Your distribution runs **repositories** — curated, cryptographically **signed** online collections of packages.
- A **package manager** (`apt` on Debian/Ubuntu, `dnf` on RHEL/Rocky) downloads packages from repositories, verifies their signatures, resolves the full dependency graph, installs everything in the right order, and records what it did so software can be cleanly upgraded or removed.

This is worth a moment of appreciation, because *this handbook's target job is literally about this machinery*. When the DESY posting says "build and distribution systems for scientific software such as Debian, Conda, Pip, or Docker packages", it means: someone must *create* those packages, *sign* them, and *run* the repositories that beamline hosts install from. In Part 2 you will stand on both sides of the counter — first as a consumer (`apt install`), then as a producer (building `.deb` files and serving your own signed repository).

Key `apt` vocabulary you'll use constantly:

| Command | Meaning |
|---|---|
| `sudo apt update` | refresh the local *index* of what repositories offer (downloads no software) |
| `sudo apt install <pkg>` | install a package and its dependencies |
| `sudo apt remove <pkg>` | uninstall (keep its config files); `purge` removes config too |
| `apt search <word>` / `apt show <pkg>` | find and inspect packages |
| `apt list --installed` | what's installed |
| `sudo apt upgrade` | upgrade everything to the newest indexed versions |
| `sudo apt full-upgrade` | as above, but also allows removing packages when needed to complete upgrades |

**Callout — Why signatures matter.** When `apt` installs a package, it verifies a GPG cryptographic signature proving the package really came from the repository's owner and wasn't tampered with in transit. In Part 2 you will generate your own GPG key and sign your own repository — at which point this stops being trivia and becomes your job.

### LAB 1.2 — Drive the package manager

```bash
sudo apt update                     # refresh the index; read the output — it lists repositories
apt search cowsay                   # find a (silly) package
apt show cowsay                     # read its metadata: version, dependencies, description
sudo apt install -y cowsay          # -y answers "yes" to the confirmation prompt
cowsay "packages are just files plus metadata"
dpkg -L cowsay                      # dpkg is apt's low-level engine: -L lists every file the package installed
dpkg -s cowsay                      # status/metadata of an installed package
sudo apt remove -y cowsay           # clean removal — every file dpkg -L listed is gone
```

Now the tools this lab actually needs later:

```bash
sudo apt install -y git curl wget htop tree
git --version
```

**Checkpoint 1.2:** you can explain the difference between `apt update` and `apt upgrade`, and you can find out which files a package installed.

**Try it.** Run `less /etc/apt/sources.list` (and `ls /etc/apt/sources.list.d/`). These plain-text files are the complete definition of where your system gets software. In Part 2 you will add a line here pointing at *your own* repository.

### THEORY 1.6 — Processes, services, and systemd

A **process** is a running program. Every process has a numeric **PID**, an owning user, and a parent (the process that started it). `ps aux` lists all of them; `htop` shows them live.

Most processes you care about on a server are **services** (also called **daemons**): long-running background programs with no window — a web server, a database, the SSH server, the libvirt daemon that manages your VMs, and later the Tango device servers of Part 3. Something must start services at boot, restart them when they crash, and collect their logs. On every modern distribution that something is **systemd**, and you talk to it with two commands:

- **`systemctl`** — control services: start, stop, restart, enable (start at every boot), and query status.
- **`journalctl`** — read the logs systemd collected from them.

A service is defined by a **unit file** — a short text file saying what to run and how:

```ini
# /etc/systemd/system/example.service  (illustrative)
[Unit]
Description=An example long-running service
After=network.target          ; start only after networking is up

[Service]
ExecStart=/usr/local/bin/my-server --port 9000
Restart=on-failure            ; auto-restart if it crashes
User=svcuser                  ; run as a limited user, not root

[Install]
WantedBy=multi-user.target    ; belong to the normal boot sequence
```

Read that once carefully. In Chapter 7 you will *write* a unit file exactly like this — a small service that regenerates SSH host keys when they are missing, which is how your cloned VMs heal themselves on first boot. In Part 3 you will write unit files for Tango device servers, and in Part 2 your `.deb` packages will *ship* unit files so that installing a package automatically registers its service. Unit files are how software becomes *operable* rather than merely runnable.

### LAB 1.3 — Processes and services hands-on

```bash
ps aux | head -15               # a=all users, u=user-oriented, x=include daemons
ps aux | grep ssh               # find SSH-related processes
htop                            # live view; F6 sorts, F9 kills, q quits
```

Work with a real service — the SSH server:

```bash
sudo apt install -y openssh-server
systemctl status ssh            # is it running? since when? recent log lines at the bottom
sudo systemctl stop ssh         # stop it
systemctl status ssh            # now "inactive (dead)"
sudo systemctl start ssh        # start again
sudo systemctl enable ssh       # ensure it starts at every boot
journalctl -u ssh --since "10 minutes ago"   # logs for this unit only
journalctl -f                   # follow ALL system logs live (Ctrl+C to stop)
```

**Checkpoint 1.3:** you can check whether a service is running, stop/start/enable it, and pull up its recent logs — the three moves that make up half of all real-world troubleshooting.

**Callout — Why this matters for the job.** "Is the process up? What do its logs say? Does it start at boot?" is the diagnostic reflex for every layer of Parts 2–4: GitLab down, kubelet crashlooping, a device server not exporting, `libvirtd` not managing VMs. You just learned the universal version of it. In Chapter 7 you will use exactly `systemctl status ssh` to diagnose why a freshly cloned VM refuses SSH connections.

### THEORY 1.7 — Networking essentials: addresses, ports, names, SSH

The lab you build is a small *network* of machines, so five ideas must be solid:

**1. IP addresses.** Every machine on a network has a numeric address, e.g. `10.10.10.20`. Addresses starting `10.`, `172.16–31.`, and `192.168.` are **private** — valid only inside a local network, unreachable from the internet. The lab lives entirely in `10.10.10.0/24` (the `/24` means "the first three numbers identify the network, the last identifies the host — so hosts `10.10.10.1` through `.254`").

**2. Ports.** One machine runs many services, so an address alone isn't enough: each service listens on a numbered **port**. `10.10.10.10:80` means "the web server on that host". Conventions you'll meet: 22 = SSH, 80 = HTTP, 443 = HTTPS, 3306 = MySQL/MariaDB, 6443 = Kubernetes API, 10000 = Tango database server, 9090 = Prometheus.

**3. Names and DNS.** Humans use names (`gitlab.lab`), machines use addresses; **DNS** is the global system translating one to the other. A lab doesn't need real DNS: the file **`/etc/hosts`** provides local name→address mappings, one per line, and the lab uses it heavily:

```
10.10.10.10   svc.lab
10.10.10.30   tango.lab
```

**4. Clients and servers.** A **server** process waits, listening on a port; a **client** initiates a connection to `address:port`. Nearly everything in this book is client/server: your browser → GitLab; `kubectl` → Kubernetes API; a Taurus GUI → a Tango device server.

**5. SSH — the tool you'll use most.** The **Secure Shell** gives you an encrypted terminal on a remote machine: `ssh user@10.10.10.20` and you are *there*, running commands as if seated at it. SSH supports password login, but the professional standard is **key pairs**: you generate a mathematically linked pair of files — a **private key** (secret, stays on your machine, is never sent anywhere) and a **public key** (freely shareable). You place the public key on the server; SSH then proves you hold the private key without transmitting it. Keys are stronger than passwords and, crucially, **enable automation**: Ansible (Chapter 5, Part 2) is essentially "SSH into many machines and configure them", and it can only be non-interactive because keys remove the password prompt. In Chapter 7 you set up exactly this — a key on your control node that reaches all six VMs without a password.

**Host keys, a related idea you *will* meet.** Separately from *your* key pair, every SSH server has its own **host key** — an identity that lets clients confirm "this is the same machine I connected to last time." The first time you connect, SSH shows the host's fingerprint and asks you to accept it, storing it in `~/.ssh/known_hosts`. This matters in Chapter 7 for two reasons: cloned VMs must each generate their *own* unique host key (or they'd all claim the same identity — a security problem), and when a machine's IP changes, its old `known_hosts` entry becomes stale and SSH warns you (harmless, cleared with one command).

### LAB 1.4 — Networking and SSH hands-on

```bash
ip addr show            # your interfaces and IP addresses; 'lo' (127.0.0.1) is the machine talking to itself
ip route                # where packets go by default
ping -c 3 127.0.0.1     # 3 probe packets to yourself
cat /etc/hosts          # current local name mappings
ss -tlnp | head         # which ports are LISTENING right now (t=tcp l=listening n=numeric p=process)
```

Generate the SSH key pair pattern you will use for the lab (in Chapter 7 you'll do this specifically on the control node):

```bash
ssh-keygen -t ed25519 -C "lab key"
# Accept the default path (~/.ssh/id_ed25519). For a throwaway lab an empty passphrase is acceptable.
ls -l ~/.ssh
cat ~/.ssh/id_ed25519.pub    # the PUBLIC half — this is what gets copied to servers
```

SSH to your own machine to see the full loop (works because Lab 1.3 installed `openssh-server`):

```bash
ssh-copy-id localhost        # installs your public key into ~/.ssh/authorized_keys (asks password once)
ssh localhost                # now logs in with the key — no password
hostname && whoami           # prove where you are
exit
```

**Checkpoint 1.4:** you can state your machine's IP address, explain what `10.10.10.20:22` denotes, and SSH somewhere using a key instead of a password.

**Callout — Pitfall.** The private key file must be readable only by you (`chmod 600 ~/.ssh/id_ed25519` if SSH ever complains about permissions), and it never leaves your machine. If you ever paste a *private* key into a chat, a repo, or a server, consider it burned and generate a new pair. In Chapter 7 the private key lives on your control node and is, in a real sense, the "keys to the kingdom" for the whole lab — worth protecting accordingly.

### THEORY 1.8 — Chapter 1 in one page

- Linux = kernel + distribution; two families (Red Hat / Debian) differing mainly in packaging; the lab uses Ubuntu, the concepts transfer. The same Linux kernel becomes a *hypervisor* when KVM is loaded (Chapter 2).
- The shell turns administration into **text**, and text is what can be scripted, versioned, and automated — the precondition for everything in Chapter 5, and the reason we build VMs with commands, not wizards.
- One filesystem tree; configuration in `/etc`, logs in `/var/log`, VM disks in `/var/lib/libvirt/images`, your work in `~`.
- Normal user + `sudo` for administrative acts; permissions are a feature, not an obstacle.
- Software arrives as **signed packages** from **repositories** via a **package manager** — machinery you will consume in Part 1 and *build* in Part 2.
- Long-running software is a systemd **service**: `systemctl` controls it, `journalctl` shows its logs, a **unit file** defines it. You will write one in Chapter 7.
- Machines find each other by IP address and port; names come from DNS or `/etc/hosts`; **SSH with keys** is both your remote terminal and the substrate of automation. Host keys are a separate machine-identity concept that matters for cloned VMs.

---
## Chapter 2 — Virtualization from zero: KVM and libvirt

The lab needs six Linux machines. You own one. Virtualization resolves that contradiction, and it is also — through KVM, VMware, and cloud computing generally — a core professional topic in its own right. This edition builds the lab on **KVM/libvirt**, the Linux-native virtualization stack, and this chapter explains what that means from the ground up.

### THEORY 2.1 — What a virtual machine is

A **virtual machine (VM)** is a complete computer implemented in software: it has (virtual) CPUs, memory, a disk, and network cards, it boots its own operating system, and software running inside it cannot tell it isn't physical hardware. The physical machine that hosts VMs is the **host** (`labhost` in this book); each VM is a **guest**.

The program that makes this possible is the **hypervisor**. It carves out slices of the host's real CPU, RAM, and disk and presents them to each guest as if they were whole machines, while keeping guests isolated from the host and from each other. Modern CPUs contain hardware support for this (Intel **VT-x**, AMD **AMD-V**), which is why virtualization is fast enough to be ubiquitous — and why Chapter 7 makes you verify those extensions are enabled.

Hypervisors come in two types:

- **Type 1 (bare-metal)** — the hypervisor *is* the OS layer on the hardware; guests run on top. Examples: **VMware ESXi** (the enterprise standard, clustered under vCenter), Microsoft Hyper-V, Xen, and **KVM**. Data centres and facility server rooms run type 1.
- **Type 2 (hosted)** — the hypervisor is an application on a normal desktop OS. Examples: **VirtualBox**, VMware Workstation. Simpler to set up; common on personal laptops.

**Where does KVM fit?** This is the key thing to understand about this edition's choice. **KVM (Kernel-based Virtual Machine) is not a separate application — it is a module built into the Linux kernel itself.** When the `kvm_intel` (or `kvm_amd`) module is loaded, the running Linux kernel *becomes* a type-1-class hypervisor, using the CPU's virtualization extensions directly. There is no application layer sitting between your guest and the hardware the way VirtualBox or VMware Workstation interpose one. This is why KVM is lighter and faster than a type-2 hypervisor — and on a resource-constrained host, that efficiency is real, not cosmetic.

### THEORY 2.2 — The three pieces: KVM, QEMU, libvirt

KVM by itself only virtualizes CPU and memory. A usable virtualization stack on Linux is actually three cooperating pieces, and knowing all three prevents confusion later:

| Piece | What it does | Analogy for someone from VMware/VirtualBox |
|---|---|---|
| **KVM** | kernel module; turns Linux into a hypervisor (CPU + memory virtualization) | the core engine inside the hypervisor |
| **QEMU** | emulates the *devices* a VM needs: virtual disks, network cards, USB, a display, a serial port | the device-emulation half of VirtualBox/VMware |
| **libvirt** | the management layer: a daemon (`libvirtd`), a CLI (`virsh`), a GUI (`virt-manager`), plus management of storage pools and virtual networks | the VirtualBox/VMware application and its GUI + CLI |

You interact almost entirely with **libvirt**, via two tools:

- **`virsh`** — the command-line VM manager. `virsh start`, `virsh console`, `virsh list`, `virsh net-edit`, and dozens more. This is the KVM-world equivalent of VMware's `vmrun` or VirtualBox's `VBoxManage`.
- **`virt-manager`** — an optional graphical console, similar in spirit to the VMware Workstation window: see running VMs, open their consoles, watch resource graphs. You don't *need* it for this book (everything is driven from `virsh` and SSH), but it's a comforting visual fallback.

Two more libvirt concepts you'll meet in Chapter 7:

- A **storage pool** is a directory (or LVM group, or NFS share) where libvirt keeps VM disk images. The default is `/var/lib/libvirt/images`; this lab also uses a dedicated `/lab/vm-images` directory.
- A **virtual network** is a software-defined network connecting VMs. libvirt's default (`default`, on bridge `virbr0`, subnet `192.168.122.0/24`) provides NAT'd internet access. This lab adds a second, isolated network — `labnet` on `10.10.10.0/24` — for lab-internal traffic. VMs get two virtual NICs: one on `default` for internet (downloading packages), one on `labnet` for talking to each other.

Some vocabulary that recurs constantly:

| Term | Meaning |
|---|---|
| **Golden image / template** | a fully-prepared VM whose disk is cloned to stamp out new VMs in seconds instead of running an installer each time — this edition's replacement for Vagrant boxes |
| **Clone** | a new VM created by copying a golden image's disk (`virt-clone`) |
| **Snapshot** | a saved point-in-time state of a VM you can roll back to — the single greatest gift virtualization gives a learner |
| **Generalization (sysprep)** | stripping machine-specific identity (SSH host keys, machine-id, logs) from a golden image so its clones are each unique |
| **qcow2** | KVM's preferred disk-image format — thin-provisioned (grows only as used) and snapshot-capable, versus `raw` (a plain full-size disk file) |

### THEORY 2.3 — Why build the lab on VMs (and not containers, and not clouds)

Three alternatives were possible; the choice of VMs is deliberate:

1. **VMs vs several physical machines.** VMs cost nothing extra, are created and destroyed in minutes, can be snapshotted before risky steps, and make the entire lab reproducible from a golden image. Physical mini-PCs work too but add cost and cabling without adding learning.
2. **VMs vs containers.** Containers (Docker/Podman — properly introduced in Part 2) share the host's kernel and virtualise only the process environment. They are lighter than VMs but are *not* full machines: you cannot practise host provisioning, disk layout, systemd-at-boot, or kubeadm cluster building inside them realistically. The lab therefore uses VMs as the **substrate** (standing in for a facility's physical hosts) and containers as **payload** running on that substrate — exactly the layering a real facility has.
3. **VMs vs cloud instances.** Renting machines from AWS would work mechanically, but the target environment is a facility whose control networks are **isolated from the internet by design**. Building on-prem, on your own metal, with zero cloud dependencies, is not a compromise — it is the skill being demonstrated.

**Callout — Why KVM/libvirt specifically maps to the CV.** KVM/libvirt is what production Linux virtualization actually uses — it is the substrate beneath OpenStack and most enterprise Linux virtualization, and it is the closest open-source analogue to the VMware vSphere/ESXi clusters that facilities like DESY run. Building the lab on KVM/libvirt — and being able to compare it fluently to VMware, which many engineers already know — is directly interview-relevant in a way that a laptop-only VirtualBox setup is not. Being able to say "I built this on bare-metal Linux with KVM/libvirt, the way a facility host works" is a stronger, truer statement than "I ran VirtualBox on my laptop."

### THEORY 2.4 — The golden-image workflow (this edition's core method)

The original edition of this handbook used **Vagrant** — a tool that downloads a pre-built "box" and boots it from a text file. This edition uses the **golden-image-plus-clone** workflow instead, because it is how real facilities (and vSphere shops) actually operate, and because building it by hand teaches you what Vagrant would hide.

The workflow has four stages, each a chapter-7 lab:

1. **Build one golden image.** Install Ubuntu Server once, into a single VM (`golden-ubuntu2404`), making deliberate choices (no LVM, OpenSSH installed, serial console enabled) that make cloning clean. Prepare it: update, install baseline tools, install a self-healing SSH-host-key service.
2. **Generalize it** with `virt-sysprep` — strip its unique identity (SSH host keys, machine-id, logs, network identity) so its clones don't all claim to be the same machine.
3. **Snapshot it** — so you can always return to the clean, ready-to-clone state, or roll back to *modify* the image later.
4. **Clone it** with `virt-clone`, six times — each clone gets a fresh disk copy and a new MAC address in seconds, no installer run. Then assign each its identity (static IP, hostname) after the fact.

**Callout — Why this is better than Vagrant *for this book*.** Vagrant is excellent, but it is a black box: `vagrant up` conjures a machine and you never see the OS install, the disk layout, the identity generalization, or the network bring-up. The golden-image workflow makes every one of those steps explicit and yours — which is precisely the point of the "manual first, then automate" philosophy. It is also closer to the vSphere template/ISO-library workflow you'd meet at a facility, so the skill transfers directly. In the automation pass (Part 2 onward), Ansible and cloud-init take over the per-clone configuration — but you'll understand exactly what they're doing, because you did it by hand first.

### LAB 2.1 — Understand your host's virtualization support and the libvirt stack

Before creating any VM, confirm your host can virtualize and that the stack is present. (Chapter 7 does the full install; this lab is just orientation — run it on `labhost`.)

**Confirm CPU virtualization extensions — run on labhost:**
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo    # >0 means VT-x/AMD-V is present and enabled
```
If it prints `0`, reboot into your firmware (BIOS/UEFI) settings and enable *Intel VT-x* / *AMD-V*. On Dell business machines this is sometimes under a "Virtualization Support" submenu; VT-x and VT-d may be separate toggles (you need VT-x; VT-d is not required for this lab).

**See whether the libvirt tools exist yet — run on labhost:**
```bash
which virsh virt-install virt-clone 2>/dev/null || echo "not installed yet — Chapter 7 installs them"
```

**If libvirt is already installed, look at what exists — run on labhost:**
```bash
virsh list --all          # every defined VM and its state (none yet, on a fresh host)
virsh net-list --all      # virtual networks (you'll see 'default')
virsh pool-list --all     # storage pools
```

Read the output like a map: `virsh list` is your VM inventory, `virsh net-list` your virtual switches, `virsh pool-list` your disk-image storage. These three commands answer "what's here?" for the whole KVM/libvirt world — the equivalent of opening the VirtualBox/VMware main window.

**Checkpoint 2.1:** `egrep -c '(vmx|svm)' /proc/cpuinfo` returns a non-zero number (your CPU can virtualize), and you can name the three pieces of the stack (KVM = kernel hypervisor, QEMU = device emulation, libvirt = management) and the two tools you'll use (`virsh` CLI, `virt-manager` GUI).

### THEORY 2.5 — Snapshots: the learner's safety net

One virtualization feature deserves special emphasis because it changes how you learn: **snapshots**. A snapshot saves a VM's exact state at a moment in time; you can then do something risky, and if it goes wrong, roll back to the snapshot in seconds. This is what lets you experiment fearlessly.

In this lab you'll take snapshots at meaningful milestones — for example, `pre-sysprep` (the golden image fully prepared but not yet generalized, so you can return to *modify* it) and `golden-clean-v2` (generalized and ready to clone). With `virsh`:

```bash
virsh snapshot-create-as <vm> <name> "description"   # take one
virsh snapshot-list <vm>                              # list them
virsh snapshot-revert <vm> <name>                     # roll back
```

**Callout — Why snapshots and golden images are complementary.** The golden image is your *template* — the clean starting point you stamp clones from. Snapshots are your *undo button* — for the golden image itself (so you can modify and re-generalize it) and for any VM mid-experiment. Together they mean: you can always get back to a known-good state, and you can always produce a fresh machine. That combination is what makes a lab a safe place to make mistakes — and you *will* make mistakes; that's how the learning happens.

### THEORY 2.6 — From hand-made VMs to automation (the bridge to Chapter 5)

Creating and configuring six VMs by hand — installing the golden image, cloning, then setting each one's IP, hostname, and access — is genuine work, and Chapter 7 walks you through all of it. Doing it once by hand is the point: you will understand every layer. But a facility has hundreds of machines, and no one configures those by hand.

The second pass, from Part 2 onward, replaces the manual per-clone configuration with **Ansible** (which SSHes into each machine and applies declarative configuration) and, for image customization, **cloud-init** (which lets a VM configure itself on first boot from supplied data). The golden image + Ansible together are the KVM-world equivalent of "infrastructure as code" — and because you built it by hand first, the automation will be transparent rather than magical.

That is the first half of Infrastructure as Code (Chapter 5, Pillar 1). The mental progression of the whole book is: *do it once by hand to understand it → express it as code so it's repeatable → run the code against many machines.* Chapter 7 is the "by hand" step for the lab's foundation.

---
## Chapter 3 — Git from zero: version control as the foundation of everything

Every later chapter stores something in Git: application code, Ansible roles, CI pipeline definitions, Kubernetes manifests, even this handbook if you like. GitOps (Part 2) goes further and makes Git the authoritative definition of what is *running*. So Git cannot remain vague. (In this lab specifically, your Ansible roles, netplan templates, and lab scripts will all live in Git on the control node — so the version-control habit starts early.)

### THEORY 3.1 — The problem version control solves

Anyone who has worked on documents knows the failure mode: `report_final_v2_REALLY_final.docx`. Software has it worse — multiple people editing many files simultaneously, needing to know exactly what changed, when, by whom, and why, and needing the ability to reconstruct any past state (for example: *the version of the control software running the day the magnet quenched*).

A **version control system (VCS)** solves this by recording the project's history as a sequence of **commits** — snapshots of the entire project tree, each carrying an author, a timestamp, a message explaining *why*, and a pointer to its parent(s). **Git** (written by Linus Torvalds in 2005 for Linux kernel development) is the universal choice today. Two properties matter:

- Git is **distributed**: every copy ("clone") of a repository contains the *full* history and works offline; copies synchronise by exchanging commits.
- Git identifies every commit by a cryptographic **hash** of its contents and ancestry — histories cannot be silently rewritten without detection.

Vocabulary that must become native:

| Term | Meaning |
|---|---|
| **repository (repo)** | a directory whose full history Git tracks (stored in a hidden `.git/` folder inside it) |
| **commit** | one recorded snapshot + message; the atom of history |
| **staging area (index)** | the loading dock — you `git add` chosen changes there, then `git commit` records exactly that set |
| **branch** | a movable pointer to a line of development; `main` is the conventional trunk; work happens on short-lived feature branches |
| **merge** | combining one branch's commits into another |
| **conflict** | when two branches changed the same lines and Git asks a human to decide |
| **remote** | another copy of the repo you sync with (e.g. on a GitLab server); conventionally named `origin` |
| **clone / push / pull** | copy a remote repo down / send your commits up / fetch and integrate others' commits |

### THEORY 3.2 — The professional workflow: branches, merge requests, review

Solo Git is useful; Git's real power is the collaboration pattern built on it, which the DESY posting names explicitly ("modern software engineering practices, including code reviews, Git workflows and testing"):

1. Nobody commits straight to `main`. You branch: `git switch -c fix/magnet-ramp-rate`.
2. You commit your work on the branch — small, focused commits with messages that explain *why*.
3. You push the branch and open a **merge request** (GitLab's term; GitHub says *pull request*): a proposal to merge your branch into `main`, presented as a reviewable diff.
4. A colleague **reviews** — reading the diff, commenting, requesting changes. Simultaneously, **CI pipelines** (Part 2) run the tests automatically on your branch.
5. Only when review approves *and* tests pass does the merge happen. `main` therefore stays permanently releasable.

This pattern — *changes flow through reviewed, tested merge requests* — repeats at every layer of the platform you'll build: application code, infrastructure definitions, and (via GitOps) even the deployed state of the cluster. Learn it once here; recognise it everywhere later.

**Callout — Why commit messages matter.** A year from now, `git log` is the only honest history of *why* the system is the way it is. "fix stuff" helps nobody. The convention used in this book: a short imperative summary line, e.g. `magnet-ps: clamp ramp rate to hardware limit`, optionally followed by a blank line and a paragraph of reasoning.

### LAB 3.1 — Core Git, hands-on

On any machine with Git installed (`sudo apt install -y git`). On the reference build, a natural place is the control node (`admin.lab`), since that's where your lab-config repositories will live — but any machine works for practice.

**Step 1 — Identify yourself (goes into every commit you make):**

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

**Step 2 — Create a repository and your first commits:**

```bash
mkdir -p ~/lab/git-practice && cd ~/lab/git-practice
git init                        # the directory is now a repo (see the .git folder: ls -a)
echo "# Beamline notes" > README.md
git status                      # README.md is "untracked"
git add README.md               # stage it
git status                      # now "staged" — green
git commit -m "docs: add README"
git log --oneline               # one commit, with its short hash
```

**Step 3 — Make history and inspect it:**

```bash
echo "Magnet PS nominal current: 120 A" >> README.md
git diff                        # exactly what changed, line by line, before staging
git add README.md && git commit -m "docs: record magnet nominal current"
echo "Vacuum setpoint: 1e-8 mbar" >> README.md
git add -A && git commit -m "docs: record vacuum setpoint"
git log --oneline               # three commits — a real history
git show HEAD                   # full detail of the latest commit (HEAD = "where I am now")
git diff HEAD~2 HEAD            # everything that changed across the last two commits
```

**Step 4 — Branch, merge, and resolve a conflict (the rite of passage):**

```bash
git switch -c feature/add-camera         # create and switch to a branch
echo "Camera: Ce:YAG screen, 30 fps" >> README.md
git add -A && git commit -m "docs: add camera line"

git switch main                          # back to main — the camera line is absent here
echo "Camera: Basler acA1300, 60 fps" >> README.md    # a CONFLICTING edit on main
git add -A && git commit -m "docs: add camera line (different spec)"

git merge feature/add-camera             # Git reports: CONFLICT in README.md
cat README.md                            # see the <<<<<<< / ======= / >>>>>>> markers
nano README.md                           # edit: keep the line you want, DELETE all marker lines
git add README.md
git commit                               # concludes the merge (accept the default message)
git log --oneline --graph --all          # see the branch-and-merge shape of history
```

**Step 5 — Time travel:**

```bash
git log --oneline                        # pick the hash of your FIRST commit
git checkout <first-hash>                # the working directory becomes the past; look at README.md
cat README.md
git switch main                          # and back to the present
```

**Checkpoint 3.1:** you can create a repo, stage and commit, read a diff, branch and merge, resolve a conflict without panic, and check out an old commit. If any of those feels shaky, repeat the lab — Part 2 assumes all of them cold.

**Try it.** Delete the whole practice repo and reconstruct the same history from memory in under five minutes. Fluency, not familiarity, is the goal.

---

## Chapter 4 — Scientific control systems and the beamline problem

Now the domain. This chapter assumes you have never seen an accelerator and explains what the machines are, what a control system does, why there are two major frameworks, and — the punchline — why a control group needs the DevOps machinery of Chapter 5.

### THEORY 4.1 — Accelerators, synchrotrons, and beamlines in five minutes

A **particle accelerator** uses electric fields to push charged particles (electrons, in our context) to very high energies, and magnetic fields to steer and focus them. Two geometries matter here:

- A **LINAC** (LINear ACcelerator) accelerates particles down a straight line. The simulated machine in Part 3 is a **6–20 MeV electron LINAC** — "MeV" (mega-electron-volt) is an energy unit; 6–20 MeV is the class of machine used for irradiation, radiography, and as injectors for bigger rings.
- A **synchrotron** is a ring in which electrons circulate for hours. When high-energy electrons are bent by magnets, they emit extraordinarily intense, focused X-rays — **synchrotron radiation**. A **synchrotron light source** is a ring built to exploit that: the X-rays stream off tangentially into **beamlines** — long experimental stations where scientists put samples (proteins, batteries, catalysts, fossils, microchips) into the beam and measure what happens.

DESY's flagship machines make this concrete: **PETRA III** is a 2.3-km synchrotron light source ring feeding dozens of beamlines; **FLASH** is a free-electron laser (a LINAC-based light source producing ultrashort flashes); and **PETRA IV** is the planned upgrade of PETRA III into a fourth-generation, ultra-low-emittance source — informally, "the ultimate 4D X-ray microscope". Each beamline at such a facility is effectively its own small machine: optics, slits, sample stages, detectors, vacuum, cryogenics — all of which must be **controlled**.

Inventory of a single beamline's hardware, to make "heterogeneous" tangible:

| Subsystem | Example hardware | What must be controlled/read |
|---|---|---|
| Magnets & optics | dipole/quadrupole magnets, mirrors, monochromators | power-supply currents, positions, temperatures |
| Motion | stepper/servo motors on stages, slits, goniometers | positions, speeds, limits, homing |
| Diagnostics | Ce:YAG screens + cameras, Faraday cups, beam-current transformers | images, currents, waveforms |
| Vacuum | ion pumps, turbo pumps, gauges, gate valves | pressures, pump states, valve interlocks |
| Detectors & DAQ | area detectors producing GB/s | triggering, readout, data routing |
| Safety | interlock PLCs, shutters, radiation monitors | permit chains, alarm states |

Dozens of vendors, dozens of electrical protocols (serial lines, Ethernet, fieldbuses, PLCs), spread over hundreds of metres. That is the problem statement.

### THEORY 4.2 — What a control system actually is

A **control system** is the software and hardware layer that lets humans and programs observe and command a physical machine. Its job is to turn all the heterogeneous hardware above into a **uniform, networked interface**, so that an operator in a control room — or an automated scan, or a physicist's Python script — can say "set dipole magnet 3 to 120 amps", "read the beam current", or "move the sample stage to 10 mm" *without caring what brand of power supply or motor controller is underneath*.

Three properties make this hard, and they shape every design decision in this handbook:

1. **It is distributed.** No single computer talks to all the hardware. The control system is many cooperating processes on many hosts, communicating over a network. A failure in one subsystem must not take down the rest — the vacuum system must keep protecting the machine even if the camera server crashes.
2. **It is heterogeneous.** A magnet power supply, a camera, and a vacuum gauge speak completely different electrical and software protocols. The control system hides that behind a common abstraction, so that everything above the device layer is uniform.
3. **It is safety-critical and continuous.** Beam can damage equipment and, mishandled, people. Interlocks must be respected in software and hardware; the machine state must be observable at all times; and history must be **archived**, both for physics (correlating a bad measurement with a drifting magnet) and for compliance (proving what the machine was doing when).

If you have met **SCADA** systems (Supervisory Control And Data Acquisition, the industrial-automation world of Siemens PLCs and WinCC), the shape is familiar: field devices at the bottom, a supervisory layer above, HMIs for operators, historians for data. Scientific control frameworks solve the same problem, tuned for physics facilities: open source, deeply scriptable (Python everywhere), built for unusual custom hardware, and designed to archive everything at high rates.

### THEORY 4.3 — The two frameworks: EPICS and Tango

The scientific-facility world converged on two open-source control frameworks; you will hear both named constantly at DESY, which historically runs large amounts of both (plus its in-house DOOCS at the accelerators themselves):

**EPICS** (Experimental Physics and Industrial Control System) models the world as a flat, global namespace of **Process Variables (PVs)** — individually named values like `LINAC:MAG:DIP3:CURRENT` — served by processes called **IOCs** (Input/Output Controllers) and accessed over the **Channel Access** (or newer pvAccess) network protocol. Any client anywhere on the network can read, write, or subscribe to any PV by name. The ecosystem around it: **Phoebus** (operator display manager), the **Archiver Appliance** (history), the **BEAST** alarm server. EPICS was born at Los Alamos/Argonne, is dominant in the US accelerator community, and is extremely mature.

An EPICS interaction, concretely:

```bash
caget LINAC:MAG:DIP3:CURRENT         # read a PV → 119.98
caput LINAC:MAG:DIP3:CURRENT 120.0   # write it
camonitor LINAC:VAC:GAUGE1:PRESSURE  # subscribe: print every update as it happens
```

**Tango Controls** models the world as **devices** — named objects like `linac/mag/dip3` — each an instance of a device **class** (e.g. `MagnetPS`) running inside a **device server** process. A device exposes three kinds of member:

- **attributes** — readable (and often writable) values with metadata: `current`, `pressure`, `image`. Attributes carry units, limits, and alarm thresholds as first-class properties.
- **commands** — actions: `On()`, `Off()`, `Reset()`, `Ramp(target)`.
- **properties** — per-device configuration stored centrally (e.g. which serial port the real hardware is on).

All devices register in a central **Tango Database**, which acts as naming service and configuration store. Around the core: **PyTango** (the Python binding you'll write device servers in), **Sardana** (scan/experiment orchestration), **Taurus** (GUI toolkit), **HDB++** (archiving). Tango originated at the ESRF in Grenoble and is the backbone of most European light sources — ESRF, ALBA, SOLEIL, MAX IV, ELETTRA, SOLARIS, and DESY's photon-science beamlines.

A Tango interaction, concretely, from Python:

```python
import tango
dip3 = tango.DeviceProxy("linac/mag/dip3")   # look the device up via the Tango DB
print(dip3.current)                          # read an attribute → 119.98
dip3.current = 120.0                         # write it
dip3.On()                                    # invoke a command
print(dip3.state())                          # every device has a state machine: ON, MOVING, FAULT...
```

The comparison in one table:

| Aspect | EPICS | Tango |
|---|---|---|
| World model | flat namespace of **values** (PVs) | hierarchy of **objects** (devices) with attributes/commands |
| Unit of deployment | IOC process serving records | device server process hosting device instances |
| Naming/config | distributed (each IOC owns its records) | central Tango Database |
| Wire protocol | Channel Access / pvAccess | CORBA + ZeroMQ |
| Natural fit | very large numbers of simple signals | rich, stateful instruments with behaviour |
| Stronghold | US accelerator labs | European light sources |

They solve the same problem with different shapes: **EPICS gives you values; Tango gives you objects that own values.** Neither is "better"; large facilities often run both, bridged. This handbook builds Project 3 on **Tango** (matching the European light-source ecosystem) while noting that the DevOps platform of Part 2 is framework-agnostic — it would package and deliver EPICS IOCs just as happily, and the archiver database, alarm concepts, and GUI patterns have direct EPICS analogues (Archiver Appliance ↔ HDB++, BEAST ↔ Tango alarms, Phoebus ↔ Taurus).

### THEORY 4.4 — A worked example: "set dipole 3 to 120 A", traced end to end

To make the layers concrete, follow one operator action through the whole Tango stack. The operator drags a slider in a **Taurus GUI**:

1. **GUI → proxy.** The Taurus widget holds a `DeviceProxy` for `linac/mag/dip3`. Setting the slider calls `proxy.write_attribute("current", 120.0)`.
2. **Name resolution.** The first time that proxy was created, the Tango **database** (itself a device server backed by MariaDB/MySQL) was asked: *on which host and port does the device server exporting `linac/mag/dip3` live?* The answer is cached; subsequent traffic goes **directly** device-to-client — the database is a phone book, not a router, so it is not a bottleneck or a single point of failure for running traffic.
3. **Device server.** On host `tango.lab`, a Python process — the `MagnetPS` device server — receives the write. Your code (Part 3) runs: it validates the request against configured limits (write outside `[0, 200]` A → rejected with an exception), checks the device state (writes refused in `FAULT`), and begins **ramping** the (simulated) supply at a bounded rate rather than stepping instantly, because real magnets forbid current jumps.
4. **Hardware layer.** In a real facility the device server would now speak the vendor's protocol over Ethernet or serial. In the lab, a simulator object integrates a first-order response toward the setpoint with realistic noise. *Everything above this line is identical in lab and facility* — which is precisely why simulator-first development works.
5. **Feedback loop.** The GUI subscribes to the `current` attribute via Tango **events**; as the ramp proceeds, updates stream back and the display follows. The device's **state** shows `MOVING` during the ramp, `ON` at target.
6. **Archiving.** Independently, **HDB++** subscribes to the same events and writes timestamped values into its database — so next week anyone can plot exactly how the magnet ramped today.
7. **Alarms/interlocks.** An alarm rule watches related attributes (say, magnet temperature); if a threshold trips, the alarm system raises it, and interlock logic can force the device to a safe state regardless of what any GUI asks.

Every phrase in bold above becomes a lab in Part 3. If you can narrate this diagram from memory, you understand Tango's architecture:

```
Taurus GUI / Python script / Sardana scan
        │  (attribute read/write, commands — via DeviceProxy)
        ▼
   Tango Database  ──(name lookup only)──►  direct client↔server connection
        │
        ▼
 MagnetPS device server (PyTango)  ──►  hardware protocol │ or simulator
        │ events
        ├──────────►  HDB++ archiver ──► history database
        └──────────►  alarm system  ──► operator notification / interlock action
```

### THEORY 4.5 — Why a control group needs DevOps

Here is the bridge to Chapter 5, and the thesis of the entire handbook.

Historically, control-system software was installed by hand: an expert logged into each beamline host, compiled or copied the software, edited config files, and moved on. Consider what that means at facility scale:

- **PETRA IV scale.** Dozens of beamlines, each with multiple control hosts, plus servers for databases, archiving, and GUIs — hundreds of Linux machines that must run *consistent, known* versions of dozens of software components.
- **The archaeology problem.** A hand-configured host that dies cannot be rebuilt; its configuration lived in one expert's memory and undocumented edits. ("Configuration drift": every hand-touched host slowly becomes unique.)
- **The Friday-afternoon problem.** With no automated tests and no staged rollout, every software update is a leap of faith performed against a machine whose beam time costs thousands of euros per hour and is scheduled months ahead.
- **The collaboration problem.** Beamline software is developed by many people across groups and partner institutes. Without version control, review, and CI, integrating their work is chaos.

The modern answer is to treat control-system software like a serious software **product**:

- keep every line of code *and configuration* in **version control** (Chapter 3),
- build it through **automated pipelines** that run tests on every change,
- package it into **versioned, signed artifacts** (native `.deb`, Conda, containers),
- deploy those artifacts **declaratively** from Git,
- and **monitor** the running result.

That list is exactly the five pillars of Chapter 5, and it is why the two projects of this handbook belong together: Part 3's control system is the *payload*; Part 2's platform is the *delivery machine*; Part 4 fuses them.

**Callout — the manual-first philosophy is itself the lesson.** Notice that "an expert logged into each host and configured it by hand" is exactly what you do in Chapter 7 for six VMs — and it's fine *at six*, precisely because six is small enough to understand. The whole point of building the lab manually first, then automating, is to *feel* where the hand-configured approach stops scaling, so that when you reach for Ansible you understand viscerally what problem it solves. You are living the "archaeology problem" in miniature and then solving it — which is a far better teacher than being handed the automated solution first.

**Checkpoint 4 (conceptual):** in one sentence each, without notes: what is a beamline; what is a device in Tango; what is a PV in EPICS; why are control systems distributed; name two concrete reasons a control group benefits from CI/CD and packaging.

---
## Chapter 5 — What DevOps means for scientific software

"DevOps" is an overloaded word. Strip the marketing and it means one concrete thing: **treating operations — building, deploying, and running software — with the same rigour, tooling, and version control that we treat writing the software itself.** This chapter defines the five pillars this handbook builds, each with a concrete example. These are the pillars named, in different words, by the DESY posting; recognising them there is part of the point.

### THEORY 5.1 — Pillar 1: Infrastructure as Code (IaC)

**The idea:** a machine's entire configuration — which packages, which users, which services, which files, which network settings — is described in **text files kept in Git**, and a tool *applies* that description to produce the machine. You never hand-configure a server; you write (or edit) its description and run the tool. The machine becomes reproducible, reviewable, and disposable.

**Why it matters:** it kills the "archaeology problem" from Chapter 4.5. A host that dies is rebuilt by re-running the code. A change is a Git commit — reviewable, revertible, auditable. Every host built from the same code is identical; configuration drift disappears.

**What this handbook uses:** two complementary tools, and understanding how they divide the work is central to this edition.

- **The golden image** handles the *base machine*: a known Ubuntu install with baseline tooling, built once and cloned. This is the "provisioning" half — producing a running OS. In this edition you build it by hand with `virt-install` (Chapter 7), then later can express it as code with cloud-init.
- **Ansible** handles *configuration*: given a running machine (a clone), it SSHes in and declaratively enforces the desired state — install these packages, write these files, enable these services, in an **idempotent** way (running it twice changes nothing the second time). Ansible is agentless (it needs only SSH and Python on the target, both of which your golden image has), and describes desired state in YAML "playbooks" organised into reusable "roles".

**Callout — how this edition differs from the Vagrant original.** The earlier edition used Vagrant to *both* create the VM and trigger Ansible. This edition splits those cleanly: the golden-image/clone workflow creates the machine (Chapter 7, by hand), and Ansible configures it (Part 2). This split is actually the more honest picture of real infrastructure — provisioning and configuration management are genuinely different concerns, often handled by different tools (Terraform/vSphere-templates/cloud-init for provisioning; Ansible/Puppet for configuration). The DESY posting names Puppet and Ansible specifically as the configuration-management layer — exactly this pillar.

A tiny Ansible taste (you'll write real ones in Part 2), run from the control node `admin.lab` against all six VMs:

```yaml
# common.yml — a fragment of a real role
- name: Ensure baseline packages are present
  ansible.builtin.apt:
    name: [git, curl, htop, chrony]
    state: present
    update_cache: yes

- name: Ensure chrony (time sync) is running and enabled
  ansible.builtin.service:
    name: chrony
    state: started
    enabled: yes
```

Read it as a *description of desired state*, not a script of steps: "these packages should be present; this service should be running and enabled." Ansible figures out what to change to make it true.

### THEORY 5.2 — Pillar 2: CI/CD (Continuous Integration / Continuous Delivery)

**The idea:** every time someone pushes a commit, an automated **pipeline** runs — building the software, running its tests, linting it, and (if all pass) producing artifacts. **Continuous Integration** is the "test every change automatically" half; **Continuous Delivery** is the "automatically produce deployable artifacts from every good change" half.

**Why it matters:** it replaces the Friday-afternoon leap of faith with a gate. No change reaches an artifact — let alone a beamline host — without passing tests and review. Integration problems surface in minutes, on a branch, not months later on the machine.

**What this handbook uses:** self-hosted **GitLab** (Part 2) provides both the Git server and the CI engine. A `.gitlab-ci.yml` file in each repo defines the pipeline; **GitLab Runners** (on your `svc` node, and later in the Kubernetes cluster) execute the jobs. A pipeline for a Tango device server (Part 4) will: lint the Python, run the `pytest` suite against the device using Tango's test harness, build the `.deb` and conda packages, and — only on the `main` branch — sign and publish them.

```yaml
# .gitlab-ci.yml sketch
stages: [test, build, publish]

test:
  stage: test
  script:
    - pip install -e .[test]
    - pytest -q                      # device-server tests via DeviceTestContext

build-deb:
  stage: build
  script:
    - dpkg-buildpackage -us -uc      # produce the native .deb
  artifacts:
    paths: ["../*.deb"]

publish:
  stage: publish
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'   # only from main, only after test+build passed
  script:
    - ./scripts/publish-to-reprepro.sh    # sign + add to the apt repo
```

### THEORY 5.3 — Pillar 3: Packaging and distribution

**The idea:** software is delivered as **versioned, signed packages** pulled from **repositories**, never copied by hand. This is the pillar closest to the target job's title.

**Why it matters:** a beamline host should install `tango-magnet-ps` version `1.4.2` from a trusted repository with one command, get exactly the tested bytes, and be upgradeable and rollback-able. Hand-copied files have no version, no signature, no dependency resolution, and no upgrade path.

**What this handbook builds** (Part 2), matching the DESY posting's "Debian, Conda, Pip, or Docker packages":

- **Native Debian `.deb`** via `debhelper`/`dh-python` — the format a Debian/Ubuntu beamline host installs with `apt`. Your `.deb` ships the code *and* its systemd unit file, so installing it registers the service.
- **Conda** packages and a self-hosted channel — the scientific-Python community's cross-platform format.
- **Pip/wheel** — for pure-Python libraries.
- **Podman/OCI images** — containers for the Kubernetes-deployed pieces.
- **Apptainer** images — the HPC-friendly single-file container format used on compute clusters.

All published to a **self-hosted, GPG-signed apt repository** (via `reprepro`) and conda channel — so the signature you learned about in Chapter 1.5 is now *yours*, and beamline hosts trust your key.

### THEORY 5.4 — Pillar 4: GitOps

**The idea:** the desired state of what's *deployed* lives in a Git repository, and an automated agent continuously **reconciles** the running cluster to match Git. You don't run deploy commands; you commit a change to the desired-state repo, and the agent notices and converges the cluster.

**Why it matters:** Git becomes the single source of truth for what is running, with a full audit trail. Rollback is `git revert`. "What is deployed?" is answered by reading a repo, not by interrogating machines. Drift is impossible — the agent constantly corrects it.

**What this handbook uses:** **Argo CD** (Part 2) watches a Git repo of Kubernetes manifests and reconciles your **kubeadm** cluster (`cp`, `w1`, `w2` nodes) to match. You'll deploy the containerised parts of the system by committing manifests; Argo CD does the rest.

```
   Developer ──commit──►  Git (desired state)
                              │  Argo CD watches
                              ▼
                     reconciles ⇄ Kubernetes cluster (actual state)
```

### THEORY 5.5 — Pillar 5: Observability

**The idea:** a running system continuously emits **metrics** (numeric time series: CPU, request rates, queue depth, magnet current), **logs** (event records), and **traces**; you collect, store, visualise, and **alert** on them, so you know the system's health without logging into it.

**Why it matters:** control systems run continuously and must be watched continuously. "Is it healthy? What changed before it broke?" must be answerable from a dashboard at 3 a.m. Observability is also how you connect a physics anomaly to an infrastructure event.

**What this handbook uses** (Part 2): **Prometheus** scrapes metrics from every component (nodes, GitLab, Kubernetes, and — in Part 4 — the Tango system itself via an exporter); **Grafana** dashboards visualise them; **Alertmanager** routes alerts. You'll design dashboards around the **Golden Signals** (latency, traffic, errors, saturation) and add control-system-specific panels (device states, archiver lag).

### THEORY 5.6 — The five pillars as one system

The pillars aren't a menu — they interlock into a single flow, which is the whole handbook in one diagram:

```
IaC (golden image + Ansible)     ──► reproducible hosts to run everything on
        │
Version control (Git/GitLab)     ──► every change is a reviewed commit
        │
CI/CD (GitLab CI)                ──► every commit is tested + built automatically
        │
Packaging (.deb/conda/OCI, signed)──► good builds become trusted, versioned artifacts
        │
GitOps (Argo CD)                 ──► artifacts are deployed by committing desired state
        │
Observability (Prometheus/Grafana)──► the running result is watched and alerted on
```

A single change to a Tango device server (Part 4) will traverse this entire chain automatically: commit → CI tests and builds signed packages → GitOps deploys them → the change appears on an operator's screen → monitoring confirms health. That end-to-end loop, on hardware you built from a golden image, is the demonstrable competency this handbook exists to produce.

**Checkpoint 5 (conceptual):** name the five pillars and, for each, the specific tool this handbook uses and the one-sentence problem it solves. If you can do that from memory, you have the map for all of Part 2.

---
## Chapter 6 — The architecture we are building

You now have the concepts. This chapter assembles them into the concrete system you will build across Parts 2–4, and defines the lab topology — the machines, their roles, and the network — that Chapter 7 stands up. If you can draw this chapter from memory, you understand the destination.

### THEORY 6.1 — The two systems and the bridge, restated concretely

**System A — the DevOps platform (Part 2).** Everything needed to develop, test, package, and deliver software, self-hosted with no cloud:

- **GitLab** (on `svc`) — the Git server, container registry, and CI engine; the single source of truth for code.
- **GitLab Runners** — execute CI jobs (on `svc` initially, later in Kubernetes).
- **A Kubernetes cluster** (`cp` control-plane + `w1`, `w2` workers) built with **kubeadm** — where containerised workloads run.
- **Argo CD** (in the cluster) — GitOps reconciliation from a Git repo of manifests.
- **Packaging + a signed apt repository and conda channel** (on `svc`) — where built artifacts are published.
- **Observability** (Prometheus/Grafana/Alertmanager, on `svc`) — metrics, dashboards, alerts.

**System B — the Tango control system (Part 3), on `tango`:**

- The **Tango Database** + MariaDB backend — the naming/config service.
- Six simulated **device servers** — magnet PS, camera, Faraday cup, vacuum, motion, interlock.
- **Sardana** — scan/experiment orchestration.
- **Taurus** — operator GUIs.
- **HDB++** — archiving to a database, with a Grafana view.

**The bridge (Part 4).** The Tango software is developed, tested, packaged, and deployed *through* System A — closing the loop from commit to operator screen.

### THEORY 6.2 — The lab topology: one host, six VMs, a control node

This edition's topology differs from the original in two deliberate ways: the hypervisor is **KVM/libvirt** (not VirtualBox), and there are **six** VMs, not five — a dedicated **control node** (`admin`) is added to run Ansible and act as the administrative jump host, separating "the machine that configures everything" from "the machines being configured." That separation is good practice and mirrors how a real facility keeps a management host apart from the fleet.

```
                       Home network 192.168.100.0/24
                                    │
                        ┌───────────┴────────────┐
                        │  labhost  (Dell 9020)   │   Ubuntu Desktop 24.04
                        │  KVM / libvirt hypervisor│  192.168.100.22
                        └───────────┬─────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────────┐
         │            libvirt virtual networks                      │
         │  default (NAT, 192.168.122.0/24) → internet for VMs      │
         │  labnet  (isolated, 10.10.10.0/24, bridge virbr10)       │
         └──────────────────────────┬──────────────────────────────┘
                                    │  (each VM has TWO NICs: enp1s0→default/NAT, enp2s0→labnet)
   ┌──────────┬──────────┬──────────┼──────────┬──────────┬──────────┐
   ▼          ▼          ▼          ▼          ▼          ▼
 admin.lab  svc.lab    cp.lab     w1.lab     w2.lab    tango.lab
 10.10.10.5 10.10.10.10 10.10.10.20 10.10.10.21 10.10.10.22 10.10.10.30
 control    GitLab,    k8s        k8s        k8s        Tango DB,
 node:      registry,  control-   worker     worker     device servers,
 Ansible,   Prometheus, plane                           Sardana, Taurus,
 SSH jump   Grafana,   (kubeadm)                        HDB++
 host       apt repo
```

**The host-to-role table** — memorise this; every later chapter refers to these names:

| VM (hostname) | labnet IP | RAM (guide) | Role from Part 2/3 onward |
|---|---|---|---|
| **admin.lab** | 10.10.10.5 | 2 GB | Ansible control node; SSH jump host; administration; holds the lab's SSH private key |
| **svc.lab** | 10.10.10.10 | 6 GB | GitLab, container registry, Prometheus/Grafana/Alertmanager, signed apt repo + conda channel |
| **cp.lab** | 10.10.10.20 | 4 GB | Kubernetes control-plane (kubeadm) |
| **w1.lab** | 10.10.10.21 | 4 GB | Kubernetes worker |
| **w2.lab** | 10.10.10.22 | 4 GB | Kubernetes worker |
| **tango.lab** | 10.10.10.30 | 6 GB | Tango DB + MariaDB, device servers, Sardana, Taurus, HDB++ |

**Callout — Why `10.10.10.0/24` and not `192.168.x`.** The lab network was deliberately chosen to be *visually distinct* from the home network (`192.168.100.x`) and libvirt's default NAT network (`192.168.122.x`). At a glance you can tell which network an address belongs to: `10.10.10.*` is unambiguously "the lab." This is a small clarity win that pays off constantly when reading logs and `/etc/hosts` across seven machines. (The original edition used `192.168.56.x`; if you follow an older draft you may see that — the scheme is interchangeable, only the numbers differ.)

**Callout — RAM budgeting on a 32 GB host.** The full six-VM fleet sums to ~26 GB of guest RAM; with the host's own needs that is tight but workable on 32 GB, especially since you rarely run *all* six under load simultaneously. On a 16 GB host, run the **reduced lab**: drop `w2` and trim `svc`, which most chapters tolerate. The control node (`admin`) is deliberately small (2 GB) — it only runs Ansible and SSH.

### THEORY 6.3 — The two-NIC design, and why it matters

Every VM has **two** virtual network interfaces, and understanding why is worth a paragraph because it shapes all the networking in Chapter 7:

- **`enp1s0` → the `default` NAT network (`192.168.122.x`, DHCP).** This gives each VM outbound internet access — needed to `apt install` packages, pull container images, and download during the build. Addresses here are dynamic and don't matter; you never reference them long-term.
- **`enp2s0` → the `labnet` isolated network (`10.10.10.x`, static).** This is the lab's own private network, where the VMs talk to *each other* by stable, memorable addresses. It has no route to the internet by design — it is the isolated "control network," mirroring how a facility keeps its control LAN separate.

So each machine's *identity* lives on `labnet` (static `10.10.10.x`, in `/etc/hosts`), while its *internet access* rides the throwaway NAT interface. This is a faithful small-scale model of a real control network: a stable, isolated internal fabric, with separately-managed outbound access for maintenance.

### THEORY 6.4 — The data flow you must be able to draw

The single most important diagram in the handbook — the end-to-end delivery loop that Part 4 demonstrates:

```
   Developer edits a Tango device server, commits, pushes a branch
        │
        ▼
   GitLab (svc.lab) — merge request, code review
        │  triggers
        ▼
   GitLab CI pipeline (runner)
        │  lint → pytest (DeviceTestContext) → build .deb + conda + OCI image
        │  on main: sign + publish to apt repo/conda channel (svc.lab) + push image to registry
        ▼
   Argo CD (Kubernetes: cp/w1/w2) — sees updated manifest in Git, reconciles
        │  OR: tango.lab apt-installs the new signed .deb
        ▼
   New device-server version runs on tango.lab
        │
        ▼
   Taurus operator GUI shows the change; HDB++ archives; Prometheus/Grafana confirm health
```

Trace one change along that path and you have traced the entire handbook. Everything in Parts 2–4 is a matter of building each box and each arrow — on the six-VM, `10.10.10.0/24` foundation you are about to stand up in Chapter 7.

**Checkpoint 6 (conceptual):** from memory, draw the six-VM topology with each machine's name, `10.10.10.x` address, and role; explain the two-NIC design in one sentence; and trace a single commit from a developer's keyboard to an operator's screen naming the systems it passes through.

---
## Chapter 7 — Preparing the lab foundation (LAB)

This is the chapter where you build. By its end you will have a KVM/libvirt host, a reusable golden image, and six configured VMs — the exact foundation Part 2 begins from. Everything is done **manually**, by hand, on purpose: this is the "first pass" of the manual-first philosophy. In a later automation pass you will re-express much of it as cloud-init and Ansible, but you will understand it because you did it by hand first.

The chapter is long because it is the real work. It is organised as the golden-image workflow from Chapter 2.4: **install the host stack → build the golden image → prepare and generalize it → clone → configure each clone → establish the control node.** Every command names the machine it runs on. Do not skip checkpoints.

### THEORY 7.1 — What we are about to do, and the order it must happen in

The build has a strict dependency order, and knowing it prevents locking yourself out:

1. **Install KVM/libvirt on `labhost`** and create the `labnet` network.
2. **Build one golden image** (`golden-ubuntu2404`) by installing Ubuntu Server once — with three deliberate choices (no LVM, OpenSSH installed, serial console) that make everything downstream clean.
3. **Prepare the golden image**: update it, install baseline tools, and install a self-healing SSH-host-key service so clones repair themselves on first boot.
4. **Snapshot, then generalize** it with `virt-sysprep` (strip its identity), then snapshot again.
5. **Clone six VMs** from it with `virt-clone`.
6. **Configure each clone**: static `10.10.10.x` IP, hostname, `/etc/hosts`, unique machine-id — done over the **serial console**, because that never depends on the network we are changing.
7. **Establish `admin` as the control node**: generate an SSH key, distribute it to all six, verify passwordless reach.

**Callout — the golden rule of network changes.** Whenever you change a machine's IP, do it over a channel that does *not* depend on that IP — the **serial console** (`virsh console`). If you change a VM's network over an SSH session that rides that same network, you cut your own connection mid-change. The serial console is immune to this, which is exactly why Chapter 2 made a point of enabling it in the golden image.

### LAB 7.1 — Install KVM/libvirt on the host and verify virtualization

**Run on labhost.**

**Step 1 — Confirm the CPU can virtualize:**
```
egrep -c '(vmx|svm)' /proc/cpuinfo
```
A number greater than 0 is required. If it's 0, enable Intel VT-x / AMD-V in the BIOS/UEFI first.

**Step 2 — Install the stack:**
```
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager libguestfs-tools
```
- `qemu-kvm` — QEMU + the KVM integration (the engine)
- `libvirt-daemon-system`, `libvirt-clients` — the `libvirtd` daemon and `virsh`
- `virtinst` — `virt-install` and `virt-clone`
- `virt-manager` — the optional GUI
- `libguestfs-tools` — `virt-sysprep`, `virt-customize`, `virt-filesystems` (offline image tools)

**Step 3 — Add yourself to the libvirt/kvm groups** (so you can manage VMs without `sudo` every time):
```
sudo usermod -aG libvirt,kvm $USER
```
Log out and back in (or reboot) for the group change to take effect.

**Step 4 — Verify libvirt is running and the default network exists:**
```
systemctl status libvirtd --no-pager
virsh net-list --all
```
You should see `libvirtd` active and a `default` network (it may be inactive; the next step activates it).

**Step 5 — Ensure the default NAT network is active and autostarts:**
```
virsh net-start default 2>/dev/null; virsh net-autostart default
virsh net-list --all
```

**Checkpoint 7.1:** `egrep -c '(vmx|svm)' /proc/cpuinfo` > 0; `virsh list --all` runs without a permission error; the `default` network is active.

### LAB 7.2 — Create the isolated lab network (labnet, 10.10.10.0/24)

The lab needs its own isolated network. We define it as XML and load it. **Run on labhost.**

**Step 1 — Write the network definition:**
```
cat > /tmp/labnet.xml <<'EOF'
<network>
  <name>labnet</name>
  <bridge name='virbr10' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
EOF
```
Note there is **no `<forward>` element** — that makes `labnet` an *isolated* network with no route to the internet, exactly the "control LAN" model of Chapter 6.3. The gateway is `10.10.10.1`; the DHCP range (`.100–.200`) sits above the static addresses (`.5–.30`) your VMs will use, so they never collide.

**Step 2 — Define, start, and autostart it:**
```
virsh net-define /tmp/labnet.xml
virsh net-start labnet
virsh net-autostart labnet
virsh net-list --all
```

**Checkpoint 7.2:** `virsh net-list --all` shows both `default` and `labnet` active and set to autostart; `ip addr show virbr10` shows `10.10.10.1`.

### LAB 7.3 — Prepare the storage directory and fetch the ISO

**Run on labhost.**

**Step 1 — A tidy home for lab artifacts:**
```
sudo mkdir -p /lab/{iso,vm-images,backups,ansible,git,scripts,docs,snapshots}
sudo chown -R $USER:$USER /lab
```

**Step 2 — Get the Ubuntu Server ISO** (24.04.x live-server). If you already have it, place it in `/lab/iso/`. Otherwise download it there, and verify its checksum:
```
cd /lab/iso
# (download ubuntu-24.04.x-live-server-amd64.iso here, plus SHA256SUMS)
sha256sum -c SHA256SUMS 2>/dev/null | grep live-server
```
Look for `OK`. A verified ISO now saves you from chasing phantom install bugs later.

**Checkpoint 7.3:** the ISO is in `/lab/iso/` and its checksum verifies.

### LAB 7.4 — Build the golden image with virt-install (serial-console install)

This is the crucial step, and the flags matter. **Run on labhost.** Substitute your actual ISO filename.

```
virt-install \
  --name golden-ubuntu2404 \
  --memory 4096 \
  --vcpus 2 \
  --disk path=/lab/vm-images/golden-ubuntu2404.qcow2,size=30,format=qcow2,bus=virtio \
  --network network=default,model=virtio \
  --network network=labnet,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --os-variant ubuntu24.04 \
  --location /lab/iso/ubuntu-24.04.4-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
  --extra-args 'console=ttyS0,115200n8 ---'
```

Flag by flag, because this is the "explain every option" chapter:

- `--memory 4096 --vcpus 2` — the golden image's resources; clones inherit these and can be adjusted per-clone later.
- `--disk ...size=30...bus=virtio` — a 30 GB **qcow2** disk on the paravirtualized **virtio** bus. *virtio vs emulated SATA/IDE:* virtio is near-native speed and always the right choice for Linux guests; emulated buses exist only for OSes without virtio drivers. *qcow2 vs raw:* qcow2 is thin-provisioned (grows as used) and snapshot-capable; raw is a plain full-size file, marginally faster but larger and less flexible.
- `--network network=default` and `network=labnet` — the two NICs of Chapter 6.3, both virtio.
- `--graphics none --console pty,target_type=serial` plus `--location ... --extra-args 'console=ttyS0...'` — **this is the key combination.** It performs the install *over the serial console* rather than a graphical window, which guarantees the serial console works from the very first boot. The `---` in `--extra-args` separates installer kernel arguments from arguments passed through to the *installed* system, so `console=ttyS0` persists into the final OS too.

**Callout — Pitfall (learned the hard way): use `--location`, not `--cdrom`.** A `--cdrom` install does *not* route the installer's output to the serial console by default — you get a running domain but a blank console, an install proceeding invisibly. `--location` with explicit `kernel=casper/vmlinuz,initrd=casper/initrd` boots the installer with your serial-console kernel arguments, so you actually see and drive it. If `virt-install` warns *"CDROM media does not print to the text console,"* you used the wrong option.

Running the command drops you straight into the Ubuntu Server text installer, in your terminal. Work through it, and make **three deliberate choices**:

1. **Network:** accept DHCP on both interfaces for now. Do *not* set static addresses here — the golden image must stay generic; identity is assigned after cloning.
2. **Storage — the important one:** choose "Use an entire disk," then find **"Set up this disk as an LVM group" and UNCHECK it.** A plain-partition layout clones cleanly; the default LVM layout causes offline-tooling grief (see the callout below). Confirm the summary shows a plain `/` partition, not an `ubuntu-vg`.
3. **SSH:** **check "Install OpenSSH server."** This makes SSH work from first boot.

Set the username to `falak` (or your choice), hostname `golden-ubuntu2404`, skip all featured snaps, and let it install. When it says "Reboot Now," accept — then boot into the installed system.

**Callout — Pitfall (learned the hard way): choose NO LVM.** The Ubuntu installer defaults to an LVM disk layout. LVM is excellent in production, but for *cloned* VMs it creates duplicate volume-group UUIDs across clones, which breaks the offline image tools (`virt-customize`, `virt-sysprep`) with confusing "cannot mount root filesystem" / "not a directory" errors. A plain-partition install sidesteps the entire class of problem, mounts offline trivially, and loses you nothing this lab needs. If you ever inherit an LVM golden image and see those mount errors, this is why — the fix is to rebuild non-LVM, which is faster than fighting it.

**Checkpoint 7.4:** you can log into the installed golden image over the serial console; `lsblk` shows root `/` on a plain partition (e.g. `vda2`), **not** an `ubuntu-vg`; `systemctl is-active ssh` reports the SSH service (enable it with `sudo systemctl enable --now ssh` if it's installed but inactive).

### LAB 7.5 — Prepare the golden image: tools and a self-healing SSH-key service

The golden image boots and accepts SSH. Now prepare it so its clones are healthy from birth. You can work over the serial console, or (easier) SSH in via its NAT address — find it from `labhost` with `virsh domifaddr golden-ubuntu2404 --source agent` once the guest agent is installed, or `--source arp` before then. Everything in this lab runs **on golden-ubuntu2404**.

**Step 1 — Update and install baseline tools:**
```
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent git curl wget vim tree dnsutils jq htop bash-completion net-tools
```
`qemu-guest-agent` lets `labhost` query the VM's IP and coordinate clean shutdowns — install it now so every clone inherits it.

**Step 2 — Enable the guest agent:**
```
sudo systemctl enable --now qemu-guest-agent
systemctl is-active qemu-guest-agent
```
You will see a "static unit, not meant to be enabled" notice — that is harmless; `--now` starts it, which is what matters. Confirm it says `active`.

**Step 3 — Install the self-healing SSH-host-key service.** This is the fix that makes generalization safe. Create the unit:
```
sudo tee /etc/systemd/system/regenerate-ssh-host-keys.service > /dev/null <<'EOF'
[Unit]
Description=Regenerate SSH host keys if missing
Before=ssh.service
ConditionPathExistsGlob=!/etc/ssh/ssh_host_*_key

[Service]
Type=oneshot
ExecStart=/usr/bin/ssh-keygen -A
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable regenerate-ssh-host-keys.service
```
Read the unit against Chapter 1.6: the clever part is `ConditionPathExistsGlob=!/etc/ssh/ssh_host_*_key` — the `!` means "run only if these files do **not** exist." On a normal boot it does nothing; on a clone's *first* boot after generalization wiped the host keys, it runs `ssh-keygen -A` just before sshd starts, regenerating unique keys. Self-healing, zero maintenance.

**Callout — Why this service exists (learned the hard way).** Generalization (`virt-sysprep`, next lab) deletes the SSH host keys — correctly, so every clone gets its *own* unique identity rather than all sharing one. But `sshd` refuses to start with no host keys, so without this service a freshly cloned VM answers SSH with "connection refused." The naive fixes (regenerate by hand on each clone, or over a console that may not exist yet) are painful. This one small unit, baked into the golden image, makes every clone repair itself on first boot — the difference between a smooth clone workflow and hours of per-VM firefighting.

**Step 4 — Ensure SSH and the serial console are permanently enabled:**
```
sudo systemctl enable ssh
sudo systemctl enable serial-getty@ttyS0.service
```
The `serial-getty@ttyS0` service is what presents a *login prompt* on the serial console. Enabling it permanently means `virsh console` gives you a usable login on every clone — your guaranteed way in if SSH ever misbehaves.

**Step 5 — Clean up and shut down:**
```
sudo apt clean && sudo apt autoremove -y
sudo shutdown now
```

**Checkpoint 7.5:** the guest agent is `active`; `regenerate-ssh-host-keys.service` and `serial-getty@ttyS0.service` are both enabled; the golden image shuts down cleanly.

### LAB 7.6 — Snapshot, generalize, snapshot again

**Run on labhost.** The golden image is now shut off.

**Step 1 — Take a pre-generalization snapshot** (so you can return here to *modify* the image later, then re-generalize):
```
virsh snapshot-create-as golden-ubuntu2404 pre-sysprep "Golden image fully prepped, BEFORE generalization - rollback point"
```

**Step 2 — Generalize with virt-sysprep:**
```
sudo virt-sysprep -d golden-ubuntu2404 --operations defaults,-lvm-uuids
```
`virt-sysprep` strips machine-specific identity: it deletes SSH host keys (the self-healing service will regenerate them), resets `/etc/machine-id`, clears logs, bash history, the DHCP lease, and network identity — so each clone is genuinely distinct. `--operations defaults,-lvm-uuids` runs all the default operations *except* `lvm-uuids` (belt-and-suspenders, since your non-LVM image has no LVM UUIDs to change anyway).

**Step 3 — Snapshot the clean, generalized state:**
```
virsh snapshot-create-as golden-ubuntu2404 golden-clean-v2 "Non-LVM, self-healing SSH keys, serial console, sysprepped - ready to clone"
virsh snapshot-list golden-ubuntu2404
```

You now have two meaningful snapshots: `pre-sysprep` (return here to change the image, then re-run sysprep) and `golden-clean-v2` (the ready-to-clone state). This is the proper golden-image maintenance workflow.

**Callout — Pitfall: offline tools and LVM.** If `virt-sysprep` (or `virt-customize`) fails with errors about a `dm-uuid-LVM-...` path or "you must call 'mount' first," the image is LVM-based and the offline tooling can't cleanly activate its logical volume. On a non-LVM image (which yours is, per Lab 7.4) this class of error simply doesn't occur — which is the entire reason we chose no-LVM. The clean, fast fix for an LVM image is to rebuild it non-LVM.

**Checkpoint 7.6:** `virsh snapshot-list golden-ubuntu2404` shows both `pre-sysprep` and `golden-clean-v2`; `virt-sysprep` completed without errors.

### LAB 7.7 — Clone the six VMs

**Run on labhost.** One command produces all six:
```
for vm in admin.lab svc.lab cp.lab w1.lab w2.lab tango.lab; do
  virt-clone --original golden-ubuntu2404 --name "$vm" --file "/lab/vm-images/${vm}.qcow2"
done
```
`virt-clone` copies the golden disk, assigns each clone a **new MAC address**, and writes a fresh domain definition — in seconds per VM, no installer. With ~2.7 TB of disk you can afford full independent clones (the default); each is standalone and doesn't depend on the golden disk surviving.

**Verify:**
```
virsh list --all
ls -lh /lab/vm-images/
```

**The payoff test — prove a clone boots SSH-ready with no intervention:**
```
virsh start admin.lab
```
Wait ~45 seconds (first boot regenerates SSH host keys via your self-healing service), then:
```
virsh domifaddr admin.lab --source agent
```
SSH in using the `192.168.122.x` (NAT) address it reports:
```
ssh falak@192.168.122.XXX
```
It should connect, prompting you to accept a **freshly generated, unique host key** — proof the self-healing service worked. No console fixing, no manual keygen.

**Checkpoint 7.7:** all six clones exist; `admin.lab` accepts SSH on first boot with a unique host key, with no manual repair.

**Callout — Why the payoff test matters.** This single successful SSH — connecting to a clone that has never been touched since cloning — validates the entire golden-image design: non-LVM (so sysprep and cloning were clean), self-healing SSH keys (so sshd started), guest agent (so `domifaddr` found the IP). If this works on `admin`, all five other clones will behave identically. It is the moment the "build it right once" investment pays back.

### LAB 7.8 — Configure each clone: static IP, hostname, /etc/hosts

Each clone is an identical copy; now give each its identity on `labnet`. We do this over the **serial console** (`virsh console`), because we are changing the very network SSH would ride — the golden rule from Theory 7.1. The procedure is identical per VM; only the IP and hostname change.

**The per-VM values:**

| VM | labnet static IP | hostname |
|---|---|---|
| admin.lab | 10.10.10.5 | admin.lab |
| svc.lab | 10.10.10.10 | svc.lab |
| cp.lab | 10.10.10.20 | cp.lab |
| w1.lab | 10.10.10.21 | w1.lab |
| w2.lab | 10.10.10.22 | w2.lab |
| tango.lab | 10.10.10.30 | tango.lab |

**The procedure, shown for `admin.lab`** (repeat for each, changing the two values):

**Run on labhost** — start the VM and attach to its console:
```
virsh start admin.lab
```
Wait ~30 seconds, then:
```
virsh console admin.lab
```
Press Enter to get a login prompt (it will still show the golden hostname — that's expected), and log in.

**Run on admin.lab** (over the console) — the whole configuration block:
```
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg > /dev/null <<'EOF'
network: {config: disabled}
EOF

sudo tee /etc/netplan/99-labnet-static.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: no
      addresses:
        - 10.10.10.5/24
EOF

sudo chmod 600 /etc/netplan/99-labnet-static.yaml
sudo netplan generate && sudo netplan apply
sudo hostnamectl set-hostname admin.lab
sudo sed -i 's/127.0.1.1.*golden-ubuntu2404/127.0.1.1\tadmin.lab/' /etc/hosts

sudo tee -a /etc/hosts > /dev/null <<'EOF'
10.10.10.5   admin.lab admin
10.10.10.10  svc.lab svc
10.10.10.20  cp.lab cp
10.10.10.21  w1.lab w1
10.10.10.22  w2.lab w2
10.10.10.30  tango.lab tango
EOF

ip -4 addr show enp2s0
```

What each part does:
- The **cloud-init disable** file stops cloud-init from regenerating netplan on future boots and fighting your static config. (Ubuntu Server ships cloud-init, which manages networking by default; you are taking that over.)
- The **netplan file** sets `enp2s0` (the labnet NIC) to the static `10.10.10.x` address. `enp1s0` (NAT) is left on DHCP for internet — you don't touch it.
- **`chmod 600`** is mandatory: netplan refuses to apply a world-readable config (the Chapter 1.4 permissions-are-a-feature callout, made real).
- **`hostnamectl set-hostname`** sets the persistent hostname; the `sed` line tidies the leftover `127.0.1.1` entry.
- The **`/etc/hosts` block** is *identical on every VM* — every machine should know all six by name, since there is no DNS server.

Confirm `enp2s0` now shows `10.10.10.5/24`, then detach from the console with **`Ctrl+]`** (Ctrl and the right square bracket).

**Verify from labhost** that the static address is live:
```
ssh falak@10.10.10.5
```
(This connects over `labnet` via the new static IP. It prompts for a password — `labhost` uses passwords; only the control node will use keys.)

**Now repeat** the whole procedure for the other five, changing only the netplan `addresses:` line and the `hostnamectl`/`sed` name each time (`svc.lab`→`.10`, `cp.lab`→`.20`, `w1.lab`→`.21`, `w2.lab`→`.22`, `tango.lab`→`.30`).

**Callout — Pitfall: the NAT address reshuffle.** As you start and stop clones, the throwaway `192.168.122.x` NAT addresses get reassigned among them, so SSHing to a *NAT* address may hit a different VM than you expect and trigger a "REMOTE HOST IDENTIFICATION HAS CHANGED" warning. This is harmless — it's DHCP reuse, not an attack. Once each VM has its static `labnet` IP, stop using NAT addresses entirely and reach VMs by their stable `10.10.10.x` address or name. Clear any stale warning with `ssh-keygen -R <old-ip>`.

**Checkpoint 7.8:** each of the six VMs, when reached at its static `10.10.10.x` address, shows the correct hostname (prompt reads `falak@admin`, `falak@svc`, …), `enp2s0` on the right IP, and the full six-line roster in `/etc/hosts`.

### LAB 7.9 — Give each clone a unique machine-id

`virt-sysprep` resets `/etc/machine-id`, but the way clones regenerate it can leave several sharing the golden image's original ID. A shared machine-id causes subtle problems later (DHCP clients present the same identifier, and Kubernetes keys node identity off it), so fix it now while it's fresh. **Run on each VM** (SSH in at its static IP, or over console):
```
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo reboot
```
Truncating (not deleting) `/etc/machine-id` is the correct signal to systemd: "generate a fresh one at next boot." The symlink keeps dbus's machine-id linked to systemd's, Ubuntu's normal arrangement.

**Verify uniqueness** — after all six have rebooted, from `labhost` check each:
```
for ip in 10.10.10.5 10.10.10.10 10.10.10.20 10.10.10.21 10.10.10.22 10.10.10.30; do
  echo -n "$ip: "; ssh -o StrictHostKeyChecking=accept-new falak@$ip cat /etc/machine-id
done
```
You want six **different** hex strings.

**Checkpoint 7.9:** all six machine-ids are unique.

### LAB 7.10 — Establish admin.lab as the control node (SSH keys)

The final step turns `admin` from "just a VM" into the Ansible control node: it must reach all six machines by SSH **without a password**, because Ansible runs non-interactively. **Run on admin.lab** (SSH in at `10.10.10.5`).

**Step 1 — Generate a keypair on the control node:**
```
ssh-keygen -t ed25519 -C "admin.lab-ansible-control"
```
Accept the default path; use an empty passphrase (a lab automation key — the accepted tradeoff, since Ansible must run unattended).

**Step 2 — Distribute the public key to all six** (including `admin` itself, so it can manage itself). This is the last time you type each password:
```
ssh-copy-id falak@admin
ssh-copy-id falak@svc
ssh-copy-id falak@cp
ssh-copy-id falak@w1
ssh-copy-id falak@w2
ssh-copy-id falak@tango
```

**Step 3 — Verify passwordless reach to all six:**
```
for h in admin svc cp w1 w2 tango; do echo -n "$h: "; ssh -o BatchMode=yes falak@$h hostname; done
```
`BatchMode=yes` means "fail rather than fall back to a password" — so six hostnames printing with zero prompts proves key auth works everywhere. That silent, instant output *is* the moment `admin` becomes a real control node.

**Callout — the control node's private key is the lab's crown jewels.** The private key now on `admin` (`~/.ssh/id_ed25519`) grants passwordless root-capable access to every machine in the lab. In a real facility this key would be passphrase-protected, held in an agent, and tightly access-controlled. In your lab it's fine as-is, but internalise *why* `admin` is the machine you'd guard most — it's exactly the kind of security-mindset point interviewers probe.

**Checkpoint 7.10:** from `admin.lab`, `ssh falak@<any lab host> hostname` returns instantly with no password prompt, for all six.

### THEORY 7.11 — What you have built, and what changes next

You now have the complete lab foundation:

- **`labhost`** — a KVM/libvirt hypervisor with an isolated `labnet` (`10.10.10.0/24`) network.
- **A reusable golden image** — non-LVM, self-healing SSH keys, serial console, with `pre-sysprep` and `golden-clean-v2` snapshots so you can modify-and-re-generalize or re-clone anytime.
- **Six configured VMs** — each with a static `labnet` IP, hostname, full `/etc/hosts` roster, unique machine-id, and cloud-init networking disabled.
- **A control node** (`admin.lab`) with passwordless SSH to all six — ready to drive Ansible.

**Callout — you just lived the manual-first philosophy.** Everything above you did *by hand*: installed an OS, cloned it, and configured six machines' networking, identity, and access one at a time. That is exactly the "expert logs into each host" approach from Chapter 4.5 that doesn't scale — and doing it at six machines is precisely the point, because six is small enough to fully understand yet large enough to *feel* the repetition. In Part 2 you replace this per-VM handwork with an Ansible `common` role that enforces the same baseline (packages, time sync, `/etc/hosts`, firewall) as code, and you may re-express the image build with cloud-init. Because you did it by hand first, that automation will read as obvious rather than magical — you will recognise every task as something you already did with your own fingers.

**Checkpoint 7.11 (the whole foundation):** from `admin.lab`, this loop prints six `OK` lines —
```
for h in admin svc cp w1 w2 tango; do ping -c1 -W1 $h >/dev/null 2>&1 && echo "$h OK" || echo "$h FAIL"; done
```
— proving static IPs, hostnames, `/etc/hosts` resolution, and the isolated `labnet` all work together, lab-wide, from the control node. When all six say `OK`, Part 1 is complete and Part 2 can begin.

---
## End of Part 1 — Self-assessment and glossary

### Self-assessment

If you can answer these from memory, you are ready for Part 2. They are grouped by chapter.

**Linux (Ch 1).**
1. What is the difference between a kernel and a distribution? Which two families dominate, and how do they differ?
2. What does `sudo` do, and why not log in as root?
3. Where do configuration files, logs, and VM disk images live in the filesystem tree?
4. What does a pipe (`|`) do? Give an example.
5. What are the three commands that answer "is this service up, what are its logs, does it start at boot"?
6. Explain SSH key-pair authentication, and why it (not passwords) enables automation. What is a *host key*, and how does it differ from your key pair?

**Virtualization / KVM (Ch 2).**
7. What is a hypervisor? What distinguishes type 1 from type 2, and where does KVM fit?
8. Name the three pieces of the Linux virtualization stack (KVM, QEMU, libvirt) and what each does.
9. What is a golden image, and how does the golden-image-plus-clone workflow differ from Vagrant? Why does this edition prefer it?
10. What is a snapshot, and how does it complement a golden image?

**Git (Ch 3).**
11. What is a commit? A branch? A merge conflict, and how do you resolve one?
12. Describe the merge-request-with-review workflow and why `main` stays releasable.

**Control systems (Ch 4).**
13. What is a beamline? Why are control systems distributed, heterogeneous, and safety-critical?
14. In Tango, what is a device, an attribute, a command? What is the Tango Database's role — and why is it not a bottleneck?
15. Contrast EPICS PVs with Tango devices in one sentence.
16. Give two concrete reasons a control group benefits from CI/CD and packaging.

**DevOps (Ch 5).**
17. Name the five pillars, the tool this handbook uses for each, and the problem each solves.
18. How do the golden image and Ansible divide the Infrastructure-as-Code work?

**Architecture & lab (Ch 6–7).**
19. Draw the six-VM topology from memory: names, `10.10.10.x` addresses, roles.
20. Explain the two-NIC design (NAT vs labnet) in one sentence.
21. Why choose a non-LVM golden image? Why enable the serial console before you need it? What does the self-healing SSH-key service do, and why?
22. Why configure a clone's network over the serial console rather than SSH?
23. Trace a single commit from a developer's keyboard to an operator's screen, naming the systems it passes through.

### Glossary (Part 1)

- **apt / dpkg** — Debian package manager (`apt`) and its low-level engine (`dpkg`).
- **Ansible** — agentless configuration-management tool; enforces declarative desired state over SSH; runs from the control node.
- **Argo CD** — GitOps controller reconciling a Kubernetes cluster to match a Git repo.
- **attribute / command / property (Tango)** — a device's readable-writable value / action / central config.
- **bridge (virbrN)** — the virtual switch backing a libvirt network (`virbr0` for default, `virbr10` for labnet).
- **CI/CD** — Continuous Integration / Continuous Delivery: automated test-and-build on every commit.
- **cloud-init** — first-boot VM self-configuration; disabled for networking in this lab so static netplan wins.
- **clone (virt-clone)** — a new VM made by copying a golden image's disk, with a fresh MAC.
- **commit / branch / merge (Git)** — a recorded snapshot / a line of development / combining branches.
- **control node** — `admin.lab`; the machine that runs Ansible and holds the lab's SSH private key.
- **device / device server (Tango)** — a named control object / the process hosting device instances.
- **DeviceProxy** — a client handle to a Tango device.
- **distribution (distro)** — kernel + package manager + tooling (Ubuntu, RHEL…).
- **EPICS / PV / IOC** — the other control framework / its named value / the process serving records.
- **/etc/hosts** — local name→IP file used instead of DNS in the lab.
- **generalization (virt-sysprep)** — stripping a golden image's unique identity so clones are distinct.
- **GitOps** — Git as the source of truth for deployed state, reconciled by an agent.
- **golden image** — a prepared template VM cloned to produce lab machines (`golden-ubuntu2404`).
- **Grafana / Prometheus / Alertmanager** — dashboards / metrics collection / alert routing.
- **HDB++** — Tango's archiving system.
- **host / guest** — the physical machine running VMs (`labhost`) / a VM.
- **host key** — an SSH server's own identity (distinct from your user key pair); each clone must have a unique one.
- **hypervisor** — software that runs VMs; type 1 (bare-metal) vs type 2 (hosted).
- **IaC (Infrastructure as Code)** — machine configuration as version-controlled text applied by a tool.
- **KVM** — the Linux kernel module that turns the kernel into a hypervisor.
- **kubeadm / Kubernetes** — the tool that builds a cluster / the container orchestrator (`cp`, `w1`, `w2`).
- **labnet** — the isolated lab network, `10.10.10.0/24`, bridge `virbr10`, no internet route.
- **libvirt / virsh / virt-manager** — the VM management layer / its CLI / its GUI.
- **LVM** — Logical Volume Manager; deliberately *not* used in the golden image (clone-tooling friction).
- **machine-id** — a unique per-install identifier; must be regenerated per clone.
- **merge request (MR)** — a reviewable proposal to merge a branch (GitLab term; GitHub: pull request).
- **netplan** — Ubuntu's YAML network configuration; the static `labnet` config lives in `99-labnet-static.yaml`.
- **NAT network (default)** — libvirt's `192.168.122.0/24` network giving VMs outbound internet.
- **package / repository / package manager** — software+metadata / signed collection / the installer tool.
- **qcow2** — thin-provisioned, snapshot-capable VM disk format (vs raw).
- **QEMU** — device emulation for VMs; pairs with KVM.
- **reprepro** — tool for building a signed apt repository (Part 2).
- **Sardana / Taurus** — Tango's scan-orchestration framework / GUI toolkit.
- **serial console (virsh console)** — text console over an emulated serial port; the IP-independent way in.
- **snapshot** — a saved VM state you can roll back to.
- **SSH / key pair** — encrypted remote shell / private+public keys enabling passwordless, automatable login.
- **storage pool** — a libvirt-managed location for VM disk images (`/var/lib/libvirt/images`, `/lab/vm-images`).
- **sudo / root** — run one command as administrator / the all-powerful user.
- **systemd / systemctl / journalctl / unit file** — the service manager / its control command / its log reader / a service definition.
- **Tango Database** — Tango's central naming and configuration service.
- **virt-install** — the command that creates a VM (used to build the golden image).
- **virtio** — paravirtualized device model; near-native speed; the right choice for Linux guests.
- **VM (virtual machine)** — a complete computer implemented in software.

---

*End of Part 1 (KVM/libvirt edition). Part 2 begins from exactly the state Checkpoint 7.11 leaves you in: six reachable VMs on `10.10.10.0/24`, a control node with passwordless SSH, and a reusable golden image — ready to install Ansible on `admin.lab`, write the first inventory and playbook, and begin building the DevOps platform as code.*
