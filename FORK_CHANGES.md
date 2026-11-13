# Fork Changes

Every deliberate divergence of `UniversityOfSaintThomas/aquinas-ai-platform` from upstream `Boise-State-Development/agentcore-public-stack`.

> [!IMPORTANT]
> This file is the living register. The Deployment Plan and SDLC document includes a summary; this file is the authoritative detail. If they disagree, this file is correct; update the plan to match.

**Branches:**
- `main` follows upstream, updated with GitHub's *Sync fork*, and carries no St. Thomas changes.
- `develop` is where St. Thomas changes are made and deploys to the development environment.
- `production` deploys to the production environment.

A promotion is the pull request from `develop` into `production`. Each branch's upstream version is in its `VERSION` file.

**Maintained on `develop`.** This file reaches `production` with the next promotion. It is kept off `main`, because a local commit there means `main` no longer matches upstream, and *Sync fork* can then hit conflicts that GitHub asks you to resolve with a pull request.

**Not every entry is on every branch.** Entry 1 exists only on `production`, so a reader on `develop` will not find it in their working tree, and that is expected. Entries 2 to 10 exist on `develop` and reach `production` at the next promotion.

## Summary

| # | Change | Files | Branches | Upstream status | Retires when |
|---|---|---|---|---|---|
| 1 | Workflow trigger retargeting | 3 workflows | `production` only | Not applicable, local policy | Never |
| 2 | SPA hosted zone lookup | `spa-distribution-construct.ts` | `develop`, both after promotion | Fix ready; pull request not yet opened | Upstream fixes it |
| 3 | Rebranding | `brand.config.ts` plus assets | `develop`, both after promotion | Not applicable, local identity | Never |
| 4 | `favicon.ico` hand-maintained | `public/favicon/favicon.ico` | `develop`, both after promotion | Defect, not yet reported | Upstream fixes it |
| 5 | `log_success` in teardown script | `scripts/teardown/managed-kb.sh` | `develop`, both after promotion | Fixed upstream in v1.33.0 (#1808) | The 1.35.0 upgrade |
| 6 | St. Thomas base system prompt | `system_prompt_builder.py` (6 lines) | `develop`, both after promotion | Not applicable; a configurable prompt is an issue candidate | Upstream makes it configurable |
| 7 | St. Thomas web crawler user agent | `crawler.py` (1 line) | `develop`, both after promotion | Not applicable; a configurable user agent is an issue candidate | Upstream makes it configurable |
| 8 | System prompt date in Central time | `timezone.py` (2 lines), `test_timezone.py` | `develop`, both after promotion | A configurable timezone is an issue candidate | Upstream makes it configurable |
| 9 | Backport: attachment eviction tests | `test_attachment_index.py` (test only) | `develop`, both after promotion | Fixed upstream in `e4b1961` (v1.31.0) | The 1.35.0 upgrade |
| 10 | Backport: conversation search tests | `test_conversation_search_route.py` (test only) | `develop`, both after promotion | Fixed upstream in `12643da3` (v1.34.0) | The 1.35.0 upgrade |

---

## 1. Workflow trigger retargeting

**Branches:** `production` only

**Files:** `.github/workflows/platform.yml`, `backend.yml`, `frontend-deploy.yml`

**Commits:** `91d006bc` on `production` (the 18 lines that existed at 1.22.0) and `bb035449` (the 7 lines added by 1.25.0 to 1.28.0, retargeted in the 1.30.0 promotion)

Upstream deploys on pushes to `main`. Our `main` receives *Sync fork* updates and must deploy nowhere, so on the `production` branch every deploy trigger and environment check is pointed at `production` instead:

```yaml
# upstream                         →  # ours (production branch only)
branches: [develop, main]          →  branches: [develop, production]
github.ref == 'refs/heads/main'    →  github.ref == 'refs/heads/production'
```

Make these edits during each promotion merge, on the branch that is merged into `production`, before it is merged. `main`'s own copies are left untouched, so *Sync fork* stays conflict-free.

**Upstream keeps adding references, and nothing announces it.** New jobs bring new `refs/heads/main` lines, and the merge that brings them is clean, so no conflict flags them. Any reference left unedited on `production` sends its job to `development`, because the environment expression falls back to `'development'`. The `development` environment allows only the `develop` branch, so that job fails on its environment protection rules instead of deploying, and the production deploy is left incomplete. The worst case is in `frontend-deploy.yml`:

```yaml
SPA_BUILD_CONFIGURATION: ${{ (github.ref == 'refs/heads/main' && 'production') || 'dev-deploy' }}
```

Left unedited on `production`, it resolves to `dev-deploy`, so the production build uses `environment.development.ts` instead of `environment.production.ts`, with a green build. The two files have the same feature flags today, but upstream turns new features on in the development file first, so they can differ.

**Check after every sync** (this also catches the three `branches:` lines, which don't contain `refs/heads/`):

```bash
grep -cE "refs/heads/main|branches: \[develop, main\]" \
  .github/workflows/{platform,backend,frontend-deploy}.yml
```

To check `production` without switching to it, read each file from that branch:

```bash
for f in platform backend frontend-deploy; do
  git show production:.github/workflows/$f.yml | grep -cE "refs/heads/main|branches: \[develop, main\]"
done
```

| Branch | Expect | Meaning |
|---|---|---|
| `develop` | `2 20 3` | upstream's counts; a higher number means new references arrived: edit them on `production` at the next promotion, and update `2 20 3` here and in Maintenance step 3 |
| `production` | `0 0 0` | every reference retargeted; anything else means one was missed |

**What actually stops `main` from deploying** is the deployment branch policy on the `production` GitHub Environment, which allows only the `production` branch. `main`'s workflows still resolve `environment: production` on every *Sync fork* push, so this policy is a required setup step, not optional hardening. After each *Sync fork*, expect Platform Stack, Backend Deploy and Frontend Deploy to fail on `main` with "Branch "main" is not allowed to deploy to production due to environment protection rules." That failure is the policy working: their tests run, the deploy job is refused, and nothing reaches AWS. Do not loosen the policy to make them pass. To confirm it is set: GitHub → Settings → Environments → `production` → Deployment branches should show one rule, branch `production`.

---

## 2. SPA hosted zone lookup

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `infrastructure/lib/constructs/spa/spa-distribution-construct.ts`

**Commit:** `459f2b9` on `develop`

```typescript
// upstream
domainName: config.domainName

// ours
domainName: config.infrastructureHostedZoneDomain ?? config.domainName
```

Upstream looks the hosted zone up by the record's own domain. When the deployment domain sits beneath the hosted zone, `fromLookup` matches nothing and synth fails with `Found zones: [] for … but wanted exactly 1`.

The artifacts, MCP sandbox and ALB DNS constructs already look up by `infrastructureHostedZoneDomain`. The SPA construct was the only outlier, which is the strongest argument that this is an upstream defect rather than a preference. The commit also adds a `// FORK CHANGE:` comment above the line.

**The `??` fallback rather than a non-null assertion is deliberate.** The `!` form breaks a transport-security test that constructs the SPA distribution (`transport-security.test.ts`) with `domainName` set and no `infrastructureHostedZoneDomain`, so with `!` the lookup would receive no domain.

**Currently inert.** It only manifests where domain differs from hosted zone. Development uses `dev.nonprod-ai.stthomas.edu` for both, and production is planned to use `ai.stthomas.edu` for both, so no environment triggers it. The fix stays in place because it is correct in general and because a future flat-zone environment would need it.

The fix is ready to submit upstream; the pull request has not been opened yet (todo item 12).

---

## 3. Rebranding

**Branches:** `develop`; reaches `production` at the next promotion

**Files:** under `frontend/ai.client/`: `src/branding/brand.config.ts`, `src/styles/generated/brand-theme.css`, `public/img/logo-light.png`, `public/img/logo-dark.png`, `public/favicon-source.png`, `public/favicon/` (five PNGs, `favicon.ico`, `site.webmanifest`), and `src/version.ts` (see below)

**Commit:** `0fc9d15` on `develop`

`frontend/ai.client/src/branding/brand.config.ts`:

| Setting | Value |
|---|---|
| `appName` | `Aquinas AI` |
| `pageTitle` | `Aquinas AI` |
| primary | `#510c76` |
| secondary | `#8348ad` |
| tertiary | `#4c9c2e` |

Source assets, under `frontend/ai.client/`: `public/img/logo-light.png` and `public/img/logo-dark.png` (both 500x500, byte identical), `public/favicon-source.png` (512x512), and `public/favicon/favicon.ico` (see entry 4).

`surfaces` is left at `DEFAULT_SURFACES`, and the greeting arrays are left at their defaults, all of which were verified institution-neutral.

**`appName` must stay at 12 characters or fewer.** `generate-favicons.ts` performs an unguarded `substring(0, 12)` when writing the web manifest's `short_name`, so a longer value is silently truncated mid-word.

**Generated output is committed.** The following travel with this change: `src/styles/generated/brand-theme.css`, the five `public/favicon/*.png` files, and `site.webmanifest`. Only `brand-theme.css` is pinned by a spec (`brand-theme-golden.spec.ts`). Any future edit to `brand.config.ts` or `favicon-source.png` must be followed by `npm run prestart`, with the regenerated files committed. A colour change left unregenerated fails `brand-theme-golden.spec.ts` in CI; a new source image left unregenerated leaves the old favicon PNGs in place, with no error.

**`frontend/ai.client/src/version.ts` also differs from upstream, and is not a deliberate change.** The npm `prebuild` hook runs `scripts/gen-version.js`, which writes it from the root `VERSION` file. `0fc9d15` committed it at `'1.22.0'`, the `VERSION` at the time (the commit message calls it prebuild churn), while upstream's committed copy at v1.30.0 reads `'1.14.0'`. The file's own header says production builds overwrite it. Upstream's copy is restored at the 1.35.0 upgrade, after which this paragraph is removed.

**Not covered by this change.** Upstream hard-codes "Boise State" in user-facing text that `brand.config.ts` does not reach: the home link on the shared artifact page (`shared-artifact-view.page.ts`, `boisestate.ai`), the agent share dialog (`share-agent-dialog.component.ts`, "Everyone at Boise State"), the store submission dialog (`submit-listing-dialog.component.ts`, "everyone at Boise State can open it") and the marketplace publisher hint (`marketplace.model.ts`, "agents that speak for Boise State"). They are left as upstream until a pull request makes them configurable (PR TODO).

---

## 4. `favicon.ico` is hand-maintained

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `frontend/ai.client/public/favicon/favicon.ico`

**Commit:** `0fc9d15` on `develop` (the same commit as entry 3)

Listed separately from entry 3 on purpose, because it behaves unlike every other favicon file.

`generate-favicons.ts` never writes `favicon.ico`. It logs `ℹ️  favicon.ico: using existing file (set by preexisting build)` and never produces one: `generateIco` (line 84) is an unfinished stub that returns `null` and is never called. This is despite the file's header comment claiming it emits "favicon.ico (all sizes)".

So the `.ico` does **not** track `favicon-source.png`. Anyone who replaces the source image and runs the generators will watch five PNG files update and reasonably conclude the `.ico` updated too. It did not.

Current file is a three-resolution ICO at 48, 32 and 16 pixels, 32-bit BMP encoding, built by hand.

Replacing it requires a tool that writes multi-resolution ICO files, run on `favicon-source.png` at 48, 32 and 16 pixels. Note that the inaccurate header comment is itself worth reporting upstream, since it is what makes the behaviour misleading rather than merely limited. Not yet reported upstream; to be raised as its own issue (PR TODO).

---

## 5. `log_success` defined in `managed-kb.sh`

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `scripts/teardown/managed-kb.sh`, 10 lines added

**Commit:** `34eecec` on `develop`

An upstream defect from at least v1.25.1 until v1.33.0, not an adaptation to our environment.

`managed-kb.sh` calls `log_success` at lines 228, 293 and 313 in upstream's v1.25.1 to v1.30.0 file (238, 303 and 323 in ours, after the fix), but sources only `scripts/common/load-env.sh`, which defines `log_info`, `log_warn`, `log_error` and `log_config` and nothing else. `destroy.sh` does define its own `log_success` at line 20, but invokes this script with `bash` at line 174, a separate process, so the definition is not inherited. Standalone invocation has the same gap.

**It blocked teardown entirely for any environment with zero managed knowledge bases**, which is the default state for any deployment that never enabled the managed KB backend. The undefined function returns 127, so the `exit 0` immediately below line 228 is never reached, and `destroy.sh` reads that exit code as knowledge base teardown having failed. It then halts before deleting any stack, which is correct behaviour on a false signal: the Bedrock service role lives in `PlatformStack`, so deleting that first would strand real knowledge bases.

Lines 293 and 313 are worse in principle. They fire after deletions have already succeeded, so an environment that does have managed knowledge bases would delete them and then report failure.

Fixed by defining the function once, guarded with `declare -f` in the same style as the source block directly above it, rather than patching three call sites:

```bash
if ! declare -f log_success >/dev/null 2>&1; then
    log_success() {
        echo -e "${GREEN:-}[SUCCESS]${NC:-} $1"
    }
fi
```

The `:-` defaults on `GREEN` and `NC` are deliberate. Those variables come from `load-env.sh`, whose sourcing the guard above may skip. The script runs with `set -u`, so an unset `GREEN` or `NC` would otherwise stop it with an "unbound variable" error.

**Fixed upstream in v1.33.0 (#1808), with the same guarded definition.** Upstream's v1.35.0 `managed-kb.sh` differs from ours only in the four comment lines above the guard. At the 1.35.0 upgrade, take upstream's file, then delete this entry and its summary row.

---

## 6. St. Thomas base system prompt

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `backend/src/agents/main_agent/core/system_prompt_builder.py`, 6 lines in `DEFAULT_SYSTEM_PROMPT`

**Commit:** `e3999d6` on `develop`

Upstream's base prompt, sent at the start of every chat, introduces the assistant as "boisestate.ai… created for Boise State University". It is a hard-coded constant: admin system prompts, agent instructions and personal instructions are added after it and cannot replace it.

**Changed:** the identity sentence ("You are Aquinas AI, an AI assistant created for University of St. Thomas Minnesota…") and three "Boise State" → "St. Thomas" swaps, at lines 57 to 59, 65, 76 and 86. Everything else is upstream's text.

**On a sync conflict:** keep upstream's new text and reapply these four changes. A conflict happens only when upstream changes these lines or the ones next to them.

**Left as upstream:** Boise mentions in the backend that users never see (`.env.example`, a docstring in `proxied_redirect.py`, an example in `scheduled_prompts/models.py`). User-visible Boise text in the web app is in entry 3.

Retires if upstream makes the base prompt configurable.

---

## 7. St. Thomas web crawler user agent

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `backend/src/apis/app_api/web_sources/crawler.py`, line 58

**Commit:** `5d1188f` on `develop`

```python
# upstream
USER_AGENT = "BoiseStateAI-Assistant-Crawler/1.0 (+contact)"

# ours
USER_AGENT = "AquinasAI-Assistant-Crawler/1.0 (+https://www.stthomas.edu)"
```

Every site crawled for a web source sees this string, so upstream's value identified us as Boise State, with a placeholder where a contact should be. The constant is also what the crawler checks robots.txt against (`parser.can_fetch(USER_AGENT, url)`), so rules aimed at `BoiseStateAI-…` no longer apply to us; only `*` rules and rules for `AquinasAI-…` do. No test refers to the string. The `fetch_url_content` tool sends its own generic user agent (`StrandsAgent/1.0`, in `url_fetcher.py`), which does not name an institution and is left as upstream.

The contact is the St. Thomas homepage for now. A dedicated ITS page or mailbox could be better before production, but it is not required.

Retires if upstream makes the crawler user agent configurable.

---

## 8. System prompt date in Central time

**Branches:** `develop`; reaches `production` at the next promotion

**Files:** `backend/src/agents/main_agent/utils/timezone.py` (2 lines), `backend/tests/agents/main_agent/utils/test_timezone.py`

**Commit:** `cfb2954` on `develop`

The base system prompt ends with `Current date: …`, from `get_current_date_pacific()`, which upstream hard-codes to `America/Los_Angeles` with no override. For St. Thomas that meant a PDT/PST label on every prompt, and the previous day's date from 00:00 to 01:59 Central every night.

Both zone lookups (the `zoneinfo` branch and the `pytz` fallback) now use `America/Chicago`, each with a trailing `# UST fork: Central, see FORK_CHANGES.md` comment, so the diff stays at two lines.

**Deliberately left as upstream:** the function name `get_current_date_pacific`, the variable `pacific_tz`, and the docstring, comments and log message that still say Pacific. Renaming the function would touch its one call site, the exports in two `__init__.py` files, and the prompt-builder tests, which mock it by name. The behaviour is Central; the names are not.

**Prompt caching is unaffected.** The date line is part of the Bedrock prompt-cache prefix and must stay the same all day. With `America/Chicago` it renders one string per Central day; on the two daylight saving change days the label flips once (CDT↔CST), as the Pacific version already did.

`test_timezone.py` pinned Pacific behaviour. Six of its tests now use Central (four renamed from Pacific to Central), and a new test (`test_uses_central_date_not_pacific_date`) pins the fix itself: at 05:30 UTC the date must be Central's, not Pacific's previous day.

Retires if upstream makes the prompt timezone configurable.

---

## 9. Backport: attachment eviction tests no longer expire on a calendar date

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `backend/tests/shared/test_attachment_index.py` (+12 −3, test only)

**Commit:** `c0f7486c` on `develop`, cherry-picked with `-x` from upstream `e4b1961` ("test(files): stop the attachment eviction tests expiring on a calendar date (#1704)", upstream author kept)

An upstream defect in a test, not in the product, and already fixed upstream. `TestEviction` and `TestReconcile` compared rows against a fixed `NOW` of 2026-11-20 but built those rows with today's real creation date. From 00:00 UTC on 2026-10-21 the gap fell under the 30-day retention window and both eviction assertions failed. Because every Backend build job depends on `test-backend`, this blocked all Backend deploys from `develop` on v1.30.0.

Upstream fixed it on `develop` on 2026-10-21, three days after v1.30.0 was tagged. The fix pins the rows' creation time to the same `NOW`-relative instant. Production code is unchanged.

**Temporary.** Upstream's copy of the file at v1.31.0 and v1.35.0 is identical to ours, so the upgrade merge should apply cleanly or report the change as already present. After the 1.35.0 upgrade, confirm the file matches upstream and delete this entry and its summary row.

---

## 10. Backport: conversation search tests no longer expire on a calendar date

**Branches:** `develop`; reaches `production` at the next promotion

**File:** `backend/tests/shared/test_conversation_search_route.py` (+7 −1, test only)

**Commit:** `3a6dc678` on `develop`, cherry-picked with `-x` from upstream `12643da3` ("test(search): date the fake KB turns relative to now", upstream author kept)

An upstream defect in a test, not in the product, and already fixed upstream. The fake knowledge base document's default `created_at` was a fixed 2026-10-07. With the test's 30-day retention it aged past the window on 2026-11-06, and `test_turns_past_retention_are_dropped` failed. Because every Backend deploy job depends on `test-backend`, this blocked all Backend deploys on v1.30.0, including production's first (2026-11-13).

Upstream fixed it on `develop` on 2026-11-06, and released it in v1.34.0. The fix dates the fake turns an hour before now. Production code is unchanged.

**Temporary.** With this backport, our copy of the file is identical to upstream's at v1.34.0 and v1.35.0, so the upgrade merge should apply cleanly. After the 1.35.0 upgrade, confirm the file matches upstream and delete this entry and its summary row.

---

## Maintenance

After every *Sync fork* onto `main`:

1. Update your local `main` from GitHub (`git fetch origin` and `git checkout main && git pull`), then review the incoming diff before merging to `develop`. *Sync fork* updates GitHub's `main`, not your local copy.
2. Merge to `develop` and validate there first, never straight to `production`.
3. On `develop`, run the check in entry 1 and expect `2 20 3`. A higher count means upstream added a reference and `production` will need a matching edit during the next promotion.
4. At the next promotion, after the merge into `production`, run the same check (or the `git show` loop in entry 1) and expect `0 0 0`.
5. Check whether any upstream change has made an entry here redundant (see the summary's "Retires when" column). Leaving a stale fork change in place is its own source of conflict, so remove an entry as soon as upstream's version takes over.
6. Record any new divergence here in the same commit, or the same pull request, that introduces it. A fork change that is not recorded here is indistinguishable from an accident on the next sync. This includes turning on a web app flag: those flags are set in `frontend/ai.client/src/environments/environment.development.ts` and `environment.production.ts`, not by GitHub variables, so changing one is a fork change.
7. Check that the St. Thomas changes survived the sync. Each command should print the number shown:
   - entry 2: `grep -c "infrastructureHostedZoneDomain ?? config.domainName" infrastructure/lib/constructs/spa/spa-distribution-construct.ts` → `1`
   - entry 3: `grep -c 'appName: "Aquinas AI"' frontend/ai.client/src/branding/brand.config.ts` → `1`
   - entry 6: `grep -ciE "boise" backend/src/agents/main_agent/core/system_prompt_builder.py` → `0`
   - entry 7: `grep -c "AquinasAI-Assistant-Crawler" backend/src/apis/app_api/web_sources/crawler.py` → `1`
   - entry 8: `grep -c "America/Chicago" backend/src/agents/main_agent/utils/timezone.py` → `2`
