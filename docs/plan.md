# Plan (STUB): B-RTR (Router) — C++ example

> **STATUS: STUB.** Seed facts below. Expand from
> [`bacnet-profile-plan-template.md`](../../bacnet-profile-plan-template.md) after the
> sample plans ([B-LD](../../BACnetProfileExample-B-LD-CPP/docs/plan.md),
> [B-BC](../../BACnetProfileExample-B-BC-CPP/docs/plan.md)) are reviewed.

**Profile:** B-RTR · **Family:** Annex L.7 (Miscellaneous) · **Role:** B ·
**Archetype:** Infrastructure · **Difficulty:** 4/5 · **Phase:** B (deferred — infrastructure)

**Thesis:** a BACnet router — forwards messages between **two BACnet networks**
(e.g. two BACnet/IP networks, or BACnet/IP ↔ MS/TP). Requires **≥2 Network Port
objects** — the largest structural deviation from the single-port baseline.

## Required BIBBs (profiles.md L.7)
`DS-RP-B, DS-WP-B; DM-DDB-A, DM-DOB-B, DM-LM-B, NM-RC-B`.

## Services to enable
- ReadProperty (1), WriteProperty (15), + network-layer router services.

## Objects (baseline + )
- A **second Network Port** on a different network number. The router forwards
  between the two and answers Who-Is-Router-To-Network / I-Am-Router-To-Network.

## Shared features
- **DEFINE:** F-ROUTER (NM-RC-B router config + DM-LM-B list manipulation;
  DM-DDB-A I-Am-Router-To-Network initiate).
- **REUSE:** F-OUTPUTS (B-SA writes).

## Known stack gaps
- Confirm the standard DLL's **routing API** and how `AddNetworkPortObject`
  composes with a second network. profiles.md: ✅ S65 (286 `*Router*:*Network*`
  gtests cover every Clause-6 router-config / network-layer service).

## Notes / open questions
- Master plan §7 risk 6: the baseline has one Network Port; routing needs two and a
  routing table. **Spike the multi-port + routing API before writing the plan's
  code section** — this is the most structurally different B-role example.
