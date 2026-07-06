# [Feature]: Add recipe support for Vera Rubin (vr200) on bare-metal RKE2 (Rancher)

> **CONFIDENTIALITY:** vr200 is not publicly announced hardware. Per the `#aicr`
> "VR/BM/RKE2" Slack decision (2026-07-06), all vr200 recipe work stays in the
> internal recipes repo (`gitlab-master.nvidia.com/dgxcloud/platform/aicr/recipes`)
> until clearance. Do not file this publicly or reference pre-release
> driver/CUDA/hardware details in NVIDIA/aicr.

## Summary

Add an AICR recipe for the Vera Rubin (vr200) NVL72 bare-metal RKE2 cluster:
criteria `service: rke2 / accelerator: vr200 / os: ubuntu / intent: training`,
staged in the internal recipes repo via runtime criteria registration (`--data`),
modeled on the public `bcm` bare-metal overlay chain. Promote to the public
catalog later as a follow-up PR once vr200 is cleared (see NVIDIA/aicr#1141,
which tracks the public ask).

## Motivation / Context

- A Rancher-managed bare-metal RKE2 cluster with VR NVL72 (vr200) hardware is
  available until **2026-07-12** (extendable) — Slack `#aicr` thread
  "VR/BM/RKE2" (p1783365375628159).
- Nodes are tuned (nodewright gb200-equivalent profile applied, confirmed
  complete) and the cluster is otherwise bare: "ready for the full AICR
  treatment".
- Prior context: `#aicr` thread "AICR support for VR200" (p1780328233712869,
  June 1–4), NVIDIA/aicr#1141 (public feature request, `--service bcm`
  variant), nodewright-packages#50 (vr200 nvidia-setup/nvidia-tuned POC).
- Related: internal recipes repo precedents for runtime-registered criteria —
  gb300 (MR !12), nke service (MR !7), b40 side-load demo.

## Verified cluster configuration (inspected 2026-07-06)

| Item | Finding |
|---|---|
| Kubernetes | v1.35.6+rke2r1 (RKE2), containerd 2.2.5-k3s2, Calico CNI, Traefik, CoreDNS, metrics-server |
| Nodes | 3× control-plane/etcd (tainted `control-plane:NoSchedule`, `etcd:NoExecute`) + 2× VR NVL72 GPU workers, untainted |
| Arch / OS | arm64, Ubuntu 26.04 LTS, NVIDIA 64k-page kernel `7.0.0-2008-nvidia-bos-64k`; 352 CPUs, ~2.7 TiB RAM per GPU node |
| GPUs | 4 per node, ~286 GB memory each; `nvidia-smi` product name reads "NVIDIA Graphics Device" (pre-release driver exposes no marketing name) |
| Driver | Pre-baked in BOS image: **615.23 open kernel module (aarch64), CUDA 13.4**. Host driver only — see toolkit note below |
| Container toolkit | **Manually installed** by the cluster owner (stock NCT v1.19.1 via apt, CNS docs), containerd `nvidia` handler wired in `/var/lib/rancher/rke2/agent/etc/containerd/config.toml`, RuntimeClasses `nvidia`/`nvidia-experimental` registered via `rke2-runtimeclasses` chart. **Not** part of the BOS image |
| Fabric | IMEX service active (NVL72 MNNVL, GPU Fabric GUID populated) + 36 Mellanox mlx5 IB devices per node (CX9) for scale-out |
| Tuning | nodewright/skyhook v0.17.1 installed; GPU nodes labeled `skyhook.nvidia.com/status_tuning-new=complete` (gb200-equivalent profile) |
| Labels | Manual `accelerator=vr200` on GPU nodes; no NFD/GFD labels (no GPU operator yet) |
| Storage | **No StorageClass at all** (RKE2 does not ship local-path, unlike k3s) |
| Otherwise | Bare cluster — no gpu-operator, device plugin, or monitoring |

## Driver / GPU Operator / DRA findings

Confirmed in Slack with the GPU Operator and DRA teams (June + July threads):

- **GPU Operator VR support formally lands in v26.7.0**; 26.3.x (current AICR
  base pin: 26.3.2) is not QA'd for VR — "too early to tell". The June VR test
  succeeded with the **driver pre-installed on the host**; the driver-container
  path is untested on VR.
- **Driver drop trajectory on VR hardware**: test systems ran the "0.6 drop"
  (~R610 branch); the June BCM VR cluster had 615.04 (CUDA UMD 13.4); this
  cluster has 615.23 (built 2026-06-09) — the fleet tracks the 615 bring-up
  branch.
