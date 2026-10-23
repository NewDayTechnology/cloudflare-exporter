# Public-fork GitHub hardening review and action plan

Assessment date: 23 October 2026.
Repository: `NewDayTechnology/cloudflare-exporter`.
Reviewed default-branch revision: `64d366f3f143ec1db36262e84b63345504a13bb9`.

## Objective and scope

Keep this repository public and accept pull requests from external forks while
preventing untrusted contributions from gaining publication authority, accessing
credentials or internal infrastructure, or consuming uncontrolled CI resources.

This was a read-only review of live repository settings, available organization
settings, all six workflow files, ownership configuration, and associated build
and release inputs. The clean checkout matched the live default-branch revision;
inspected workflow/build files were verified against that revision. No GitHub
settings were changed during the initial review, no workflows were triggered,
and no secret values or workflow logs were retrieved. Subsequently authorized
implementation changes are recorded separately below.

No confirmed anonymous fork-PR-to-write-token or secret-exfiltration exploit was
established. Findings below describe observed control gaps and conditional risks,
not evidence of compromise. This is not an exhaustive application-code audit.
Denied or unsupported APIs are recorded as unknown, not as disabled controls.

## Executive findings

| Priority | Severity | Observed gap | Impact and qualification |
|---|---|---|---|
| P0 | High | No applicable branch/tag rulesets; `master` explicitly unprotected | A compromised or malicious writer can change trusted code or publishing refs without enforced review. Opening a public PR does not itself grant this authority. |
| P0 | High | Default workflow token permissions are write; workflow PR approvals enabled | Jobs without explicit permissions inherit excessive authority in eligible trusted contexts. Ordinary external-fork PR tokens remain subject to GitHub's read-only restrictions. |
| P0 | High | No CODEOWNERS file or enforced code-owner reviews | No ownership boundary protects workflow changes, publishing inputs, or ownership rules themselves. |
| P1 | Medium | Mutable action references; SHA enforcement disabled | Compromised upstream references could affect trusted publishing execution if the workflows are enabled and permitted. |
| P1 | Medium | 39 effective admins, four writers; two admin teams | Large privileged-access surface requires justification and least-privilege review. Counts are effective roles, not necessarily direct grants or unnecessary access. |
| P1 | Medium | Twelve organization GitHub Apps installed for all repositories | Several have broad write permissions. No malicious installation was identified; selected-repository installations require further applicability checks. |
| P1 | Medium | Inherited Docker Hub destination is `lablabs/cloudflare_exporter` | Confirm authorization and intended namespace before enabling publication or supplying credentials. No usable registry credential was established. |
| P2 | Medium | Code-scanning default setup not configured; no scanning workflow found | Detection gap, not proof of an application vulnerability. |
| P2 | Low | Private vulnerability reporting and immutable releases disabled | Improve disclosure and release integrity. This fork currently has no releases. |

Confidence is high in the observed values. Severity describes consequences under
the prerequisites above, not the likelihood of anonymous exploitation.

### Existing controls to preserve

- Approval is required for **all external contributors'** fork PR workflows.
- Actions use a selected-actions policy rather than unrestricted execution.
- All six workflow definitions use GitHub-hosted Ubuntu runners.
- No `pull_request_target`, `workflow_run`, reusable workflow, local action, or
  explicit cross-workflow artifact handoff was found.
- Secret scanning, push protection, Dependabot alerts and security updates are
  enabled. Organization two-factor authentication is required.
- Some publishing action references are already pinned to full commit SHAs.

Approval is a spending/review gate, not a sandbox or proof that PR code is safe.
Do not weaken isolation after approving a run.

## Authorized high-priority implementation

The findings and inventory below describe the original assessment. The following
changes were subsequently authorized and applied on 23 October 2026:

| Control | Current implementation status |
|---|---|
| Default token permissions | Live read-back confirms `default_workflow_permissions: read`. |
| Workflow PR approvals | Live read-back confirms `can_approve_pull_request_reviews: false`. |
| External-fork approval | Preserved and verified as `all_external_contributors`. |
| Protected `master` | Active ruleset `25719082`: PR required, one approval, code-owner review, stale-review dismissal, latest-push approval, conversation resolution, no force pushes or deletion, no bypass actors. |
| Tag creation | Active ruleset `25719081`: all tags restricted to `@NewDayTechnology/platform-engineering` (team ID `8825124`). This is the only creation-rule bypass. |
| Tag immutability | Active ruleset `25719083`: all tag updates and deletions blocked, with no bypass actors. |
| CODEOWNERS | Proposed in this PR: `* @NewDayTechnology/platform-engineering`, including ownership of CODEOWNERS and workflows. Team visibility and effective admin/write access verified. **Not effective until merged into the base branch.** |
| Explicit workflow permissions | Proposed in this PR: chart PR checks get `contents: read`; existing binary/chart release jobs get only `contents: write` rather than inheriting broad defaults. |

