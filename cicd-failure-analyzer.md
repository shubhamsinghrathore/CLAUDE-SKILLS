---
name: cicd-failure-analyzer
description: >
  Autonomous CI/CD failure triage and remediation framework for GitHub Actions,
  GitLab CI, Jenkins, Tekton, CircleCI, and ArgoCD. Use this skill for ANY
  pipeline failure: red builds, failing or flaky tests, dependency/lockfile
  breakage, runner contamination, Docker-in-Docker problems, OOMKilled build
  jobs, pod evictions, security gate failures, or "the pipeline is randomly
  failing". Trigger even for a single pasted log snippet or a vague "CI is
  broken", because the classification tree here applies to every failure.
  Every recommendation ships with a quantified confidence score and risk
  assessment.
reasoning_effort: extra-high
compatibility: "any CI/CD system with accessible logs; optional tools: kubectl, docker, npm/pip audit, trivy"
---

# Autonomous CI/CD Failure Analyzer

Operate as a Principal SRE with EXTRA HIGH reasoning effort. Two non-negotiable
rules govern every output:

1. **No suggestion without quantified uncertainty.** Every root cause carries a
   Confidence Score (Section 6 formula); every fix carries a Risk Assessment.
   A diagnosis you cannot score is a hypothesis, and hypotheses are labeled as
   such with the next diagnostic step attached.
2. **Evidence over pattern-matching.** Cite the exact log line, exit code,
   metric, or timestamp behind every claim. If the logs provided are
   insufficient, the deliverable is the minimal list of commands/artifacts
   needed to disambiguate, not a guess dressed as an answer.

**Task routing:**
- Pasted failure log / red build → Section 1 classification, then Section 4 RCA+fix.
- "Test sometimes fails" / retry fixed it → Section 2 flaky protocol.
- Build broke with no code change → Section 3 dependency/environment audit first.
- Docker/K8s runner errors, OOMKilled, evictions → Section 5.
- Security scanner blocking the pipeline → Section 5.2.
- Confidence below threshold or risk HIGH+ → Section 6.3 escalation.

---

## Section 1: Deep-Trace Log Parsing & Classification

### 1.1 Layer 1: Failure Origin Detection

Classify into exactly one primary origin. Signatures are entry points, not
verdicts; confirm with temporal markers before committing.

**A. Application code failure**
- Signatures: non-zero exit with stack trace; Java `Exception in thread`/
  `NullPointerException`; Python `Traceback`/`ImportError`/`AssertionError`;
  Go `panic:`/`fatal error:`; Node `TypeError`/`ReferenceError`; compiler
  errors referencing changed files.
- Temporal marker: deterministic, same step, every run; first appeared with a
  specific commit.
- Next move: map stack trace frames to files changed in the failing
  commit/PR diff (`git diff HEAD~1 --stat`). A trace pointing at unchanged code
  usually means the change is in a caller or a dependency, not the frame.

**B. Infrastructure / runtime failure**
- Signatures: `OOMKilled`, bare `Killed`/SIGKILL (exit 137), `context deadline
  exceeded`, `No space left on device`, `Cannot allocate memory`,
  `503 Service Unavailable` from internal services, runner lost/agent
  disconnected.
- Temporal marker: failure correlates with resource pressure or runner
  identity, not with code; different steps fail across runs.
- Next move: runner metrics (`df -h`, `free -m`, `docker system df`), node
  conditions if K8s-hosted, and whether the same commit passes on rerun.

**C. Configuration / drift failure**
- Signatures: `connection refused`, `Host not found`, `SSL certificate verify
  failed`, `Invalid credentials`/401/403, `Config file not found`, undefined
  env var (`$FOO: unbound variable`).
- Temporal marker: started at a clock time, not a commit; correlates with a
  deploy, secret rotation, certificate expiry date, or infra change.
- Next move: diff env vars between last-green and first-red run; check secret
  rotation/expiry timestamps; verify the endpoint independently (`curl -v`).

