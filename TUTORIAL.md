# Tutorial - extending and reviewing the B-RTR example

[README.md](README.md) says what this example *is*. This document is the
*how*: how to extend it into your own router, what each object type needs the
application to serve, how the two Network Ports and the router configuration
fit together, how to review the result for conformance, and what goes wrong
when you get it subtly right.

Read this once before you start changing `main.cpp`. The most expensive
mistake in this example is silent, and the section it lives in is
[Add a second analog input](#add-a-second-analog-input).

- [Extending the example](#extending-the-example)
- [F-MULTIPORT: two Network Ports, two UDP sockets](#f-multiport-two-network-ports-two-udp-sockets)
- [F-ROUTER: router configuration](#f-router-router-configuration)
- [What each object type needs you to serve](#what-each-object-type-needs-you-to-serve)
- [Who serves what: the application or the stack?](#who-serves-what-the-application-or-the-stack)
- [Reviewing your device](#reviewing-your-device)
- [Troubleshooting](#troubleshooting)

## Extending the example

The example is intentionally small so it's easy to change.

**Change an object's value or name** - edit the constants / callbacks in
`main.cpp` (e.g. the initial value of `g_analogInput1Value`, or the
`"Bronze"` string in `GetPropertyCharString`).

**Change the device identity before you ship** - vendor ID, vendor name,
model name, description, firmware revision, device name and the two network
numbers are all in the `CHANGE ALL OF THIS BEFORE YOU SHIP` block at the top
of `main.cpp`, with a per-field comment on each saying what to change it to.
That block is the authoritative checklist; it is in the source rather than
here so it cannot be skipped by someone who only reads the code.

### Add a second analog input

Read this whole recipe before starting — the last step is the one that is
easy to miss and the one BTL will fail you for.

> **Why there are four edits, not three — and why skipping one is SILENT.**
> Most of the `GetProperty*` callbacks match on **both** object type *and*
> instance (`objectInstance == ANALOG_INPUT_INSTANCE`), so a new instance
> falls through every one of them. `GetPropertyBool` is the exception for
> `Out_Of_Service`: it matches on type only, so it works for a new instance
> for free.
>
> Here is the part that matters, and it is the opposite of what most people
> assume: falling through a callback does **not** reliably produce an error.
> The stack errors only for the few properties it refuses to invent -
> `Present_Value`, `Number_Of_States`, `Relinquish_Default`, and a Network
> Port's `APDU_Length`. For everything else it **silently substitutes a
> default**:
>
> | Property | If you forget to serve it | Loud? |
> |---|---|:--:|
> | `Present_Value` | Error (`read-access-denied`) | yes |
> | `Object_Name` | reads back as the string **`"undefined"`** | **no** |
> | `Units` | reads back as **`no-units` (95)** | **no** |
> | `Out_Of_Service` | served on type alone — works by accident | n/a |
>
> **Doesn't the `errorCode` out-parameter fix this?** Only if you use it, and
> only where it is right to. Each `GetProperty*` callback ends with a
> `uint32_t* errorCode` that the stack presets to `success` and reads only
> when you return `false`, so you *can* turn any decline into a chosen BACnet
> error. But ending every callback with `*errorCode = unknown-property`
> breaks the device: the stack's decline-and-fabricate path is what answers
> required properties an application is not expected to serve — the Device's
> `Max_APDU_Length_Accepted`, `APDU_Timeout` and `Number_Of_APDU_Retries`
> among them. Name an error on the catch-all and those start failing instead
> of answering. Set `errorCode` only where *this device* knows the read is
> wrong; `main.cpp` does it in exactly one place, `State_Text` with an
> out-of-range array index.
>
> It is worse than "wrong value": the object's `Property_List` **still
> advertises `Units` (117)**. So the object actively claims to have the
> property, and then answers with a default. Nothing on the wire says you
> forgot anything.
>
> So a half-added object does not look broken; it looks **healthy**. Add two
> of them and both report `Object_Name "undefined"` — duplicate object names
> inside one device, which is a spec violation and a hard BTL failure that
> every scan tool will render as a perfectly good object. **"It scanned OK"
> is exactly the failure mode, not evidence against it.**

```cpp
// 1) a new instance number (in section 1).
//    Naming: a second object of a type is "<Colour> 2" - so Analog Input 2 is
//    "Bronze 2", NOT a new colour. Each object TYPE owns one colour series-wide.
static const uint32_t ANALOG_INPUT_2_INSTANCE = 2;   // "Bronze 2"
static float g_analogInput2Value = 23.1f;            // its live value

// 2) add the object (in main, next to the other BACnetStack_AddObject calls).
//    Check the return, like every other stack call in this file.
if (!BACnetStack_AddObject(g_deviceInstance, OBJECT_TYPE_ANALOG_INPUT, ANALOG_INPUT_2_INSTANCE)) {
    printf("Error: Failed to add Analog Input 2 (Bronze 2).\n");
    return 1;
}

// 3) serve its Present_Value + Object_Name:
//    GetPropertyReal:        AI/2 + Present_Value -> *value = g_analogInput2Value;
//    GetPropertyCharString:  AI/2 + Object_Name   -> "Bronze 2"

// 4) DO NOT SKIP: serve its Units, in GetPropertyEnumerated.
//    Units is a REQUIRED property of an Analog Input. The existing check reads
//    `objectInstance == ANALOG_INPUT_INSTANCE`, which is instance 1 - so without
//    this, reading Analog Input 2's Units returns an ERROR and the object is
//    NON-CONFORMANT. It will still appear in the Object_List and its
//    Present_Value will read back perfectly, so the device looks healthy right
//    up until BTL certification.
//    GetPropertyEnumerated:  AI/2 + Units -> *value = ENGINEERING_UNITS_DEGREES_CELSIUS;
```

Then re-run the README's Verify steps **against Analog Input 2**, not just
Analog Input 1 — read every required property, on both networks, and **diff
it against Analog Input 1**. Any property that comes back `"undefined"`,
`no-units`, or `0` where object 1 returns something real is a step you
missed. Because the failure is silent (see the table above), this diff is
the only thing that catches it.

## F-MULTIPORT: two Network Ports, two UDP sockets

Most examples in this series own exactly one Network Port and one UDP socket.
This example is the series' canonical example of owning two:
`CASExampleHelper::SetupUDP(port, networkPortInstance)` binds an additional
socket keyed to a specific Network Port instance, and the shared
receive/send callbacks (`common/CASExampleHelper.cpp`) dispatch on that
instance internally - `main.cpp` just calls `SetupUDP` twice and
`SendIAm(deviceInstance, networkPortInstance)` once per port. See
`common/CHANGELOG.md`'s 2.3.0 entry and the "WHY THIS IS SAFE FOR
SINGLE-PORT EXAMPLES" comment in `CASExampleHelper.cpp` for why this did not
change any single-port example's behaviour.

Both ports run on **one host** and are distinguished purely by **UDP port
number** (`--port` / `--port2`, default `--port + 1`) - they report the same
`IP_Address`/`IP_Subnet_Mask` and different `BACnet_IP_UDP_Port` /
`Network_Number`.

Adding a third port to a device you build from this example follows the same
pattern: a third `NETWORK_PORT_x_INSTANCE`, a third `SetupUDP(port,
instance)` call, a third `SendIAm(deviceInstance, instance)` call, and (if it
should also route) a third `BACnetStack_AddRouterPort`.

## F-ROUTER: router configuration

At start-up, `main.cpp`:

1. `BACnetStack_AddRouterPort` once per Network Port, registering the network
   it directly connects (network 1 on Vermilion, network 2 on Vermilion 2).
2. `BACnetStack_AddRouterRoute` for one DEMONSTRATION-ONLY network (3), one
   hop beyond port 2, with no next-hop MAC configured - this exercises the
   API; no device in this example series actually sits on network 3 (see
   `main.cpp`'s `DEMO_REMOTE_NETWORK` comment).
3. `BACnetStack_SetRouterEnabled(deviceInstance, true)` - routing is opt-in
   even with ports configured.
4. Broadcasts **I-Am-Router-To-Network** on both ports (`networkNumbers =
   NULL`, announcing this router's full configured routing table per the
   stack's `#1014` fix), and again whenever the **`r`** key is pressed.
5. Broadcasts **Network-Number-Is** on both ports, announcing each network's
   own (configured) number.
6. Broadcasts **Who-Is-Router-To-Network** (unrestricted) on both ports, the
   polite router start-up probe for any other router already present (none
   is expected to answer in this example).

`BACnetStack_AddRouterPort`'s `networkType` parameter is the stack's own
internal datalink enum (`BACnetPacket::NetworkType_IP = 0`), which is a
**different enumeration** from the Network Port object's `Network_Type`
property (`NETWORK_PORT_NETWORK_TYPE_IPV4 = 5`, used with
`AddNetworkPortObject`) - passing the wrong one fails with "unknown network
type"; see the `ROUTER_NETWORK_TYPE_IP` comment in `main.cpp`.

> **The pinned CAS BACnet Stack genuinely accepts this router's
> configuration** (`AddRouterPort`, `AddRouterRoute`, `SetRouterEnabled` all
> succeed; `Network_Number` and `Network_Number_Quality` read back correctly
> on both ports; I-Am-Router-To-Network / Who-Is-Router-To-Network /
> Network-Number-Is all send) - but it does **NOT** forward NPDUs between the
> two ports, because both are BACnet/IP and the stack holds only one
> internal datalink instance per network type at this pin (a scope gap left
> by stack issue #304, which fixed this only for BACnet/SC; see `TODO.md`
> and [stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037)).
> A client reachable only through the OTHER port is **not** actually routed
> to. Each port answers ReadProperty/WriteProperty directly and correctly on
> its own network - what does not work is forwarding traffic **between** the
> two networks. Do not remove or soften this warning without first
> re-verifying the stack gap is closed.

**`Routing_Table` (cl. 12.56.62) is deliberately NOT enabled or claimed.**
Verified on the wire: enabling it and reading it aborts the request, because
the pinned stack's `BACnetBusinessLogic::UseCallbackGetList` has no case for
Network Port/Routing_Table outside a test-tool build. See `TODO.md` and
[stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038).

**DM-LM-B (AddListElement/RemoveListElement, services 8/9) is not
implemented.** The B-RTR profile card calls for it, and the stack genuinely
implements both services end-to-end against a Group object's
`List_Of_Group_Members` - but only when compiled with
`STACK_OPTION_DM_LM_LIST_MANIPULATION`, which the customer-facing STATIC
build of the pinned stack does not define. See `TODO.md` and
[stack issue #2033](https://github.com/chipkin/cas-bacnet-stack/issues/2033)
for the full verification trail. Do not add a Group object or claim these
services until that stack gap is confirmed fixed.

## What each object type needs you to serve

The application must serve every REQUIRED property the stack does not
generate. It differs per type — this is the checklist, so you do not have to
infer it:

| Object type | You must serve | Plus |
|---|---|---|
| Analog Input | `Present_Value` (Real), `Object_Name`, `Units` | — |
| Binary Input | `Present_Value` (Enumerated), `Object_Name` | `Polarity` |
| Multi-State Input | `Present_Value` (Unsigned), `Object_Name` | `Number_Of_States` |
| Analog Output (commandable) | `Object_Name`, `Units`, `Relinquish_Default` | 16-slot Priority_Array via `GetPropertyReal`/`GetPropertyBool`; `Present_Value` resolved by the stack |
| Binary Output (commandable) | `Object_Name`, `Polarity`, `Relinquish_Default` | 16-slot Priority_Array via `GetPropertyEnumerated`/`GetPropertyBool`; `Present_Value` resolved by the stack |
| Multi-State Output (commandable) | `Object_Name`, `Number_Of_States`, `Relinquish_Default` | 16-slot Priority_Array via `GetPropertyUnsignedInteger`/`GetPropertyBool`; `Present_Value` resolved by the stack |
| Network Port | `Object_Name`, `Network_Type`, `Protocol_Level`, `Changes_Pending`, `IP_Address`/`IP_Subnet_Mask`/`IP_Default_Gateway`, `BACnet_IP_UDP_Port` | one instance per network the device joins |

## Who serves what: the application or the stack?

The single most common question when reading this file is "who answers this
property?" For Analog Output 1 (the representative commandable object), the
whole picture:

| Property | Served by | How |
|---|---|---|
| `Object_Identifier` | **stack** | generated from the object you added |
| `Object_Type` | **stack** | generated |
| `Object_List` | **stack** | generated (Device object) |
| `Property_List` | **stack** | generated |
| `Status_Flags` | **stack** | generated |
| `Event_State` | **stack**, sort of | no intrinsic alarming here, so nothing serves it — it reads `normal` only because `normal` is the enumeration's zero value and the stack substitutes a datatype default. Correct by coincidence, not design. |
| `Out_Of_Service` | **you** | `GetPropertyBool` — matched on object **type only** |
| `Present_Value` | **stack** | resolved from the `Priority_Array` slots you serve; **writable** — `SetPropertyReal` stores the slot |
| `Priority_Array` | **stack**, backed by **you** | `GetPropertyBool` returns whether each slot is set; `SetPropertyReal`/`SetPropertyNull` write/relinquish a slot |
| `Current_Command_Priority` | **stack** | computed from the Priority_Array slots |
| `Relinquish_Default` | **you** | `GetPropertyReal` |
| `Object_Name` | **you** | `GetPropertyCharString` |
| `Units` | **you** | `GetPropertyEnumerated` |

Every object, not just this one, is in [docs/PICS.md](docs/PICS.md).

Going beyond this (alarming/events, COV, scheduling, trending) means
implementing a richer profile — see the series table in
[README.md](README.md).

## Reviewing your device

After you have changed anything, review it against the conformance statement
rather than against "it looked fine in the explorer":

1. Regenerate [docs/PICS.md](docs/PICS.md) after editing `docs/objects.json`
   (see [Keeping the PICS honest](#keeping-the-pics-honest) below). A ⚠ row
   is a required property nothing serves.
2. Read **every** property listed for **every** object with a BACnet client,
   on **both** ports independently, and compare the value against the PICS.
   `"undefined"`, `no-units` and `0` are the three shapes a missed callback
   takes.
3. Diff a new object of a type against the existing one of that type.
   Anything that differs and shouldn't is a callback that matched on
   instance.
4. WriteProperty each commandable output at a priority, re-read it, then
   write NULL to relinquish and confirm it falls back to
   `Relinquish_Default`.
5. Confirm the router announcement services fire: watch the TX log at
   start-up and on the `r` key for `I-Am-Router-To-Network` and
   `Network-Number-Is` on both ports.
6. **Do not test, or claim, cross-network forwarding** — it does not work at
   this stack pin (see [F-ROUTER: router configuration](#f-router-router-configuration)
   above). A test that assumes it works will fail, correctly.
7. Confirm services this device does **not** implement are still rejected —
   AddListElement/RemoveListElement (services 8/9), and any alarm/event,
   COV, or scheduling service.

### Keeping the PICS honest

`docs/PICS.md` is partly generated. `docs/objects.json` describes each
object and who serves which property; the series tool regenerates the
object tables from it plus the stack's own
`docs/property-profile-reference.md` at the pinned commit:

```bash
python tools/gen-objects-properties.py BACnetProfileExample-B-RTR-CPP            # rewrite
python tools/gen-objects-properties.py BACnetProfileExample-B-RTR-CPP --check    # fail if stale
```

(That tool lives in the example-series repository, not in this one. If you
only have this repository, edit the generated block by hand and keep it
matching the callbacks in `main.cpp`.)

When you add an object or a property to `main.cpp`, update
`docs/objects.json` in the same change and regenerate. The `app` list is
what the callbacks serve; `accepted` is for a required property you
deliberately leave to the stack's default, and each one needs a
justification. Anything required, not in `app` and not in `accepted`, comes
out as a ⚠ row - that is a defect, not a feature.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| Start-up prints `BindRouterPort ... second port of networkType=[0] ... issue #304` and `FindPortByNetworkType ... Cannot attribute an incoming NPDU` | **Expected, not your bug.** The pinned stack cannot route between two ports of the same network type (both BACnet/IP here) - see the F-ROUTER section above, `TODO.md`, and [stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037). Each port still works fine on its own network. |
| ReadProperty of `Routing_Table` aborts, or fails with `unknown-property` | Expected - not enabled in this example; see `TODO.md` and [stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038). |
| On start-up the app prints a wall of red `Error:` lines but the device works | **Expected — this is not your bug.** Several benign sources, all from the stack's own debug logging: (1) each port receives its **own** broadcast I-Am and logs a decode cascade — any BACnet/IP device that listens for broadcasts hears itself; (2) the `BindRouterPort`/`FindPortByNetworkType` lines above; (3) a one-time *"UUID has not been set. A UUID must be set for the BACnetSC device to start."* — the stack starts a BACnet/SC datalink this IP-only example never configures. |
| CMake error: *"CAS BACnet Stack adapter not found under: ..."* | Submodules not initialized. Run `git submodule update --init --recursive` (or pass `-D CAS_STACK_DIR=...`). |
| `CASBACnetStackDLL.h: No such file or directory` | Same - submodules not checked out. |
| Windows: *"No CMAKE_CXX_COMPILER could be found"* | Install Visual Studio with the "Desktop development with C++" workload, then re-run from a fresh terminal. |
| First build seems stuck for minutes | Normal - it's compiling ~600 stack files. Only the first build is slow. |
| `Failed to bind UDP port ...` | Another program is using that port. Stop it, or pass a different `--port`/`--port2`. |
| `--port2` equals `--port` | Rejected on purpose - they must be two different UDP ports (two different simulated networks). |
| Client sends Who-Is but sees no I-Am | Firewall is blocking the UDP port, or the client and device are on different subnets (Who-Is is a broadcast). Allow both ports; test on the same subnet first. |
| Replies show an unexpected device instance or vendor | Another BACnet device is already answering on this host/port. Stop the other device, or use `--port`/`--port2`. |
