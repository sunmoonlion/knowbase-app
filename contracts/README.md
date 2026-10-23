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
| The reading app (knowbench workbench) | service identity with `knowbase:user-data-read`; knowbase maps the service subject to the reading app (`USER_DATA_READER_APPS_JSON`) | `GET /grants/{id}`, `GET /grants/{id}/items/{item}/content`, `GET /grants/{id}/items/{item}/format` |
| knowmart backend | service identity with `knowbase:user-data-purchase` | `POST /purchases` |

Service scopes follow the cross-app `<target>:<relation>` convention, so callers
use the usual `KNOWBASE_INTERNAL_URL` and `KNOWBASE_<RELATION>_SERVICE_*` settings.

Uploads send the file itself as the request body (`?name=` carries the file
name); content responses carry `X-Content-SHA256`. Errors are the usual
`{code, message}` bodies; codes include `item_not_found`, `grant_not_found`,
`consent_not_found`, `in_use`, `sha256_mismatch`, `grant_inactive`,
`format_only`, `outside_consent`, `partition_invalid`, `type_not_allowed`,
`too_large`, `quota_exceeded`, `storage_mismatch`.

Breaking changes require a new major directory.
