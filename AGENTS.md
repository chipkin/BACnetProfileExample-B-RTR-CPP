# AGENTS.md

Guidance for AI coding agents working in this repository. See
<https://agents.md/> for the format. Human contributors should read
[README.md](README.md) first.

## What this project is

A **tutorial** C++ example that implements the BACnet **B-RTR (Router)** device
profile using the CAS BACnet Stack. It is one of a series - one git repo per
BACnet profile - seeded from B-ASC (Application Specific Controller) minus
DeviceCommunicationControl, plus a second Network Port and physical
inter-datalink routing. It is the series' canonical example for
**F-MULTIPORT** (more than one Network Port) and **F-ROUTER** (router
configuration). The top priority is that the code reads like a tutorial a
customer can learn from and copy-paste. Favour clarity over cleverness.

## Layout

This repository is self-contained:

- `main.cpp` - the example device.
- `common/` - the shared helper, vendored in-repo (not referenced via a path
  outside the repository).
- `submodules/cas-bacnet-stack/` - the **CAS BACnet Stack** as a git submodule
  (private). After cloning, run `git submodule update --init --recursive`.

## Build

This example links the CAS BACnet Stack as a prebuilt **STATIC** library - build
the library once from the pinned submodule commit, then configure and build:

```bash
git submodule update --init --recursive   # once, if not cloned with --recursive
tools/build-stack-static.sh BACnetProfileExample-B-RTR-CPP   # from the series root
cmake -B build -S . -DCAS_BACNET_STACK_LINK=STATIC
cmake --build build --config Release
```

The stack library build compiles the whole stack (~600 files) once and takes a
few minutes; the example itself then builds in seconds, and later incremental
rebuilds are fast. Use `-D CAS_STACK_DIR=...` only if your stack lives outside
the bundled submodule. The adapter also offers a SOURCE mode (compiles the
stack straight into the executable, no library build); this example builds and
ships STATIC only.

On Windows, if `msbuild`/CMake pick a toolset the stack's `.vcxproj` lacks,
force `v143` on both: `TOOLSET=v143 tools/build-stack-static.sh ...` and
`cmake -B build -S . -T v143 ...`.

## Run

```bash
./build/BACnetExampleBRTR [--port 47808] [--port2 47809] [--deviceID 389018]   # Linux/macOS
.\build\Release\BACnetExampleBRTR.exe [--port 47808] [--port2 47809] [--deviceID 389018]   # Windows
```

Interactive keys while running: `h` help, `q` quit, up/down nudge Analog Input 1,
`r` re-send I-Am-Router-To-Network on both ports.

## Conventions

- Device is named "Rainbow"; objects use the series' colour names; vendor id 389.
- Implement **only** the services and objects the B-RTR profile requires - but
  expose **every required property** of each object for Protocol_Revision 24.
- Outputs are **commandable**: store the 16-slot `Priority_Array` +
  `Relinquish_Default` in the app (the `Commandable` struct); let the stack
  resolve `Present_Value`. Writes land via the `SetProperty*` callbacks (value)
  and `SetPropertyNull` (relinquish). This part is unchanged from B-SA/B-ASC.
- **Two Network Ports, two UDP sockets (F-MULTIPORT).** Use
  `CASExampleHelper::SetupUDP(port, networkPortInstance)` /
  `SendIAm(deviceInstance, networkPortInstance)` (`common/` 2.3.0+) - never
  reach for a second, separate socket implementation. The shared receive/send
  callbacks already dispatch on `networkPortInstance`.
- **Router setup (F-ROUTER).** `BACnetStack_AddRouterPort`'s `networkType`
  parameter is the STACK'S OWN internal enum (`BACnetPacket::NetworkType_IP =
  0`), NOT the Network Port object's `Network_Type` property
  (`NETWORK_PORT_NETWORK_TYPE_IPV4 = 5`). Mixing them up fails with "unknown
  network type" - see `ROUTER_NETWORK_TYPE_IP` in `main.cpp`.
- **Do not enable `Routing_Table`.** Verified against the pinned stack: it
  aborts on read outside a test-tool build (stack issue #2038). Do not
  re-enable it without first checking whether that stack gap is fixed.
- **Do not claim cross-network forwarding works.** The pinned stack cannot
  route between two ports of the same network type (stack issue #2037) - only
  router *configuration* is verified. If a future stack pin fixes this,
  update README's gap notice, `TODO.md`, and the "Verify" section together.
- Every `GetProperty*` callback ends with `uint32_t* errorCode`. Leave it alone
  on a catch-all decline (the stack's decline-and-fabricate default answers
  required properties this app does not serve); set it only where this device
  knows the read is wrong (`State_Text` out of range is the one case here).
- Match the surrounding code style: `const`-correct parameters, check every stack
  return value, keep `main.cpp` linear and well-commented.
- **Never edit `common/` in this repo alone** - it is a vendored copy shared by
  every example in the series, with its own version (`COMMON_VERSION`) and
  changelog (`common/CHANGELOG.md`). To change it: edit it in
  `BACnetProfileExample-B-SS-CPP` (the source of truth) on a branch, bump the
  version, add a changelog entry, open a PR there, merge, THEN re-copy
  `common/` into every example repository including this one.

## How to verify a change

There are no unit tests; verification is behavioural:

1. Build, then run one instance with `--port`/`--port2` on clear UDP ports.
2. With a BACnet client (e.g. bacpypes3 or CAS BACnet Explorer), send
   **Who-Is** on EACH port and confirm **I-Am** from the device instance on
   both.
3. **ReadProperty** every required property of every object (both Network
   Ports too) directly against each port and confirm the values; confirm
   `Protocol_Revision` is 24.
4. **WriteProperty** a commandable output's `Present_Value` at a priority,
   re-read it, then write NULL to relinquish.
5. **Router announcement**: confirm `I-Am-Router-To-Network` and
   `Network-Number-Is` are sent (TX log lines) at start-up and on the `r` key.
6. **Do NOT assume cross-network forwarding works** - it currently does not
   (see Conventions above); do not write a test that assumes it does without
   first re-checking the stack gap is still open.

## Releasing

Bump `APP_VERSION` in `main.cpp` and add an entry to [CHANGELOG.md](CHANGELOG.md),
then tag `vX.Y.Z`. The GitHub Actions workflow builds and publishes the release.

## License

The example source code is dedicated to the public domain under
[CC0-1.0](LICENSE). The CAS BACnet Stack is a separate, commercially licensed
product and is not covered by that dedication.
