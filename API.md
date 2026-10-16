# Fascia Identity API

The identity service of the Digital Community Passport (DCP). It serves the public keys and
revocation lists that let anyone check a passport holder's membership claims, verifies a
presented request for a host that does not run its own verifier, and holds the register of
communities.

It issues nothing and stores no member's key or claim. Claims pass from a steward's phone to a
member's phone; the service first sees one when a member presents it, and checks it with public
keys alone.

**Contract:** [identity.v1.yaml](identity.v1.yaml). Where this page and the contract disagree,
the contract wins.

**Production:** `https://identity.fascia.network`


**Host**, used throughout: a platform that builds the passport into its own app. Its members'
phones hold the passports; the host's backend relays their requests and holds a host key.

## Who calls what

| Caller | Credential | May call |
| --- | --- | --- |
| Anyone | none | `/health`, `/.well-known/fascia-identity`, a community's keys and revocation list |
| A host's backend | host key | verify, publish a revocation list, hand over a community's key |
| A Fascia rail service | service key | audit entries |
| The operator | operator key | create a community, suspend or reactivate one |

Keys are bearer tokens: `Authorization: Bearer <key>`. A host key is issued per host and held
server-side. It is never in an app bundle, never in a browser, never in a log.

## The presentation

A member's phone attaches a presentation to every request it makes to a Fascia service, and to
any request to its host that the host wants to verify. It is one header:

```
Fascia-Presentation: v1.<canonical JSON>.<signature>.<claims>
```

Each part is base64url without padding.

| Part | Content |
| --- | --- |
| Canonical JSON | `{ v, aud, method, path, body_sha256, iat, nonce, key, claims_sha256 }` in RFC 8785 form. See `schemas/presentation.json` |
| Signature | By the phone's device key (`key`, a `did:key`) over the canonical JSON's bytes. Ed25519, or ECDSA P-256 as raw `r‖s` with low `s` |
| Claims | One DAG-CBOR array of chains; each chain an array of UCAN 1.0 delegations (`ucan/dlg@1.0.0`), root first. `claims_sha256` in the JSON is the SHA-256 of these bytes |

The header is at most 8,192 bytes and carries at most four chains. `aud` names the service the
request is for, so a presentation made for one service is refused by every other. `nonce` makes
it single use. `iat` must be within a minute of the receiver's clock, allowing two minutes of skew.

### The nine checks

Every receiver runs the same checks, in this order, and stops at the first failure.

1. **Host key** resolves to a host (Fascia services only).
2. **Form:** the header parses, is canonical, is within the limits.
3. **Audience:** `aud` is the receiver's own.
4. **Freshness:** `now - 180 <= iat <= now + 120`.
5. **Replay:** the `(key, nonce)` pair has not been seen.
6. **Binding:** `method`, `path` (with query, byte for byte) and the body's SHA-256 match the request received.
7. **Signature:** the device key's signature verifies, and the claims hash to `claims_sha256`.
8. **Claims:** every chain is rooted in a key the community's register lists, and passes the Fascia profile rules below.
9. **Revocation:** no claim or key in any chain is on that community's revocation list.

### The Fascia profile

UCAN lets any holder pass on what it holds. These rules sit on top of UCAN's own:

- A chain's first link is issued by its subject, and the subject is an `active` or `retiring` key of the community named in `meta.fascia.community_id`.
- Each link's issuer is the previous link's audience; the last audience is the presenting key.
- Every link is in date, no link outlives its parent or its type's longest life.
- Only a chain ending in `/fascia/member` proves membership.
- Every link names the same community.
- Only the community's agent, or a steward under an `issuer` link, may name a different `dcp_id` from the link above. Everyone else's delegations carry their own.
- A membership chain is: an optional `issuer`, one `membership`, an optional `device`, an optional `session`. Four links at most.
- Ring and role never widen down a chain.

The seven claim types and their fields are in `schemas/claim-meta.json`.

## Endpoints

### Discovery

```
GET /.well-known/fascia-identity
```

```json
{
  "aud": "https://identity.fascia.network",
  "schema_versions": [1],
  "curves": ["Ed25519", "P-256"],
  "claim_envelope": "ucan/dlg@1.0.0",
  "freshness_seconds": 60,
  "skew_seconds": 120,
  "max_presentation_bytes": 8192,
  "max_chains": 4,
  "list_refresh_seconds": 300,
  "list_max_age_seconds": 86400
}
```

Every Fascia service serves the same document at the same path, with its own `aud`. A phone reads
it to know what to name.

### A community's keys

```
GET /v1/identity/communities/{community_id}/keys
```

```json
{
  "community_id": "6f1c2a9e-4b7d-4e8a-9c21-7d4e5f6a8b90",
  "status": "active",
  "keys": [
    { "key": "did:key:zDn...new", "state": "active", "since": 1793779200,
      "handover": { "from": "did:key:zDn...old", "at": 1793779200, "sig": "..." } },
    { "key": "did:key:zDn...old", "state": "retiring", "since": 1785830400, "retire_after": 1809331200 }
  ],
  "updated_at": 1793779200
}
```

