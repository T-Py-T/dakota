# NOT Bluefin — Dakota Build Context

**Load this skill FIRST before any dakota task.** Dakota is fundamentally different from bluefin.

---

## Dakota uses BuildStream 2 (BST). Not dnf. Not RPM. Not Containerfile.

| Bluefin Pattern | Dakota Reality | What to do instead |
|---|---|---|
| `dnf5 install <pkg>` | ❌ PROHIBITED | Write a `.bst` element file |
| `dnf5 copr enable` | ❌ PROHIBITED | BST has no COPR concept |
| `copr_install_isolated()` | ❌ PROHIBITED | Not applicable |
| `copr-helpers.sh` | ❌ PROHIBITED | Does not exist in dakota |
| Containerfile `RUN dnf5...` stage | ❌ PROHIBITED | Use BST elements for packages |
| `.spec` files / RPM build | ❌ PROHIBITED | BST elements only |
| Fedora package names (RPM) | ⚠️  May differ | Verify in BST element definitions |

## What dakota DOES use

- **BuildStream 2** (`.bst` files) — the only way to add packages
- **BST element kinds**: `autotools`, `cmake`, `meson`, `pip`, `manual`, `stack`, `junction`
- **Upstream junctions**: `freedesktop-sdk.bst` and `gnome-build-meta.bst` provide most packages
- **OCI assembly**: A `Containerfile` exists for final image assembly only — not for package installation

## Adding a package to dakota

See `docs/skills/add-package.md`. Never reach for `dnf5`.

## The Containerfile is NOT for packages

Dakota has a `Containerfile` for final OCI image assembly (copying BST artifacts into the image). It does NOT install packages. Do not add `RUN dnf5 install` to it.

## The Justfile is NOT the same as bluefin's

Dakota's `Justfile` has different semantics. `just lint` validates BST templates. There is no `just check` equivalent.
