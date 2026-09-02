# signbyte

**Open-source qualified electronic signature platform (eIDAS).** Sign, validate and
manage documents with a national eID — built as small services around a
procedures-only data layer, with signed container images for every service.

## The platform, in reading order

| Repository | What it is |
|---|---|
| [signbyte-bff](https://github.com/signbyte/signbyte-bff) | The browser-facing edge — cookie session, no tokens in the browser, DPoP + token exchange to the services behind it |
| [signbyte-spa](https://github.com/signbyte/signbyte-spa) | The portal web app — sign in with a national eID, upload documents, build signing workflows, sign and validate |
| [signflow](https://github.com/signbyte/signflow) | The signing conductor — turns a sign-this-slot request into a completed, validated qualified signature |
| [envelope](https://github.com/signbyte/envelope) | Multi-document, multi-signer workflows — signer slots, signing order, the state machine from draft to completed |
| [document-store](https://github.com/signbyte/document-store) | Single source of truth for document bytes and digests — encrypted storage, ASiC-E assembly, signed-PDF store |
| [eparaksts-signer](https://github.com/signbyte/eparaksts-signer) | The signing service — five eParaksts/Entrust flows producing XAdES, PAdES and ASiC-E signatures at B-LT/B-LTA |
| [signbyte-database](https://github.com/signbyte/signbyte-database) | The PostgreSQL data layer — every schema as Flyway migrations, shipped as one signed migration image |

The reusable infrastructure the platform composes — the authorization server
(`authbyte`), EU trust-list ingestion (`trust-anchor`), the audit sinks, Web eID
certificate verification, document preview — lives in the
[go-make-bytes](https://github.com/go-make-bytes) organisation.

## Run it

The services publish **signed container images** at `ghcr.io/signbyte/<name>` — pin a
version tag or a digest, never a moving branch tag. Start with the data layer:
[signbyte-database](https://github.com/signbyte/signbyte-database)'s README walks a
deployment from an empty PostgreSQL to a migrated platform database, roles included.
Each service README documents its configuration and its place in the whole.

## Licence, contributing, security

**AGPL-3.0** across the organisation — copyright SIA "Go Make Bytes". Issues are
welcome on every repository (see each `CONTRIBUTING.md`). Anything exploitable goes
through **private vulnerability reporting** on the affected repository, never a
public issue — see each `SECURITY.md`.
