# Stellar Forge — SubQuery GraphQL Merchant

A standalone merchant/operator product for selling approved SubQuery GraphQL
queries over Stellar x402 payment rails. It captures a GraphQL schema, lets a
seller author a fixed set of approved queries with bounded buyer variables,
stages them as immutable releases, and serves them behind original x402
challenge or native payment terms — with an outbox delivery journal, private
operator operations, and a generated Linux host deployment bundle.

This repository contains the complete product source closure. It is a real
merchant/operator integration product, not a demonstration or test package.

## Layout

`stellar/scf-starforge-20261009/`

| Component | Role |
|---|---|
| `sf105-subquery-graphql-merchant/` | The product: GraphQL schema capture, approved-query authoring, revision history, deployment compiler, operator workspace UI (`workspace/`), seller launch entry (`launch.mjs`) |
| `sf30-seller-sdk/` | Seller runtime: route compilation, upstream fetch/serve, x402 resource server, OAuth connections, operations console (`operations/`), developer client generator (`developer/`) |
| `sf61-openapi-seller-import/` | OpenAPI source importer used to compile release inputs (spec → service configuration + bindings) |
| `sf82-seller-pilot/` | Release store and pilot: stage/list/inspect/diff/activate/rollback, cache control, facilitator review, seller runner, private console |
| `sf98-merchant-delivery/` | Delivery journal: outbox, delivery lock, recovery/requeue |
| `sf99-merchant-deploy/` | Host bundle generator: systemd units, Caddy fragment, recovery plan, runbook, smoke checks |
| `sf30-seller-discovery/` | Seller discovery document (`.well-known` offer catalog consumed by the seller service) |
| `sf31-buyer-client/`, `sf85-live-402-capture/`, `sf89-live-request-admission/` | Shared transport/admission helpers consumed by the seller runtime |
| `stellar-forge/payment-auto-catalog/` | Shared bazaar-extension schema validators consumed by the resource server |

## Requirements

- Node.js 22 or later (uses `node:sqlite`, `node:perf_hooks`).
- `npm install` for the single required dependency: `graphql@16.14.2`.
- For **native** x402 payment mode, the operator additionally installs the
  pinned x402 SDK (`@x402/core@2.28.0`, `@x402/stellar@2.28.0`,
  `@x402/extensions@2.28.0`). These are loaded dynamically only in native mode;
  challenge mode does not need them.
- The operator workspace and generated deployment target Linux; the workspace
  uses `flock`, and deployment bundles use systemd and Caddy 2.

## Use

Author and launch a merchant seller:

```sh
node stellar/scf-starforge-20261009/sf105-subquery-graphql-merchant/launch.mjs \
  --seller SELLER.json --queries QUERIES.json [--check] [--port 8423] \
  [--facilitator HTTPS_URL]
```

`--check` compiles offers without starting listeners (no provider calls).
`--facilitator` selects native mode; without it the seller issues unpaid
challenges only. Merchant state, OAuth stores and private state directories
are operator-provided paths — see the generated runbook.

The merchant workspace provides schema capture, query authoring, revision
history, previews and export through its own local server. Start it from this
repository root on Linux:

```sh
node stellar/scf-starforge-20261009/sf105-subquery-graphql-merchant/workspace/server.mjs \
  --workspace /srv/query-merchant/workspace --port 0
```

Open the complete local URL printed by that command in the same machine's
browser. The server binds `127.0.0.1` and serves both the operator interface and
its onboarding, authoring and deployment endpoints. Ctrl-C drains the managed
merchants and release consoles before closing the workspace.

Releases:

```sh
node stellar/scf-starforge-20261009/sf82-seller-pilot/release-cli.mjs \
  init --store DIR
node .../release-cli.mjs stage --store DIR --id RELEASE --spec OPENAPI \
  --config SELECTIONS --bindings BINDINGS
node .../release-cli.mjs list|inspect|diff|activate|rollback|status --store DIR
node .../release-cli.mjs cache-status|cache-invalidate --store DIR
node .../release-cli.mjs facilitators|facilitator-review|facilitator-stage
node .../release-cli.mjs run --store DIR          # foreground seller
node .../release-cli.mjs console --store DIR      # private operator console
```

Deployment bundle (generate → inspect → check → smoke):

```sh
node stellar/scf-starforge-20261009/sf99-merchant-deploy/cli.mjs \
  generate --plan PLAN.json --bundle DIR
node .../cli.mjs inspect|check|smoke --bundle DIR
```

OAuth upstream connections (used by the generated runbook):

```sh
node stellar/scf-starforge-20261009/sf30-seller-sdk/oauth-connections/cli.mjs \
  init|connect|status --directory DIR
```

## Configuration boundary

All provider URLs, upstream endpoints, facilitator URLs, OAuth profiles,
merchant/payment terms, state directories and environment-file references are
explicit operator configuration. The tree contains no credentials, no live
provider, no enrolled customer and no production deployment. The generated
`seller-client.mjs` is produced per merchant by
`sf30-seller-sdk/developer-portal.mjs` — it is runtime output, not a checked-in
artifact.

## License

MIT — see `LICENSE` and `NOTICE`.
