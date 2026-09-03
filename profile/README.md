# signbyte

**Open-source qualified electronic signature platform (eIDAS).** Sign, validate and
manage documents with a national eID — built as small services around a
procedures-only data layer, with signed container images for every service.

## How it fits together

![The signbyte platform. A green border marks a service created by Go Make Bytes; all green marks a qualified trust service reused rather than built; a plain border marks another service signbyte needs in order to run. The browser reaches only the signbyte-bff boundary through the edge; the boundary calls seven application services with delegated DPoP-bound tokens, while web-eid is reached only by authbyte, trust-anchor only by web-eid, and eidas-audit only by the event stream. Only eparaksts-signer reaches the qualified trust services. Underneath sits the shared platform every service stands on: Redis, PostgreSQL, NATS JetStream, an S3 object store and Gotenberg.](signbyte-platform.png)

Everything with a **green border** was built by Go Make Bytes. Everything **all green** is a
qualified trust service we reuse rather than rebuild. A **plain border** is a service signbyte needs
in order to run, but that we did not write.

**One public API.** The browser holds a cookie and never a token. `signbyte-bff` holds the session,
the tokens and the sender-constraint key, and composes every downstream call with a delegated,
DPoP-bound token (RFC 8693). It calls seven services. Three more take no call from it at all:
`web-eid` is reached only by `authbyte`, `trust-anchor` only by `web-eid`, and `eidas-audit` only by
the event stream — nothing the browser sends can address them.

**Two audit sinks, on purpose.** Both are fed by every service above. Signing evidence travels
through NATS, so a slow sink never slows a signature. Personal-data access is written synchronously,
because a read that was not recorded must not have happened.

**One path for the document.** `document-store` is the only holder of document bytes. `signflow`
takes a digest from it and never the document itself; `previewbyte` reads it on the user's behalf and
renders inside a sandbox; `eparaksts-signer` is the only service allowed to reach the qualified
provider, and it holds no keys and no bytes. The anonymous verification page is the one path that
skips `signflow` — the boundary forwards a rate-limited upload straight to the signer under its own
identity, with no user attached.

**The platform underneath is standard.** PostgreSQL behind stored procedures: one database, ten
migration locations, an EXECUTE-only login role per service, and the schema shipped as one signed
image from `signbyte-database`. Redis for session and flow state. NATS JetStream for audit and
domain events. Any S3-compatible object store for encrypted document bytes, reached by one service
only. Gotenberg to turn office documents into PDF before the same sandboxed render path as any other
file. None of it is bespoke and all of it is swappable; the roles, the schema, the streams and the
bucket are created at deploy time, never by a running service.

**Every service is observable the same way.** Traces and metrics over OTLP, one JSON line per event
to stdout — one collector, one Grafana, no per-container digging.

## Every signature qualified. None of the trust services rebuilt.

signbyte is a complete e-signing platform — upload, envelopes, signer order, preview, signing,
validation, long-term archival — built as twelve small services around one deliberate decision: the
qualified trust services stay with LVRTC. We hold no signing keys, issue no certificates and stamp no
timestamps. eParaksts does that, as it already does for the Latvian state. signbyte is the product,
the workflow, and the evidence around it.

**4** signing flows — card, phone, cloud, seal · **3** audit regimes, kept
separate on purpose · **0** access tokens ever reach the browser · **B-LTA** signature level, XAdES ·
PAdES · ASiC-E

### The qualified half is already Latvia's

Becoming a qualified trust service provider takes years, an audit regime and an HSM estate. signbyte
does not attempt it. Every cryptographically qualified act — certificate issuance, key custody,
timestamping, trust-list publication — happens inside services LVRTC already runs under eIDAS
supervision. That is not a gap in the product; it is the shortest credible route to a legally binding
signature.

| Reused, not rebuilt | What it provides |
|---|---|
| **eParaksts identity provider** — LVRTC | Plain OAuth 2.0 authorization code — not OpenID Connect — against eParaksts' own identification scope. Two login methods are accepted, **eParaksts Mobile** and **eID Scan**; anything else fails closed, and eID cards sign in through Web eID instead. The same consent also returns the person's signing identities, so a later signature needs no second authorisation |
| **SignAPI + EU DSS** — LVRTC | The signature spine: container packaging to XAdES, PAdES and ASiC-E, qualified timestamping, and validation against the EU trust lists. Validation reports come back verbatim — relayed, never reinterpreted |
| **TrustedX and the QSCD** — LVRTC | eParaksts Mobile, eID Scan and qualified e-seals. The signing key lives in the qualified device; the person confirms on eParaksts' own screen with their own passcode |
| **Certificate status (OCSP)** — LVRTC | Live revocation checks on the eID certificate presented at login and at card signing. A revoked card fails closed |
| **EU List of Trusted Lists** — European Commission | The registry of every qualified provider in the Union. signbyte ingests it, verifies its signature, and derives the trusted-CA set every trust decision is made against |