**D. Intermittent / flaky failure**
- Signatures: same commit passes and fails; retry succeeds with zero changes;
  failure moves between tests.
- Quantified threshold: pass-rate variance > 20% over recent runs of the same
  code = flaky. Route to Section 2; do NOT write an RCA pretending it is
  deterministic.

### 1.2 Layer 2: Contextual Correlation

Build the causality chain across steps, both directions:

- **Backward** (what caused failing step X): walk steps 1..X-1 for triggering
  events: env var mutations, dependency install output (version actually
  resolved), cache restore hits/misses, infra events (runner replaced,
  autoscale), and external changes (API deprecations) in the failure window.
- **Forward** (what did change Y poison): a change in step 2 (e.g., different
  package version cached) can detonate in step 9; trace the artifact, not the
  step number.
- **Output format**: `[Step 2: cache restored stale node_modules] →
  [Step 4: webpack uses old loader API] → [Step 4: build error TS2307]`,
  each link annotated with its own confidence. If overall chain confidence
  < 70%, list the strongest alternative chain alongside it.

### 1.3 Layer 3: Severity & Impact Scoring

Attach to every classified failure:

- **Criticality**: Critical (blocks prod release / security / data loss) |
  High (blocks merge to main) | Medium (failing but workaround exists) |
  Low (informational).
- **Blast radius**: single test (<10 tests) | single job | entire pipeline |
  multiple pipelines | production-impacting.
- **TTR estimate**: <5 min (config/timeout) | 5–30 min (clear root cause) |
  30 min–2 h (investigation/code change) | >2 h (architectural; consider
  escalation per 6.3).

---

## Section 2: Flaky Test Elimination Protocol

### 2.1 Historical pattern analysis (run for every suspected flake)

1. **History**: pull the last 20–50 runs of the test (same suite/environment),
   with timestamp, pass/fail, duration, runner type, branch. Fewer than 5
   runs → label "insufficient pattern data", recommend a 20× rerun job before
   concluding anything.
2. **Metrics**:
   - `pass_rate = passes / total_runs × 100`; below 90% = quarantine candidate.
   - Clustering: if >60% of failures land in <30% of time windows
     (day-of-week, hour, post-deploy, branch), clustering is significant and
     points at environment, not the test.
   - Cross-environment variance: pass-rate gap >30% between runner types or
     branches = environmental dependency; record which.
3. **Root-cause class** (pick by targeted experiment, not vibes):
   - **Race condition**: fails only in parallel. Test: 50× serial (passes?)
     then parallel with the suite (fails?). Look for shared ports, files,
     DB rows, non-unique IDs. Fix: isolation, unique IDs, locks.
   - **Network latency**: timeouts, DNS delays. Test: log request start/end;
     correlate failures with latency spikes. Fix: raise timeout 1.5–2×, add
     exponential backoff (3 retries: 1s/2s/4s), or mock the external service.
   - **DB state pollution**: order-dependent results. Test: shuffle test
     order; outcome changes = confirmed. Fix: per-test transactions,
     explicit teardown, snapshots.
   - **Timing-dependent assertions**: `sleep()`, `Date.now()`, hardcoded
     dates. Test: run with mocked/advanced clock. Fix: event-driven waits,
     time-mocking libraries, parameterized dates.
   - **Noisy neighbor**: passes on dedicated runner, fails on shared under
     load. Test: monitor CPU/mem during execution vs co-scheduled jobs.
     Fix: bigger/isolated runner, retries with backoff, circuit breakers.

### 2.2 Fix-or-quarantine decision tree

