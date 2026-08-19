# PMG (SafeDep Package Manager Guard) — CI Integration Report

Repo: `docker/login-action` (Razorpay fork)
Reference: https://docs.safedep.io/package-security/pmg/github-actions

Scope: **every job in every workflow** under `.github/workflows/`, regardless of
whether that job installs packages. A job is skipped only where PMG *cannot* run
without breaking it.

## Summary

| Workflow | Jobs | PMG integrated | Skipped |
|---|---|---|---|
| `ci.yml` | 13 | 13 | 0 |
| `test.yml` | 1 | 1 | 0 |
| `genesis.yml` | 1 | 0 | 1 |
| **Total** | **15** | **14** | **1** |

Nine of the 13 `ci.yml` jobs are `ubuntu-latest` + `windows-latest` matrices; the
Windows legs carry no PMG (the action is Linux-only). See *Constraint 1*.

## The applied pattern

```yaml
    - name: Checkout
      uses: actions/checkout@v3
    - name: Setup PMG proxy               # after checkout / toolchain setup
      uses: safedep/pmg@v1
      with:
        server-mode: true
        api-key: ${{ secrets.PMG_PUBLIC_REPOS_TOKEN }}
        tenant-id: ${{ secrets.PMG_TENANT_ID }}
    # ... existing job steps, unchanged ...
    - name: Enforce PMG policy            # must be the LAST step in the job
      if: always()
      run: pmg proxy stop --fail-on-violation
```

Invariants, machine-verified across all 14 integrated jobs:

- Setup comes **after** `actions/checkout`, so the action source is on disk before
  any proxy is in play.
- Enforce is **the last step** in every job. `pmg proxy stop` leaves `HTTP_PROXY`
  set, so any network step after it would point at a dead proxy.
- Enforce carries `if: always()`, so violations surface even when an earlier step
  fails.

---

## Test results — does PMG break anything?

Tested against `pmg 0.25.0` (commit d9240e) locally, driving the real proxy the
way the action does (`pmg proxy start --daemon` + `pmg proxy env`). **Nothing
broke.** Full detail below.

| # | Test | Result |
|---|---|---|
| T1 | `docker login ghcr.io` through the proxy | ✅ `Login Succeeded` |
| T2 | `docker manifest inspect alpine:3.19` (Docker Hub anon auth + manifest) | ✅ succeeded |
| T3 | Proxy daemon survives a full job's worth of steps | ✅ still running |
| T4 | `npm install lodash` through the proxy | ✅ succeeded — no false positive |
| T5 | `npm install safedep-test-pkg@0.1.3` (known-flagged) | ✅ **blocked**, HTTP 403 on the tarball |
| T6 | `pmg proxy stop --fail-on-violation` after a violation | ✅ **exit 1**, `1 package(s) were blocked` |
| T7 | `docker buildx bake validate` (the real `test.yml` build) | ✅ exit 0, build succeeded |
| T8 | PMG's view of that build | ⚠️ `0 package(s) blocked` — the in-container `yarn install` is invisible to PMG |

### Why nothing breaks: PMG intercepts selectively

The concern going in was that `server-mode: true` exports job-wide
`HTTP_PROXY`/`HTTPS_PROXY`, the Docker CLI is a Go binary honouring
`http.ProxyFromEnvironment`, and PMG advertises its CA only through
`SSL_CERT_FILE` / `NODE_EXTRA_CA_CERTS` / `REQUESTS_CA_BUNDLE` / `PIP_CERT` /
`YARN_HTTPS_CA_FILE_PATH` — none of which Docker reads. That would have made every
`docker login` in this repo fail certificate verification.

**It does not happen, because PMG only MITMs package-registry hosts.** Observed
TLS issuers through the proxy:

| Host | Certificate issuer seen | Behaviour |
|---|---|---|
| `registry.npmjs.org` | `O=SafeDep PMG; CN=SafeDep PMG Proxy CA` | **intercepted** |
| `registry.yarnpkg.com` | `O=SafeDep PMG; CN=SafeDep PMG Proxy CA` | **intercepted** |
| `auth.docker.io` | `C=US; O=Let's Encrypt; CN=YE1` | plain CONNECT tunnel |
| `ghcr.io` | `C=GB; O=Sectigo Limited; …` | plain CONNECT tunnel |
| `public.ecr.aws` | `C=US; O=Amazon; CN=Amazon RSA 2048 M01` | plain CONNECT tunnel |

Container-registry traffic reaches the origin with its **real** certificate
untouched, so Docker's trust chain is never involved. Verified end to end: with
the proxy provably in path (`ghcr.io` traffic routed via a proxy `CONNECT`),
`docker login ghcr.io` still returned `Login Succeeded`.

