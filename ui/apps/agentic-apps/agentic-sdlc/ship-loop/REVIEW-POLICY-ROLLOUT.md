# Quiet PR review policy: staged rollout and rollback

## Status and approval

This change is prepared for review. **Production deployment and saved reviewer
configuration migration are on hold until explicitly approved.** Merging source
or publishing an image does not authorize a production rollout.

The task and agent configuration is persisted independently of the app image.
Deploying the app does not migrate enabled reviewers automatically. Conversely,
editing a saved prompt does not deploy the new app dispatcher. Plan and verify
both changes together.

## Before a rollout

1. Select the approved environment and an isolated test repository. Inventory its
   app binding, native task and Dynamic Agent, including their owner and scope.
   App-created tasks must match their binding/repository metadata. Review
   pre-existing managed tasks separately; do not adopt an unrelated reviewer.
2. Snapshot the image digest and the non-secret configuration fields below in
   protected operator storage. Record owner/team, enabled state, model/tools,
   attached skills, trigger deduplication and binding activation time for later
   comparison. Keep signing keys and credential values in their existing stores.
3. Inventory active and pending runs. Existing runs retain their captured
   instructions; wait for them to finish or handle them through the owner's
   normal run controls before declaring the new policy effective.
4. Require passing CI and record the exact source revision. Build and pin its
   immutable app image digest in the environment's deployment configuration.
   Start with the approved test/canary environment.

## Saved configuration migration

Use the existing authorized CAIPE management path for each selected task/agent.
The source of truth is the policy generators, not copied production records:

| Target field | Proposed value |
| --- | --- |
| Native task `prompt` | `reviewPrompt(fullName, task.id)` from `src/lib/agentic-sdlc/review-policy.ts` |
| Native task `trigger.filter` | `reviewTriggerFilter(fullName)` from `src/lib/agentic-sdlc/review-eligibility.ts` |
| Dynamic Agent `system_prompt` | `reviewSystem(fullName)` from `src/lib/agentic-sdlc/review-policy.ts` |
| Dynamic Agent configuration version | Advance `updated_at` through the normal update path so cached runtimes detect the change |

The native filter must require the exact repository, event `pull_request`, action
`opened`/`synchronize`/`ready_for_review`, `pull_request.state` equal to `open`, and
`pull_request.draft` equal to boolean false. Native scalar filters encode that
last value as the string `"false"`. Preserve any additional custom restrictions.

For app-created reviewers, first confirm the saved prompt/system matches the
previous managed template. For a custom managed reviewer, review the diff and
retain its repository, model and other custom restrictions; incorporate
`REVIEW_PUBLICATION_POLICY` into both task and system instructions and update its
eligible-action wording. Do not replace a custom prompt wholesale.

Apply only the reviewed fields, guarded by their snapshotted values. If another
operator has changed them, stop and review the new diff. A reviewer already on
the intended version is a no-op. Preserve signing keys, credential references,
ownership, enabled state, model/tools, skills, delivery deduplication and the
binding's key, state and `enabled_at`. Disable/re-enable is not a migration: it
rotates the signing key and resets activation time.

Verify the effective runtime as well as database readback. Autonomous has an
in-process webhook-task registry updated by the normal management path; a raw
database edit alone does not prove that its live filter/prompt changed. Dynamic
Agents invalidates cached configurations using `updated_at`. Use the supported
scoped update/refresh path; do not restart shared production services just to
test this change.

## Canary acceptance, then production approval

1. Confirm the selected app runs the pinned new image and its saved reviewer
   fields match the reviewed migration. Confirm the effective native filters
   reject draft/closed payloads and accept non-draft/ready payloads. Fixture
   checks must not dispatch a run or post to a product PR.
2. In the approved test repository, verify draft pushes create no review run and
   changing to ready reaches the reviewer. Verify clean or incomplete-CI results
   remain in task history without a public report. A genuine new test finding
   should produce one concise COMMENT review; a later head retaining the same
   finding should not repeat it. Keep the run IDs and actual GitHub review URLs
   as evidence. Publishing behavior is an agent instruction, so filter tests
   alone do not establish findings-only publication.
3. Verify queued draft deliveries from before the upgrade are skipped by the
   new dispatcher. Preserve stored events and delivery IDs. A formerly non-draft
   snapshot can become stale: the reviewer must still recheck current GitHub
   head/open/draft state before writes.
4. Compare the preserved fields against the snapshot, and inspect pending work
   and errors. Record limitations, including any runs carrying older prompts.
5. Present the canary evidence, exact production image digest and per-reviewer
   configuration diff for explicit production approval. Only after approval,
   apply the same staged image/configuration changes to the selected production
   reviewers and repeat acceptance checks. Do not replay historical product PR
   events or post test reviews merely to populate the dashboard.

## Rollback

Restore the prior image and each task/agent's snapshotted prompt/filter through
the authorized deployment/management paths, limited to the approved environment.
Compare-and-set only fields still equal to the version being rolled back; leave
later operator edits for review. Advance the agent configuration timestamp so a
fresh runtime loads the restored prompt, and verify the webhook registry refresh.

Preserve keys, credentials, bindings, delivery IDs, deduplication records and
queued events. Do not use disable/re-enable or delete/recreate to restore a
reviewer. Inspect already-running work separately: changing saved configuration
does not rewrite a running agent's instructions. Record database readback and
effective runtime evidence separately; do not claim one proves the other.

Restoring the old policy also restores its per-head public-review behavior.
Document that consequence and keep the quiet-policy rollout on hold until its
next explicit approval.