```
IF criticality == CRITICAL AND pass_rate < 70%:
  → QUARANTINE now (@Flaky/@Skip + ticket). It is blocking too much to fix in place.
ELSE IF root_cause IN (race_condition, db_pollution):
  → FIX (1–2 h): isolation/locks/teardown. Validate: 100 parallel iterations, >99% pass.
ELSE IF root_cause == network_latency:
  → FIX (1–3 h): timeout 1.5–2× + backoff. Validate: >95% pass under network load.
ELSE IF root_cause == timing_dependency:
  → FIX (2–4 h): event-driven waits + time mocking. Validate: >95% pass with clock advanced 1 h.
ELSE IF root_cause == noisy_neighbor:
  → FIX (1–2 h): resources/retries. Validate: >95% isolated, >85% over 50 shared runs.
ELSE IF estimated_fix_effort > 4 h AND criticality < HIGH:
  → QUARANTINE + document + sprint ticket. Critical-path tests get the hours, not this one.
ELSE:
  → FIX + add observability (structured start/end/duration logs, flakiness metric).
    Validate: 50 runs, >95% pass.
```

### 2.3 Per-flake output format (mandatory)

```
#### Test: module.Class.method  (file:line)
Flakiness: X% over N runs | Severity | Clustering: pattern or none |
Env variance: yes/no (which)
Root cause: [class] — Evidence: [log excerpt, metric, order-experiment result]
Action: Fix | Quarantine — Effort: X h — Confidence: X% (per Section 6 formula)
Implementation: [exact diff]
Validation: [iterations, condition, required pass rate]
Prevention: [lint rule / CI flake-detector job / doc note] (max 2 items)
```

---

## Section 3: Dependency & Environment Audit Engine

### 3.1 Upstream poisoning detection

Run when a build breaks without a code change, or after dependency updates.

1. **Parse lockfiles** (package-lock.json / yarn.lock / pnpm-lock.yaml /
   poetry.lock / Pipfile.lock / Gemfile.lock / go.sum / Cargo.lock /
   gradle.lockfile): extract declared ranges, resolved versions, and the
   transitive graph. Diff the lockfile between last-green and first-red
   commit FIRST; the answer is usually in that diff.
2. **Anomaly classes** (flag each found):
   - Unexpected version jump within a range (`^4.17.0` resolved 4.18.x today
     but 4.17.x last week): a new release landed; check its changelog/diff for
     breaking changes shipped as a minor/patch.
   - Pre-release/RC versions (-alpha/-beta/-rc) in a lockfile: unvetted, flag.
   - Deprecated/EOL packages: flag with the registry deprecation notice.
   - Known CVEs in transitive deps (`npm audit`, `pip-audit`, trivy):
     determine exploitability: is the vulnerable function reachable from our
     code (`grep -r` the API)? Severity is CVSS × reachability, not CVSS alone.
   - Transitive version conflicts (direct requires lodash ^4, something pins
     3.x): identify the puller (`npm ls lodash`, `pipdeptree -r -p lodash`).
3. **Resolution simulation**: for each anomaly, answer "why was this version
   selected": new release date vs lockfile date, which dependent forced it,
   whether the install bypassed the lockfile (`npm install` in CI instead of
   `npm ci` is itself a critical finding: report it).
4. **Output**: anomaly, severity, affected package, locked vs expected
   version, root cause (who pulled it and why), risk (security/compat/
   functional), exact pin/upgrade command, and validation (`npm ci` clean +
   full suite green).

### 3.2 Runner contamination detection

Run when failures correlate with specific runners or appear after long runner
uptime. All checks are read-only diagnostics; cleanups are listed separately
with risk labels.

- **Disk**: `df -h` (alert: <10% free CRITICAL, 10–20% HIGH);
  `docker system df` (dangling images+volumes >5 GB);
  `du -sh /tmp /var/log $CI_WORKDIR` (any >2 GB).
- **Processes**: leftover language runtimes from prior runs
  (`ps aux | grep -E 'java|python|node' | grep -v grep`, >20 = contaminated);
  zombies (`ps aux | awk '$8=="Z"'`, >5 = reaper problem);
  stopped containers (`docker ps -a`, >20 = no cleanup step).
- **Network**: stale DNS (compare `dig +short` vs `nslookup` for one known
  host); port squatting on common ports (`ss -tlnp | grep -E
  ':3000|:5000|:8080'` before the job binds them); missing default route.
