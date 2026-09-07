# Visiblaze SCA

Find vulnerable dependencies on your own runner.

```yaml
name: Dependency scan
on: [push]

permissions:
  contents: read
  id-token: write        # required — this is how the action authenticates

jobs:
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Visiblaze/visiblaze-sca-scan@v1
        with:
          tenant: your-tenant-id
          api-url: https://<the endpoint we give you>
```

That is the whole setup. There is no secret to add.

---

## What leaves your network

Your source code is never uploaded. What does leave depends on the `mode` you run, and the
difference between the modes is the point of this action rather than a configuration detail.

| mode | what is uploaded | who decides what is vulnerable |
|---|---|---|
| `inventory` (default) | the packages your manifests declare — name, version, ecosystem, manifest | **Visiblaze**, by matching against our advisory corpus |
| `scanner` | the findings a third-party scanner produced on your runner | **your runner** |
| `both` | each of the above, uploaded independently | both, so you can compare them |

**Neither mode uploads a lockfile.** A lockfile is not source code, but it is a complete
statement of what you build with, and you did not ask us to hold that. Only the resolved
package list goes, and only the fields above.

Run `dry-run: true` first. It prints the exact payload and uploads nothing — in full, not
summarised, because deciding whether to trust this action means reading what it sends.

---

## Why there are two modes

The two answer the same question by different means, and they fail differently. Rather than
pick for you, this action can run both and let you see the difference on your own repositories.

**`inventory` sends less and checks more.** The package list is a claim about your repository's
own contents, which your runner is genuinely the authority on. We then do the matching, so
every check on our side applies: whether our advisory data is fresh enough to be trusted,
whether enough of your packages resolved to be worth a verdict, and whether a version
comparison could actually be made. When any of those fails, the result says so instead of
reporting a clean repository.

**`scanner` reaches the verdict here.** That is faster to a red build and needs nothing from
us but storage — and it has one failure mode you should know about, because we measured it: a
vulnerability database that is present but **expired** scans successfully, exits zero, and
reports nothing. Nothing in the scanner's own output distinguishes that from a clean
repository.

So this action reads the database's age and reports it, warns above seven days, and warns
again if it cannot establish the age at all. That is what `database-age-days` is for.

---

## Inputs

| input | default | notes |
|---|---|---|
| `tenant` | — | required |
| `api-url` | — | required; region-specific, we give you the exact URL |
| `mode` | `inventory` | `inventory`, `scanner` or `both` |
| `repo-key` | this repository | `github#<numeric id>`; the server rejects reporting for another repository |
| `path` | `.` | in a monorepo, one job per package pointed at its own directory |
| `fail-on` | *(empty)* | `critical`, `high`, `medium` or `low` |
| `fail-on-error` | `false` | fail the job if the scan or upload itself fails |
| `dry-run` | `false` | print the payload, upload nothing |

### `fail-on` only works in `scanner` or `both`

In `inventory` mode the verdict is reached on our side, **after this job has already
finished**. A gate here would therefore block on nothing, or block on the previous run's
answer. The action refuses to pretend otherwise: it warns and reports
`policy=not-applied` rather than passing a check that never ran.

### `fail-on` is empty by default, deliberately

A repository with existing dependency debt would otherwise be permanently red, and a check
that can never go green is one people learn to route around. Findings are reported either way.

---

## Monorepos

Point one job per package at its own directory:

```yaml
strategy:
  matrix:
    package: [services/api, services/web]
steps:
  - uses: actions/checkout@v4
  - uses: Visiblaze/visiblaze-sca-scan@v1
    with:
      tenant: your-tenant-id
      api-url: https://<endpoint>
      path: ${{ matrix.package }}
```

The action **declares the manifests it read** as that run's coverage. Without that, each job's
upload would resolve every other job's findings — a repository with three jobs would show
whichever one finished last. This is handled for you; there is nothing to configure.

---

## Authentication

GitHub's own OIDC. The action mints a token per upload that lives for minutes and states
cryptographically which repository produced it. Nothing is stored in your repository secrets,
so there is nothing to rotate, leak or revoke, and a token captured from a log cannot be
replayed against another tenant.

The two modes use **different audiences**, which is a security property rather than tidiness.
An inventory upload claims what your repository declares; a findings upload claims the verdict
itself. A token minted for the first cannot perform the second, so a compromised runner cannot
assert findings it never produced.

Your Visiblaze admin authorises your organisation or repository once, on our side. Until they
do, this action gets a `403` and says so plainly.

---

## Pinning

The action downloads up to two executables. Both are pinned to an exact version in
[`pins.env`](./pins.env) and both have their SHA-256 verified **before execution**; a mismatch
aborts and runs nothing. There is no `latest` anywhere.

The expected digests live in `pins.env`, which is committed here, rather than in a checksum
file downloaded next to each binary. The difference matters: anyone able to replace a release
asset can replace a checksum file sitting beside it, so verifying against one proves only that
the download was not corrupted in transit. A digest in a committed file changes only through a
visible commit.

Our own binary is built reproducibly — `-trimpath` and an empty `-buildid`, so the digest
depends on the source and not on the machine that built it. `pins.env` records the exact
command, and you can rebuild it and compare the number yourself.

The one thing not pinned is the third-party scanner's vulnerability database, and it cannot be —
yesterday's database cannot know about today's advisory. `pins.env` explains what that means.

It is a composite action: a shell script you can read, running on your runner. For a tool that
reads your dependency manifests, readable beats convenient.

---

## What it does not do

It reads. It does not write, comment, annotate, or open pull requests. The only thing it can do
to your pipeline is set an exit code, and only if you ask it to with `fail-on`.

---

## Licence

Apache-2.0. See [`NOTICE`](./NOTICE) for third-party attributions.
