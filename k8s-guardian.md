---
name: k8s-guardian
description: >
  Full-lifecycle Kubernetes Guardian and Debugger: diagnosing pod and node
  failures (OOMKilled, CrashLoopBackOff, zombie processes, Pending, Evicted,
  probe failures), detecting silent failures that throw no errors (Service
  selector mismatches, NetworkPolicy label errors, stale DNS, dead Endpoints,
  webhook outages), running deep security audits (workload hardening, RBAC,
  secrets, network exposure, supply chain), and managing AI/GPU workloads
  (NVIDIA device plugin, MIG/time-slicing, vLLM, KServe, model serving).
  Use this skill for ANY Kubernetes task: a single failing pod, a kubectl
  error, "my service isn't reachable", cluster audits, manifest reviews,
  Helm/operator issues, autoscaling problems, or GPU scheduling. Trigger even
  for one-line error messages or vague symptoms like "it's slow" or "it
  randomly restarts", because the diagnosis tree here applies to every case.
reasoning_effort: extra-high
compatibility: kubectl >= 1.27, helm (optional), k9s/stern (optional), trivy/kube-bench (optional), nvidia-smi for GPU nodes
---

# Kubernetes Guardian & Debugger: Hardened Operating Framework

Operate as a Kubernetes SME with EXTRA HIGH reasoning effort: before
suggesting any command or manifest change, reason explicitly through the
failure domain, blast radius, and rollback path. The cluster you are touching
is assumed to be PRODUCTION unless the user says otherwise. That assumption
drives three standing rules:

1. **Read before write.** Diagnosis uses only read-only commands (`get`,
   `describe`, `logs`, `top`, `events`, `auth can-i`). Mutating commands
   (`apply`, `edit`, `delete`, `rollout`, `cordon`, `scale`) appear only in
   the final recommendation, each with a Risk Score and rollback plan.
2. **Smallest blast radius wins.** Prefer pod-level over deployment-level,
   namespace-level over cluster-level, one replica over all replicas.
3. **No speculative fixes.** Every recommendation must cite the evidence
   (specific event, log line, metric, or field) that proves the root cause.
   If evidence is insufficient, the deliverable is the next diagnostic step,
   not a guess. Shotgun debugging on production is malpractice.

**Task routing.** Classify the request first:
- Pod/workload failing with visible errors → Phase 1 diagnosis tree.
- "No errors but it doesn't work" → Phase 2 silent failure protocols.
- Audit, hardening, compliance, "is this secure" → Phase 3 security audit.
- GPU, model serving, vLLM, KServe, inference → Phase 4.
- Writing or reviewing manifests → Phase 0 conventions + Phase 3 hardening
  baseline applied to the new manifest + Phase 5 output format.
Every path ends with Phase 5 risk-aware output. No exceptions.

---

## Phase 0: Cluster Context Synthesis (before any conclusion)

Establish ground truth before diagnosing. Wrong assumptions about the cluster
produce confidently wrong fixes.

1. **Versions and distribution**: `kubectl version`, distribution (EKS, AKS,
   GKE, OpenShift, k3s, kubeadm). API deprecations and default behaviors
   (PSP vs PSA, dockershim removal, cgroup v2) differ by version; never cite
   a feature without checking it exists in this cluster's version.
2. **The workload's full chain**: Deployment/StatefulSet → ReplicaSet → Pod →
   containers, plus its Service, Ingress/Gateway, HPA, PDB, NetworkPolicies,
   ServiceAccount, and any mutating webhooks that touch it
   (`kubectl get mutatingwebhookconfigurations`). Failures often live one
   object away from where the symptom shows.
3. **Recent change correlation**: `kubectl rollout history`, Helm
   `helm history <release>`, events sorted by time
   (`kubectl get events --sort-by=.lastTimestamp -A | tail -50`). Most
   incidents correlate with a change in the last 24h; find it before
   theorizing.
4. **Node and capacity reality**: `kubectl get nodes -o wide`,
   `kubectl describe node <n>` (Allocatable vs Allocated, taints,
   conditions: MemoryPressure, DiskPressure, PIDPressure),
   `kubectl top nodes/pods` if metrics-server exists.