Public, with an `ETag` and `Cache-Control: max-age=300`. A `retiring` key still roots chains until
`retire_after`, so claims it signed stay valid while they last. A suspended community's claims fail
check 8.

### A community's revocation list

```
GET /v1/identity/communities/{community_id}/revocations
```

The signed list in `schemas/revocation-list.json`. A community with nothing revoked answers `seq`
0 and no entries. Served `no-cache` with an `ETag`: revalidate every 300 seconds, and stop trusting
a copy you could not refresh for 24 hours.

A `claim` entry fails any chain containing that claim. A `key` entry fails any chain in which that
key issued or received a link, so revoking a steward's key revokes everything the steward issued.

```
PUT /v1/identity/communities/{community_id}/revocations       (host key)
```

The host relays a list a steward's phone signed. It is accepted on its own signature, by the
community's agent with an empty `proof` or by a steward with its `issuer` delegation as `proof`,
and only if `seq` grows and every entry still in date is kept. Answers `{ "seq": 43 }`.

### Verify, for hosts

```
POST /v1/identity/verify       (host key)
```

```json
{
  "presentation": "v1.eyJ...",
  "method": "POST",
  "path": "/functions/v1/call-token",
  "body_sha256": "...",
  "aud": "https://host.example"
}
```

`aud` must be the host's registered audience: ask for it to be registered when your host key is
issued. That stops a host from checking, or using up the nonce of, a presentation meant for a
Fascia service.

The answer is always `200` when the call itself was well formed:

```json
{
  "ok": true,
  "dcp_id": "dcp:k4x2...",
  "key": "did:key:zDn...",
  "memberships": [
    { "community_id": "6f1c...", "role": "member", "ring": 2,
      "claim_ids": ["..."], "expires_at": 1801555200, "via": ["issuer", "membership"] }
  ],
  "standing": [],
  "locality": null
}
```

or `{ "ok": false, "error": { "code": "claim_revoked" } }`.

### Audit, for Fascia services

```
POST /v1/identity/audit       (service key)
```

Batches of up to 1,000 entries. The service replaces each `dcp_id` with a pseudonym that changes
every 30 days before it stores the row, deletes the key that made the pseudonym when those 30 days
end, and deletes the row after 90 days.

### The register

```
POST /v1/register/communities                       (operator key)
POST /v1/register/communities/{id}/status           (operator key)
POST /v1/register/communities/{id}/keys/handover    (host key)
```

A community is created with its agent's first key and that key's signature over
`{ name, first_key }`. A handover `{ from, to, at, sig }` is signed by the current active key; the
old key becomes `retiring` for 180 days.

## Errors

`{ "error": { "code", "message" } }`. Switch on `code`, never on the status alone.

| Code | Status | Means |
| --- | --- | --- |
| `host_key_invalid` | 401 | No host key, or not one we issued |
| `presentation_missing` | 401 | The request carried no presentation |
| `presentation_malformed` | 400 | It does not parse, is not canonical, or has an unknown version |
| `presentation_too_large` | 400 | Over 8,192 bytes or four chains |
| `audience_mismatch` | 401 | It was made for another service |
| `presentation_stale` | 401 | Signed too long ago or in the future. `server_time` says the receiver's clock; correct and retry once |
| `presentation_replayed` | 401 | Its nonce was used. Sign again |
| `request_mismatch` | 401 | Method, path or body changed after signing. A relay that rewrites requests causes this |
| `signature_invalid` | 401 | The device signature fails, or the claims are not the signed ones |
| `claim_invalid` | 401 | A chain breaks a rule; `rule` names it. If the rule is `time`, renew |
| `claim_revoked` | 401 | A claim or key in the chain is revoked |
| `identity_unavailable` | 503 | Revocations could not be confirmed. Retry later |
| `ring_denied` | 403 | Valid, but the member's rings do not reach this record (rail services) |
| `audience_not_registered` | 403 | Verify named an audience that is not the host's |
| `revocation_malformed`, `revocation_signer_invalid`, `revocation_seq_stale`, `revocation_not_superset` | 400, 403, 409, 422 | A revocation list was refused; see the contract |
| `community_unknown` | 404 | No such community |
| `not_found` | 404 | No such path, with or without a key |

## What a host must do

- **Keep the host key server-side.** The phone calls your backend; your backend adds the key.
- **Relay requests unchanged.** The phone signed the method, the path with its query, and the body. Change any of them and the request fails.
- **Store the `dcp_id` only.** Never store a member's claims, keys or presentations.
- **Switch on `error.code`.**
- **Prove your own verifier against the vectors** if you write one: every file in `vectors/` gives a request, the state to load and the expected result.

## Conformance vectors

`vectors/` holds 47 cases: 8 that must pass and 39 that must fail with the named code and rule.
They are deterministic: fixed keys, fixed nonces, a fixed clock. A verifier that gives the expected
result for every step of every file conforms.

## Versioning

`info.version` moves on every change to the document, the schemas or the vectors. A new field,
endpoint or response is a minor version; editorial changes are a patch. Breaking changes do not
happen inside v1: the path carries the major version. The service reports the version it implements
in the `x-identity-contract` header on every response.
