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

## 3. #13239 — fix implemented, pushed, ready to open

Issue is open, assigned to hhzhang16 and sttts, no linked PR. Stefan Schimanski's recent commits
(#15216 B300, #15711 GB200, #15703 VR200) add SKU constants, enum entries, CRD regeneration, and
docs, but none touches the raw-string fallback; `git log -S` on that line shows nothing after the
June squash import. Because the issue is assigned, post this on #13239 first:

> I have a small PR ready for the fallback part of this: drop the raw `nvidia.com/gpu.product`
> write in `enrichHardwareFromDiscovery`, leave `gpuSku` unset, and fail with an error that
> names the product and says to recreate the request with the hardware fields set. Regression
> test uses the RTX PRO 6000 product string on both the DCGM and node-label paths, plus a line
> in the DGDR reference and troubleshooting docs. It does not touch the enum/AIC divergence,
> which I think deserves its own issue. Happy to defer if either of you is already on it;
> otherwise I will open it today.

Branch, pushed from your kruthiwusirika5 fork by a sibling session (this session could not push):

- https://github.com/kruthiwusirika5/dynamo/tree/fix/dgdr-gpusku-unknown-product-13239
- based on upstream `22698fe0`; one commit, authored and DCO-signed as
  `Sai Kruthi Wusirika <kruthiwusirika@gmail.com>`. Sign it with your key before opening the PR
  (`git commit --amend -S --no-edit`); upstream wants Verified commits on fork PRs for automatic
  `/ok to test`.
- patch file next to this note: `0001-fix-operator-reject-unrecognized-GPU-products-in-DGD.patch`.

What the commit does (after the adversarial pass; two things changed from the first draft):

- `enrichHardwareFromDiscovery`: no raw fallback. The error names the product and says to
  recreate the DGDR with `spec.hardware.{gpuSku,vramMb,numGpusPerNode,totalGpus}` set. The
  longer advice is deliberate: the request lands in the terminal `Failed` phase (pre-existing,
  documented behavior, editing the spec in place does nothing), and discovery filtered by a SKU
  cannot select nodes of an unsupported product, so setting only `gpuSku` fails again.
- The `GetGPUDiscoveryFailureReason` case from the first draft is gone. Its only production
  caller classifies the DCGM error before node-label fallback, so the case was dead code and
  the commit message claim about logging was false.
- Tests: the DCGM table case now uses the DCGM form of the product (with spaces); the node-label
  regression test uses the GPU Feature Discovery form (`NVIDIA-RTX-PRO-6000-Blackwell-Server-Edition`).
  The `KeepsUserProvidedGPUSKU` test from the first draft was removed because it never reached
  the `gpuSku` block (same flaw as the upstream test it replaced).
- Docs: one paragraph in the reference page's SKU format section (the curated page
  `deploy/operator/AGENTS.md` says to update in the same PR) and one Troubleshoot row.
- The message surfaces on the `Profiling` condition (reason `JobCreationFailed`), the
  `Succeeded` condition, and a `ProfilingJobFailed` warning event.

Disclose in the PR: the removed fallback was intentional upstream ("for GPUs not in the AIC
support matrix, the raw GFD product name is used as a fallback") but could never succeed end to
end, because the profiler's own Python validation also accepts only enum values. The fix changes
the error, not the outcome.

Validation on upstream main with Go 1.26.8:

```
go build ./...                                                        ok
go vet ./internal/controller/                                         ok
gofmt -s -l <changed files>                                           clean
golangci-lint v1.62.2 (the version CI's Dockerfile installs)          0 issues on ./internal/controller/...
golangci-lint v1.64.8 (Makefile pin)                                  0 issues
go test ./internal/controller/ -run 'TestEnrichHardwareFromDiscovery' PASS
go test ./internal/gpu/                                               ok
go test ./internal/controller/ (full package incl. 75-spec envtest)   ok (83s)
```

PR description (fills the repo template):

```
fix(operator): reject unrecognized GPU products in DGDR hardware discovery

## Overview:

DGDR hardware auto-discovery wrote the raw `nvidia.com/gpu.product` string into
`spec.hardware.gpuSku` when `InferHardwareSystem` did not recognize the GPU. That value is
not in the `gpuSku` CRD enum, so the DGDR update was rejected with an opaque
`Unsupported value` error and discovery failed without saying why. The fallback could never
succeed end to end: the profiler's own DGDR validation accepts only enum values too, so this
change replaces an opaque failure with an actionable one.

## Details:

- `enrichHardwareFromDiscovery`: remove the raw-string fallback. When neither discovery nor
  `InferHardwareSystem` yields a SKU, leave `gpuSku` unset and return an error that names the
  product and tells the user to recreate the request with
  `spec.hardware.{gpuSku,vramMb,numGpusPerNode,totalGpus}` set. The request enters the
  terminal `Failed` phase as before; discovery filtered by a SKU cannot select nodes of an
  unsupported product, so all four fields are needed.
- Tests: the DCGM-path table case uses the DCGM form of the RTX PRO 6000 product; the old
  `FallsBackToModelForUnknownGPU` test (which set `gpuSku` by hand and never exercised the
  fallback) is replaced by `RejectsUnknownGPUProduct`, which drives node-label discovery with
  the GPU Feature Discovery form and checks that `gpuSku` stays empty.
- Docs: a paragraph in the DGDR reference SKU format section and a Troubleshoot row.

Out of scope: reconciling the CRD enum with AIConfigurator's systems list. I will open a
separate issue for that so this one can close.

## Where should the reviewer start?

`deploy/operator/internal/controller/dynamographdeploymentrequest_controller.go`, the
`if hw.GPUSKU == ""` block in `enrichHardwareFromDiscovery`, then
`enrich_hardware_test.go` `TestEnrichHardwareFromDiscovery_RejectsUnknownGPUProduct`.

## Related Issues

- Closes #13239
```

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
session started on `kruthiwusirika5/dynamo` applied the patch and pushed the branch above. For
future work on Dynamo, start sessions with `kruthiwusirika5/dynamo` selected as the source.
