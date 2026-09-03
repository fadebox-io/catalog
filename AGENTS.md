# Working on this catalog with an agent

This file is for coding agents (Claude Code, Cursor, Copilot and the like) and for the people
driving them. It says what a catalog is, what fadebox checks, and how to prove an entry before it
is published. The README covers the same ground for a human reader.

## What you are editing

A catalog is a git repository with an index and one directory per entry:

```
index.yaml                    # browsing metadata; all the Catalog page renders
<entry>/template.yaml         # the payload an import copies: spec plus optional files
schema/index.schema.json      # JSON Schema for index.yaml
schema/template.schema.json   # JSON Schema for template.yaml
```

Both files carry a `# yaml-language-server: $schema=…` line so an editor validates them; the
schemas are also usable from any JSON Schema tool. They describe the shape. They cannot check
the Compose YAML inside `spec`, which is what actually decides whether an entry works.

Two rules make the update hint work:

- **An entry's `slug` never changes.** Imported templates record it as provenance; renaming
  orphans every copy.
- **`version` is the only update signal.** A commit that changes `template.yaml` without bumping
  `version` is invisible to installations that already imported the entry.

## The template dialect

`spec` is Docker Compose YAML with fadebox's rules on top. The full reference is
<https://fadebox.io/docs/guides/template-authoring>; the parts an author gets wrong first:

- **One main service plus helpers.** Exactly one service carries no `fadebox.role` label; every
  other service is `fadebox.role: init` (runs once and exits) or `fadebox.role: sidecar`. Peers
  such as an application and its database are separate entries composed in an environment, not
  one entry.
- **Ingress is a label.** `fadebox.ingress.port: "8080"` exposes a container port on the
  instance URL; `fadebox.ingress.path` and `fadebox.ingress.strip` route it by path prefix.
  Published `ports:` are not needed for that.
- **The Compose subset is static and strict.** Top level: `services`, `networks`, `volumes`
  (`version` and `name` are accepted and ignored). Per service: `image`, `environment`,
  `ports`, `volumes`, `command`, `entrypoint`, `depends_on`, `healthcheck`, `restart`,
  `networks`, `mem_limit`, `cpus`, `deploy.resources.limits`, `privileged`, `user`, `labels`.
  Anything else — `build:`, `secrets:`, `configs:`, `profiles:`, `x-*` extensions — is refused,
  not ignored, and the refusal names the key.
- **Placeholders.** `{{instance.host}}`, `{{ingress.scheme}}` and `{{ingress.domain}}` resolve
  at deploy time. `{{service.<name>.host}}` and `{{service.<name>.env.VAR}}` refer to a sibling
  service in the environment and only resolve there — an entry using them cannot be test-run on
  its own, and its index row should list the sibling in `requires`.
- **Conventions this catalog keeps**, and that fadebox warns about when missing: image tags
  pinned (never `latest`), a healthcheck on every long-running service (init helpers are exempt),
  an ingress port on anything with a UI, and throwaway credentials stated in the description.

## Proving an entry

The parser that decides is fadebox's, so the check has to run there. With a fadebox installation
connected over MCP (<https://fadebox.io/docs/guides/mcp>), the loop is:

1. `validate_template` with the entry's `spec` — returns the refusal a save would give, or a
   readback of what fadebox understood (roles, ingress ports, healthchecks, volume mounts) plus
   advisory warnings. Fix and repeat until it is clean. Choose `files[].targetPath` outside the
   volume mounts the readback lists.
2. Commit and push.
3. `create_catalog_source` once for the repository, then `sync_catalog_source` after every push.
   The answer is the commit now served and its entry count, or what is wrong with `index.yaml`.
4. `get_catalog_entry` to see what an import would copy, `import_catalog_entry` to make a real
   template from it, and `start_test_run` / `get_test_run` / `get_test_run_logs` /
   `stop_test_run` to prove the template starts on a runtime.

Without MCP, the same validation is `POST /api/template-validation` with `{"spec": "…"}` and an
API key, and a source is registered under **Catalog → Add source** in the UI.

## Adding or changing an entry

1. Create `<slug>/template.yaml`; `slug`, `name` and `version` repeat the index row.
2. Add the row to `index.yaml` under the matching comment group, with `tags` and, for an entry
   that expects a sibling, `requires`.
3. Validate the spec as above; a new entry should come up on its own in a test run.
4. On any later change to `template.yaml`, bump `version` in both files.

Entries here ship throwaway development credentials on purpose, so an evaluation comes up with
no configuration; say so in the description, as the existing entries do.
