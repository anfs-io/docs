# Log: wsm-spaces/05-push-spec (r-20261023-1, core 589bacf, wsm 0.11.0)

- What: `wsm repo push [spec|repository...]` takes a space spec, resolved as `repo ls` resolves it (bare id = that space, `cnfs/` = the subtree); an exact repository name wins; expanded in list order, deduplicated, to the clones with an upstream (ahead: pushed, else `up to date`); a bare id with none of its own but children with some: `did you mean cnfs/?`; completion offers clones, installed ids and prefixes.
- Why: `--all` spans every space (it would push anfs-dev's docs too); `push cnfs/` is the scoped form. Tests: manage.bats (temp bare remotes); the full `packages/anfs/tests/all` 663 passed, 0 failed.
- By hand: `ppm install core/wsm`, then `wsm repo push -n cnfs/` (read-only); the real push is your call.

## Follow-up (core eedf1ca, wsm 0.11.1)

- `wsm repo push cnfs` where `cnfs` is no space (no space.yml) but `cnfs/apps` etc. are: was the unknown-repository error; now exit 1, `no space or repository 'cnfs'; did you mean cnfs/?`. A real space with none of its own keeps `no repositories in space ...`. Test: manage.bats (`acme/lab`, only `acme/lab/tech` defined); full suite 663 passed, 0 failed.
- By hand: `ppm install core/wsm`, then `wsm repo push -n cnfs` shows the hint.
