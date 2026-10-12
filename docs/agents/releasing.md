# Releasing

A release is a version of the specification on `main`. The site deploys from
`main`, so a version is released on the day it lands there, not the day its
branch was opened. Every release gets a git tag.

## Picking the version

The specification follows [Semantic Versioning](https://semver.org). Before
1.0, a breaking change (anything marked **Conformance-affecting** in the
changelog) bumps the minor version; everything else bumps the patch version.

## Steps

1. Branch `release/X.Y.Z` from an up-to-date `main`.
2. In `openapi.yaml`, set `info.version` and the `apiVersion` examples to
   `X.Y.Z`.
3. Update `apiVersion` examples in `docs/`. Leave blog posts alone: they
   describe the version they announced.
   `grep -rn 'OLD.VERSION' docs openapi.yaml` finds what is left.
4. In `CHANGELOG.md`, rename `## [Unreleased]` to `## [X.Y.Z] - YYYY-MM-DD`,
   dated the day the release PR merges.
5. Run `pnpm lint` and `pnpm build`.
6. Open a PR and merge it.
7. Tag the merged commit on `main` with a signed, annotated tag, and push it:

   ```bash
   git switch main && git pull --rebase
   git tag -s vX.Y.Z -m "vX.Y.Z"
   git push origin vX.Y.Z
   ```

Never move or delete a pushed tag. If a release is wrong, fix it in the next
version.
