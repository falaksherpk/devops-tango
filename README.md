# Building a Reproducible On-Prem DevOps Platform and a Tango Controls System

## PART 1 of 4 — Foundations from Zero: Linux, Virtualization, Git, Control Systems, and the DevOps Mental Model

*A from-scratch lab handbook. Part 1 assumes **no prior knowledge** of Linux, virtualization, Git, DevOps, or control systems. By its end you will have a working lab foundation on your own hardware and the complete mental model needed for Parts 2–4.*

---

## How to use this book

This handbook has four parts:

| Part | Title | What you build |
|---|---|---|
| **Part 1** | Foundations from Zero *(this document)* | The mental model + a prepared lab host with hypervisor, Vagrant, Ansible, and Git |
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
| **Checkpoint** | a concrete, testable "this now works" milestone — do not continue past a failed checkpoint |
| **Callout — Why** | the reasoning behind a design choice |
| **Callout — Pitfall** | a common failure and how to avoid it |
| **Try it** | a small optional experiment that deepens understanding |
| **Path A / Path B** | the two install routes used from Part 2 onward: **A = conda-forge**, **B = apt/`.deb`** |
| `$` at line start | a command you type as a normal user (do not type the `$`) |
| `#` at line start inside a code block | either a comment, or a command run as root — context makes it clear |

### The two install paths (read this once, matters from Part 2 on)

Wherever the conda-forge and Debian/apt routes genuinely differ, the book shows both:

- **Path A — conda-forge.** Fastest, most portable, fewest packaging quirks. Everything installs into a conda environment. Choose this if you just want the system working quickly.
- **Path B — apt / `.deb`.** Closest to how a real Debian-based beamline host is operated: system packages, systemd units, native `.deb` artifacts served from a signed apt repository. Slower, but it teaches the packaging skills at the centre of the DevOps role this handbook targets.

You can build the whole book on one path; the labs note the few places where a later chapter assumes a particular one.

### What you will build (the destination)

Two systems, and the bridge between them:

**The On-Prem DevOps Platform (Part 2).** A reproducible, fully automated, entirely open-source platform that develops, packages, and delivers Linux-based scientific control-system software with **no cloud dependency**: infrastructure-as-code host provisioning (Vagrant + Ansible), self-hosted GitLab with CI runners, multi-format packaging (native Debian `.deb`, Conda, Pip, plus Podman/OCI and Apptainer images) published to a self-hosted GPG-signed apt repository and conda channel, GitOps delivery (Argo CD reconciling a kubeadm Kubernetes cluster from Git), and an observability stack (Prometheus, Grafana, Alertmanager).

**The Tango Controls System (Part 3).** A distributed control system for a simulated 6–20 MeV electron-LINAC beamline, built on Tango Controls: PyTango device servers for magnet power supplies, a Ce:YAG beam-profile camera, a Faraday cup / beam-current transformer, a UHV vacuum station, and motion; Sardana for scan orchestration; Taurus operator GUIs; HDB++ archiving; alarm and interlock monitoring.

**The bridge (Part 4).** The Tango software is built, tested, packaged, and deployed **through** the DevOps platform, so a single code change flows automatically: *commit → CI tests and builds → signed artifacts → deployment → visible to operators*. That loop is the point of the whole handbook.

**A note on honesty and scope.** A software lab has no real magnets, cameras, or vacuum pumps. Every "device" in Part 3 is a **simulator** returning physically plausible values. This is not a fudge: real controls software is *always* developed against simulators first, because beam time is scarce and hardware is dangerous. Writing good simulators is itself part of the professional skill set.

### What Part 1 specifically covers

Because we assume no prior knowledge, Part 1 builds the ladder rung by rung:

1. **Chapter 1 — Linux from zero.** What an operating system, a kernel, and a distribution are; the shell; the filesystem; users and permissions; packages; processes and services; basic networking and SSH. With labs.
2. **Chapter 2 — Virtualization from zero.** What a virtual machine is, what a hypervisor does, why labs are built on VMs, snapshots, and a hands-on first VM.
3. **Chapter 3 — Git from zero.** Why version control exists, commits, branches, merges, remotes, and the review workflow every later chapter relies on. With labs.
4. **Chapter 4 — Scientific control systems and the beamline problem.** What accelerators and beamlines are, what a control system does, EPICS vs Tango, and a worked example traced through the stack.
5. **Chapter 5 — What DevOps means for scientific software.** The five pillars — IaC, CI/CD, packaging, GitOps, observability — each explained with concrete examples.
6. **Chapter 6 — The architecture we are building.** The whole picture, host topology, and the data flow you must be able to draw from memory.
7. **Chapter 7 — Preparing the lab foundation (LAB).** Installing the hypervisor, Vagrant, and Ansible; creating the pinned, version-controlled workspace Part 2 starts from.
8. **Glossary and self-assessment.**

Estimated effort for Part 1: **2–4 days** for a genuine newcomer working through every lab; a few hours if you only need the control-system and DevOps chapters.

### Hardware you need

One reasonably capable computer you are allowed to install software on:

| Resource | Minimum | Comfortable |
|---|---|---|
| RAM | 16 GB (reduced lab) | **32 GB** |
| CPU | 4 cores with virtualization extensions (VT-x / AMD-V) | 8+ cores |
| Disk | 200 GB free SSD | 500 GB SSD |
| OS | Any 64-bit Linux (labs shown on Ubuntu 22.04/24.04); Windows/macOS possible with adaptations noted | Ubuntu LTS |

If your daily machine runs Windows, the cleanest route for this handbook is to dedicate a spare machine to Ubuntu, or dual-boot. Running the whole lab under Windows is possible (VirtualBox + Vagrant work there) but every command in this book is written for a Linux host.

---

# PART 1 — FOUNDATIONS FROM ZERO

*Read Part 1 once for the mental model and do its labs once for the muscle memory; you will refer back to it constantly.*

---

## Chapter 1 — Linux from zero: the operating system under everything

Everything in this handbook — GitLab, Kubernetes, Tango, the archiver database — runs on Linux. Every scientific facility you might work at runs its control systems on Linux. So we start here, assuming you have never used it.

### THEORY 1.1 — What an operating system actually is

A computer's hardware — CPU, memory, disks, network cards — is useless raw. The **operating system (OS)** is the software layer that:

1. **manages hardware** — deciding which program gets the CPU next, which memory belongs to whom, how bytes get to the disk;
2. **provides abstractions** — programs see *files* instead of disk sectors, *processes* instead of CPU time slices, *sockets* instead of network packets;
3. **enforces boundaries** — one misbehaving program should not crash the others, and one user should not read another's files.

The core program that does all this is the **kernel**. "Linux", strictly speaking, is a kernel — started by Linus Torvalds in 1991, developed since by thousands of contributors, and open source, meaning anyone can read, modify, and redistribute its code.

A kernel alone is not usable. What you actually install is a **distribution** ("distro"): the Linux kernel plus a package manager, system services, libraries, and tooling, assembled and maintained by some organisation. Distributions cluster into families, and two families dominate scientific computing:

| Family | Members you'll meet | Package format | Package manager | Where you see it |
|---|---|---|---|---|
| **Red Hat family** | RHEL, Rocky Linux, AlmaLinux, Fedora, CentOS (historic) | `.rpm` | `dnf` / `yum` | Enterprise IT, many accelerator labs, certification world (RHCSA/RHCE) |
| **Debian family** | Debian, **Ubuntu**, Linux Mint | `.deb` | `apt` | Servers everywhere, developer desktops, **DESY beamline hosts**, this handbook's lab |

The families differ in packaging and administrative details but share the same kernel, the same shell, and the same concepts. Learn one deeply and the other is a week of adjustment. This handbook uses **Ubuntu** (Debian family) for the lab VMs because the target role centres on Debian packaging — but everything conceptual transfers to RHEL/Rocky.

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

**Callout — Why professionals prefer the terminal.** A GUI action cannot be saved, reviewed, repeated on 200 hosts, or diffed against last month. A command can. When Chapter 5 defines Infrastructure as Code, it is really saying: *convert every click into text, so it can be versioned and replayed.*

### THEORY 1.3 — The filesystem: one tree, everything is a file

Linux has no drive letters. There is a single directory tree rooted at `/`, and every disk, USB stick, or network share is **mounted** (grafted) somewhere onto that tree. The standard layout (the *Filesystem Hierarchy Standard*) gives every kind of file a conventional home:

| Directory | What lives there | You will touch it when… |
|---|---|---|
| `/home/<user>` | each user's personal files | always — your workspace is `~/lab` (`~` means "my home") |
| `/etc` | system-wide **configuration**, plain text files | configuring SSH, apt sources, `/etc/hosts` |
| `/var` | **variable** data: logs (`/var/log`), databases, caches | reading logs, hosting the apt repo in Part 2 |
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

**Callout — Pitfall.** If a command fails with `Permission denied`, the answer is *sometimes* `sudo` — but first ask *why* it was denied. Blindly sudo-ing everything (or worse, `chmod 777`, granting everyone every permission) is how newcomers create insecure, unmaintainable systems. In a control-system context, permissions are part of the safety story.

### LAB 1.1 — First steps in the shell

You need any Linux machine for Chapter 1's labs. If you don't have one yet, that's fine — **do Chapter 2 first** (it shows you how to create an Ubuntu virtual machine), then return here and run these labs inside it. The labs are written so either order works.

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
grep -r "PermitRoot" /etc/ssh 2>/dev/null   # search recursively; discard permission errors
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

Now something real that later chapters need:

```bash
sudo apt install -y git curl wget htop tree
git --version
curl --version | head -1
```

**Checkpoint 1.2:** you can explain the difference between `apt update` and `apt upgrade`, and you can find out which files a package installed.

**Try it.** Run `less /etc/apt/sources.list` (and `ls /etc/apt/sources.list.d/`). These plain-text files are the complete definition of where your system gets software. In Part 2 you will add a line here pointing at *your own* repository.

### THEORY 1.6 — Processes, services, and systemd

A **process** is a running program. Every process has a numeric **PID**, an owning user, and a parent (the process that started it). `ps aux` lists all of them; `htop` shows them live.

Most processes you care about on a server are **services** (also called **daemons**): long-running background programs with no window — a web server, a database, the SSH server, and later the Tango device servers of Part 3. Something must start services at boot, restart them when they crash, and collect their logs. On every modern distribution that something is **systemd**, and you talk to it with two commands:

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

Read that once carefully. In Part 3 you will write files exactly like this for Tango device servers, and in Part 2 your `.deb` packages will *ship* unit files so that installing a package automatically registers its service. Unit files are how software becomes *operable* rather than merely runnable.

### LAB 1.3 — Processes and services hands-on

```bash
ps aux | head -15               # a=all users, u=user-oriented, x=include daemons
ps aux | grep ssh               # find SSH-related processes
htop                            # live view; F6 sorts, F9 kills, q quits
```

Work with a real service — the SSH server (install it if absent):

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

**Callout — Why this matters for the job.** "Is the process up? What do its logs say? Does it start at boot?" is the diagnostic reflex for every layer of Parts 2–4: GitLab down, kubelet crashlooping, a device server not exporting. You just learned the universal version of it.

### THEORY 1.7 — Networking essentials: addresses, ports, names, SSH

The lab you build is a small *network* of machines, so five ideas must be solid:

**1. IP addresses.** Every machine on a network has a numeric address, e.g. `192.168.56.10`. Addresses starting `10.`, `172.16–31.`, and `192.168.` are **private** — valid only inside a local network, unreachable from the internet. The lab lives entirely in `192.168.56.0/24` (the `/24` means "the first three numbers identify the network, the last identifies the host — so hosts `192.168.56.1` through `.254`").

**2. Ports.** One machine runs many services, so an address alone isn't enough: each service listens on a numbered **port**. `192.168.56.10:80` means "the web server on that host". Conventions you'll meet: 22 = SSH, 80 = HTTP, 443 = HTTPS, 3306 = MySQL/MariaDB, 6443 = Kubernetes API, 10000 = Tango database server, 9090 = Prometheus.

**3. Names and DNS.** Humans use names (`gitlab.lab`), machines use addresses; **DNS** is the global system translating one to the other. A lab doesn't need real DNS: the file **`/etc/hosts`** provides local name→address mappings, one per line, and the lab uses it heavily:

```
192.168.56.10   gitlab.lab
192.168.56.30   tango.lab
```

**4. Clients and servers.** A **server** process waits, listening on a port; a **client** initiates a connection to `address:port`. Nearly everything in this book is client/server: your browser → GitLab; `kubectl` → Kubernetes API; a Taurus GUI → a Tango device server.

**5. SSH — the tool you'll use most.** The **Secure Shell** gives you an encrypted terminal on a remote machine: `ssh user@192.168.56.10` and you are *there*, running commands as if seated at it. SSH supports password login, but the professional standard is **key pairs**: you generate a mathematically linked pair of files — a **private key** (secret, stays on your machine, is never sent anywhere) and a **public key** (freely shareable). You place the public key on the server; SSH then proves you hold the private key without transmitting it. Keys are stronger than passwords and, crucially, **enable automation**: Ansible (Chapter 5, Part 2) is essentially "SSH into many machines and configure them", and it can only be non-interactive because keys remove the password prompt.

### LAB 1.4 — Networking and SSH hands-on

```bash
ip addr show            # your interfaces and IP addresses; 'lo' (127.0.0.1) is the machine talking to itself
ip route                # where packets go by default
ping -c 3 127.0.0.1     # 3 probe packets to yourself
cat /etc/hosts          # current local name mappings
ss -tlnp | head         # which ports are LISTENING right now (t=tcp l=listening n=numeric p=process)
```

Generate the SSH key pair you will use for the entire lab:

```bash
ssh-keygen -t ed25519 -C "lab key"
# Accept the default path (~/.ssh/id_ed25519). A passphrase is good practice;
# for a throwaway lab an empty one is acceptable.
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

**Checkpoint 1.4:** you can state your machine's IP address, explain what `192.168.56.10:22` denotes, and SSH somewhere using a key instead of a password.

**Callout — Pitfall.** The private key file must be readable only by you (`chmod 600 ~/.ssh/id_ed25519` if SSH ever complains about permissions), and it never leaves your machine. If you ever paste a *private* key into a chat, a repo, or a server, consider it burned and generate a new pair.

### THEORY 1.8 — Chapter 1 in one page

- Linux = kernel + distribution; two families (Red Hat / Debian) differing mainly in packaging; the lab uses Ubuntu, the concepts transfer.
- The shell turns administration into **text**, and text is what can be scripted, versioned, and automated — the precondition for everything in Chapter 5.
- One filesystem tree; configuration in `/etc`, logs in `/var/log`, your work in `~`.
- Normal user + `sudo` for administrative acts; permissions are a feature, not an obstacle.
- Software arrives as **signed packages** from **repositories** via a **package manager** — machinery you will consume in Part 1 and *build* in Part 2.
- Long-running software is a systemd **service**: `systemctl` controls it, `journalctl` shows its logs, a **unit file** defines it.
- Machines find each other by IP address and port; names come from DNS or `/etc/hosts`; **SSH with keys** is both your remote terminal and the substrate of automation.

---
## Chapter 2 — Virtualization from zero

The lab needs five or six Linux machines. You own one. Virtualization resolves that contradiction, and it is also — through Vagrant, VMware, and cloud computing generally — a core professional topic in its own right.

### THEORY 2.1 — What a virtual machine is

A **virtual machine (VM)** is a complete computer implemented in software: it has (virtual) CPUs, memory, a disk, and network cards, it boots its own operating system, and software running inside it cannot tell it isn't physical hardware. The physical machine that hosts VMs is the **host**; each VM is a **guest**.

The program that makes this possible is the **hypervisor**. It carves out slices of the host's real CPU, RAM, and disk and presents them to each guest as if they were whole machines, while keeping guests isolated from the host and from each other. Modern CPUs contain hardware support for this (Intel **VT-x**, AMD **AMD-V**), which is why virtualization is fast enough to be ubiquitous — and why Chapter 7 makes you verify those extensions are enabled.

Hypervisors come in two types:

- **Type 1 (bare-metal)** — the hypervisor *is* the OS layer on the hardware; guests run on top. Examples: **VMware ESXi** (the enterprise standard, clustered under vCenter), Microsoft Hyper-V, Xen, and **KVM** (which is built into the Linux kernel itself — a running Linux *is* a type-1 hypervisor). Data centres and facility server rooms run type 1.
- **Type 2 (hosted)** — the hypervisor is an application on a normal desktop OS. Examples: **VirtualBox**, VMware Workstation. Simpler to set up; ideal for a personal lab.

Some vocabulary that recurs constantly:

| Term | Meaning |
|---|---|
| **Image / box / template** | a pre-built disk file containing an installed OS, used to stamp out new VMs in seconds instead of running an installer |
| **Snapshot** | a saved point-in-time state of a VM you can roll back to — the single greatest gift virtualization gives a learner |
| **Nested virtualization** | running a hypervisor *inside* a VM; must be explicitly enabled, and matters if your "host" is itself a VM |
| **Virtual network** | software-defined networks connecting VMs to each other and (optionally) the outside; the lab uses a **host-only** network (`192.168.56.0/24`) that exists purely inside your machine |

### THEORY 2.2 — Why build the lab on VMs (and not containers, and not clouds)

Three alternatives were possible; the choice of VMs is deliberate:

1. **VMs vs several physical machines.** VMs cost nothing, are created and destroyed in minutes, can be snapshotted before risky steps, and make the entire lab reproducible with two commands (`vagrant destroy && vagrant up`). Physical mini-PCs work too (the book notes differences where relevant) but add cost and cabling without adding learning.
2. **VMs vs containers.** Containers (Docker/Podman — properly introduced in Part 2) share the host's kernel and virtualise only the process environment. They are lighter than VMs but are *not* full machines: you cannot practise host provisioning, disk layout, systemd-at-boot, or kubeadm cluster building inside them realistically. The lab therefore uses VMs as the **substrate** (standing in for a facility's physical hosts) and containers as **payload** running on that substrate — exactly the layering a real facility has.
3. **VMs vs cloud instances.** Renting machines from AWS would work mechanically, but the target environment is a facility whose control networks are **isolated from the internet by design**. Building on-prem, on your own metal, with zero cloud dependencies, is not a compromise — it is the skill being demonstrated.

**Callout — Why this maps to the CV.** The virtualization chapter is also professionally load-bearing: enterprise facilities run exactly this pattern at scale — VMware vSphere clusters (ESXi hosts under vCenter, with live-migration and shared iSCSI/NFS storage) hosting the Linux VMs on which control-system services run. Your lab's VirtualBox-or-KVM host is a one-machine miniature of that architecture, and being able to say so precisely is interview material.

### LAB 2.1 — Install a hypervisor and create your first VM by hand

We create one VM manually — clicking through the real Ubuntu installer once — *before* automating everything, because you cannot appreciate what Vagrant automates until you've done the manual version. If you have no Linux machine yet, this lab is also how you get one for Chapter 1's labs.

**Step 1 — Verify hardware virtualization is available.**

On a Linux host:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo    # >0 means VT-x/AMD-V is present and enabled
```

