# Fadebox Official catalog

The service-template catalog that ships with [Fadebox](https://fadebox.io) — databases, messaging,
identity, search, storage, cloud emulators and dev tools, ready to import as templates.

Every Fadebox installation already carries a copy of this catalog inside the application, so a
fresh install has a populated Catalog page with no network access at all. This repository is the
same catalog published as a git source, for installations that want entries as they are published
rather than as each release ships them.

## Using it

In Fadebox: **Catalog → Add source**, with

| Field | Value |
| --- | --- |
| Slug | `official` (or whatever you like — it is recorded on the templates you import) |
| Repository URL | `https://github.com/fadebox-io/fadebox-catalog.git` |
| Branch or tag | `main` |

Entries are then browsable beside the bundled ones. Importing one copies it into an ordinary,
editable template; nothing stays linked to this repository afterwards.

Registering this source lists its entries alongside the bundled copies of the same entries. If you
would rather see each entry once, switch the bundled catalog off in the same Sources list.

## The format

A catalog is an index plus one directory per entry:

```
index.yaml
postgres/
  template.yaml
```

`index.yaml` is browsing metadata — it is all the Catalog page renders, so listing never opens an
entry:

```yaml
apiVersion: fadebox.catalog/v1
name: Fadebox Official
entries:
  - slug: postgres            # stable: imported templates record it as provenance
    name: PostgreSQL
    version: 1.0.0            # the update signal
    description: Single-node PostgreSQL 18 with healthcheck and persistent storage.
    tags: [ database ]
    requires: [ ]             # entries this one expects at their well-known service names
    path: postgres/template.yaml
```

`<entry>/template.yaml` is the payload an import copies — `spec` is a Docker Compose document,
`files` are injected into the container at deploy time:

```yaml
slug: postgres
name: PostgreSQL
version: 1.0.0
description: Single-node PostgreSQL 18 with healthcheck and persistent storage.
spec: |
  services:
    postgres:
      image: postgres:18-alpine
      ...
```

Two rules make the update hint work, and they apply to any catalog you write yourself:

- **An entry's `slug` never changes.** Templates imported from it record the slug; renaming it
  orphans every copy, which then stops reporting updates.
- **`version` is the only update signal.** A new commit with an unchanged version is not an update.

Writing your own catalog is the same shape: copy this repository's layout, keep your own entries,
and register it as a source. See the [Catalog sources
guide](https://fadebox.io/docs/guides/catalog-sources).

## Credentials in these entries

Every entry ships throwaway development credentials, stated in its description on purpose
(`postgres`/`postgres`, `admin`/`admin`, and so on). They exist so an entry comes up on its own for
an evaluation. Override them at the environment level for anything shared or long-lived.

## Licence

Apache-2.0 — see [LICENSE](LICENSE). The catalog is published under it so you can fork it, trim it,
or use it as the starting point for a catalog of your own; Fadebox itself is a separate, commercial
product.

The entries are curated with the product, and the copy inside the Fadebox build is the source of
truth: what is here is what that build ships.
