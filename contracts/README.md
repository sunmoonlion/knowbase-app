# Knowledge Provider Contracts

This directory is the only editable source of truth for Knowledge ingestion and
retrieval provider contracts. Generated fixtures and downstream client models
must not be edited as substitute schemas.

## Artifact ingestion

- Current compatible major: `artifact/v1`
- Request schema: `artifact/v1/knowfeed-knowbase-artifact.schema.json`
- Release manifest and digest: `artifact/v1/contract-manifest.json`
- Producer: Knowfeed App
- Provider/consumer: Knowledge App

The v1 contract accepts exactly one immutable, versioned `s3://` object. The
provider resolves it with its own read-only identity and verifies storage
version, size, media type and SHA-256 before any RAGFlow upload. Database IDs are
lineage only; the provider never calls back into the Info database to resolve an
artifact.

Breaking changes require a new major directory and a dual-version migration
window. CI publishes the schema together with its SHA-256 digest.

## Retrieval and citation

- Current compatible major: `retrieval/v1`
- Request schema: `retrieval/v1/knowledge-retrieval-request.schema.json`
- Provider response schema: `retrieval/v1/knowledge-retrieval-response.schema.json`
- Browser-safe citation schema: `retrieval/v1/citation.schema.json`
- Release manifest and digests: `retrieval/v1/contract-manifest.json`
- Provider: Knowledge App
- Service consumer: Research App

Clients submit stable `dataset_key` values only. RAGFlow dataset, document and
chunk identifiers are private provider bindings and are never domain identity.
Every Evidence item is traceable through a stable Knowledge document/version
to an Info document/version. Browsers receive only the citation projection,
including an authorized same-origin source path instead of a raw provider URL.

Breaking changes require a new major directory. Additive optional response
fields remain compatible, but consumers must pin the manifest digests and run
consumer-driven contract tests before accepting a provider release.

## User data (per user x app knowledge bases, SDD 0042)

- Current compatible major: `user-data/v1`
- Schemas: `user-data/v1/user-data.schema.json` (generated from
  `knowbase-backend/app/app/interfaces/schemas/user_data.py`; the test
  `tests/test_user_data_contract.py` keeps them byte-identical)
- Release manifest and digest: `user-data/v1/contract-manifest.json`
- Provider: knowbase-app; service consumers: knowbench-app, knowlab-app, knowmart-app

All paths are under `/api/internal/v1/user-data` and reachable only from inside
the cluster. Two kinds of caller:

