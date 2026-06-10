# IDC-API v2 — Analysis

## What it is

This repo is the **server-side REST API for NCI's Imaging Data Commons (IDC)** — the
"official" HTTP API that backs the documented endpoints at
`api.imaging.datacommons.cancer.gov`. It's a Google Cloud project, maintained by ISB
(Institute for Systems Biology), deployed via Google Cloud Endpoints + App Engine Flex.

Its job: let a programmatic client **build a cohort from DICOM metadata filters and get
back a manifest** (lists of studies/series/files with their GCS/AWS URLs and counts), plus
expose supporting metadata (IDC versions, collections, analysis results, filterable
attributes, allowed field values).

The active endpoints (`openapi-appengine.v2.pub.yaml`, registered in `api/__init__.py`):

- `GET /v2/versions`, `/collections`, `/analysis_results`, `/filters`,
  `/filters/values/{filter}`, `/fields/{version}` — metadata discovery
- `POST /v2/cohorts/manifest/preview` + `GET .../preview/nextPage` — the core: submit
  filters, page through a manifest
- `GET /v2/about`, `/swagger` — docs/landing

Notably, the **stateful cohort endpoints are disabled**: `cohorts_bp` and `user_bp` are
commented out (`api/__init__.py`), and the spec's `/cohorts`, `/cohorts/{cohort_id}`,
`/users/account_details` paths have no live implementation. So in practice the API today is
a **stateless cohort-preview/manifest service** — there is no "save a cohort to my account"
capability wired up.

## How it's implemented

It's a thin **Flask** app (blueprints per version, gunicorn with 3 workers, 70s timeout)
that is mostly an **orchestrator/proxy in front of two backends**:

1. **The IDC webapp** (`settings.BASE_URL`, the `canceridc.dev` portal). For metadata, the
   API just forwards to the webapp's internal `collections/api/v2/...` endpoints with a
   shared `X-API-AUTH` service token (`metadata_views.py`, `auth.py`). For cohorts, it POSTs
   the validated filterset to the webapp's `cohorts/api/v2/preview/query/` endpoint, **which
   returns a generated SQL string** (`manifest_views.py`).

2. **BigQuery.** The API takes that SQL and runs it directly against BigQuery via a low-level
   `BigQuerySupport` wrapper (`bq_support.py`), polling the job, paging results, and packing
   rows into JSON (`manifest_views.py`). Pagination is a synthetic
   `jobId:location:pageToken` triple so clients can resume long queries.

The genuinely interesting / fragile part is **SQL post-processing via string surgery**.
Because the webapp generates the SQL, the API rewrites it textually to:

- inject `COUNT(DISTINCT ...)` columns at the right hierarchy level for `counts`
  (`manifest_utils.py`),
- force-add then strip a `Modality` column to coax well-formed SQL out of the generator
  (`manifest_views.py`, `manifest_utils.py`),
- handle `StudyDate`/`StudyDescription` aggregation by slicing the query into
  SELECT/FROM/WHERE/GROUP/ORDER substrings and reassembling them.

Input safety relies on JSON-schema validation of the filterset (`schemas/filters.py`), an
enum check of field names, and a regex blacklist (`<script>`, `<iframe>`, …). The actual
values reach BigQuery as **parameterized** query params, which is good; the regex blacklist
on top is a weak secondary layer.

Dependencies are pinned-but-stale and heavy (`requirements.txt` — full Google API stack),
and the `Dockerfile` still installs **MySQL + m2crypto + xmlsec** even though none of the
live code touches a database or SAML — leftover from the ISB-CGC lineage this was forked
from (copyright headers say "2019, Institute for Systems Biology", routes still reference
cohort/user concepts from that codebase).

## Utility, pros and cons for IDC users

### Pros

- **Language-agnostic.** Any HTTP client in any language gets IDC manifests — the only
  first-class option for non-Python consumers (web apps, R, Java, Go, JS).
- **Server-side cohort logic.** The complex filter→SQL translation lives on the server and
  is shared with the portal, so an API manifest matches what the website shows. Clients
  don't need to know the BigQuery schema.
- **No GCP account / billing needed by the caller.** BigQuery is executed under IDC's
  service account; the user just hits a URL.
- **Always current.** Queries hit live BigQuery, so the freshest IDC version and the full
  DICOM attribute set are queryable, including fields not shipped in any local index.
- **Stable hosted contract** with Swagger UI and OpenAPI spec.

### Cons

- **Operationally fragile.** SQL-by-string-manipulation is brittle: any change in how the
  webapp generates SQL can silently break `counts`/`StudyDate`/modality handling. There's no
  SQL parser — it's regex and substring offsets.
- **Latency and timeouts.** Every call is API→webapp→BigQuery (cold BQ jobs, 70s gunicorn
  timeout, 202-and-retry semantics). Manifests of large cohorts page slowly. Compare to
  idc-index answering most queries in milliseconds locally.