**Consequence:** the `NO_PROXY` registry-exemption steps carried in the first draft
of this integration were unnecessary and have been removed. The workflows now
carry only the setup and enforce steps.

### Enforcement genuinely works

T5/T6 confirm the proxy blocks on its own — no shell shim or `pmg npm` wrapper
needed, which is what makes the CI pattern viable. The flagged package was refused
with a 403 on the tarball fetch, and the enforce step exited 1 with
`ProxyPolicyViolation — 1 package(s) were blocked by the proxy`. A job with a
malicious dependency will go red.

### The gap T8 exposes

`docker buildx bake validate` built cleanly under the proxy, and the build log
confirms the install genuinely ran (`yarn install v1.22.19`,
`[1/4] Resolving packages...` — not a cache hit). PMG still reported **0 packages
blocked**: it logged 11 events, all from buildx's own registry traffic, and saw
none of the yarn packages.

That is the expected result — BuildKit executes `RUN` steps in the daemon's network
namespace, which does not inherit the runner's proxy env, and `node:16-alpine`
carries no PMG CA. **It also means PMG currently guards nothing in this repo**; see
*Recommendations*.

### Test caveats

- Run on macOS with the standalone `pmg` binary. CI uses `safedep/pmg@v1` on
  `ubuntu-latest`. The selective-interception behaviour is a property of the pmg
  binary's host-matching, not of the OS, so it should transfer — but the exact
  intercept list is version-dependent and could widen in a future release.
- The proxy daemon appeared to die between test batches. This was an artifact of
  the test harness — each shell invocation exited and took its child process group
  with it. Inside a single shell (T1–T8 above) the daemon stayed up throughout,
  which matches how a GitHub job runs. **Not a PMG defect.**
- Windows matrix legs were not exercised; the action refuses to run there by design.

---

## The two constraints that shaped placement

### 1. `safedep/pmg@v1` is Linux-only

The composite action explicitly checks `runner.os == 'Linux'` and exits otherwise.
Nine `ci.yml` jobs are ubuntu + windows matrices, so setup and enforce are gated on
`if: runner.os == 'Linux'` (`if: always() && runner.os == 'Linux'` for enforce).
Unguarded, the Windows leg of all nine would fail outright.

**This is the "PMG can break it" exemption, applied at matrix-leg granularity
rather than dropping whole jobs.** Nothing goes unguarded: those jobs install zero
packages.

### 2. Nothing in this repo installs packages on the runner

All 13 `ci.yml` jobs check out, run the local action against a registry, and assert
the login worked. `test.yml` delegates its install to BuildKit. So PMG is present
for uniform coverage, not because there is anything to guard — which is also why
the registry-exemption question turned out to be moot.

---

## Per-workflow / per-job detail

### `ci.yml` — Registry login matrix (nightly `schedule`, `push`, `workflow_dispatch`)

| # | Job | Runner(s) | PMG | Notes |
|---|---|---|---|---|
| 1 | `stop-docker` | ubuntu | ✅ | Stops `dockerd` mid-job; PMG installs via `curl`/`tar` and is unaffected. Setup precedes the stop step so daemon state never matters. |
| 2 | `logout` | ubuntu (matrix ×2) | ✅ | Matrix is over the `logout` input, not the OS — both legs covered. |
| 3 | `dind` | ubuntu | ✅ | Has a container action (`uses: docker://docker`) and a `docker pull`; both are daemon-side and do not read the job's `HTTP_PROXY`. |
| 4 | `acr` | ubuntu | ✅ | |
| 5 | `dockerhub` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 6 | `ecr` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 7 | `ecr-aws-creds` | ubuntu + **windows** | ⚠️ Linux leg only | `configure-aws-credentials@v1` may call STS; tunnelled, not intercepted. |
| 8 | `ecr-public` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 9 | `ecr-public-aws-creds` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 10 | `github-container` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 11 | `gitlab` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 12 | `google-artifact` | ubuntu + **windows** | ⚠️ Linux leg only | |
| 13 | `google-container` | ubuntu + **windows** | ⚠️ Linux leg only | |

**No `permissions:` block was added.** Jobs 1, 2 and 10 use `GITHUB_TOKEN` as the
registry password for `ghcr.io`, which needs the `packages` scope — see *Why
`permissions:` was deliberately left out* below.

### `test.yml` — Build and test (`push`, `pull_request`)

#### `test` — ✅ integrated (coverage is nominal, see T8)

PMG sits after checkout; enforce is last, after the Codecov upload — deliberately,
since `pmg proxy stop` leaves `HTTP_PROXY` pointing at a dead daemon. Verified
green end to end (T7).

The `yarn install` this job triggers happens inside BuildKit and is not visible to
PMG (T8). To actually guard it, PMG has to move into `dev.Dockerfile` or the
install has to happen on the runner.

