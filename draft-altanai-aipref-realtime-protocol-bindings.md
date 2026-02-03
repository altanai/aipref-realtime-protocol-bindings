---
title: "AI Preferences for Real-Time Protocol Bindings"
abbrev: "AI Pref RT Bindings"
category: std

docname: draft-altanai-aipref-realtime-protocol-bindings-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "AI Preferences"
keyword:
 - AI preferences
 - realtime communications
 - SIP
 - media control
venue:
  group: "AI Preferences"
  type: "Working Group"
  mail: "ai-control@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ai-control/"
  github: "altanai/aipref-realtime-protocol-bindings"
  latest: "https://altanai.github.io/aipref-realtime-protocol-bindings/draft-altanai-aipref-realtime-protocol-bindings.html"

author:
 -
    fullname: "Altanai Bisht"
  organization: "Cisco Meraki"
  email: "albisht@cisco.com"

normative:
  RFC2119:
  RFC3261:
  RFC3264:
  RFC3311:
  RFC6665:
  RFC8174:
  RFC8830:

informative:
  RFC3265:
  RFC4566:
  RFC7587:
  RFC8831:
  I-D.aipref-network-privacy-control:

...

--- abstract

This document defines how Artificial Intelligence (AI) preference expressions are bound to signaling and media protocols used for real-time, session-based communications such as the Session Initiation Protocol (SIP) and associated Session Description Protocol (SDP) offers. It specifies a reusable binding model, concrete SIP header field conventions, and SDP attributes that allow endpoints, intermediary services, and AI assistants to advertise, negotiate, and enforce requirements about AI-driven processing of session metadata, media control events, and telemetry. The goal is to align real-time protocol behavior with the AI Preferences (AIPREF) vocabulary without disrupting existing call control semantics.

--- middle

# Introduction

Real-time communications (RTC) deployments are increasingly assisted by AI-driven components that evaluate signaling metadata, media control messages, and quality of experience (QoE) measurements. Examples include AI-based call routing, automated troubleshooting, adaptive encoding, or compliance monitoring. When these components operate on user or service provider data, the AI Preferences (AIPREF) working group requires that stakeholders can express and enforce preferences that describe what AI systems may inspect, retain, or export.

Existing AIPREF documents define preference vocabularies and repository behavior, but do not specify how those preferences are conveyed through session control protocols. This document fills that gap for SIP-based systems and applies the same pattern to other RTC bindings that reuse SIP constructs (including SIP events, PUBLISH, and WebRTC gateways). The binding guidance is protocol-agnostic where possible so that additional RTC protocols (such as XMPP Jingle or proprietary session controllers) can follow the same pattern.

## Goals

The binding framework MUST:

1. Preserve backwards compatibility with SIP user agents, gateways, and middleboxes that are unaware of AI preference signaling.
2. Permit endpoints and administrative domains to advertise locally enforced AI policies and to consume peer policies before AI processing begins.
3. Support mid-dialog updates so that AI processing can adapt when session context changes (e.g., escalation from audio to video, invoking transcription services, or triggering automated remediation workflows).
4. Allow binding of AI preferences to both signaling-layer artifacts (message bodies, header fields, event packages) and media-layer descriptions (SDP attributes, record routes, and telemetry streams).

## Scope

This document describes:

- A binding model that maps AIPREF vocabulary elements to SIP dialog state and SDP descriptions.
- A compact token and URI-based encoding carried in SIP header fields and bodies.
- Procedures for preference discovery, negotiation, and error handling across dialogs, subscriptions, and conferencing primitives.
- Operational recommendations for intermediaries, policy servers, and AI enforcement points.

This document does not standardize AI algorithms, privacy-preserving enforcement techniques, or the semantics of individual AIPREF vocabularies beyond referencing existing definitions in other working group documents such as [I-D.aipref-network-privacy-control].

## Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

The terminology defined in RFC3261, RFC3264, and the AIPREF framework documents applies. This document additionally uses the following conventions:

- **AI Preference Token (APT)**: A compact, possibly signed representation of one or more AIPREF statements, retrievable via URI or carried inline.
- **Preference Attacher**: The SIP entity that injects an APT into signaling or SDP (e.g., originating user agent, outbound proxy, or application server).
- **Preference Enforcement Point (PEP)**: A network element or AI component that validates and enforces APT requirements prior to processing protected data.
- **Real-Time Preference Channel**: Any mechanism that conveys updated APTs after a dialog is established (e.g., re-INVITE, UPDATE, INFO, SUBSCRIBE/NOTIFY pair).

# Binding Requirements

The following requirements guide bindings for SIP and related RTC protocols:

- **Visibility**: APTs SHOULD be visible to the entities that are expected to enforce them, including downstream AI assistants. When hop-by-hop protection (e.g., TLS) is applied, intermediaries outside the trust domain MUST NOT rely on APTs they cannot validate.
- **Integrity**: APTs SHOULD be signed or integrity-protected when transported across untrusted proxies to prevent unauthorized downgrades.
- **Idempotency**: Repeated delivery of identical APTs MUST NOT cause state divergence. Implementations MAY treat the most recent valid APT as authoritative.
- **Granularity**: Bindings MUST allow preferences scoped to dialogs, media sections, or individual features (e.g., telemetry streams, AI feature lists). APTs therefore include `target` metadata identifying the scope (dialog-id, media mid, or subscription identifier).
- **Fallback**: Endpoints that cannot satisfy received preferences MUST reject or redirect the request using standard SIP procedures (e.g., `488 Not Acceptable Here`) and SHOULD include diagnostic information.

# Binding Model

The model in Figure 1 illustrates how preferences flow through RTC signaling.

```
 ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
 │ Preference   │ APT  │ SIP Binding  │ APT  │ Enforcement  │
 │ Originator   ├─────▶│ Transport    ├─────▶│ Point / AI   │
 └──────────────┘      └──────────────┘      └──────────────┘
         ▲                      │                     │
         │ Publish / Fetch      │ Dialog signaling    │ Media / Data
         └──────────────────────┴─────────────────────┘
```

1. An originator (user agent, operator policy engine, or regulator) issues an APT identifier or token.
2. The preference attacher embeds the token using one or more bindings defined in this document.
3. Enforcement points inside AI workloads retrieve, validate, and apply the referenced constraints before consuming protected data.

Bindings MUST reference the same canonical APT identifier when expressing preferences across signaling and media layers to avoid ambiguity.

# SIP Signaling Binding

## AI-Pref Header Field

This document defines the `AI-Pref` SIP header field. Each instance conveys metadata that allows receivers to retrieve or validate an APT.

```
AI-Pref  = "AI-Pref" HCOLON pref-value *(COMMA pref-value)
pref-value = pref-id *(SEMI pref-param)
pref-id  = token / quoted-string ; identifier or opaque token
pref-param = ("scope" EQUAL token) /
             ("type" EQUAL token) /
             ("version" EQUAL 1*DIGIT) /
             ("integrity" EQUAL token) /
             ("uri" EQUAL LAQUOT absoluteURI RAQUOT)
```

### Usage Rules

1. **Initial INVITE**: The originating user agent server (UAC) SHOULD include an `AI-Pref` header referencing all preferences that govern AI handling of dialog metadata or media diagnostics. The header MAY appear multiple times when different scopes are advertised (e.g., `scope=dialog`, `scope=media`).
2. **Provisional Responses**: Proxies and user agent servers (UAS) MAY add `AI-Pref` headers to responses in order to enumerate additional policies that downstream AI components MUST accept before the session is confirmed.
3. **ACK and PRACK**: These messages MUST echo the latest accepted `AI-Pref` identifiers when the UAS requires confirmation. Absence of `AI-Pref` in ACK implies acceptance of the most recent set.
4. **Mid-Dialog Updates**: Re-INVITE and UPDATE requests MUST include `AI-Pref` whenever the preference scope of any media stream is modified. This allows AI systems that adapt encoders, perform transcription, or modify layouts to obey updated constraints.
5. **SUBSCRIBE/NOTIFY**: Event packages (e.g., dialog package, KPML, presence) MAY carry `AI-Pref` headers so that AI assistants consuming event payloads adhere to the advertised constraints.

### Compact Tokens and URIs

`pref-id` values can represent:

- Inline tokens that encode the full preference statement (e.g., CBOR Web Token referencing vocabulary keys).
- Handles that require dereferencing via HTTPS using the `uri` parameter.
- Versioned identifiers that map to entries in a policy repository.

Receivers MUST treat unknown parameters according to RFC3261 rules (ignore them) and MUST NOT assume that the absence of `AI-Pref` implies permission to process data without AI constraints.

### Error Handling

- If a UAS cannot comply with mandatory preferences, it SHOULD reply with `488 Not Acceptable Here` and include a `Warning` header value of `399 aipref "Preference unsupported"`.
- When integrity verification fails, the recipient SHOULD respond with `403 Forbidden` and MAY include diagnostic details in a `Reason` header referencing `AI-Integrity-Failure`.
- Gateways that strip `AI-Pref` MUST insert a `History-Info` entry explaining the removal so downstream entities understand why enforcement data is missing.

## SIP Body Considerations

