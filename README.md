# uv multi-version pin — repro

Minimal reproduction for a problem in [renovatebot/renovate#47482](https://github.com/renovatebot/renovate/pull/47482), which surfaces `uv.lock` transitive dependencies for security updates.

## The problem

A `uv.lock` can hold the **same package at several versions**, one per resolution marker. This project asks for `requires-python = ">=3.8"` and declares a single dependency, `requests`, which leaves `urllib3` locked three times:

```toml
# uv.lock
name = "urllib3"
version = "2.2.3"   # python_full_version < '3.9'
version = "2.6.3"   # python_full_version == '3.9.*'
version = "2.8.0"   # python_full_version >= '3.10'
```

The PR maps each package name to its **lowest** locked version, so `urllib3` is reported as `2.2.3`, which many advisories affect. The lookup picks a fixed version, and the artifact step runs a **pinned** upgrade so that `vulnerabilityFixStrategy` is honoured:

```
uv lock --upgrade-package urllib3==2.8.0
```

`uv` treats that pin as a hard, universal constraint rather than a floor for the matching marker. The result is not a targeted fix:

```
$ uv lock --upgrade-package urllib3==2.8.0
Updated idna     v3.15, v3.20                -> v3.20
Updated requests v2.32.4, v2.32.5, v2.34.2   -> v2.15.1, v2.34.2
Updated urllib3  v2.2.3, v2.6.3, v2.8.0      -> v2.8.0
```

`urllib3` 2.8.0 needs Python >= 3.9, so to satisfy the pin on the `< 3.9` branch `uv` backtracks `requests` to **2.15.1**, released in 2017. A security update therefore introduces [CVE-2018-18074](https://nvd.nist.gov/vuln/detail/CVE-2018-18074) and [CVE-2023-32681](https://nvd.nist.gov/vuln/detail/CVE-2023-32681).

The other failure mode, when no such backtrack exists, is a hard error:

```
$ uv lock --upgrade-package numpy==2.2.6     # requires-python = ">=3.9"
  × No solution found when resolving dependencies
  hint: numpy==2.2.6 only supports >=3.10
```

And a pin which every marker can satisfy collapses the lock to a single entry, downgrading the others:

```
$ uv lock --upgrade-package numpy==2.0.1
Updated numpy v2.0.2, v2.2.6, v2.4.6, v2.5.4 -> v2.0.1
```

## Reproduce

```bash
git clone https://github.com/khvn26/renovate-uv-multi-version-pin-repro
cd renovate-uv-multi-version-pin-repro
uv lock --upgrade-package urllib3==2.8.0
git diff uv.lock
```

Or against the Renovate branch, to see the command it chooses:

```bash
LOG_LEVEL=debug node /path/to/renovate/lib/renovate.ts --platform=local
```

```
DEBUG: Vulnerability GHSA-8988-9cw3-xx77 affects urllib3 2.2.3
DEBUG: Setting allowed version >= 2.8.0 to fix vulnerability GHSA-8988-9cw3-xx77 in urllib3 2.2.3
       "command": "uv lock --upgrade-package urllib3==2.8.0"
```

## An unpinned upgrade is safe

`uv lock --upgrade-package urllib3` leaves the marker structure intact and upgrades each branch as far as it can go, so it never downgrades another package. It does not honour `vulnerabilityFixStrategy=lowest`, which is why the PR pins in the first place — the pin is only safe for a package locked at a single version.
