# Upstream suggestions for vital-core

Noted 2026-10-23 while setting up `vital-core-nsf`.

**Goal:** `vital-core` becomes a generic engine. Each scanned fleet (CMS, NSF,
and future ones) lives in its own child repo, with its own `-data` repo.
Starting a new fleet should mean cloning, editing a few config files and
setting secrets. Pulling in upstream improvements should be a conflict-free
`git merge upstream/main`.

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
- **Fail fast on push access, before scanning.** Right after the data repo
  checkout:
  - fail with a clear `::error::` if the secret is empty (pass it through
    `env:`, since `secrets` can't be read in `if:`)
  - then run `git push --dry-run origin HEAD`

  A read-only or missing key should then fail in seconds, without loading the
  target site. Use the same check in `report.yml` before "Aggregate".
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
