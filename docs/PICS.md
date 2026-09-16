# BACnet Protocol Implementation Conformance Statement (PICS)

For the **BACnet B-RTR (Router) C++ example** -
see [README.md](../README.md).

> This is the PICS **for the example as shipped**. It describes a tutorial
> device announcing itself as a Chipkin demo, not a product. When you turn this
> example into your own device, this document is one of the things you rewrite:
> the vendor, model and version rows all come from the
> `CHANGE ALL OF THIS BEFORE YOU SHIP` block at the top of `main.cpp`. The
> example has **not** been submitted for BTL certification.

## 1. Product description

| | |
|---|---|
| **Vendor Name** | Chipkin Automation Systems |
| **Vendor Identifier** | 389 |
| **Product Name** | CAS BACnet Stack Example - B-RTR |
| **Product Model Number** | CAS BACnet Stack Example - B-RTR |
| **Application Software Version** | 1.0.0 |
| **Firmware Revision** | 1.0.0 |
| **BACnet Protocol Version** | 1 |
| **BACnet Protocol Revision** | 24 |

**Product Description:** a stand-alone BACnet router built on the CAS BACnet
Stack. It owns two BACnet/IP Network Port objects, one per network it joins,
and the shared minimum/commandable object pattern (three read-only inputs,
three writable commandable outputs) used across this example series. It
configures router ports and routes (`AddRouterPort`/`AddRouterRoute`) and
announces itself as a router (`I-Am-Router-To-Network`,
`Who-Is-Router-To-Network`, `Network-Number-Is`), but at the pinned stack
commit it does **not** forward NPDUs between the two networks - see section 9
below. It is a tutorial for implementers of the B-RTR profile.

## 2. BACnet standardized device profile (Annex L)

**B-RTR - BACnet Router.**

This device claims exactly one profile. Because the B-RTR requirements are a
superset of B-GENERAL's, a conformant B-RTR device also satisfies
**B-GENERAL** (Annex L.8); that is subsumption, not a second claim.

## 3. BIBBs supported (Annex K)

