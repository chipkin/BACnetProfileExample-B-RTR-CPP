# TODO / known gaps

## DM-LM-B (AddListElement / RemoveListElement, services 8 / 9) — NOT implemented

The B-RTR profile card calls for services 8 (AddListElement) and 9 (RemoveListElement)
against a list-valued property. The CAS BACnet Stack genuinely implements both services
end-to-end (`source/BACnetAddListElementProcessor.h`, `BACnetRemoveListElementProcessor.h`,
`BACnetBusinessLogic::AddListElement`/`RemoveListElement`) and natively serves them against
a Group object's `List_Of_Group_Members` (`BACnetStack_AddGroupObject`/`AddGroupMember`/
`RemoveGroupMember`, `source/CASBACnetStackDLL.h`) — **but only when the stack is compiled
with `STACK_OPTION_DM_LM_LIST_MANIPULATION` defined**.

Verified against the pinned stack commit `abd4cee1c7f28ca8e1af4720849c4081082bbe82`
(`source/CASBACnetStackOptions.h`, `source/CIBuildSettings.h`,
`projects/msvs/CASBACnetStack/CASBACnetStack.vcxproj`):

- `STACK_OPTION_DM_LM_LIST_MANIPULATION` is commented out by default and is auto-defined
  only when `BACNET_STACK_TESTTOOL` is set (forbidden for this series — customer-facing
  interface only).
- It is **not** included in the `STACK_OPTION_ENABLE_ALL_BIBBS` group that
  `STACK_OPTION_TARGET_FULL` pulls in, so `CIBuildSettings.h` at this pin does not enable it.
- The `ReleaseLib|x64` MSVC configuration (the one `tools/build-stack-static.sh` builds, and
  the only STATIC configuration this series ships) does not define it either — only
  `DebugTestTool|x64` gets `BACNET_STACK_TESTTOOL`.

So the customer-facing STATIC library this series links — built the one way every example
in the series is required to build it (`runbook-update-examples.md` §1: one pin, one build
process) — does not compile in AddListElement/RemoveListElement support. This example
cannot fake that support, so it does not claim services 8/9 or DM-LM-B, and does not add a
Group object.

Filed: <https://github.com/chipkin/cas-bacnet-stack/issues/2033>, asking either for
`STACK_OPTION_DM_LM_LIST_MANIPULATION` to join `STACK_OPTION_ENABLE_ALL_BIBBS` (so a
`STACK_OPTION_TARGET_FULL` customer build gets it, matching that preset's own "every object
type and BIBB" documentation), or for a supported way to opt a customer-facing STATIC build
in without hand-editing the vendored `CIBuildSettings.h` per example (which would break the
series' "one pin, one build" rule).

If a future series-wide stack bump compiles this in, re-verify with
`grep -c "STACK_OPTION_DM_LM_LIST_MANIPULATION" submodules/cas-bacnet-stack/source/CIBuildSettings.h`
and, if present, add a Group object and the two services here.

## F-ROUTER: inter-network NPDU forwarding between the two BACnet/IP ports does NOT work

`BACnetStack_AddRouterPort`, `BACnetStack_AddRouterRoute`, and `BACnetStack_SetRouterEnabled`
all succeed (return `true`) for both Network Port 1 (Vermilion) and Network Port 2
(Vermilion 2), and the `Routing_Table` property on both ports reads back the configured
ports/routes correctly. `BACnetStack_SendIAmRouterToNetwork` / `SendWhoIsRouterToNetwork` /
`SendNetworkNumberIs` all send successfully on both ports too.

**But the stack does not actually forward NPDUs between the two ports**, because both are
`BACnetPacket::NetworkType_IP` and the pinned stack holds exactly ONE internal BACnet/IP
datalink instance regardless of how many router ports of that type exist
(`source/BACnetDataLinkLayer.cpp`, `BindRouterPort`/`FindPortByNetworkType`, hardcoded
`this->m_dataLinkIpv4[0]`). Observed at start-up and on every inbound packet:

```
BindRouterPort() - Error: Router port=[2] is the second port of networkType=[0]; the datalink is
bound to port=[1] and only one instance exists, so ingress attribution is disabled for both.
Inbound routing between them needs the per-type datalink instancing of issue #304.

FindPortByNetworkType() - Error: Cannot attribute an incoming NPDU: [2] routed ports share
networkType=[0]. An explicit ingress port id is needed to route between two ports of the same type.
```

Issue #304 ("Routing: ingress port attribution + more than one datalink per network type") was
closed, but scoped to and implemented for BACnet/SC only (one SC datalink per Network Port
object's `Network_Number`, per its own closing comment); `NetworkType_IP` and `NetworkType_MSTP`
were left exactly where they started — single hardcoded instance, ingress attribution refused.

**Consequence for this example:** a client whose ReadProperty targets a device reachable only
through the OTHER Network Port is NOT correctly routed — the request is not forwarded. This is
the runbook's own wire-verification checklist item for B-RTR ("ReadProperty sent to a B-SS
instance reachable only via net B is correctly routed") and it cannot be demonstrated as working
at this pin. Do not claim end-to-end routing/forwarding for this example; it demonstrates router
CONFIGURATION and ANNOUNCEMENT (NM-RC-B's Who-Is-Router-To-Network/I-Am-Router-To-Network/
Network-Number-Is, plus `Routing_Table`) genuinely, and does not claim inter-network packet
forwarding.

Filed: <https://github.com/chipkin/cas-bacnet-stack/issues/2037>, asking for the same
per-Network-Port-object multi-instance approach #304 shipped for BACnet/SC to be extended to
BACnet/IP and MS/TP.

Re-verify once fixed: run two B-SS-CPP instances, one per network, with this router between
them, and confirm a ReadProperty to the far instance succeeds (README "Verification" documents
the exact non-working steps as of this release).