When APTs are too large for header fields, this document RECOMMENDS embedding them inside a multipart body part with media type `application/aipref+json` or `application/aipref+cbor` and referencing that part via `Content-ID`. This allows richer metadata (e.g., signed manifests) without overloading SIP header processing.

# SDP and Media Binding

## `a=aipref` Attribute

An SDP media description MAY include one or more `a=aipref` attributes, each binding a specific media stream to an APT identifier.

```
a=aipref:<scope> <identifier> [<parameter>=<value> ...]
```

Valid scopes include `session`, `group`, and `mid`. Parameters align with the AI Pref vocabulary, for example:

- `features=media-metrics,rtcp-xr`
- `retention=24h`
- `export=aggregated-only`

Endpoints MUST ensure that SDP attributes remain consistent with SIP-level `AI-Pref` headers. If SDP renegotiation removes an attribute, the corresponding AI processing MUST stop or transition to the remaining allowed scope.

## RTP Control and Telemetry

RTP control protocols such as RTCP, RTCP XR, and RTP/RTCP extensions for feedback may carry AI-relevant telemetry. Implementations SHOULD map telemetry streams to the same APT identifiers declared in SDP by including the identifier in RTCP SDES items or header extensions defined for this purpose. When that is not feasible, implementations MAY rely on the dialog-level `AI-Pref` scope while documenting the implicit association.

# Preference Discovery and Synchronization

## Retrieval via HTTPS

Preferences referenced via `uri` MUST be retrievable over HTTPS with mutual authentication when sensitive. Servers SHOULD support conditional requests and caching so intermediaries can reuse validated APTs across multiple dialogs.

## Repository Interaction

Policy repositories MAY expose a REST interface where clients submit dialog metadata (Call-ID, From-tag, To-tag) and receive the authoritative list of applicable APT identifiers. This pattern is especially useful for large conferencing services where centralized policy engines coordinate AI workloads.

## Conflict Resolution

When multiple APTs apply to the same resource, the following precedence rules apply unless a policy repository states otherwise:

1. User-specific preferences override domain defaults.
2. Domain-level regulatory requirements override individual relaxations.
3. The most restrictive constraint wins when two preferences conflict on the same vocabulary key.

Endpoints MAY advertise their conflict-resolution policy through the `policy` parameter inside `AI-Pref` headers (e.g., `policy=strictest-wins`).

# Operational Considerations

- **Logging**: Preference attachers SHOULD log emitted APT identifiers alongside Call-ID values to support audits and incident response.
- **Testing**: Interoperability testing MUST verify that dialogs proceed when `AI-Pref` is absent, ensuring that legacy devices remain compatible.
- **Scaling**: Implementations SHOULD compress header fields using SIP over HTTP/3 (RFC9397) or similar transports when large preference sets are common.
- **Federation**: Peering domains MAY translate local preference identifiers into a shared namespace. Translation MUST NOT weaken constraints without explicit consent from the originator.

# Security Considerations

Preferences often contain sensitive information about user intent, regulatory exposure, or organizational policy. Therefore:

- Transport security such as TLS (for SIP over TLS, WebSocket, or HTTP/3) MUST be used whenever an `AI-Pref` header or body carries tokens that could be replayed or tampered with.
- APT signatures SHOULD be validated before AI systems act on the encapsulated instructions. Validation includes issuer authentication, expiration checks, and revocation status.
- Implementations MUST guard against downgrade attacks where a malicious intermediary strips `AI-Pref` headers. Techniques include SIPS-only routing, end-to-end integrity with S/MIME, or redundant signaling through policy repositories.
- Preference tokens SHOULD minimize personally identifiable information. Instead of embedding explicit user identifiers, use pseudonymous handles that map to access-controlled directories.
- Systems MUST treat AI enforcement failures as security incidents when they result in unauthorized data processing. Telemetry SHOULD be rate-limited to avoid revealing preference patterns to attackers probing the signaling fabric.

# IANA Considerations

IANA is requested to perform the following actions.

## SIP Header Field Registration

Register the `AI-Pref` header field in the "Header Fields" sub-registry under "Session Initiation Protocol (SIP) Parameters" with the following values:

- Header Name: AI-Pref
- Compact Form: none
- Reference: This document

## SDP Attribute Registration

Register the `aipref` attribute in the "att-field (media level only)" registry defined by RFC4566 with the following values:

- Attribute name: aipref
- Type of attribute: media / session
- Subject to charset: no
- Purpose: Associates SDP media sections with AI preference tokens
- Reference: This document

# Acknowledgments
{:numbered="false"}

This work is informed by discussions within the AIPREF working group, including contributions on network privacy controls and media quality automation.