- **Security context**: docker.sock ownership vs the CI user's groups;
  SELinux/AppArmor denials in audit logs when tests fail with EPERM.
- **Remediation**: cleanup commands (`docker system prune -af --volumes`,
  `rm -rf ${TMPDIR:?}/*`) each carry a risk note (prune kills caches: cold
  builds get slower), plus the prevention: a scheduled/ pre-job cleanup step
  or ephemeral runners. Ephemeral runners are the Tier 3 fix for every
  contamination class; say so whenever contamination recurs.

---

## Section 4: Auto-Remediation & Patching Engine

### 4.1 RCA structure (mandatory for every deterministic failure)

```
## RCA: <title>
Failure signature: exact error (≤200 chars) | exit code | first 10 stack frames
Timeline: T+0 [line N] event → T+x [line M] progression → T+y failure,
          each with the log excerpt and any metric reading
Primary cause (Confidence: X% per Section 6):
  - Technical cause in 1–2 sentences
  - Why it produces THIS symptom (2–3 sentences)
  - Evidence: log line + metric + temporal correlation + comparative
    (works-on-A/fails-on-B) — cite at least two independent types
Contributing factors: amplifiers and enabling conditions, each with confidence
Red herrings: what the logs misleadingly suggested and what ruled it out
Alternative hypotheses: ranked, each with the evidence that refutes it
  (mandatory whenever confidence < 85%)
```

### 4.2 Corrective Action Plan: three tiers, always

Deliver all three tiers for every RCA. Skipping Tier 3 is how the same
incident returns next quarter.

- **Tier 1 — Immediate mitigation** (<15 min, low risk): config/env/timeout/
  retry/feature-flag change. Include the exact diff or YAML before/after,
  why it reduces failure rate, the one-line rollback, and what metric to
  watch for 24 h. Tier 1 treats the symptom and must be labeled as such.
- **Tier 2 — Root fix** (1–4 h, medium risk): the code/dependency/test change
  that removes the cause. Include the diff, the tests that prove it
  (unit/integration, and an N-iteration soak for formerly flaky paths),
  expected failure-rate delta (e.g., 10% → <0.5%), rollout (branch → staging
  → flag → canary), regression risk % with the most likely regression named
  and its mitigation.
- **Tier 3 — Long-term prevention** (sprint-scale): the structural change that
  makes the failure class impossible or loudly visible: ephemeral runners,
  bigger runner spec, hermetic builds, lockfile-only installs in CI, a flake
  quarantine lane, dependency-update bot with CI gating. Include phases with
  owners/duration/success criteria, alternatives considered with the reason
  the chosen one wins, and an honest ROI line: current failures/week, expected
  after, cost, payback period. If ROI is poor, the honest recommendation is
  "defer", and say so.

### 4.3 Risk assessment matrix (attached to every proposed fix)

Rate and justify each dimension in a table: Blast radius (single test → prod),
Regression risk (%), Performance impact, Rollback speed (instant / 5 min /
1 h / 1 d+), Dependency risk, Security risk, Cost. Then derive the gate:

```
CRITICAL risk → manual approval (security + eng lead), staged 5%→25%→100%
               over ≥3 days, monitoring every 5 min
HIGH        → manual approval + feature flag, 1–2 days, monitor every 30 min
MEDIUM      → peer review + 10% canary, 1 day, monitor every 30 min
LOW         → may auto-apply, immediate, hourly monitoring with anomaly alerts
```

Do not deflate ratings to unlock auto-apply; an understated risk score in an
autonomous system is the most dangerous output this skill can produce.

---

## Section 5: Multi-Stack Edge Cases

### 5.1 Kubernetes & container-native runners

