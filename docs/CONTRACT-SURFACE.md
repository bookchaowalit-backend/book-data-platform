# Contract surface: book-data-platform

This page documents the API, event, CLI, storage and configuration surface that
`book-data-platform` owns or consumes. The machine-readable list is `interfaces` in
[`contract.json`](../contract.json); `scripts/check.py` fails when an interface
listed there is missing from this page.

- Platform: `data` (contract `book-platform.contract.v1`, API `v1`)
- Status: `scaffolded`; source status: `current-implementation-in-solo-empire`
- Current implementation: `solo-empire` at `infra/scripts/data_lake`, `systems/ai-data/data-lake-architecture.md`
- Depends on: `security`, `observability`

Interfaces marked `current-in-solo-empire` are served by the `solo-empire`
control plane today. This repository does not serve them yet; they are the
compatibility surface a migration must preserve. The `source` column is a path
in the implementing repository.

## Interfaces

| ID | Kind | Direction | Interface | Contract | Auth | Source | Status |
|---|---|---|---|---|---|---|---|
| `data.cli.ingest` | cli | provides | `python infra/scripts/data_lake/ingest.py --input --source --domain --dataset` | - | - | `infra/scripts/data_lake/ingest.py` | current-in-solo-empire |
| `data.object-store.landing` | object-store | provides | `landing/source=<source>[/provider=<p>]/batch_id=<id>/payload.<ext>` | - | - | `infra/scripts/data_lake/ingest.py` | current-in-solo-empire |
| `data.object-store.bronze` | object-store | provides | `bronze/domain=<d>/dataset=<ds>/schema_version=<v>/ingest_date=<date>/source=<s>/part-<batch>.parquet` | - | - | `infra/scripts/data_lake/parquet.py` | current-in-solo-empire |
| `data.object-store.manifest` | object-store | provides | `control/manifests/source=<source>/batch_id=<id>.json` | - | - | `infra/scripts/data_lake/ingest.py` | current-in-solo-empire |
| `data.object-store.checkpoint` | object-store | provides | `control/checkpoints/source=<source>/batch_id=<id>.json` | - | - | `infra/scripts/data_lake/ingest.py` | current-in-solo-empire |

## Behaviour notes

Derived from the current source; re-check the source before changing a
consumer.

- `ingest.py` writes three immutable artifacts per batch: exact source bytes
  (landing), a Parquet Bronze envelope, and a control manifest; a checkpoint
  allows a retry to resume after a partial outage. No SQLite or downstream
  operational table is written.
- Bronze columns (all strings): `event_id`, `source`, `source_record_id`,
  `domain`, `dataset`, `schema_version`, `received_at`, `event_time`,
  `content_type`, `raw_object_key`, `raw_sha256`, `payload_json`,
  `metadata_json`, `ingest_run_id`, `privacy_class`, `retention_class`.
- `event_id` is the SHA-256 of `source` plus the source record id (or the
  canonical payload/raw hash), so replaying a batch is idempotent.
- Input formats: `auto`, `json`, `jsonl`, `ndjson`, `csv`, `html`, `binary`.
  `--privacy-class` is one of `public`, `internal`, `private` (default),
  `restricted`; `--retention-class` defaults to `operational`.

## Migration gates

- bronze parity
- schema versioning
- replay evidence

## Out of scope

No production provider integration, credential, database writer or customer
payload lives in this repository. Record parity, privacy and rollback evidence
in the parent platform registry before any cutover.