| Caller | Authorization | Endpoints |
| --- | --- | --- |
| The user, via an app backend | `Bearer <user token>`: the app exchanges the user's Casdoor access token for one with `aud` = knowbase (RFC 8693); `sub` is the user, `azp` names the app partition | `POST/GET /items`, `GET /items/{id}`, `GET /items/{id}/content`, `DELETE /items/{id}`, `POST /consents`, `DELETE /consents/{id}`, `POST /grants` (delegation, session, incubation), `POST /grants/{id}/revoke` |
| knowlab backend | service identity with `knowbase:user-data-grant-test` | `POST /test-grants`, `POST /test-grants/{id}/revoke` |
| An app moving its own old data in (knowlab, once) | service identity with `knowbase:user-data-import`; the verified caller is the app it imports for | `POST /imports?owner=&name=&ref=` (body is the file; only into the caller's own app partition; idempotent by `ref`, same `ref` with other content is 409 `origin_ref_reused`) |
| The reading app (knowbench workbench) | service identity with `knowbase:user-data-read`; the verified caller is the reading app | `GET /grants/{id}`, `GET /grants/{id}/items/{item}/content`, `GET /grants/{id}/items/{item}/format`, `GET /grants/{id}/items/{item}/check` (same checks as a read, no content: `{item_id, sha256}`; a reader calls it before serving anything it cached from an earlier read, and it is logged as `check`), `POST /grants/{id}/release` (hand back its own grant when the delegation or session ends; idempotent) |
| knowmart backend | service identity with `knowbase:user-data-purchase` | `POST /purchases` |

Service scopes follow the cross-app `<target>:<relation>` convention, so callers
use the usual `KNOWBASE_INTERNAL_URL` and `KNOWBASE_<RELATION>_SERVICE_*` settings.
knowbase checks each relation per caller, the same way knowmart does:
`SERVICE_AUTH_<RELATION>_CALLERS` lists the callers and each caller has
`SERVICE_AUTH_<RELATION>_<CALLER>_AUDIENCE` (its client id),
`_SUBJECT_ALLOWLIST` and `_REQUIRED_SCOPE` (`knowbase:<relation>`); the hyphens
in a relation become underscores (`SERVICE_AUTH_USER_DATA_READ_CALLERS`). The
caller found this way must be knowbench, knowlab or knowmart, and reads and
imports act for that app.

Uploads send the file itself as the request body (`?name=` carries the file
name); content responses carry `X-Content-SHA256`. Errors are the usual
`{code, message}` bodies; codes include `item_not_found`, `grant_not_found`,
`consent_not_found`, `in_use`, `sha256_mismatch`, `grant_inactive`,
`format_only`, `outside_consent`, `partition_invalid`, `type_not_allowed`,
`too_large`, `quota_exceeded`, `storage_mismatch`.

Withdrawing a consent (`DELETE /consents/{id}`) only stops grants issued after it;
grants already issued from that consent stay valid until they are handed back,
revoked or expire (decided 2026-10-25, Fable review D03). Grants keep their
`consent_id`, so revoking them along with the consent can be added later if a
user-facing "withdraw consent" ever needs it.

Breaking changes require a new major directory.

## User data v4 (`user-data/v2`, SDD 0043)

- Schema: `user-data/v2/user-data.schema.json`, generated from
  `knowbase-backend/app/app/interfaces/schemas/` (`user_library.py`,
  `user_runs.py`, `user_resolution.py`); `tests/test_user_library_contract.py`
  keeps it byte-identical. Not released yet: it lives next to `user-data/v1`,
  which keeps working unchanged until the apps switch over (SDD 0043 section 9).
- All paths are under `/api/internal/v2/user-data`. Materials are no longer
  partitioned by app: a user token's `azp` only names the app acting for the
  user, and what an app sees is governed by the user's consent (`app:<app>`).

| Caller | Authorization | What |
| --- | --- | --- |
| The user, via an app backend | user token | materials, versions, descriptions, licences, shelf, uses (`app:*`, `market`, `expert:*`), platform consents, subscriptions (knowfeed only), processing consent, account, run grants (delegation, session, incubation), conversion of outputs, reads log, renditions, lookup, resolutions, confirmations, preferences (read only) |
| knowlab | `knowbase:user-data-grant-test` | test and platform run grants, ending platform processes |
| knowlab | `knowbase:user-data-directory` | pushing the expert directory, freezing bindings at delivery |
| knowbench, knowlab | `knowbase:user-data-output` | writing the output zone |
| knowfeed | `knowbase:user-data-supply` | pending processing, processing grants, renditions, checking confirmations |
| knowbench, knowlab, knowfeed | `knowbase:user-data-read` | reading by run grant, releasing grants |
| knowmart | `knowbase:user-data-purchase` | recording a purchase as a holding |

## Data needs (`knowbot-data-needs/v2`, SDD 0043 section 3.1)

- Schema: `knowbot-data-needs/v2/data-needs.schema.json`, generated from
  `knowbase-backend/app/app/interfaces/schemas/data_needs.py`; the vocabulary it
  draws its types and formats from is `knowbot-data-needs/v2/vocabulary.json`;
  digests in `knowbot-data-needs/v2/contract-manifest.json`
  (`tests/test_data_needs_contract.py` keeps all three byte-identical).
- This is the one definition of a data need. knowlab (writing declaration
  templates), knowmart (manifest v2) and knowbench (resolving and checking data)
  each write the same models and pin the schema digest in a lock; a vocabulary
  change changes the digest.