A signature produced through signbyte is created by the same qualified devices and stamped by the
same qualified authority as one produced on eparaksts.lv — verifiable by any EU validator,
independently of us. **If signbyte disappeared tomorrow, every signed container would still validate,
because the evidence is inside the file, not inside our database.**

## Four ways to sign, one spine

Each flow varies exactly one step — how the signature value is obtained. Everything around it
(session, digest calculation, encoding normalisation, finalisation, delivery) is the same code path,
which is why a new flow does not mean a new set of bugs.

| Flow | What the signer uses | Where the key lives | Result |
|---|---|---|---|
| `webEid` | eID card in a reader, PIN entered in the browser | On the card, never transmitted | QES |
| `eidScan` | Phone push with a verification code shown on screen | Qualified device at the provider | QES |
| `eparakstsMobile` | eParaksts Mobile approval, batch-capable | Qualified device at the provider | QES |
| `eparakstsMobileEseal` | Organisation seal identity, same approval | Qualified device at the provider | QSeal |

**Signature levels.** Every signature is produced at B-LT — signature timestamp plus revocation data
embedded, so the file proves itself without calling anyone. An archive timestamp lifts it to B-LTA
for long-term preservation, and can be re-applied in place as algorithms age.

**Confidential documents.** A caller that cannot release document bytes can sign by hash: signbyte
registers the digest and never receives the content. The signed result is assembled on the caller's
side.

## Three audit regimes, because three laws ask different questions

A single audit log cannot serve three masters. A court asks whether a signature legally happened. A
data protection officer asks who looked at a person's file. A cyber-security regulator asks when you
first knew. Those answers need different integrity, different retention and different readers — so
signbyte runs three mechanisms rather than one pipeline with three taps.

| | Signing evidence | Personal-data access | Security operations |
|---|---|---|---|
| **The question it answers** | Did this signature legally happen? | Who accessed whose data, and why? | When did you first become aware? |
| **Basis** | eIDAS Regulation (EU) 910/2014; ETSI EN 319 102-1, 319 132 / 142 / 162 | GDPR Art. 5(2) accountability, Art. 30 records, Art. 15 access, Art. 32 security, Art. 33/34 breach | NIS2 Directive (EU) 2022/2555, Art. 23 |
| **Integrity** | Hash chain, append-only, update and delete revoked in the database | Per-row seal plus sealed period checkpoints — tamper-evident and still purgeable | Write-once log store, integrity-protected incident register, synchronised time |
| **Retention** | The legal lifetime of the document, plus legal hold | A configurable accountability window; legal hold blocks purge | The forensic and regulatory minimum |
| **Output** | Per-envelope evidence package; validation report on demand | Subject access export: everything held, and every access to it | Detection, triage, and the 24-hour / 72-hour / one-month notification timeline |

The differences are load-bearing. Signing evidence travels asynchronously, so audit can never slow a
signature, and it is hash-chained so tampering is visible. Access records are written synchronously,
because an accountability record that can be dropped is worthless — and sealed per row rather than
chained, so the log can still be purged at the end of its window without destroying the proof of what
remains. Security telemetry is high-volume and goes to the operator's own SIEM, where the incident
clocks are run.

**Two breach clocks, not one.** One incident can trigger both regimes, to different authorities, on
different timelines. signbyte keeps the two evidence trails separate, so each notification is
answered from the right source and the shorter clock is not missed.

**Audit is not an add-on here.** Each service emits the events its regimes require from its first
commit, and a service is not considered done until those events are observed arriving in the sink —
not assumed. Audit cannot be retrofitted: anything a service did before the emitter existed is
permanently unauditable.

## Small services are a security posture, not a fashion

Bytes live in one place, keys in another, evidence in a third, and each one is reachable only from
where it needs to be.

- **`document-store`** is the only service that holds document content. Envelope encryption: a fresh
  per-object AES-256 data key seals the bytes, the data key is stored only in wrapped form, and the
  ciphertext goes to S3-API object storage. The key-management seam takes an external KMS without
  touching the storage layer.
- **`previewbyte`** is the only service that opens untrusted bytes, and the parser runs inside a
  WebAssembly sandbox — a malicious file gets a sandbox, not a process. Output is inert page images
  plus a text layer: no scripts, no embedded actions, no remote fetches.
- **`eparaksts-signer`** is the only service allowed to leave the network toward the qualified
  provider, and it holds no signing key of its own. The private key never leaves the card, the phone
  or the provider's HSM.
- **The data tier** takes not a single line of application SQL. Services call stored procedures
  through a role that may execute and nothing else — no table access, so a compromised service
  cannot read the tables. Audit tables are append-only at the database level, with update and delete
  revoked.
