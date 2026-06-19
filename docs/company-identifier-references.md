# Company identifier references in the Federated Catalogue

How the catalogue can act as a **discoverable registry** that links a company's canonical **DID** to other identifiers and attributes (legal name, address, BPN, IBAN, …) in a data space for use as a Contruct-X Registry.

## Idea

In many data spaces the **company DID** is the primary key for a participant. Other identifiers are secondary lookups:

| Reference type | Example | Typical use |
|----------------|---------|-------------|
| Legal name | `deltaDAO AG` | Human search, UI display |
| Address | Hamburg, DE | Jurisdiction, KYC |
| BPN | `BPNL000000000111` | Catena-X / automotive partner matching |
| IBAN | `DE89…` | Payment routing (usually restricted) |
| VAT / LEI | `DE123456789` | Gaia-X notarisation (`gx:legalRegistrationNumber`) |

The Federated Catalogue does not need a separate “registry microservice”. It already **ingests Verifiable Credentials**, projects `credentialSubject` claims into an RDF graph, and exposes them via **`POST /query`**.

## How it works today

1. A provider publishes a signed VC whose `credentialSubject.id` is the company anchor (DID or stable participant IRI).
2. Arbitrary RDF properties on that subject become **queryable triples** (RDF-star reification in Fuseki).
3. Consumers resolve `BPN → DID`, `name → DID`, etc. with SPARQL.

Custom vocabularies are supported the same way as Gaia-X or DCS templates: declare a namespace in `@context` and use typed predicates.

## Recommended modelling

Use **one anchor subject per company** — preferably the DID:

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://w3id.org/gaia-x/2511#",
    {
      "schema": "https://schema.org/",
      "cx": "https://w3id.org/catenax/ontology/participant#",
      "ids": "https://w3id.org/idsa/core/"
    }
  ],
  "type": ["VerifiableCredential", "gx:LegalPerson"],
  "issuer": "did:web:acme.example",
  "credentialSubject": {
    "id": "did:web:acme.example",
    "type": "gx:LegalPerson",
    "schema:name": "ACME GmbH",
    "gx:legalAddress": {
      "type": "gx:Address",
      "gx:countryCode": "DE",
      "vcard:locality": "Munich",
      "vcard:street-address": "Examplestraße 1"
    },
    "cx:bpn": "BPNL000000000111",
    "ids:iban": "DE89370400440532013000"
  }
}
```

Alternatively, split concerns:

- **Participant VC** — Gaia-X `LegalPerson` with name, address, `gx:legalRegistrationNumber`
- **Reference VC** — lightweight credential that only asserts `did ↔ BPN` / `did ↔ IBAN`, issued by a trusted registry operator

Both approaches use the same ingestion path: `POST /assets` with `application/vc+jwt` or `application/ld+json`.

## Discovery examples

After ingestion, consumers query the graph (Fuseki + SPARQL-star):

**BPN → company DID**

```sparql
PREFIX cred: <https://www.w3.org/2018/credentials#>
PREFIX cx:   <https://w3id.org/catenax/ontology/participant#>
SELECT ?company WHERE {
  <<(?company cx:bpn "BPNL000000000111")>> cred:credentialSubject ?company .
}
```

**Legal name → DID**

```sparql
PREFIX cred:   <https://www.w3.org/2018/credentials#>
PREFIX schema: <https://schema.org/>
SELECT ?company ?name WHERE {
  <<(?company schema:name ?name)>> cred:credentialSubject ?company .
  FILTER(CONTAINS(LCASE(STR(?name)), "acme"))
}
```

**Address in a country**

```sparql
PREFIX cred: <https://www.w3.org/2018/credentials#>
PREFIX gx:   <https://w3id.org/gaia-x/2511#>
SELECT ?company WHERE {
  <<(?company gx:legalAddress ?addr)>> cred:credentialSubject ?company .
  <<(?addr gx:countryCode "DE")>>       cred:credentialSubject ?company .
}
```

See [`examples/queries/verify-against-fuseki.hurl`](../examples/queries/verify-against-fuseki.hurl) for executable patterns on Gaia-X fixtures.

## Trust and governance

| Concern | Catalogue behaviour |
|---------|---------------------|
| Who can publish? | Keycloak roles (`ASSET_CREATE`, `Ro-MU-CA`, …) gate `POST /assets` |
| Is the mapping authoritative? | Enable trust-framework validation + VC signatures; optional GXDCH compliance (`POST /assets/{id}/compliance-check`) |
| Stale data | Revoke or supersede assets (`/assets/{id}/versions`, `prov:wasDerivedFrom` lineage) |
| Sensitive fields (IBAN) | Prefer restricted credentials, separate assets, or off-catalogue vaults; the graph is readable by anyone with `QUERY_EXECUTE` |

The catalogue stores and indexes claims; **trust** comes from who issued the VC and which verification toggles the operator enables.

## Operational notes

- **Schema**: register a custom ontology via `POST /schemas` if you want SHACL validation for BPN/IBAN shapes.
- **Updates**: publish a new VC version; discovery queries can prefer the latest approved entry (same pattern as the DCS template demo).
- **Federation**: configure partner catalogues under `federated-catalogue.query.partners` so `POST /query/search` can resolve identifiers across nodes.

## Summary

The Federated Catalogue can host arbitrary company references by treating the **company DID as `credentialSubject.id`** and publishing signed RDF claims for every secondary identifier. No new API is required — only a agreed vocabulary, ingestion policy, and SPARQL discovery queries.