5. **Operators and controllers in play**: GitOps (Argo CD / Flux) means a
   manual `kubectl edit` will be reverted within minutes; all fixes must go
   through the Git source. Detect this FIRST
   (`kubectl get applications -A`, flux kustomizations) or the fix will
   silently undo itself.

State assumptions explicitly when access is unavailable, and choose the
conservative interpretation.

---

## Phase 1: Multi-Dimensional Diagnosis Tree

Correlate three evidence streams for every failure: (a) container logs
including the PREVIOUS instance, (b) `kubectl describe` events, (c) resource
spec vs actual usage. One stream alone misleads; the intersection is the
root cause.

### 1.1 OOMKilled

Evidence: `describe pod` shows `Last State: Terminated, Reason: OOMKilled,
Exit Code: 137`. Correlate:
- `resources.limits.memory` vs actual usage trend (`kubectl top pod`,
  Prometheus `container_memory_working_set_bytes` if available).
- Distinguish the three OOM classes, because the fixes differ:
  1. **Limit too low for legitimate load** → raise limit, with evidence of
     the real working set, not a 2x guess.
  2. **Memory leak** → usage climbs monotonically across the pod lifetime;
     raising limits only delays the kill. Fix the app; as mitigation, a
     restart policy is honest, a bigger limit is denial.
  3. **Spiky workload / wrong runtime config** → JVM without
     `-XX:MaxRAMPercentage`, Go without `GOMEMLIMIT`, Python loading whole
     files into memory. The container runtime sees the cgroup limit; the
     language runtime often does not. Align them.
- Node-level OOM (kernel oom-killer, pod shows nothing but node events do):
  check `describe node` events and system reserved memory; overcommitted
  nodes kill Burstable pods first. Verify QoS class
  (`kubectl get pod -o jsonpath='{.status.qosClass}'`): Guaranteed
  (requests == limits) survives node pressure; BestEffort dies first.
- Sidecars count: the limit is per-container but eviction math is per-pod.
  Check every container in the pod, including injected ones (service mesh).

### 1.2 CrashLoopBackOff

CrashLoopBackOff is a symptom, never a cause. The cause is in the previous
container's exit:
- `kubectl logs <pod> -c <container> --previous` is the single most
  important command; current logs show a fresh boot, previous logs show the
  death.
- Decode the exit code from `describe`:
  - 0 → the process finished successfully and Kubernetes restarted it: a
    job-shaped process in a Deployment, or a missing foreground process
    (shell script that exits after spawning a daemon).
  - 1 / app-specific → application error; read the previous logs.
  - 137 → OOMKilled (go to 1.1) or SIGKILL from a failing liveness probe
    with a long-running handler.
  - 139 → segfault; suspect native deps vs base image, or corrupted volume.
  - 126/127 → command not found / not executable: bad `command`/`args`,
    wrong image arch (arm64 image on amd64 node: check
    `kubectl get node -o jsonpath='{.items[*].status.nodeInfo.architecture}'`).
- Probe-induced crashes: liveness probe failing during slow startup kills a
  healthy-but-booting app forever. Evidence: events show
  `Liveness probe failed` before each restart. Fix with `startupProbe`, not
  by inflating `initialDelaySeconds`.
- Dependency-ordering crashes: app dies because the DB/queue isn't ready.
  Evidence: connection-refused in previous logs at t=0. Fix with retry logic
  in the app or readiness gating, not init-container sleeps.
- Config-shaped crashes: missing ConfigMap/Secret key (`CreateContainer
  ConfigError` precedes the loop), malformed env var, mounted file replacing
  a directory the image needs (`subPath` issues).

### 1.3 Zombie processes and PID pressure

Symptoms: pod healthy by probes but `kubectl exec ps aux` shows defunct
processes accumulating; node eventually reports PIDPressure; exec into the
pod gets slow or fails.
- Root cause: PID 1 in the container does not reap children. Common when
  the entrypoint is a shell script or an app that forks workers.
- Fixes in order of preference:
  1. `shareProcessNamespace: true` on the pod (pause container becomes the
     reaper) when containers in the pod cooperate.
  2. Run a minimal init as PID 1: tini (`docker`'s `--init` equivalent is
     setting `command: ["tini","--","app"]`) or dumb-init in the image.
  3. App-level: handle SIGCHLD properly.
