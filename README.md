# vuln-yarn-repo-devdep
populate devdeps into the project's yarn.lock file


# yarn: dev dependencies reported as production when `workspace:*` resolves to a `workspace:<path>` lockfile entry

## Description

In a yarn v1 workspace project, Trivy normally works out dev dependencies from the `package.json` files and drops them from the report. That fails when both of the following hold:

1. A workspace depends on another workspace using the `workspace:*` protocol in `package.json`.
2. `yarn.lock` contains that workspace as an entry whose version is a `workspace:<path>` string, not a semver version (for example `version "workspace:packages/core"`). bun writes `yarn.lock` this way (`bun install --yarn`). Other tools may as well.

Trivy logs the following, then treats **every** package in `yarn.lock` as a production dependency:

```
WARN [yarn] Unable to parse package.json to remove dev dependencies file_path="package.json"
     err="unable to resolve workspace dependencies: unable to resolve dependencies: unable to walk dependencies: unable to match version for *"
```

As a result, vulnerabilities in dev-only transitive dependencies are reported as production findings. With `--list-all-pkgs`, every package has `Dev: false` and no `Relationship`.

It looks like `workspace:*` is reduced to the constraint `*`, which then can't be matched against the non-semver version `workspace:packages/core`. One fallback failure then turns off dev-dependency detection for the whole lockfile.

## Layout

```
failing/   reproduces the bug
working/   identical except one line: "@repro/core": "*" instead of "workspace:*"
```

Both contain the same structure:

- `package.json`: private root with `"workspaces": ["packages/*"]`
- `packages/core`: `@repro/core`, depends on `ms`
- `packages/webapp`: depends on `@repro/core`, and has **`solid-js` as a devDependency**. `solid-js@1.9.15` depends on `seroval ~1.5.4`, which resolves to 1.5.6 (CVE-2026-104846).

## Steps to reproduce

```sh
trivy fs --scanners vuln ./failing
trivy fs --scanners vuln ./working

# to see the Dev / Relationship fields per package:
trivy fs --scanners vuln --list-all-pkgs -f json ./failing
```

## Expected

`solid-js`, `seroval` and `seroval-plugins` are reachable only through `webapp`'s `devDependencies`. They should be marked `Dev: true` and left out of the report, as they are for `./working`.

## Actual

| Case        | Warning                                                 | seroval 1.5.6 reported                            |
|-------------|---------------------------------------------------------|---------------------------------------------------|
| `./working` | none                                                    | no (correctly dropped as dev)                     |
| `./failing` | `Unable to parse package.json to remove dev dependencies` | yes: CVE-2026-104846 (CRITICAL) plus 5 others     |

## What I ruled out

| Variant                                                                | Result   |
|------------------------------------------------------------------------|----------|
| Lockfile written by yarn 1.22.22, `"*"` in package.json                 | correct  |
| Lockfile written by yarn 1.22.22, `"workspace:*"` in package.json       | no warning; dev deps dropped correctly, but the workspace package is reported with the name `*` |
| bun-style workspace entries in lockfile, `"*"` in package.json (`./working`) | correct |
| bun-style workspace entries in lockfile, no workspace-to-workspace dependency | correct |
| bun-style workspace entries in lockfile + `"workspace:*"` (`./failing`)  | **bug**  |

## Environment

- Trivy 0.76.2 (latest; also reproduced on 0.73.0)