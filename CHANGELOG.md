# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
- Live-client (bacpypes3) wire verification: ReadProperty of the Device and
  both Network Port objects directly on each port; Network_Number = 1/2 and
  Network_Number_Quality = configured on the respective ports; WriteProperty
  + readback on Analog Output 1; I-Am-Router-To-Network transmission on both
  ports at start-up and on the `r` key.
