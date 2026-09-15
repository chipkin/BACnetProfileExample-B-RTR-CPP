# BACnet B-RTR (Router) - C++ example

A minimal, copy-paste-friendly example showing how to implement the BACnet
**B-RTR (Router)** device profile in C++ using the
[CAS BACnet Stack](https://store.chipkin.com/services/stacks/bacnet-stack).
It is a stand-alone BACnet router: it owns **two BACnet/IP Network Port
objects**, one per network it joins, and configures physical inter-datalink
routing between them, answers **ReadProperty**/**WriteProperty**, and is
discoverable via **Who-Is / I-Am**.

Part of the CAS BACnet Stack **BACnet profile example series** - one repository
per BACnet device profile. This example claims **only** B-RTR, and is the
series' canonical example for **F-ROUTER** and **F-MULTIPORT** (a device
owning more than one Network Port).

Reading order: this repository stands on its own - **you can start here.** If you also want the
gentler introductions to the shared sensor/actuator objects, [B-SS (Smart
Sensor)](https://github.com/chipkin/BACnetProfileExample-B-SS-CPP) is the first example in the
series; this one repeats everything it needs.

> **Versions:** this document describes **example v1.0.0**, built and verified
> against **CAS BACnet Stack 6.0.21** (`6.x` @ `abd4cee1`), linked as a static
> library, at **Protocol_Revision 24**, with the vendored `common/` helper at
> **v2.5.0**. Running the example prints all three - if what it prints
> disagrees with this line, trust the program and check `CHANGELOG.md`.

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

## Quickstart

You need a CAS BACnet Stack licence and access to its private submodule (see
[Requires the CAS BACnet Stack](#requires-the-cas-bacnet-stack-licensed-product)).
Then:

```bash
git clone --recursive https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP.git
cd BACnetProfileExample-B-RTR-CPP
tools/build-stack-static.sh BACnetProfileExample-B-RTR-CPP    # from the series root; builds the static library
cmake -B build -S . -DCAS_BACNET_STACK_LINK=STATIC
cmake --build build --config Release
./build/Release/BACnetExampleBRTR.exe        # Windows; drop Release/ on Linux
```

The device binds two UDP ports, announces itself on both networks, and prints
`Press 'h' for help`.

> **You will see one or more red `Error:` lines at start-up. Most of the
> device is fine** - see [Troubleshooting](#troubleshooting); the
> `FindPortByNetworkType` / `BindRouterPort` lines specifically are the known,
> documented routing gap above, not a bug in this example.

## What is a B-RTR (Router) profile?

A **device profile** is a standard "template" defined in Annex L of ANSI/ASHRAE
135. It lists the capabilities a class of device must support so that any
compliant client knows what to expect, and the BACnet Testing Laboratories (BTL)
certify devices against it. (New to BACnet in general? See Chipkin's
[What is BACnet?](https://docs.chipkin.com/protocols/bacnet/) guide.)

**B-RTR (Router)** is a stand-alone BACnet router (ANSI/ASHRAE 135 clause 6.6):
a device with two (or more) Network Port objects that forwards NPDUs between
the BACnet networks they connect, and speaks the network-layer messages other
devices and routers use to discover it and learn the internetwork's topology.

**Reading the capability names.** Each capability below is a **BIBB** (BACnet
Interoperability Building Block). Every BIBB name ends in **-A** or **-B**:
**-A** = the device that *initiates* a request; **-B** = the device that
*responds*. `DM-DDB-A` means this device also **initiates** Who-Is (not just
answers it) - appropriate for a router, which is expected to actively discover
devices on the networks it joins.

**What the profile requires:**

- **Data Sharing - ReadProperty - B side (DS-RP-B):** answer **ReadProperty**.
- **Data Sharing - WriteProperty - B side (DS-WP-B):** accept **WriteProperty**
  to its objects.
- **Device Management - Dynamic Device Binding - A/B (DM-DDB-A/B):** answer
  **Who-Is** with **I-Am**, broadcast an unsolicited I-Am on each of its
  networks at start-up, and (A side) itself send **Who-Is** to discover other
  devices.
- **Device Management - Dynamic Object Binding - B side (DM-DOB-B):** answer
  **Who-Has** with **I-Have**.
- **Network Management - Router Configuration - B side (NM-RC-B):** configure
  router ports/routes; answer **Who-Is-Router-To-Network** with
  **I-Am-Router-To-Network**; announce **Network-Number-Is** on each network.

**What the profile does NOT require** - and this example omits on purpose:
**alarming / event reporting**, **scheduling**, **trending**, and (see the
gap notice above and `TODO.md`) **AddListElement/RemoveListElement (DM-LM-B)**,
which the pinned stack's customer-facing build does not compile in.

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
minimum/commandable pattern every example in this series carries (inherited
from B-SA/B-ASC, minus B-ASC's DeviceCommunicationControl). The **two Network
Ports** are this example's own contribution - see F-MULTIPORT/F-ROUTER below.

## F-MULTIPORT: two Network Ports, two UDP sockets

Every other example in this series owns exactly one Network Port and one UDP
socket. This example is the series' first with two, and is canonical for that
pattern (`common/` 2.3.0+): `CASExampleHelper::SetupUDP(port, networkPortInstance)`
binds an additional socket keyed to a specific Network Port instance, and the
shared receive/send callbacks (`common/CASExampleHelper.cpp`) dispatch on that
instance internally - `main.cpp` just calls `SetupUDP` twice and
`SendIAm(deviceInstance, networkPortInstance)` once per port. See
`common/CHANGELOG.md`'s 2.3.0 entry and the "WHY THIS IS SAFE FOR SINGLE-PORT
EXAMPLES" comment in `CASExampleHelper.cpp` for why this did not change any
other (single-port) example's behaviour.

Both ports run on **one host** and are distinguished purely by **UDP port
number** (`--port` / `--port2`, default `--port + 1`) - they report the same
`IP_Address`/`IP_Subnet_Mask` and different `BACnet_IP_UDP_Port` /
`Network_Number`.

## F-ROUTER: router configuration

At start-up, `main.cpp`:

1. `BACnetStack_AddRouterPort` once per Network Port, registering the network
   it directly connects (network 1 on Vermilion, network 2 on Vermilion 2).
2. `BACnetStack_AddRouterRoute` for one DEMONSTRATION-ONLY network (3), one hop
   beyond port 2, with no next-hop MAC configured - this exercises the API; no
   device in this example series actually sits on network 3 (see `main.cpp`'s
   `DEMO_REMOTE_NETWORK` comment).
3. `BACnetStack_SetRouterEnabled(deviceInstance, true)` - routing is opt-in
   even with ports configured.
4. Broadcasts **I-Am-Router-To-Network** on both ports (`networkNumbers = NULL`,
   announcing this router's full configured routing table per the stack's
   `#1014` fix), and again whenever the **`r`** key is pressed.
5. Broadcasts **Network-Number-Is** on both ports, announcing each network's
   own (configured) number.
6. Broadcasts **Who-Is-Router-To-Network** (unrestricted) on both ports, the
   polite router start-up probe for any other router already present (none is
   expected to answer in this example).

`BACnetStack_AddRouterPort`'s `networkType` parameter is the stack's own
internal datalink enum (`BACnetPacket::NetworkType_IP = 0`), which is a
**different enumeration** from the Network Port object's `Network_Type`
property (`NETWORK_PORT_NETWORK_TYPE_IPV4 = 5`, used with
`AddNetworkPortObject`) - passing the wrong one fails with "unknown network
type"; see the `ROUTER_NETWORK_TYPE_IP` comment in `main.cpp`.

**`Routing_Table` (cl. 12.56.62) is deliberately NOT enabled or claimed.**
Verified on the wire: enabling it and reading it aborts the request, because
the pinned stack's `BACnetBusinessLogic::UseCallbackGetList` has no case for
Network Port/Routing_Table outside a test-tool build. See `TODO.md` and
[stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038).

## What this example supports

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

### Services (executed / B-side)

| Service | Notes |
|---------|-------|
| ReadProperty (12) | Responds to property reads on both networks (DS-RP-B). Verified with a live client. |
| WriteProperty (15) | Accepts writes to the commandable outputs' Present_Value (DS-WP-B). Verified with a live client. |
| Who-Is / I-Am | Answers Who-Is with I-Am; broadcasts I-Am on start-up on BOTH networks (DM-DDB-A/B). |
| Who-Has / I-Have | Answers Who-Has with I-Have (DM-DOB-B). |
| Who-Is-Router-To-Network / I-Am-Router-To-Network | Sent on both ports at start-up and on the `r` key (NM-RC-B). Verified sent (TX on the wire); not verified that this device *answers* an incoming Who-Is-Router-To-Network with a matching reply - see Verify below. |
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

## Before you ship

This example is a tutorial, and it identifies itself as one. Everything in this
table is read by clients and shown to the operator in **every discovery tool on
the network**. Left as-is, your product appears on a real site announcing itself
as a Chipkin demo. None of it is cosmetic.

| Constant (`main.cpp`) | Ships as | Change it to |
|---|---|---|
| `VENDOR_IDENTIFIER` | `389` (Chipkin) | **Your** company's vendor ID. Assigned by ASHRAE, free: <https://bacnet.org/assigned-vendor-ids/> |
| `VENDOR_NAME` | `Chipkin Automation Systems` | Your company name — must match the vendor ID above. |
| `DEVICE_NAME` | `"Rainbow"` | Your device's `Object_Name`. **Must be unique across the BACnet internetwork.** |
| `MODEL_NAME` | `CAS BACnet Stack Example - B-RTR` | Your model designation. |
| `DEVICE_DESCRIPTION` | a description of *this example* | What your device actually is. |
| `FIRMWARE_REVISION` / `APPLICATION_SOFTWARE_VERSION` | `1.0.0` | Your real versions. |
| Device instance | `389018` (`--deviceID` overrides) | Must be unique on the internetwork. |
| `NETWORK_NUMBER_PORT_1` / `NETWORK_NUMBER_PORT_2` | `1` / `2` | Your site's actual network numbers - these must be unique across the whole BACnet internetwork, same as a device instance. |

`main.cpp` marks this block with a `CHANGE ALL OF THIS BEFORE YOU SHIP` banner.

## Requires the CAS BACnet Stack (licensed product)

This example **builds against the CAS BACnet Stack, which is a commercial Chipkin
product** - it is not free or open source, and there is no public/trial build.
The stack is referenced here as the **private** git submodule
`submodules/cas-bacnet-stack`; you can only fetch and build it once you have a CAS
BACnet Stack license and access to that repository.

**To get the CAS BACnet Stack (and access to build this example), contact
Chipkin:** <https://store.chipkin.com/services/stacks/bacnet-stack> or
sales@chipkin.com.

You do not need a stack licence to *read* this example. Every file outside
submodules/ is CC0 public domain.

## What's in this repository

- `main.cpp` - the example device.
- `common/` - the shared helper (UDP, callbacks, CLI, keyboard) vendored in.
- `submodules/cas-bacnet-stack/` - the **CAS BACnet Stack as a git submodule**
  (private; requires a license). Built into a prebuilt **STATIC** library by
  the stack's own project files (`tools/build-stack-static.sh`), then linked -
  no DLL is shipped.

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

## Get the code

```bash
git clone --recursive https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP.git
cd BACnetProfileExample-B-RTR-CPP

# already cloned without --recursive? fetch the submodule:
git submodule update --init --recursive
```

## Build

This example links the CAS BACnet Stack as a prebuilt **STATIC** library. Build
the library once from the pinned submodule commit, then configure and build the
example against it:

```bash
tools/build-stack-static.sh BACnetProfileExample-B-RTR-CPP   # from the series root
cmake -B build -S . -DCAS_BACNET_STACK_LINK=STATIC
cmake --build build --config Release
```

> **The stack library build takes a few minutes** the first time - it compiles
> the entire CAS BACnet Stack (~600 source files) once. The example itself then
> builds in seconds and rebuilds incrementally. Build in parallel to cut that
> down: `cmake --build build --config Release --parallel`.
>
> On Windows, if `msbuild` picks a toolset the stack's `.vcxproj` does not have
> installed, force the one Visual Studio 2022 ships (`v143`):
> `TOOLSET=v143 tools/build-stack-static.sh BACnetProfileExample-B-RTR-CPP`, and
> configure CMake with the matching generator toolset: `cmake -B build -S . -T v143 ...`.

If your CAS BACnet Stack lives somewhere other than the bundled submodule, point
CMake at it: `cmake -B build -S . -D CAS_STACK_DIR=/path/to/cas-bacnet-stack`.

### Link mode

This example links the stack through the `CASBACnetStack::Adapter` CMake target
in **STATIC** mode - `-DCAS_BACNET_STACK_LINK=STATIC` links the prebuilt
`CASBACnetStack_x64_Release.lib` / `libCASBACnetStack_x64_Release.a` built by
`tools/build-stack-static.sh` above. Every mode requires calling
`LoadBACnetFunctions()` once at the top of `main()` before any other
`BACnetStack_*` call. On MSVC the adapter also forces the static CRT (`/MT`) to
match how the library is built. A **SOURCE** mode also exists (compiles the
stack's `source/*.cpp` straight into the executable) - this example is built
and published in **STATIC** mode only.

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
| `r` | Re-send I-Am-Router-To-Network on both ports now (`common/` 2.4.0+). |

## Verify

You need a BACnet client. [bacpypes3](https://github.com/JoelBender/bacpypes3)
or a GUI tool such as [YABE](https://sourceforge.net/projects/yetanotherbacnetexplorer/)
both work.

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
  This was the runbook's own wire-verification target for this example and it
  genuinely fails at this stack pin; it is not a gap in this example's code.
- **`Routing_Table` readback.** Deliberately not enabled - see `TODO.md`
  ([stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038)).
- **This device answering an incoming Who-Is-Router-To-Network.** Only the
  outbound (start-up / `r`-key) direction was exercised in verification; a
  peer router's unsolicited query was not tested against this build.
- **The `DEMO_REMOTE_NETWORK` (3) route.** No device sits on it; `AddRouterRoute`
  accepting the call is confirmed, an actual forwarded packet is not (and
  could not be, given the forwarding gap above).

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| Start-up prints `BindRouterPort ... second port of networkType=[0] ... issue #304` and `FindPortByNetworkType ... Cannot attribute an incoming NPDU` | **Expected, not your bug.** The pinned stack cannot route between two ports of the same network type (both BACnet/IP here) - see the gap notice at the top of this README, `TODO.md`, and [stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037). Each port still works fine on its own network. |
| ReadProperty of `Routing_Table` aborts | Expected - not enabled in this example; see `TODO.md` and [stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038). |
| On start-up the app also prints a `UUID has not been set` line | Benign - the stack starts a BACnet/SC datalink this BACnet/IP-only example never configures. Same as every other example in the series. |
| CMake error: *"CAS BACnet Stack adapter not found under: ..."* | Submodules not initialized. Run `git submodule update --init --recursive`. |
| `Failed to bind UDP port ...` | Another program is using that port. Stop it, or pass a different `--port`/`--port2`. |
| `--port2` equals `--port` | Rejected on purpose - they must be two different UDP ports (two different simulated networks). |
| First build seems stuck for minutes | Normal - it's compiling ~600 stack files. Only the first build is slow. |
| `git submodule update` fails with *Permission denied* / *repository not found* | The CAS BACnet Stack submodule is a **private** repo - see [Requires the CAS BACnet Stack](#requires-the-cas-bacnet-stack-licensed-product). |

## The BACnet profile example series

<!-- PROFILE-TABLE:BEGIN (generated from cas-bacnet-stack-examples/docs/profile-table.md - do not edit here) -->
The CAS BACnet Stack supports every standardized device profile in ASHRAE 135-2024 Annex L. One example repository per profile shows how. ✅ = the required BIBB (service) is supported by the CAS BACnet Stack; the **Example** column is the state of that profile's tutorial repository.

### Controllers (Annex L.4)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-SS** Smart Sensor | [B-SS-CPP](https://github.com/chipkin/BACnetProfileExample-B-SS-CPP) ✅ | ✅ DS-RP-B · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-SA** Smart Actuator | [B-SA-CPP](https://github.com/chipkin/BACnetProfileExample-B-SA-CPP) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-ASC** Application Specific Controller | [B-ASC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ASC-CPP) ✅ · [B-ASC-Node](https://github.com/chipkin/BACnetProfileExample-B-ASC-Node) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B |
| **B-AAC** Advanced Application Controller | [B-AAC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AAC-CPP) ✅ | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ AE-N-I-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ AE-CRL-B · ✅ SCHED-I-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B |
| **B-BC** Building Controller | [B-BC-CPP](https://github.com/chipkin/BACnetProfileExample-B-BC-CPP) 📝 | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-RPM-B · ✅ DS-WP-A · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ AE-N-I-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ AE-CRL-B · ✅ SCHED-E-B · ✅ T-VMT-I-B · ✅ T-ATR-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B · ✅ DM-BR-B |

### Life safety controllers (Annex L.5)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-LSC** Life Safety Controller | [B-LSC-CPP](https://github.com/chipkin/BACnetProfileExample-B-LSC-CPP) 📝 | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ DS-COV-B · ✅ AE-LS-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B |
| **B-ALSC** Advanced Life Safety Controller | [B-ALSC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ALSC-CPP) 📝 | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ DS-COV-B · ✅ AE-LS-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ AE-EL-I-B · ✅ SCHED-I-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B |

### Access control controllers (Annex L.6)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-ACC** Access Control Controller | [B-ACC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACC-CPP) 📝 | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ DS-COV-B · ✅ DS-ACUC-B · ✅ DS-ACSC-B · ✅ AE-AC-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ AE-EL-I-B · ✅ SCHED-I-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B · ✅ DM-BR-B |
| **B-AACC** Advanced Access Control Controller | [B-AACC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AACC-CPP) 📝 | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-RPM-B · ✅ DS-WP-A · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ DS-COV-A · ✅ DS-COV-B · ✅ DS-ACAD-A · ☐ DS-ACCDI-A · ✅ DS-ACUC-B · ✅ DS-ACSC-B · ✅ AE-AC-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ AE-EL-I-B · ✅ SCHED-I-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B · ✅ DM-BR-B |

### Lighting controllers (Annex L.11)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-LD** Lighting Device | [B-LD-CPP](https://github.com/chipkin/BACnetProfileExample-B-LD-CPP) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DS-LO-B / DS-BLO-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B |
| **B-LS** Lighting Supervisor | [B-LS-CPP](https://github.com/chipkin/BACnetProfileExample-B-LS-CPP) 📝 | ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-WP-B · ✅ DS-WG-E-B · ✅ DS-ALO-A · ✅ SCHED-E-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B |

### Elevator controllers (Annex L.13)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-EM** Elevator Monitor | [B-EM-CPP](https://github.com/chipkin/BACnetProfileExample-B-EM-CPP) ✅ | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-COV-B · ✅ DS-COVM-B · ✅ AE-N-I-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B |
| **B-EC** Elevator Controller | [B-EC-CPP](https://github.com/chipkin/BACnetProfileExample-B-EC-CPP) 📝 | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ DS-COV-B · ✅ DS-COVM-B · ✅ AE-N-I-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-RD-B |
| **B-AEC** Advanced Elevator Controller | [B-AEC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AEC-CPP) 📝 | ✅ DS-RP-B · ✅ DS-RPM-B · ✅ DS-WP-B · ✅ DS-WPM-B · ✅ DS-COV-B · ✅ DS-COVM-B · ✅ AE-N-I-B · ✅ AE-ACK-B · ✅ AE-INFO-B · ✅ AE-EL-I-B · ✅ SCHED-I-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B · ✅ DM-OCD-B · ✅ DM-RD-B · ✅ DM-BR-B |

### Authentication and authorization (Annex L.14)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-AS** Authorization Server | [B-AS-CPP](https://github.com/chipkin/BACnetProfileExample-B-AS-CPP) 📝 | ✅ DS-RP-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ AA-AS-B |

### Miscellaneous (Annex L.7, combinable with any one family)

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-BBMD** Broadcast Management Device | [B-BBMD-CPP](https://github.com/chipkin/BACnetProfileExample-B-BBMD-CPP) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ NM-BBMDC-B |
| **B-ACDC** Access Control Door Controller | [B-ACDC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACDC-CPP) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DS-ACAD-B · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-ACCR** Access Control Credential Reader | [B-ACCR-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACCR-CPP) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DS-COV-B · ✅ DS-ACCDI-B · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-RTR** Router | [B-RTR-CPP](https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP) 📝 | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DM-DDB-A · ✅ DM-DOB-B · ✅ DM-LM-B · ✅ NM-RC-B |
| **B-GW** Gateway | [B-GW-CPP](https://github.com/chipkin/BACnetProfileExample-B-GW-CPP) 📝 | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ GW-EO-B / GW-VN-B |
| **B-DAP** Device Address Proxy | [B-DAP-CPP](https://github.com/chipkin/BACnetProfileExample-B-DAP-CPP) ✅ | ✅ DS-RP-B · ✅ DS-WP-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DAB-B |
| **B-SCHUB** BACnet/SC Hub | [B-SCHUB-CPP](https://github.com/chipkin/BACnetProfileExample-B-SCHUB-CPP) 📝 | ✅ DS-RP-B · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ NM-SCH-B |
| **B-GENERAL** General device (Annex L.8) | *(satisfied by every example above)* | ✅ DS-RP-B · ✅ DM-DDB-B · ✅ DM-DOB-B |

### Operator interfaces and workstations (Annex L.1–L.3, L.9–L.10, L.12) — client-side profiles

| Profile | Example | Required BIBBs (services) |
|---|---|---|
| **B-OD** Operator Display | [B-OD-CPP](https://github.com/chipkin/BACnetProfileExample-B-OD-CPP) ✅ | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-V-A · ✅ DS-M-A · ✅ AE-N-A · ✅ AE-VN-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-OWS** Operator Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-V-A · ✅ DS-M-A · ✅ AE-N-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-VM-A · ✅ AE-VN-A · ✅ SCHED-VM-A · ✅ T-V-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-MTS-A |
| **B-AWS** Advanced Operator Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-AV-A · ✅ DS-AM-A · ✅ AE-N-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-AVM-A · ✅ AE-AVN-A · ✅ AE-ELVM-A · ✅ SCHED-AVM-A · ✅ T-AVM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A · ✅ DM-DDA-A · ✅ NM-CC-A · ✅ AR-AVM-A |
| **B-XAWS** Extended Advanced Operator Workstation | planned | ✅ union of B-AWS + B-AACWS + B-ALWS + B-AEWS |
| **B-LSAP** Life Safety Annunciator Panel | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-LSV-A · ✅ AE-N-A · ✅ AE-LS-A · ✅ AE-ACK-A · ✅ AE-LSVN-A |
| **B-LSWS** Life Safety Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-LSV-A · ✅ DS-LSM-A · ✅ AE-N-A · ✅ AE-LS-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-LSVM-A · ✅ AE-LSAVN-A · ✅ AE-ELV-A · ✅ SCHED-VM-A · ✅ T-V-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A |
| **B-ALSWS** Advanced Life Safety Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-LSAV-A · ✅ DS-LSAM-A · ✅ AE-N-A · ✅ AE-LS-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-LSAVM-A · ✅ AE-LSAVN-A · ✅ AE-ELVM-A · ✅ SCHED-AVM-A · ✅ T-AVM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A · ✅ AR-AVM-A |
| **B-ACSD** Access Control Security Display | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-ACV-A · ✅ DS-ACM-A · ✅ AE-N-A · ✅ AE-AC-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-ACAVN-A · ✅ AE-ELV-A · ✅ SCHED-VM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-MTS-A |
| **B-ACWS** Access Control Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-ACAV-A · ✅ DS-ACM-A · ✅ DS-ACUC-A · ✅ AE-N-A · ✅ AE-AC-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-ACVM-A · ✅ AE-ACAVN-A · ✅ AE-ELV-A · ✅ SCHED-VM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A |
| **B-AACWS** Advanced Access Control Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-ACAV-A · ✅ DS-ACAM-A · ✅ DS-ACUC-A · ✅ DS-ACSC-A · ✅ AE-N-A · ✅ AE-AC-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-ACAVM-A · ✅ AE-ACAVN-A · ✅ AE-ELVM-A · ✅ SCHED-AVM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A · ✅ AR-AVM-A |
| **B-LOD** Lighting Operator Display | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-LV-A · ✅ DS-WG-A · ✅ DS-ALO-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-ALWS** Advanced Lighting Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-LAV-A · ✅ DS-LAM-A · ✅ DS-WG-A · ✅ DS-ALO-A · ✅ AE-N-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-AVM-A · ✅ AE-AVN-A · ✅ AE-ELVM-A · ✅ SCHED-AVM-A · ✅ T-AVM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A |
| **B-LCS** Lighting Control Station | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-LO-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B |
| **B-ALCS** Advanced Lighting Control Station | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-WG-A · ✅ DS-ALO-A · ✅ SCHED-E-B · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B · ✅ DM-DCC-B · ✅ DM-TS-B / DM-UTC-B |
| **B-ED** Elevator Display | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-WP-A · ✅ DS-EV-A · ✅ AE-N-A · ✅ AE-ACK-A · ✅ AE-EVN-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-DOB-B |
| **B-EWS** Elevator Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-COVM-A · ✅ DS-EV-A · ✅ DS-EM-A · ✅ AE-N-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-EVM-A · ✅ AE-EAVN-A · ✅ SCHED-VM-A · ✅ T-V-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A |
| **B-AEWS** Advanced Elevator Workstation | planned | ✅ DS-RP-A · ✅ DS-RP-B · ✅ DS-RPM-A · ✅ DS-WP-A · ✅ DS-WPM-A · ✅ DS-COVM-A · ✅ DS-EAV-A · ✅ DS-EAM-A · ✅ AE-N-A · ✅ AE-ACK-A · ✅ AE-AS-A · ✅ AE-EAVM-A · ✅ AE-EAVN-A · ✅ AE-ELVM-A · ✅ SCHED-AVM-A · ✅ T-AVM-A · ✅ DM-DDB-A · ✅ DM-DDB-B · ✅ DM-ANM-A · ✅ DM-ADM-A · ✅ DM-DOB-B · ✅ DM-DCC-A · ✅ DM-MTS-A · ✅ DM-OCD-A · ✅ DM-RD-A · ✅ DM-BR-A |

Profile definitions: ANSI/ASHRAE 135-2024 Annex L. BIBB definitions: Annex K. Get the stack: <https://store.chipkin.com/services/stacks/bacnet-stack>.
<!-- PROFILE-TABLE:END -->

## Objects and properties

<!-- OBJECTS-PROPERTIES:BEGIN (generated by tools/gen-objects-properties.py from docs/objects.json - do not edit here) -->
Every object this example creates, and every REQUIRED property of each (per ANSI/ASHRAE 135-2024 clause 12 and the stack's `docs/property-profile-reference.md`), plus the optional properties the example turns on. **Served by** says who answers a ReadProperty: the **stack** generates it, or the **app** serves it from a `GetProperty*` callback in `main.cpp`. A ⚠ row is a required property the app does not serve and the stack would fill with a default - that is a defect, not a feature.

### Device 389018 "Rainbow" - vendor 389 (Chipkin Automation Systems); instance configurable with --deviceID. The properties in 'accepted' are not served from a GetProperty callback because the stack itself is the source of truth for them - Protocol_Revision/Protocol_Version are stack build constants, Protocol_Services_Supported/Protocol_Object_Types_Supported are computed from the BACnetStack_SetServiceEnabled/AddObject calls this example already makes, Object_List and Device_Address_Binding are live stack-maintained tables, System_Status/Database_Revision/Max_APDU_Length_Accepted/Segmentation_Supported/APDU_Timeout/Number_Of_APDU_Retries are the stack's own configuration defaults for a device this example does not override

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| System_Status | BACnetDeviceStatus | stack default, accepted (Generic Enumerated default: `0`) | no |
| Vendor_Name | CharacterString | app | no |
| Vendor_Identifier | Unsigned16 | app | no |
| Model_Name | CharacterString | app | no |
| Firmware_Revision | CharacterString | app | no |
| Application_Software_Version | CharacterString | app | no |
| Description *(optional, enabled)* | CharacterString | app | no |
| Protocol_Version | Unsigned | stack default, accepted (`BACNET_PROTOCOL_VERSION`) | no |
| Protocol_Revision | Unsigned | stack default, accepted (`BACNET_PROTOCOL_REVISION`) | no |
| Protocol_Services_Supported | BACnetServicesSupported | stack default, accepted (computed from which services are enabled) | no |
| Protocol_Object_Types_Supported | BACnetObjectTypesSupported | stack default, accepted (computed from which object types are supported) | no |
| Object_List | BACnetARRAY[N] of BACnetObjectIdentifier | stack default, accepted (None known - a read fails with `unknown-property` or an empt) | no |
| Max_APDU_Length_Accepted | Unsigned | stack default, accepted (`CAS_BACNET_DEVICE_DEFAULT_MAX_APDU_LENGTH_ACCEPTED`) | no |
| Segmentation_Supported | BACnetSegmentation | stack default, accepted (`BACnetSegmentation::noSegmentation`) | no |
| APDU_Timeout | Unsigned | stack default, accepted (`CAS_BACNET_DEVICE_DEFAULT_APDU_TIMEOUT`) | no |
| Number_Of_APDU_Retries | Unsigned | stack default, accepted (`CAS_BACNET_DEVICE_DEFAULT_NUMBER_OF_APDU_RETRIES`) | no |
| Device_Address_Binding | BACnetLIST of BACnetAddressBinding | stack default, accepted (the live Device_Address_Binding (DAB) table) | no |
| Database_Revision | Unsigned | stack default, accepted (Generic UnsignedInteger default: `0`) | no |

### Analog Input 1 "Bronze" - REAL, degrees Celsius; starts at 21.5; read-only

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Present_Value | Real | app | no |
| Status_Flags | BACnetStatusFlags | stack | no |
| Event_State | BACnetEventState | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Units | BACnetEngineeringUnits | app | no |

### Binary Input 1 "Emerald" - starts inactive; read-only

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Present_Value | BACnetBinaryPV | app | no |
| Status_Flags | BACnetStatusFlags | stack | no |
| Event_State | BACnetEventState | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Polarity | BACnetPolarity | app | no |

### Multi-state Input 1 "Hot Pink" - state 1 of 3: On, Off, Auto; read-only

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Present_Value | Unsigned | app | no |
| Status_Flags | BACnetStatusFlags | stack | no |
| Event_State | BACnetEventState | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Number_Of_States | Unsigned | app | no |
| State_Text *(optional, enabled)* | BACnetARRAY[N] of CharacterString | app | no |

### Analog Output 1 "Chartreuse" - REAL setpoint, default 20.0 C; commandable via a 16-slot Priority_Array (WriteProperty, DS-WP-B) - Present_Value/Priority_Array/Current_Command_Priority are resolved by the stack from the slots the app serves through GetPropertyReal/GetPropertyBool

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Present_Value | Real | stack | yes |
| Status_Flags | BACnetStatusFlags | stack | no |
| Event_State | BACnetEventState | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Units | BACnetEngineeringUnits | app | no |
| Priority_Array | BACnetARRAY[16] of BACnetOptionalReal | stack | no |
| Relinquish_Default | Real | app | no |
| Current_Command_Priority | BACnetOptionalUnsigned | stack | no |

### Binary Output 1 "Fuchsia" - active/inactive, default inactive; commandable via a 16-slot Priority_Array (WriteProperty, DS-WP-B) - Present_Value/Priority_Array/Current_Command_Priority are resolved by the stack from the slots the app serves through GetPropertyEnumerated/GetPropertyBool

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Present_Value | BACnetBinaryPV | stack | yes |
| Status_Flags | BACnetStatusFlags | stack | no |
| Event_State | BACnetEventState | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Polarity | BACnetPolarity | app | no |
| Priority_Array | BACnetARRAY[16] of BACnetOptionalBinaryPV | stack | no |
| Relinquish_Default | BACnetBinaryPV | app | no |
| Current_Command_Priority | BACnetOptionalUnsigned | stack | no |

### Multi-state Output 1 "Indigo" - state 1 of 3, default state 1; commandable via a 16-slot Priority_Array (WriteProperty, DS-WP-B) - Present_Value/Priority_Array/Current_Command_Priority are resolved by the stack from the slots the app serves through GetPropertyUnsignedInteger/GetPropertyBool

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Present_Value | Unsigned | stack | yes |
| Status_Flags | BACnetStatusFlags | stack | no |
| Event_State | BACnetEventState | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Number_Of_States | Unsigned | app | no |
| Priority_Array | BACnetARRAY[16] of BACnetOptionalUnsigned | stack | no |
| Relinquish_Default | Unsigned | app | no |
| Current_Command_Priority | BACnetOptionalUnsigned | stack | no |

### Network Port 1 "Vermilion" - BACnet/IP, network 1, configured (F-MULTIPORT/F-ROUTER port A). Network_Type/Protocol_Level are set from BACnetStack_AddNetworkPortObject()'s arguments at start-up (as is Network_Number = 1 / Network_Number_Quality = configured, verified readable on the wire; not listed as a separate row here because the stack's own property-profile-reference.md does not carry those two properties for the type-agnostic 'Network Port' heading this generator reads). Changes_Pending is likewise computed and answered natively. Reliability has no fault condition this example detects, so it is accepted at the generic default (normal). Routing_Table is deliberately NOT enabled - verified on the wire that the pinned stack cannot serve it outside a test-tool build (TODO.md, stack issue #2038).

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Status_Flags | BACnetStatusFlags | stack | no |
| Reliability | BACnetReliability | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Network_Type | BACnetNetworkType | app | no |
| Protocol_Level | BACnetProtocolLevel | app | no |
| Changes_Pending | Boolean | app | no |

### Network Port 2 "Vermilion 2" - BACnet/IP, network 2, configured (F-MULTIPORT/F-ROUTER port B). Same notes as Network Port 1 (Vermilion) apply, with Network_Number = 2. Physically routes to network 1 via BACnetStack_AddRouterPort/SetRouterEnabled, but see TODO.md and stack issue #2037: this stack pin cannot forward NPDUs between two ports of the same network type, so cross-network traffic is not verified end-to-end even though both ports individually answer ReadProperty/WriteProperty correctly.

| Property | Datatype | Served by | Writable |
|---|---|---|:---:|
| Object_Identifier | BACnetObjectIdentifier | stack | no |
| Object_Name | CharacterString | app | no |
| Object_Type | BACnetObjectType | stack | no |
| Status_Flags | BACnetStatusFlags | stack | no |
| Reliability | BACnetReliability | stack default, accepted (Generic Enumerated default: `0`) | no |
| Out_Of_Service | Boolean | app | no |
| Network_Type | BACnetNetworkType | app | no |
| Protocol_Level | BACnetProtocolLevel | app | no |
| Changes_Pending | Boolean | app | no |

<!-- OBJECTS-PROPERTIES:END -->

## Footprint

| Metric | Value |
|---|---|
| Binary size | not yet released |
| SHA-256 (prefix) | not yet released |
| Start-up time to `ready` | not yet released |
| Stack commit | not yet released |
| Link mode | not yet released |
| Compiler | not yet released |

<!-- METRICS -->