No `permissions:` block added, for consistency with `ci.yml`.

### `genesis.yml` — Quality Checks (nightly `schedule`)

#### `Analysis` — ❌ NOT integrated (structurally impossible)

```yaml
jobs:
  Analysis:
    uses: razorpay/genesis/.github/workflows/quality-checks.yml@master
    secrets: inherit
```

A reusable-workflow call, not a `steps:` job. The Actions schema forbids `steps:`
alongside `uses:`, so adding PMG here would make the file invalid and the nightly
job would not run at all. There is no caller-side hook — no pre/post step, no way
to inject env into the callee's runner.

**This is the one hard skip, and it is a "PMG would break it" case in the strongest
sense: the file would stop parsing.**

Coverage must be added inside
`razorpay/genesis/.github/workflows/quality-checks.yml`. Every repo calling it
inherits the coverage once it lands there.

`permissions:` was deliberately **not** added: the callee uses `secrets: inherit`
and its required token scopes are unknown, so restricting them from the caller
risks breaking the nightly checks.

`genesis.yml` is left **byte-identical to HEAD** — no PMG, so no edit.

---

## Changes made

- `ci.yml` — PMG setup + enforce in all 13 jobs,
  `runner.os == 'Linux'`-gated on the 9 jobs with a Windows matrix leg.
- `test.yml` — PMG setup + enforce in the single job.
- `genesis.yml` — **unchanged.** It receives no PMG, so the file is not touched.
- All three files re-parsed and the PMG invariants verified programmatically. The
  two edited files are byte-identical to HEAD once the PMG steps are removed — no
  existing line was modified or deleted, and nothing outside PMG was added.

## Secrets required

| Secret | Used by |
|---|---|
| `PMG_PUBLIC_REPOS_TOKEN` | all 14 integrated jobs (`api-key`) |
| `PMG_TENANT_ID` | all 14 integrated jobs (`tenant-id`) |

Names match the convention already used in the sibling forks (`checkout-action`,
`create-comment`). Both are optional per SafeDep's docs — PMG still blocks locally
without them — but they are needed to sync events to SafeDep Cloud before the runner
is destroyed. Absent secrets resolve to empty strings and the action runs local-only;
nothing fails.

## Why `permissions:` was deliberately left out

The PMG pattern documented in the workspace `CLAUDE.md` pairs the proxy steps with
`permissions: contents: read`. **That is not safe in this repo and was not applied.**

Per GitHub's workflow-syntax docs: *"If you specify the access for any of these
permissions, all of those that are not specified are set to `none`."* A block of
`contents: read` therefore sets `packages: none`.

Three `ci.yml` jobs — `stop-docker`, `logout`, and `github-container` — use
`${{ secrets.GITHUB_TOKEN }}` as the **registry password for ghcr.io**. GitHub's own
ghcr.io examples pair that with `packages: write`. Zeroing the `packages` scope
risks failing the very logins those jobs exist to assert. (`dind` uses a PAT,
`GHCR_PAT`, and is unaffected.)

`permissions:` is unrelated to PMG — the proxy needs no token scope at all — so the
hardening was dropped rather than bundled into a PMG rollout. Upstream
`docker/login-action` sets no `permissions:` block either. If the repo wants it
later, the correct block for `ci.yml` is `contents: read` **plus** `packages: read`,
landed as its own change.

## Known limitations

1. **PMG guards nothing in this repo today.** No job installs packages on the
   runner (T8). This is coverage in form.
2. **The real install target is inside BuildKit.** `dev.Dockerfile`'s `yarn install`
   is where this repo's supply-chain exposure lives, and a runner-level proxy cannot
   reach it.
3. **Non-Linux runners are unsupported.** Nine Windows matrix legs cannot carry PMG.
4. **Reusable-workflow calls cannot be instrumented from the caller** — `genesis.yml`
   coverage must be added in `razorpay/genesis`.
5. **Interception scope is version-dependent.** If a future pmg release widens its
   MITM host list to cover container registries, the `docker login` jobs would need
   `NO_PROXY` exemptions reinstated. Re-run the issuer probe (see *Test results*)
   after upgrading the action.

## Recommendations (not done — out of scope of this pass)

- **Cover the real install path.** Either add PMG inside `dev.Dockerfile`'s `deps`
  stage, or add a runner-side `yarn install --frozen-lockfile` verification job
  behind PMG. The latter is cheap and gives genuine lockfile coverage on every PR.
- **Add a `pmg-test.yml` verification workflow** (as `checkout-action` has) that
  installs a known-clean and a known-flagged package, so enforcement is re-proven on
  the actual runners rather than locally.
- **Push PMG into `razorpay/genesis`** to close the `genesis.yml` gap for every repo
  that calls it.
