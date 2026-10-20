# Fascia Identity — API contract

The OpenAPI 3.0 document for the Fascia Identity v1 API: the service behind the Digital Community
Passport (DCP). It is published so that a host can read it, vendor a copy, generate a client and
diff against it when it changes, with the prose reference, the schemas and the conformance vectors
beside it.

- **[identity.v1.yaml](identity.v1.yaml)** — the contract. Current version: **0.5.0**.
- **[API.md](API.md)** — the API documentation: the presentation a member's phone sends, the nine
  checks every receiver runs, each endpoint with worked responses, and what a host must do. Where it
  and the contract disagree, the contract wins.
- **[schemas/](schemas/)** — JSON Schemas for every signed object: the presentation, the claim
  fields, the key card, the inception record, the revocation list and the sealed envelope.
- **[vectors/](vectors/)** — 47 conformance cases. A verifier that gives the expected result for
  every step of every file conforms.
- **[Browse the contract](https://daniel-kruger-rsa.github.io/fascia-identity-contract/)** — the
  same file rendered as a page (Redoc), always the version on `main`.
- Every published version is tagged (`v0.5.0`), so a vendored copy can pin one:
  `https://raw.githubusercontent.com/daniel-kruger-rsa/fascia-identity-contract/v0.5.0/identity.v1.yaml`

**Host**, used throughout: a platform that builds the passport into its own app. Its members'
phones hold the passports; the host's backend relays their requests and holds a host key.

## What the service is

A member's phone holds a signing key that never leaves it, and signed membership claims from the
stewards of their communities. With each request to a Fascia service the phone signs the request
and attaches the claims that request needs. The receiver checks the signatures against the
community's public key, which this service publishes, and stores nothing about who belongs where.

The service issues nothing and stores no member's key or claim.

## Versioning

`info.version` moves on every change to the document, the schemas or the vectors.

| Change | Version |
| --- | --- |
| A new field, endpoint or response | minor (`0.1` → `0.2`) |
| Editorial only — wording, examples, descriptions, no interface change | patch (`0.1.0` → `0.1.1`) |

**[CHANGELOG.md](CHANGELOG.md)** has every published version, newest first, with one column that
answers the only question a diff really asks: does your client care? Nothing is published without
an entry in it, and nothing is published before the live service reports that version in its
`x-identity-contract` header.

Breaking changes do not happen inside v1. The path carries the major version.

## Licence

The contract, API.md, the schemas and the vectors are MIT (see [LICENSE](LICENSE)): vendor them,
diff them, generate clients from them, test your verifier against them. The licence covers these
files, not access to the API.

## Contact

Fascia Identity. Issues on this repo are read.
