# UI Automation Framework Blueprint

A generic, transferable architecture extracted from a mature production Python/Playwright/pytest
suite (~4,500 tests, 54 CI pipelines, two environments). Nothing here is app-specific — read it as
a design document for building the *next* framework, in any language, against any app.

Two layers:

- **Layer 1 — the test framework itself** (§1–§9)
- **Layer 2 — the AI-assisted QA workflow around it** (§10)

Each section states the **pattern**, the **reason** it exists, and the **generic rule** to carry over.
§11 lists what *not* to copy.

---

## 1. Directory contract and dependency direction

```
Tests/            test files only — no locators, no selectors, no driver calls
Pages/            page objects — one per screen/tab, thin, intent-named
Data/Locators/    selectors as plain module-level tuples
Data/TestData/    constants, payloads, expected values
Fixtures/         auth, session, test-data lifecycle, failure artefacts
WebControls/      the driver-wrapper layer (the keystone — see §4)
Utilities/        auth codes, API client, waits, diagnostics, generic helpers
Configuration/    config.yml (tracked) + credentials file (gitignored)
Scripts/          CI-side tooling: result recovery, report enrichment, data cleanup
Reports/          Latest/ + Archive/ + raw result JSON
```

**The invariant that matters more than the layout** is a strict one-way dependency:

```
Tests ──▶ Pages ──▶ WebControls ──▶ driver
  │         │
  └─────────┴──▶ Data, Utilities
```

- Tests never touch a selector or the driver.
- Pages never assert.
- `Pages` must not import `Pages` at module level. When one page object needs another (shared
  controls living on a sibling tab), use a type-checking-only import plus a local runtime import:

```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from Pages.fixtureTabPage import FixtureTabPage   # IDE navigation only

class SummaryTabPage:
    def frPage(self) -> "FixtureTabPage":
        from Pages.fixtureTabPage import FixtureTabPage   # runtime, inside the method
        return FixtureTabPage(self.element.page)
```

**Generic rule:** declare the dependency direction on day one and enforce it in review. Every
framework that rots does so by letting selectors leak into tests and assertions leak into pages.

---

## 2. One test = one file = one requirement ID

```
Tests/FunctionalTest/BuildReadiness/test_54682734.py
    └── def test_54682734(cardinal, systemUsers):
        └── description = "54682734", "Verify Part Number filter performance…", "owner-id"
```

The external test-case id appears three times: filename, function name, and a `description` tuple
carrying `(id, title, owner)` that is rendered into the report.

**Why**

- Perfectly greppable — `test_<id>.py` is the join key between automation, the test-management
  system, and any defect that references the case.
- Pairs with `--dist loadfile` (§7): one file is one atomic parallel unit, so there is no
  intra-file ordering hazard and no shared mutable state between tests on a worker.
- Per-test git blame and per-test ownership, for free.

**Cost:** ~4,500 files. Directory listings are useless; you navigate by grep, not by eye.

**Generic rule:** adopt this *only* when your tests are externally catalogued (a test-management
system, a requirements tracker). If tests are not externally identified, group by behaviour instead
— the file-per-test tax buys you nothing.

---

## 3. Locators: plain tuples, isolated, append-only

```python
# Data/Locators/buildReadinessLocators.py
reportPageTitle      = "xpath", "//h1[text()='Build Readiness Reports']"
reportCard           = "xpath", "//div[@data-testid='build-readiness-report-card']"
lobOptionInDropdown  = "xpath", "(//div[contains(@class,'dropdown')]//span)[{}]"   # parametrized
treeItemCheckbox     = "xpath", "//span[text()='{}']/ancestor::div[@role='treeitem']//input"
```

- A locator is a **2-tuple `(strategy, selector)`** — not a class attribute, not an enum, not a
  `By.XPATH` object. Plain data, so any layer can normalize it (§4).
- Parametrized selectors use `{}` and are `.format()`-ed at the call site.
- **Append-only, with a comment naming the case that introduced each one.** New locators go at the
  end; existing ones are never reordered. This keeps diffs legible across hundreds of UI changes.

**Why it's worth a whole directory:** selectors are the single highest-churn artefact in UI
automation. Isolating them means a UI release touches one file per screen — never a test, never a
page method, never logic.

**Generic rule:** locators are *data*, stored separately from the code that uses them, in whatever
shape your driver-wrapper can normalize.