**Docker-in-Docker (DinD)**
- `Cannot connect to the Docker daemon` / `dial tcp: lookup docker ... no such
  host`: distinguish the two architectures first: socket-mount (job talks to
  host daemon via /var/run/docker.sock) vs true DinD service (daemon sidecar,
  DOCKER_HOST=tcp://docker:2375). The fix differs:
  - Socket-mount: the socket volume is missing or the CI user lacks the docker
    group: mount the hostPath socket / add the group. Note honestly: socket
    mounting hands the job root-equivalent control of the host: flag it as a
    security trade-off, prefer rootless or Kaniko/BuildKit for image builds.
  - DinD service: sidecar not started, not privileged
    (`securityContext.privileged: true` required), TLS mismatch
    (DOCKER_TLS_CERTDIR), or the `docker:dind` service name not resolving:
    check service block in the CI config and daemon startup logs.
- Hangs/timeouts in DinD: /dev/shm or ephemeral-storage too small; overlayfs
  on top of overlay without the right storage driver. Verify with
  `df /dev/shm` inside the job and dind logs.
- Always include a preflight: socket exists → `docker ps` succeeds →
  `/dev/shm` usage <90%, as a copy-pasteable script step.

**OOMKilled during builds**
- Confirm: exit 137 + pod status `OOMKilled` (`kubectl describe pod`), or
  `dmesg | grep -i "out of memory"` on the runner host.
- Distinguish: container limit too low (raise limit, with measured peak from
  `kubectl top` / cgroup stats, not a guess) vs build genuinely unbounded
  (Gradle/webpack workers × heap: cap parallelism or per-worker heap:
  `-XX:MaxRAMPercentage=75`, `NODE_OPTIONS=--max-old-space-size`) vs node
  overcommit (QoS class: requests==limits for Guaranteed).
- Counter-intuitive but real: lowering build parallelism (`make -j2` instead
  of `-j8`) often fixes OOM cheaper than doubling memory; offer both with the
  time/cost trade-off.

**Pod eviction**
- Read the eviction reason verbatim from events: ResourceQuota exceeded
  (`kubectl describe resourcequota -n ns`: raise quota or right-size
  requests), NodeMemoryPressure / NodeDiskPressure (`kubectl describe node |
  grep -A5 Conditions`: image garbage, log growth, co-tenant pressure:
  clean + consider dedicated CI node pool), or preemption by higher
  PriorityClass (give CI jobs an explicit priority).
- Prevention tier: dedicated, tainted CI node pool with cluster-autoscaler:
  evictions from app-workload contention disappear by construction.

### 5.2 Security gate failures

Decision framework: legitimate vulnerability vs false positive vs policy
violation vs supply-chain issue. Walk the four questions IN ORDER and show
work:

1. **Known CVE?** ID, CVSS, affected version range vs our locked version.
2. **Is the package even used?** `grep -r "import.*pkg" src/` plus lockfile
   reachability (`npm ls pkg`): an unused transitive dep is lower urgency but
   NOT auto-dismissible (it still ships in the artifact).
3. **Is the vulnerable function called?** Search for the specific API named in
   the advisory. Not called = strong false-positive signal.
4. **Exploitable in our context?** SQLi advisory + we use parameterized
   queries everywhere = not exploitable here; document the reasoning, not just
   the verdict.

Verdicts and their mandatory artifacts:
- **Legitimate** → BLOCK: upgrade path (exact command), clean re-scan, test
  suite green, PR link.
- **False positive** → SUPPRESS: written justification, the scanner-specific
  suppression config (e.g., trivy `.trivyignore` with CVE + expiry date,
  Snyk ignore with reason), an expiry/review date (never permanent), and a
  named approver. A suppression without expiry and approver is a finding in
  itself.
- **Policy violation** → BLOCK + escalate: which policy, the compensating
  control, exception link, 30-day review date.
- **Supply chain** (typosquat / compromised / unsigned) → QUARANTINE the
  version + escalate: author check, download-spike check, repo legitimacy,
  signature verification. Never "fix" a suspected compromise by upgrading
  within the same package without provenance checks.

---

## Section 6: Safety & Confidence Loop

### 6.1 Confidence scoring formula (use for every root-cause claim)

```
CONFIDENCE = Base × Corroboration × Exclusivity

Base:           95% exact error string + visible stack trace
                80% pattern match + aligned contextual signals
                60% symptom consistent with multiple causes (inferred)
Corroboration:  1.0 metric evidence confirms (CPU/mem/latency)
                0.9 historical repeat (same failure ≥5 times)
                0.8 single corroborating log/metric
                0.6 inferred from related symptoms
                0.5 none (speculative)
Exclusivity:    1.0 alternatives ruled out by evidence
                0.8 one live alternative
                0.6 2–3 competing hypotheses
                0.5 ambiguous
```

Worked anchors: exact `context deadline exceeded` + latency spike + others
ruled out = 95×1.0×1.0 = 95% (act on it). Flaky with no clear cause =
70×0.6×0.6 = 25% (do NOT prescribe a fix: quarantine + investigate). Dependency
bump broke build, one alternative live = 90×0.9×0.8 = 65% (pin the version as
Tier 1, investigate before Tier 2). Round to whole percent; show the three
factors, not just the product.

### 6.2 Risk gating

Apply the Section 4.3 matrix and deployment gates to every recommendation.
Confidence and risk are independent axes: a 95%-confidence fix can still be
HIGH risk (e.g., correct RBAC change with cluster blast radius) and still
needs the human gate.

### 6.3 Escalate to a human when ANY of:

1. Confidence < 50% → ask for the missing context (which step, recent changes,
   a full log) instead of guessing.
2. Overall risk ≥ HIGH → human approval before applying anything.
3. Multiple plausible root causes → human prioritizes; fixing one may mask
   another.
4. Fix requires infrastructure change → infra team owns the blast radius.
5. Security-related failure → security team review; this skill classifies, it
   does not adjudicate security exceptions alone.
6. Flaky with no identified cause → quarantine + ticket beats an uncertain fix.
7. Customer-impacting → incident commander; this is now business judgment.

---

## Worked Examples (abbreviated to the decisive moves)

**1. Exit 137 in `npm run build`, GitHub Actions, started after adding a
package.** Classify: infra signature (137) but temporal marker is a commit →
check both. `docker stats` equivalent unavailable on hosted runner; reproduce
locally with `--memory=7g`: OOM at webpack. Cause: new package pulled
source-map generation into prod build; peak RSS 7.4 GB vs 7 GB runner.
Confidence: 80×1.0×0.8 = 72%. Tier 1: `NODE_OPTIONS=--max-old-space-size=6144`
+ disable prod source maps (rollback: revert env). Tier 2: code-split config.
Tier 3: larger runner class for build jobs with measured headroom. Risk: LOW.

**2. Integration test passes on retry ~30% of the time.** Route to Section 2,
not RCA. History: 41/50 pass (82% → flaky). Order-shuffle experiment: fails
only after `test_bulk_import` → DB pollution. Confidence: 80×0.9×1.0 = 72%.
Action: FIX, wrap suite in per-test transactions; validate 100 shuffled runs
>99%. Quarantine in the meantime since it gates merges (criticality HIGH).

**3. Pipeline red overnight, zero commits.** Section 3 first. Lockfile
untouched, but CI runs `npm install` not `npm ci` → resolved minor bump of a
transitive dep at 02:00 (registry timestamp matches first red run).
Confidence: 90×0.9×0.9 = 73%. Tier 1: pin the dep. Tier 2: switch CI to
`npm ci`. Tier 3: dependency-update bot so bumps arrive as reviewable PRs,
never ambient. Risk: LOW. Note the real root cause is the install command, not
the package.

---

## Delivery format

Every analysis ends with, in order: classification (origin, criticality,
blast radius, TTR), RCA or flake report with confidence shown as the
three-factor product, three-tier CAP, risk matrix + gate decision, and a
"watch after deploy" line (metric, threshold, duration). During active
incidents, suppress Tier 3 detail to a one-line pointer and keep the response
to the decisive evidence: no padding, no speculative side-quests. If evidence
is insufficient, say exactly that and list the ≤5 commands or artifacts that
will disambiguate, ordered by information value.
