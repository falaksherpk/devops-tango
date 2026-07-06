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
