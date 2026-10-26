# llm-d-router personal contribution handoff

Audit date: 2026-10-26 (America/Chicago). Repository: fork `htang7415/llm-d-router` of `llm-d/llm-d-router`, local checkout of `main`.

Goal: understand EPP request handling, scheduling plugins, and tests well enough to submit small, defensible upstream contributions.

## Current status

- Phase: orientation and triage complete; no contribution branch, claim, or PR exists.
- Next action: create an isolated upstream baseline worktree and run the focused picker tests for #3533 (step 1-2 below).
- Blocked on: a public design decision for #3533; dependency readiness for #3558.

## Publication

This file is the canonical status. A copy of it is published as the only file on the orphan branch `contrib-status` of the fork, for reading across sessions. Raw evidence (request transcripts, cluster and GitHub snapshots) and personal notes stay in local ignored directories (`.local-contrib/evidence/`, `local-contrib/`) and are not published; references to them below are local-only.

Evidence labels: **Verified** means observed during this audit; **Historical** means an existing note or artifact; **Unknown** means no sufficient execution evidence. GitHub availability is a dated observation, not a reservation.

## Verified Git state

| Item | Observation |
| --- | --- |
| Fork / origin | `https://github.com/htang7415/llm-d-router.git` (fetch and push) |
| Upstream | `https://github.com/llm-d/llm-d-router.git` (fetch and push) |
| Local branches | Only `main`, tracking `origin/main` |
| Local HEAD and live fork main | `732c959b0090fbc0188a136d93f53a85e5236987` |
| Latest upstream main / HEAD | `b5cb5f881ce85f340739a4d310e16295af01d5a6` |
| Divergence | `main...upstream/main`: 0 local-only, 34 upstream-only commits; local main equals origin/main |
| Worktrees | Only this checkout, on main; no detached or additional worktree |
| Work in progress | No staged, unstaged, or visible untracked files |
| Ignored work preserved | `local-contrib/`, `coverage/epp.out`, `.DS_Store` |
| Status branch | `contrib-status` on origin only; orphan history containing only `PROJECT_STATUS.md`; no local branch |

Ran `git fetch --no-tags --no-prune upstream` and confirmed the upstream SHA with `git ls-remote`. Fetch updated remote-tracking refs and fetch metadata; local branches and working files were left unchanged. Origin main was checked live without fetching origin; other origin branch refs were not refreshed. The upstream range changes 223 files, including EPP scheduling, data-layer contracts, telemetry, and e2e setup.

`/.local-contrib/` is excluded through `.git/info/exclude`; `git check-ignore -v` confirms the rule. The tracked `.gitignore`, `AGENTS.md`, and `CLAUDE.md` were not edited. `CLAUDE.md` is a symlink to `AGENTS.md`. The existing `local-contrib/` remains the source of detailed workflow and learning notes; this handoff references it without moving or replacing it.

## Local environment (verified)

| Item | Observation |
| --- | --- |
| Host | macOS 27.0.1, arm64 |
| Container runtime | Docker Desktop 4.96.0, client/server 29.9.1, linux/arm64 server, context `desktop-linux` |
| Kind | v0.33.0; one cluster, `llm-d-router-dev` |
| Kubernetes context | `kind-llm-d-router-dev`; explicit context used for cluster inspection |
| Node | One Ready control-plane node, Kubernetes v1.31.12 |
| Tools | Git 2.56.0, gh 2.102.0, Go 1.27.3, Helm 4.3.0; Python 3, curl, jq, rg available; Podman not found |
| Make | `make` is GNU Make 3.81; use `/opt/homebrew/bin/gmake` (4.4.1) for the documented Make 4+ requirement |
| kubectl | `/usr/local/bin/kubectl` v1.36.1; five minor versions above the API server |

All four default-namespace deployments have one ready replica:

- Gateway: Istio proxy 1.29.2; Gateway Accepted/Programmed, HTTPRoute Accepted/ResolvedRefs. NodePort `30080` is published to the host.
- EPP: `ghcr.io/llm-d/llm-d-router-endpoint-picker:dev`, ext_proc service port 9002; metrics auth and leader election disabled in the deployment arguments.
- Decode simulator: `ghcr.io/llm-d/llm-d-inference-sim:v1.0.0`, TinyLlama model, echo mode, KV cache disabled, data-parallel size 1.
- Renderer: `vllm/vllm-openai-cpu:v0.21.0`, port 8082. Recent readiness and liveness probe timeouts exist in Warning events; it was ready and served the smoke checks.

The InferencePool selects the simulator and references EPP with `FailClose`. No separate prefill, encoder, or routing-sidecar deployment is present. No Prometheus pod was observed. Restart counts at inspection: Gateway 2, EPP 1, simulator 7, renderer 2; counts alone do not establish their causes.

The deployed `epp-config` ConfigMap explicitly configures approximate-prefix production, a decode filter, prefix-cache scoring (weight 2), max-score picking, and a single-profile handler. This is a single decode-endpoint experiment; it does not demonstrate affinity across multiple endpoints.

