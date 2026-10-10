# Dynamo PR revival and new work — status as of 2026-10-09

Upstream `ai-dynamo/dynamo` main at `22698fe0` (docs: increment release docs to v1.5.1, #15785).
Everything below was checked against that commit and the public GitHub pages, then re-verified
by a 33-agent adversarial pass (two review lenses over the fix, two refuters per finding, four
fact-checkers over the claims in this file, one completeness critic). Corrections from that pass
are folded in. I could not read the hidden reviewer comments on the two DEPs (GitHub API 403), so
the DEP drafts are based on the DEP bodies and the current operator code.

## 1. The three unmerged PRs

| PR | Issue | PR state | Issue state | Verdict |
|---|---|---|---|---|
| #8848 webhook cert commonName | #8736 | Closed by you, Jun 12 | Closed | Fixed differently by a maintainer (#8776 per the GitHub page; main truncates the CN). Nothing to do. |
| #8770 orphaned workloads on multinode toggle | #4749 | Closed by julienmancuso, Jul 13, no comment | Closed | The toggle is rejected at admission on both DGD and DCD updates since Jul 9 (#11479). Do not resubmit. |
| #8898 profiler name-length rule | #8480 | Auto-closed by stale bot, Jun 8 | Closed | Issue fixed by its reporter (#9848). Operator rule changed shape; the profiler check now diverges three ways. Ask before resubmitting (draft below). |

What the review threads actually contained (so nothing from reviewers is dropped):

- **#8848**: no human review. Bots only (copy-pr-bot validation notice, CodeRabbit "no actionable
  comments"). You noted the pre-commit failure was pre-existing on main, pinged tmonty12 on
  May 30, and closed it on Jun 12 as superseded.
- **#8770**: no human review. CodeRabbit left three nitpicks (a NotFound-gap test, use
  `metav1.IsControlledBy`, a golangci config error); you applied the `IsControlledBy` change on
  Apr 30 and argued the gap case was already covered, and CodeRabbit agreed. Stale-labelled
  May 30, label removed Jun 2 after you rebased and pinged tmonty12. Closed Jul 13 by
  julienmancuso with no comment.
- **#8898**: no human review. CodeRabbit's one nit (module-level imports in the test) you declined
  with a rationale and CodeRabbit accepted. Stale Jun 2, auto-closed Jun 8. Your Jun 12 comment
  asked NVIDIA reviewers for attention; nobody answered.

### #8848 → superseded, done

`deploy/helm/charts/platform/components/operator/templates/webhook-certificates.yaml` on main
renders `commonName: <fullname | trunc 56 | trimSuffix "-">-webhook` (and `-root-ca`). The
attribution to #8776 comes from the GitHub page, not from git (upstream history before late June
is one squash import). No action.

### #8770 → obsolete, note and move on

Why #4749 closed: the project took the "disallow it and validate" option the issue author raised,
and did so earlier than it looks:

- The DGD webhook already rejected flipping a service between single-node and multinode
  ("cannot change node topology ... after creation") in the base #8770 was rebased onto on
  May 30. You can see it in the PR's own tree: `git show 096abf72c:deploy/operator/internal/webhook/validation/dynamographdeployment.go` lines 173-187.
- The v1beta1 webhook port (#10934, Jun 29) and the structural-validation refactor (#11090,
  Jul 6) kept it, and #11479 (Jul 9) extended it to direct DynamoComponentDeployment updates.
  #8770 was closed four days later.
- `multinode.nodeCount` immutability came with #14477; #14419 (Sep 8) only restructured the
  block. Today the rule lives in `shared_v1beta1.go` around line 1075.

With the toggle impossible through any spec update, the orphan cleanup in #8770 has nothing to
clean up. The only remaining ways to flip a DCD between Deployment and LWS are an operator
restart with the LWS feature gate changed, or a bypassed webhook; neither is what #4749 reported.

Optional one-liner to leave on #8770 so the thread ends cleanly (not required):

