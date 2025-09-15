+++
title = 'XEP-XXXX: Message Scheduling'
date = 2025-09-14T22:55:52+02:00
draft = true
+++
* **Abstract**: This document defines an XMPP protocol extension for schedule future sends.
* **Authors**: Kompowiec2
* **Copyright**: © 1999 – 2025 XMPP Standards Foundation. SEE LEGAL NOTICES.
* **Status** Active
* **Type**: Standards Track
* **Version** 1.0.0 (2025-09-14)

## 1. Summary

This XEP defines a standard mechanism for scheduling the delivery of XMPP messages at a specific future time or after a specified delay. It provides a client- and server-compatible protocol for scheduling one-off and recurring messages, support for time zones and absolute timestamps, status queries, cancellation, and policy controls for servers and components that choose to implement scheduling. The goal is to enable interoperable "send later" features across XMPP clients and services.

## 2. Scope

This XEP covers:

* An XML namespace and stanzas for scheduling instructions attached to a message or sent as a separate scheduling request to a server/component.
* Timestamp formats and time zone handling (based on XEP-0082 profiles).
* Message lifecycle states (scheduled, queued, sent, failed, cancelled).
* Optional support for recurring scheduled messages.
* Access control and privacy considerations.

This XEP intentionally **does not** mandate server-side persistence policies or retention durations; it provides a framework that server operators may implement according to local policy.

## 3. Conventions and Terminology
The key words MUST, SHOULD, MAY are to be interpreted as described in RFC 2119/8174.

* **Scheduler** — an entity (server, component, or bot) that accepts scheduling requests and performs future delivery.
* **Scheduler-enabled server** — an XMPP server that advertises support for this XEP.
* **Client-side scheduling** — when the client itself holds and sends messages at scheduled times without server help.
* **Scheduled message** — a message with metadata instructing delivery at a specific future time.
## 4. Namespace and Discovery

This XEP uses the namespace `urn:xmpp:scheduling:0`.
A server or component that accepts scheduling requests MUST advertise the feature in service discovery: `urn:xmpp:scheduling:0`.
Clients SHOULD query their server or the dedicated scheduling component for capabilities, including maximum scheduling time window, allowed recurrence types, and quota limits.

## 5. Scheduling Models

Two interoperable models are defined:

### 5.1 Inline Scheduling (Client-originated)

Clients may attach a `<schedule/>` child to the message stanza to indicate that the message SHOULD be delivered at a specified future instant. Example:

```xml
<message to='alice@example.org' type='chat'>
  <body>Happy birthday!</body>
  <schedule xmlns='urn:xmpp:scheduling:0'>
    <deliver-at>2025-12-01T09:00:00Z</deliver-at>
    <timezone>Europe/Warsaw</timezone>
    <id>msg-1234</id>
  </schedule>
</message>
```

If the server supports inline scheduling, it MUST accept the message and take responsibility for delivering it at the requested time. If it does not support scheduling, it SHOULD return a stanza error of type `feature-not-implemented` (see §8: Errors) and the client SHOULD fall back to client-side scheduling.

### 5.2 Scheduling via Scheduler Component (Out-of-band)

A client MAY send a scheduling request to a scheduler service (server or component) using a `set` IQ to the scheduler JID. This model separates scheduling instructions from the message content and is useful when the client wants centralised scheduling (e.g., for multi-device synchronization).

Example request:

```xml
<iq to='scheduler.example.org' type='set' id='s1'>
  <schedule xmlns='urn:xmpp:scheduling:0'>
    <to>alice@example.org</to>
    <body>Happy birthday!</body>
    <deliver-at>2025-12-01T09:00:00Z</deliver-at>
    <timezone>Europe/Warsaw</timezone>
    <id>job-5678</id>
  </schedule>
</iq>
```

The scheduler responds with a `result` IQ containing the assigned job id and optionally metadata (quota, expiry):

```xml
<iq from='scheduler.example.org' type='result' id='s1'>
  <schedule xmlns='urn:xmpp:scheduling:0'>
    <id>job-5678</id>
    <status>scheduled</status>
    <expires>2026-12-01T09:00:00Z</expires>
  </schedule>
</iq>
```

## 6. Time Formats and Time Zones

Timestamps MUST conform to the profile of XEP-0082 (dateTime) with UTC offsets. Servers and clients MUST be prepared to accept and interpret timestamps provided with an explicit timezone or as UTC (`Z`).

For user-friendly display and scheduling, the `<timezone>` element can be included to indicate the originator's timezone (e.g. `Europe/Warsaw`). The scheduler MUST store timestamps in UTC and use the provided timezone only for user display or recurring rule interpretation.

## 7. Elements and Attributes

### 7.1 `<schedule/>` common child elements

* `<id>` — client or server-assigned opaque identifier.
* `<deliver-at>` — an XEP-0082 formatted absolute timestamp in UTC or with offset.
* `<delay-seconds>` — alternative relative delay in seconds (mutually exclusive with `<deliver-at>`).
* `<timezone>` — IANA timezone name (optional).
* `<recurrence>` — optional recurrence rule (see §7.3).
* `<expires>` — optional timestamp after which the scheduler MUST discard the scheduled job if not executed.
* `<status>` — for responses: `scheduled`, `queued`, `sent`, `failed`, `cancelled`.
* `<metadata>` — arbitrary metadata such as tags or delivery hints (server MAY ignore unknown metadata).

Fields not recognized SHOULD be ignored by the receiver.

### 7.2 Security attributes

A `<schedule/>` may include a `signed='true'` attribute to indicate that the scheduling request is cryptographically signed (e.g., using XEP-0374/OpenPGP or XEP-0384/OMEMO). Servers MAY require authentication or signing for scheduling requests depending on policy.