---

## 4. The driver-wrapper layer — the keystone

A single module (~1,800 lines here) sits between page objects and the driver. Page objects call
only this. It is where all the hard-won reliability lives, and it is the most transferable idea in
the whole framework.

It owns six responsibilities:

### 4.1 Locator normalization
Accepts the `(strategy, selector)` tuples from §3 and converts them to native driver selectors.
Because page objects never build a native selector, **swapping the driver is a change to one file.**

### 4.2 Timeout budgeting
```python
ACTION_BUDGET_S = 120

def actionTimeoutMs(self):
    # per-attempt wait × retry attempts can never exceed the action budget
    return int(min(configuredWait, ACTION_BUDGET_S / attempts)) * 1000
```
Retries inside a wrapper silently multiply wall-clock. Cap *wait × attempts*, not just the wait.

### 4.3 Typed failure classification — the most valuable single idea here

```python
class InfraError(Exception):
    """Raised when the PAGE shows an environmental failure: 5xx, DB exception,
    unexpected-error alert. The retry flag matches this class NAME."""

class ResourceDeletedError(InfraError):
    """Subclasses InfraError so `except InfraError` still catches it — but the
    reported class name differs, so the name-matched retry does NOT fire.
    A deleted resource cannot come back mid-run; retrying it is guaranteed waste."""

def detectInfraError(page): ...
```

Paired with the runner config:

```ini
--only-rerun=InfraError    # retry ONLY environmental failures
--reruns=1
```

**What this buys you:** a genuine locator bug or a real assertion failure hard-fails *immediately*,
with no wasted reruns, while a backend wobble gets its retry. Blanket `--reruns=2` does the
opposite — it triples the cost of every real bug and hides nothing but your signal.

Note the deliberate subtlety: a *non-recoverable subclass of a recoverable class*. `except
InfraError` still catches it for cleanup purposes, but the name-based retry matcher skips it.

**Generic rule: flaky-retry must be typed, never blanket.** Classify the failure at the point you
can still see the page, then retry only the class that can actually succeed on a second attempt.

### 4.4 Auto-diagnosis on failure
On a failed interaction, dump the page's available test ids and classify *why*:

- selector did not resolve at all → wrong/stale selector
- resolved but not actionable → timing, overlay, animation
- resolved to multiple elements → ambiguous selector

This turns "element not found" into an actionable sentence and is the difference between a 2-minute
and a 2-hour triage.

### 4.5 Intent-level compound actions
`clickUntilVisible(target, expectedResult)`, `isDisappeared(locator, maximumTime)`,
`selectorResolvesVisible(...)`. Page objects express *intent*; the wrapper owns the retry loop.
Without this, every page object grows its own ad-hoc `while` loop and they all differ.

### 4.6 A compatibility shim for migration
A `CompatElement` class exposing the *old* driver's API (`.text`, `.click()`, `.send_keys()`,
`.is_displayed()`, `find_elements(by, value)`) on top of the *new* driver's locators.

This is how a large Selenium suite ports to Playwright (or any A→B driver migration) **without
rewriting thousands of page-object methods**: implement the shim, point the wrapper at the new
driver, and migrate page objects opportunistically rather than in a big bang.

**Generic rule:** if you ever expect to change drivers, the wrapper layer is not optional — it is
the thing that makes the change affordable.

---

## 5. Configuration: one declarative profile table

```yaml
envQA:            # a named run profile
  module: Tests/FunctionalTest        # a path…
  url: https://app-qa.example.com
  browser: chromium
  maximumWait: 30
  parallelTestExecution: 6
  retryCount: 0
  headed: false
smokeQA:
  module: -m smoke                    # …or a marker expression…
  parallelTestExecution: 6
  retryCount: 1
standAlone:
  module: ''                          # …or empty, meaning "whatever was passed on the CLI"
Performance:
  parallelTestExecution: 1            # must stay serial
```

The crucial property: **every entry point reads this same file** — the local runner, the CI pytest
step, and the CI browser-launch probe all resolve `module`, `browser`, and worker count from it.

```bash
# CI resolves the same values the local runner does — no duplicated env knowledge
MODULE=$(python3 -c "import yaml;print(yaml.safe_load(open('Configuration/config.yml'))['$ENV']['module'])")
```

