# Early Gate Block 4 — Jenkins workflows (POC copies)

`earlygate-smoke-trigger.yaml` and `monitor-jenkins-job.yaml` in this directory
are POC copies of the files of the same name in
[`red-hat-data-services/rhods-devops-infra`](https://github.com/red-hat-data-services/rhods-devops-infra/tree/main/.github/workflows).

They exist here so the POC can point Block 4's `github-workflow-repo` param at a
repository it controls, and so the input contract can be fixed on both sides at
once. The Jenkins calls are the real ones — nothing is stubbed.

**The filenames are the addresses.** Block 4's Tekton tasks dispatch these
workflows by filename through the `workflow_dispatch` REST API. Renaming either
file breaks the dispatch with a 404.

---

## Why the copies were needed

Block 4 (`konflux-central` PR #55) dispatches both workflows. Pointed at the
production repo, the monitor works and the trigger does not:

| | Production `earlygate-smoke-trigger.yaml` | What Block 4 sends |
|---|---|---|
| `repositories` | required | ✅ sent |
| `pr_number` | required | ✅ sent |
| `correlation_id` | required | ✅ sent |
| `fbc_tag` | **required** | ❌ never sent |
| `rhoai_version` | not declared | ✅ sent |
| `operator_image` | not declared | ✅ sent |
| `bundle_image` | not declared | ✅ sent |
| `fbc_image` | not declared | ✅ sent |

GitHub rejects a dispatch carrying undeclared inputs, and rejects one missing a
required input. Either alone is **HTTP 422** before the workflow starts. Block 4
diagnoses it by name:

> `422 is usually one of: … does not declare one of the inputs above; the three
> image inputs are the recent addition`

This is a design difference, not an oversight. Production installs the catalog
**by tag**; the early gate hands images over **by digest**, because a tag is
mutable — rebuild the same commit and it moves, so a smoke test that resolved a
tag could install something other than what the gate built and verified.

---

## Input mapping

| Block 4 sends | This copy declares | Forwarded to Jenkins as |
|---|---|---|
| `repositories` | `repositories` (required) | `GITHUB_REPO` |
| `pr_number` | `pr_number` (required) | — (correlation only) |
| `correlation_id` | `correlation_id` (required) | `CORRELATION_ID` |
| `rhoai_version` | `rhoai_version` *(new)* | `RHOAI_VERSION` |
| `operator_image` | `operator_image` *(new)* | `OPERATOR_IMAGE` † |
| `bundle_image` | `bundle_image` *(new)* | `BUNDLE_IMAGE` † |
| `fbc_image` | `fbc_image` *(new)* | `FBC_IMAGE` † |
| — | `fbc_tag` *(now optional)* | `RHOAI_VERSION` fallback |
| — | `send_digests` *(new, default `false`)* | — |

† only when `send_digests` is `true`. See below.

---

## Adaptations, and why each one

Every divergence is marked `# EG ADAPTATION` in the file.

1. **`fbc_tag` required → optional.** Block 4 does not send it. Kept rather than
   deleted so existing `repository_dispatch` callers keep working.
2. **`rhoai_version` added.** Production overloads `fbc_tag` to mean both the tag
   to install and the version to test. Block 4 separates them.
3. **`operator_image`, `bundle_image`, `fbc_image` added.** The substantive
   change — the gate's by-digest handover.
4. **`send_digests` added, default `false`.** The Jenkins job
   `devops/early-gate-tests` declares `RHOAI_VERSION`, `GITHUB_REPO` and
   `CORRELATION_ID` and nothing else. Forwarding digests is only useful once
   Jenkins declares parameters to receive them, and Jenkins drops — or on a
   strict job, rejects — undeclared parameters. **Off by default means the first
   run exercises the proven call.** Flip it on when Jenkins is ready; no logic
   changes.
5. **Resolve-RHOAI-version step added.** `RHOAI_VERSION` now comes from
   `rhoai_version`, falling back to `fbc_tag`, and fails loudly if neither
   arrives rather than handing Jenkins an empty string.
6. **Release-tag rejection step added.** `trigger-test-pipeline` already enforces
   DR-6, but this workflow is also reachable by hand and by
   `repository_dispatch`, where that check does not run.

`monitor-jenkins-job.yaml` has **no adaptations** — it is byte-identical to
production below its header comment. Its `queue-url`/`job-url` inputs are both
optional, which is exactly what Block 4 dispatches, and it already echoes
`jenkins-job-url=` and `jenkins-job-status=` to stdout.

> Both workflows must keep echoing those values to **stdout**, not only to
> `GITHUB_OUTPUT`. `workflow_dispatch` runs do not expose job outputs over the
> REST API — only `workflow_call` does, and a reusable workflow cannot be
> dispatched — so Block 4 scrapes the run log. The same applies to
> `Queue URL: …` in the trigger.

---

## Prerequisites before the first run

None of these are in this repo today. All three are required.

| Prerequisite | Needed for | Status in `rhoai-rhtap/RHOAI-Build-Config-gap` |
|---|---|---|
| Secrets `JENKINS_USERNAME`, `JENKINS_TOKEN` | Jenkins crumb + build trigger | ❌ no repo secrets exist |
| Environment `redhat-internal` | Both jobs declare it | ❌ no environments exist |
| Self-hosted runner labelled `self-hosted, devops` | `jenkins-csb-rhods-opendatascience.dno.corp.redhat.com` is **internal** — a GitHub-hosted runner cannot reach it | `devops-ci-host-7` carries the `devops` label; confirm it is reachable from this repo |

Also needed outside this repo:

- **`eg-github-token` must carry `actions:write` on this repository**, or the
  dispatch returns 404 with a message that looks identical to a missing file.
- **Block 4's `github-workflow-repo`** must point at
  `rhoai-rhtap/RHOAI-Build-Config-gap` (its current default) and
  **`workflow-ref`** at the branch these files land on.
- **`rhods-ci` secret with key `jenkins-url`**, or `jenkins-base-url` passed
  explicitly — Block 4 uses it to recognise the queue URL in the logs.

---

## What production `rhods-devops-infra` needs

This file set is the specification for the handover PR. Only the trigger
workflow changes:

1. Make `fbc_tag` optional with a default of `""`.
2. Declare `rhoai_version`, `operator_image`, `bundle_image`, `fbc_image`.
3. Resolve `RHOAI_VERSION` from `rhoai_version` with an `fbc_tag` fallback.
4. Optionally adopt `send_digests` and the release-tag guard.

`monitor-jenkins-job.yaml` needs **nothing** — production already satisfies the
contract on both sides.

> ⚠ **Do not carry the `runs-on` line back.** Both copies here run on
> `[self-hosted, devops]` because this fork's runner is `devops-ci-host-7`;
> production runs on the konflux runners. That divergence is environmental,
> not contractual, and is marked `EG ADAPTATION` at the line in both files.

**Separately, and not blocking:** for the gate to install what it actually
built, the Jenkins job `devops/early-gate-tests` has to accept a by-digest
image reference instead of resolving `RHOAI_VERSION` to a tag. Until then
`send_digests` stays `false` and the smoke test installs the tag — which
validates the plumbing end to end, but not the gate's central promise. That
Jenkins change is owned by DevOps.