Running EPP image ID: `sha256:7e8c00e8e65f9bad68862d0d8afc7299bdb85567264c89f45015940d1b40da08`. The host's `:dev` image has no revision labels. Its source commit and equivalence to the running image are **Unknown**; a mutable tag and creation date do not prove provenance.

## Completed experiments

- **Historical, owner-reported:** created a Kind environment with Gateway, EPP, and vLLM Simulator; model listing and chat requests succeeded. Existing notes do not contain the original raw request transcripts.
- **Historical, code reading:** local technical notes (`local-contrib/notes.md`) trace handlers, Director, scheduling profiles, plugin hooks, and state lifetime at `732c959b`.
- **Historical, investigation:** local issue #3468 notes (`local-contrib/issues/3468.md`) analyze producer state on early returns. The proposed regression has not been reproduced locally; reading another contributor's PR is not test execution.

## Test evidence

Three bounded requests were run at 2026-10-26 16:04:59-16:05:02 UTC against `http://127.0.0.1:30080`; each had a 12-second timeout. The inference requests used `max_tokens: 3`.

| Check | Verified result | Evidence / limit |
| --- | --- | --- |
| GET `/v1/models` | HTTP 200; TinyLlama listed | Local raw result: `smoke-models.json` |
| POST `/v1/completions` | HTTP 200; response text `hi`, valid usage | Local request and response: `smoke-completion.json` |
| POST `/v1/chat/completions` | HTTP 200; assistant content `hi`, valid usage | Local request and response: `smoke-chat.json` |
| Existing EPP coverage profile | Atomic profile; 75.0% statement coverage | `coverage/epp.out`, 1,916,556 bytes, mtime 2026-10-22 01:37:22 UTC; original command, SHA, and test exit status unknown |
| `gmake help` | Exit 0 | Duplicate `bin` target recipe warnings; no build or tests executed |
| Unit/race, hermetic integration, build, presubmit, e2e | Not run in this audit | No current-upstream pass claim |

Smoke results establish that the existing simulated deployment is usable. Real model generation, GPU KV transfer, P/D or encoder disaggregation, streaming, performance, and multi-pod scheduling are **Unknown**. Historical coverage cannot establish a complete successful suite or validate the 34 new upstream commits.

## Upstream guidance and contribution status