If it prints `0`, reboot into your machine's firmware (BIOS/UEFI) settings and enable *Intel VT-x* / *AMD-V* / *SVM*. On Windows, check Task Manager → Performance → CPU → "Virtualization: Enabled".

**Step 2 — Install VirtualBox.**

- Linux host: `sudo apt install -y virtualbox`
- Windows/macOS host: download the installer from virtualbox.org and run it.

(KVM/libvirt is the leaner choice on a Linux host and Chapter 7 shows it; VirtualBox is used here because it is identical on all three host OSes and pairs with Vagrant out of the box.)

**Step 3 — Download an Ubuntu Server ISO.**

Fetch the **Ubuntu Server 22.04 LTS** ISO from ubuntu.com/download/server (~2 GB). "Server" means no graphical desktop — matching real beamline hosts and using far less RAM.

**Step 4 — Create the VM.**

In VirtualBox: **New** →

- Name: `first-vm`; Type: Linux; Version: Ubuntu (64-bit)
- Memory: 2048 MB; CPUs: 2
- Disk: 20 GB, dynamically allocated (grows only as used)
- Settings → Storage → attach the ISO to the virtual optical drive
- Start the VM.

**Step 5 — Install Ubuntu inside it.**

The installer walks you through: language, keyboard, network (accept defaults), disk ("use entire disk" — it means the entire *virtual* disk), your name/username/password, and — **important** — tick **"Install OpenSSH server"** when offered. Skip the featured snaps. Reboot when prompted (VirtualBox auto-detaches the ISO). Log in at the text console with the user you created.

**Step 6 — Look around: it is simply a Linux machine.**

```bash
whoami && hostname
ip addr show        # note its IP address
free -h             # its 2 GB of "RAM"
df -h /             # its 20 GB "disk"
sudo apt update && sudo apt install -y htop
```

Everything from Chapter 1 applies verbatim in here. If you skipped Chapter 1's labs for lack of a Linux machine, do them now inside this VM.

**Step 7 — Take a snapshot, break the machine, roll back.**

This exercise, more than any lecture, teaches why virtualization changes how you learn:

1. In VirtualBox: Machine → Take Snapshot → name it `clean-install`.
2. Inside the VM, do something destructive: `sudo rm -rf /usr/bin` (yes, really — it deletes most programs; the machine is now badly broken; commands stop working).
3. Power off the VM. Snapshots panel → select `clean-install` → **Restore**. Boot.
4. The machine is whole again. Thirty seconds, total recovery.

**Checkpoint 2.1:** you created a VM, installed Ubuntu in it, can SSH-or-console into it, and can restore it from a snapshot. You also now viscerally know what "the installer" involves — remember the ~15 minutes it took, because Chapter 7 replaces it with one command and 90 seconds.

**Try it.** With the VM running, on the *host* run `VBoxManage list runningvms` and `VBoxManage showvminfo first-vm | head -30`. VirtualBox is fully controllable from the command line — which is precisely the hook Vagrant uses to automate it.

### THEORY 2.3 — From hand-made VMs to Vagrant (the bridge to Chapter 5)

Creating one VM by hand took you a quarter of an hour of choices — each one unrecorded and unrepeatable. The lab needs five VMs, rebuilt possibly many times. **Vagrant** closes that gap: you describe the fleet in one text file (a `Vagrantfile` — names, memory, CPUs, IP addresses, base image), and `vagrant up` builds it, using pre-installed base **boxes** so no installer ever runs. `vagrant destroy` deletes everything; `vagrant ssh <name>` connects. The `Vagrantfile` lives in Git, so *your infrastructure has a version history*.

That is the first half of Infrastructure as Code. The second half — configuring what runs *inside* the machines — is Ansible's job, and both are set up for real in Chapter 7 and used in anger from Part 2 onward.

---

## Chapter 3 — Git from zero: version control as the foundation of everything

Every later chapter stores something in Git: application code, the `Vagrantfile`, Ansible roles, CI pipeline definitions, Kubernetes manifests, even this handbook if you like. GitOps (Part 2) goes further and makes Git the authoritative definition of what is *running*. So Git cannot remain vague.

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

On any machine with Git installed (`sudo apt install -y git`).

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

**Checkpoint 4 (conceptual):** in one sentence each, without notes: what is a beamline; what is a device in Tango; what is a PV in EPICS; why are control systems distributed; name two concrete reasons a control group benefits from CI/CD and packaging.

---

## Chapter 5 — What DevOps means for scientific software

"DevOps" is one of the most overloaded words in computing — a culture, a job title, a toolbox, a buzzword. For this handbook it decomposes into **five concrete capabilities**. Each gets: the problem, the idea, a worked example, and the specific tool Part 2 uses. Read this chapter slowly; it is the map for everything that follows.

### THEORY 5.0 — Where the word comes from, in one paragraph

Traditionally, **Dev**elopers wrote software and threw it over a wall to **Op**erations, who ran it. The wall produced misery: software that worked "on my machine" but not in production, releases so painful they happened twice a year, and blame in both directions. The DevOps movement (from ~2009) dissolved the wall: the same practices, the same tooling, and often the same people carry software from a code change to a monitored production system, with **automation replacing hand-offs**. In a scientific control group the "production system" is a beamline, and the "release" is a new device server or analysis tool reaching experiment hosts — but the logic is identical.

### THEORY 5.1 — Pillar 1: Infrastructure as Code (IaC)

**The problem.** Chapter 2's Lab took you ~15 minutes of unrecorded clicking to make *one* machine. Facilities have hundreds. Hand-built machines drift apart, can't be audited, and can't be rebuilt.

**The idea.** Don't *perform* configuration — **describe** it, in text files, and let a tool make reality match the description. Two complementary flavours:

- **Provisioning** — creating the machines themselves. Tool here: **Vagrant** (for lab VMs; clouds use Terraform; a vSphere shop uses templates/APIs — same idea).
- **Configuration management** — installing and configuring software *on* the machines. Tool here: **Ansible**, which connects to hosts over plain SSH (Chapter 1's keys!) and applies **playbooks**: YAML files listing desired states.

**Worked example.** A fragment of a real Ansible task list, with translation:

```yaml
# roles/common/tasks/main.yml
- name: Ensure chrony (NTP) is installed        # ← human-readable intent
  apt:
    name: chrony                                 # ← package to ensure present
    state: present

- name: Ensure chrony is enabled and running
  service:
    name: chrony
    state: started
    enabled: true                                # ← start at boot, too

- name: Deploy hardened SSH configuration
  template:
    src: sshd_config.j2                          # ← a template in Git
    dest: /etc/ssh/sshd_config
  notify: restart ssh                            # ← only restart if the file CHANGED
```

Notice the grammar: not "run these commands" but "**this is how the world should be**" — `state: present`, `state: started`. That declarative style yields the key property to internalise, **idempotence**: running the playbook *twice* changes nothing the second time, because everything already matches. Idempotence is what makes automation safe to re-run — on one host or three hundred — and you will prove it experimentally in Part 2 (a second `ansible-playbook` run reports `changed=0`).

**The payoff.** The machine is now *defined by files in Git*: rebuildable identically, reviewable in merge requests, auditable (`git log` of your infrastructure), and self-documenting. When a lab host dies, recovery is `vagrant up` + playbook, not archaeology.

### THEORY 5.2 — Pillar 2: Continuous Integration / Continuous Delivery (CI/CD)

**The problem.** Two developers work for a month on separate copies of the code; merging their work becomes a crisis ("integration hell"). And even solo, nothing guarantees the code in Git actually builds and passes tests *right now* — until someone checks, usually at the worst moment.

**The idea.** **Continuous Integration**: every push to the repository triggers an automated system that checks the code out on a clean machine, builds it, and runs the tests — so breakage is detected within minutes of being introduced, while the change is small and the author remembers everything. **Continuous Delivery** extends the pipeline: after tests pass, it *packages* the software into deployable, versioned artifacts automatically — so a releasable build exists at all times, and "making a release" stops being an event.

**The tools here.** **GitLab CI**: pipelines are defined in a file named `.gitlab-ci.yml` *in the repository itself* (pipeline-as-code — notice the pattern repeating), and executed by **runners** — worker processes you will self-host in Part 2.

**Worked example.** A miniature but realistic pipeline for the Part 3 device-server code:

```yaml
# .gitlab-ci.yml
stages: [test, build]

lint:                          # job 1 — style/static checks
  stage: test
  image: python:3.11           # run inside a clean container
  script:
    - pip install ruff
    - ruff check src/

unit-tests:                    # job 2 — runs in parallel with lint
  stage: test
  image: python:3.11
  script:
    - pip install -e .[test]
    - pytest -q                # any failing test fails the pipeline

build-deb:                     # job 3 — only reached if stage 'test' passed
  stage: build
  script:
    - dpkg-buildpackage -us -uc
  artifacts:
    paths: ["../*.deb"]        # keep the built package
```

Read the control flow: push → both `test` jobs run on runners → only if both succeed does `build-deb` run → the pipeline's product is a `.deb` file. Combine this with Chapter 3's merge requests and you get the **quality gate**: a merge request shows red until its pipeline is green, and team policy (enforced by GitLab) says red doesn't merge. *No artifact reaches a host without passing the gate* — a sentence you should be able to defend in an interview.

### THEORY 5.3 — Pillar 3: Software packaging

**The problem.** A directory of source files is not deployable. "Deploy" must mean something precise: put **version X** of component Y, with **all its dependencies**, onto host Z, verifiably and reversibly. The thing that makes that possible is a **package**: a versioned, dependency-declaring, integrity-checkable, installable artifact (Chapter 1.5, now from the producer's side).

**The complication — and the reason this role exists.** Scientific software must serve several ecosystems *at once*, because a facility's users live in all of them:

| Format | What it is | When it's the right tool | Built with |
|---|---|---|---|
| **Debian `.deb`** | native OS package; installs into system paths; can ship systemd units, users, config | the *operated* beamline host: software that must start at boot and be managed like the OS itself | `debhelper`, `dh-python` |
| **Conda** | packages into self-contained *environments*, independent of the OS; excels at scientific stacks with C/Fortran guts | scientists' analysis environments; multi-version coexistence; the whole conda-forge ecosystem | `conda-build` / `rattler-build` |
| **Pip / wheel** | Python's own format, installed into a Python environment | pure-Python libraries; anything consumed as `import beamline_utils` | `build`, `pyproject.toml` |
| **OCI container image** | a frozen filesystem + process definition, run isolated by Podman/Docker | services deployed on Kubernetes (Part 2); identical bits from CI to production | `Containerfile` + `podman build` |
| **Apptainer (`.sif`)** | container format designed for HPC clusters (no root daemon, single-file images) | running on shared compute clusters where Docker is unwelcome | `apptainer build` |

Part 2 builds **one example component in all five formats**, because being fluent in the trade-offs — *why* the archiver is a `.deb` but the analysis library is Conda and the web service is OCI — is exactly the judgment the job requires.

**Distribution closes the loop.** Packages are only useful if hosts can fetch them: Part 2 stands up a **GPG-signed apt repository** (with `reprepro`) and a **conda channel**, self-hosted, so that a lab VM can run `sudo apt install beamline-utils` and receive *your* signed artifact from *your* repository — the full producer-to-consumer chain with no cloud in between.

### THEORY 5.4 — Pillar 4: GitOps

**The problem.** CI produced versioned artifacts — but *deployment* can still be a human typing commands at servers, which is unrecorded, error-prone, and drifts.

**The idea.** **GitOps** makes deployment declarative and automatic: the desired running state of your systems — *which* components, *which* versions, *how many* replicas, *what* configuration — is described in files in a Git repository, and an **agent** runs in the target environment continuously **reconciling** reality against that description. Reality drifted (someone hand-edited something, a process died)? The agent corrects it. Want to change production? You change Git — through a reviewed merge request, of course.

The consequences are profound and easy to state:

- **Audit**: `git log` of the deployment repo *is* the complete deployment history — who changed what, when, why.
- **Rollback**: `git revert`, and the agent walks production back.
- **Self-healing**: drift is detected and corrected continuously, not discovered during an incident.
- **Disaster recovery**: the cluster's entire desired state survives in Git; rebuild the cluster, point the agent at the repo, and it restores everything.

**The tools here.** The agent is **Argo CD**; the environment it reconciles is a **Kubernetes** cluster. Kubernetes gets a full theory section in Part 2; for now the one-paragraph version: *Kubernetes is an orchestrator that runs container images across a pool of machines. You give it declarative manifests — "run 2 replicas of image `registry.lab/magnet-sim:1.4.2`, expose port 10000, restart it if it dies" — and it continuously makes the cluster match.* Notice that Kubernetes is itself a reconciliation engine; Argo CD simply extends the reconciliation all the way back to Git. Declarative-and-reconciled is the same idea you met as idempotence in Pillar 1 — one mental model, three scales.

**Worked example.** In Part 4 the loop closes like this: CI builds `magnet-sim:1.4.3` and pushes it to the registry; a one-line change in the manifests repo bumps the image tag; the merge request is reviewed and merged; Argo CD notices Git changed, rolls the deployment, and the new simulator is live — with zero human contact with the cluster. That end-to-end story is the demo this whole handbook builds toward.

### THEORY 5.5 — Pillar 5: Observability

**The problem.** You now operate a dozen services across half a dozen hosts. "Is everything OK?" must have a better answer than "no one has complained yet" — especially at a facility, where the person who notices breakage may be a scientist mid-experiment at 3 a.m.

**The idea.** Instrument everything to emit **metrics** — numeric time series like *requests per second*, *CPU %*, *queue depth* — collect them centrally, graph them, and define **alerts** that page a human when defined conditions hold. (Metrics are one of observability's three classic signal types, alongside **logs** — which you met in `journalctl` — and **traces**; this lab focuses on metrics + logs.)

**The tools here.** **Prometheus** *scrapes* metrics: every service exposes a plain-text `/metrics` HTTP endpoint, and Prometheus polls them all on a schedule, storing the time series. **Grafana** draws dashboards from that store. **Alertmanager** routes alerts (deduplicating, grouping, silencing) to humans.

**What to watch — the Golden Signals.** The discipline of *choosing* metrics is captured by four signals, defined in Google's SRE practice, which you will implement for the lab's services:

1. **Latency** — how long requests take (e.g. attribute-read time on a device server);
2. **Traffic** — demand (requests/second, event rate into the archiver);
3. **Errors** — rate of failed requests (exceptions per minute on a Tango server);
4. **Saturation** — how *full* the system is (CPU, memory, disk, queue depth) — the leading indicator of trouble ahead.

**Worked example.** An alert rule in Prometheus's language, with translation:

```yaml
- alert: HostDiskAlmostFull
  expr: (node_filesystem_avail_bytes{mountpoint="/"} 
         / node_filesystem_size_bytes{mountpoint="/"}) < 0.10
  for: 10m                      # condition must hold 10 minutes (no flapping)
  labels: {severity: warning}
  annotations:
    summary: "Root filesystem on {{ $labels.instance }} below 10% free"
```

*If any host's root disk stays under 10% free for ten minutes, alert.* An archiver quietly filling a disk is a classic facility incident; this rule is how it becomes a Tuesday-afternoon ticket instead of a Saturday-night outage. Closing the loop operationally, alerts should link to **runbooks** — written diagnosis-and-fix procedures — so the responder at 3 a.m. isn't improvising; Part 2 has you write one.

### THEORY 5.6 — Why "on-prem" and "fully open-source" are requirements, not constraints

Two deliberate properties of the Part 2 platform deserve explicit defence, because they are typical of scientific facilities and atypical of internet tutorials:

- **On-prem, no cloud.** Facility control networks are commonly **isolated from the internet** — for security (a beamline PLC must never be reachable from outside) and for autonomy (an experiment cannot pause because a cloud region hiccuped). Therefore *every* platform component — the Git server, CI runners, package repositories, container registry, monitoring — must be **self-hosted** and must function air-gapped. Most online DevOps education assumes GitHub + AWS; this handbook deliberately does not, because the self-hosted skill set is the one facilities value.
- **Fully open-source.** No licences to count across hundreds of hosts, no vendor who can discontinue or re-price your stack, full source access for audit and patching over the decades-long life of a facility — and the ability to *contribute back*, which research institutions both value and (as the DESY posting says) expect: "significant portions of your work contribute to open-source projects."

**Callout — Pitfall.** Newcomers assume "DevOps = the cloud". It does not. Every concept in this chapter — IaC, CI/CD, packaging, GitOps, observability — is fully realisable on bare metal in an air-gapped room, and that realisation *is* this handbook.

### THEORY 5.7 — The five pillars on one card

Commit this table to memory; it is the skeleton of Part 2 and a legitimate interview answer:

| Pillar | Problem it kills | Core idea | Tools in this lab |
|---|---|---|---|
| Infrastructure as Code | hand-built, drifting, unrebuildable hosts | describe machines in versioned text; tools reconcile; idempotence | Vagrant + Ansible |
| CI/CD | late integration; untested code; manual release pain | every push is built and tested; artifacts produced automatically; green-gated merges | GitLab CI + self-hosted runners |
| Packaging | "a pile of files" is not deployable | versioned, signed, dependency-aware artifacts in the formats users need | debhelper/dh-python, conda-build, pip/wheel, Podman, Apptainer; reprepro apt repo + conda channel |
| GitOps | manual, unrecorded, drifting deployments | Git holds desired state; an agent reconciles continuously | Argo CD + Kubernetes (kubeadm) |
| Observability | flying blind; users discover your outages | metrics scraped centrally; dashboards; alerts on Golden Signals; runbooks | Prometheus + Grafana + Alertmanager |

**Checkpoint 5 (conceptual):** for each pillar, you can state — without notes — the problem, the idea, and the tool. Then explain to an imaginary colleague why idempotence, declarative manifests, and GitOps reconciliation are the *same idea at three scales*.

---
## Chapter 6 — The architecture we are building

Before touching a keyboard in Part 2, hold the end state in your head. Everything in this chapter must become drawable from memory.

### THEORY 6.1 — The whole picture

The lab is a small cluster of Linux hosts (VMs on your workstation). On them:

- a **services host** (`svc`) runs GitLab (source control + container registry), the GPG-signed apt repository and conda channel, and the observability stack (Prometheus, Grafana, Alertmanager);
- a small **Kubernetes cluster** — one control-plane node (`cp`) and two workers (`w1`, `w2`), built the explicit way with **kubeadm** — is the deployment target, reconciled from Git by **Argo CD**;
- a **Tango host** (`tango`) runs the control system of Part 3: the Tango database (MariaDB-backed), the simulated device servers, Sardana, Taurus, and HDB++.

Code flows left to right: a developer pushes to GitLab; CI runners test the change and build multi-format packages, pushing them to the registry and repositories; Argo CD (for cluster workloads) or Ansible/apt (for host services) deploys them; Prometheus watches everything.

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

### THEORY 6.2 — Narrating the diagram (the story you must be able to tell)

Practise saying this aloud until fluent; it is Part 4's demo and a superb interview answer:

> "A developer changes the magnet device server and pushes a branch to our self-hosted GitLab. That push triggers a CI pipeline on our own runners: linting, unit tests against a simulated device, then package builds — a native `.deb` via debhelper, a conda package, a wheel, and an OCI image. The `.deb` and conda artifacts are GPG-signed and published to our self-hosted apt repository and conda channel; the image goes to GitLab's container registry. Nothing merges to main unless the pipeline is green and a colleague has reviewed the diff. Deployment is GitOps: a manifests repository declares what should run on the Kubernetes cluster, and Argo CD continuously reconciles the cluster to it — so releasing the new simulator is a one-line, reviewed change of an image tag. Host-installed services like the archiver update from the signed apt repo under Ansible control. Prometheus scrapes every layer — hosts, cluster, and the control system itself — Grafana dashboards it, and Alertmanager pages on Golden-Signals rules with runbooks attached. Every environment is rebuildable from Git: Vagrant and Ansible define the hosts, kubeadm's configuration is code, and the cluster's contents are whatever the manifests repo says."

Every clause of that paragraph is a chapter in Parts 2–4.

### THEORY 6.3 — Host topology and sizing

The book assumes **Topology A: one workstation running several VMs**, because it is cheap, snapshot-able, and fully reproducible with `vagrant destroy && vagrant up`.

| VM | Role | RAM | vCPU | Disk | IP (host-only net) |
|---|---|---|---|---|---|
| `svc` | GitLab, registry, apt/conda repos, observability | 6 GB | 2 | 60 GB | 192.168.56.10 |
| `cp` | Kubernetes control-plane | 4 GB | 2 | 30 GB | 192.168.56.20 |
| `w1` | Kubernetes worker | 4 GB | 2 | 30 GB | 192.168.56.21 |
| `w2` | Kubernetes worker | 4 GB | 2 | 30 GB | 192.168.56.22 |
| `tango` | Tango DB, device servers, Sardana, Taurus, HDB++ | 4–6 GB | 2 | 30 GB | 192.168.56.30 |

Totals: ~22–24 GB RAM, 10 vCPUs (oversubscription is fine), ~180 GB disk. On a 16 GB machine, run a **reduced lab**: drop `w2`, give `svc` 5 GB, and expect GitLab to be slow but functional; every lab notes where the reduction matters.

**Topology B** (several physical mini-PCs on a real switch) works identically — substitute real IPs and skip Vagrant in favour of manual OS installs plus the same Ansible — and is mentioned where it differs, but is not required.

**Callout — Why VMs and not just containers (reprise).** Building the platform on VMs teaches host-level provisioning and lets you practise kubeadm against "real" machines — the substrate is deliberately host-shaped, matching a facility, while containers appear *inside* the platform as CI jobs, images, and pods.

**Callout — Why kubeadm and not an easier Kubernetes.** One-command distributions (k3s, minikube, kind) are excellent tools — and k3s appears as a sidebar in Part 2 — but they hide exactly the parts (control-plane components, certificates, networking bring-up) that teach you how Kubernetes actually fits together. kubeadm makes you perform each step explicitly once; after that, the easy tools are a convenience rather than a mystery.

### THEORY 6.4 — Naming and networks

All VMs share a **host-only network**, `192.168.56.0/24` — a virtual switch that exists only inside your workstation. Your workstation itself gets `192.168.56.1` on it, so your browser can reach lab services. Names are managed with `/etc/hosts` entries (Chapter 1.7) on the workstation and, via Ansible, on every VM:

```
192.168.56.10  gitlab.lab  registry.lab  repo.lab  grafana.lab  prometheus.lab
192.168.56.20  cp.lab
192.168.56.21  w1.lab
192.168.56.22  w2.lab
192.168.56.30  tango.lab
```

Each VM also has a NAT interface for outbound internet (downloading packages during builds); nothing inbound crosses from the internet into the lab. This mirrors the facility pattern: an internal service network, controlled egress, no public exposure.

**Checkpoint 6:** close the book and (a) draw the data-flow diagram, (b) recite the five VMs with roles, (c) explain in one sentence each why the lab is VM-based, kubeadm-based, and cloud-free.

---

## Chapter 7 — Preparing the lab foundation (LAB)

Everything so far was ladder-building. This chapter does the first *real* work: preparing your physical host with the three tools Part 2 assumes — a hypervisor, Vagrant, and Ansible — and creating the version-pinned, Git-tracked workspace the whole platform will live in.

### THEORY 7.1 — What the host needs and why

Only three things are installed on the physical host, and it matters *why* only three:

1. a **hypervisor** — to run the VMs (Chapter 2);
2. **Vagrant** — to define and create those VMs from a file (provisioning half of IaC);
3. **Ansible** — to configure everything inside them (configuration-management half).

Everything else — GitLab, Kubernetes, Tango, databases, monitoring — is installed **by Ansible, inside VMs**. The host stays clean, and the entire lab remains disposable: nothing of value lives outside Git and the VMs it can regenerate.

Two hypervisor options are supported; pick **one**:

| | VirtualBox | KVM/libvirt |
|---|---|---|
| Host OS | Linux, Windows, macOS(Intel) | Linux only |
| Weight | heavier, GUI included | lighter (kernel-native), CLI/virt-manager |
| Vagrant support | built-in, zero-config | via `vagrant-libvirt` plugin |
| Choose it if… | you want the smoothest path or a non-Linux host | your host is Linux and you want facility-realistic tooling |

The labs default to VirtualBox commands and note libvirt differences.

### LAB 7.1 — Verify the host

```bash
# CPU virtualization extensions (must be > 0; else enable VT-x/AMD-V in firmware):
egrep -c '(vmx|svm)' /proc/cpuinfo

# Resources (compare against Chapter 6 sizing):
free -h                 # total RAM
nproc                   # CPU cores
df -h ~                 # free disk where the lab will live
```

**Callout — Pitfall (nested virtualization).** If your "host" is itself a VM (e.g. a cloud instance or a VM on a big server), the guest VMs will not boot unless **nested virtualization** is enabled on the outer hypervisor. On genuinely physical hardware this doesn't apply — but the firmware toggle for VT-x/AMD-V still does.

### LAB 7.2 — Install the host tooling

Shown for an Ubuntu host; adapt package names for Fedora/RHEL (`dnf` equivalents noted inline).

```bash
sudo apt update
sudo apt install -y git curl gnupg2 make build-essential
```

**Hypervisor — choose ONE:**

```bash
# Option 1: VirtualBox (simplest with Vagrant)
sudo apt install -y virtualbox

# Option 2: KVM/libvirt (lighter; Linux hosts)
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst
sudo usermod -aG libvirt $USER      # allow your user to manage VMs; log out/in to apply
```

**Vagrant** — installed from HashiCorp's own apt repository, and the commands are worth reading closely because *adding a third-party signed repository* is precisely the machinery you'll build for yourself in Part 2:

```bash
# 1. Download HashiCorp's GPG public key and store it where apt looks for keys:
wget -O- https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg

# 2. Add a repository definition line: "packages from this URL, verified by that key":
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 3. Refresh the index (apt now knows HashiCorp's packages) and install:
sudo apt update && sudo apt install -y vagrant

# If you chose libvirt, add the provider plugin:
sudo apt install -y libvirt-dev ruby-dev
vagrant plugin install vagrant-libvirt
```

Pause on what steps 1–2 did: you fetched a publisher's **public key**, told apt to trust packages **signed** by it, and pointed apt at the publisher's **repository**. In Part 2 you are the publisher: you generate the key, sign the packages, serve the repository — and a lab VM performs these exact three steps against *you*.

**Ansible:**

```bash
sudo apt install -y ansible
ansible --version        # confirm core >= 2.14
```

Smoke-test Ansible against the only host you have so far — itself:

```bash
ansible localhost -m ping        # module "ping": full round-trip test → "pong"
ansible localhost -m setup | head -40   # "facts": everything Ansible discovers about a host
```

### LAB 7.3 — Create the workspace and pin versions

Reproducibility starts with **pinning**: recording the exact versions of everything, so the lab you build today can be rebuilt identically next year — and so that when something breaks after an upgrade, you know what changed. Create the workspace as a Git repository from minute one:

```bash
mkdir -p ~/lab/devops-platform && cd ~/lab/devops-platform
git init

cat > versions.md <<'EOF'
# Pinned versions (update as you install; keep the lab reproducible)
Host OS:           (record yours, e.g. Ubuntu 24.04)
Hypervisor:        (VirtualBox x.y.z  |  libvirt x.y)
Vagrant:           (vagrant --version)
Ansible:           (ansible --version, core line)
Ubuntu guest box:  22.04 LTS (bento/ubuntu-22.04)
Kubernetes:        v1.30.x        (pin exact patch in Part 2)
containerd:        distro package (record exact version in Part 2)
GitLab CE:         record exact omnibus version at install time
Argo CD:           record manifest version at install time
Tango / PyTango:   record exact versions in Part 3
EOF

cat > README.md <<'EOF'
# On-Prem DevOps Platform — lab workspace
Built following the four-part handbook. Every host, service, and deployment
in this lab is defined by files in this repository.
EOF

git add . && git commit -m "chore: lab skeleton and version pins"
```

Fill in the versions you actually have:

```bash
vagrant --version
ansible --version | head -1
vboxmanage --version 2>/dev/null || virsh --version
# edit versions.md accordingly, then:
git add versions.md && git commit -m "chore: record host tool versions"
```

**Checkpoint 7:** `vagrant --version`, `ansible --version`, and your hypervisor all run; `egrep -c '(vmx|svm)' /proc/cpuinfo` is non-zero; `ansible localhost -m ping` says `pong`; and `git log --oneline` in `~/lab/devops-platform` shows your two commits. **This state is exactly where Part 2, Chapter 1 begins.**

### LAB 7.4 — (Optional but recommended) a 90-second taste of Vagrant

To close the loop Chapter 2 opened — and to verify the whole toolchain end to end — bring up one throwaway VM the automated way:

```bash
mkdir -p ~/lab/vagrant-taste && cd ~/lab/vagrant-taste
cat > Vagrantfile <<'EOF'
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  config.vm.hostname = "taste"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 1
  end
end
EOF

vagrant up            # downloads the base box once (~500 MB), then boots in ~90 s
vagrant ssh           # you are inside a fresh Ubuntu VM
hostname && whoami && exit
vagrant destroy -f    # gone without a trace
```

Compare: Chapter 2's manual install was ~15 minutes of unrecorded clicking; this was one committed text file and three commands. Multiply by five VMs and by every rebuild you'll ever do, and you have felt — not just read — the case for Infrastructure as Code.

**Callout — Pitfall.** If `vagrant up` fails with a VT-x/AMD-V error while VirtualBox is installed alongside KVM, the two hypervisors are contending for the CPU's virtualization extensions; unload KVM (`sudo modprobe -r kvm_intel kvm` — or `kvm_amd`) or standardise on one hypervisor.

---

## End of Part 1 — Self-assessment and glossary

### Self-assessment (answer without notes; revisit the named section otherwise)

1. What is the difference between a kernel and a distribution, and which family does Ubuntu belong to? *(1.1)*
2. Why do professionals prefer commands over GUIs for administration — what property do commands have that clicks lack? *(1.2, 5.1)*
3. Where does system configuration conventionally live in the filesystem, and where do logs live? *(1.3)*
4. What does `sudo` do, and why not simply work as root? *(1.4)*
5. What three guarantees does a package manager give you that "download and run an installer" does not? *(1.5)*
6. A service isn't running after reboot. Which two `systemctl` sub-commands and which log command do you reach for? *(1.6)*
7. What does `192.168.56.10:8080` denote, piece by piece? What file maps `gitlab.lab` to that address without DNS? *(1.7)*
8. Why do SSH keys, not passwords, make automation possible? *(1.7)*
9. What is a hypervisor, and what is the practical difference between type 1 and type 2? *(2.1)*
10. Give two reasons the lab is built on VMs rather than containers, and one reason it is cloud-free. *(2.2, 5.6)*
11. What is a commit, and why does a merge conflict exist at all? *(3.1)*
12. Describe the merge-request workflow and where CI attaches to it. *(3.2, 5.2)*
13. What is a beamline? Why is PETRA IV described as needing control-system software at scale? *(4.1)*
14. Contrast EPICS and Tango in two sentences: what is the unit of the world-model in each? *(4.3)*
15. Trace "set dipole 3 to 120 A" through the Tango stack: name at least five components it touches. *(4.4)*
16. Define idempotence and explain why it makes automation safe to re-run. *(5.1)*
17. What does "no artifact reaches a host without passing the gate" mean mechanically? *(5.2)*
18. When is a `.deb` the right format, and when is Conda? One sentence each. *(5.3)*
19. What does Argo CD continuously do, and what three operational benefits follow? *(5.4)*
20. Name the four Golden Signals and give a beamline-flavoured example of each. *(5.5)*
21. Draw the Chapter 6 architecture from memory and narrate one commit's journey through it. *(6.1–6.2)*
22. Which three tools live on the physical host, and why only those three? *(7.1)*

### Glossary

| Term | Definition |
|---|---|
| **Ansible** | agentless configuration-management tool; applies declarative YAML playbooks to hosts over SSH |
| **Apptainer** | container format/runtime designed for HPC clusters; single-file `.sif` images, no root daemon |
| **apt / dpkg** | Debian-family package manager and its low-level engine |
| **Argo CD** | GitOps agent: continuously reconciles a Kubernetes cluster to manifests stored in Git |
| **attribute (Tango)** | a named, typed, readable/writable value exposed by a device, with units, limits, alarm metadata |
| **beamline** | an experimental station receiving X-rays from a light source; effectively a small machine of its own |
| **branch (Git)** | a movable pointer to a line of development; merged back via merge requests |
| **Channel Access** | EPICS's classic network protocol for reading/writing/subscribing to PVs |
| **CI/CD** | continuous integration (build+test every change) / continuous delivery (auto-produce deployable artifacts) |
| **commit** | one recorded snapshot of a repository with author, message, and parent(s) |
| **Conda / conda-forge** | environment-based packaging system for (especially scientific) software; community channel of recipes |
| **container (OCI image)** | a frozen filesystem + process definition run in isolation by Podman/Docker; the deployable unit on Kubernetes |
| **control system** | the software/hardware layer giving a uniform networked interface to a physical machine's heterogeneous hardware |
| **daemon / service** | long-running background process, managed at boot and in failure by systemd |
| **`.deb`** | Debian-native package: files + metadata + dependencies + (optionally) systemd units and install scripts |
| **device / device server (Tango)** | an object representing an instrument (attributes, commands, properties) / the process hosting such objects |
| **DNS / `/etc/hosts`** | global name→address system / its local, file-based stand-in used throughout this lab |
| **EPICS** | control framework modelling the world as a flat namespace of Process Variables served by IOCs |
| **GitOps** | delivery pattern: Git holds desired state; an agent continuously reconciles reality to it |
| **Golden Signals** | latency, traffic, errors, saturation — the four metrics to watch first on any service |
| **GPG signature** | cryptographic proof of a package's origin and integrity; verified by apt/conda clients |
| **HDB++** | Tango's event-driven archiving system storing attribute history in a database |
| **hypervisor** | software that runs virtual machines (type 1: ESXi, KVM; type 2: VirtualBox) |
| **IaC (Infrastructure as Code)** | defining machines and their configuration in versioned text files applied by tools |
| **idempotence** | property that re-applying the same configuration changes nothing once the state is correct |
| **interlock** | hardware/software logic forcing a system to a safe state when conditions are violated |
| **IOC (EPICS)** | Input/Output Controller: the process serving a set of PVs |
| **kubeadm** | the explicit, step-by-step official tool for bootstrapping a Kubernetes cluster |
| **Kubernetes** | orchestrator that runs container workloads across a pool of machines, reconciling toward declarative manifests |
| **LINAC** | linear accelerator; the simulated 6–20 MeV electron machine of Part 3 |
| **merge request** | a reviewable, CI-gated proposal to merge a branch (GitHub: pull request) |
| **PETRA III / PETRA IV / FLASH** | DESY's synchrotron light source / its fourth-generation upgrade project / DESY's free-electron laser |
| **Prometheus / Grafana / Alertmanager** | metrics scraper+store / dashboards / alert routing — the observability stack |
| **PV (Process Variable)** | a named value in EPICS's global namespace |
| **PyTango** | the Python binding for Tango; the language of Part 3's device servers |
| **repository (package)** | a signed, indexed collection of packages that hosts install from |
| **repository (Git)** | a directory whose complete history Git tracks |
| **root / sudo** | the all-powerful administrative user / the command to run a single command as root |
| **runbook** | a written diagnose-and-fix procedure linked from an alert |
| **runner (GitLab)** | a worker process that executes CI jobs |
| **Sardana** | Tango-based experiment orchestration: macros, scans, measurement groups |
| **snapshot (VM)** | a saved point-in-time VM state that can be restored instantly |
| **SSH / key pair** | encrypted remote shell / private+public key authentication enabling password-less automation |
| **synchrotron (light source)** | electron storage ring engineered to produce intense X-ray beams for beamlines |
| **systemd / systemctl / journalctl** | the service manager / its control command / its log reader |
| **Tango Controls** | control framework modelling the world as devices registered in a central database |
| **Taurus** | Tango's Python/Qt GUI toolkit for operator interfaces |
| **unit file** | the text file defining a systemd service (what to run, as whom, restart policy) |
| **Vagrant** | tool that creates and manages VM fleets from a versioned `Vagrantfile` |
| **VM (virtual machine)** | a complete computer implemented in software by a hypervisor |

### Where Part 2 begins

You now have: the vocabulary (Linux, VMs, Git), the domain (beamlines, Tango vs EPICS), the mental model (five pillars, one architecture), and a prepared host with Vagrant, Ansible, and a version-pinned Git workspace at `~/lab/devops-platform`.

**Part 2 — The On-Prem DevOps Platform** starts exactly there: Chapter 1 of Part 2 writes the real five-VM `Vagrantfile` and the first Ansible roles, and by its end you will have GitLab answering at `http://gitlab.lab` on infrastructure that you can destroy and rebuild with two commands.

---

*End of Part 1 of 4.*
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
# Building a Reproducible On-Prem DevOps Platform and a Tango Controls System

## PART 3 of 4 — The Tango Controls System

*Part 3 builds the payload: a distributed control system for a simulated 6–20 MeV electron-LINAC beamline, on the Tango Controls framework — the Tango database, PyTango device servers with physically plausible simulators (magnet power supply, motion, Ce:YAG camera, Faraday cup, vacuum gauge, interlock), Sardana scan orchestration, Taurus operator GUIs, HDB++ archiving wired into Part 2's Grafana, cross-device alarms routed into Part 2's Alertmanager, and a pytest suite that runs in Part 2's CI gate. Everything runs on the `tango` VM (192.168.56.30) that has been waiting since Part 2, Chapter 1 — and everything is code in the `tango-devices` repository, because the discipline does not relax when the subject changes from infrastructure to physics.*

---

## How Part 3 is organised

| Chapter | Builds | Depends on |
|---|---|---|
| **1** | The Tango device model — the mental furniture for everything else | Part 1, Ch. 4 |
| **2** | A running Tango installation: database, `TANGO_HOST`, a verified test device | Part 2, Ch. 1 (the `tango` VM) |
| **3** | Your first PyTango device server: a minimal magnet power supply, registered as code | Ch. 2 |
| **4** | Simulated beamline I: a *ramping* magnet with a real state machine, and motion | Ch. 3 |
| **5** | Simulated beamline II: camera, Faraday cup, vacuum gauge, safety interlock | Ch. 4 |
| **6** | Sardana: the beamline becomes scannable (`ascan` in spock) | Ch. 4–5 |
| **7** | Taurus: a live operator panel and trend plot | Ch. 4–5 |
| **8** | HDB++ archiving → Grafana; cross-device alarms → Alertmanager | Ch. 4–5 + Part 2, Ch. 9 |
| **9** | Testing device servers with `DeviceTestContext`, wired into Part 2's CI | Ch. 4 + Part 2, Ch. 3 |
| **10** | Part 3 review: self-assessment, glossary additions, demo script | — |

Estimated effort: **1.5–2.5 weeks**. The device servers of Chapters 4–5 are where most of the time goes — deliberately, because writing them *is* the domain skill this part exists to demonstrate.

**Two framing rules for Part 3, read once:**

**Simulators are the honest architecture, not a substitute for it.** Every "device" in this part returns computed, physically plausible values instead of talking to electronics. This is exactly how real controls software is developed — against simulators, before beam, because beam time is scarce and hardware is dangerous — and the *entire stack above the simulation boundary is identical* to a hardware deployment: same device classes, same database, same GUIs, same archiver, same tests. Where the lab genuinely diverges from a facility (safety systems above all), the text says so explicitly, and you should too.

**Nothing exists only in a GUI click.** Tango ships graphical admin tools (Jive foremost), and you will meet them — but every device registration, property, polling period, and archiving subscription in this part is done *as a script committed to `tango-devices`*, run by Ansible or CI. A control system whose configuration lives in someone's memory of Jive sessions is precisely the archaeology problem Part 2 was built to eliminate. The GUI tools are for *inspecting*; Git is for *defining*.

**Conventions reminder** (full table in Part 1): **THEORY** then **LAB** then **Checkpoint**; `$` = your normal user; **Path A** = conda-forge, **Path B** = apt/`.deb`; commands run on the `tango` VM unless stated otherwise; do not pass a failed **Checkpoint**.

---

# PART 3 — THE TANGO CONTROLS SYSTEM

---

## Chapter 1 — The Tango device model

Part 1, Chapter 4 introduced Tango from orbit. This chapter lands: the device model in full, precisely enough that every later chapter is "just" an application of it. This is the chapter to over-learn — the model is *the* whiteboard drawing of a Tango interview.

### THEORY 1.1 — Why a "device", and not just a program with an API

Everything in Tango is a **device**. A device is not a physical object; it is a **software object that represents one** — a magnet power supply, a camera, a vacuum gauge — or represents something with no physical body at all: a scan sequencer, an interlock evaluator, the database itself. The genius of the model is its uniformity: *every* device, whatever it represents, presents exactly the same five-part interface, so every client — GUI, archiver, alarm system, physicist's script — can be written once against the model and work against anything.

The five parts, in the depth you'll actually use:

**1. A name**, always three levels: **`domain/family/member`** — e.g. `linac/magnet/q1`, `linac/vacuum/gauge1`, `linac/beam/faradaycup1`. The hierarchy is convention, not enforcement, but good naming is real design: `domain` scopes the installation area (`linac`, `beamline1`), `family` the equipment type, `member` the instance. Facilities publish naming conventions and enforce them in review, because ten years and ten thousand devices later, the namespace *is* the map of the machine.

**2. Attributes** — the device's typed, named data: `current` on a magnet, `pressure` on a gauge, `image` on a camera. Attributes are much richer than variables:

- a **data type and format**: *scalar* (one value), *spectrum* (1-D array — a waveform), or *image* (2-D array — the camera delivers one);
- an **access mode**: read-only, read-write, or write-only;
- **metadata that travels with the value**: unit (`"A"`, `"mbar"`), display format, description — so a generic GUI widget can label itself correctly with no custom code;
- **limits**: `min_value`/`max_value` reject out-of-range *writes* at the server before your code even sees them;
- **alarm thresholds**: `min_alarm`/`max_alarm` (and warning levels) that Tango evaluates automatically on every *read* — crossing one flips the attribute's **quality** and drives the device toward `ALARM` state with zero code from you;
- a **quality flag** on every reading: `VALID`, `INVALID`, `CHANGING`, `ALARM`, `WARNING` — clients get told not just the value but whether to trust it;
- a **timestamp** — every reading is stamped at source, which is what makes archived history meaningful.

**3. Commands** — the device's verbs: `On()`, `Off()`, `Reset()`, `Stop()`, `PermitBeam()`. Commands take at most one (possibly compound) input argument and return at most one output. The design idiom: *state that changes continuously is an attribute; an action you request is a command.*

**4. Properties** — per-device **configuration**, stored in the central database and loaded at device init: which serial port the real hardware sits on, a calibration constant, a PLC's IP address. Properties are how one device *class* serves many differently-wired *instances* — the code is identical, the properties differ. (Change a property, run the device's `Init` command, and it reloads — no redeploy.)

**5. A state**, from a fixed, framework-wide enumeration: `ON`, `OFF`, `MOVING`, `FAULT`, `ALARM`, `STANDBY`, `RUNNING`, `INIT`, `DISABLE`, `UNKNOWN`… plus a free-text **status** string elaborating it. Because the enum is universal, generic tools can do real work with no device-specific knowledge: a synoptic display colours every widget by state; Sardana waits for `MOVING → ON` to know a motion finished; an operator's eye reads a wall of green with one red at a glance. Writing a *correct* state machine — every code path leaves the device in the truthful state — is most of what separates professional device servers from demos, and it is the explicit focus of Chapter 4.

### THEORY 1.2 — The cast of a running Tango system

| Component | Role |
|---|---|
| **Tango Database** (`DataBaseds`) | a device server, backed by MySQL/MariaDB, holding the registry: every device name, its class, its properties, and *which server process exports it*. The phone book of Part 1, Ch. 4.4 — consulted at connection time, then out of the data path. |
| **Device server** | an OS process (Python via PyTango here; C++/Java exist) that hosts one or more device **classes** and exports one or more device **instances**. Started as `<Executable> <instance-name>`; one executable can run as many named instances with different device inventories. |
| **Device class** | the code — attributes, commands, properties, state logic — shared by all instances of a type: one `MagnetPowerSupply` class; instances `q1`, `q2`, `d1`… |
| **Client** | anything holding a `DeviceProxy`: Taurus GUIs, Python scripts, Sardana, HDB++, the alarm system, `tango_admin`. |
| **Events** | the push channel (§1.3). |
| **Admin tools** | `Jive` (GUI database browser — the *inspection* tool), `tango_admin` (its scriptable CLI), `Astor` (fleet supervisor, via per-host `Starter` servers — noted, not used in this lab). |

Under the hood, requests travel over CORBA and events over ZeroMQ. You will *never* touch either directly — PyTango wraps them completely — but knowing the two-protocol fact explains otherwise-mysterious things later (like why events need their own ports open, and why `giop:tcp` appears in the database server's arguments).

### THEORY 1.3 — Polling vs events: how data actually moves

A client can always **synchronously read** an attribute — `proxy.pressure` — a request/response round trip. That model collapses at scale: a GUI with 200 fields polling at 10 Hz is 2,000 requests/second of mostly-unchanged values, multiplied by every open GUI, multiplied again by the archiver.

Tango's answer is **events**: the *server* publishes attribute updates over ZeroMQ, and any number of clients **subscribe** — one publisher, N listeners, updates only when something is worth saying. The event types you'll use: **change events** (value moved beyond a configured absolute/relative threshold), **periodic events** (heartbeat at a fixed interval), and **archive events** (a separate threshold tuned for HDB++'s appetite rather than a GUI's).

Who decides when a change event fires? Either your device code pushes explicitly (`self.push_change_event(...)` — full control, used by our simulators), or you let the **Tango polling** mechanism do it: the device server itself can poll its own attributes on a configured period and emit events when thresholds trip. Polling periods and event thresholds are *per-attribute configuration stored in the database* — which means, per this part's framing rule, they are set by a committed script, not by clicking in Jive. Both Taurus (Ch. 7) and HDB++ (Ch. 8) are event subscribers; getting events flowing correctly is the single most common integration hiccup in this Part, so the concept earns this section.

### THEORY 1.4 — A client's-eye tour (the five-line proof of the model)

Before writing any server, fix in your mind what the model gives a *consumer*. Every line below works identically against a simulator and against real electronics:

```python
import tango

q1 = tango.DeviceProxy("linac/magnet/q1")   # name → proxy (DB lookup happens here, once)

q1.state()                # DevState.ON — the universal enum
q1.status()               # "The device is in ON state." — the free-text elaboration
q1.read_attribute("current")   # full reading: value + quality + timestamp + ...
q1.current                # sugar for the value alone
q1.current = 120.0        # attribute write (server validates against min/max first)
q1.Reset()                # command invocation
q1.get_property("calibration_constant")     # per-instance config, from the DB
q1.attribute_query("current").unit          # "A" — metadata travels with the model
```

**Callout — Why this matters for the interview.** The device/attribute/command/property/state model is *the* thing to draw on a whiteboard. Asked "how would you integrate a new piece of hardware?", the shape of the answer is always: *write a device class (attributes for its continuous quantities, commands for its actions, properties for its wiring), register instances in the database, and every existing client — GUIs, archiver, scans, alarms — works against it immediately, unchanged.* No client anywhere needs to know the hardware exists.

**Checkpoint 1 (conceptual):** without notes — the three levels of a device name; the five things every device has; scalar vs spectrum vs image; where properties live and why they exist; polling vs events in two sentences; and why a GUI cannot tell a simulator from real hardware.

---

## Chapter 2 — Installing Tango Controls (both paths)

### THEORY 2.1 — What "installing Tango" actually means

Four things must exist before any custom device runs, and naming them prevents the usual blur:

1. **MariaDB/MySQL** with a `tango` schema — the storage under the registry;
2. **`DataBaseds`** — the database *device server* (yes, the phone book is itself a device) — running and reachable on a known port, conventionally **10000**;
3. **`TANGO_HOST`** — the one environment variable every Tango process and client reads to find the database: `host:port`. It is Tango's equivalent of DNS bootstrap;
4. the **libraries and tools** — PyTango, `tango_admin`, `TangoTest`, Jive — on whichever path you choose.

Both install paths land on the `tango` VM, and both are delivered the Part 2 way: as a `tango` Ansible role added to `site.yml` (the `tango` group has been sitting in the inventory since Part 2, Lab 1.2, waiting for this moment). The lab shows the raw commands first — you must see what the role automates — then you fold them into the role, using the `creates:`-guard patterns that are second nature by now.

**Path A — conda-forge** is the fast path and the lab's recommendation for the *development* machine: the entire Tango stack (including Sardana, Taurus, and HDB++ later) is maintained on conda-forge, versions are current, and everything lives in one disposable environment. **Path B — native Debian packages** is how a production beamline *host* is actually operated — system packages, systemd units — and it is the path that connects to Part 2's packaging chapters: the `.deb`s you install here are the kind of artifact your own pipeline will produce in Part 4. Doing A for speed and B once for understanding is a legitimate strategy; the text flags the few later points that assume one or the other.

### LAB 2.1 — Path A: conda-forge

On `tango` (`vagrant ssh tango`), with miniforge installed (Part 2, Lab 7.3's installer, if it isn't already):

```bash
conda create -n tango-lab -c conda-forge \
    python=3.11 pytango tango-database tango-test tango-admin mysql pymysql -y
conda activate tango-lab       # add 'conda activate tango-lab' to ~/.bashrc for the lab
```

`tango-database` brings `DataBaseds` and the schema; `tango-test` brings the reference device used to verify the plumbing before you trust any code of your own.

### LAB 2.2 — Path B: native Debian packages

```bash
sudo apt update
sudo apt install -y mysql-server tango-db tango-common tango-test python3-tango
```

The `tango-db` package's installer prompts (via dbconfig) for MySQL credentials and creates the `tango` schema itself — a live example of the maintainer-script machinery from Part 2, Theory 7.2. Record the credentials in your lab secrets file (the gitignored `group_vars` pattern from Part 2, Lab 3.1), never in the playbook.

### LAB 2.3 — Bring up the database server and set `TANGO_HOST`

Path A needs the schema created and the server started by hand (Path B's packages did both — check with `systemctl status tango-db`):

```bash
# Path A only — create the schema:
mysql -u root -e "CREATE DATABASE IF NOT EXISTS tango;"
mysql -u root tango < "$CONDA_PREFIX/share/tango/db/create_db.sql"   # ships in the package

# Path A only — start the database device server on the conventional port:
DataBaseds 2 -ORBendPoint giop:tcp::10000 &
```

Now the variable everything depends on — set it, and *persist* it system-wide so no process can ever miss it:

```bash
export TANGO_HOST=tango.lab:10000
echo 'TANGO_HOST=tango.lab:10000' | sudo tee -a /etc/environment
```

Using the name `tango.lab` (not `localhost`) matters: clients on *other* hosts — a Taurus GUI on your workstation, CI jobs, Grafana — will use the same value, and the `common` role already resolves it everywhere. Open the ports while you're here (the Part 2 discipline — rules live where the service lives; these lines go in the `tango` role):

```bash
sudo ufw allow 10000/tcp       # the database server
sudo ufw allow 10100:10300/tcp # a lab range for device servers + their event channels
```

**Callout — Pitfall (the number-one Tango bug).** The single most common "nothing works" in all of Tango is a missing or wrong `TANGO_HOST` — a server registered against one database while the client asks another, or an unset variable falling back to nothing. Symptoms: "device not defined in database", "connection failed", tools hanging on startup. *Check `echo $TANGO_HOST` first, before anything else, every time.* Write that in your runbook now; it will earn its keep.

### LAB 2.4 — Verify with `TangoTest`, then meet the tools

`TangoTest` is the framework's reference device — dozens of attributes of every type, exercising every mechanism. Register an instance *as a script* (your first taste of registration-as-code; Chapter 3 explains each line) and run it:

```bash
python3 - <<'EOF'
import tango
db = tango.Database()
info = tango.DbDevInfo()
info.name   = "test/tangotest/1"
info._class = "TangoTest"
info.server = "TangoTest/lab"
db.add_device(info)
EOF

TangoTest lab &
tango_admin --check-device test/tangotest/1 && echo "PLUMBING OK"
```

Exercise it from Python — this is Theory 1.4 made live:

```bash
python3 - <<'EOF'
import tango
t = tango.DeviceProxy("test/tangotest/1")
print(t.state())                    # RUNNING
print(t.double_scalar)              # a random-walking scalar
print(t.wave[:5])                   # a spectrum attribute — first 5 points
t.double_scalar_w = 42.0            # a write
print(t.read_attribute("double_scalar_w").value)
EOF
```

If you have a graphical session available (or X-forwarding: `vagrant ssh tango -- -X`), open **Jive** once (`jive &`), browse the Device tree to your two registrations, and look at the Server and Property tabs — fixing the mental map between the database's contents and the tools' views. Then close it; per the framing rule, Jive is for looking.

Finally, fold Labs 2.1–2.4 into `ansible/roles/tango/tasks/main.yml` (packages, schema, a systemd unit for `DataBaseds` on Path A, `/etc/environment` line via `lineinfile`, ufw rules), enable `tango` for the `tango` group in `site.yml`, commit, and prove it: `vagrant destroy tango -f && vagrant up tango` must end with `tango_admin --check-device test/tangotest/1` succeeding. *The control system is now infrastructure-as-code too.*

**Checkpoint 2:** `DataBaseds` runs (as a service), `TANGO_HOST=tango.lab:10000` is persisted, `tango_admin --check-device test/tangotest/1` reports success, you have read and written `TangoTest` attributes from Python — and the whole state is reproducible from Git.

---
## Chapter 3 — Your first PyTango device server

### THEORY 3.1 — The anatomy of a PyTango device class

PyTango's high-level API makes a device class a normal Python class with declarative members. The full anatomy, so the lab's code reads as prose:

- **Inherit `tango.server.Device`.** The base class carries the whole framework contract: state, status, the `Init` command, event machinery.
- **`device_property(...)`** class members declare the properties (§1.1, item 4) the database will inject at init.
- **`attribute(...)`** declares attributes, either as class-level members with paired `read_<name>`/`write_<name>` methods, or via a decorator style (`@attribute` on a getter, `@<name>.write` on a setter). Both are idiomatic; this book uses the *method* style in Chapter 3's minimal server (fewest moving parts) and shows the *decorator* style in Chapter 4 (reads better once logic grows).
- **`@command`** decorates methods that become commands; `dtype_in`/`dtype_out` type them.
- **`init_device(self)`** is the constructor-equivalent: called at startup *and* whenever a client invokes the built-in `Init` command. Always call `super().init_device()` first (that's what loads your properties), then set initial internal state, then declare the device's Tango state honestly.
- **`set_state(...)` / `set_status(...)`** are how your code tells the framework — and thereby every client — the truth about the device.
- **`green_mode`** selects the concurrency model. `GreenMode.Asyncio` lets the device use `async`/`await` internally — the natural way to write a background ramp (Chapter 4) without threads. It costs nothing in Chapter 3 and buys Chapter 4, so we set it from the start.
- **`run_server()`** in `__main__` turns the file into a device-server executable: `python3 magnet_ps.py <instance>`.

### THEORY 3.2 — Pogo, and why this book scaffolds by hand

**Pogo** is Tango's official code generator: describe a class's attributes/commands/properties/states in its GUI, and it emits a scaffolded class with the boilerplate wired and documentation stubs in place. In a team it has a second, arguably larger value: the Pogo model file *is* the reviewed interface definition, kept in Git, from which regeneration is repeatable.

This book nonetheless writes classes by hand, for two honest reasons: a headless lab VM makes GUI round-trips awkward, and — more importantly — hand-writing a first device teaches the model in a way accepting generated code never does. Know what Pogo is, be ready to use it in a team, and be able to say exactly that.

### LAB 3.1 — The minimal magnet power supply

Work inside your `tango-devices` clone — this code is the repository's payload from now on:

```
tango-devices/
└── src/
    └── beamline_devices/
        ├── __init__.py
        └── magnet_ps.py
```

```python
# src/beamline_devices/magnet_ps.py
from tango import AttrWriteType, DevState, GreenMode
from tango.server import Device, attribute, command, device_property


class MagnetPowerSupply(Device):
    """A LINAC magnet power supply (simulated). Chapter 3: minimal version."""

    green_mode = GreenMode.Asyncio                      # buys us async ramping in Ch. 4

    calibration_constant = device_property(             # per-instance config from the DB
        dtype=float, default_value=1.0,
        doc="Amps-to-field calibration for this magnet",
    )

    current = attribute(
        dtype=float,
        access=AttrWriteType.READ_WRITE,
        unit="A", format="%6.2f",
        min_value=0.0, max_value=200.0,                 # writes outside → rejected by Tango
        min_alarm=-1.0, max_alarm=190.0,                # reads beyond → ALARM, automatically
        doc="Output current",
    )

    def init_device(self):
        super().init_device()                           # ← loads calibration_constant
        self._current = 0.0
        self.set_state(DevState.ON)
        self.set_status("Power supply ready (simulated).")

    def read_current(self):
        return self._current

    def write_current(self, value):
        self._current = value

    @command
    def Reset(self):
        """Return to a safe, known state."""
        self._current = 0.0
        self.set_state(DevState.ON)
        self.set_status("Reset to 0 A.")


if __name__ == "__main__":
    MagnetPowerSupply.run_server()
```

Read it against Theory 3.1 until every line maps to a bullet. Note what you did *not* write: no networking, no serialization, no state-query handler, no `Init` command — the base class and the declarations carry all of it.

### LAB 3.2 — Registration as code

A device instance exists when the database says so. The registration script is a first-class, committed artifact:

```python
# src/beamline_devices/register_devices.py
"""Idempotently register the beamline's device inventory in the Tango DB."""
import tango

INVENTORY = [
    # (device name,            class,               server instance)
    ("linac/magnet/q1",        "MagnetPowerSupply", "MagnetPowerSupply/lab"),
]

db = tango.Database()
for name, klass, server in INVENTORY:
    try:
        db.get_device_info(name)
        print(f"exists : {name}")
    except tango.DevFailed:
        info = tango.DbDevInfo()
        info.name, info._class, info.server = name, klass, server
        db.add_device(info)
        print(f"created: {name}")
```

Two things to notice. The **server field** `MagnetPowerSupply/lab` binds this device to *instance `lab`* of the executable — start `python3 magnet_ps.py lab` and the framework asks the database "which devices belong to `MagnetPowerSupply/lab`?" and exports exactly those. And the **try/except makes it idempotent** — the Part 2 reflex, applied to a new kind of state. Every future device in Chapters 4–5 is one line added to `INVENTORY`.

**Callout — Why script registration instead of clicking in Jive?** Because the whole point of Part 2's discipline is that nothing exists only in someone's memory or a GUI session. This script lives in Git next to the device it registers, runs under Ansible or CI on environment build, and makes the device inventory reviewable, diffable, and reproducible. A beamline whose device list can only be reconstructed by remembering Jive clicks is the archaeology problem again, wearing a lab coat.

### LAB 3.3 — Run it, poke it, break it (deliberately)

```bash
cd ~/tango-devices/src/beamline_devices
python3 register_devices.py                 # created: linac/magnet/q1
python3 magnet_ps.py lab &                  # the server exports its inventory
tango_admin --check-device linac/magnet/q1  # reachable
```

Now the client's-eye tour against *your own* device:

```bash
python3 - <<'EOF'
import tango
q1 = tango.DeviceProxy("linac/magnet/q1")

print(q1.state(), "-", q1.status())
print("unit:", q1.attribute_query("current").unit)      # metadata: "A"

q1.current = 120.0                                       # a legal write
print("current:", q1.current)

try:
    q1.current = 500.0                                   # illegal: max_value=200
except tango.DevFailed as e:
    print("rejected by the FRAMEWORK, not our code:", e.args[0].reason)

q1.current = 195.0                                       # legal write, but > max_alarm=190
q1.read_attribute("current")                             # a read makes Tango evaluate alarms
print("state after alarm-range read:", q1.state())       # ALARM — again, zero code of ours

q1.Reset()
print("after Reset:", q1.state(), q1.current)
EOF
```

Three behaviours in that transcript came free from attribute metadata — write rejection, alarm evaluation, unit reporting. Declarative metadata doing real work is a theme worth noticing: it is the same idea as Part 2's declarative manifests, one level down.

**Checkpoint 3:** `linac/magnet/q1` is registered *by a committed script*, running, and reachable; you can read/write `current` from a plain `DeviceProxy`; an out-of-range write is rejected and an in-alarm-range value drives the device to `ALARM` — and you can say which of those behaviours you wrote (none). Commit everything.

---

## Chapter 4 — The simulated beamline, part 1: a magnet that ramps, and motion

### THEORY 4.1 — What makes a simulator credible

Chapter 3's magnet stores a number. Real magnet supplies do not teleport: current slews at a finite **ramp rate**, the supply reports a *busy* condition while slewing, and hardware protection refuses abusive requests. A simulator earns the phrase *credible domain grounding* when it reproduces exactly those behaviours — because they are the behaviours every layer above depends on:

- Sardana (Ch. 6) decides a scan step is complete by watching `MOVING → ON`;
- Taurus (Ch. 7) trend plots are only interesting if values *evolve*;
- HDB++ (Ch. 8) archives are only meaningful if there is a time series to record;
- the tests (Ch. 9) assert ramp physics, not variable assignment.

The pattern to internalise — it recurs in every simulated device in this book and in most real ones — is **setpoint/readback with a background process**: a *writable* `setpoint`, a *read-only* `current` that relaxes toward it at a bounded rate, and a **state machine telling the truth throughout** (`ON` at rest, `MOVING` while ramping, `ALARM`/`FAULT` when protection logic says so). The write handler must return *immediately* — a client writing a setpoint must not block for the seconds the ramp takes — which is why the ramp runs as a background `asyncio` task, and why Chapter 3 set `GreenMode.Asyncio` in advance.

One more ingredient: **events**. Our devices push change events explicitly on every simulated tick (`push_change_event`), so subscribers — Taurus plots, HDB++ — receive the ramp as it happens rather than polling for it. Declaring `self.set_change_event("current", True, False)` in `init_device` tells the framework "this device pushes its own events; don't require polling."

### LAB 4.1 — The ramping magnet (full listing)

Replace `magnet_ps.py` — this is the complete Chapter 4 version, decorator style, annotated:

```python
# src/beamline_devices/magnet_ps.py
import asyncio

from tango import AttrWriteType, DevState, GreenMode
from tango.server import Device, attribute, command, device_property


class MagnetPowerSupply(Device):
    """A LINAC magnet power supply, simulated with finite ramp rate."""

    green_mode = GreenMode.Asyncio

    calibration_constant = device_property(dtype=float, default_value=1.0)
    ramp_rate = device_property(                       # per-instance physics, from the DB:
        dtype=float, default_value=5.0,                # a big dipole ramps slower than a
        doc="Slew rate in A/s",                        # corrector — same class, different property
    )

    TICK = 0.2  # seconds per simulation step

    # ---- attributes ------------------------------------------------------
    current = attribute(
        dtype=float, access=AttrWriteType.READ,        # READBACK: clients cannot write it
        unit="A", format="%6.2f",
        min_alarm=-1.0, max_alarm=190.0,
    )

    setpoint = attribute(
        dtype=float, access=AttrWriteType.READ_WRITE,  # the DEMAND
        unit="A", min_value=0.0, max_value=200.0,
    )

    # ---- lifecycle -------------------------------------------------------
    def init_device(self):
        super().init_device()
        self._current = 0.0
        self._setpoint = 0.0
        self._ramp_task = None
        self.set_change_event("current", True, False)  # "I push my own events"
        self.set_state(DevState.ON)
        self.set_status("Ready (simulated).")

    # ---- readback --------------------------------------------------------
    def read_current(self):
        return self._current

    # ---- demand ----------------------------------------------------------
    def read_setpoint(self):
        return self._setpoint

    def write_setpoint(self, value):
        """Accept the demand and return IMMEDIATELY; the ramp runs in background."""
        self._setpoint = value
        if self._ramp_task is None or self._ramp_task.done():
            self._ramp_task = asyncio.ensure_future(self._ramp())

    # ---- the physics -----------------------------------------------------
    async def _ramp(self):
        self.set_state(DevState.MOVING)
        self.set_status(f"Ramping to {self._setpoint:.2f} A "
                        f"at {self.ramp_rate:.1f} A/s.")
        step = self.ramp_rate * self.TICK
        while abs(self._current - self._setpoint) > step:
            self._current += step if self._setpoint > self._current else -step
            self.push_change_event("current", self._current)   # subscribers see the ramp
            await asyncio.sleep(self.TICK)
        self._current = self._setpoint                          # settle exactly
        self.push_change_event("current", self._current)
        self.set_state(DevState.ON)
        self.set_status("At setpoint.")

    # ---- commands ----------------------------------------------------------
    @command
    def Reset(self):
        if self._ramp_task and not self._ramp_task.done():
            self._ramp_task.cancel()
        self._setpoint = 0.0
        self._current = 0.0
        self.push_change_event("current", self._current)
        self.set_state(DevState.ON)
        self.set_status("Reset to 0 A.")


if __name__ == "__main__":
    MagnetPowerSupply.run_server()
```

Walk the design decisions, because they are the interview content: **setpoint and current are separate attributes** with different access — the universal controls idiom (demand vs readback), and the reason an operator screen can show both "what I asked for" and "what I'm getting". **The write returns immediately**; the coroutine owns the slow physics. **`ramp_rate` is a property**, not a constant — the same class will serve fast correctors and slow dipoles, configured per instance in the database. **The state machine never lies**: any client polling `state()` during a ramp sees `MOVING`, and Sardana's scan logic in Chapter 6 is built on exactly that promise. **Events are pushed on every tick**, so Chapter 7's plot and Chapter 8's archiver get the ramp for free.

### LAB 4.2 — Watch the ramp

Restart the server (`kill %1; python3 magnet_ps.py lab &`) and observe from a second shell:

```bash
python3 - <<'EOF'
import tango, time
q1 = tango.DeviceProxy("linac/magnet/q1")
q1.Reset()
q1.setpoint = 50.0                         # returns instantly
t0 = time.time()
while q1.state() == tango.DevState.MOVING:
    print(f"{time.time()-t0:5.1f}s  state={q1.state()}  current={q1.current:6.2f} A")
    time.sleep(0.5)
print(f"{time.time()-t0:5.1f}s  state={q1.state()}  current={q1.current:6.2f} A  <- settled")
EOF
```

Expect ~10 s of `MOVING` lines climbing 5 A/s to 50 A, then `ON`. Change the instance's `ramp_rate` *without touching code* — the property mechanism in action:

```bash
python3 - <<'EOF'
import tango
db = tango.Database()
db.put_device_property("linac/magnet/q1", {"ramp_rate": ["20.0"]})
tango.DeviceProxy("linac/magnet/q1").Init()   # reload properties
EOF
# rerun the watch script: the same ramp now takes ~2.5 s
```

### LAB 4.3 — Motion: the second pattern, and the second device class

Motion devices (the actuator that inserts the Faraday cup into the beam, sample stages, slits) are the same setpoint/readback/background-task shape with different physics — position and velocity instead of current and ramp rate — plus one thing magnets don't need: a **`Stop()` command that matters**, because real motion must be abortable mid-move.

```python
# src/beamline_devices/motor_sim.py
import asyncio
from tango import AttrWriteType, DevState, GreenMode
from tango.server import Device, attribute, command, device_property


class MotorSim(Device):
    """A linear actuator (simulated): position slews at constant velocity."""

    green_mode = GreenMode.Asyncio
    velocity = device_property(dtype=float, default_value=10.0, doc="mm/s")
    TICK = 0.1

    position = attribute(dtype=float, access=AttrWriteType.READ_WRITE,
                         unit="mm", min_value=0.0, max_value=100.0)

    def init_device(self):
        super().init_device()
        self._position = 0.0
        self._target = 0.0
        self._task = None
        self.set_change_event("position", True, False)
        self.set_state(DevState.ON)

    def read_position(self):
        return self._position

    def write_position(self, value):
        self._target = value
        if self._task is None or self._task.done():
            self._task = asyncio.ensure_future(self._move())

    async def _move(self):
        self.set_state(DevState.MOVING)
        step = self.velocity * self.TICK
        while abs(self._position - self._target) > step:
            self._position += step if self._target > self._position else -step
            self.push_change_event("position", self._position)
            await asyncio.sleep(self.TICK)
        self._position = self._target
        self.push_change_event("position", self._position)
        self.set_state(DevState.ON)

    @command
    def Stop(self):
        """Abort motion NOW: the target becomes wherever we are."""
        self._target = self._position
        self.set_state(DevState.ON)
        self.set_status("Stopped by operator.")


if __name__ == "__main__":
    MotorSim.run_server()
```

Add to the inventory and bring it up:

```python
# register_devices.py — INVENTORY grows:
    ("linac/motion/fc-actuator", "MotorSim", "MotorSim/lab"),
```

```bash
python3 register_devices.py && python3 motor_sim.py lab &
```

Exercise the one new behaviour: start a long move (`position = 90`), and while `MOVING`, call `Stop()` — the device settles instantly, `ON`, at wherever it was. That is the semantics every scan system assumes an abort has.

**Callout — Pitfall (keep state machines local).** Do not centralise the ramp/move logic of several devices into one shared "simulation engine" with a big `if/elif` over device types. Each device class owns its own state machine and its own background task. Sardana's orchestration (Ch. 6) relies on each device *independently* and truthfully reporting `MOVING` vs `ON`; a shared tangled state machine breaks that independence, and — worse — it is not how you would structure real hardware drivers, so it would teach the wrong architecture.

**Checkpoint 4:** writing `setpoint` shows `ON → MOVING → ON` with `current` visibly ramping (not jumping) when polled mid-transition; `ramp_rate` is changed via a database property + `Init()`, no code edit; the motor moves at its property-configured velocity and `Stop()` aborts a move cleanly. Both devices registered by the script, everything committed.

---
## Chapter 5 — The simulated beamline, part 2: camera, Faraday cup, vacuum, interlock

Four more devices complete the beamline. Each introduces one new concept: **image attributes** (camera), **cross-device correlation** (Faraday cup), **automatic alarming on a log-scale quantity** (vacuum), and **a device that gates other devices** (interlock). By the end of this chapter the device inventory matches the beamline the CV describes.

### THEORY 5.1 — The diagnostic chain, physically

Thirty seconds of beam physics so the simulators mean something. A LINAC's electron beam is invisible; **diagnostics** make it observable. A **Ce:YAG screen** is a scintillating crystal moved *into* the beam path: electrons strike it, it glows, a camera photographs the glow — the image is a direct map of the beam's transverse profile (position and size). A **Faraday cup** is a metal block that swallows the beam entirely and measures the collected charge — the definitive beam-current measurement, but destructive (no beam continues past it), which is why it sits on the Chapter 4 actuator and is inserted only when measuring. **Vacuum** is not a diagnostic but a life-support system: the beam pipe must stay near 10⁻⁸–10⁻⁹ mbar, because beam colliding with residual gas is lost beam and radiation; pressure excursions are the classic **interlock** condition — vacuum bad ⇒ no beam permitted.

Notice what the simulators must therefore do: the camera must produce a *plausible spot* (a downstream image widget will display it as real); the Faraday cup's reading should *correlate with the magnet* (a scan that moves Q1 and reads the cup must show structure, or Chapter 6's scans are meaningless); the vacuum gauge must produce *log-normal noise around a good base pressure* with a way to force it bad; and the interlock must *actually consult* the gauge.

### LAB 5.1 — Ce:YAG beam-profile camera (an image attribute)

```python
# src/beamline_devices/ceyag_camera.py
import numpy as np
from tango import AttrWriteType, DevState, GreenMode
from tango.server import Device, attribute


class CeYAGCamera(Device):
    """Beam-profile camera on a Ce:YAG screen (simulated Gaussian spot)."""

    green_mode = GreenMode.Asyncio
    WIDTH, HEIGHT = 640, 480

    image = attribute(
        dtype=((float,),),                       # 2-D → an IMAGE attribute
        max_dim_x=WIDTH, max_dim_y=HEIGHT,
        doc="Beam profile, arbitrary intensity units",
    )

    exposure = attribute(dtype=float, access=AttrWriteType.READ_WRITE,
                         unit="ms", min_value=0.1, max_value=1000.0)

    def init_device(self):
        super().init_device()
        self._exposure = 10.0
        self.set_state(DevState.ON)

    def read_exposure(self):
        return self._exposure

    def write_exposure(self, value):
        self._exposure = value

    def read_image(self):
        """A Gaussian spot + noise, scaled by exposure — freshly computed per read."""
        y, x = np.mgrid[0:self.HEIGHT, 0:self.WIDTH]
        cx, cy = self.WIDTH / 2, self.HEIGHT / 2
        spot = 4000 * np.exp(-(((x - cx) ** 2) / 2000 + ((y - cy) ** 2) / 1500))
        noise = np.random.normal(0, 50, (self.HEIGHT, self.WIDTH))
        return (spot + noise) * (self._exposure / 10.0)


if __name__ == "__main__":
    CeYAGCamera.run_server()
```

The new concept is the **dtype**: `((float,),)` — a tuple-of-tuples — declares a 2-D *image* attribute (one nesting level, `(float,)`, would be a 1-D *spectrum*). Everything else is the model you know. A worthwhile extension once the basics run: make the spot's centre drift with `linac/magnet/q1`'s current (a `DeviceProxy` in `init_device`, offset ∝ current) — then Chapter 7's image widget visibly shows *steering*, which is a lovely demo.

### LAB 5.2 — Faraday cup, correlated with the machine

```python
# src/beamline_devices/faraday_cup.py
import numpy as np
import tango
from tango import DevState, GreenMode
from tango.server import Device, attribute, device_property


class FaradayCup(Device):
    """Beam-charge monitor (simulated). Reads more charge when the upstream
    quadrupole is near its focusing optimum — so scans show structure."""

    green_mode = GreenMode.Asyncio
    magnet_device = device_property(dtype=str, default_value="linac/magnet/q1")
    optimum_current = device_property(dtype=float, default_value=35.0)

    charge = attribute(dtype=float, unit="pC", format="%7.2f")

    def init_device(self):
        super().init_device()
        self._magnet = tango.DeviceProxy(self.magnet_device)   # cross-device wiring — via a PROPERTY
        self.set_state(DevState.ON)

    def read_charge(self):
        i = self._magnet.current
        focusing = np.exp(-((i - self.optimum_current) ** 2) / (2 * 12.0 ** 2))
        return max(0.0, 120.0 * focusing + np.random.normal(0.0, 3.0))


if __name__ == "__main__":
    FaradayCup.run_server()
```

Two teaching points. The cup holds a **proxy to another device** — devices are clients too, and composing them is normal architecture. And *which* device it watches is a **property**, not a hardcoded name: re-pointing the cup at a different magnet is a database edit, not a code change. The Gaussian response around `optimum_current=35 A` is the payoff for Chapter 6: an `ascan` of Q1 from 0→70 A will trace a visible peak.

### LAB 5.3 — Vacuum gauge: alarms with no alarm code

```python
# src/beamline_devices/vacuum_gauge.py
import numpy as np
from tango import DevState, GreenMode
from tango.server import Device, attribute, command


class VacuumGauge(Device):
    """Cold-cathode gauge (simulated): log-normal noise around base pressure,
    with a command to simulate a leak for interlock/alarm testing."""

    green_mode = GreenMode.Asyncio

    pressure = attribute(
        dtype=float, unit="mbar", format="%9.2e",
        max_warning=1e-7,                       #   > 1e-7 → quality WARNING
        max_alarm=1e-6,                         #   > 1e-6 → quality ALARM, device ALARM
    )

    def init_device(self):
        super().init_device()
        self._base_exponent = -8.0              # ~1e-8 mbar: healthy UHV
        self.set_state(DevState.ON)

    def read_pressure(self):
        return 10 ** np.random.normal(self._base_exponent, 0.15)

    @command
    def SimulateLeak(self):
        """Test hook: degrade vacuum past the alarm threshold."""
        self._base_exponent = -5.5              # ~3e-6 mbar: bad
        self.set_status("SIMULATED LEAK ACTIVE")

    @command
    def RepairLeak(self):
        self._base_exponent = -8.0
        self.set_state(DevState.ON)
        self.set_status("Vacuum recovered.")


if __name__ == "__main__":
    VacuumGauge.run_server()
```

The alarm behaviour costs zero lines: `max_warning`/`max_alarm` in the attribute declaration, and Tango evaluates every read against them — quality flags flip, the device drops to `ALARM`, and (crucially for Chapter 8) the transition is visible to every subscriber. The `SimulateLeak` command is a **deliberate test hook** — simulators *should* carry fault-injection commands, because testing the unhappy path is most of what simulators are for. Real gauges obviously lack this command; that asymmetry is fine and worth saying out loud.

### LAB 5.4 — The interlock: a device that gates devices

```python
# src/beamline_devices/interlock.py
import tango
from tango import DevState, GreenMode
from tango.server import Device, attribute, command, device_property


class Interlock(Device):
    """Beam-permit logic (simulated, software-level — see the honesty callout).
    Beam is permitted only if vacuum is good AND the Faraday-cup actuator
    is not blocking a configured 'must be clear' condition."""

    green_mode = GreenMode.Asyncio
    gauge_device = device_property(dtype=str, default_value="linac/vacuum/gauge1")
    pressure_limit = device_property(dtype=float, default_value=1e-6)

    beam_permitted = attribute(dtype=bool)

    def init_device(self):
        super().init_device()
        self._gauge = tango.DeviceProxy(self.gauge_device)
        self.set_state(DevState.ON)

    def read_beam_permitted(self):
        return self._evaluate()

    @command(dtype_out=bool)
    def PermitBeam(self):
        """Explicit permit request — the command a 'gun' device would call."""
        return self._evaluate()

    def _evaluate(self):
        ok = self._gauge.pressure <= self.pressure_limit
        if ok:
            self.set_state(DevState.ON)
            self.set_status("Beam permitted: all conditions satisfied.")
        else:
            self.set_state(DevState.ALARM)
            self.set_status(f"Beam INHIBITED: pressure above "
                            f"{self.pressure_limit:.1e} mbar.")
        return ok


if __name__ == "__main__":
    Interlock.run_server()
```

**Callout — Why, and an honesty boundary that matters.** This is deliberately a *small, honest* interlock: a device that consults others and gates a permit — the *pattern*, demonstrated. It is **not** a safety system, and you must never present it as one. Real machine-protection and personnel-safety interlocks are implemented in hardware — hardwired logic and safety-rated PLCs (the Siemens SIS world), independent of the control-system software, precisely so that a crashed device server cannot un-protect a machine. The software layer *mirrors and reports* interlock state; it does not constitute it. Being crisp about that boundary — software supervision above, certified hardware protection below — demonstrates real accelerator-safety literacy far better than an overclaimed demo would.

### LAB 5.5 — Full inventory, one process tree, and the smoke test

Grow the inventory to its final shape and register:

```python
# register_devices.py — final INVENTORY:
INVENTORY = [
    ("linac/magnet/q1",          "MagnetPowerSupply", "MagnetPowerSupply/lab"),
    ("linac/motion/fc-actuator", "MotorSim",          "MotorSim/lab"),
    ("linac/camera/ceyag1",      "CeYAGCamera",       "CeYAGCamera/lab"),
    ("linac/beam/faradaycup1",   "FaradayCup",        "FaradayCup/lab"),
    ("linac/vacuum/gauge1",      "VacuumGauge",       "VacuumGauge/lab"),
    ("linac/safety/interlock1",  "Interlock",         "Interlock/lab"),
]
```

Start the five servers (a `start_all.sh` in the repo is fine for now; Part 4 replaces it with systemd units shipped in a `.deb` and, for some, Kubernetes deployments — say that sentence to yourself, it is the whole book's plot):

```bash
python3 register_devices.py
for s in magnet_ps motor_sim ceyag_camera faraday_cup vacuum_gauge interlock; do
    python3 $s.py lab & 
done
```

Now the integration smoke test — the beamline behaving *as a system*:

```bash
python3 - <<'EOF'
import tango
gauge = tango.DeviceProxy("linac/vacuum/gauge1")
ilk   = tango.DeviceProxy("linac/safety/interlock1")

print("healthy vacuum :", f"{gauge.pressure:.2e} mbar -> permit:", ilk.PermitBeam())

gauge.SimulateLeak()
print("after leak     :", f"{gauge.pressure:.2e} mbar -> permit:", ilk.PermitBeam())
print("interlock state:", ilk.state())         # ALARM
print("gauge state    :", gauge.state())        # ALARM (from max_alarm, automatically)

gauge.RepairLeak()
print("after repair   :", f"{gauge.pressure:.2e} mbar -> permit:", ilk.PermitBeam())
EOF
```

**Checkpoint 5:** six devices registered by the script and running; the camera serves a 640×480 image; the Faraday-cup reading visibly peaks as you ramp Q1 through ~35 A (try it: three setpoints, three reads); `SimulateLeak()` drives *both* the gauge (automatic, threshold) and the interlock (explicit, logic) into `ALARM` and flips the permit to `False`; `RepairLeak()` restores everything. All committed.

---

## Chapter 6 — Sardana: the beamline becomes scannable

### THEORY 6.1 — What Sardana adds on top of raw devices

Individual devices are necessary but not sufficient. What a beamline scientist actually does, dozens of times per shift, is run **scans**: *"step Q1's setpoint from 0 to 70 A in 14 steps; at each step, wait for the ramp to settle, then integrate the Faraday cup for half a second; record everything."* You could script that with `DeviceProxy` calls — and understanding that you could is important — but every beamline would then reinvent stepping, settling, data recording, abort handling, and scan bookkeeping, badly and differently.

**Sardana** is the standard Tango-ecosystem layer that industrialises it (born at ALBA, used across the European light sources — and a named keyword on the CV). Its architecture is three ideas:

- the **Pool** — a device server that owns *scannable elements*: motors, counters/experimental channels, and measurement groups. Anything that can "go to a position and say when it's done" can be a Pool **motor**; anything that can "produce a number when asked" can be a **counter**. Note the abstraction: a magnet's current is a "position" as far as scanning is concerned;
- **controllers** — small adapter classes you write, mapping the Pool's generic motor/counter interface onto your actual Tango devices. This is the layer where *your* beamline plugs into *standard* machinery;
- the **MacroServer** — a device server that executes **macros**: the scan procedures themselves. Standard macros (`ascan`, `dscan`, `mesh`, `ct`…) cover the classics; custom macros are plain Python. **Spock** is the IPython-based console operators drive it all from.

The payoff of Chapter 4's discipline lands here: Sardana knows a motion is complete because the underlying device reports `MOVING → ON`. Your magnet already speaks that language, so it becomes a scannable axis with a ~15-line adapter and *zero changes to the device itself* — Chapter 1's uniformity argument, paying out.

### LAB 6.1 — Install and create the Sardana instances

```bash
conda install -n tango-lab -c conda-forge sardana itango -y   # Path A (Path B: python3-sardana)

# Register + start a Pool and a MacroServer (device servers like any other):
Pool lab &
MacroServer lab &        # when prompted/configured, attach it to Pool "lab"
```

(First-run specifics vary slightly by version — Sardana will create its own device entries in the Tango DB; confirm with `tango_admin --check-device` on the names it prints. Fold the working sequence into your `tango` role notes.)

### LAB 6.2 — Controllers: the ~15-line adapters

```python
# src/beamline_devices/sardana_ctrl/magnet_motor_ctrl.py
import tango
from sardana import State
from sardana.pool.controller import MotorController


class MagnetMotorController(MotorController):
    """Expose MagnetPowerSupply.setpoint/current as a Sardana motor."""

    MaxDevice = 8
    ctrl_properties = {"device_name": {"Type": str,
                                       "Description": "Tango device to wrap"}}

    def AddDevice(self, axis):
        self._proxy = tango.DeviceProxy(self.device_name)

    def ReadOne(self, axis):
        return self._proxy.current                    # "where am I" = readback

    def StartOne(self, axis, position):
        self._proxy.setpoint = position               # "go there" = write demand

    def StateOne(self, axis):
        s = self._proxy.state()
        if s == tango.DevState.MOVING:
            return State.Moving, "ramping"
        if s == tango.DevState.ALARM:
            return State.Alarm, "device in ALARM"
        return State.On, "at setpoint"

    def AbortOne(self, axis):
        self._proxy.Reset()                           # best available abort
```

```python
# src/beamline_devices/sardana_ctrl/fcup_counter_ctrl.py
import tango
from sardana.pool.controller import CounterTimerController


class FCupCounterController(CounterTimerController):
    """Expose FaradayCup.charge as a Sardana counter."""

    MaxDevice = 1
    ctrl_properties = {"device_name": {"Type": str,
                                       "Description": "Tango device to wrap"}}

    def AddDevice(self, axis):
        self._proxy = tango.DeviceProxy(self.device_name)

    def ReadOne(self, axis):
        return self._proxy.charge
```

**Callout — Pitfall (thin controllers).** Controllers are adapters, nothing more: `ReadOne`/`StartOne`/`StateOne` as pass-throughs. All physics — ramping, settling, limits — lives in the device class (Chapter 4), where it serves *every* client. Duplicating ramp logic in a controller is the classic Sardana over-engineering mistake: it desynchronises two state machines and helps nobody.

### LAB 6.3 — Wire up in spock, and scan

```bash
spock    # Sardana's IPython console; first run asks which MacroServer — pick lab's
```

Inside spock, define the elements against your controllers, then run the canonical scan:

```
Door_lab [1]: defctrl MagnetMotorController q1ctrl device_name linac/magnet/q1
Door_lab [2]: defelem q1mot q1ctrl 1
Door_lab [3]: defctrl FCupCounterController fcctrl device_name linac/beam/faradaycup1
Door_lab [4]: defelem fccnt fcctrl 1
Door_lab [5]: defmeas mg1 fccnt

Door_lab [6]: ascan q1mot 0 70 14 0.5
```

Read `ascan q1mot 0 70 14 0.5` as a sentence: *absolute scan of motor `q1mot` from 0 to 70 in 14 intervals, counting 0.5 s per point.* Watch what happens at each step: Sardana writes the setpoint, your magnet reports `MOVING` while it ramps at its property-configured rate, Sardana waits for `ON`, then reads the Faraday-cup counter, records the row, and steps on. The printed table's count column should trace the Chapter 5 Gaussian — **a peak near 35 A**. You have just performed, against simulators, the exact operation a beamline runs against hardware, through the exact software stack.

Two more macros to try while you're there: `dscan q1mot -10 10 8 0.5` (a *relative* scan around the current position — the everyday optimisation move) and `ct 1.0` (a bare 1-second count). And note for the record: spock definitions (`defctrl`, `defelem`…) write configuration into the Pool — which is Tango-database-backed state, so your environment-build script should replay them; a `sar_demo`-style setup macro or a small itango script committed to the repo keeps this reproducible.

**Checkpoint 6:** `ascan q1mot 0 70 14 0.5` completes in spock; the recorded counts are visibly structured (peaked near the property-configured optimum), proving the scan really moved the magnet and re-read the counter; aborting a scan mid-flight (Ctrl-C in spock) leaves the magnet in a sane state.

---
## Chapter 7 — Taurus: operator GUIs

### THEORY 7.1 — Model-based widgets: the client half of the device model

**Taurus** is the Tango ecosystem's Python/Qt GUI toolkit, and its design is one idea applied relentlessly: **widgets bind to models by name**. A Taurus widget is given a *model* — an attribute name like `linac/magnet/q1/current` — and from that point it labels itself (unit and format from the attribute's metadata), displays the live value, colours itself by quality and device state, and **updates via Tango events** with zero networking code written by you. A form widget given a list of models builds a whole panel; a plot widget given a model draws a live trend; an image widget given the camera's image attribute renders the beam spot.

Stop and notice what is converging. Chapter 1 promised that metadata travels with the model — Taurus is where unit strings, formats, and alarm colours you declared in Chapters 4–5 appear on screen, unasked. Chapter 4 pushed change events on every ramp tick — Taurus is a subscriber, so plots follow ramps in real time. And the uniformity claim — *a GUI cannot tell simulator from hardware* — is about to be demonstrated rather than asserted: the panel you build here would run unmodified in front of real electronics.

The practical prerequisite is a display. Options in comfort order: run the GUI on your **workstation** (install `taurus` + `pytango` locally; `TANGO_HOST=tango.lab:10000` and the lab's `/etc/hosts` line make the devices reachable across the host-only network — a nice proof of "distributed" in itself); X-forward from the VM (`vagrant ssh tango -- -X`); or install a minimal desktop on the VM. The lab assumes the first.

### LAB 7.1 — Zero-code Taurus: the built-in tools

Before writing a panel, meet the two commands that give you a GUI for free:

```bash
# a live form for any device, generated from its interface:
taurus form linac/magnet/q1

# a live trend of any attribute(s):
taurus trend linac/magnet/q1/current linac/vacuum/gauge1/pressure
```

Write a new setpoint in the form and watch the trend draw the ramp. For quick diagnosis at a terminal, the ecosystem's other gift is **itango** (`itango`) — an IPython with Tango conveniences preloaded; `q1 = Device("linac/magnet/q1")` and tab-completion on attributes makes it the fastest inspection tool you'll use.

### LAB 7.2 — The operator panel

Now the committed artifact — a small application file, because real operator screens are versioned code, not ad-hoc commands:

```python
# src/beamline_devices/operator_panel.py
"""LINAC lab operator panel: live values, a trend, and the beam image."""
import sys

from taurus.qt.qtgui.application import TaurusApplication
from taurus.qt.qtgui.panel import TaurusForm
from taurus.qt.qtgui.plot import TaurusPlot
from taurus.qt.qtgui.extra_guiqwt import TaurusImageDialog


def main():
    app = TaurusApplication(sys.argv, cmd_line_parser=None)

    form = TaurusForm()
    form.model = [
        "linac/magnet/q1/setpoint",        # writable — the form renders a write widget
        "linac/magnet/q1/current",         # readback, live via events
        "linac/motion/fc-actuator/position",
        "linac/vacuum/gauge1/pressure",    # colours by quality: WARNING amber, ALARM red
        "linac/beam/faradaycup1/charge",
        "linac/safety/interlock1/beam_permitted",
    ]
    form.setWindowTitle("LINAC lab — operator panel")
    form.show()

    plot = TaurusPlot()
    plot.setModel(["linac/magnet/q1/current"])
    plot.setWindowTitle("Q1 current trend")
    plot.show()

    beam = TaurusImageDialog()
    beam.setModel("linac/camera/ceyag1/image")
    beam.setWindowTitle("Ce:YAG beam profile")
    beam.show()

    sys.exit(app.exec_())


if __name__ == "__main__":
    main()
```

Run it (`python3 operator_panel.py`) and drive the beamline from the screen:

1. Type `50` into the **setpoint** field → the readback field climbs at 5 A/s and the trend plot draws the ramp — those are your Chapter 4 change events arriving;
2. watch the **pressure** field's ordinary jitter, then run `SimulateLeak()` from itango → the field turns alarm-red and `beam_permitted` flips to false — attribute quality and interlock logic, on one screen;
3. the **image window** shows the noisy Gaussian spot; if you built the steering extension from Lab 5.1, ramp Q1 and watch the spot walk.

(Widget module paths occasionally shift between Taurus releases — if an import fails, `taurus form`/`taurus trend` still prove the stack while you check the current module name; that resilience-first habit is worth keeping.)

**Callout — Why this is worth building, not skipping.** "I built the pipeline" (Part 2) and "I built the payload" (Part 3) are both true — but abstract. A **five-minute screen recording** of this panel — writing a setpoint, watching the ramp trend, tripping the simulated leak, watching the alarm colour and the permit flip — is the single most concrete, lowest-effort interview artifact this handbook produces: the entire story, on one screen, in real time. Record it when Checkpoint 7 passes; Part 4's demo script slots it in.

**Checkpoint 7:** the form shows live values for all six models with correct units; writing `setpoint` is visible simultaneously in the form and as a ramp in the trend plot; the vacuum alarm is visible as widget colour; the image window renders the beam spot. Panel file committed.

---

## Chapter 8 — HDB++ archiving and the Tango alarm system

### THEORY 8.1 — Why archiving is a separate concern from live display

Taurus shows *now*. Operators and physicists equally need *history*: what did the vacuum do over the last shift; exactly when did the interlock trip, and what was Q1 doing in the minute before; is this Tuesday's beam current subtly lower than last Tuesday's? At a facility, the archive is also an accountability record — Part 1's "safety-critical and continuous" property has a memory requirement built in.

**HDB++** is the Tango ecosystem's historical archiving system, and its architecture is exactly what Chapters 1 and 4 set up: it is an **event subscriber**. Two cooperating device servers — see the pattern? *everything* is a device — do the work:

- the **ConfigurationManager** (`hdbpp-cm`): the control surface — you tell it which attributes to archive and with what strategy;
- one or more **EventSubscribers** (`hdbpp-es`): the workers — each subscribes to its assigned attributes' *archive events* and writes timestamped values into the **HDB++ schema** in MySQL/MariaDB (per-datatype tables; Cassandra/TimescaleDB backends exist at larger scales).

Decoupling matters and is worth stating in interviews: archiving continues whether or not any human is watching; it scales by adding subscribers; and it is configured *per attribute* — the archiver's event thresholds (archive events) are tuned separately from GUI-oriented change events, because a plot wants smoothness while an archive wants significance.

The lab adds one deliberate twist: rather than standing up a new visualisation tool, we point **Part 2's Grafana at the HDB++ MySQL backend** with a SQL datasource. One dashboard stack for infrastructure metrics *and* machine history — a small integration with an outsized "this person thinks operationally" signal.

### LAB 8.1 — Stand up HDB++ (Path A)

```bash
conda install -n tango-lab -c conda-forge hdbpp-cm hdbpp-es libhdbpp-mysql -y

# a dedicated schema for history:
mysql -u root -e "CREATE DATABASE IF NOT EXISTS hdbpp;"
mysql -u root hdbpp < "$CONDA_PREFIX/share/hdbpp/hdb++_mysql_schema.sql"   # per-type tables

# register the two servers in the Tango DB (extend register_devices.py's INVENTORY):
#   ("archiving/hdbpp/cm-1", "HdbConfigurationManager", "hdbpp-cm/lab"),
#   ("archiving/hdbpp/es-1", "HdbEventSubscriber",      "hdbpp-es/lab"),
python3 register_devices.py

# point them at the schema via device properties (DB credentials via your secrets pattern),
# then start:
hdbpp-cm lab &
hdbpp-es lab &
```

(Exact property names for the DB connection — host, user, password, dbname — ship in the packages' docs; set them with `db.put_device_property(...)` in a committed script, credentials injected from the gitignored secrets file. The mechanism is Chapter 4, Lab 4.2's — you know it.)

### LAB 8.2 — Subscribe attributes, as code

```python
# src/beamline_devices/setup_archiving.py
"""Register and start archiving for the beamline's key attributes."""
import tango

TO_ARCHIVE = [
    "tango://tango.lab:10000/linac/magnet/q1/current",
    "tango://tango.lab:10000/linac/vacuum/gauge1/pressure",
    "tango://tango.lab:10000/linac/beam/faradaycup1/charge",
]

cm = tango.DeviceProxy("archiving/hdbpp/cm-1")
for attr in TO_ARCHIVE:
    cm.command_inout("AttributeAdd", [attr, "1"])     # assign to subscriber 1
    cm.command_inout("AttributeStart", attr)
    print("archiving:", attr)
```

Run it, generate some history (a few ramps; a `SimulateLeak()`/`RepairLeak()` cycle), then verify at the SQL layer — always look at the storage once:

```bash
mysql -u root hdbpp -e "
  SELECT att_conf_id, data_time, value_r
  FROM att_scalar_devdouble_ro ORDER BY data_time DESC LIMIT 8;"
```

Timestamped rows of your ramp: the archive is real.

### LAB 8.3 — History in Grafana

In Part 2's Grafana (`http://192.168.56.20:3000`): add a **MySQL datasource** pointing at `tango.lab:3306`, database `hdbpp`, a read-only user (create one: `GRANT SELECT ON hdbpp.* TO 'grafana'@'%' ...` — least privilege, note it). Then a panel with a query of this shape (join the config table once to find your attribute's `att_conf_id`, then):

```sql
SELECT data_time AS "time", value_r AS "pressure"
FROM att_scalar_devdouble_ro
WHERE att_conf_id = <gauge-id> AND $__timeFilter(data_time)
ORDER BY data_time;
```

Build the small **beam-history dashboard**: Q1 current, vacuum pressure (log scale — right-click the axis), Faraday-cup charge. Ramp the magnet and refresh: infrastructure metrics and machine history, one pane of glass.

### THEORY 8.2 — Two alarm layers, and where notifications should go

Keep the two alarm mechanisms distinct — conflating them is a common muddle:

- **Built-in, per-attribute** (you have it since Chapter 5): `min/max_alarm` thresholds on one attribute flip quality and device state. Local, automatic, zero configuration beyond the declaration.
- **Rule-based, cross-device**: conditions spanning several attributes — *"vacuum is bad **and** beam charge is nonzero"* is qualitatively different information from either fact alone (it means beam is being lost into bad vacuum *right now*). The Tango ecosystem's alarm systems (PyAlarm/Panic historically, and successors) evaluate boolean **formulas** over attribute sets, latch/acknowledge alarm states, and act on transitions:

```python
formula = "(linac/vacuum/gauge1/pressure > 1e-6) and (linac/beam/faradaycup1/charge > 5.0)"
```

Where should a firing alarm *go*? Part 2 already answered: **Alertmanager**. Route the alarm system's notification action to a small webhook receiver (or Alertmanager's API) so control-system alarms enter the same pipeline as infrastructure alerts — same grouping, same silencing, same runbook links. If the packaged alarm-system route fights you on versions, implement the lab's fallback honestly: a 30-line **watchdog device server** (you can write one cold by now) that polls the formula's inputs at 1 Hz and POSTs to Alertmanager on state change — architecturally identical, and you built it.

**Callout — Why one notification pipeline.** "Disk filling on `svc`" and "vacuum interlock tripped" reaching the on-call human through **one** deduplicating, silencing, runbook-linking channel is precisely the consolidation that distinguishes an operated system from two disconnected projects. It is also, concretely, the sentence that ties Part 2 and Part 3 together in an interview.

**Checkpoint 8:** a Grafana panel plots `linac/vacuum/gauge1/pressure` from the HDB++ backend over the last 10 minutes of lab activity; ramps and leak events appear in the archive; and the cross-device alarm (packaged system or watchdog) fires into Part 2's Alertmanager when — and only when — the leak is simulated *while* charge is present.

---

## Chapter 9 — Testing device servers with `DeviceTestContext`

### THEORY 9.1 — The missing piece: tests that need no infrastructure

Part 2, Chapter 3 built the quality gate: nothing merges red. But everything Part 3 has run so far needed a live Tango database — useless inside a clean CI container. PyTango's **`DeviceTestContext`** closes the gap: it launches a device class **in-process**, with a throwaway in-memory substitute for the database machinery, hands you a working `DeviceProxy`, and tears everything down afterwards — a real device server, testable inside a plain `pytest` run, no `DataBaseds`, no `TANGO_HOST`.

That makes three test layers, and naming them is itself good engineering hygiene: **unit tests** of pure logic (no Tango at all — e.g. a ramp-step calculation, if factored out); **device tests** with `DeviceTestContext` (the device's contract: states, limits, alarm behaviour, physics) — this chapter's subject and CI's staple; **integration tests** against the live lab (Sardana scans, archiver rows) — valuable, but run on the lab host, not in the merge gate.

### LAB 9.1 — The magnet's device tests

```python
# tests/test_magnet_ps.py
import time

import pytest
from tango import DevState
from tango.test_context import DeviceTestContext

from beamline_devices.magnet_ps import MagnetPowerSupply


@pytest.fixture
def magnet():
    """A fresh in-process MagnetPowerSupply per test."""
    with DeviceTestContext(MagnetPowerSupply,
                           properties={"ramp_rate": 50.0},   # fast ramps: fast tests
                           process=True) as proxy:
        yield proxy


def test_initial_state(magnet):
    assert magnet.state() == DevState.ON
    assert magnet.current == 0.0


def test_write_out_of_range_is_rejected(magnet):
    with pytest.raises(Exception):
        magnet.setpoint = 500.0          # max_value=200 — the framework must refuse
    assert magnet.setpoint == 0.0


def test_ramp_reaches_setpoint_and_states_are_truthful(magnet):
    magnet.setpoint = 40.0
    assert magnet.state() == DevState.MOVING          # immediately after write: ramping
    deadline = time.time() + 5.0
    while magnet.state() == DevState.MOVING and time.time() < deadline:
        time.sleep(0.05)
    assert magnet.state() == DevState.ON
    assert magnet.current == pytest.approx(40.0, abs=0.5)


def test_alarm_threshold(magnet):
    magnet.setpoint = 195.0                            # legal write, above max_alarm=190
    deadline = time.time() + 6.0
    while magnet.state() == DevState.MOVING and time.time() < deadline:
        time.sleep(0.05)
    magnet.read_attribute("current")                   # a read triggers alarm evaluation
    assert magnet.state() == DevState.ALARM


def test_reset_from_anywhere(magnet):
    magnet.setpoint = 30.0
    magnet.Reset()
    assert magnet.state() == DevState.ON
    assert magnet.current == 0.0
```

Points of craft worth absorbing: the fixture injects a **fast `ramp_rate` via properties** — the test controls the physics, so the suite runs in seconds; the ramp test **polls state with a deadline** rather than sleeping a fixed guess — less flaky, and it *asserts the state machine's honesty*, not just the endpoint; and the out-of-range test pins down behaviour you didn't write (framework rejection) — exactly the kind of contract a refactor could silently break.

```bash
pip install -e . && pytest tests/ -v      # all green locally before it goes near CI
```

**Callout — Pitfall (wall-clock time in tests, and how to say so).** Even with deadline-polling, these tests spend real seconds waiting for simulated physics — tolerable at lab scale, a smell at suite scale (slow CI, occasional flakes on loaded runners). The known fix: make time injectable — the ramp coroutine takes a sleep/clock function, tests supply one that advances instantly. Flagging this honestly — *a recognised simplification, with the remedy already understood* — reads far better in review or interview than presenting a lab suite as production-grade. (Your CV's honest-positioning principle, applied to test engineering.)

### LAB 9.2 — Into the gate

Give `tango-devices` a real `pyproject.toml` (name `beamline-devices`, `src/` layout — Part 2, Lab 7.1's shape, now with `pytango` in dependencies), then the pipeline — which is Part 2, Chapter 3's pattern verbatim, pointed at this code:

```yaml
# tango-devices/.gitlab-ci.yml
stages: [test]

device-tests:
  stage: test
  tags: [lab]
  image: condaforge/miniforge3:latest
  script:
    - conda create -n ci -c conda-forge python=3.11 pytango pytest -y
    - conda run -n ci pip install -e .
    - conda run -n ci pytest tests/ -v
```

Push; watch it go green on your own runner. Then perform the ritual that proves the gate guards *this* code too: on a branch, break the ramp (`step` sign flipped, say), push, open the MR — red, unmergeable; fix, green, merge. A Tango device server failing its physics tests now blocks its own merge **exactly like any other component** — there is no separate, weaker quality bar for control-system code, and in Part 4 the same red pipeline will also be what blocks package builds and deployment.

**Checkpoint 9:** `pytest tests/ -v` passes locally in seconds; the same suite runs green in a GitLab CI job on Part 2's runner; a deliberately broken ramp is visibly unmergeable. Committed, merged, and the branch deleted like a professional.

---

## Chapter 10 — End of Part 3: review, self-assessment, and the demo script

### What you now have

The payload, complete — and every piece of it code in `tango-devices`:

| Layer | Implementation | Chapter |
|---|---|---|
| The model | device/attribute/command/property/state, events vs polling — cold | 1 |
| Running framework | `DataBaseds` + `TANGO_HOST` as a service, both install paths, reproducible via the `tango` role | 2 |
| First device | minimal `MagnetPowerSupply`; registration-as-code; framework freebies demonstrated | 3 |
| Credible physics | ramping magnet (setpoint/readback, async task, truthful states, pushed events, property-tuned); abortable motion | 4 |
| The beamline | image-serving camera, magnet-correlated Faraday cup, auto-alarming gauge with fault injection, honest interlock | 5 |
| Orchestration | Sardana Pool + MacroServer + thin controllers; `ascan` tracing a physical peak in spock | 6 |
| Operator surface | Taurus panel: live form, ramp trend, beam image, alarm colours — recorded on video | 7 |
| Memory | HDB++ event-driven archive → Part 2's Grafana; cross-device alarm → Part 2's Alertmanager | 8 |
| The gate | `DeviceTestContext` suite asserting states, limits, alarms, physics — green in Part 2's CI | 9 |

### The 10-minute demo script (rehearse it; combine with Part 2's for the full twenty)

1. **The model, on a whiteboard** — device name, five parts, "no client can tell simulator from hardware." (60 seconds, from Checkpoint 1.)
2. **The ramp** — write `setpoint=50` in the Taurus panel; narrate demand-vs-readback, `MOVING→ON`, events driving the trend. *This is the screen recording if live demo isn't possible.*
3. **The scan** — `ascan q1mot 0 70 14 0.5` in spock; point at the peak near 35 A: "the scan is real — it moved the magnet and the diagnostic responded."
4. **The trip** — `SimulateLeak()`: gauge red, permit false, alarm into Alertmanager — then the honesty line: "and the real protection layer would be a hardware SIS below all of this; software mirrors it, never constitutes it."
5. **The history** — the Grafana beam-history dashboard showing everything you just did, archived.
6. **The gate** — the red MR from Lab 9.2's ritual: "control-system code merges under the same rules as everything else."

### Self-assessment (answer without notes; section references in brackets)

1. Parse `linac/magnet/q1` — what are the three levels, and what enforces the convention? *(1.1)*
2. Attribute vs command vs property: give the design idiom that assigns a feature to each. *(1.1)*
3. Name four things Tango attaches to an attribute *besides* its value. *(1.1)*
4. Why can generic tools (synoptics, Sardana, alarm systems) work against devices they've never heard of? *(1.1, 1.5)*
5. What exactly does the Tango Database do at connection time — and what does it *not* do afterwards? *(1.2)*
6. Polling vs events: when does each make sense, and which two subsystems in this Part are event subscribers? *(1.3, 7.1, 8.1)*
7. What is `TANGO_HOST`, and what is the first thing you check when "nothing works"? *(2.3)*
8. Why register `TangoTest` before writing any code of your own? *(2.4)*
9. What does `super().init_device()` do, and when besides startup does `init_device` run? *(3.1, Lab 4.2)*
10. Which three behaviours in Lab 3.3's transcript came from attribute metadata rather than your code? *(Lab 3.3)*
11. Why is device registration a committed script rather than Jive clicks? *(Lab 3.2)*
12. Describe the setpoint/readback idiom and why the write handler must return immediately. *(4.1)*
13. Why is `ramp_rate` a property and not a constant — and how do you change it on a live instance? *(Lab 4.1–4.2)*
14. What promise does the state machine make that Sardana depends on? *(4.1, 6.1)*
15. What makes `Stop()` semantically different from just writing the current position? *(Lab 4.3)*
16. Scalar, spectrum, image: how does the camera declare a 2-D attribute? *(Lab 5.1)*
17. Why does the Faraday cup hold a `DeviceProxy`, and why is its target a property? *(Lab 5.2)*
18. Why do simulators *deserve* fault-injection commands like `SimulateLeak`? *(Lab 5.3)*
19. State the software/hardware interlock boundary in two honest sentences. *(Lab 5.4)*
20. Pool, controller, MacroServer, spock — one clause each. *(6.1)*
21. Why must Sardana controllers stay thin? *(Lab 6.2)*
22. Read `ascan q1mot 0 70 14 0.5` aloud as a sentence, and say what proves the scan was real. *(Lab 6.3)*
23. What does a Taurus widget derive from the model name alone? *(7.1)*
24. HDB++'s two device servers — which configures, which writes, and what do they subscribe to? *(8.1)*
25. Built-in vs rule-based alarms: what can the second express that the first cannot? *(8.2)*
26. Why route control-system alarms into Part 2's Alertmanager rather than a second pipeline? *(8.2)*
27. What does `DeviceTestContext` remove the need for, and which test layer does it serve? *(9.1)*
28. Two crafts in the test suite: why inject `ramp_rate`, and why poll-with-deadline instead of sleep? *(Lab 9.1)*
29. What is the honest limitation of wall-clock physics tests, and the known fix? *(Lab 9.1 callout)*
30. In one sentence: what did Chapter 9 make true about control-system code relative to every other component? *(Lab 9.2)*

### Glossary additions (extends Parts 1–2)

| Term | Definition |
|---|---|
| **archive event** | the event type tuned for HDB++'s significance thresholds, distinct from GUI-oriented change events |
| **Astor / Starter** | Tango's fleet supervisor GUI / the per-host device server it drives (noted; superseded here by systemd/K8s in Part 4) |
| **change event** | a server-pushed attribute update, fired on threshold-crossing change or explicitly via `push_change_event` |
| **ConfigurationManager / EventSubscriber (HDB++)** | the archiving control surface / the worker that subscribes and writes history rows |
| **controller (Sardana)** | the thin adapter class mapping Pool's generic motor/counter interface onto specific Tango devices |
| **demand / readback** | the written request (`setpoint`) vs the measured reality (`current`) — the universal controls idiom |
| **`DeviceTestContext`** | PyTango's in-process harness: a device class + proxy inside pytest, no database required |
| **device property** | per-instance configuration stored in the Tango DB, loaded by `super().init_device()`, reloaded by `Init` |
| **`DevState`** | the framework-wide state enum: `ON`, `OFF`, `MOVING`, `FAULT`, `ALARM`, `STANDBY`, `RUNNING`, `INIT`… |
| **fault injection** | deliberate test hooks on simulators (`SimulateLeak`) enabling unhappy-path testing |
| **green mode** | PyTango's concurrency selector; `Asyncio` enables `async` device internals (background ramps) |
| **image / spectrum attribute** | 2-D / 1-D array attributes (`dtype=((float,),)` / `(float,)`) |
| **interlock (software vs hardware)** | supervisory gating logic in the control system vs the certified PLC/hardwired protection beneath it — never conflate |
| **itango** | IPython preloaded with Tango conveniences; the fastest inspection shell |
| **Jive** | the Tango DB browser GUI — for inspecting, not defining (definitions live in Git) |
| **macro / MacroServer / spock** | a scan procedure / the device server executing macros / the operator's IPython console |
| **measurement group** | the Sardana set of channels read together at each scan point |
| **Pogo** | Tango's class-scaffolding code generator; its model file doubles as a reviewed interface definition |
| **Pool** | the Sardana device server owning scannable elements (motors, counters, measurement groups) |
| **quality (attribute)** | per-reading flag — `VALID`, `WARNING`, `ALARM`, `CHANGING`, `INVALID` — travelling with every value |
| **registration-as-code** | device inventory defined by committed, idempotent scripts run at environment build |
| **`TANGO_HOST`** | `host:port` of the Tango database — the one variable every process needs; check it first |
| **TangoTest** | the reference device used to verify plumbing before trusting custom code |
| **Taurus model** | the attribute/device name a widget binds to, from which it derives value, unit, format, and colours |

### Where Part 4 begins

Two complete systems now idle side by side: a delivery platform with a toy payload (Part 2 ships `beamline-utils`), and a control system started by a shell script. **Part 4 — Integration, Troubleshooting, and Schedule** fuses them: the device servers get real packaging (systemd-carrying `.deb`s for the host-installed pieces, OCI images for the cluster-hosted simulators), the `tango-devices` pipeline grows Part 2's full build-and-publish stages, `platform-gitops` learns the beamline's manifests — and then the book's thesis is demonstrated live: **one commit** changing the magnet's ramp behaviour flows through tests, packages, signing, and Argo CD until an operator's trend plot draws a different slope, untouched by human hands. Plus the troubleshooting compendium spanning both systems, and the realistic week-by-week schedule.

---

*End of Part 3 of 4.*