`module` being polymorphic (path | marker expression | empty) is what lets one mechanism express
"the whole functional tree", "everything tagged smoke", and "whatever I type locally".

**Generic rules**
- One profile table, consumed everywhere. The moment CI hardcodes a URL or a worker count that
  also lives in config, they drift and you debug the difference instead of the product.
- **Config is tracked; credentials are not.** The credentials file is gitignored and injected by CI
  from a secret store, with explicit fail-fast:

```bash
if [ -f "/secrets/SYSTEM_USERS_YML" ]; then
  cp /secrets/SYSTEM_USERS_YML Configuration/systemUsers.yml
else
  echo "ERROR: secret not set in project settings"; exit 1
fi
```

- Keep a profile override parameter (`MODULE_OVERRIDE`) so one pipeline can point at a subdirectory
  while still using a profile's host/browser/wait settings. This is what lets you split a suite by
  module without writing a new profile per slice.

---

## 6. The test and fixture contract

### 6.1 What a test looks like

```python
description = "54682734", "Verify Part Number filter performance with large datasets", "owner"

@allure.description_html(Utilities.getHTMLDescription(description))
@pytest.mark.parametrize("systemUsers", ['departmentEP'])          # ← the ROLE
def test_54682734(cardinal, systemUsers):

    # Prerequisite: user is logged in as Department EPM and a report with a large dataset exists
    app = cardinal(systemUsers)                                    # ← logged-in container
    homePage   = app.homePage()
    reportPage = app.buildReadinessReportPage()

    # 1. Navigate to the home page
    expect(homePage.isHomePageTitleDisplayed(), "Home Page title is not displayed.")

    # 2. Open the Build Readiness tab
    reportPage.clickOnBuildReadinessTab()

    # 3. Time the filter popup
    start = time.time()
    reportPage.clickOnPartNumberFilterIcon()
    shown = reportPage.isPartNumberFilterPopupDisplayed()
    elapsed = time.time() - start

    expect(shown, "Part Number filter popup is not displayed with its options.")
    expect(elapsed < maxSeconds,
           "Filter popup took {:.1f}s, exceeding the {}s budget.".format(elapsed, maxSeconds))

    assert_expectations()
```

Five patterns in that one file:

**(a) Soft assertions by default.** `expect(...)` throughout, one `assert_expectations()` at the
end. A UI test reports *every* broken expectation in one run instead of dying on the first. Each
message is a full sentence stating what should have been true — the report is then readable by
someone who has never seen the code.

**(b) Numbered step comments mirroring the manual test case.** `# 1.`, `# 2.`, `# 3.` and a
`# Prerequisite:` block. The test file doubles as the manual script, so anyone can diff the
automation against the written case without reading Python.

**(c) Role parametrization.** `@pytest.mark.parametrize("systemUsers", ['departmentEP'])` — the
test declares which *role* it runs as. The role lands in the pytest node id, making RBAC coverage
visible in the report and expandable by adding one list element.

**(d) A fixture that is a page-object factory.**

```python
@pytest.fixture
def cardinal(request, playwrightDriver) -> Callable[..., "App"]:
    # returns a callable; calling it with a role logs in and hands back a container
```

`app.homePage()` lazily constructs each page object already bound to the live page. Tests get one
import instead of six, and construction logic lives in exactly one place.

**(e) Timing assertions in functional tests.** Capturing `time.time()` around an interaction and
asserting against a budget from test data turns a functional test into a cheap performance guard.
Keep the budget in `Data/TestData`, never inline.

### 6.2 Test-data lifecycle through the API, not the UI

```
Scripts/testDataHelpers/
    apiSession.py            authenticated session against the product's own API
    cleanup/
        forecastReports.py
        demandPlans.py
        approvals.py
```

Fixtures register what a test created; teardown deletes it via the API. Cleanup helpers are written
to be **idempotent and leak-aware**:

```python
def deleteForecastReport(session, reportUrl):
    match = re.search(r'/forecast/(\d+)', reportUrl or '')
    if not match:
        return False                      # test registered a list page: nothing was created
    version = session.request('GET', '/forecasts/versions/' + match.group(1))
    if version.status_code == 404:
        return True                       # a test that ends in "delete" already removed it —
                                          # that is NOT a leak, do not report it as one
    assert version.ok, f'{reportUrl}: lookup failed ({version.status_code})'
    ...
```

