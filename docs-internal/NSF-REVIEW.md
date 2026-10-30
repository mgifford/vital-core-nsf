# vital-core-nsf: CMS-reference review and go-live checklist

Reviewed 2026-10-23 against `mgifford/vital-core-nsf` at `c3d3486` ("updating
for nsf") and `mgifford/vital-core` at `f5bac30`.

## Status at 2026-10-30

**Live.** Since 2026-10-25, every scheduled scan and report has succeeded.
`vital-core-nsf-data` has scan data for all five domains, including 2026-W44.
Reports publish daily to <https://mgifford.github.io/vital-core-nsf/>, and
weekly summaries go to the "Weekly scan reports" issue (#7).

**Done:**

| What | Where |
|---|---|
| `targets.yml` parse error; tests that assumed CMS targets | [#1](https://github.com/mgifford/vital-core-nsf/pull/1) |
| Scheduled scans never started a domain with no crawl state | [#2](https://github.com/mgifford/vital-core-nsf/pull/2) |
| A failed data push throws away the scan (recorded upstream) | [#3](https://github.com/mgifford/vital-core-nsf/pull/3) |
| Push-access check runs before any scan or build | [#4](https://github.com/mgifford/vital-core-nsf/pull/4) |
| Deploy key and `VITAL_DATA_DEPLOY_KEY` secret set up | repo settings |
| Actions moved to Node 24 releases; `.nvmrc` holds the Node version; Dependabot for actions | [#6](https://github.com/mgifford/vital-core-nsf/pull/6) |
| Legal baseline (Section 508, WCAG 2.0 AA) vs WCAG 2.2 AA target; separate AAA and best-practice list; 45 Alfa rule names fixed | [#8](https://github.com/mgifford/vital-core-nsf/pull/8) |
| All of the above offered upstream, with setup docs | [vital-core#276](https://github.com/mgifford/vital-core/pull/276), merged |
| History joined to vital-core (`f5bac30` recorded as the shared base), then upstream merged: PDF checks, blocked-site notices, canonical URLs, spelling locale, diagnose workflow | [#10](https://github.com/mgifford/vital-core-nsf/pull/10) |
| PDF checks on for every NSF site (`pdf_audit` in `defaults`) | [#10](https://github.com/mgifford/vital-core-nsf/pull/10) |
| Delete-on-merge turned on for this repo | repo settings |

**Pulling in upstream from now on** is an ordinary
`git fetch upstream main && git merge upstream/main`, on a branch, opened as a
PR and merged with **"Create a merge commit"**. A squash or rebase merge
drops the link to upstream, and the next merge then conflicts. Upstream fixes
are made in vital-core first (CLAUDE.md, "Fleets made from this template").

**PDF checks: what to expect.** At 2026-W44 the sites link to 6,481 PDFs on
NSF's own domain:

| Site | Own-domain PDFs | First pass at 25 a day |
|---|---|---|
| ncses.nsf.gov | 6,194 | about 8 months |
| www.nsf.gov | 273 | about 11 days |
| iucrc.nsf.gov | 7 | 1 day |
| seedfund.nsf.gov | 7 | 1 day |
| par.nsf.gov | 0 | — |

Newest files go first. 396 NCSES PDFs are listed under both `ncses.nsf.gov`
and `www.ncses.nsf.gov`, so until that's fixed upstream they're checked
twice (UPSTREAM-SUGGESTIONS §13). Raising NCSES's `pdf_audit.per_run`
would shorten the first pass but put more load on NCSES each day, so the
upstream default stays for now. The first daily run is at 22:41 UTC. To see
results sooner, run Actions → pdf-audit by hand for `iucrc.nsf.gov` or
`seedfund.nsf.gov`, which have only a few files each.

**Still open:**

- **NSF wording in the docs.** The README introduction, `WAF-ALLOWLIST.md`,
  `.well-known/*.json`, the MCP/API examples, `config/profiles/` (plus an
  `nsf.yml` for dashboard branding), the spelling allowlist and the
  `acr.js` feedback URL. See "CMS references" below. Upstream's planned
  "instance identity" config block (UPSTREAM-SUGGESTIONS §2) would make most
  of these unnecessary.
- **NSF side:** tell NSF's web and Section 508 teams (step 14), and check
  whether the sites use USWDS (step 15). The first scans show no firewall
  blocking.
- **Translations:** the new report text needs French, Japanese and Dutch.
- **Dependabot:** review the security alerts it raises for this repo.
- **Branch clean-up:** delete `docs/session-learnings`. It merged as #9,
  before delete-on-merge was on.

## Lessons learned

1. **Check the config loads before pushing it.** A duplicate `targets:` key
   stopped every workflow, and the tests only reported it indirectly. A
   `check:config` step would name the line straight away.
2. **A green run isn't proof that anything happened.** A manual scan "succeeded"
   while skipping all five domains, and reports "succeeded" before any data
   existed. Check outcomes instead: commits in the data repo, pages on the
   site, the weekly issue.
3. **Fail early, before touching target sites.** The missing deploy key only
   showed up at the final push, after each run had crawled NSF. A preflight
   check now stops the run in seconds and names the cause.
4. **Do irreversible steps last.** Crawl progress was saved before the data
   commit, so failed pushes threw away about 970 pages of scans while still
   marking them done for the week.
5. **Report errors where people and tools can read them.** Step logs couldn't
   be fetched through the API, but annotations could. Putting git's message
   in an annotation made the deploy-key problem diagnosable in one run.
6. **Don't trust hand-maintained rule metadata.** 45 Alfa rule names
   described other rules, and a "label exists" check never noticed. Derive
   names from the installed package, and check them against it.
7. **A finding isn't automatically a requirement.** Ranking only by pages
   affected put AAA and best practices above legal requirements, and pushed
   the legally required findings off the list. Rank and label by legal
   baseline, then target, then AAA, then best practice.
8. **Keep the fork connected to upstream.** Copying files into a new repo
   created an unrelated history, so upstream fixes can't simply be merged.
   New fleets should be cloned and pushed (README "Setup", step 1).
9. **Update the setup docs when the architecture changes.** The README
   predated the separate data repo, and following it reproduced every NSF
   failure. A `doctor` setup-check workflow would catch the rest.
10. **Small CI traps:**
    - the `scan-global` concurrency group cancels queued runs, so start one
      scan at a time
    - `.gitignore`'s `data/` also matches `src/data/`, so `git add` refuses
      the tracked `src/data/*.json`; use `git add -u`, or anchor the pattern
      as `/data/`
    - pin action versions and let Dependabot move them, rather than waiting
      for deprecation warnings

## Original review (2026-10-23)

- **This repo is a copy of `vital-core` with one changed file.** A recursive diff
  against upstream `main` finds only `config/targets.yml` changed. Every doc,
  profile, test and code string is still the CMS version. No document except
  `targets.yml` mentions NSF.
- **Blocker: `config/targets.yml` does not parse.** It has two top-level
  `targets:` keys, at lines 175 and 262. The `yaml` parser stops with
  `Map keys must be unique at line 262`, so `loadConfig()` throws. The scan
  matrix, `scripts/data-repo.js` and the report build all fail on their first
  step. Nothing could run until this was fixed. Fixed in
  [PR #1](https://github.com/mgifford/vital-core-nsf/pull/1).
- **Go-live gotcha: scheduled scans never started a new domain.** A domain with
  no saved crawl state was skipped by the scheduled matrix as "frontier empty".
  Fixed in [PR #2](https://github.com/mgifford/vital-core-nsf/pull/2) (see
  step 11).
- **The history is not shared with upstream.** This repo has two commits.
  "Initial commit" (`e5b744c`) is a full copy of upstream's files, as a new root
  commit. "updating for nsf" (`c3d3486`) changes only `config/targets.yml`.
  Upstream changes can't be merged cleanly until the two histories are joined
  (see step 3).

## CI status

The `tests` workflow
[run 40018952284](https://github.com/mgifford/vital-core-nsf/actions/runs/40018952284)
on `main` (`c3d3486`) failed at its "Unit tests" step. The previous run, on
"Initial commit" (`e5b744c`), passed. So the `targets.yml` change is the cause.

The raw CI log couldn't be fetched from the review session. Running the unit
tests locally at `c3d3486` gives these 9 failures. Each throws
`Map keys must be unique at line 262, column 1` from `loadConfig()`:

| Test | Location |
|---|---|
| config: loadConfig exposes languages on every target | `tests/unit/config.test.js:39` |
| config: normalizes supported design_system values | `tests/unit/config.test.js:48` |
| config: webmcpEnabled resolves to a strict boolean for every real target | `tests/unit/config.test.js:98` |
| config: loadConfig surfaces reportBaseUrl with no trailing slash | `tests/unit/config.test.js:108` |
| config: findingPolicy is available on every target | `tests/unit/config.test.js:114` |
| config: the real targets.yml resolves to a single CI data repo | `tests/unit/config.test.js:187` |
| state-release: every configured hf_only target is refused | `tests/unit/state-release.test.js:16` |
| url_exclude_patterns: config loads with url_exclude_patterns field | `tests/unit/url-exclusion.test.js:6` |
| url_exclude_patterns: default is empty array when not set | `tests/unit/url-exclusion.test.js:15` |

After step 1, 7 of these pass. The other 2 (`config.test.js:48` and
`url-exclusion.test.js:6`) still fail because they expect `www.cms.gov` (step 2).
[PR #1](https://github.com/mgifford/vital-core-nsf/pull/1) applies both fixes.
With them, `npm run i18n:check`, all 533 unit tests and `npm run test:e2e`
pass locally.

## Go-live checklist

### In this repo

1. **Fix `config/targets.yml`** (done in PR #1). Delete the second `targets:`
   line (262), so the five NSF entries sit under the first `targets:` (175),
   after its comment block. The file then parses and resolves to the five NSF
   hosts and `mgifford/vital-core-nsf-data`.
2. **Fix two unit tests that hard-code CMS targets from the live config**
   (done in PR #1). Otherwise `tests.yml` fails on every push:
   - `tests/unit/config.test.js:48`: expected `www.cms.gov` with
     `design_system: cms-ds`
   - `tests/unit/url-exclusion.test.js:6`: expected `www.cms.gov` with
     `url_exclude_patterns: [".aspx"]`

   Both now use inline YAML with `example.gov`, like the tests around them.
   The design-system test now passes `CMS-DS` in mixed case, so it really
   checks normalization.
3. **Join the history to upstream (one time).** The tree is identical to
   upstream `f5bac30` apart from `targets.yml`, so an `ours` merge records
   upstream as an ancestor without changing any file:

   ```sh
   git remote add upstream https://github.com/mgifford/vital-core
   git fetch upstream main
   git merge -s ours --allow-unrelated-histories upstream/main \
     -m "Record vital-core f5bac30 as upstream base"
   ```

   After this, `git merge upstream/main` only brings in new upstream work. Do
   this before making the NSF doc edits, so those edits show up as ordinary
   changes on top of upstream.
4. **Optional: brand the dashboard.** With no profile, the fleet index reads
   "Weekly quality ledger" with the generic intro. To brand it, add
   `config/profiles/nsf.yml` (name, the five targets, title, intro) and set
   `VITAL_PROFILE: nsf` on the Aggregate step in `.github/workflows/report.yml`.
   Then delete `config/profiles/cms.yml`, `va.yml` and
   `www.cms.gov-top-tasks.txt`. `VITAL_PROFILE=cms` would now throw, because
   those targets are gone.
5. **Check the budget.** `scan.yml` has 48 scheduled runs a week: 6 crons on
   each of 5 weekdays, plus 9 crons on each weekend day. At
   `pages_per_run: 25`, each domain reaches its `max_pages_per_week: 500` cap
   after 20 runs, around Thursday. After that, `list-scan-domains.js` drops it
   in the cheap matrix job, so the remaining runs cost little. That's fine as
   it is. If you'd rather spread the 500 pages over the whole week, lower
   `pages_per_run` to about 10. The comment at the top of `targets.yml`
   (about 100 pages per run and 5 crons) is stale either way.

### GitHub settings (could not check from here; the session had read-only access)

6. **Pages:** Settings → Pages → Source: **GitHub Actions**. The `github-pages`
   environment is created on the first deploy. `report_base_url` already
   assumes `https://mgifford.github.io/vital-core-nsf`.
7. **Workflow permissions:** Settings → Actions → General → **Read and write
   permissions**. Actions needs `contents: write` for the `scan-state` release
   (created automatically on the first scan) and `issues: write` for the
   "Weekly scan reports" issue.
8. **Issues must be enabled.** `report.yml` creates the "Weekly scan reports"
   issue and comments on it.
9. **Data repo deploy key:**
   - Generate a keypair:
     `ssh-keygen -t ed25519 -N "" -C vital-core-nsf-data -f nsf-data-key`
   - Add `nsf-data-key.pub` to **mgifford/vital-core-nsf-data** → Settings →
     Deploy keys, with **Allow write access** checked.
   - Add the private key `nsf-data-key` to **mgifford/vital-core-nsf** →
     Settings → Secrets and variables → Actions as **`VITAL_DATA_DEPLOY_KEY`**.

   The data repo already exists, is public, and has an initial commit on
   `main`. That's enough for the checkout step.

   **Push access is checked first.** Since
   [PR #4](https://github.com/mgifford/vital-core-nsf/pull/4), `scan.yml` (the
   `matrix` job) and `report.yml` (the `gate` job) each start with "Check data
   repo push access" (`scripts/check-data-push.sh`). If the key can't push,
   that step fails before any scan job starts or any NSF page is requested. Its
   annotation, at the top of the run page, carries git's exact message.

   Before that, `actions/checkout` silently fell back to the workflow token
   when `ssh-key` was empty. That token can read the public data repo but not
   push to it. So the whole scan ran, then "Commit data" failed after 5 push
   attempts, after crawl state had already marked the pages scanned. That
   happened on every run from 2026-10-24 (for example
   [report run 40051043543](https://github.com/mgifford/vital-core-nsf/actions/runs/40051043543)).

   What the annotation tells you:
   - "The VITAL_DATA_DEPLOY_KEY repository secret is missing or empty": add
     the private key as a **repository** secret in this repo, named exactly
     that.
   - `The key you are authenticated with has been marked as read only`: the
     deploy key on the data repo doesn't have **Allow write access** ticked.
     Delete it and add it again with the box ticked.
   - `Permission to … denied to deploy key`: the public key was added to the
     wrong repo.
   - `protected branch` or `rule violations`: a branch rule or ruleset on the
     data repo's `main` blocks the key.
   - `Permission denied (publickey)` (at the checkout step instead): the secret
     doesn't match the public key on the data repo.
10. **Keep the data repo public.** The report `gate` job reads
    `.repo-size-kb`, `.last-report-date` and the repo size through
    `gh api repos/<data_repo>` using the workflow's own `github.token`. That
    token can't read another private repo, so the size check would fail the
    job.
11. **First run.** After step 9 is confirmed: Actions → scan → Run workflow,
    with the domain left blank, a small budget (for example 10), and "Run report"
    ticked. That scans all five domains in parallel, commits to the data repo,
    and publishes to Pages.

    Before [PR #2](https://github.com/mgifford/vital-core-nsf/pull/2), the
    matrix filter (`budgetStatus()` in `src/lib/state.js`, used by
    `scripts/list-scan-domains.js`) treated a domain whose crawl state had no
    pages as "frontier empty". So a new domain, or one whose `scan-state` asset
    was lost, was never picked, by the crons or by a blank "Run workflow". That's
    why [scan run 40050709114](https://github.com/mgifford/vital-core-nsf/actions/runs/40050709114)
    succeeded without scanning anything. PR #2 treats an empty state as needing
    a seed, which matches the `Object.keys(state.pages).length === 0` check in
    `scan.js`.

    Don't start several scan runs back to back. `scan.yml` has a `scan-global`
    concurrency group, and GitHub keeps only the newest waiting run, cancelling
    the others.
12. **Watch for scheduled workflows being turned off.** GitHub turns off
    scheduled workflows in a public repo after 60 days without repository
    activity. Scan data commits go to the data repo, not this one, so this repo
    can go quiet. GitHub emails before it does this; re-enable from the Actions
    tab, or push a commit now and then.

### On the NSF side

13. **Crawlability and WAF.** The NSF hosts were blocked by this session's
    network policy, so I could not check them. Locally, run
    `npm run check:public-interest www.nsf.gov par.nsf.gov ncses.nsf.gov iucrc.nsf.gov seedfund.nsf.gov`
    and confirm `robots.txt` and sitemaps. If the hosts return 403s, rewrite
    `WAF-ALLOWLIST.md` for NSF (it is still about HHS and NIH, with the
    upstream user agent) and send NSF the allowlist request.
14. **Tell NSF before calling this official.** The user agent now points site
    owners to `github.com/mgifford/vital-core-nsf`. Contact NSF's web team and
    Section 508 program office in advance, so the first report is not a
    surprise.
15. **Design system:** if NSF's sites are built on USWDS (worth confirming per
    host), set `design_system: uswds` on those targets for component
    clustering. This value is already supported.
16. **Spelling allowlist:** add NSF terms such as `NSF`, `NCSES`, `IUCRC`,
    `SBIR`, `STTR`, `PAPPG` and `NSB`. The current list is health-agency
    vocabulary (Medicaid, Medicare, CHIP, telehealth and similar).

## CMS references, by what to do with them

"CMS" also means *content management system* in this codebase. Those uses are
generic and listed under "Leave as-is".

### Change in this repo (wrong or misleading for NSF)

| Where | What |
|---|---|
| `config/targets.yml:258-261` | Comment about hhs.gov and nih.gov being WAF-blocked. Remove it. |
| `config/targets.yml:177-235` | Example comments (cdc.gov, data.cms.gov, medicare/medicaid, `www.cms.gov-exclude.txt`, CMS spelling examples). Swap in NSF examples or generic ones. |
| `README.md` | No NSF introduction. Line 27 links the data repo to `mgifford/vital-core-data`. Add a short "This instance monitors…" section listing the five hosts, the report URL, and "based on vital-core". |
| `CLAUDE.md:1,9,19,23,117-130` | Title "vital-core", `www.cms.gov` command examples, VA/`hf_only` security rules, and the upstream data repo default. |
| `WAF-ALLOWLIST.md` | All of it: the HHS/NIH status table, the upstream user agent, upstream report and project URLs, HHS/NIH contacts. |
| `.well-known/vital.json` (16), `.well-known/ai.json` (3) | Homepage, repository, API base and docs URLs all point to `vital-core` and `mgifford.github.io/vital-core`. |
| `A11Y-STRATEGY.md:19,49,53,109,146` | Examples built on cms.gov, medicare.gov and medicaid.gov, and a walkthrough using the CMS Physician Fee Schedule Search page. |
| `API.md:37-173` | `www.cms.gov` in every endpoint example. |
| `MCP.md:68-69,96-97` | Sample `.vital.yml` with `api: https://mgifford.github.io/vital-core/api/v1/` and `domain: www.cms.gov`. Copied as-is, it points an NSF user's MCP client at CMS data. |
| `ARCHITECTURE.md:95,222,260-266,338-342` | Upstream data repo default, VA notes, and a `www.cms.gov` config example. |
| `config/profiles/` | `cms.yml`, `va.yml` and `www.cms.gov-top-tasks.txt` (234 CMS URLs). The README says "cms.yml is the only real profile". |
| `config/spelling-allowlist.txt` | Health vocabulary (see item 16 above). |
| `src/lib/acr.js:229-230` | The ACR's `feedback:` and `repository:` fields point to the upstream issue tracker, so feedback on NSF reports would land in vital-core. |
| `src/lib/llms-txt.js:7,60`, `src/lib/api-catalog.js:4` | The published `/llms.txt` and `/.well-known/api-catalog` link to the upstream repo. The docs are identical, so the links work, but they name the wrong project. |
| `src/api/schema/*.schema.json` | `$id` is `https://mgifford.github.io/vital-core/api/v1/schema/…`, but the schemas are served from `vital-core-nsf`. The `index.schema.json` description says "e.g. www.cms.gov". |
| Report UI copy: `src/report-html.js:2020, 3065, 4255`, plus the matching keys in `src/locales/{template,fr,ja,nl}.json` | Visible on every NSF report: the exclusion example `/medicare/`, the acronym example "Centers for Medicare & Medicaid Services (CMS)", and the URL-lookup placeholder "e.g. /medicare or cms.gov/provider". Because the catalogs use the English text as keys, fix these upstream (see `UPSTREAM-SUGGESTIONS.md`) to avoid four-file merge conflicts. |
| `scripts/check-public-interest.js:10-12,30` | `www.cms.gov` usage examples, and a user agent pointing to upstream. |
| `.github/workflows/scan.yml:18,122` | Comment naming the upstream data repo default, and the error-message example `www.cms.gov`. |
| `.env.example` | `HF_TOKEN` (the Hugging Face appliance isn't used here) and upstream's LAN Ollama address `192.168.50.171`. |

### Leave as-is

- **Credits and links to the parent project:** `src/lib/bug-report.js:295`,
  `src/lib/acr.js:383` ("Generated by vital-core"), the Jira footer at
  `src/report-html.js:4308`, the `rel="help"` and MCP links at
  `src/report-html.js:982-983,1022,1026`, and links to upstream issues such as
  `MCP.md:8` (#214).
- **CMS meaning content management system:** `FEATURES.md:37-38,104`,
  `AGENTS.md:170`, `src/scan.js:350`, `src/lib/priority.js`,
  `src/lib/tech-findings.js`, `src/lib/remediation-prompts.js`,
  `MCP.md:178-180`, and the report strings at `src/report-html.js:3087, 3169,
  3217, 4090, 4169, 4180`.
- **Generic capability:** `cms-ds` design-system support in `src/lib/config.js`
  and `src/lib/component-clusters.js`.
- **Upstream history and planning records:** `kitty-specs/`, `kitty-ops/`,
  `.kittify/` and `docs-internal/`. Editing them only creates merge conflicts.
- **Test fixtures:** `tests/fixtures/mcp/.vital*.yml` and test inputs that use
  `www.cms.gov` as sample data.
- **The Hugging Face/VA appliance:** `Dockerfile`, `src/serve-hf.js`,
  `.github/workflows-archived/`. Not used here; you can leave them or delete
  them.
