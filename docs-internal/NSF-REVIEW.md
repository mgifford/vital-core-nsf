# vital-core-nsf: CMS-reference review and go-live checklist

Reviewed 2026-10-23 against `mgifford/vital-core-nsf` at `c3d3486` ("updating
for nsf") and `mgifford/vital-core` at `f5bac30`.

## Summary

- **This repo is a copy of `vital-core` with one changed file.** A recursive diff
  against upstream `main` finds only `config/targets.yml` changed. Every doc,
  profile, test and code string is still the CMS version. No document except
  `targets.yml` mentions NSF.
- **Blocker: `config/targets.yml` does not parse.** It has two top-level
  `targets:` keys, at lines 175 and 262. The `yaml` parser stops with
  `Map keys must be unique at line 262`, so `loadConfig()` throws. The scan
  matrix, `scripts/data-repo.js` and the report build all fail on their first
  step. Nothing will run until this is fixed.
- **The history is not shared with upstream.** This repo has two commits, a
  GitHub "Initial commit" and the copy. Upstream changes can't be merged
  cleanly until the two histories are joined (see step 1).

## Go-live checklist

### In this repo

1. **Fix `config/targets.yml`.** Delete the second `targets:` line (262), so the
   five NSF entries sit under the first `targets:` (175), after its comment
   block. I tested this: the file parses and resolves to the five NSF hosts.
2. **Fix two unit tests that hard-code CMS targets from the live config.**
   Otherwise `tests.yml` fails on every push:
   - `tests/unit/config.test.js:48`: expects `www.cms.gov` with
     `design_system: cms-ds`
   - `tests/unit/url-exclusion.test.js:6`: expects `www.cms.gov` with
     `url_exclude_patterns: [".aspx"]`

   With step 1 applied, these were the only config-related failures. In my
   sandbox, 13 Playwright-based tests also failed, but that was a local
   browser-version mismatch. CI installs Chromium itself.
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
10. **Keep the data repo public.** The report `gate` job reads
    `.repo-size-kb`, `.last-report-date` and the repo size through
    `gh api repos/<data_repo>` using the workflow's own `github.token`. That
    token can't read another private repo, so the size check would fail the
    job.
11. **First run:** Actions → scan → Run workflow, with domain `www.nsf.gov`, a
    small budget (for example 10), and "Run report" ticked. Check the Pages URL,
    then let the crons take over.
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
