# Upstream suggestions for vital-core

Noted 2026-10-23 while setting up `vital-core-nsf`.

**Goal:** `vital-core` becomes a generic engine. Each scanned fleet (CMS, NSF,
and future ones) lives in its own child repo, with its own `-data` repo.
Starting a new fleet should mean cloning, editing a few config files and
setting secrets. Pulling in upstream improvements should be a conflict-free
`git merge upstream/main`.

**Status at 2026-10-27.**
[vital-core#276](https://github.com/mgifford/vital-core/pull/276) ports the
finished work: the tests from §5, the bug fixes in §6 and §10, the fail-fast
push check from §7, the action updates (vital-core-nsf#6), the Setup rewrite
from §11, and a Section 508 baseline for the CMS config. These sections are
still only suggestions:

- §1: repo layout and the rename caveat
- §2: an instance-identity block
- §3: fleet files kept apart from engine files
- §4: generic report text
- the rest of §7: commit before saving state, deploy Pages anyway, warn on an
  empty matrix
- §8: repo variables and keepalive
- §9: a new-fleet script
- the follow-ups at the end of §10
- §11: the `doctor` workflow
- §12, below

## 1. Repo layout and the rename plan

The plan: rename `vital-core` → `vital-core-cms` and `vital-core-data` →
`vital-core-cms-data`, then have a generic `vital-core`.

Two GitHub behaviours matter here:

- **Project Pages URLs don't redirect after a rename.** GitHub redirects
  everything else, but `mgifford.github.io/vital-core/` would 404 the moment
  the repo is renamed. That breaks the published CMS reports, pinned
  `#finding` links, CSV links built from `report_base_url`, and MCP clients
  whose `.vital.yml` uses `.../vital-core/api/v1/`. This goes against the
  "never break a URL" rule in `CLAUDE.md`.
- **Reusing the old name removes the redirect.** If you rename `vital-core` →
  `vital-core-cms` and then create a new `vital-core`, GitHub stops
  redirecting `github.com/mgifford/vital-core/...`. Every existing link would
  then point at the new generic repo, which doesn't have those issues or that
  history. That includes the issue links throughout the docs, code and specs
  (#209, #214 and others), contributors' git remotes, and the contact URL in
  the user agent that site owners see.

**Recommended route:** reach the same end state without renaming the main repo.

1. Keep `mgifford/vital-core` as the generic upstream. It keeps its issues,
   history, stars and every existing link.
2. Create `mgifford/vital-core-cms` as a child, the same way as
   `vital-core-nsf`, and move the CMS fleet config there.
3. Rename `vital-core-data` → `vital-core-cms-data`. This is safe as long as
   nothing reuses the old name: git URLs redirect and deploy keys survive.
   Still set `data_repo` explicitly.
4. Keep the old report URLs working. Have the generic repo's Pages site publish
   a small `404.html` (or per-path redirect stubs, matching the existing
   `redirectStub()` pattern) that sends `/vital-core/<path>#hash` to
   `/vital-core-cms/<path>#hash`.
5. If VA should continue, decide whether it becomes `vital-core-va` too.

If you do rename the main repo anyway, do steps 3 and 4 the same way, and never
create a new repo called `vital-core`. The generic upstream would then need a
different name, such as `vital-core-engine`.

**Create child repos with shared history.** `vital-core-nsf` was made by
copying files into a new repo, so its history isn't connected to upstream and
can't merge it cleanly. For future fleets, clone upstream and push it to the new
remote instead of using "Use this template" or copying files:

```sh
git clone https://github.com/mgifford/vital-core vital-core-foo
cd vital-core-foo
git remote rename origin upstream
git remote add origin git@github.com:mgifford/vital-core-foo.git
git push -u origin main
```

GitHub won't let you fork your own repo into the same account, so a plain
clone with an `upstream` remote is the way to do this. For NSF, the one-time
`git merge -s ours --allow-unrelated-histories upstream/main` described in
`NSF-REVIEW.md` repairs it.

## 2. One "instance identity" block instead of hard-coded URLs

The upstream repo URL and Pages URL are hard-coded in about a dozen places. A
child has to patch code to stop pointing at the parent, which then conflicts
with every upstream merge. Suggestion: a single block in `config/targets.yml`,
with defaults taken from `GITHUB_REPOSITORY` in CI:

```yaml
instance:
  name: NSF
  title: "NSF Accessibility & Sustainability"
  intro: "Automated weekly monitoring of National Science Foundation web properties."
  repo: mgifford/vital-core-nsf             # default: $GITHUB_REPOSITORY
  pages_url: https://mgifford.github.io/vital-core-nsf/   # default: derived from repo
  upstream: mgifford/vital-core
  contact: https://github.com/mgifford/vital-core-nsf/issues
```

Then derive these from it instead of hard-coding them:

| Currently hard-coded | Where |
|---|---|
| Upstream repo docs/blob base | `src/lib/llms-txt.js:7,60`, `src/lib/api-catalog.js:4` |
| ACR `feedback:` and `repository:` | `src/lib/acr.js:229-230` |
| `rel="help"` and MCP doc links in every report | `src/report-html.js:982-983,1022,1026` |
| JSON Schema `$id` | `src/api/schema/*.schema.json` (rewrite at build time) |
| `DEFAULT_DATA_REPO = 'mgifford/vital-core-data'` | `src/lib/config.js:9` (default to `<repo>-data`) |
| User agent contact URL | `defaults.user_agent` in `targets.yml`, `scripts/check-public-interest.js:30` |
| `report_base_url` | `targets.yml` (default to `pages_url`) |
| Fleet dashboard title and intro | currently only possible through a profile |
| Static `.well-known/vital.json` and `ai.json` | repo root (generate them at build time) |

Keep the "Generated by vital-core" credit links pointing to upstream. Those are
correct as they are.

## 3. Keep fleet-specific files apart from engine files

Aim for a child that differs from upstream only in a small, known set of fleet
files:

- `config/targets.yml`: ship `targets.example.yml` upstream, or keep the
  engine defaults (sampling, finding policy, languages) in a separate
  `config/defaults.yml` that the child doesn't touch.
- `config/profiles/*`, top-task lists, `config/exclusions/*`: fleet-only.
  Upstream ships only the README and an example profile.
- `config/spelling-allowlist.txt`: split it into a generic base list and a
  `spelling-allowlist.local.txt` for each fleet. Today it's CMS health
  vocabulary.
- `WAF-ALLOWLIST.md`: a request template upstream, with the per-host status
  table in the child.
- `A11Y-STRATEGY.md`, `API.md`, `MCP.md`, `ARCHITECTURE.md`, `CLAUDE.md`,
  `scripts/check-public-interest.js` usage, and the `scan.yml` error message:
  use `www.example.gov` in examples instead of `www.cms.gov`. A child then
  only needs to edit `README.md`.
- `CLAUDE.md` security rules about VA/`hf_only`: present them as a feature
  ("if a target is `hf_only`…") rather than a fixed fact about this fleet.
- `.env.example`: remove the LAN Ollama address (`192.168.50.171`) and the
  `HF_TOKEN` line, or mark them as examples.

## 4. Generic UI copy in the reports

Three visible strings use CMS examples on every fleet's reports. Their English
text is the translation key in `src/locales/{template,fr,ja,nl}.json`, so the
fix belongs upstream. A child can't change them without four-file merge
conflicts.

- `src/report-html.js:2020`: exclusion help example `/medicare/` → for
  example `/news/`
- `src/report-html.js:3065`: acronym example "Centers for Medicare & Medicaid
  Services (CMS)" → for example "World Wide Web Consortium (W3C)"
- `src/report-html.js:4255`: URL lookup placeholder
  "e.g. /medicare or cms.gov/provider" → for example
  "e.g. /about or example.gov/contact"

Better still, take the placeholder examples from the fleet's own first target.

## 5. Tests shouldn't depend on the live fleet

`tests/unit/config.test.js:48` and `tests/unit/url-exclusion.test.js:6` assert
CMS-specific values from the real `config/targets.yml`. Any child fails CI as
soon as it changes its fleet. Both are rewritten with inline YAML in
[vital-core-nsf#1](https://github.com/mgifford/vital-core-nsf/pull/1), and those
two changes can be ported upstream as they are. More generally: tests may
check that the live config loads and validates, but never what it contains.
Also consider a `check:config` step that parses `targets.yml` and every
profile, so a broken config fails with a clear message. The duplicate
`targets:` key in `vital-core-nsf` broke 9 tests with an indirect error.

## 6. Bug: scheduled runs never seed a new domain

`scripts/list-scan-domains.js` builds the scheduled matrix with
`budgetStatus()` from `src/lib/state.js`. For a domain with no saved state,
`loadState()` returns `pages: {}`, so `frontierEmpty` is `true` and the domain
is skipped as "frontier empty (no scannable pages)". Seeding from the sitemap
and homepage only happens inside `src/scan.js` (`seededAt`), so it never gets
the chance.

This affects:

- every target added to an existing fleet
- every target of a brand-new child repo
- any domain whose `<domain>.crawl.json.gz` asset on the `scan-state` release is
  lost. That contradicts the "missing asset means the domain reseeds from its
  sitemap" note in `scan.yml` and `README.md`.

A "Run workflow" with the domain left blank goes through the same filter. Only
single-domain dispatches seed.

Fixed in [vital-core-nsf#2](https://github.com/mgifford/vital-core-nsf/pull/2),
which can be ported upstream as it is. `budgetStatus()` now reports
`frontierEmpty: false` when the state has no pages, matching the
`Object.keys(state.pages).length === 0` seed check in `scan.js`. The PR also
adds an "unseeded" case to the `budgetStatus` unit test in
`tests/unit/lib.test.js`. A seed always adds at least the homepage, so a seeded
domain never stays empty, and once its pages are scanned or failed out it is
skipped as before.

## 7. Bug: a failed data push throws away the scan

In `.github/workflows/scan.yml`, the scan job runs these steps in this order:

1. "Restore crawl state"
2. "Scan"
3. "Save crawl state" (`if: always() && steps.restore.outcome == 'success'`)
4. "Commit data"

So crawl progress is uploaded to the `scan-state` release before the page
records are committed. If the push to the data repo fails, those pages stay
marked `lastScannedWeek: <this week>`, but their records are lost when the
runner is discarded. They won't be scanned again until the next ISO week.

**Seen in `vital-core-nsf` on 2026-10-24.** The deploy key couldn't push. Ten
scan runs (one manual, nine scheduled) each failed at "Commit data". By then,
about 970 NSF pages were marked done for 2026-W43, with no data kept. The
scheduled runs kept crawling NSF and discarding the results.

The problem gets in quietly. `actions/checkout` with an empty `ssh-key` falls
back to the workflow token. That token can read a public data repo but can't
push to it. So with the `VITAL_DATA_DEPLOY_KEY` secret missing, or the deploy
key read-only, checkout, Chromium install and the whole scan succeed. Only the
final push fails, after five retries.

Suggested changes, in order of value:

- **Commit before saving crawl state.** Give "Commit data" an `id: commit`,
  move it above "Save crawl state", and gate the save on it:
  `if: always() && steps.restore.outcome == 'success' && steps.commit.outcome == 'success'`.
  Rescanning is safe. Page records are written to
  `data/<domain>/<week>/pages/<pageId>.json` (`src/scan.js:421`), so scanning
  the same page twice in a week overwrites one file. At worst it adds an extra
  `runs/*.json` log.
- **Fail fast on push access, before scanning.** Implemented in
  [vital-core-nsf#4](https://github.com/mgifford/vital-core-nsf/pull/4); it can
  be ported as it is. `scripts/check-data-push.sh` does two things:
  - fails with a clear `::error::` if the secret is empty (passed as
    `HAS_DEPLOY_KEY` through `env:`, since `secrets` can't be read in `if:`)
  - otherwise runs `git push --dry-run origin HEAD` and puts git's message in
    the annotation

  It runs early in each workflow:
  - in the `scan.yml` `matrix` job, against a root-files-only checkout of the
    data repo, so no scan job starts
  - in the `report.yml` `gate` job, so the report job never builds

  Annotations can be read through the checks API, unlike step logs, so the
  cause shows on the run page.
- **Don't let a data-repo problem take the site down.** In `report.yml`,
  "Configure Pages" and the deploy steps come after "Commit summaries and
  pruning". So a push failure also stops the site from updating, even though
  `docs/` was already built. Consider deploying when the commit fails, with a
  `::warning::`. `aggregate.js` recomputes summaries from the page records on
  the next run, so nothing is lost by deploying first.
- **Make an empty matrix visible.** A manual "Run workflow" where every domain
  is skipped (the §6 bug, or every domain at its cap) ends green with only
  `::notice::` lines. For a `workflow_dispatch`, emit a `::warning::` and put the
  reasons in the job summary, so a scan that scanned nothing doesn't look like
  a successful scan.

## 8. Workflows a child can configure without editing them

- Let `report.yml` read the profile from a repo variable
  (`VITAL_PROFILE: ${{ vars.VITAL_PROFILE }}`). A child can then brand its
  dashboard from Settings → Variables, which matches the profile README's
  rule that profiles shouldn't need workflow edits.
- Consider repo variables for the cron timezone notes and the
  `CEILING_MB` / `CODE_CEILING_MB` size limits too.
- **Scheduled workflows can be turned off for inactivity.** Data commits now go
  to the data repo, so a code repo can go 60 days without activity, and GitHub
  then turns off scheduled workflows in public repos. Document this, or add a
  lightweight keepalive.

## 9. A new-fleet setup script (optional)

A `scripts/new-fleet.sh` using `gh` could:

- create `<name>` and `<name>-data` (public, with an initial commit)
- generate the deploy key, add the public half to the data repo with write
  access, and set the `VITAL_DATA_DEPLOY_KEY` secret
- set Pages to GitHub Actions and workflow permissions to read/write
- create the `instance:` block and an empty `targets.yml` from the example
- print the "first run" steps

This would turn the GitHub settings part of `NSF-REVIEW.md` into one command.

## 10. Separate legal requirements from the build target, AAA and best practice

Recorded 2026-10-25. Implemented in
[vital-core-nsf#8](https://github.com/mgifford/vital-core-nsf/pull/8), and every
part of it applies upstream unchanged.

On the NSF reports, "Fix these first" ranked findings by pages × severity ×
reach and nothing else. AAA criteria (1.4.6, 1.4.8, 2.5.5) and best practices
filled all eight rows, while the three findings against required WCAG 2.0
A/AA and 2.2 AA criteria didn't appear at all. Four separate problems
combined:

1. **The ranking ignored the finding policy.** `rankBugs()` in
   `src/lib/priority.js` never read `obligation`, even though
   `applyFindingPolicy()` had already classified every finding as required,
   aspirational, advisory or unmapped. The "Biggest available win" callout and
   the fleet's "Worst offenders" table used the same ranking.
2. **45 Alfa rule names had drifted onto other rules.** In
   `config/alfa-rule-labels.json`, most titles from about `sia-r40` onwards
   described a different rule. For example:
   - `sia-r57` (text in a landmark) was called "Scrolling content can be
     accessed by keyboard", which is `sia-r84`
   - `sia-r61` (page starts with an h1) was called "Form field label is
     visible"
   - `sia-r73` (paragraph line height) was called "Text has minimum contrast
     ratio of 4.5:1"

   `check:alfa-metadata` only checks that a label exists, so nothing caught it.
3. **Alfa best-practice rules were "Undetermined".** Only axe's
   `best-practice` tag was recognised. Twenty Alfa rules that declare only an
   Alfa best practice or a WAI-ARIA authoring rule fell through to
   `unmapped`, and their summaries read "(WCAG criterion undetermined)".
4. **There was one conformance target, with no legal baseline.** The policy
   couldn't express "WCAG 2.0 AA is the legal minimum (Section 508), and WCAG
   2.2 AA is what we build to". It also put an A/AA criterion newer than the
   target (for example 2.5.8 under a 2.0 target) in the AAA lane.

What the PR changes, all of it portable as is:

- **`finding_policy.conformance.legal_baseline`:** an optional
  `{ name, version, level }`, validated so it can't sit above the target.
  `applyFindingPolicy()` adds a `conformance_tier` to each finding:
  - `legal`, a required finding within the baseline
  - `target`, a required finding in the rest of the target
  - `required`, when no baseline is configured
  - otherwise the finding's obligation (`aspirational`, `advisory`,
    `unmapped`, `not-applicable`)

  Obligations themselves are unchanged, so the ACCESSIBILITY.md finding
  schema vocabulary still holds. Out-of-target A/AA criteria now get
  `not_required_for_conformance` instead of `wcag_aaa`.
- **`src/lib/priority.js`:** ranks by tier first (`CONFORMANCE_TIER_ORDER`),
  then by the old score.
- **"Fix these first":** lists required findings only, with a new
  "Requirement" column, for example "Legally required · WCAG 1.3.1 (2.0,
  Level A)". AAA, best-practice and unmapped findings go in a separate
  "Beyond the requirements" table. The "biggest win" callout, the fleet
  table and each finding on the Accessibility page show the same label.
- **Alfa best practices:** `scripts/generate-alfa-wcag-rules.mjs` now writes a
  `best_practice` list to `src/data/alfa-wcag-rules.json`, taken from each
  rule's own `requirements[]`. `classifyFinding()` and `classifyBaseLane()`
  use it. Technique-only and requirement-less rules are deliberately left
  Undetermined.
- **Labels:** the 45 wrong labels are corrected against each rule's
  `requirements[]` and `Outcomes`, `sia-r84` gets a label, and regression
  tests are added.

Still to do upstream:

- **Set the baseline for the CMS fleet too.** CMS is a federal agency, so the
  same Section 508 baseline applies:
  `legal_baseline: { name: Section 508, version: "2.0", level: AA }`.
- **Make the label check catch drift.** Store each label with the requirement
  it was written against, for example
  `"sia-r57": { "title": "...", "requirements": ["ARIA:landmark-regions"] }`.
  `check:alfa-metadata` can then fail when the installed package's
  `requirements[]` differ, which is when a label needs re-checking. Run it in
  `tests.yml`, and after every `@siteimprove/alfa-rules` upgrade.
- **Alfa severity.** Alfa has no impact levels. `severityFor()` gives every
  Alfa finding "Moderate", raised to "Serious" when it affects half the pages
  or more. So severity on Alfa AAA and best-practice findings says little.
  The tier column now carries the meaning, but consider showing Alfa severity
  as "—" or "Not rated".
- **Show the tier in exports.** `conformance_tier` is in `bugs.json` and the
  JSON API (both allow extra fields), but not yet in `bugs.csv`. Adding a
  column changes `bugsCsvTable()` and the browser serializer that mirrors it.

## 11. Setup documentation and a setup check

Recorded 2026-10-25, from the NSF go-live.

The README "Setup" section was written before scan data moved to its own repo
on 2026-10-14. It has no step for the data repo, the deploy key or the
`VITAL_DATA_DEPLOY_KEY` secret. ARCHITECTURE.md and CLAUDE.md mention the key
only as something that already exists. So a new deployment that follows the
README ends up where NSF did: every scan and report failed at the final push.
Suggested replacement steps:

1. Clone vital-core and push it to the new repo, so the history stays shared
   with upstream (§1). Don't copy files or use "Use this template".
2. Create a public `<name>-data` repo with an initial commit. Set `data_repo:`
   and the `user_agent` contact URL in `config/targets.yml`. Set
   `finding_policy.conformance` and, where a law applies, `legal_baseline`
   (§10).
3. Create a deploy key:
   - public half on the data repo, with **Allow write access** ticked
   - private half as the `VITAL_DATA_DEPLOY_KEY` *repository* secret in the
     code repo
4. Configure the code repo:
   - Settings → Pages: source **GitHub Actions**
   - Settings → Actions: **read and write** workflow permissions
   - Issues turned on
5. Run `scan` with the domain left blank and **Run report** ticked. Don't start
   several scans at once: GitHub keeps only the newest waiting run.
6. Check that it worked: commits in the data repo, the Pages site, and a
   "Weekly scan reports" issue. A green run alone doesn't prove anything was
   scanned.

Also fix the "wasa" typo in the README's Hugging Face paragraph.

**A `doctor` workflow.** A hand-run workflow that checks a deployment in under
a minute, without loading any target site, and reports pass or fail for each
item on the run page:

- `targets.yml` and every profile parse
- the data repo exists and is public
- the secret is set and `git push --dry-run` is accepted (reuse
  `scripts/check-data-push.sh`)
- Pages is set to GitHub Actions, and Issues is on
- the `scan-state` release can be created

Run it after setting up a fleet and whenever settings change. It would have
caught the NSF parse error and the missing deploy key on day one.

## 12. Smaller things found on the way

Recorded 2026-10-27.

- **Anchor the data ignore rule.** `.gitignore`'s `data/` also matches
  `src/data/`, so `git add src/data/alfa-wcag-rules.json` fails with "paths
  are ignored" even though the file is tracked. That aborted a chained
  commit during the NSF work. Use `/data/`, plus `/data/www.va.gov/` and the
  other VA entries, so the rule only matches the checkout at the repo root.
- **Make delete-on-merge the default.** Turn on Settings → General →
  "Automatically delete head branches" in vital-core and every child repo, so
  merged PR branches don't pile up.
- **Clear the existing Dependabot security alerts.** vital-core-nsf inherited
  10 on day one (5 high, 5 moderate) from the shared `package-lock.json`, and
  they're cleared upstream once for every fleet. The github-actions
  Dependabot config only covers action versions.
- **Mention the session's restrictions in agent docs.** The Claude Code
  session used for NSF couldn't open Actions step logs, call GraphQL (`gh pr
  create`, `gh release download`), or reach the scanned sites. Workflows that
  put their important errors in annotations (§7, `check-data-push.sh`) stay
  diagnosable from such sessions.