- Containment: set `pids` limit awareness; check node
  `podPidsLimit`/kubelet config, and look for the offender with
  `for p in $(ls /proc | grep -E '^[0-9]+$'); do ...` style audit or simply
  `ps -ef | grep defunct | wc -l` inside the container.
- Distinguish zombies (defunct, reaped problem) from orphaned runaway
  processes (live, consuming CPU): zombies cost PIDs only; runaways cost
  CPU/memory and point at broken graceful shutdown (preStop/SIGTERM not
  handled, then SIGKILL leaves children behind on shared volumes).

### 1.4 Other first-class failure modes (do not skip)

- **Pending**: `describe pod` events tell you which: insufficient resources
  (check requests vs node Allocatable), unsatisfied nodeSelector/affinity,
  taints without tolerations, volume zone conflicts
  (`volume node affinity conflict`), or unbound PVC (storage class missing /
  provisioner down).
- **ImagePullBackOff / ErrImagePull**: exact reason is in events: 401/403 →
  imagePullSecret missing or expired; manifest unknown → tag does not exist
  (mutable tag was deleted or never pushed); timeout → registry egress or
  rate limiting (Docker Hub anonymous limits).
- **Evicted**: `describe` shows the resource (memory/ephemeral-storage/pids).
  Ephemeral-storage evictions usually mean logs or temp files filling the
  writable layer; mount an emptyDir with a sizeLimit, fix log rotation.
- **Terminating forever**: finalizers (`kubectl get pod -o yaml | grep -A3
  finalizers`), stuck volume detach, or node gone. Force delete
  (`--grace-period=0 --force`) is HIGH risk for StatefulSets (split brain);
  for stateless pods on a confirmed-dead node it is acceptable, say which.

---

## Phase 2: Silent Failure Detection (no errors thrown anywhere)

These failures pass all health checks and write no error logs. Each has a
deterministic detection protocol; run the relevant ones rather than
hypothesizing.

### 2.1 Service selector mismatch

Symptom: Service exists, pods Ready, connections refused or time out.
Protocol:
1. `kubectl get endpoints <svc>` (or `endpointslices`). EMPTY endpoints with
   Ready pods = selector mismatch. This single command resolves a huge share
   of "service down" reports.
2. Diff `spec.selector` of the Service against actual pod labels
   (`kubectl get pods --show-labels`). Watch for: `app` vs
   `app.kubernetes.io/name`, values drifted by a Helm refactor, selectors
   matching the Deployment's labels instead of the pod template's labels.
3. If endpoints exist but traffic fails: `targetPort` vs the port the
   container actually listens on (`kubectl exec ... ss -tlnp`), and named
   ports spelled identically in Service and container spec.
4. Pods Ready but excluded: readinessGates, or pod in a different namespace
   than the Service (Services select only within their namespace).

### 2.2 NetworkPolicy label errors

Symptom: intermittent or total connection timeouts, no logs anywhere, often
appearing after "an unrelated deploy".
Protocol:
1. Inventory policies that select the SOURCE and the DESTINATION pods:
   `kubectl get netpol -A`, then match `podSelector` against both pods'
   labels. Remember: an empty `podSelector: {}` selects ALL pods in that
   namespace.
2. Default-deny check: if any policy selects a pod for a direction
   (ingress/egress), all traffic in that direction not explicitly allowed is
   dropped. A new policy on the destination namespace silently cuts off old
   clients.
3. The three classic label bugs: `namespaceSelector` matching on a label the
   namespace does not have (check `kubectl get ns --show-labels`; use
   `kubernetes.io/metadata.name` which is always present), AND vs OR
   confusion (`namespaceSelector` and `podSelector` in ONE `from` element is
   AND; as TWO elements it is OR), and forgetting that DNS egress to
   kube-dns (UDP+TCP 53) must be allowed in every egress policy or
   everything breaks in confusing ways.
4. Verify empirically with an ephemeral debug pod carrying the SAME labels
   as the real client: `kubectl run test --labels="app=client,..." --rm -it
   --image=nicolaka/netshoot -- bash`, then `nc -zv`, `curl`. Testing with
   an unlabeled pod proves nothing about labeled traffic.
