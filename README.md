# immediately.run UI release registry

This **public** repository is the global registry of **UI releases** for
[immediately.run](https://immediately.run) (see `UI_RELEASES_SPEC` in the private
`docs` repo). It exists solely to host the registry on GitHub Pages — the host app
(`immediately-run-site-main`) is private, so it can't serve a public Pages site,
hence this dedicated repo.

A *release* is a named, immutable `region → repo@commit` set that composes the
immediately.run chrome (landing, spaces manager, file explorer, editor, …). A
deployment (or user) selects one by name, letting parallel UI builds — e.g. a
CodeMirror vs a Monaco editor — coexist without forking the host.

**A release carries code, not authority.** It only repoints each region's repo.
Capability ceilings, contracts, and working-tree exposure stay in the host's
checked-in build defaults (the TCB). Nothing here can grant a region more power.

## Served at

```
https://immediately-run.github.io/ui-releases/index.json        # the registry
https://immediately-run.github.io/ui-releases/<name>.lock.json  # one per release
```

A deployment opts in via its config:
`"release": { "name": "base" }` (the registry URL defaults to the above).

## Files

- **`<name>.json`** — *authoring input* (hand-edited, reviewed). Sparse: an
  overlay may `extends` a base and list only the regions it swaps.
  - `base.json` is the full default composition (all regions on `@main`).
  - Overlay example `monaco-2026-06.json`:
    ```json
    { "id": "monaco-2026-06", "label": "Monaco editor",
      "extends": "base", "apps": { "task.edit-file": "github:immediately-run/monaco-editor@main" } }
    ```
  - `id` MUST equal the filename (without `.json`).
- **`<name>.lock.json`** — *generated, committed*: every region resolved to a
  commit. Deterministic, so its sha-256 is stable.
- **`index.json`** — *generated, committed*: each release's lock url + sha-256
  (the integrity anchor the host verifies against).

## Adding or updating a release

```sh
npx @immediately-run/cli pin-release --dir .
```

Resolves each `@ref` to a commit (`git ls-remote`), writes the lock(s), rebuilds
`index.json`. Commit the changed `*.lock.json` + `index.json`; the
`publish.yml` workflow validates (`pin-release --check`, networkless) and deploys
to Pages on push.

**Immutable by name.** Re-pinning an existing name to *different* content is
refused — a published name is frozen. Ship a new composition under a new name
(`monaco-2026-07`); to correct a mistake, pass `--republish`.

## The defaults-derived base map (2026-09-16)

`base.json` + `defaults-map.json` are **derived mechanically** from site-main's
`BUILD_DEFAULTS` (`npm run export:release-base` in the site-main checkout — the
registry repo never reads the private host source). Every build-default region
appears; an omitted region would silently re-couple to the moving `@main`
(`UI_RELEASES_SPEC` §6.3 fallback), which is the opposite of a collection.
`pin-release --check` fails at publish naming any region the base lock omits
relative to `defaults-map.json` — re-run the export after every site-main
registry change, commit both files, and republish `base`
(`--only base --republish`, deliberate).

## The `testing` channel (2026-09-16)

`testing.json` is the channel *template*: `extends base`, no deltas — the base
authoring map at `@main`, i.e. **the latest of origin/main**. The
[`testing.yml`](.github/workflows/testing.yml) workflow (hourly + dispatch)
resolves it into a **new dated immutable lock** (`testing-YYYY-MM-DD-<sha8>`),
repoints `channels.testing` in `index.json` (the only mutable write in this
registry), bakes the `zips/<ns>/<repo>/<sha>.zip` artifacts (§3.4 — the app
repos' own Pages evict older shas), and commits to main; the push triggers
`publish.yml`, which validates and serves Pages. A production deployment may
never name a channel — it pins a concrete `name`+`sha256`; the `testing`
channel is for dev/preview deployments and user selection.

Historical locks (dated targets whose authoring is gone) are part of the
committed registry: `pin-release` keeps them in the index verbatim, and
`--check` fails if one is dropped.

### Retention — the store is append-only, and its ceiling is stated

**Nothing here evicts anything automatically, by design.** The sentence above is
why: `pin-release` keeps every historical lock verbatim and `--check` fails if
one is dropped, so an hourly dated channel adds a lock whenever the composition
changes and nothing ever removes one. The `zips/` artifacts accumulate with them.

That matters because [`publish.yml`](.github/workflows/publish.yml) copies the
whole tree into `_site`, and **GitHub Pages refuses a published site over 1 GB**.
Past that, `index.json` itself stops serving and every host boot falls to the
§6.3 built-in fallback — the failure this registry exists to remove.

Measured 2026-09-17: a full bake is **32 zips / ~21 MB** (~660 KiB mean).
Resident zips are reused, so a no-op run adds nothing.

So the bound is a **tripwire, not a policy**. `testing.yml` checks `zips/` before
it commits:

| `zips/` size | what happens |
|---|---|
| under 256 MiB | nothing |
| 256 MiB or more | `::warning::` — prune soon |
| 512 MiB or more | `::error::`, **the run fails before committing** |

512 MiB is half of Pages' refusal point, and roughly 25× a full bake — an order
of magnitude of headroom, on purpose. The workflow does not prune, because
deciding a dated target is no longer deployable is a judgement, and a wrong one
cannot be undone from a workflow.

**To prune a retired dated target, an operator removes all three of:**

1. its entry in `index.json`,
2. its lock file, and
3. its authoring entry.

All three, together. Dropping only the index entry leaves the registry
inconsistent — measured: `pin-release --check` then fails with *"index.json is
out of date or has wrong digests"*, while removing all three passes with
*"N release(s) consistent with index.json"*. Re-run `--check` after pruning, and
never prune a target a deployment still pins by `name`+`sha256`.