The distinction between "nothing to clean" (`False`), "already gone, fine" (`True`), and "cleanup
genuinely failed" (`assert`) is what stops teardown noise from drowning real failures.

**Generic rule:** create and destroy test data through the API. Drive the UI only for the thing
actually under test. UI-based setup is the largest single source of flake and runtime in most
suites.

### 6.3 Failure artefacts need a hook, not good intentions

```python
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Stash each phase's report as rep_setup / rep_call / rep_teardown."""
    outcome = yield
    setattr(item, 'rep_' + outcome.get_result().when, outcome.get_result())
```

The teardown fixture reads `item.rep_setup` / `item.rep_call` to decide whether to capture a
screenshot and stop the trace. **Without this hook those attributes never exist, so failure
screenshots silently never fire** — and nobody notices until they need one.

**Generic rule:** verify your failure-artefact path by deliberately failing a test. "We capture
screenshots on failure" is the single most commonly believed-and-false statement in test
frameworks.

---

## 7. Execution: parallelism, timeouts, and worker safety

### 7.1 Three timeout layers, all environment-overridable

```python
class TimeoutConfig:
    ACTION_TIMEOUT     = int(os.environ.get('ACTION_TIMEOUT', '120'))
    NAVIGATION_TIMEOUT = int(os.environ.get('NAVIGATION_TIMEOUT', '120'))
    TEST_TIMEOUT       = int(os.environ.get('TEST_TIMEOUT', '900'))     # wall-clock backstop

def pytest_collection_modifyitems(config, items):
    for item in items:
        item.add_marker(pytest.mark.timeout(TimeoutConfig.TEST_TIMEOUT))
```

Applying the wall-clock timeout as a *marker at collection time* keeps the value env-driven while
the mechanism stays declared in config.

### 7.2 The timeout method is not a detail

```ini
timeout_method = signal
```

`signal` raises in the test's own thread and ends in a normal test failure with a real traceback;
the parallel worker keeps its queue. The `thread` method reaches `os._exit(1)` — **killing the
whole worker and every test still queued on it.** One hung test silently becomes fifty missing
results.

**Generic rule:** whatever your runner, find out what its timeout mechanism does to the *worker*,
not just to the test.

### 7.3 Parallelism is bounded by the system under test, not the runner

Measured: the CI runner allowed 12 workers. The suite runs at **6**, because the shared app
instance and the auth service drop connections when many workers authenticate in the same instant —
two builds lost 17 and 131 tests to exactly that.

**Generic rule:** find the real bottleneck (shared DB, auth service, rate limit, app instance).
Past that point, extra workers convert throughput into flake, and flake costs more than the time
you saved.

### 7.4 Distribution strategy pairs with file layout
`--dist loadfile` + one-test-per-file (§2) = each file is an atomic unit on one worker. Choose the
distribution mode that matches how your tests share state.

### 7.5 Controller-vs-worker guards on every session-scoped side effect

```python
def pytest_configure(config):
    if hasattr(config, 'workerinput'):     # worker, not controller
        return
    # wipe stale result JSON exactly once — on a worker this would race the others

def pytest_sessionfinish(session):
    if hasattr(session.config, 'workerinput'):
        return
    # the controller's hook runs after every worker has finished
```

**Generic rule:** in any parallel runner, every "once per run" action needs an explicit
controller-only guard. Without it, N workers each wipe the results directory.

### 7.6 The local runner stays thin

A ~90-line entry point that: validates the requested profile names against the YAML → creates
report directories → spawns one process per profile (so several suites run concurrently) → zips the
results into a timestamped archive.

```python
subprocess.run(['pytest', *shlex.split(module), f'-n={workers}', '--dist', 'loadfile',
                '--max-worker-restart=100', '--reruns', str(retryCount), ...])
```

Note `shlex.split(module)`, not `[module]` — because `module` may be a path, a marker expression
(`-m smoke`), or empty.

**Generic rule:** the custom runner resolves config into runner arguments and nothing else. Every
behaviour belongs in the test runner itself, where `-k`, `-m`, `--lf` and the IDE still work.

---

## 8. Platform portability: fix trust, don't disable it

Two examples worth copying wholesale.

**(a) Internal CA.** Auth died in test *setup* with `CERTIFICATE_VERIFY_FAILED` because the
corporate root CA is not in the Python cert bundle. The documented workaround was
`SSL_VERIFY=false` on every command, which (i) turns verification off rather than fixing trust, and
(ii) is silently missed by IDE runners where nobody sets shell env vars.