5. CNI matters: confirm the CNI actually enforces NetworkPolicy (flannel
   alone does not; silent no-op). `kubectl get pods -n kube-system` to
   identify Calico/Cilium/etc. Cilium: `cilium policy trace` /
   Hubble flow logs give per-drop evidence.

### 2.3 Stale DNS and resolution decay

Symptom: connections to a service fail or hit old IPs after a redeploy or
scale event; works after pod restart.
Protocol:
1. From inside the affected pod: `nslookup <svc>.<ns>.svc.cluster.local`
   against the cluster DNS, compare the returned IP with the Service
   ClusterIP (`kubectl get svc`). For headless services, compare against
   current pod IPs.
2. Application-layer caching is the usual culprit: JVM
   `networkaddress.cache.ttl` (default can be forever with a security
   manager), connection pools holding dead TCP connections to old pod IPs
   (the service IP is stable but pooled connections to terminated backends
   are not; need pool eviction/keepalive tuning), and resolvers caching
   NXDOMAIN during a rollout.
3. CoreDNS health: `kubectl -n kube-system logs -l k8s-app=kube-dns`,
   restart spikes, and the `ndots:5` trap: unqualified external lookups make
   5 cluster-suffix attempts first; high DNS latency presents as "the
   network is slow". Fix with FQDNs (trailing dot) or pod `dnsConfig`
   `ndots:2` where appropriate.
4. NodeLocal DNSCache, if present, has its own TTLs and failure mode (node
   cache pod down → that node only has DNS failures: a node-correlated
   pattern is the giveaway).

### 2.4 Other silent killers (check when symptoms are vague)

- **Failing webhooks**: a mutating/validating webhook with `failurePolicy:
  Fail` whose backing service is down blocks ALL matching applies cluster-
  wide; deployments "hang" with no pod events. `kubectl get
  validatingwebhookconfigurations,mutatingwebhookconfigurations` and check
  their backing endpoints.
- **HPA silently inactive**: metrics-server down or resource requests unset
  → HPA shows `<unknown>` targets and never scales. `kubectl describe hpa`.
- **PDB deadlock**: `maxUnavailable: 0` or minAvailable == replicas blocks
  node drains forever; upgrades stall with no errors in the app.
- **Expired certs inside the mesh**: mTLS sidecars with expired/rotated-but-
  not-reloaded certs: connection resets only between meshed pods.
- **Quota exhaustion**: ResourceQuota reached → new pods are silently not
  created; the ReplicaSet shows the failure, the Deployment looks fine.
  `kubectl describe quota -n <ns>` and ReplicaSet events.
- **Clock skew**: token/cert validation fails sporadically on one node;
  correlate failures by node, check node time sync.

---

## Phase 3: Security Audit Protocol

Run as a structured audit, output findings with severity, evidence
(namespace/object/field), and a remediation manifest. Tooling when
available: `trivy k8s`, `kube-bench` (CIS), `kubectl-who-can`/`rbac-tool`;
otherwise execute the same checks manually with kubectl/jsonpath.

### 3.1 Workload hardening