- **NVLink observation**: `nvidia-smi nvlink -s` on a GPU node shows all links
  `[Traffic Disabled]` even though IMEX is running and the fabric GUID is
  populated — worth revisiting before any NCCL/performance measurement.
- **Toolkit**: operator-managed is the recommendation (Tariq, 2026-07-06):
  "better to just install the toolkit and configure the container runtimes via
  gpu-operator". Toolkit container is validated on arm64 + Ubuntu 26.04; RKE2
  not explicitly QA'd but "GPU Operator supports RKE". The manual host install
  will be removed before first bundle deploy so the operator starts clean.
- **DRA driver**: no changes needed for basic VR validation; topology-alignment
  enhancement targeted for v0.5.0 (kubernetes-sigs/dra-driver-nvidia-gpu#1123).
  ComputeDomains-on-VR verification status pending (asked in thread).
- **Public state** (validates internal-only stance): public driver docs list
  branches only through R610; the 615 branch and CUDA 13.4 are not publicly
  documented. Public GPU Operator release notes stop at 26.3.3 with no Rubin
  mention. Driver/CUDA milestones tracked on the internal "VR NVL72 Schedule"
  Confluence page.

## Proposed recipe

### Taxonomy

`rke2` is a **service** (like eks/gke/bcm/ocp), not a platform (platform =
kubeflow/dynamo/slurm/nim). Criteria:

```
service: rke2
accelerator: vr200
os: ubuntu
intent: training
```

Both `rke2` and `vr200` are new criteria values. No public enum change needed
initially: register them at runtime via `--data` (the gb300/nke/b40 mechanism —
criteria passed as CLI flags, never baked into `spec.recipe.criteria`, per the
internal repo's `ci/build-bundle.sh` convention).

### Overlay chain (internal repo `overlays/`)

| File | Content |
|---|---|
| `rke2.yaml` | Service root, `base: base`. Modeled on public `bcm.yaml`: bare-metal, self-managed; storage + control-plane toleration handling; exclude cloud-only components (aws-efa, aws-ebs-csi-driver, gke-nccl-tcpxo) |
| `rke2-training.yaml` | Intent layer, mirrors `bcm-training.yaml` |
| `vr200-rke2-ubuntu-training.yaml` | Leaf: criteria above; own OS constraints (`ID=ubuntu`, `VERSION_ID "26.04"`) — do **not** reuse the `os-ubuntu` mixin (it pins 24.04 / kernel >= 6.8; this cluster is 26.04 / kernel 7.0) |

### Key componentRef decisions (leaf)

- **gpu-operator**: `driver.enabled: false` (host-managed 615.23 — the only
  QA'd path today), **`toolkit.enabled: true`** (operator-managed per GPU
  Operator team recommendation). Set RKE2 containerd paths in toolkit env:
  `CONTAINERD_CONFIG=/var/lib/rancher/rke2/agent/etc/containerd/config.toml`,
  `CONTAINERD_SOCKET=/run/k3s/containerd/containerd.sock` — baked into the
  `rke2` service overlay so it applies to every rke2 recipe. Running this also
  gives us a concrete data point for the "GPU Operator on RKE2" QA gap
  (validated on arm64 + Ubuntu 26.04; RKE2 not explicitly QA'd). Likely
  `cdi.enabled: true` (gb200/gb300 pattern).
- **nvidia-dra-driver-gpu**: `nvidiaDriverRoot: /` (host-managed driver
  lockstep); pin chart v0.5.0 for NVL72 topology alignment; ComputeDomains for
  MNNVL/IMEX (K8s 1.35 satisfies the >= 1.34 GA-DRA constraint gb300 uses).
- **nodewright-customizations**: gb200-equivalent profile (already applied on
  the cluster and confirmed by tuning owner); `kernelAllowNewer: true` for the
  arm64 64k-page kernel; watch tuned 2.24+ profile behavior change.
- **Storage**: add a `local-path-provisioner` component (internal
  `components/`) as dependency for kube-prometheus-stack — RKE2 ships no
  StorageClass. (Alternative: emptyDir for Prometheus.)
- **Exclude**: aws-efa, aws-ebs-csi-driver, gke-nccl-tcpxo, aws-net-dra.
- **Networking (phase 2)**: 36× mlx5 (CX9) per node need an RDMA exposure
  decision — network-operator/SR-IOV vs dranet (AWS leaning dranet per June
  thread). Defer, consistent with deferring the performance phase.

### Validation

- **Deployment + conformance phases only initially** (gb300 precedent):
  operator-health, expected-resources, check-nvidia-smi; platform-health,
  dra-support, gang-scheduling, accelerator-metrics, etc.
- Adjust/skip `gpu-operator-version` driver expectations for the host-managed
  driver; express a host driver constraint (>= 615.x) instead. When the recipe
  later moves to operator-managed driver, gate on gpu-operator >= v26.7.0.
- **Defer the performance (NCCL) phase**: gb200 bandwidth floors are
  fabric-baselined and unmeasured on VR. Capture informal NCCL numbers during
  validation for a future floor.

### Known gaps / risks

1. **SKU auto-detection**: `nvidia-smi` reports "NVIDIA Graphics Device", so
   `pkg/fingerprint/gpu_sku.go` token matching and GFD `nvidia.com/gpu.product`
   labels cannot identify vr200. Short-term: manual `accelerator=vr200` node
   label + flag-driven generation. Longer-term: PCI device-ID match or a
   driver drop that exposes the real name.
2. **arm64 everywhere**: validator/agent images must be multi-arch; local
   builds need `docker buildx --platform linux/amd64,linux/arm64`.
3. **`+rke2r1` version suffix**: verify `K8s.server.version >= 1.34.0`
   evaluates correctly against `v1.35.6+rke2r1` (cf. the EKS `-0` kubeVersion
   suffix issue).
4. **expected-resources check** may need a vr200 entry
   (`validators/deployment/expected_resources.go`).
5. **Scheduling**: GPU nodes currently untainted; pass
   `--accelerated-node-selector accelerator=vr200` plus standard tolerations
   (skyhook runtime-required taint flow, NVIDIA/aicr#1614) at bundle time.

## Open questions (pending in Slack)

1. Do GFD/device-plugin/DCGM handle VR correctly while the driver reports the
   generic "NVIDIA Graphics Device" product name (gpu.product label, metrics)?
2. Rough ETA for GPU Operator 26.7.0 (gates the operator-managed-driver
   version constraint for the eventual public recipe).
3. Did the ComputeDomains-on-VR (DRA/IMEX) verification planned in June
   happen? Anything to watch on K8s 1.35/RKE2?
4. Scale-out fabric exposure for the CX9 NICs: network-operator/SR-IOV vs
   dranet (AWS reportedly leaning dranet) — needed before the performance
   phase.

## Cluster access note

Access is via a Rancher-generated kubeconfig that uses an exec credential
plugin (`rancher token`): install the `rancher` CLI and complete an Azure AD
device-code SSO login once; the token is cached (~24h TTL) and plain `kubectl`
works from then on. The kubeconfig embeds no credentials, so it is safe to
pass around internally but useless without SSO.

## Execution plan

1. `aicr snapshot` of the cluster; stash under `/tmp/aicr/vr/`.
2. Internal repo MR: three overlays + `local-path-provisioner` component;
   build/verify via `make` + `build-bundle.sh` (`--data` + criteria flags).
3. Deploy bundle to the VR cluster (coordinate manual-toolkit removal with the
   cluster owner first); run `aicr validate` deployment + conformance phases.
4. Capture informal NCCL numbers for a future performance floor.
5. Public promotion PR once vr200 is cleared: L40S-style enum audit
   (criteria.go, all `server.yaml` enum blocks, gpu_sku.go, docs, issue
   templates, UAT) — mechanical, well-precedented.

## Success criteria

```
aicr recipe --data <internal-recipes> --service rke2 --accelerator vr200 \
  --os ubuntu --intent training --output vr200-recipe.yaml
aicr bundle -r vr200-recipe.yaml -o ./bundles \
  --accelerated-node-selector accelerator=vr200 <tolerations...>
aicr validate -r vr200-recipe.yaml -s snapshot.yaml --phase deployment
aicr validate -r vr200-recipe.yaml -s snapshot.yaml --phase conformance
```

All deployment + conformance checks pass on the VR NVL72 RKE2 cluster.

## References

- Slack `#aicr`: [VR/BM/RKE2 thread (2026-07-06)](https://nvidia.slack.com/archives/C0A457AAWUC/p1783365375628159);
  [AICR support for VR200 thread (2026-06-01)](https://nvidia.slack.com/archives/C0A457AAWUC/p1780328233712869)
- NVIDIA/aicr#1141 (public feature request — bcm variant)
- kubernetes-sigs/dra-driver-nvidia-gpu#1123 (topology alignment, v0.5.0)
- nodewright-packages#50 (vr200 nvidia-setup/nvidia-tuned POC)
- Internal recipes repo MRs: !12 (gb300 EKS RoCE), !7 (nke service)
- Internal Confluence: "VR NVL72 Schedule" (CUDA/driver milestones)