- **Nine shared libraries** carry the cross-cutting behaviour: logging with always-on redaction,
  correlation and tracing, token issue and validation, card validation, container building, the
  upload gate, the validation answer, and the three audit emitters. A control fixed in the library is
  fixed in every service on the next build.

signbyte is not a qualified validation service and does not pretend to be one. The controls described
here are the ones implemented in the services; edge protection, the secret store, network policy and
the SIEM are deployment-tier components that live in the operating environment. Notifications and
self-service subject-access tooling are not built yet.

## The platform, in reading order

The order follows the diagram above: left to right, top to bottom.

| Repository | Where it lives | What it is |
|---|---|---|
| [signbyte-spa](https://github.com/signbyte/signbyte-spa) | `signbyte` | The portal web app — sign in with a national eID, upload documents, build signing workflows, sign and validate |
| [signbyte-bff](https://github.com/signbyte/signbyte-bff) | `signbyte` | The browser-facing edge — cookie session, no tokens in the browser, DPoP + token exchange to the services behind it |
| [authbyte](https://github.com/go-make-bytes/authbyte) | `go-make-bytes` | The authorization server — authorization code with PKCE, DPoP-bound tokens, step-up and federated logout; issues every user and service token |
| [web-eid](https://github.com/go-make-bytes/web-eid) | `go-make-bytes` | The card engine — validates the eID card's login token and the signature made with its signing certificate, against a managed trusted-CA set and live revocation |
| [trust-anchor](https://github.com/go-make-bytes/trust-anchor) | `go-make-bytes` | EU trust-list ingestion — verifies the List of Trusted Lists and the national lists, and publishes the trusted-CA set every trust decision is made against |
| [envelope](https://github.com/signbyte/envelope) | `signbyte` | Multi-document, multi-signer workflows — signer slots, signing order, the state machine from draft to completed |
| [signflow](https://github.com/signbyte/signflow) | `signbyte` | The signing conductor — turns a sign-this-slot request into a completed, validated qualified signature |
| [document-store](https://github.com/signbyte/document-store) | `signbyte` | Single source of truth for document bytes and digests — encrypted storage, ASiC-E assembly, signed-PDF store |
| [previewbyte](https://github.com/go-make-bytes/previewbyte) | `go-make-bytes` | Document rendering — PDF, image, text and office files become inert page images plus a text layer, with the parser inside a WebAssembly sandbox |
| [eidas-audit](https://github.com/go-make-bytes/eidas-audit) | `go-make-bytes` | The signing-evidence sink — consumes signing events from the durable stream into an append-only, hash-chained record |
| [access-audit](https://github.com/go-make-bytes/access-audit) | `go-make-bytes` | The personal-data access record — sealed on write and indexed by data subject, so a subject access request can actually be answered |
| [eparaksts-signer](https://github.com/signbyte/eparaksts-signer) | `signbyte` | The signing service — the eParaksts signing flows, producing XAdES, PAdES and ASiC-E signatures at B-LT/B-LTA |
| [signbyte-database](https://github.com/signbyte/signbyte-database) | `signbyte` | The PostgreSQL data layer — every schema as Flyway migrations, shipped as one signed migration image |

Two organisations, one platform. `signbyte` holds what is specific to signing documents;
[go-make-bytes](https://github.com/go-make-bytes) holds the parts that are not — an authorization
server, EU trust-list ingestion, card verification, document rendering and the audit sinks are useful
to anything that needs them, so they are built to be reused rather than tied to this product. The
shared Go libraries all of these are built from live in [gmb-lib](https://github.com/gmb-lib).

## What it needs to run

Nothing bespoke, and all of it swappable:

- **PostgreSQL** — one database, ten migration locations, an EXECUTE-only login role per service
- **Redis or Valkey** — session and flow state only, in three separable logical databases
- **NATS JetStream** — durable audit and domain events
- **An S3-compatible object store** — encrypted document bytes; reached by `document-store` alone
- **Gotenberg** — office documents to PDF, before the same sandboxed render path as any other file;
  reached by `previewbyte` alone

Roles, schema, streams and bucket are all created at deploy time, never by a running service.

## Run it

The services publish **signed container images** at `ghcr.io/<organisation>/<name>`, under whichever
of the two organisations the repository lives in — pin a version tag or a digest, never a moving
branch tag. Start with the data layer:
[signbyte-database](https://github.com/signbyte/signbyte-database)'s README walks a
deployment from an empty PostgreSQL to a migrated platform database, roles included.
Each service README documents its configuration and its place in the whole.

## Licence, contributing, security

**AGPL-3.0** across the organisation — copyright SIA "Go Make Bytes". Issues are
welcome on every repository (see each `CONTRIBUTING.md`). Anything exploitable goes
through **private vulnerability reporting** on the affected repository, never a
public issue — see each `SECURITY.md`.