### 7.3 Recurrence

Recurrence rules MAY be expressed using a reduced iCalendar RFC 5545-like format within `<recurrence>`. For interoperability, schedulers SHOULD support at least simple daily/weekly/monthly recurrences and an optional `count` or `until` parameter.

Example:

```xml
<recurrence>FREQ=MONTHLY;BYMONTHDAY=1;COUNT=12</recurrence>
```

Full iCalendar semantics are out of scope, but servers MAY support them and advertise support in disco info.

## 8. Errors

If a server cannot accept the scheduling request, it SHOULD reply with a suitable stanza error. Typical errors include:

* `feature-not-implemented` — the server does not support scheduling.
* `policy-violation` — request exceeds quota or scheduling window.
* `not-allowed` — client lacks permission to schedule to the target JID.
* `bad-request` — malformed timestamp or invalid recurrence rule.

## 9. Status Queries and Management

Clients SHOULD be able to query the status of scheduled jobs via IQ `get` to the scheduler JID:

```xml
<iq to='scheduler.example.org' type='get' id='q1'>
  <schedule-query xmlns='urn:xmpp:scheduling:0'>
    <id>job-5678</id>
  </schedule-query>
</iq>
```

Server response:

```xml
<iq from='scheduler.example.org' type='result' id='q1'>
  <schedule-query xmlns='urn:xmpp:scheduling:0'>
    <id>job-5678</id>
    <status>scheduled</status>
    <deliver-at>2025-12-01T09:00:00Z</deliver-at>
    <created>2025-09-14T11:15:00Z</created>
  </schedule-query>
</iq>
```

### 9.1 Cancellation

To cancel a job, send an IQ `set` with `<cancel/>` containing the job id:

```xml
<iq to='scheduler.example.org' type='set' id='c1'>
  <schedule xmlns='urn:xmpp:scheduling:0'>
    <cancel>job-5678</cancel>
  </schedule>
</iq>
```

Reply with a `result` containing the updated status.

## 10. Delivery Semantics

When the scheduled time arrives, the scheduler MUST deliver the message as if the originating client had sent it at that time. The delivered stanza SHOULD include a `<delay/>` (XEP-0203) only if the message was stored offline after being sent by a client earlier; in the scheduling model, the delivered message SHOULD include a `<scheduled-by/>` element indicating the scheduler JID and job id, to aid client UX in showing that the message was scheduled.

Example delivered stanza:

```xml
<message from='me@example.com' to='alice@example.org' type='chat'>
  <body>Happy birthday!</body>
  <scheduled-by xmlns='urn:xmpp:scheduling:0'>
    <scheduler>scheduler.example.org</scheduler>
    <id>job-5678</id>
    <delivered-at>2025-12-01T09:00:01Z</delivered-at>
  </scheduled-by>
</message>
```

If delivery fails (e.g., target unavailable), the scheduler SHOULD retry according to its policy and MAY send a status update to the originator.

## 11. Privacy and Access Control

Servers and scheduler components MUST authenticate scheduling requests using normal XMPP authentication methods. Servers SHOULD provide controls allowing users to:

* Approve or deny scheduling requests targeted at them (anti-spam).
* Limit who can schedule messages to a user (roster-only, contacts-only, everyone).
* Set quotas on number of scheduled jobs and maximum scheduling window (e.g., no more than 1 year into the future).

Schedulers MUST not expose scheduled message contents to unauthorized parties. For privacy-sensitive content, clients SHOULD use end-to-end encryption (OMEMO or OpenPGP) and sign or otherwise authenticate scheduling requests to prevent tampering.

## 12. Security Considerations

* **Authentication & Authorization:** Only authenticated users SHOULD be permitted to schedule messages. Authorization policies for scheduling to other JIDs MUST be enforced by the scheduler.
* **Replay and Tampering:** Scheduling requests that include message bodies SHOULD be signed or encrypted to prevent tampering between scheduling and delivery. Servers MAY refuse unsigned scheduling of messages for recipients that require encryption.
* **Resource Exhaustion:** Schedulers MUST implement quotas and rate-limiting to avoid DoS via large numbers of scheduled jobs.
* **Privacy Leakage:** Servers that persist scheduled messages are responsible for their secure storage and for complying with jurisdictional retention laws.

## 13. IANA Considerations

This XEP requests the registration of the XML namespace `urn:xmpp:scheduling:0` and that the feature be added to the list of XMPP advertising features.

## 14. Examples

Several usage examples are provided in-line above. Additional examples for recurring schedules, cancellation flows, and error handling are included in an appendix (omitted from this draft).

## 15. Backwards Compatibility

Clients that do not understand scheduling SHOULD ignore `<schedule/>` child elements per normal stanza-ignorance rules. When sending inline scheduling, clients SHOULD query server support first or fallback to client-side scheduling.

# Appendix A: Document Information
* Series: XEP
* Number: XXXX
* Publisher: XMPP Standards Foundation
* Status: Experimental
* Type: Standards Track
* Version: 1.0.0
* Last Updated: 2025-09-14
* Approving Body: XMPP Council
* Dependencies: XMPP Core, XEP-0030, XEP-0060, XEP-0163, XEP-0082, XEP-203, XEP-0374, XEP-384
* Supersedes: None
* Superseded By: None
* Short Name: scheduling

## 16. Open Questions / Implementation Notes

* How far into the future SHOULD servers accept scheduled jobs? (Suggested default: 1 year; server-configurable.)
* Should servers support full iCalendar recurrence rules, or a limited subset tailored to messaging (e.g., simple daily/weekly repeats)?
