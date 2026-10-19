# Fascia Identity contract — changelog

Every published version of [identity.v1.yaml](identity.v1.yaml), its schemas and its vectors,
newest first. A host or rail service vendors a copy pinned to a tag and reads this to decide
whether a diff needs its attention.

| Version | Date | Change | Does your client care? |
| --- | --- | --- | --- |
| `0.4.0` | 2026-10-19 | Credential families. A credential schema may name its `family` (`badge`, `rating` or `identity`) and its `w3c_type` (ending in `Credential`); both are checked when present. `listDocuments` returns each credential schema's `family`. | Only if you publish credential schemas or group credentials for display. Both fields are optional additions. |
| `0.3.0` | 2026-10-19 | Credentials. A new claim type, `credential` (command `/fascia/member/credential`, at most 365 days), naming its type by `schema` and pinning it by `schema_hash`; `may_issue_schemas` limits a steward to some types. Communities publish credential schemas, policy profiles and trust lists: `publishDocument`, `listDocuments`, `getDocument`. `/verify` gains `credentials`. New codes `document_malformed`, `document_invalid`, `document_unknown`. Vectors unchanged. | Only if you issue or accept credentials. The new field is an addition; a client that ignores it keeps working. |
| `0.1.0` | 2026-10-16 | First version. Ten endpoints; the four-part `Fascia-Presentation` header with claims carried once as DAG-CBOR and bound by `claims_sha256`; claims as UCAN 1.0 delegations under the Fascia profile; 47 vectors (8 passing, 39 failing). First published version. | n/a |