| BIBB | Description | Supported |
|---|---|:---:|
| DS-RP-B | Data Sharing - ReadProperty - B | yes |
| DS-WP-B | Data Sharing - WriteProperty - B | yes |
| DM-DDB-A | Device Management - Dynamic Device Binding - A (initiates Who-Is) | yes |
| DM-DDB-B | Device Management - Dynamic Device Binding - B | yes |
| DM-DOB-B | Device Management - Dynamic Object Binding - B | yes |
| NM-RC-B | Network Management - Router Configuration - B | partial - router configuration and announcement are implemented and verified; inter-network NPDU **forwarding** is not, at this stack pin (see section 9) |
| DM-LM-B | Device Management - List Manipulation - B (AddListElement/RemoveListElement) | **no** - the customer-facing STATIC build of the pinned stack does not compile in `STACK_OPTION_DM_LM_LIST_MANIPULATION` ([TODO.md](../TODO.md), [stack issue #2033](https://github.com/chipkin/cas-bacnet-stack/issues/2033)) |

No other BIBBs are supported. In particular this device does **not** support
DS-RPM-B (ReadPropertyMultiple), DS-COV-B, any alarm and event (AE-*) BIBB,
scheduling (SCHED-*), trending (T-*), or DM-DCC-B
(DeviceCommunicationControl) - unlike this example's B-ASC seed, this profile
omits DM-DCC-B.

## 4. Application services supported

| Service | Initiate | Execute |
|---|:---:|:---:|
| ReadProperty | no | **yes** |
| WriteProperty | no | **yes** |
| Who-Is | **yes** | **yes** |
| I-Am | **yes** | - |
| Who-Has | no | **yes** |
| I-Have | **yes** | - |
| Who-Is-Router-To-Network | **yes** | not verified (see section 9) |
| I-Am-Router-To-Network | **yes** | - |
| Network-Number-Is | **yes** | - |
| AddListElement / RemoveListElement | no | no - not implemented (DM-LM-B, see above) |

An unsolicited I-Am is broadcast on **both** networks at start-up (DM-DDB-A/B).
I-Am-Router-To-Network and Network-Number-Is are likewise broadcast on both
networks at start-up and again whenever the `r` key is pressed; this is the
router's polite start-up announcement, not a response to a request.

Any other confirmed service is rejected. That rejection is part of the
profile boundary, not a limitation to work around.

## 5. Segmentation capability

Segmentation is **not supported** in either direction
(`Segmentation_Supported` = `no-segmentation`). `Max_APDU_Length_Accepted` is
1476 octets, the BACnet/IP maximum, on both Network Ports.

## 6. Standard object types supported

No object is dynamically creatable or deletable.

| Object type | Instance | Object_Name | Writable | Optional properties supported |
|---|:---:|---|:---:|---|
| Device | 389018 | Rainbow | no | Description |
| Analog Input | 1 | Bronze | no | - |
| Binary Input | 1 | Emerald | no | - |
| Multi-State Input | 1 | Hot Pink | no | State_Text |
| Analog Output | 1 | Chartreuse | **yes** (commandable) | - |
| Binary Output | 1 | Fuchsia | **yes** (commandable) | - |
| Multi-State Output | 1 | Indigo | **yes** (commandable) | - |
| Network Port | 1 | Vermilion | no | - |
| Network Port | 2 | Vermilion 2 | no | - |

The device instance is configurable at run time with `--deviceID` (BACnet
requires the device instance to be configurable).

## 7. Data link layer options

**BACnet/IP (Annex J)**, two independent instances on one host, distinguished
by UDP port: Network Port 1 (Vermilion) on `--port` (default 47808), Network
Port 2 (Vermilion 2) on `--port2` (default `--port` + 1 = 47809).

BBMD is not supported, Foreign Device registration is not supported, and
BACnet/SC, MS/TP, Ethernet (Annex H) and PTP are not supported.

## 8. Device address binding

Static device binding is **not supported**. The device answers Who-Is/Who-Has
and additionally initiates Who-Is (DM-DDB-A) to discover other devices; it
does not otherwise bind a peer address ahead of time.

## 9. Networking options

**This device is a router** (two directly-connected BACnet/IP networks, one
per Network Port) - but read this section precisely before relying on it.

**Works and is verified on the wire, against this build:**

- Router **configuration**: `BACnetStack_AddRouterPort` (both ports),
  `BACnetStack_AddRouterRoute` (one demonstration-only static route to network
  3, one hop beyond port 2), and `BACnetStack_SetRouterEnabled` all succeed.
- `Network_Number` / `Network_Number_Quality` read back correctly on both
  Network Ports (`1`/`configured` and `2`/`configured`).
- **I-Am-Router-To-Network** and **Network-Number-Is** transmission - both
  confirmed sent (TX on the wire) on both ports at start-up and on the `r`
  key.
- ReadProperty and WriteProperty work correctly on **each network
  independently** - a client on network 1 (port 47808) or network 2 (port
  47809) gets correct answers from this device directly.

**Does NOT work at this stack pin - do not rely on these:**

- **Inter-network NPDU forwarding.** A request addressed to a device reachable
  only through the *other* Network Port is **not forwarded** between the two
  networks. The pinned stack holds exactly one internal BACnet/IP datalink
  instance regardless of how many router ports of that type exist, so it
  cannot attribute inbound traffic to the correct egress port. This is the
  single most important limitation of this example: it demonstrates router
  **configuration and announcement**, not working packet forwarding. See
  [TODO.md](../TODO.md) and
  [stack issue #2037](https://github.com/chipkin/cas-bacnet-stack/issues/2037).
- **`Routing_Table` (cl. 12.56.62) read.** Deliberately not enabled - verified
  on the wire that reading it aborts the request outside a test-tool build of
  the stack. See [TODO.md](../TODO.md) and
  [stack issue #2038](https://github.com/chipkin/cas-bacnet-stack/issues/2038).
- **This device answering an incoming Who-Is-Router-To-Network from a peer
  router.** Only the outbound (start-up / `r`-key) direction has been
  exercised against this build.
- **The demonstration route to network 3.** `AddRouterRoute` accepting the
  call is confirmed; no device sits on network 3 in this example series, and
  an actual forwarded packet over that route is not demonstrated (and could
  not be, given the forwarding gap above).

Neither BBMD nor Foreign Device functionality is implemented by this router.

## 10. Character sets supported

UTF-8 (ANSI X3.4). Supporting a character set does not imply the device can
handle data in all character sets.

## 11. Objects and properties

<!-- OBJECTS-PROPERTIES:BEGIN (generated by tools/gen-objects-properties.py from docs/objects.json - do not edit here) -->
Every object this example creates, and every REQUIRED property of each (per ANSI/ASHRAE 135-2024 clause 12 and the stack's `docs/property-profile-reference.md`), plus the optional properties the example turns on. **Served by** says who answers a ReadProperty: the **stack** generates it, or the **app** serves it from a `GetProperty*` callback in `main.cpp`. A ⚠ row is a required property the app does not serve and the stack would fill with a default - that is a defect, not a feature.

### Device 389018 "Rainbow" - vendor 389 (Chipkin Automation Systems); instance configurable with --deviceID. The stack rows are device-wide facts only the stack knows - the protocol version and revision it implements, the services and object types it was configured with, the live object list and address-binding table. The accepted rows are the stack's configured defaults for APDU limits, segmentation, system status and database revision; an application that answered them from its own constants could contradict the stack, so this example does not

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
| Protocol_Version | Unsigned | stack | no |
| Protocol_Revision | Unsigned | stack | no |
| Protocol_Services_Supported | BACnetServicesSupported | stack | no |
| Protocol_Object_Types_Supported | BACnetObjectTypesSupported | stack | no |
| Object_List | BACnetARRAY[N] of BACnetObjectIdentifier | stack | no |
| Max_APDU_Length_Accepted | Unsigned | stack default, accepted (`CAS_BACNET_DEVICE_DEFAULT_MAX_APDU_LENGTH_ACCEPTED`) | no |
| Segmentation_Supported | BACnetSegmentation | stack default, accepted (`BACnetSegmentation::noSegmentation`) | no |
| APDU_Timeout | Unsigned | stack default, accepted (`CAS_BACNET_DEVICE_DEFAULT_APDU_TIMEOUT`) | no |
| Number_Of_APDU_Retries | Unsigned | stack default, accepted (`CAS_BACNET_DEVICE_DEFAULT_NUMBER_OF_APDU_RETRIES`) | no |
| Device_Address_Binding | BACnetLIST of BACnetAddressBinding | stack | no |
| Database_Revision | Unsigned | stack default, accepted (Generic UnsignedInteger default: `0`) | no |
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |

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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |

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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |

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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |

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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |
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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |
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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |
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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |

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
| Property_List | BACnetARRAY[N] of BACnetPropertyIdentifier | stack | no |

<!-- OBJECTS-PROPERTIES:END -->

## 12. References

- ANSI/ASHRAE Standard 135-2024, Annex A (PICS template), Annex K (BIBBs),
  Annex L (device profiles), Clause 6.6 (routers), Clause 12 (object types).
- [README.md](../README.md) - what this example is and how to build it.
- [TUTORIAL.md](../TUTORIAL.md) - how to extend it, and how to keep this
  document honest when you do.
- [TODO.md](../TODO.md) - the router-forwarding and DM-LM-B gaps in detail,
  with the stack issues tracking them.
