# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed — documentation restructure, single-command build

- **README.md cut down to what this example is.** It now covers only the
  B-RTR profile, the device this example creates, the BIBBs/services/objects
  it supports, how to build and run it, and how to verify it. It links to the
  releases page and to the new TUTORIAL.md/docs/PICS.md near the top. Removed:
  the "part of the series" / "you can start here" framing, the generic
  explanation of what a device profile is, the detailed F-MULTIPORT/F-ROUTER
  narrative (moved to TUTORIAL.md), the "Before you ship" table (moved into
  `main.cpp` comments), the "Get the code" and "Link mode" sections, the
  Troubleshooting table (moved to TUTORIAL.md), and the "Objects and
  properties" section (moved to `docs/PICS.md`).
- **New [TUTORIAL.md](TUTORIAL.md)** holds the long-form material the README
  used to carry: extending the example (including the silent-failure warning
  for adding a second object), the F-MULTIPORT and F-ROUTER walkthroughs
  (including the router-forwarding limitation, carried over verbatim), what
  each object type needs the application to serve, who serves what for a
  representative commandable object, how to review a change against the
  conformance statement, and troubleshooting.
- **New [docs/PICS.md](docs/PICS.md)**, a Protocol Implementation Conformance
  Statement in the ANSI/ASHRAE 135 Annex A shape (product description, profile
  claimed, BIBBs, services, segmentation, object types, data link layer,
  address binding, networking options, character sets), with the generated
  objects-and-properties tables as its penultimate section. Section 9
  (networking options) spells out precisely what router configuration does
  and does not do at this stack pin - it does not claim cross-network
  forwarding works.
- **`docs/objects.json`'s Device entry now separates `stack` from `accepted`**,
  matching the shape used across the series: device-wide facts the stack
  computes (`Object_List`, `Protocol_Version`/`Revision`,
  `Protocol_Services/Object_Types_Supported`, `Device_Address_Binding`) are
  now labelled plain `stack` in the generated PICS instead of `stack default,
  accepted`; the actual stack-configured defaults (APDU limits, segmentation,
  system status, database revision) stay under `accepted`. Regenerated with
  `tools/gen-objects-properties.py` - zero ⚠ rows.
- **Build is `cmake -B build -S .` and nothing else, on every platform.** The
  documented build no longer calls `tools/build-stack-static.sh`, which lives
  in the example-series repository and is therefore not available to a
  customer who downloads this repository on its own, and no longer links a
  prebuilt STATIC library. The example now builds in the adapter's default
  SOURCE mode: the stack's sources are compiled into the executable, so there
  is no library to build first. `CMakeLists.txt`, `AGENTS.md` and the release
  workflow were updated to match; the workflow no longer builds/caches the
  STATIC library or carries matrix `lib:` entries, configures without a
  link-mode flag, asserts `CAS_BACNET_STACK_LINK=SOURCE` (was `STATIC`), sets
  `"link_mode": "SOURCE"` in the published metrics JSON, and packages
  `TUTORIAL.md`/`docs/PICS.md` alongside the binary.
- **The "Before you ship" guidance moved into `main.cpp`**, as a comment next
  to each constant in the `CHANGE ALL OF THIS BEFORE YOU SHIP` block,
  including the warning that `DEVICE_NAME` is a compile-time constant here and
  must be made per-unit configurable (serial number, DIP switches, config
  file, or a `--deviceName` argument) in a real product, and that the two
  network numbers must be unique across the whole internetwork like the
  device instance.
- **Footprint table now documents the SOURCE-mode build.** The v1.0.0 numbers
  were measured from a STATIC-linked build; the table now notes that the next
  release refreshes them under the documented SOURCE build, matching the
  series convention.

## [1.0.0] - unreleased

### Added

- First release: implements the **B-RTR (Router)** profile - a stand-alone
  BACnet router with two BACnet/IP Network Port objects ("Vermilion",
  "Vermilion 2") and physical inter-datalink routing between them.
- BIBBs: DS-RP-B, DS-WP-B, DM-DDB-A/B, DM-DOB-B, NM-RC-B (partial - see below).
- Objects: Device 389018 "Rainbow"; Analog/Binary/Multi-State Input 1
  ("Bronze"/"Emerald"/"Hot Pink"); commandable Analog/Binary/Multi-State
  Output 1 ("Chartreuse"/"Fuchsia"/"Indigo"); two Network Ports ("Vermilion" /
  network 1, "Vermilion 2" / network 2).
- Seeded from `BACnetProfileExample-B-ASC-CPP` minus DeviceCommunicationControl
  (DM-DCC-B), per the profile card.
- **F-MULTIPORT** (series-canonical): uses `common/` 2.3.0's multi-port
  `SetupUDP(port, networkPortInstance)` / `SendIAm(deviceInstance,
  networkPortInstance)` overloads to own two independent UDP sockets, one per
  Network Port.
- **F-ROUTER** (series-canonical): `BACnetStack_AddRouterPort`,
  `BACnetStack_AddRouterRoute` (one demonstration-only route),
  `BACnetStack_SetRouterEnabled`, `BACnetStack_SendIAmRouterToNetwork` (also
  bound to the new `r` key, `common/` 2.4.0), `BACnetStack_SendWhoIsRouterToNetwork`,
  `BACnetStack_SendNetworkNumberIs`.
- Pinned to CAS BACnet Stack `6.x` @ `abd4cee1` (reports 6.0.21), linked as a
  prebuilt STATIC library (`tools/build-stack-static.sh`).
- Vendors `common/` 2.5.0 (adds `PROPERTY_IDENTIFIER_ROUTING_TABLE` and the
  `r` key on top of 2.3.0's multi-port sockets).

### Known gaps (see `TODO.md`)

- **DM-LM-B (AddListElement/RemoveListElement, services 8/9) not implemented.**
  The customer-facing STATIC build of the pinned stack does not compile in
  `STACK_OPTION_DM_LM_LIST_MANIPULATION`. [stack issue #2033](https://github.com/chipkin/cas-bacnet-stack/issues/2033)
- **Inter-network NPDU forwarding does not work.** The pinned stack holds one
  internal datalink instance per network type; two BACnet/IP router ports
  cannot both attribute ingress traffic, so nothing is actually forwarded
  between the two networks, even though router configuration
  (`AddRouterPort`/`AddRouterRoute`/`SetRouterEnabled`) succeeds.
  [stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037)
- **`Routing_Table` cannot be read.** Enabling it and reading it aborts the
  request; the pinned stack has no `UseCallbackGetList` case for Network
  Port/Routing_Table outside a test-tool build. Not enabled in this example.
  [stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038)

### Verified

- STATIC build, zero warnings from `main.cpp`/`common/`.
- Live-client wire verification: ReadProperty of the Device and
  both Network Port objects directly on each port; Network_Number = 1/2 and
  Network_Number_Quality = configured on the respective ports; WriteProperty
  + readback on Analog Output 1; I-Am-Router-To-Network transmission on both
  ports at start-up and on the `r` key.