- **Tight coupling to private webapp internals.** The API can't function standalone; it
  depends on undocumented `collections/api/v2` and `cohorts/api/v2/preview/query` endpoints
  and a shared secret token. That's a single point of failure and a maintenance tax.
- **No download story.** It returns manifests (URLs), not data. The user still needs
  s5cmd/gsutil/idc-index to actually download. So it's half a workflow.
- **Reduced surface.** Saved cohorts and user accounts are advertised in the spec but not
  deployed — confusing for users reading the docs.
- **Maintenance signals.** The git log is dominated by dependency bumps and
  `testing_branch.py` churn; the Dockerfile carries dead MySQL/SAML weight; there's a stray
  bug (`logger.error(f"... {e}")` referencing undefined `e` in `metadata_routes.py`). It
  reads like a maintained-but-not-actively-developed component.

## Contrast with idc-index (the Python API)

These are two fundamentally different philosophies for the same goal:

| | **IDC-API (this repo)** | **idc-index** |
|---|---|---|
| Shape | Hosted REST service | `pip install` Python library |
| Where compute runs | Server: webapp + BigQuery | Client: a downloaded Parquet index queried locally with DuckDB/pandas |
| Metadata coverage | **Full** live DICOM schema, any IDC version | A **curated subset** of commonly used columns (the index), plus optional extended indices |
| Freshness | Live BigQuery, always current | Pinned to the index version shipped with the package |
| Latency | Network + BQ job per query | Milliseconds, offline after first index download |
| Download | Returns URLs only | Built-in: wraps **s5cmd** to actually pull DICOM from AWS/GCS |
| Audience | Any language, web/portal integrations | Python/data-science/ML users |
| Failure modes | Service availability, timeouts, webapp coupling | Index staleness, local disk/memory for big queries |
| Auth | None for caller (server holds token) | None |

In short: **idc-index optimizes the data-scientist's end-to-end loop** (query + download,
fast, reproducible, pinned) and is now the de-facto recommended path for IDC users.
**IDC-API optimizes reach and freshness** — it's what you reach for when you're *not* in
Python, when you need a column idc-index doesn't ship, or when you're building a
service/portal that needs server-side cohort semantics. They're complementary, but for the
typical "I want CT scans from collection X" task, idc-index is faster, simpler, and
self-contained, which is why this REST API has drifted toward maintenance mode.

## Suggestions for improvement

### Strategic

1. **Decide and document the niche.** Position IDC-API explicitly as "the non-Python /
   live-BigQuery / web-integration path" and point Python users to idc-index in the README
   (which currently is one line). Stop advertising disabled cohort/user endpoints, or finish
   them.
2. **Consider sharing the index.** idc-index already publishes a Parquet manifest; the API
   could serve manifests *from the same index* for the common case and fall back to BigQuery
   only for columns outside it — cutting latency and the webapp dependency dramatically.

### Architectural / correctness

3. **Replace SQL string surgery with structured query building.** The `counts`, `StudyDate`,
   `Modality`-injection, and substring slicing in `manifest_utils.py` are the biggest risk.
   Either have the webapp return a structured query object (columns, filters, group-by) the
   API composes safely, or build the SQL in the API from the validated filterset directly
   using `sqlglot`/the BQ client query builder. This removes the most fragile code and the
   fragile coupling at once.
4. **Decouple from webapp internals.** The metadata endpoints are pure pass-throughs to
   `collections/api/v2`. Either cache those responses (they change only per IDC release) or
   read the same source the webapp reads, so an API call doesn't fan out to another service
   synchronously.
5. **Fix the latent bugs:** undefined `e` in the `metadata_routes.py` exception handler;
   `categorical_values` builds a `bigquery.Client('idc-dev-etl')` it never uses and hardcodes
   a project; `is_job_done` returns a 202 dict whose shape differs from the success path,
   complicating clients.

### Operational / hygiene

6. **Slim the Dockerfile.** Drop MySQL, m2crypto, xmlsec, swig, build-essential reinstalls —
   none of the live code needs them. This cuts image size, build time, and CVE surface
   massively.
7. **Modernize dependency management.** Move to a lockfile (`uv`/`pip-tools`) instead of a
   hand-pinned `requirements.txt` mixing exact pins and ranges; several pins are old. Add
   Dependabot.
8. **Add real CI tests against a live or recorded backend.** The `tests/` tree has many
   `_test_*`/`testx_*` disabled files; wire up the active ones in CircleCI (it currently only
   builds/deploys) with recorded webapp/BQ fixtures so SQL-rewriting regressions are caught.
9. **Add response caching + rate limiting** on the metadata endpoints (CDN/`Cache-Control`),
   since they're effectively static between releases.

### Developer experience

10. **Publish a thin client** (or OpenAPI-generated SDKs) and richer examples, and surface
    the `next_page`/202-retry pattern clearly — it's the part most likely to trip up API
    consumers.
