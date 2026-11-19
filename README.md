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

## Output with Debugs
```sh
@rahg0 ➜ /workspaces/vuln-yarn-repo-devdep/failing (main) $ trivy fs . --debug
2026-11-19T08:01:36Z    DEBUG   Default config file "file_path=trivy.yaml" not found, using built in values
2026-11-19T08:01:36Z    DEBUG   Cache dir       dir="/home/codespace/.cache/trivy"
2026-11-19T08:01:36Z    DEBUG   Cache dir       dir="/home/codespace/.cache/trivy"
2026-11-19T08:01:36Z    DEBUG   Parsed severities       severities=[UNKNOWN LOW MEDIUM HIGH CRITICAL]
2026-11-19T08:01:36Z    DEBUG   Ignore statuses statuses=[]
2026-11-19T08:01:36Z    DEBUG   DB update was skipped because the local DB is the latest
2026-11-19T08:01:36Z    DEBUG   DB info schema=2 updated_at=2026-11-19T01:05:40.842299544Z next_update=2026-11-20T01:05:40.842299414Z downloaded_at=2026-11-19T08:01:18.959615544Z
2026-11-19T08:01:36Z    DEBUG   [pkg] Package types     types=[os library]
2026-11-19T08:01:36Z    DEBUG   [pkg] Package relationships     relationships=[unknown root workspace direct indirect]
2026-11-19T08:01:36Z    INFO    [vuln] Vulnerability scanning is enabled
2026-11-19T08:01:36Z    INFO    [secret] Secret scanning is enabled
2026-11-19T08:01:36Z    INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2026-11-19T08:01:36Z    INFO    [secret] Please see https://trivy.dev/docs/v0.76/guide/scanner/secret#recommendation for faster secret detection
2026-11-19T08:01:36Z    DEBUG   Initializing scan cache...      type="memory"
2026-11-19T08:01:36Z    DEBUG   [secret] No secret config detected      config_path="trivy-secret.yaml"
2026-11-19T08:01:36Z    DEBUG   [notification] Running version check and sending anonymous telemetry
2026-11-19T08:01:36Z    DEBUG   [fs] Analyzing...       root="."
2026-11-19T08:01:36Z    DEBUG   Created process-specific temp directory path="/tmp/trivy-3102852627"
2026-11-19T08:01:36Z    DEBUG   [secret] Using streaming scanner        file_path="package.json"
2026-11-19T08:01:36Z    DEBUG   [secret] scanStream called      file_path="package.json" buffer_size=65536 overlap_size=4096
2026-11-19T08:01:36Z    DEBUG   [secret] scanChunk called       file_path="package.json" content_len=87 num_rules=105
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="package.json" rule_id="maven-settings-password"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="package.json" rule_id="maven-settings-passphrase"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="package.json" rule_id="maven-settings-security-master"
2026-11-19T08:01:36Z    DEBUG   [secret] scanChunk called       file_path="package.json" content_len=87 num_rules=105
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="package.json" rule_id="maven-settings-password"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="package.json" rule_id="maven-settings-passphrase"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="package.json" rule_id="maven-settings-security-master"
2026-11-19T08:01:36Z    DEBUG   [secret] Using streaming scanner        file_path="packages/webapp/package.json"
2026-11-19T08:01:36Z    DEBUG   [secret] scanStream called      file_path="packages/webapp/package.json" buffer_size=65536 overlap_size=4096
2026-11-19T08:01:36Z    DEBUG   [secret] scanChunk called       file_path="packages/webapp/package.json" content_len=175 num_rules=105
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/webapp/package.json" rule_id="maven-settings-password"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/webapp/package.json" rule_id="maven-settings-passphrase"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/webapp/package.json" rule_id="maven-settings-security-master"
2026-11-19T08:01:36Z    DEBUG   [secret] scanChunk called       file_path="packages/webapp/package.json" content_len=175 num_rules=105
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/webapp/package.json" rule_id="maven-settings-password"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/webapp/package.json" rule_id="maven-settings-passphrase"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/webapp/package.json" rule_id="maven-settings-security-master"
2026-11-19T08:01:36Z    DEBUG   [secret] Using streaming scanner        file_path="packages/core/package.json"
2026-11-19T08:01:36Z    DEBUG   [secret] scanStream called      file_path="packages/core/package.json" buffer_size=65536 overlap_size=4096
2026-11-19T08:01:36Z    DEBUG   [secret] scanChunk called       file_path="packages/core/package.json" content_len=111 num_rules=105
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/core/package.json" rule_id="maven-settings-password"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/core/package.json" rule_id="maven-settings-passphrase"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/core/package.json" rule_id="maven-settings-security-master"
2026-11-19T08:01:36Z    DEBUG   [secret] scanChunk called       file_path="packages/core/package.json" content_len=111 num_rules=105
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/core/package.json" rule_id="maven-settings-password"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/core/package.json" rule_id="maven-settings-passphrase"
2026-11-19T08:01:36Z    DEBUG   [secret] Skipped secret scanning as non-compliant to the rule   file_path="packages/core/package.json" rule_id="maven-settings-security-master"
2026-11-19T08:01:36Z    DEBUG   [yarn] Unable to traverse licenses      err="fs error: file does not exist"
2026-11-19T08:01:36Z    WARN    [yarn] Unable to parse package.json to remove dev dependencies  file_path="package.json" err="unable to resolve workspace dependencies: unable to resolve dependencies: unable to walk dependencies: unable to match version for *"
2026-11-19T08:01:36Z    DEBUG   OS is not detected.
2026-11-19T08:01:36Z    DEBUG   Detected OS: unknown
2026-11-19T08:01:36Z    INFO    Number of language-specific files       num=1
2026-11-19T08:01:36Z    INFO    [yarn] Detecting vulnerabilities...
2026-11-19T08:01:36Z    DEBUG   [yarn] Scanning packages for vulnerabilities    file_path="yarn.lock"
2026-11-19T08:01:36Z    DEBUG   Specified ignore file does not exist    file=".trivyignore"
2026-11-19T08:01:36Z    DEBUG   [vex] VEX filtering is disabled

Report Summary

┌───────────┬──────┬─────────────────┬─────────┐
│  Target   │ Type │ Vulnerabilities │ Secrets │
├───────────┼──────┼─────────────────┼─────────┤
│ yarn.lock │ yarn │        6        │    -    │
└───────────┴──────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


yarn.lock (yarn)

Total: 6 (UNKNOWN: 0, LOW: 0, MEDIUM: 1, HIGH: 4, CRITICAL: 1)

┌─────────┬─────────────────┬──────────┬────────┬───────────────────┬───────────────┬──────────────────────────────────────────────────────────────┐
│ Library │  Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                            Title                             │
├─────────┼─────────────────┼──────────┼────────┼───────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ seroval │ CVE-2026-104846 │ CRITICAL │ fixed  │ 1.5.6             │ 1.6.2         │ seroval: Seroval: `fromJSON()` Promise thenable assimilation │
│         │                 │          │        │                   │               │ invokes plugin-produced callables (bypass of                 │
│         │                 │          │        │                   │               │ CVE-2026-59940)...                                           │
│         │                 │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2026-104846                  │
│         ├─────────────────┼──────────┤        │                   ├───────────────┼──────────────────────────────────────────────────────────────┤
│         │ CVE-2026-104845 │ HIGH     │        │                   │ 1.6.3         │ seroval: seroval: Denial of Service via unbounded memory     │
│         │                 │          │        │                   │               │ allocation during deserialization                            │
│         │                 │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2026-104845                  │
│         ├─────────────────┤          │        │                   ├───────────────┼──────────────────────────────────────────────────────────────┤
│         │ CVE-2026-116580 │          │        │                   │ 1.6.8         │ seroval: seroval: Unintended function execution via crafted  │
│         │                 │          │        │                   │               │ serialized input                                             │
│         │                 │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2026-116580                  │
│         ├─────────────────┤          │        │                   │               ├──────────────────────────────────────────────────────────────┤
│         │ CVE-2026-116581 │          │        │                   │               │ seroval: seroval: Denial of Service via crafted sequence     │
│         │                 │          │        │                   │               │ node during deserialization                                  │
│         │                 │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2026-116581                  │
│         ├─────────────────┤          │        │                   │               ├──────────────────────────────────────────────────────────────┤
│         │ CVE-2026-116584 │          │        │                   │               │ seroval: seroval: Denial of Service via manipulated array    │
│         │                 │          │        │                   │               │ length during deserialization                                │
│         │                 │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2026-116584                  │
│         ├─────────────────┼──────────┤        │                   ├───────────────┼──────────────────────────────────────────────────────────────┤
│         │ CVE-2026-116582 │ MEDIUM   │        │                   │ 1.6.10, 1.0.0 │ seroval: seroval: Cross-site scripting via improper regular  │
│         │                 │          │        │                   │               │ expression serialization                                     │
│         │                 │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2026-116582                  │
└─────────┴─────────────────┴──────────┴────────┴───────────────────┴───────────────┴──────────────────────────────────────────────────────────────┘
2026-11-19T08:01:36Z    DEBUG   Cleaning up temp directory      path="/tmp/trivy-3102852627"
```
