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
`"release": { "name": "base" }` resolves through the `channels.base` pointer to the current dated
base lock (2026-10-08, R3-823 — `base` is a channel, never a mutable release name; a deployment
that pins `name`+`sha256` names the dated lock directly).

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
  commit, or, for an `"unpinned": true` template (`testing`), every region
  kept at its ref with no commit. Deterministic, so its sha-256 is stable.
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

**Immutable by name.** A published name is frozen — re-pinning it to *different*
content is refused with or without `--republish` (the flag is the vestigial
byte-identical repair hatch; since 2026-09-30 / cli 0.9.4 it lifts nothing — R3-823).
Ship a new composition under a new name (`monaco-2026-07`).

## The defaults-derived base map (2026-09-16)

`base.json` + `defaults-map.json` are **derived mechanically** from site-main's
`BUILD_DEFAULTS` (`npm run export:release-base` in the site-main checkout — the
registry repo never reads the private host source). Every build-default region
appears; an omitted region would silently re-couple to the moving `@main`
(`UI_RELEASES_SPEC` §6.3 fallback), which is the opposite of a collection.
`pin-release --check` fails at publish naming any region the base lock omits
relative to `defaults-map.json` — re-run the export after every site-main
registry change, commit both files, and republish `base` as a NEW DATED LOCK
(`--only base --dated base --channel base=@dated`, deliberate — `base` is a
channel template: the plain name is retired (R3-823), `channels.base` is the
one mutable pointer, and every dated base lock stays in the index verbatim).

## The `testing` channel (2026-09-16; unpinned since R3-658, 2026-10-06)

There are two channels: `base` and `testing`. `base`'s channel target is a
dated immutable lock — pinned and digest-verified: that is what production
loads. `testing` is the bleeding edge.

`testing.json` is the channel *template*: `extends base`, no deltas, and
`"unpinned": true`. The [`testing.yml`](.github/workflows/testing.yml)
workflow publishes it as a **new dated immutable lock**
(`testing-YYYY-MM-DD-<sha8>`) that names each app's **ref** and no commit,
and repoints `channels.testing` in `index.json` (the only mutable write in this
registry). Nothing is resolved to a commit and no zip is baked for it: every
region follows the head of its branch on every boot. The run's bake step covers
only the pinned locks already in the index: it reuses their resident zips and
bakes any that are absent. The
workflow then commits to main, and the push triggers `publish.yml`, which
validates and serves Pages.

The lock changes only when the authoring does, so `testing.yml` runs on a push
touching `testing.json` or `base.json`, or on dispatch. `pin-release --check`
lets only the `testing` channel target an unpinned lock (UI_RELEASES_SPEC §3.2).
A host applies an unpinned lock only where its deployment config allows it
(staging, `local.immediately.run`, `localhost`), never in production (§4.1); a
production deployment pins a concrete `name`+`sha256` and never names a
channel. A reproducible pinned snapshot is still one `pin-release --dated <id>`
away.

Historical locks (dated targets whose authoring is gone) are part of the
committed registry: `pin-release` keeps them in the index verbatim, and
`--check` fails if one is dropped.

### Retention — the store is append-only, and its ceiling is stated

**Nothing here evicts anything automatically, by design.** The sentence above is
why: `pin-release` keeps every historical lock verbatim and `--check` fails if
one is dropped, so each dated channel run adds a lock whenever the composition
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
