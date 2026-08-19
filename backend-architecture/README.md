# Backend Architecture Fundamentals

A short primer on how real backend systems are actually structured, meant to run **before**
[The Practitioner Track](../PRACTITIONER-TRACK.md) — you can't reason well about attack surface,
trust boundaries, or blast radius in a system whose shape you don't understand yet. Security is
downstream of architecture: every "confused deputy," every "internal network isn't a trust
boundary," every "this service can see too much" finding in the security curriculum assumes you
already know what a service boundary *is*.

This is deliberately compact — a working map of the concepts, not a full systems-design course.
Each file ends with how it specifically feeds into the security material.

## Contents

1. [Monoliths, Modular Monoliths, Microservices](01-monoliths-and-microservices.md) — the three
   real shapes a backend takes, and why "just use microservices" is usually the wrong first answer.
2. [Layered Architecture & Service Boundaries](02-layered-architecture-and-boundaries.md) — API →
   Business → Repository → Database, and why the dependency direction is the whole point.
3. [Synchronous Communication: REST & RPC](03-synchronous-communication.md) — request/response,
   timeouts, retries, circuit breakers — what happens when the thing you called doesn't answer.
4. [Asynchronous Communication: Queues & Events](04-asynchronous-communication.md) — message
   queues, pub/sub, event-driven architecture, and why "at-least-once" delivery is the default.
5. [WebSockets & Real-Time Communication](05-websockets-and-realtime.md) — how a WebSocket
   actually differs from HTTP polling, and what changes when a connection stays open.
6. [API Gateways & the BFF Pattern](06-api-gateways-and-bff.md) — the front door every request
   walks through, and why one size rarely fits every client.
7. [Data Consistency Across Services](07-data-consistency.md) — the CAP theorem in practice,
   eventual consistency, and the saga pattern for transactions that span services.

## How to use this

Read in order — each file assumes the last. Budget a weekend, not a month; this is orientation,
not mastery. When you're done, move to
[The Practitioner Track](../PRACTITIONER-TRACK.md#phase-0--calibration) — Phase 0 there will make
much more sense with this vocabulary already in hand.

---
**Next:** [1. Monoliths, Modular Monoliths, Microservices →](01-monoliths-and-microservices.md)