The owner team must independently review and merge this PR. No review requirement
was bypassed. No collaborator grants were changed.

Required status checks have intentionally not been configured: the workflow
registration problem must be resolved and stable checks observed before making
them mandatory. The new master ruleset therefore enforces review, not CI success.
Code-owner approval cannot protect this bootstrap PR using an ownership file that
has not yet been merged; require an independent platform-engineering reviewer.

**Intentional release restriction:** owner-team-only tag creation also prevents
chart automation from creating release tags with `GITHUB_TOKEN`. Until a separate
release identity/process is explicitly authorized, owners must create release
tags. There is no blanket automation bypass. Existing explicit package/OIDC/
attestation permissions in container jobs have not been changed; lowering token
defaults does not remove permissions explicitly requested by a job.

Read-back verified the active rulesets, effective master rules, token defaults,
and preserved external-fork approval policy. Controlled fork-PR, merge-review,
and denied-push behavior tests remain pending; no unauthorized push or destructive
tag operation was attempted as a test.

## Workflow-by-workflow review

The Actions workflow API listed only a dynamic Dependency Graph workflow; the
six source workflows returned 404 on individual workflow lookups. The run
inventory showed one dependency-graph run. The YAML files are confirmed live
source, but their registration, enablement, and actual execution in this fork
are **not established**. Investigate this before relying on any required check
or assuming a publisher is active.

| Workflow | Boundary and required changes |
|---|---|
| [golangci-lint.yml](.github/workflows/golangci-lint.yml) | `push`/`pull_request`, explicit `contents: read`. Preserve unprivileged PR execution. Pin action SHAs and the linter version instead of `latest` (line 20). |
| [lint-test.yaml](.github/workflows/lint-test.yaml) | PR-triggered chart installation into kind executes PR-controlled content. Add explicit read-only permissions; keep secrets, write tokens, persistent runners, and internal networks out. |
| [docker-master.yaml](.github/workflows/docker-master.yaml) | Publishes on `master` pushes. Protect source; remove unused `id-token: write` (lines 9-12). Distribute the exact built digest, not re-resolved `latest`. Verify Docker Hub destination (lines 64-72). |
| [docker-release.yaml](.github/workflows/docker-release.yaml) | Publishes on matching tags. Protect tag creation/update/deletion and verify reviewed source ancestry. Attestation explains OIDC/attestation permissions, but does not prove source approval. Distribute the attested digest; verify Docker Hub destination (lines 110-118). |
| [go-build.yml](.github/workflows/go-build.yml) | Release-triggered binary publication inherits token defaults. Set only needed release permissions. Replace `thatisuday/go-cross-build@v1` and `skx/github-action-publish-binaries@master` (lines 14 and 22) with reviewed immutable references or a simpler trusted publishing process. |
| [release.yaml](.github/workflows/release.yaml) | Chart publication on `master` pushes inherits token defaults. Protect source and use explicit permissions. Pin setup-Helm/chart-releaser references. Commented Helm download is not executing code. |

Additional observations:

- Checkout steps do not explicitly disable persisted credentials. Use
  `persist-credentials: false` unless a specific step needs authenticated Git.
- Mutable checkout, login, attestation and Skopeo references remain; setup-go and
  setup-ko in container publishing already demonstrate SHA pinning.
- Container publishers disable setup-go caching. No demonstrated PR-to-publisher
  cache-poisoning path was found. Ordinary fork PR cache scope is not equivalent
  to default-branch cache scope.
- Tag interpolation is constrained by numeric/dotted trigger patterns. No shell
  injection finding is asserted; prefer environment variables and validation.
- [Makefile](Makefile) builds/tests repository code and invokes end-to-end tests.
  [run_e2e.sh](run_e2e.sh) sources `.env`; no inspected workflow invokes it.
  Keep credentialed Cloudflare tests out of untrusted PR jobs.
- [dependabot.yml](.github/dependabot.yml) covers Docker and GitHub Actions,
  not Go modules. No tracked Dockerfile, local action, or ko configuration was
  found. Explicitly maintain the ko base image as well as module dependencies.
- Container builds disable SBOM production. Release images request provenance
  attestations; verify provenance and publish artifact-specific SBOMs.