The fix builds a real bundle instead:

```python
def trustInternalCa():
    if os.environ.get('REQUESTS_CA_BUNDLE') or os.environ.get('SSL_CERT_FILE'):
        return                      # an explicit bundle always wins (CI sets its own)
    bundle = buildBundle()          # certifi PEMs + org roots read from the OS keychain
    if bundle:
        os.environ['REQUESTS_CA_BUNDLE'] = bundle
        os.environ['SSL_CERT_FILE'] = bundle
```

with three details that make it robust:
- match on the **decoded** certificate subject, not the PEM text (a base64 body never contains the CN)
- **write-then-rename** (`os.replace`) so parallel workers can never read a half-written bundle
- cache it in a gitignored dir and rebuild when missing, so `pip install -U certifi` can't revert it
- no-op on other platforms and when CI has already set a bundle

**(b) Locked-down hosts.** The browser's process-singleton socket can't bind in the system temp
dir, so the framework sets a project-local `TMPDIR` before launch.

**Generic rule:** environment workarounds belong in `conftest`/bootstrap, not in a README step.
Anything a human must remember to export will be forgotten — most often by the IDE runner, which is
how most people actually run tests.

---

## 9. CI design — the part most frameworks get wrong

### 9.1 One pipeline per suite, with shared steps defined once

Each suite gets its own build history and pass-rate trend, instead of one job whose graph mixes a
40-minute smoke run with a 200-minute regression. The shared steps/machine/report blocks are
declared **once** and aliased (YAML anchors) into every pipeline — copying 160 lines per pipeline
guarantees they drift at the first merge.

> **Caveat learned the hard way:** an alias (`*steps`) whose anchor (`&steps`) is removed by a merge
> makes the entire CI file unparseable, and every pipeline then dies in 0s with no log. **Validate
> the CI config in a pre-commit hook and as a CI step.** Anchors trade duplication for a
> single point of total failure.

Only the fast gate builds on push. Nightly timers are staggered so suites don't contend for the
same app instance.

### 9.2 The eight-step pipeline, generically

```
[0] CHECKOUT   — honour an overridable branch/sha parameter
[1] SECRETS    — restore gitignored credentials; fail fast and loudly if absent
[2] DEPS       — full log to a file; tail it ONLY on failure
[3] BROWSER    — install, then PROBE-LAUNCH the engine this profile needs
[4] RUN        — under a wall-clock budget; capture the exit code, don't propagate yet
[5] RECOVER    — rebuild results if the runner was killed before writing them
[6] REPORT     — generate, with artefact-size pruning
[7] ENRICH     — make results readable *in* the CI UI
[8] PROPAGATE  — now exit with the real test exit code
```

Each of the non-obvious ones, and why:

**[3] Probe-launch the engine.** One pipeline lost all 159 tests to identical
`BrowserType.launch` failures (a missing system library). A five-line probe that launches the
engine, loads a trivial page, and exits now fails the build in step 3 with a precise error —
including a note on which alternatives are viable. Probe *only* the engine this profile needs, so a
broken engine never blocks an unrelated suite.

**[4] Budget, then capture-don't-propagate.**
```bash
set +e
timeout -k 60 ${PYTEST_BUDGET} python3 -m pytest ${MODULE} -n=${WORKERS} ...
echo $? > .pytest_rc           # capture
set -e
RC=$(cat .pytest_rc)
if [ "$RC" = "124" ] || [ "$RC" = "137" ]; then
  echo "!! hit the ${PYTEST_BUDGET}s budget and was killed — results recovered in [5]"
fi
```
The test budget must sit **under the platform's job cap with room to spare**, so the run is cut off
while there is still time for steps [5]–[7]. One build finished hundreds of tests and published
`totalTests: 0` because the job was killed before reporting ran. Propagating the exit code
immediately would skip reporting entirely — hence capture in [4], propagate in [8].

**[5] Result recovery is mandatory, not an edge case.** Allure-style reporters write one JSON per
test *the moment it finishes*; JUnit XML is written only at session end. So a killed run publishes
zero tests with every finished result sitting on disk — one build lost 117 of 141 completed tests
to exactly this. A converter rebuilds JUnit from the per-test JSON:

```bash
if [ -s junit-results/test-results.xml ]; then
  echo "runner wrote its own junit — no recovery needed"
else
  python3 Scripts/allure_to_junit.py Reports/ResultJson junit-results/recovered.xml || true
fi
```

For any suite that is *cut off by design*, this is the normal path.

**[6] Prune oversized artefacts before publishing.** Single-file HTML reports *embed* attachments.
A 476-failure run embedded hundreds of multi-MB browser traces into a 3.0 GiB report against a
1.0 GiB publish limit — and it failed **silently**: no error, just a missing download. Attachments
are named deterministically by type (`<uuid>-attachment.zip` for traces, `.png` for screenshots),
so traces can be pruned by extension above a size threshold while screenshots survive:

```bash
SRC_MB=$(du -sm Reports/ResultJson | cut -f1)
if [ "${SRC_MB:-0}" -gt 300 ]; then
  find Reports/ResultJson -name '*-attachment.zip' -delete
  echo "Pruned trace attachments (source ${SRC_MB}MiB over 300MiB). Screenshots untouched.
        For a full trace replay, re-run the failing test(s)."
fi
```
Small runs keep full trace replay; large runs still publish. And the log says exactly what was
dropped — a silent cap reads as "we covered everything" when you didn't.

**[7] Make results readable in the CI UI.** A runner names rows after functions, so the CI test tab
shows `test_54275938[chromium-departmentEP]` — an opaque id with no indication of what broke. Two
small scripts fix this:
- rewrite every JUnit row as `id · human title`, read from each test's `description` tuple
- print a human summary as the **last** thing in the build log, so nobody scrolls 4,000 lines

**Generic rules from §9**
- Budget < platform cap, with headroom for reporting.
- Capture exit codes, run reporting, propagate last.
- Every artefact needs a recovery path and a size ceiling.
- CI should display *intent*, not function names.
- Fail fast on environment problems, in a dedicated step, with the remedy in the message.

### 9.3 Close the loop back to the test-management system

A script reads which cases a CI run passed and marks them Pass in the external tracker. It replaces
"open the build, read the failures, work out which cases therefore passed, click 97 checkboxes" —
repeated after every run.

**Generic rule:** automation that doesn't write results back to wherever humans track coverage is
only half-built.

### 9.4 Comment the incident, not the mechanism

The highest-leverage cultural practice in this codebase. Every non-obvious setting carries the
build number, the symptom, and the rejected alternative:

```ini
# 1 not 2: build #15 re-executed 167 infra failures up to 3x, and an auth wobble that lasts a
# whole run does not clear on the third try — it just triples the wall clock.
--reruns=1
```

```yaml
# DO NOT RE-ADD pipelineChain HERE — IT CANNOT WORK (proven 2026-09-05). The JUnit publisher is a
# separate post-build step that re-derives the build result AFTER the script's exit code, so any
# failing test downgrades the build and the chain's "successful run" gate never fires.
# [...] Sequencing is done in a build step instead, where the publisher cannot reach it.
```

Anyone reading it knows not only *what* the setting is, but what happens if they change it back —
which is the question people actually have. **Document the decision and its evidence at the point
of the decision**, including the alternatives you rejected and why.

---

## 10. Layer 2 — the AI-assisted QA workflow around the framework

Reusable independently of the test framework.

**10.1 A skill is one named workflow.** ~40–220 lines: a trigger-phrase description, numbered
steps, hard constraints at the end. Small enough to read in one sitting, specific enough to be
followed exactly.

**10.2 Keep trigger phrases mutually exclusive.** Observed failure: five skills all firing on
"RCA" / "smart delta" / "triage" / "QA analysis" — selection becomes a coin flip. Either make the
triggers disjoint, or make one skill the explicit front door and have the others declare themselves
as sub-playbooks.

**10.3 Prefer a composer over parallel playbooks.** One front-door workflow that (a) classifies the
input, (b) routes to the right playbook, and (c) **guarantees the one artefact humans actually
want**. The example here exists specifically because that artefact was skipped twice despite being
the documented output of the underlying playbook — and the skill says so, naming both incidents.
**Encode corrective context inside the workflow**, not in someone's memory.