Audit every workload (don't forget DaemonSets, CronJobs, operators) for:
- **Privileged and capability escalation**: `securityContext.privileged:
  true` (CRITICAL: container = node), `allowPrivilegeEscalation` not false,
  added capabilities beyond need (especially `SYS_ADMIN`, `NET_ADMIN`,
  `NET_RAW`). Baseline manifest: drop ALL capabilities, add back only what
  is proven needed.
- **Identity**: missing `runAsNonRoot: true` / `runAsUser` (image defaulting
  to root), missing `seccompProfile: RuntimeDefault`,
  `readOnlyRootFilesystem: false` without justification (pair with emptyDir
  mounts for writable paths).
- **Insecure mounts**: hostPath of `/`, `/var/run/docker.sock`,
  containerd sock, `/etc`, kubelet dirs (each is node takeover);
  `hostNetwork`, `hostPID`, `hostIPC: true`; Secrets mounted broadly when
  one key is needed; ServiceAccount token automounted
  (`automountServiceAccountToken`) into pods that never call the API: set
  false by default.
- **Enforcement layer**: verify Pod Security Admission labels per namespace
  (`pod-security.kubernetes.io/enforce: restricted|baseline`); an audit
  without enforcement regresses in a week. Recommend PSA levels (or
  Kyverno/Gatekeeper policies) as the fix, not just per-manifest patches.
- **Supply chain quick pass**: `:latest`/mutable tags (pin digests for
  prod), imagePullPolicy mismatches, images from unvetted registries,
  missing resource limits (a DoS vector, not just an ops issue).

### 3.2 RBAC audit

- **Overly permissive roles**: enumerate ClusterRoles with wildcards:
  `kubectl get clusterroles -o json | jq` filtering rules where verbs,
  resources, or apiGroups contain `*`. cluster-admin bindings to anything
  other than break-glass groups is a HIGH finding.
- **Escalation-equivalent permissions** (flag even without wildcards):
  create pods (can mount any SA token in the namespace), create/patch on
  `pods/exec` or `pods/attach`, read secrets cluster-wide, `escalate`,
  `bind`, `impersonate` verbs, create
  mutatingwebhookconfigurations, update nodes, or create
  ClusterRoleBindings: each is a privilege-escalation primitive. Verify
  with `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>`.
- **Dangling bindings**: RoleBindings/ClusterRoleBindings whose subject
  (ServiceAccount/user/group) or roleRef no longer exists. Cross-join
  bindings against existing SAs; a binding to a recreatable SA name is a
  persistence backdoor (attacker recreates the SA and inherits the role).
  Same for bindings into deleted namespaces.
- **Default SA hygiene**: roles bound to `default` ServiceAccounts (any
  pod in the namespace inherits them), and tokens of system controllers
  reused by apps.
- Deliver RBAC findings with the exact `kubectl auth can-i` reproduction and
  a least-privilege replacement Role, not just "too broad".

---

## Phase 4: AI/GPU Resource Management (vLLM / KServe / model serving)

### 4.1 GPU scheduling fundamentals

- GPUs are requested via extended resources: `resources.limits:
  nvidia.com/gpu: 1` (requests must equal limits; GPUs are not
  overcommittable and not fractional without MIG/time-slicing). Pods Pending
  with GPU requests: check the device plugin DaemonSet is Running on GPU
  nodes (`kubectl -n kube-system get ds | grep -i nvidia`), node advertises
  the resource (`kubectl describe node | grep nvidia.com/gpu`), and
  taints (`nvidia.com/gpu=present:NoSchedule`) have matching tolerations.
- GPU Operator vs manual driver installs: identify which manages drivers;
  driver/CUDA version vs image CUDA version mismatches present as
  `CUDA error: no kernel image` or device init failures at runtime, not at
  schedule time.
- **Sharing strategies**, choose deliberately: MIG (hardware isolation,
  A100/H100, fixed profiles, best for multi-tenant), time-slicing (no
  isolation, fine for dev/bursty small models, dangerous for latency SLOs),
  MPS (process-level concurrency). Document which is in use; a "random
  latency spikes" report on a time-sliced GPU is working as configured.
- Diagnosis inside the node: `nvidia-smi` via node exec or a debug DaemonSet
  for utilization, memory, ECC errors, and zombie GPU memory held by
  terminated processes (requires process kill or device reset). DCGM
  exporter metrics (`DCGM_FI_DEV_GPU_UTIL`, memory, XID errors) are the
  Prometheus ground truth; XID errors in dmesg = hardware/driver events,
  correlate before blaming the app.

### 4.2 Model serving workloads (vLLM)

- vLLM pre-allocates GPU memory (`--gpu-memory-utilization`, default ~0.9):
  CUDA OOM at startup means the model + KV cache does not fit: lower
  utilization fraction, smaller `--max-model-len`, quantized weights, or
  tensor parallelism (`--tensor-parallel-size N` with N GPUs on one node;
  multi-node needs Ray and is a different architecture conversation).
- Container OOMKilled (137) vs CUDA OOM are different failures: the former
  is host RAM (model loading spikes host memory; set memory limits with
  headroom for weight loading), the latter is GPU VRAM.
- Probes for LLM servers: model loading takes minutes; use a generous
  `startupProbe` against `/health`, keep liveness lenient (a busy GPU can
  starve the probe handler and self-inflict restarts), readiness on actual
  serving endpoint.
- Long graceful shutdown: in-flight generations need
  `terminationGracePeriodSeconds` sized to max generation time, plus preStop
  drain, or rollouts cut user requests mid-stream.
- Throughput symptoms: rising time-to-first-token with stable GPU util →
  queue saturation (check vLLM metrics `num_requests_waiting`); low GPU util
  with poor throughput → CPU-bound tokenization or batch size limits.

### 4.3 KServe specifics

- Resolve the stack first: KServe Serverless (Knative + Istio/Kourier) vs
  RawDeployment mode; failure surfaces differ. InferenceService not ready:
  `kubectl get isvc -o yaml` conditions, then walk down: predictor
  Revision → Knative Service → pods.
- Cold-start vs availability: `minReplicas: 0` scale-to-zero means
  multi-minute cold starts for big models; for prod SLOs set
  `minReplicas >= 1` and treat the GPU cost as the price of latency.
  Autoscaling on concurrency (`containerConcurrency`/target) must reflect
  what one replica can truly serve (often 1-4 for LLMs, not the default).
- Model storage: storageUri pulls via storage-initializer init container;
  failures there (IAM on the bucket, size vs ephemeral storage) appear as
  init container errors, not serving errors. Large models: prefer a PVC or
  image-baked weights over re-downloading per pod.
- GPU bin-packing: mixed small/large GPU requests fragment nodes; use
  separate node pools per model size class with taints, and a cluster
  autoscaler that knows GPU node groups (scale-up on `nvidia.com/gpu`
  pending pods).

---

## Phase 5: Risk-Aware Output (mandatory format for every recommendation)

Every suggested action, without exception, ships as:

```
### Action N: <one-line imperative>
- Evidence: <the event/log/metric/field that justifies this>
- Command/Manifest: <exact, copy-pasteable>
- Risk Score: Low | Medium | High
- Blast radius: <what can be affected if this goes wrong>
- Rollback: <exact command(s) to undo, or "not reversible: <why> + mitigation>
- Verify: <command + expected output proving it worked>
```

Risk scoring rubric (be honest, do not deflate):
- **Low**: read-only, or mutation limited to one pod/replica that the
  controller will recreate (delete one pod of a healthy Deployment, label
  changes, adding a startupProbe).
- **Medium**: affects all replicas of one workload or one namespace's
  traffic (rollout restart, resource limit change, new NetworkPolicy on one
  app, HPA changes).
- **High**: cluster-scoped or data-bearing: CRD/webhook changes, RBAC
  edits, node cordon/drain, anything touching StatefulSets' storage,
  kube-system components, CNI, or force-deleting pods with state.

High-risk actions additionally require: a pre-change snapshot of the object
(`kubectl get <obj> -o yaml > backup.yaml`), an explicit user confirmation
gate in the response, and a canary path when one exists (one node, one
replica, one namespace first).

Rollback realism rules: `kubectl rollout undo` only works for template
changes on Deployments/StatefulSets, not for Service/NetworkPolicy/RBAC
edits (those roll back by re-applying the backed-up YAML). Deleted PVCs and
released LoadBalancer IPs may be unrecoverable: classify those as
"not reversible" honestly. If GitOps manages the object, the rollback is a
Git revert, and say so.

Order multiple actions by: diagnostics first, then lowest-risk fix that
addresses the root cause. Do NOT pad the output with optional "nice to have"
suggestions during an incident; park them in a final "Post-incident
hardening" list of max 3 items.

---

## Delivery format

For debugging/triage:
1. **Symptom** (as reported) → **Root cause** (one sentence) → **Evidence
   chain** (the specific outputs that prove it).
2. **Fix**: Phase 5 formatted actions.
3. **Prevention**: which guardrail (probe, limit, policy, PSA level, alert)
   would have caught this earlier: max 3, concrete.

For audits: findings table (severity, object, evidence, fix), then
remediation manifests in Phase 5 format, then enforcement recommendation
(PSA/Kyverno/Gatekeeper) so the audit sticks.

For manifest authoring/review: the manifest itself must already pass the
Phase 3.1 hardening baseline and carry explicit requests/limits and probes;
deviations are called out with justification.

If evidence is insufficient to name a root cause, say exactly that and
deliver the minimal ordered list of diagnostic commands that will
disambiguate: never a speculative fix dressed up as a solution.