- [Chart defaults](charts/cloudflare-exporter/values.yaml) reference the upstream
  image; [deployment rendering](charts/cloudflare-exporter/templates/deployment.yaml)
  uses a tag. Decide intentionally whether this fork should distribute upstream
  or fork-built images; support digest-based deployment where needed.

## Live settings inventory

API paths below are relative to `/repos/NewDayTechnology/cloudflare-exporter`
unless prefixed with `/orgs/`. Results are a point-in-time assessment.

| Surface / GET endpoint | Observed result |
|---|---|
| Repository metadata | Public, active fork of `lablabs/cloudflare-exporter`; default branch `master`; `allow_forking: true`; caller has repository admin access. |
| `/actions/permissions` | Actions enabled; `allowed_actions: selected`; `sha_pinning_required: false`. |
| `/actions/permissions/selected-actions` | GitHub-owned and verified-creator actions allowed; 45 additional patterns, generally `@*`, including an explicit `@master` reference. Broader than current dependencies. |
| `/actions/permissions/workflow` | `default_workflow_permissions: write`; `can_approve_pull_request_reviews: true`. |
| `/actions/permissions/fork-pr-contributor-approval` | `approval_policy: all_external_contributors`. |
| `/actions/permissions/fork-pr-workflows-private-repos` | 422: private-repository policy not applicable here. Not evidence of fork secret/write-token access. |
| `/actions/permissions/access` | 422: internal/private sharing policy not applicable. |
| `/actions/runners` | Zero repository-listed runners; organization runner exposure still unverified. |
| `/actions/permissions/artifact-and-log-retention` | 90 days; maximum allowed 90. |
| `/actions/cache/usage`, `/actions/artifacts` | Zero active caches/bytes and zero artifacts. |
| `/actions/cache/usage-policy` | 404; policy unverified. |
| `/actions/workflows`, `/actions/runs` | Only dynamic dependency-graph workflow/run observed; see source-registration qualification above. |
| `/branches` | Only `master`; `protected: false`. |
| `/rulesets?includes_parents=true`, `/rules/branches/master` | Empty, including applicable inherited rules visible through these endpoints. |
| `/branches/master/protection` | Explicit "Branch not protected" response. |
| `/tags/protection` | Legacy endpoint 404; not independently used as evidence that protection is absent. |
| `/codeowners/errors?ref=master` | 404; absence of CODEOWNERS independently confirmed from the live tree. No entries exist to validate. |
| `/tags`, `/releases` | Empty. |
| `/immutable-releases` | Disabled; not enforced by owner. |
| `/environments`, `/deployments` | Empty; no existing environment approval gate. |
| `/actions/oidc/customization/sub` | Default subject enabled; immutable subject enabled, repository-ID-qualified prefix. External trust policies not visible here. |
| `/actions/secrets`, `/actions/variables` | Zero repository entries; organization inheritance unknown. No values retrieved. |
| `/dependabot/secrets`, `/codespaces/secrets` | Zero repository entries. |
| `/hooks`, `/keys`, `/invitations` | No repository webhooks, deploy keys, or pending invitations. |
| `/collaborators` | Fully paginated: 39 admin, four write, 150 read. No identities reproduced. |
| `/collaborators?affiliation=outside` | Zero outside collaborators. External PR authors do not need collaborator access. |
| `/teams` | Two admin teams; one push team. |
| `/installation` | 401: requires GitHub App JWT; not evidence of absent installations. |
| `/orgs/NewDayTechnology/installations` | Fifteen installations: twelve all-repository, three selected-repository. Among all-repository installations, write grants include contents (five), workflows (two), administration (two), actions (four), PRs (six); counts overlap. |
| `/orgs/NewDayTechnology/packages?package_type=container` | 403: missing `read:packages`; package visibility/access/history unverified. |
| Repository `security_and_analysis` | Dependabot security updates, secret scanning, non-provider patterns and push protection enabled. AI detection, validity checks and delegated bypass/dismissal features disabled; not individually treated as vulnerabilities. |
| `/vulnerability-alerts`, `/automated-security-fixes` | Alerts enabled; automatic security fixes enabled and not paused. |
| `/dependency-graph/sbom` | SPDX document available, 56 packages. Not a substitute for a published image/binary SBOM. |
| `/dependabot/alerts?state=open`, `/secret-scanning/alerts?state=open` | Zero open alerts returned; not a clean bill of health. |
| `/code-scanning/default-setup` | `state: not-configured`; Go recognized. |
| `/code-scanning/analyses`, `/code-scanning/alerts` | 404 "no analysis found" with scope warning; no coverage established. |
| `/code-security-configuration` | Organization configuration attached; dependency/secret protections enabled, code-scanning setup not set; `enforcement: unenforced`. |
| `/private-vulnerability-reporting` | Disabled. |
| Merge options | Merge, squash and rebase enabled; auto-merge, automatic branch deletion, update-branch option and web signoff requirement disabled. Not vulnerabilities by themselves. |
| Features | Issues/discussions disabled; wiki/projects enabled. |
| `/pages` | 404; state not conclusively established. |
| `/interaction-limits` | Empty configuration. |
| `/properties/values`, `/autolinks` | Empty. |