> Closing the loop: the single-node/multinode shape is now immutable at admission for both DGD
> and DCD updates (#11090, #11479), so the cleanup path here is no longer reachable. Thanks for
> the earlier CodeRabbit round; nothing further to do on this one.

### #8898 → stale as written; ask on #8480 first

What changed on main since April:

- #9848 (Jonathan Tong, the issue reporter) fixed #8480 by shortening the vLLM worker names.
- The operator no longer rejects long DGD names. `PCSNameForDGD` in
  `deploy/operator/internal/dynamo/graph.go` truncates the PodCliqueSet name to
  `45 - maxComponentBudget` (minimum 8) with a 4-char hash suffix.
- The webhook rule is now `len(pcsName) + ComponentNameBudget(component) > 45`, applied only on
  the Grove pathway and only to components the operator renders itself
  (`webhook/validation/dynamographdeployment.go` ~line 400). `ComponentNameBudget`
  (`internal/dynamo/grove.go:142`) counts the component name twice plus the role suffix for
  multinode and GMS components (budget `2L+4` or `2L+6`), and once otherwise.
- The profiler moved to v1beta1 and iterates `spec.components` (#12506), so #8898's
  `services.items()` loop no longer applies.
- `_validate_dgd_service_name_lengths` in `components/src/dynamo/profiler/profile_sla.py` still
  computes `len(dgd_name) + len(component_name)` and runs unconditionally in the DGDR path.

So the profiler check now diverges from the operator three ways (numbers computed with the real
Go functions):

1. False negative, narrow: a multinode component named 17+ characters (GMS 16+) passes the
   profiler but fails the webhook. DGDR `my-dgdr`, component `VllmPrefillWorker` with
   `nodeCount: 2`: profiler sees 11+17=28, operator budget is 38, PCS truncates to 8, combined
   46 > 45, rejected. With the profiler's default names (`Frontend`, `prefill`, `decode`) this
   does not trigger, so it needs a user-renamed component.
2. False positive, real: DGDR `llama-3-70b-disagg-production-canary-01` gives a 43-char DGD name,
   the profiler raises (43+8=51) while the operator truncates the PCS name to 37 chars and
   accepts.
3. Pathway blindness: on the LWS pathway the operator applies no combined-length check at all,
   but the profiler still fails the request at 45.

A faithful profiler-side port therefore needs the workload provider and the delegated-component
rules, not just the formula. That is the argument for routing the check through the shared Go
engine the profiler already calls for DGD generation (#11187), or dropping the Python check.

Draft comment for #8480 (post before touching the branch):

> Following up on #8898, which the stale bot closed in June. #9848 fixed the vLLM names, and on
> current main the operator truncates the PodCliqueSet name (`PCSNameForDGD`) and validates
> `len(pcsName) + ComponentNameBudget(component)` on the Grove pathway only.
>
> The profiler's `_validate_dgd_service_name_lengths` still does `len(dgd_name) +
> len(component_name)` unconditionally, so it now disagrees with the webhook in three ways: it
> misses the multinode/GMS role budget (the shape this issue hit), it rejects long DGDR names the
> operator would accept after truncation (a 43-char DGD name fails the profiler at 51 but passes
> the webhook with a 37-char PCS name), and it fires on LWS deployments where the operator has no
> 45-char rule at all.
>
> Is a profiler-side check still wanted? A faithful mirror needs the workload provider and
> delegated-component rules, so I would rather route it through the shared Go engine from
> #11187 than re-implement it in Python. If you would prefer to drop the Python check and lean
> on the webhook, that is a smaller change and I can do that instead.

## 2. #13254 — already done upstream

Closed by #13486 (ayaangazali, merged Sep 10 by julienmancuso). `ValidateTerminatingUpdate` runs
only the metadata rules on terminating DGDs, with handler tests for the replace, remove,
unsupported-value, and finalizer-only cases. Nothing to pick up.

## 3. #13239 — revised on Oct 10: review #13575, open the enum-sync PR instead

The issue thread, read through the GitHub API on Oct 10, changed the plan:

- **An open PR already covers the fallback fix: #13575** (saiyambharara, opened Aug 19,
  "fix(operator): stop writing unrecognized gpu models into gpusku"). Same shape as our fix
  branch, plus a `SupportedGPUSKUs()` helper and more tests. julienmancuso reviewed on Aug 20
  (make the remediation match the discovery contract) and yankay on Sep 1 (say the DGDR must be
  recreated); both threads are resolved. Lightweight CI was green on Sep 1. The author pinged on
  Aug 29 and Sep 15, naming hhzhang16, sttts, jh-nv, and tmonty12 (CODEOWNERS for
  `discovery.go`), with no reply since. Five bot threads remain, all minor (duplicate standalone
  test, missing `t.Log` headings, assert the full error contract, mixed-cluster follow-up).
- **#13575 no longer merges.** `git merge-tree` against main `a83f4796b` conflicts in
  `deploy/operator/internal/gpu/discovery.go` because of the SKU additions in #15216, #15711,
  and #15703. It also has no docs change, which `deploy/operator/AGENTS.md` asks for.
- **Do not open our fix branch as a PR.** It stays on the fork as a fallback:
  https://github.com/kruthiwusirika5/dynamo/tree/fix/dgdr-gpusku-unknown-product-13239
- **The part of #13239 nobody has a PR for is the enum sync.** The reporter's follow-up says
  AIConfigurator ships `rtx_pro_6000_server`; vvagias reproduced the same failure for
  `NVIDIA-GB300`. AIC has shipped both systems since v0.8.0 (May 2026), well before the AIC
  version Dynamo adopted in July (#11986). The third missing AIC name, `b60`, is an Intel-class
  entry (SYCL and oneCCL in its system file) and is out of scope for `nvidia.com/gpu.product`
  discovery.

### The enum-sync PR (implemented, pushed, ready to open)

Branch https://github.com/kruthiwusirika5/dynamo/tree/fix/dgdr-gpusku-rtx-pro-6000-gb300-13239
(HEAD `8f3d1a958`, one commit on upstream main `b208989fb`), authored and DCO-signed as you,
modeled on sttts's B300 commit (#15216):

- `GPUSKUType`: add `gb300` and `rtx_pro_6000_server` (both enum markers, constants, CRD,
  Python types, generated API references).
- `InferHardwareSystem`: `NVIDIA GB300` and `NVIDIA-GB300` infer `gb300` (rule placed before
  B300, whose guard already refuses GB300). `NVIDIA RTX PRO 6000 Blackwell Server Edition` and
  the GPU Feature Discovery form infer `rtx_pro_6000_server`; the rule is a bounded token plus a
  required `SERVER` marker, so the Workstation and Max-Q editions stay unrecognized instead of
  silently profiling against the server card. Done as a generic `requires` field on `gpuRule`
  rather than another one-off guard.
- Tests: eight inference cases (both products, both label forms, two negative editions,
  `RTX PRO 60000`), two enrichment table rows, two admission cases.
- Docs: SKU-format paragraph on the curated DGDR reference page; the two generated references
  were regenerated with `make generate-api-docs`.

Validation on upstream main with Go 1.26.8:

```
go build ./... && go vet (gpu, controller, webhook/validation, api)   ok
gofmt -s -l                                                           clean
golangci-lint v1.62.2 on gpu, api, webhook/validation, controller     0 issues
go test ./internal/gpu/                                               ok
go test ./internal/controller/ -run TestEnrichHardwareFromDiscovery   ok
go test ./internal/webhook/validation/ -run TestDynamoGraphDeploymentRequestValidator (envtest)   ok
make check (regenerates everything, fails on drift)                   clean
```

Patch file next to this note: `0001-fix-operator-support-RTX-PRO-6000-Server-and-GB300-i.patch`.

Commands to sign, push, and open it (adjust the first line):

```bash
cd ~/dynamo
git fetch origin fix/dgdr-gpusku-rtx-pro-6000-gb300-13239
git checkout -B fix/dgdr-gpusku-rtx-pro-6000-gb300-13239 origin/fix/dgdr-gpusku-rtx-pro-6000-gb300-13239
git commit --amend -S --no-edit
git push --force-with-lease origin fix/dgdr-gpusku-rtx-pro-6000-gb300-13239
gh pr create --repo ai-dynamo/dynamo --base main \
  --head kruthiwusirika5:fix/dgdr-gpusku-rtx-pro-6000-gb300-13239 \
  --title "fix(operator): support RTX PRO 6000 Server and GB300 in DGDR GPU discovery" \
  --body-file /tmp/pr-body.md
```

PR description (`/tmp/pr-body.md`):

```
## Overview:

AIConfigurator has shipped `rtx_pro_6000_server` and `gb300` system definitions since v0.8.0,
but the DGDR `gpuSku` enum does not accept either name and `InferHardwareSystem` has no rule
for the products, so clusters with these GPUs cannot use hardware auto-discovery at all
(#13239 reports the RTX PRO 6000 Blackwell Server Edition; GB300 is reproduced in the same
thread). This is the enum-sync half of #13239. The raw-string fallback itself is handled by
#13575; the two changes are independent.

## Details:

- `GPUSKUType`: add `gb300` and `rtx_pro_6000_server` (enum markers, constants, regenerated
  CRD, Python DGDR types, and API references).
- `InferHardwareSystem`: `NVIDIA GB300` / `NVIDIA-GB300` infer `gb300`; the rule sits before
  B300, whose guard already refuses GB300. `NVIDIA RTX PRO 6000 Blackwell Server Edition` and
  its GPU Feature Discovery label form infer `rtx_pro_6000_server`. The RTX rule is a bounded
  token plus a required `SERVER` marker, so the Workstation and Max-Q editions, which have no
  profiler data, stay unrecognized rather than silently profiling against the server card.
  Implemented as a generic `requires` field on `gpuRule`.
- Tests: inference cases for both products and both label forms, two negative editions and
  `RTX PRO 60000`; enrichment table rows; admission cases.
- Docs: SKU-format paragraph on the curated DGDR reference page; generated references updated
  with `make generate-api-docs`. `make check` is clean.

Out of scope: `b60` (an Intel-class AIC entry, not an `nvidia.com/gpu.product` value) and the
seven enum values that have no AIC system.

## Where should the reviewer start?

`deploy/operator/internal/gpu/discovery.go` (`gpuRules` and the `requires` field), then
`TestInferHardwareSystem` in `discovery_test.go`.

## Related Issues

- Relates to #13239
```

### Comments to post, in this order, after the PR exists (replace NNNN)

On **#13575** (a PR comment, as a fellow contributor):

> I hit the same bug independently and had a near-identical patch ready, so rather than open a
> competing PR I'd like to help get this one in. Three things from checking it against today's
> main:
>
> 1. It no longer merges: `discovery.go` conflicts with the SKU additions from #15216, #15711
>    and #15703. The rebase should be mechanical, since `SupportedGPUSKUs()` sits after
>    `InferHardwareSystem`.
> 2. `deploy/operator/AGENTS.md` asks for the curated `dynamo-graph-deployment-request.mdx` page
>    to be updated in the same PR when controller behavior changes, and the SKU-format section
>    currently describes only the happy path. I have a paragraph for it plus a Troubleshoot row
>    for `auto-deploy-with-dgdr.md` on
>    https://github.com/kruthiwusirika5/dynamo/commit/df3260954 if you want to cherry-pick them.
> 3. For the "closest supported value" problem yankay and vvagias raised: #NNNN adds
>    `rtx_pro_6000_server` and `gb300` to the enum and to `InferHardwareSystem`, so the two
>    products reported in #13239 stop reaching this error path. The PRs are independent.
>
> On the bot nits, folding the standalone unknown-GPU test into the table as a node-label row
> would close two of them. Otherwise this looks ready once rebased.

On **#13239** (an issue comment):

> Two updates after reading the thread: #13575 already covers the fallback write, so I'm
> reviewing that one rather than duplicating it. For the enum-sync gap @Kennyezzz found,
> AIConfigurator has shipped `rtx_pro_6000_server` and `gb300` since v0.8.0, so #NNNN adds
> both to the CRD enum and to `InferHardwareSystem` for the DCGM and GPU Feature Discovery
> product forms, Server edition only for the RTX card (the Workstation and Max-Q editions have
> no profiler data). `b60` is an Intel-class entry in AIC and out of scope for
> `nvidia.com/gpu.product` discovery. @vvagias that covers the `NVIDIA-GB300` label you
> reproduced.

If I post either comment from a session, it carries a "Generated by Claude Code" footer. Posting
them yourself avoids that.

## 4. DEP comments

Both drafts were trimmed to three points after the fact-check; the held-back points are listed
under each with their corrections, in case the thread goes that way.

### #13255 — Transactional replica fencing for power-aware planning

> Reading this from the multinode/LWS side (I did the native LWS scaling refactor in #5468).
> The M1 scope is right to exclude LWS and Grove, and the status subtree looks reusable for them
> later. Three things I would pin down before M1 lands so the follow-ups do not change the API:
>
> 1. **Neither controller observes a status-only commit today.** The DGDSA controller watches
>    itself with `GenerationChangedPredicate` and re-enqueues on a DGD update only when
>    `componentsEqual` sees a change in component names or replica counts
>    (`dynamographdeploymentscalingadapter_controller.go`, `SetupWithManager`). The DGD
>    controller's own primary predicate (`primary_resource_predicate.go`) fires on generation,
>    deletion, annotation, and label changes only. A commit written to `DGD.status.power` wakes
>    neither unless the DEP adds a predicate or an explicit requeue. Related: today
>    `status.replicas` is copied from `spec.replicas` on every adapter reconcile, so the
>    proposed `observedGeneration` is the right addition; it just needs a watch that fires.
>
> 2. **"Return and requeue, then reread" needs an observation guard, not timing.** The cached
>    client can serve the pre-commit DGD on the next reconcile. I would gate the mirror step on
>    observing the committed status (its `observedGeneration`, or the status write's
>    resourceVersion). The DGDR controller's pattern is the nearest precedent: after its own
>    write it returns without requeue and lets the watch event for that write drive the next
>    reconcile (`createProfilingJob`'s `waitForObservation`).
>
> 3. **Keep the grant key opaque so LWS can use cohort epochs.** The operator sets no LWS
>    `restartPolicy`, so groups get the default `RecreateGroupOnPodRestart`: any pod or
>    container restart in a group recreates the whole group, and every Pod UID in it changes
>    (leader included) while the group index stays stable. Per-Pod-UID grants would be revoked
>    and re-issued for the entire group on every restart, which is why the cohort-epoch
>    alternative is the right one for LWS. If `Grants` is documented as "keyed by an opaque
>    grant id, Pod UID in M1" rather than "keyed by Pod UID", LWS can key by
>    `(lws UID, group index, epoch)` later without an API change.
>
> Happy to draft the LWS section of the follow-up DEP once M1 settles.

Held back (corrected versions):

- HPA: the HPA controller derives current replicas from the Scale subresource's
  `spec.replicas`, not `status.replicas`, so a lagging `status.replicas` does not make it
  thrash. The point reduces to "`PowerReady=False/RequestPending` has to be the operator-facing
  signal because HPA's `AbleToScale` stays True". Low value; skip unless asked.
- Deletion blocked on terminal observation: ask that the `PowerReady` message name the blocking
  Pod UIDs. Sound, but it is about the DEP as drafted (today's DGD finalizer only deletes
  auto-checkpoints), so phrase it that way if used.

### #11673 — KV Cache Controller (draft DEP)

> Commenting from the operator/deployment side, since the DGD will have to render whatever
> process topology this ends up with.
>
> 1. **Packaging of the memory-owner side process on Kubernetes.** The hot-start design only
>    works if the G2 pool outlives the engine container. Three packagings behave differently:
>    - a second container in the same Pod sharing a memory-medium emptyDir survives engine
>      container restarts but not Pod replacement;
>    - a native sidecar (init container with `restartPolicy: Always`, the mechanism #14927 uses
>      for the Dynamo runtime container) additionally starts before and outlives the engine
>      container, which matches the memory-owner lifecycle; for single-node workers the operator
>      can carry it today as a user-declared init container plus a mount of the existing
>      shared-memory volume, with no new RBAC or host access (native-sidecar mode currently
>      rejects multinode);
>    - a node-level DaemonSet with hostIPC/hostPath survives Pod replacement but makes ownership
>      node-scoped across Pods.
>    The DEP should pick one for the first implementation.
>
> 2. **Memory accounting for the pool.** tmpfs pages are charged to whichever container first
>    faults them and, hierarchically, to the Pod; a memory-medium emptyDir counts toward the Pod
>    limit and is bounded by `sizeLimit`, and OOM-killing the touching container does not free
>    the pages. The operator already sizes `/dev/shm` through `spec.components[].sharedMemorySize`
>    (default 8Gi), but nothing couples that size to any container's memory limit. Please state
>    whether the pool lives in that volume and which process faults it in (ideally the owner
>    pre-faults it), so the operator can reflect the pool in the right container's limit. Also
>    whether hugepages are expected.
>
> 3. **Inventory scope for multinode engines.** The router keys KV state per
>    `(worker_id, dp_rank)` today (`KvSourceKey`). For a TP group spanning Pods on different
>    nodes, is `BlockInventoryEntry` published per rank or per group, and which rank's peer
>    endpoint does a hint carry? Either is fine, but "controllers are identical from one node to
>    a thousand" depends on the answer.

Held back (corrected versions):

- Fence token: Pod UID fences Pod replacement but not an in-Pod engine restart (both engine
  incarnations see the same Pod UID), so the fence must be per attach, like the router's
  per-incarnation `publisher_id`. Phrase as a question if used.
- Scale-down/up relocation: the router already models source incarnations (`KvSourceId.publisher_id`)
  and resets a rank's state before a new source activates, so the ask becomes "reuse that
  identity rather than adding a parallel generation".

## 5. GitHub access: what happened and what to do next

This session was bound to `kruthiwusirika45/dynamo`, where the Claude GitHub App is not
installed, so every push returned 403. You reconnected GitHub as kruthiwusirika5 (app installed,
all repositories), but a session cannot attach a second repository with the same name. A sibling
session started on `kruthiwusirika5/dynamo` applied the patches and pushed the branches above. For
future work on Dynamo, start sessions with `kruthiwusirika5/dynamo` selected as the source.
