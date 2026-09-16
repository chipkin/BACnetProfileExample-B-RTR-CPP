# AGENTS.md

Guidance for AI coding agents working in this repository. See
<https://agents.md/> for the format. Human contributors should read
[README.md](README.md) first, then [TUTORIAL.md](TUTORIAL.md).

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
- `common/` - the shared helper (UDP, callbacks, CLI, keyboard) vendored in.
- `README.md` - what this example is. Keep it short and about THIS example only.
- `TUTORIAL.md` - how to extend and review the example: adding objects, the
  F-MULTIPORT/F-ROUTER patterns, who serves which property, and
  troubleshooting. Long-form material that would bloat the README belongs
  here.
- `docs/PICS.md` - the Protocol Implementation Conformance Statement. Its
  objects-and-properties section is GENERATED from `docs/objects.json`; do not
  hand-edit between the `OBJECTS-PROPERTIES` markers.
- `docs/objects.json` - the input to that generator. Update it in the same change
  as any `main.cpp` change that adds an object or a `GetProperty*` branch. It
  now includes the Device object as its first entry.
- `TODO.md` - the two filed stack gaps this example works around (DM-LM-B not
  compiled into the customer build; inter-network forwarding not implemented
  at this pin) and the third (`Routing_Table` read aborts). Read before
  touching router behaviour.
- `submodules/cas-bacnet-stack/` - the **CAS BACnet Stack** as a git submodule
  (private; compiled from source). After cloning, run
  `git submodule update --init --recursive`.

The `PROFILE-TABLE` block in README.md is also generated, from the example-series
repository's `docs/profile-table.md`. Edit it there, not here.

## Build

Plain CMake, identical on every platform, in the adapter's default SOURCE mode
(the stack's sources are compiled into the executable - no prebuilt library, no
DLL, no per-platform pre-step):

```bash
git submodule update --init --recursive   # once, if not cloned with --recursive
cmake -B build -S .
cmake --build build --config Release
```

The first build compiles the whole stack (~600 files) and takes a few minutes;
rebuilds after that are incremental and fast. Use `-D CAS_STACK_DIR=...` only if
your stack lives outside the bundled submodule. Do not reintroduce a link-mode
flag or a series-root build script into the documented build: a customer
downloads this repository on its own and must be able to build it with the two
commands above.

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
  update README's gap notice, `TUTORIAL.md`, `docs/PICS.md` section 9,
  `TODO.md`, and the README's "Verify" section together.
- Every `GetProperty*` callback ends with `uint32_t* errorCode`. Leave it alone
  on a catch-all decline (the stack's decline-and-fabricate default answers
  required properties this app does not serve); set it only where this device
  knows the read is wrong (`State_Text` out of range is the one case here).
- Match the surrounding code style: `const`-correct parameters, check every stack
  return value, keep `main.cpp` linear and well-commented.
- The `CHANGE ALL OF THIS BEFORE YOU SHIP` block in `main.cpp` carries a
  per-constant comment on what to change it to (vendor ID, device name/
  uniqueness, model, description, versions, device instance, and the two
  network numbers). Keep those comments in sync with `docs/PICS.md` section 1
  if the placeholder values change.
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
7. If you changed the objects or their properties, regenerate `docs/PICS.md`
   (`python tools/gen-objects-properties.py BACnetProfileExample-B-RTR-CPP` from
   the series root) and confirm no row comes out flagged with ⚠.

## Releasing

Bump `APP_VERSION` in `main.cpp` and add an entry to [CHANGELOG.md](CHANGELOG.md),
then tag `vX.Y.Z`. The GitHub Actions workflow builds and publishes the release.

## License

See [LICENSE](LICENSE). The CAS BACnet Stack is a separate, commercially
licensed product and is not covered by it.