### Organization controls and limits

Organization metadata confirms required two-factor authentication, default
repository permission `read`, and members unable to create public repositories.
New-repository defaults enable dependency graph, secret scanning and push
protection; defaults do not prove existing-repository enforcement.

The following organization APIs returned 403 due to permissions/scopes:
`actions/permissions`, `actions/permissions/selected-actions`,
`actions/permissions/workflow`, `actions/runners`, `actions/runner-groups`,
`actions/secrets`, and `actions/variables`.

Organization rulesets and hooks returned 404 with respective `admin:org` and
`admin:org_hook` scope warnings. The organization security-settings route was
unavailable. Three code-security configurations were accessible; the attached
repository configuration is described above. Enterprise policies, audit-log
history, external registry/cloud policies, and all organization settings cannot
be certified from the available access. No authentication scopes were expanded.

## Staged implementation plan

### P0: Establish merge and authorization boundaries first

Owner: repository administrators, with organization security approval.

1. Set Actions default token permissions to read-only and disable workflow PR
   approval authority. Preserve approval for all external contributors.
2. Create active rules for `master`: PRs required, independent approval, code-owner
   approval, stale-review dismissal, approval of the latest reviewable push,
   conversation resolution, and required checks from the expected GitHub App.
   Block force pushes and deletion. Minimize and document bypass actors.
3. Add CODEOWNERS using validated teams with explicit write access. Cover all
   files by default and assign security-sensitive paths to the responsible
   maintainers: `.github/**`, CODEOWNERS itself, dependency/tool manifests,
   Makefile/scripts, container/ko inputs and charts. GitHub requires visible
   teams for team ownership; verify actual review requests and enforcement.
   Multiple owners on a pattern do not require approval from every listed owner.
4. Protect release tags against unauthorized creation, updates and deletion.
   Authorize only the chosen release process, not all repository writers.
5. Review effective admins/writers and inherited team access. Retain a small
   justified admin group and documented emergency access.
6. Keep publishers disabled until source protection, destination ownership,
   workflow registration and credential scope are verified.

Acceptance: ordinary writers cannot directly push protected source, bypass review
or create release tags. Workflow/CODEOWNERS changes require an eligible owner's
independent approval. APIs show read-only defaults and PR approvals disabled.
Do not require nonexistent check names: register and test safe CI before making
checks mandatory, then complete the ruleset.

### P1: Enable useful, isolated external-fork CI

Owner: CI maintainers.

1. Resolve why live YAML workflows are absent from the Actions inventory; verify
   fork Actions enablement and allowlist compatibility without relaxing policy
   globally.
2. Keep PR tests on `pull_request` with explicit `contents: read`, no secrets,
   `persist-credentials: false`, and ephemeral GitHub-hosted runners.
3. Add unit-test coverage to PR validation alongside lint/chart checks where
   missing. Keep Cloudflare credentialed end-to-end tests separate.
4. Do not run public PR code on self-hosted/persistent runners or internal
   networks. Obtain organization-admin confirmation of runner-group access.
5. Never use `pull_request_target` to execute PR head code. Do not let privileged
   `workflow_run` jobs execute untrusted artifacts or restore attacker-writable
   caches. A separate job alone is not a complete trust boundary.
6. Add explicit job timeouts and cancellation of superseded PR runs. Scope
   concurrency to each PR/workflow so untrusted runs cannot cancel releases.
   Limit matrix fan-out and large artifacts; review scheduled/push duplication.
7. Set Actions spending alerts and any available enforceable budgets for paid
   runners, storage and other billable products. Alerts are not a spending cap;
   confirm the actual enforcement behavior. Maintain an incident procedure for
   disabling abusive workflows and temporarily limiting interactions.

Acceptance: a controlled external-fork PR waits for approval, then completes
checks without credentials/write access/internal runners. Subsequent updates
behave as intended under the approval policy. Documentation-only PRs do not
leave required checks permanently pending. Superseded runs cancel safely;
timeouts and billing controls are verified without deliberate resource abuse.