Read [DEVELOPMENT.md](https://github.com/llm-d/llm-d-router/blob/b5cb5f881ce85f340739a4d310e16295af01d5a6/DEVELOPMENT.md), [CONTRIBUTING.md](https://github.com/llm-d/llm-d-router/blob/b5cb5f881ce85f340739a4d310e16295af01d5a6/CONTRIBUTING.md), architecture, and the PR template from the fetched SHA. Local snapshots are kept with the private evidence. Also read the linked [organization contribution guide](https://github.com/llm-d/llm-d/blob/main/CONTRIBUTING.md).

- Prefer a narrow issue and existing plugin/test patterns. Discuss new behavior before implementation; public APIs or new components can require a project proposal.
- CONTRIBUTING describes `/assign`, but this audit did not claim an issue or send any public message. Repository instructions require explicit per-action authorization for public activity, and approval before pushing, dependency upgrades, or edits under `.github/` or to `OWNERS`.
- Use DCO sign-off (`git commit -s`) and the repository PR template, with the exact `release-note` fenced block. Human authors must understand and defend AI-assisted changes.
- DEVELOPMENT describes presubmit as including tests. The actual upstream recipe has branch/signature/module/format/lint/security/vulnerability/tag/boilerplate checks and no unit-test prerequisite. Run relevant tests separately. Presubmit also runs formatting and module tidy and rejects `main`.
- Upstream #3466 changes e2e targets to use the simulator for rendering. Real vLLM rendering requires a make command-line override, `VLLM_RENDER_IMAGE=...`; an environment variable does not override the target assignment. The preserved dev cluster still uses the real CPU renderer.
- Verified: no upstream PRs returned by `gh pr list --author htang7415 --state all`. No contribution branch, implementation, commit, push, review, PR, or issue claim was made during this audit.
- #3468 remains open/unassigned, but [PR #3484](https://github.com/llm-d/llm-d-router/pull/3484) is open and implements it (`REVIEW_REQUIRED`). Keep it as a learning exercise; avoid a duplicate implementation.

## Ranked candidate issues

Screened 179 open unassigned issues and 158 open PRs, then read candidate bodies/comments, paginated timelines, all-state PR reference searches, related terms, and relevant upstream code/tests. Both entries below were open/unassigned, with no implementation PR found for their specific scope. Search cannot exclude unpublished work. Acceptance ratings are judgment, not maintainer commitments. Both have `needs-triage`.

| Rank | Candidate | Scope / reproducibility | Learning value | Acceptance outlook / blocker |
| --- | --- | --- | --- | --- |
| 1 | [#3533: opt-in exploration for zero-score endpoints](https://github.com/llm-d/llm-d-router/issues/3533) | Small-to-medium picker feature after the contract is agreed. Existing unit tests reproduce default zero-score exclusion; no cluster needed. New behavior has not been tested locally. | High: plugin factory/config, weighted sampling, multiple selections, backward compatibility. | Moderate, conditional on design agreement. Related #3470 is merged and supplies baseline behavior; it does not implement exploration. Agree on normalized exploration share, score-floor versus pick-budget semantics, and `maxNumOfEndpoints > 1`. |
| 2 | [#3558: reconcile LoRA residency estimates](https://github.com/llm-d/llm-d-router/issues/3558) | Medium, stateful scorer/data-layer work. CPU tests can inject successive scrape snapshots; live validation needs residency gauges. | High: scraped state versus inferred state, pod lifecycle, concurrent response hooks. | Plausible follow-up to merged #3392, but blocked for an immediate contribution by open [#2792](https://github.com/llm-d/llm-d-router/pull/2792) and the vLLM gauge contract. Upstream scorer still explicitly documents that reconciliation is absent. |

No third candidate is recommended. Exclusions: #3626/#3627 already say a patch is ready; #3641 has another contributor offering a fix; #3444/#1950 have active implementation intentions; #2783 has an informal claimant; #1472 has an existing closed implementation PR (#1582); #2698 points to merged fixing work (#2680); #2881 has a broader design discussion and volunteered implementation. Unassigned alone is insufficient evidence of availability.

## Blockers and unknowns

- No source-pinned baseline test results for current upstream; existing EPP image provenance is unknown.
- kubectl/server skew exceeds the supported one-minor range ([policy](https://kubernetes.io/releases/version-skew-policy/#kubectl)). Read-only inspection worked; use a compatible client before infrastructure testing, without changing this cluster.
- Renderer probe timeouts warrant observation if they recur. No restart or configuration change was attempted.
- #3533 needs a public design decision; #3558 needs dependency readiness. No candidate is claimed or immediately cleared for implementation.
- No original unit-test transcript or complete presubmit/e2e record was found in the inspected local notes and coverage directory. Other locations were not searched.

## Prioritized next steps and safe synchronization plan

1. Create a separate local baseline worktree from the recorded upstream SHA. Leave this checkout, its ignored experiments, and main intact. Proposed commands below were **not executed**; first verify the branch/path are unused and recheck live upstream.

   ```sh
   git fetch --no-tags --no-prune upstream
   git ls-remote upstream HEAD refs/heads/main
   git worktree add -b audit/upstream-baseline-b5cb5f88 \
     ../llm-d-router-upstream-b5cb5f88 \
     b5cb5f881ce85f340739a4d310e16295af01d5a6
   ```

   If upstream moved, record that SHA and choose explicitly whether to reproduce this pinned baseline or test the newer one. No merge, rebase, reset, push, or remote status branch is needed. If adding personal notes inside another worktree, verify its local ignore rule first.

2. In the new worktree, read its AGENTS/CLAUDE instructions and `docs/architecture.md`. For #3533, read `pkg/epp/framework/plugins/scheduling/picker/{weightedrandom,random,maxscore}` and their tests. Use the builder (`gmake builder-shell`), then run `go test -count=1 -race ./pkg/epp/framework/plugins/scheduling/picker/weightedrandom` inside it. Builder image acquisition/build can take time; budget it separately. Record SHA, command, exit code, and actual test output.
3. Prepare a short design question for the owner to send on #3533 after rechecking ownership and PRs. Agree on the contract before substantial implementation. Then add a failing feature regression and make it pass while retaining default behavior; keep the work within this issue.
4. After the focused check, run EPP unit/race tests, hermetic integration, relevant build checks, and `gmake presubmit` on a dedicated contribution branch. Inspect any format/tidy diff. Run sidecar/framework or coordinator checks when the affected paths require them. Record failures and skips; do not equate presubmit with all CI.
5. Defer cluster tests to a separate pass. Inspect suite lifecycle and use a separate cluster, unused host ports, and an isolated kubeconfig; never point a suite that creates/cleans resources at the preserved dev environment. Rebuild with unique SHA tags and capture running digests, then repeat model/completion/chat checks and any issue-specific routing scenario. Verify the upstream e2e renderer choice before selecting simulator versus real rendering.
6. Before submission, recheck issue ownership and implementation PRs, review every added constant against repository values, keep the diff narrow, and use the required PR template. Pushes of contribution branches, claims, and PR creation remain separate authorized actions. Publishing this file to `contrib-status` follows the local procedure in `.local-contrib/SYNC.md`.

## Evidence locations (local only, not published)

`.local-contrib/evidence/smoke-*.json` contains bounded request transcripts. Command/result snapshots include Git state, live remote SHAs, pods, routes, deployments, EPP config/images, warnings, tool help, and coverage summary. `issues.json`, `prs.json`, `issue-*.json`, `timeline-*.json`, and `pr-search-*.json` support triage. `preserved-files.json` records hashes of the existing contribution notes, coverage profile, AGENTS, and tracked gitignore for preservation checks. Detailed personal workflow remains in `local-contrib/WORKFLOW.md` and `local-contrib/COMMUNITY.md`.
