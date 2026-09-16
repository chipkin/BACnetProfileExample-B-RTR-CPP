# BACnet B-RTR (Router) - C++ example

A minimal, copy-paste-friendly example showing how to implement the BACnet
**B-RTR (Router)** device profile in C++ using the
[CAS BACnet Stack](https://store.chipkin.com/services/stacks/bacnet-stack).
It is a stand-alone BACnet router: it owns **two BACnet/IP Network Port
objects**, one per network it joins, and configures physical inter-datalink
routing between them, answers **ReadProperty**/**WriteProperty**, and is
discoverable via **Who-Is / I-Am**. It is the series' canonical example for
**F-ROUTER** and **F-MULTIPORT** (a device owning more than one Network Port).

**[Download a prebuilt binary](https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP/releases)**
(Windows and Linux x64) - or build it yourself, see [Build](#build) below.

- **[TUTORIAL.md](TUTORIAL.md)** - how to extend this example and how to review
  it for conformance. Read it when you start turning this into your own device.
- **[docs/PICS.md](docs/PICS.md)** - the Protocol Implementation Conformance
  Statement: every object, every property, and who answers it.

> **Versions:** this document describes **example v1.0.0**, built and verified
> against **CAS BACnet Stack 6.0.21** (`6.x` @ `abd4cee1`), at
> **Protocol_Revision 24**, with the vendored `common/` helper at **v2.5.0**.
> Running the example prints all three - if what it prints disagrees with this
> line, trust the program and check `CHANGELOG.md`.

> **IMPORTANT - read before you rely on this example for routing.** The pinned
> CAS BACnet Stack build genuinely accepts this device's router **configuration**
> (`AddRouterPort`, `AddRouterRoute`, `SetRouterEnabled` all succeed; `Network_Number`
> and `Network_Number_Quality` read back correctly on both ports; I-Am-Router-To-Network
> / Who-Is-Router-To-Network / Network-Number-Is all send) - but it does **NOT**
> forward NPDUs between the two ports, because both are BACnet/IP and the stack
> holds only one internal datalink instance per network type at this pin
> (a scope gap left by stack issue #304, which fixed this only for BACnet/SC; see
> `TODO.md` and [stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037)).
> A client reachable only through the OTHER port is **not** actually routed to.
> Each port answers ReadProperty/WriteProperty directly and correctly on its own
> network - what does not work is forwarding traffic **between** the two networks.

## The device this example creates

```
Device 389018  "Rainbow"   (Vendor 389 - Chipkin Automation Systems)
    |
    +-- Analog Input  1       "Bronze"       Present_Value  21.5      (REAL, degrees Celsius; read-only)
    +-- Binary Input  1       "Emerald"      Present_Value  inactive  (0 = inactive / 1 = active; read-only)
    +-- Multi-State Input 1   "Hot Pink"     Present_Value  1         (state, 1..3; read-only)
    +-- Analog Output 1       "Chartreuse"   Present_Value  20.0      (REAL setpoint; WRITABLE, commandable)
    +-- Binary Output 1       "Fuchsia"      Present_Value  inactive  (0/1; WRITABLE, commandable)
    +-- Multi-State Output 1  "Indigo"       Present_Value  1         (state, 1..3; WRITABLE, commandable)
    +-- Network Port 1        "Vermilion"    BACnet/IP, network 1     (port A of the router)
    +-- Network Port 2        "Vermilion 2"  BACnet/IP, network 2     (port B of the router)
```

The three **input** objects and three **output** objects are the shared
minimum/commandable pattern every example in this series carries. The **two
Network Ports** are this example's own contribution - see [TUTORIAL.md](TUTORIAL.md)
for how `main.cpp` wires them up as F-MULTIPORT and configures the router
(F-ROUTER).

## What this example supports

The example implements exactly the capabilities below - and nothing more,
which is the point of a profile example.

### BIBBs (BACnet Interoperability Building Blocks)

| BIBB | Description | Supported |
|------|-------------|:---------:|
| DS-RP-B | Data Sharing - ReadProperty - B | ✅ |
| DS-WP-B | Data Sharing - WriteProperty - B | ✅ |
| DM-DDB-A | Device Management - Dynamic Device Binding - A (initiates Who-Is) | ✅ |
| DM-DDB-B | Device Management - Dynamic Device Binding - B | ✅ |
| DM-DOB-B | Device Management - Dynamic Object Binding - B | ✅ |
| NM-RC-B | Network Management - Router Configuration - B | 🟡 partial - see the gap notice above; configuration/announcement genuinely work, inter-network *forwarding* does not at this stack pin |
| DM-LM-B | Device Management - List Manipulation - B (AddListElement/RemoveListElement) | ❌ not implemented - `TODO.md`, [stack issue #2033](https://github.com/chipkin/cas-bacnet-stack/issues/2033) |

This example does not implement DeviceCommunicationControl (DM-DCC-B),
alarming, scheduling, or trending - not required by B-RTR.

### Services (executed / B-side)

| Service | Notes |
|---------|-------|
| ReadProperty (12) | Responds to property reads on both networks (DS-RP-B). Verified with a live client. |
| WriteProperty (15) | Accepts writes to the commandable outputs' Present_Value (DS-WP-B). Verified with a live client. |
| Who-Is / I-Am | Answers Who-Is with I-Am; broadcasts I-Am on start-up on BOTH networks (DM-DDB-A/B). |
| Who-Has / I-Have | Answers Who-Has with I-Have (DM-DOB-B). |
| Who-Is-Router-To-Network / I-Am-Router-To-Network | Sent on both ports at start-up and on the `r` key (NM-RC-B). Verified sent (TX on the wire); not verified that this device *answers* an incoming Who-Is-Router-To-Network - see [Verify](#verify). |
| Network-Number-Is | Sent on both ports at start-up (NM-RC-B). |
| AddListElement (8) / RemoveListElement (9) | **Not implemented** - see `TODO.md`. |

### Object types

| Object type | Instance | Name | Access |
|-------------|:--------:|------|--------|
| Device | 389018 | Rainbow | - |
| Analog Input | 1 | Bronze | read-only |
| Binary Input | 1 | Emerald | read-only |
| Multi-State Input | 1 | Hot Pink | read-only |
| Analog Output | 1 | Chartreuse | writable (commandable) |
| Binary Output | 1 | Fuchsia | writable (commandable) |
| Multi-State Output | 1 | Indigo | writable (commandable) |
| Network Port | 1 | Vermilion | network 1 |
| Network Port | 2 | Vermilion 2 | network 2 |

Every required property of every object, and who answers it, is in
[docs/PICS.md](docs/PICS.md).

## Requires the CAS BACnet Stack (licensed product)

This example **builds against the CAS BACnet Stack, which is a commercial Chipkin
product** - it is not free or open source, and there is no public/trial build.
The stack is referenced here as the **private** git submodule
`submodules/cas-bacnet-stack`; you can only fetch and build it once you have a CAS
BACnet Stack license and access to that repository.

**To get the CAS BACnet Stack (and access to build this example), contact
Chipkin:** <https://store.chipkin.com/services/stacks/bacnet-stack> or
sales@chipkin.com.

You do not need a stack licence to *read* this example, or to run a
[prebuilt release binary](https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP/releases).
The licence is what lets you *build* it - that is the part the stack submodule
gates.

## What's in this repository

This is a **self-contained** project. It ships:

- `main.cpp` - the example device.
- `common/` - the shared helper (UDP, callbacks, CLI, keyboard) vendored in.
- `CMakeLists.txt` - the build, the same on Windows, Linux, and macOS.
- `docs/PICS.md` - the conformance statement.
- `submodules/cas-bacnet-stack/` - the **CAS BACnet Stack as a git submodule**
  (private; requires a license - see above). Its sources are compiled into the
  executable, so there is no library or DLL to build, ship, or install.

## Prerequisites

- A C++17 compiler (MSVC, GCC, or Clang).
- CMake >= 3.15.
- Git (to fetch the stack submodule).

### Windows

- **C++ compiler** - install
  [Visual Studio Community](https://visualstudio.microsoft.com/downloads/)
  (free) and select the **"Desktop development with C++"** workload.
- **CMake** - from <https://cmake.org/download/>, or `winget install Kitware.CMake`.

### Linux / macOS

- Debian/Ubuntu: `sudo apt install build-essential cmake git`
- macOS: `xcode-select --install` and `brew install cmake`

## Build

CMake only, and the same two commands on every platform:

```bash
git clone --recursive https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP.git
cd BACnetProfileExample-B-RTR-CPP

cmake -B build -S .
cmake --build build --config Release
```

Already cloned without `--recursive`? Run `git submodule update --init --recursive`
first - the build needs the stack submodule.

> **The first build takes a few minutes** - it compiles the entire CAS BACnet
> Stack (~600 source files) into the executable. Rebuilds after that are
> incremental and take seconds.

If your CAS BACnet Stack lives somewhere other than the bundled submodule, point
CMake at it: `cmake -B build -S . -D CAS_STACK_DIR=/path/to/cas-bacnet-stack`.

## Run

```bash
# Linux / macOS
./build/BACnetExampleBRTR --port 47808 --port2 47809

# Windows
.\build\Release\BACnetExampleBRTR.exe --port 47808 --port2 47809
```

Expected output:

```
BACnet B-RTR (Router) Example - C++ v1.0.0
CAS BACnet Stack version: 6.0.21.0
Common helper (common/) version: 2.5.0
FYI: Listening for BACnet/IP on UDP port 47808 (Network Port 1).
FYI: Listening for BACnet/IP on UDP port 47809 (Network Port 2).
... (router-port bind/attribution "Error:" lines - see the gap notice above) ...
TX 21 bytes to 192.168.3.255:47808 (broadcast) (Network Port 1)
TX 21 bytes to 192.168.3.255:47809 (broadcast) (Network Port 2)
FYI: I-Am-Router-To-Network broadcast on Network Port 1 (Vermilion): sent; on Network Port 2 (Vermilion 2): sent
FYI: Device 389018 ("Rainbow") ready. Vendor ID 389. Routing network 1 (Vermilion, UDP 47808) <-> network 2 (Vermilion 2, UDP 47809). Press 'h' for help.
```

The device listens on UDP **47808** and **47809** by default. Allow both ports
through your firewall.

> **A wall of red `Error:` lines at start-up is expected and is not your bug** -
> it includes both ports hearing their own broadcast I-Am, a one-time BACnet/SC
> UUID notice, and the `BindRouterPort`/`FindPortByNetworkType` lines from the
> forwarding gap above. [TUTORIAL.md](TUTORIAL.md#troubleshooting) explains all
> of them.

### Command-line options

| Option | Default | Meaning |
|--------|---------|---------|
| `--port <n>` | `47808` | UDP port for Network Port 1 (Vermilion). |
| `--port2 <n>` | `--port + 1` | UDP port for Network Port 2 (Vermilion 2). Must differ from `--port`. |
| `--deviceID <n>` | `389018` | The device's BACnet instance number. |
| `--help`, `-h` | - | Show usage and exit. |
| `--version` | - | Print the example, stack, and `common/` helper versions, then exit. |

### Interactive commands

| Key | Action |
|-----|--------|
| `h` | Show the version information and this command list. |
| `q` | Quit. |
| up arrow | Increase Analog Input 1 (`Bronze`) by 1.1. |
| down arrow | Decrease Analog Input 1 (`Bronze`) by 1.1. |
| `r` | Re-send I-Am-Router-To-Network on both ports now. |

## Verify

You need a BACnet client, such as the
[CAS BACnet Explorer](https://store.chipkin.com/products/tools/cas-bacnet-explorer).

**Confirmed working, on the live wire, against this build:**

1. **Discover on each network independently** - ReadProperty of Device
   `Object_Name` succeeds directly against `127.0.0.1:47808` (network 1) AND
   `127.0.0.1:47809` (network 2), both returning `"Rainbow"` (DS-RP-B works on
   both ports).
2. **Network_Number / Network_Number_Quality** - ReadProperty of Network Port 1's
   `Network_Number` = `1`, `Network_Number_Quality` = `configured`; Network Port
   2's = `2` / `configured`. Both `Object_Name`s (`Vermilion` / `Vermilion 2`)
   read back correctly.
3. **WriteProperty (DS-WP-B)** - WriteProperty Analog Output 1 `Present_Value` =
   `25.5` at priority 8; re-read confirms `25.5`.
4. **I-Am-Router-To-Network transmission** - confirmed via the device's own TX
   log at start-up and on the `r` key.

**NOT verified / known not to work - do not rely on these:**

- **Cross-network forwarding.** A ReadProperty sent to a device reachable only
  through the OTHER Network Port is **not** routed - see the gap notice at the
  top of this README and `TODO.md` ([stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037)).
- **`Routing_Table` readback.** Deliberately not enabled - see `TODO.md`
  ([stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038)).
- **This device answering an incoming Who-Is-Router-To-Network.** Only the
  outbound (start-up / `r`-key) direction was exercised in verification.
- **The `DEMO_REMOTE_NETWORK` (3) route.** No device sits on it; `AddRouterRoute`
  accepting the call is confirmed, an actual forwarded packet is not (and
  could not be, given the forwarding gap above).

For a property-by-property review against the conformance statement, see
[TUTORIAL.md](TUTORIAL.md).


## The BACnet profile example series

<!-- PROFILE-TABLE:BEGIN (generated from cas-bacnet-stack-examples/docs/profile-table.md - do not edit here) -->
The CAS BACnet Stack supports every standardized device profile in ASHRAE 135-2024 Annex L, and there is one example repository per profile. Pick the profile your device claims, then the language you build in. "Ask" means the example hasn't been built yet for that language - [contact Chipkin](https://www.chipkin.com/contact/) if you need one.

### Controllers (Annex L.4)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-SS** Smart Sensor | [B-SS-CPP](https://github.com/chipkin/BACnetProfileExample-B-SS-CPP) | Ask | Ask | Ask | Ask |
| **B-SA** Smart Actuator | [B-SA-CPP](https://github.com/chipkin/BACnetProfileExample-B-SA-CPP) | Ask | Ask | Ask | Ask |
| **B-ASC** Application Specific Controller | [B-ASC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ASC-CPP) | [B-ASC-Node](https://github.com/chipkin/BACnetProfileExample-B-ASC-Node) | Ask | Ask | Ask |
| **B-AAC** Advanced Application Controller | [B-AAC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AAC-CPP) | Ask | Ask | Ask | Ask |
| **B-BC** Building Controller | [B-BC-CPP](https://github.com/chipkin/BACnetProfileExample-B-BC-CPP) | Ask | Ask | Ask | Ask |

### Life safety controllers (Annex L.5)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-LSC** Life Safety Controller | [B-LSC-CPP](https://github.com/chipkin/BACnetProfileExample-B-LSC-CPP) 🚧 | Ask | Ask | Ask | Ask |
| **B-ALSC** Advanced Life Safety Controller | [B-ALSC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ALSC-CPP) | Ask | Ask | Ask | Ask |

### Access control controllers (Annex L.6)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-ACC** Access Control Controller | [B-ACC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACC-CPP) | Ask | Ask | Ask | Ask |
| **B-AACC** Advanced Access Control Controller | [B-AACC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AACC-CPP) | Ask | Ask | Ask | Ask |

### Lighting controllers (Annex L.11)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-LD** Lighting Device | [B-LD-CPP](https://github.com/chipkin/BACnetProfileExample-B-LD-CPP) | Ask | Ask | Ask | Ask |
| **B-LS** Lighting Supervisor | [B-LS-CPP](https://github.com/chipkin/BACnetProfileExample-B-LS-CPP) | Ask | Ask | Ask | Ask |

### Elevator controllers (Annex L.13)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-EM** Elevator Monitor | [B-EM-CPP](https://github.com/chipkin/BACnetProfileExample-B-EM-CPP) | Ask | Ask | Ask | Ask |
| **B-EC** Elevator Controller | [B-EC-CPP](https://github.com/chipkin/BACnetProfileExample-B-EC-CPP) | Ask | Ask | Ask | Ask |
| **B-AEC** Advanced Elevator Controller | [B-AEC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AEC-CPP) | Ask | Ask | Ask | Ask |

### Authentication and authorization (Annex L.14)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-AS** Authorization Server | [B-AS-CPP](https://github.com/chipkin/BACnetProfileExample-B-AS-CPP) | Ask | Ask | Ask | Ask |

### Miscellaneous (Annex L.7, combinable with any one family)

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-BBMD** Broadcast Management Device | [B-BBMD-CPP](https://github.com/chipkin/BACnetProfileExample-B-BBMD-CPP) | Ask | Ask | Ask | Ask |
| **B-ACDC** Access Control Door Controller | [B-ACDC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACDC-CPP) | Ask | Ask | Ask | Ask |
| **B-ACCR** Access Control Credential Reader | [B-ACCR-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACCR-CPP) | Ask | Ask | Ask | Ask |
| **B-RTR** Router | [B-RTR-CPP](https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP) | Ask | Ask | Ask | Ask |
| **B-GW** Gateway | [B-GW-CPP](https://github.com/chipkin/BACnetProfileExample-B-GW-CPP) | Ask | Ask | Ask | Ask |
| **B-DAP** Device Address Proxy | [B-DAP-CPP](https://github.com/chipkin/BACnetProfileExample-B-DAP-CPP) | Ask | Ask | Ask | Ask |
| **B-SCHUB** BACnet/SC Hub | [B-SCHUB-CPP](https://github.com/chipkin/BACnetProfileExample-B-SCHUB-CPP) | Ask | Ask | Ask | Ask |
| **B-GENERAL** General device (Annex L.8) | *(satisfied by every example above)* | — | — | — | — |

### Operator interfaces and workstations (Annex L.1–L.3, L.9–L.10, L.12)

Client-side profiles.

| Profile | C++ | Node.js | C# | Rust | Python |
|---|---|---|---|---|---|
| **B-OD** Operator Display | [B-OD-CPP](https://github.com/chipkin/BACnetProfileExample-B-OD-CPP) | Ask | Ask | Ask | Ask |
| **B-OWS** Operator Workstation | planned | — | — | — | — |
| **B-AWS** Advanced Operator Workstation | planned | — | — | — | — |
| **B-XAWS** Extended Advanced Operator Workstation | planned | — | — | — | — |
| **B-LSAP** Life Safety Annunciator Panel | planned | — | — | — | — |
| **B-LSWS** Life Safety Workstation | planned | — | — | — | — |
| **B-ALSWS** Advanced Life Safety Workstation | planned | — | — | — | — |
| **B-ACSD** Access Control Security Display | planned | — | — | — | — |
| **B-ACWS** Access Control Workstation | planned | — | — | — | — |
| **B-AACWS** Advanced Access Control Workstation | planned | — | — | — | — |
| **B-LOD** Lighting Operator Display | planned | — | — | — | — |
| **B-ALWS** Advanced Lighting Workstation | planned | — | — | — | — |
| **B-LCS** Lighting Control Station | planned | — | — | — | — |
| **B-ALCS** Advanced Lighting Control Station | planned | — | — | — | — |
| **B-ED** Elevator Display | planned | — | — | — | — |
| **B-EWS** Elevator Workstation | planned | — | — | — | — |
| **B-AEWS** Advanced Elevator Workstation | planned | — | — | — | — |

🚧 = in progress. "Ask" = not yet built for that language; contact Chipkin if you need it. Profile definitions: ANSI/ASHRAE 135-2024 Annex L. BIBB definitions: Annex K. Get the stack: <https://store.chipkin.com/services/stacks/bacnet-stack>.
<!-- PROFILE-TABLE:END -->

## References

- **ANSI/ASHRAE Standard 135** (BACnet) - the protocol standard. Object model
  (Clause 12), services (Clause 15), routers (Clause 6.6), BACnet/IP (Annex J),
  device profiles (Annex L). Purchase / preview via the
  [ASHRAE store](https://www.ashrae.org/technical-resources/standards-and-guidelines).
- **What is BACnet?** - Chipkin's introduction:
  <https://docs.chipkin.com/protocols/bacnet/>.
- **CAS BACnet Stack** - product page and documentation:
  <https://store.chipkin.com/services/stacks/bacnet-stack>.
- **A BACnet client for testing this device** - the
  [CAS BACnet Explorer](https://store.chipkin.com/products/tools/cas-bacnet-explorer).
- **Shared helper used by this example** - [`common/README.md`](common/README.md).

See also [TUTORIAL.md](TUTORIAL.md), [docs/PICS.md](docs/PICS.md),
[TODO.md](TODO.md), [CHANGELOG.md](CHANGELOG.md), and [AGENTS.md](AGENTS.md).
