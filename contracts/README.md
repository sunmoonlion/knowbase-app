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

## User data v4 (`user-data/v2`, SDD 0043)

Collected-original delivery returns additive `readings` and `renditions` fields:
only ready kinds and their latest ready rendition number, digest and processor version,
scoped to each beneficiary's material/content version. `has_readings` retains its legacy
meaning (at least one ready kind); it never proves all requested kinds are ready.
New consumers process missing kinds per beneficiary and conservatively reprocess requested
kinds when talking to an older provider without the list. Existing request fields and paths
are unchanged; private copies must not share another beneficiary's processing grant.

- Schema: `user-data/v2/user-data.schema.json`, generated from
  `knowbase-backend/app/app/interfaces/schemas/` (`user_library.py`,
  `user_runs.py`, `user_resolution.py`); `tests/test_user_library_contract.py`
  keeps it byte-identical. `user-data/v1` (per user x app partitions, SDD 0042)
  was removed on 2026-10-28 (decisions ⑰); all apps call v2.
- All paths are under `/api/internal/v2/user-data`. Materials are no longer
  partitioned by app: a user token's `azp` only names the app acting for the
  user, and what an app sees is governed by the user's consent (`app:<app>`).

| Caller | Authorization | What |
| --- | --- | --- |
| The user, via an app backend | user token | materials, versions, descriptions, licences, shelf, uses (`app:*`, `market`, `expert:*`), platform consents (items may be empty: an account-level consent to the process), subscriptions (knowfeed only), processing consent, account, run grants (delegation, session, incubation), conversion of outputs, reads log, renditions, lookup, resolutions, confirmations, preferences (read only) |
| knowlab | `knowbase:user-data-grant-test` | test and platform run grants (a machine rerun grant may also carry the test set's public originals, the author's references and frozen bindings next to the `platform:knowlab:rerun:` items, and may carry none of the latter: `platform_use` names the process then, and the user's consent to that process must still be live; an operator grant carries platform items only), ending platform processes, reading whether a process's consent is still there (`GET /service/platform-processes/{kind}/{ref}?subject=`, read-only, `live` = how many items are still consented; 0 means the user withdrew it in knowbase) |
| knowlab | `knowbase:user-data-directory` | pushing the expert directory, freezing bindings at delivery (each frozen item comes back with its pinned version's `sha256` and the licence at that moment, for the manifest v2 `bindings`; a version is frozen once: freezing it again returns what was frozen the first time, version, licence and digest alike; `preview: true` returns what would be frozen without writing, for the pre-check), checking the bindings before review (`/service/expert-bindings/check`, read-only; items may carry `basis_ref` = `expert:<id>@<version>` to also get the licence frozen with it as `frozen_license`); voiding a freeze that never reached knowmart (`POST /experts/{id}/versions/{v}/freeze/void`, refused with 409 `frozen_in_use` once anyone but the author was granted on it); raising a frozen licence when turning own use public (`POST /experts/{id}/versions/{v}/frozen/{material}/license` with `{author, license, attest: true}`: upward only, the current licence must reach the target, logged in the material's licence history) |
| knowbench, knowlab | `knowbase:user-data-output` | writing the output zone |
| knowfeed | `knowbase:user-data-supply` | pending processing, processing grants, renditions, checking confirmations, checking a subscription scope consent is still live before making new requests |
| knowbench, knowlab, knowfeed | `knowbase:user-data-read` | reading by run grant, releasing grants |
| knowmart | `knowbase:user-data-purchase` | recording a purchase as a holding; checking in bulk whether listed material is still offered (`/service/market-offers/check`, read-only); checking an expert's bindings when a manifest arrives and in the periodic sweep (`/service/expert-bindings/check`, read-only: per item `live` / `deleted` / `disabled` / `gone`, the author's licence, the frozen licence when `basis_ref` is given, and `shareable` = live and at least `via_expert`, the frozen licence included; SDD 0043 section 4.3) |

User tokens may be acting for knowbench, knowlab, knowmart or knowfeed
(`USER_TOKEN_APPS_JSON`). knowfeed only forwards the user's consent to a
subscription's scope (SDD 0043 section 10, 0045 section 2.1): a token acting for
knowfeed reaches `GET /subscriptions` and `PUT`/`DELETE
/subscriptions/{id}/revisions/{n}` only, and gets 403 `user_token_app_denied`
everywhere else.

Readings written back by knowfeed are stored in knowbase's PostgreSQL (SDD 0043
section 5): `POST /processing/grants/{grant}/renditions/text` takes UTF-8 text as
the body, with optional `title` and `published_on` query parameters (one row per
reading version); `.../renditions/pages` takes `[{"page": 1, "text": "..."}]`
(pages from 1, no duplicates, empty text for a page without text; one row per
page). A table reading is written as fact rows (`.../facts`); `.../renditions/table`
only records a failure (`failed_reason`). A reading version's `sha256` is computed
over the canonical content (sorted keys, compact JSON), not over the request body.
Reading a pinned rendition by grant returns the text, or the pages in the same JSON
shape.

Service scopes follow the cross-app `<target>:<relation>` convention. knowbase
checks each relation per caller: `SERVICE_AUTH_<RELATION>_CALLERS` lists the
callers and each caller has `SERVICE_AUTH_<RELATION>_<CALLER>_AUDIENCE` (its
client id), `_SUBJECT_ALLOWLIST` and `_REQUIRED_SCOPE` (`knowbase:<relation>`);
the hyphens in a relation become underscores
(`SERVICE_AUTH_USER_DATA_READ_CALLERS`).

## Data request (`data-request/v1`, SDD 0043 section 3.1)

- Schema: `data-request/v1/data-request.schema.json`, generated from
  `knowbase-backend/app/app/interfaces/schemas/data_request.py`; the vocabulary it
  draws its types and formats from is `data-request/v1/vocabulary.json`, the chart
  of accounts `data-request/v1/chart-of-accounts.json`; digests in
  `data-request/v1/contract-manifest.json`
  (`tests/test_data_request_contract.py` keeps them byte-identical).
- This is the one definition of "one item of data a task needs": subject (or none),
  type, period rule, edition, topics, industry, formats, counts, reading, why, and
  whether it is essential. Since 2026-10-28 (decisions ⑭) the expert manifest no
  longer declares data needs: step 0 of each task, run by the user's own model,
  proposes these items from the expert's logic and the user's question, and the user
  confirms or edits them. knowbench (tasks) and knowlab (tests) write the same models
  and pin the schema digest in a lock; knowfeed pins the vocabulary and the chart. A
  vocabulary change changes the digest.
- Replaces `knowbot-data-needs/v2` (never released).