### P1: Restrict publishing to reviewed source

Owner: release maintainers and registry/cloud identity administrators.

1. Confirm intended GHCR, Docker Hub, chart and binary destinations. Replace or
   explicitly authorize inherited upstream destinations before adding credentials.
2. Publish only from reviewed protected history. Validate the release commit
   against approved source; a matching tag name is not proof of review.
   Ensure the publisher workflow definition also comes from protected source.
3. Set explicit permissions per job. Grant content/release write only to the
   relevant publisher; package write only to image jobs; OIDC and attestation
   authority only where consumed. Remove unused master-publisher OIDC.
4. Add a release environment with independent reviewers, prevention of
   self-review and appropriate ref restrictions. Environment approval is an
   additional gate, not isolation of untrusted code.
5. Prefer short-lived credentials and repository-scoped registry access.
   Verify OIDC audience, immutable repository identity, approved workflow/ref/
   environment, and token lifetime at the relying party.
6. Carry the exact image digest from build through distribution and attestation;
   do not re-resolve mutable image tags between jobs.
7. Enable immutable releases where appropriate; publish SBOMs and make
   provenance verification part of release acceptance.

Acceptance: fork PRs, arbitrary branches, unauthorized tags and unreviewed commits
cannot publish. Approvers can identify the exact source SHA and artifact digest.
The distributed image digest matches the built/attested digest.

### P1-P2: Reduce supply-chain and integration exposure

Owner: CI maintainers and organization administrators.

1. Require reviewed full-SHA action references; pin execution containers by digest.
   Audit transitive downloads: a pinned wrapper does not pin downloaded tools.
2. Narrow the 45-pattern action allowlist to actual needs; reconsider blanket
   verified-creator permission. Verify allowlist compatibility with SHA pinning.
3. Replace branch references and `latest` tool versions. Add Dependabot Go-module
   maintenance and an explicit ko base-image update process.
4. Review all applicable GitHub Apps, including the three selected-repository
   installations. Prefer selected-repository access, minimum permissions, and a
   documented owner. Reduce contents/workflows/administration write grants.
5. Verify organization secret availability, webhook scope/TLS, runner groups,
   package access inheritance and external identity trust using authorized admins.
6. Use separate release caches or disable them; never promote PR-controlled
   executable content directly into privileged jobs.

Acceptance: execution dependencies are immutable and updated through reviewed
PRs; every privileged integration has justified repository scope and permissions.
No publisher consumes an untrusted cache or executable PR artifact.

### P2: Detection, disclosure and ongoing operation

Owner: security maintainers.

1. Enable Go code scanning and dependency review. Confirm successful analyses,
   not just configuration flags. Establish enforceable alert ownership and
   response expectations.
2. Preserve Dependabot, secret scanning and push protection. Review bypass
   permissions; consider enforcing the attached organization security policy.
3. Enable private vulnerability reporting and document incident handling for
   token theft, malicious actions and registry/package compromise.
4. Choose log/artifact retention for investigation needs and cost; 90 days is not
   inherently unsafe. Do not shorten retention without considering incident
   detection and response windows.
5. Disable unused wiki/projects if unnecessary. Keep public PRs open; use
   interaction limits only as a temporary response to actual abuse.
6. Review admin/bypass actors, Apps, runner exposure, OIDC, credentials and release
   controls periodically and after workflow or organization policy changes.

Acceptance: scanning analyses exist for protected source, private reporting works,
and maintainers have a tested revocation/containment procedure.

## Public fork constraints

Public source can be copied and forked; `allow_forking` is not a write-access
control. Outside PR authors need no collaborator role. The right boundary is
who can merge, execute with privilege, and publish, not whether a PR may be opened.

This fork shares the public upstream network. Never place confidential content
in commits, even temporarily; deleting a branch is not reliable erasure from a
public fork network. Rotate leaked credentials and follow GitHub's removal
process if exposure occurs.

## Verification blockers before sign-off

- Explain workflow registration/enablement and test the effective action policy.
- Obtain authorized organization runner/group and inherited secret inventories.
- Determine selected-repository App applicability and organization webhook scope.
- Inspect package linkage, access, visibility and deletion/restoration controls.
- Verify external registry/cloud OIDC conditions, not just GitHub's subject format.
- Confirm organization/enterprise enforcement, emergency access and bypasses.
- Validate CODEOWNERS teams and required checks with a controlled fork PR and
  representative workflow-change PR; do not test by exposing actual secrets.

Until these are resolved, the review is a verified repository-level baseline
with explicit organization/external-policy limitations, not certification that
the entire GitHub environment is hardened.