**10.4 Durable memory, split by type, one fact per file.**
`project` (state not derivable from code) · `feedback` (guidance, each with *why* and *how to
apply*) · `reference` (links) · `user`. Plus one index file loaded every session. Cross-link
entries. The discipline that makes it work: convert relative dates to absolute, and never store
what the repo already records.

**10.5 Caches are a first-class artefact — and need an owner.** A code-paths cache and a
test-case index cache let recurring analysis *reuse* instead of re-exploring, which is the
difference between a 2-minute answer and a 20-minute one. But a cache with no refresh cadence and
no owner silently becomes a confidently-wrong answer. State the cadence *in* the cache file, and
treat staleness as a bug.

**10.6 Declare write boundaries explicitly.** "These repos are read-only, clone for investigation
only; all writes go to these four places." Ambiguity here is how an analysis session edits
production source.

**10.7 Tool-namespace drift is a real failure mode.** Integration tool names change when a server
is repackaged. Pin by *capability* (look the tool up by bare name), not by prefix — and when you
discover drift, propagate the fix to every workflow that hardcoded the old prefix, not just the one
that broke. Observed here: two agents and two skills left pointing at a namespace that no longer
serves those tools, so they could not read anything at all.

**10.8 Choose agent *or* in-session, and say which.** Shipping both a command that says "do not
spawn subagents" and an agent definition for the same job guarantees two different behaviours for
the same request.

---

## 11. What NOT to copy

Honest anti-patterns in the source framework. Avoid these from day one — they are far cheaper to
prevent than to unwind.

| Anti-pattern | Observed | Do instead |
|---|---|---|
| **Page-object monoliths** | 12 page objects hold 27.6k lines; the largest is 8,910 lines | Split per screen *and per tab/panel* from the start. Enforce a soft line ceiling in review. |
| **Hand-maintained mirrored test tree per environment** | 2,161 duplicated relative paths across two env trees; 204 had already drifted | Parametrize the environment. The profile table (§5) already holds the per-env URL — one tree, two profiles. |
| **Duplicated test-data modules per environment** | `globalTestData.py` + `globalTestDataUAT.py`, and a shared page object importing `globalTestDataUAT as globalTestData` — hardcoding one environment into a shared layer | Environment-keyed lookup in one module, resolved from the active profile. |
| **Positional / framework-class XPath** | `(//header//span)[14]`, `//div[@class='ant-card-body']` | `data-testid` first. Positional selectors break on every layout change and the failure never names the cause. |
| **CI config with no parse validation** | A merge dropped a YAML anchor whose aliases remained; the file became unparseable and all 54 pipelines died in 0s with no log | Lint/parse the CI config in a pre-commit hook *and* as a CI step. |
| **Vestigial pipelines** | Two pipelines referenced a profile absent from config, a default outside its own dropdown's choices, and a selection file that no longer exists — permanently unrunnable | Delete capability when you remove its implementation. A pipeline that cannot succeed is worse than no pipeline: it reads as coverage. |
| **Blanket retries** | (avoided here, via §4.3 — but the default everywhere) | Type your failures and retry only the recoverable class. |

---

## 12. Build order for a new framework

1. **Directory contract + dependency direction** (§1). Write it in `CONTRIBUTING.md` first.
2. **Driver-wrapper layer** (§4) — normalization, timeout budgeting, typed `InfraError`,
   auto-diagnosis. Everything else leans on this; build it before the second page object.
3. **Profile table** (§5) with one profile, plus the gitignored-credentials split.
4. **Fixtures** (§6) — role-parametrized login, the page-object factory, the
   `makereport` hook, and *verify a failure screenshot actually lands*.
5. **One page object + one locator module + one test**, in the final conventions. Review it hard;
   it will be copied several thousand times.
6. **Thin local runner** (§7.6) + parallel-safety guards (§7.5) + timeout layers (§7.1–7.2).
7. **API test-data helpers** (§6.2) before the suite grows past ~50 tests. Retrofitting UI-based
   setup is the most expensive migration in this whole list.
8. **CI: the fast gate only** — all eight steps (§9.2), including recovery and enrichment, even
   when the suite is small enough not to need them yet.
9. **Split into per-suite pipelines** once the gate exceeds ~40 minutes, with anchors *and* a CI
   config parse check.
10. **Close the loop** to the test-management system (§9.3), then add the AI workflow layer (§10).

Throughout: comment the incident, not the mechanism (§9.4). It is the practice that makes every
other decision in this document survive its author.
