# Contributing

Thanks for wanting to improve this write-up.

## Scope

**This repo is documentation.** It is not a place to:

* publish new exploits,
* host firmware or kernel dumps,
* share piracy/DRM-bypass tooling.

PRs doing any of the above will be closed.

## Good contributions

* Corrections to decompile annotations in `docs/`.
* New diagrams (prefer SVG, embed with `![](docs/img/foo.svg)`).
* Expanding the `kdlsym` index map for firmwares that are publicly documented.
* Translations of the README / docs (new file: `README.es.md`, `README.ja.md`, …).
* Linking better references.

## Workflow

1. Fork → branch → PR.
2. Keep commits small and typed:
   ```
   docs(main-flow): clarify kdlsym(0x8)+0x28 is CR3
   fix(readme): typo in resume-hook section
   ```
3. One topic per PR.
4. Rebase on `main` before asking for review.

## Reviewing policy

I'll merge anything that is:

* Technically correct (I'll re-check against the binary).
* Non-weaponizing (see Scope above).
* Formatted — markdown lint, no trailing whitespace.
