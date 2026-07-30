# Broadcom bnxt OOB Driver for GPUDirect RDMA on Kubernetes

## The Problem

### RDMA Userspace vs Kernel Drivers (UMD / KMD)

RDMA (RoCEv2) splits the driver stack into two cooperating halves:

| Layer | Name | Runs where | Example (Broadcom) | Example (NVIDIA) |
|---|---|---|---|---|
| **KMD** | Kernel-Mode Driver | Host kernel | `bnxt_re.ko` | `mlx5_ib.ko` |
| **UMD** | User-Mode Driver | Process / container | `libbnxt_re-rdmav34.so` | `libmlx5-rdmav34.so` |

The application stack (e.g. vLLM, MoRI, NCCL, NIXL, NVSHMEM) talks to
**libibverbs**, which loads a vendor **provider** (the UMD). That UMD issues
ioctls through `/dev/infiniband/uverbs*` to the host **KMD**. Memory
registration (`ibv_reg_mr`), GPUDirect / dmabuf pinning, and queue-pair setup
all cross this UMD↔KMD boundary via the provider **uABI** (the kernel **uAPI**
structs and version advertised for that driver).

```text
  ┌─────────────────────────────────────────────────────────┐
  │  Container (vLLM / MoRI / NCCL / NIXL / NVSHMEM)        │
  │                                                         │
  │   application                                           │
  │        │                                                │
  │        ▼                                                │
  │   libibverbs  ──loads──►  UMD provider                  │
  │                           (libbnxt_re / libmlx5)        │
  └─────────────────────────────┬───────────────────────────┘
                                │ ioctl / provider uABI
                                │ (kernel uAPI)
                                ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Host OS                                                │
  │   /dev/infiniband/uverbsX                               │
  │        │                                                │
  │        ▼                                                │
  │   KMD (bnxt_re.ko / mlx5_ib.ko)  ──►  NIC (RoCEv2)      │
  └─────────────────────────────────────────────────────────┘
```

If the UMD and KMD disagree on that provider uABI (or on optional feature
layouts behind it), verbs calls fail — often at memory registration — and
GPUDirect RDMA breaks.

### Broadcom BCM57608 (Thor) — Why Matching Matters

This shows up on Broadcom RoCEv2 NICs (e.g. **BCM57608 / Thor-2**) when enabling
**GPUDirect RDMA** for **CUDA** (NVIDIA GPUs / NCCL) or **ROCm** (AMD Instinct /
RCCL) workloads:

1. **In-tree (distro) UMD + KMD are insufficient for Broadcom’s supported
   GPUDirect RDMA path.**  
   Upstream / distro `bnxt_re` and the inbox `libbnxt_re` from
   `ibverbs-providers` (binary package built from the `rdma-core` source) provide
   basic RoCE verbs (host DRAM registration). Registering **GPU memory** for
   GPUDirect RDMA on these NICs requires Broadcom’s out-of-tree `bnxt_re` /
   `bnxt_en` plus matching `libbnxt_re`, and additional peer-memory plumbing
   described in the next subsection. Inbox-only hosts are not Broadcom’s
   supported GPUDirect configuration. Separately, mixing inbox UMD with an OOB
   KMD is a known failure mode: libibverbs rejects mismatched provider uABI
   ranges.

2. **Infra providers are required to install that out-of-tree KMD on the host.**  
   For Thor AI **GPUDirect RDMA** deployments (CUDA or ROCm), clusters must load
   Broadcom’s OOB `bnxt_re` (and matching `bnxt_en`, typically via DKMS / KMM /
   Broadcom’s installer) from the same Broadcom release train as the NIC
   firmware — not rely on the distro inbox modules alone.

3. **The product image still ships an inbox (or pinned) UMD.**  
   - CUDA `vllm/vllm-openai`: `libibverbs-dev` → hard-depends on
     `ibverbs-providers` → **inbox** `libbnxt_re` only.  
   - ROCm `vllm/vllm-openai-rocm`: same inbox providers, **plus** Broadcom
     `bnxt-rocelib` (e.g. `235.2.86.0`) for MoRI — still a **fixed** package
     train, not automatically equal to whatever KMD is on every host.

4. **Broadcom does not provide mlx5-style forward/backward UMD↔KMD
   compatibility between inbox and OOB (or across arbitrary OOB trains).**  
   Upstream `bnxt` / `rdma-core` keep `BNXT_RE_ABI_VERSION` at **1** and extend
   via `comp_mask`. Broadcom’s OOB `bnxt_re` historically advertises a **higher**
   provider uABI (community reports and Broadcom docs cite values such as **6**
   and **8**). Inbox `libbnxt_re` only accepts ABI **1–1**, so libibverbs refuses
   the device with:
   `Driver bnxt_re does not support the kernel ABI of N (supports 1 to 1)`.
   Broadcom’s own guidance is to install Broadcom’s `libbnxt_re` with the OOB
   KMD; a Broadcom engineer stated they maintain compatibility between the
   *out-of-tree driver and out-of-tree library*, not with distro/`rdma-core`
   UMD ([linux-rdma mailing list, Jun 2024](https://www.spinics.net/lists/linux-rdma/msg124163.html)).
   Broadcom also ships `bnxt_en` / `bnxt_re` / `libbnxt_re` as a **matched
   release bundle** (and FAQ cases cover inbox UMD reappearing after
   `rdma-core` updates). Cross-train mixes (e.g. image `bnxt-rocelib` **235.x**
   vs host OOB **231.x**, or inbox UMD vs OOB KMD) are unsupported and fail at
   provider match / device open — before reliable GPUDirect memory registration.

**Net result:** the UMD (`libbnxt_re*.so`) used inside the product must come
from the **same Broadcom release train** as the KMD (`bnxt_re`) loaded on the
host.

### GPUDirect peer-memory paths: `ib_peer_mem` vs dmabuf

GPUDirect RDMA needs a way for the RDMA stack to pin/map **GPU** memory. On
Broadcom Thor this is not just `libbnxt_re` ↔ `bnxt_re`; two host-side approaches
matter for CUDA and ROCm:

| Path | What it is | Typical stack |
|---|---|---|
| **Legacy Peer Memory Direct** | Broadcom’s peer-mem client model | `ib_peer_mem` **kernel module** (+ GPU-side peer module) |
| **dmabuf** | Upstream DMA-BUF registration | `ibv_reg_dmabuf_mr` via a capable matched UMD/KMD |

**`ib_peer_mem` is a kernel module, not a userspace library.** It ships in
Broadcom’s Peer Memory Direct / `netxtreme-peer-mem` package (often as DKMS)
alongside the OOB `bnxt_en` / `bnxt_re` modules for that same release.

**Legacy (non-dmabuf) CUDA path — yes, both modules together:** Broadcom’s
NVIDIA GPUDirect / Peer Memory Direct docs load **`ib_peer_mem`** (NIC/RDMA
peer client) **and** **`nvidia-peermem`** (NVIDIA GPU driver peer module), plus
`bnxt_en` / `bnxt_re`. They solve different sides of the P2P path; one does not
replace the other. Exception: some distro kernels already expose inbox
peer-memory APIs — Broadcom’s build may then skip its `ib_peer_mem` module, and
`nvidia-peermem` alone can be enough on that kernel.

**ROCm:** Broadcom’s AMD Instinct (e.g. MI300X) guides likewise require
`ib_peer_mem` for Peer Memory Direct with ROCm/RCCL workloads (paired with the
AMD GPU driver stack, not `nvidia-peermem`).

**Version matching on the legacy path:** treat `ib_peer_mem` like `bnxt_re` —
it must come from the **same Broadcom release train** as the loaded
`bnxt_en` / `bnxt_re` (the peer-mem DKMS bundle is versioned with them). Do not
mix an older `ib_peer_mem` with a newer OOB `bnxt_re` train.

**dmabuf path:** newer userspace/kernel registration (`ibv_reg_dmabuf_mr`) used
by modern CUDA/ROCm collectives and libraries. It still requires a
**version-matched, GPUDirect-capable** `libbnxt_re` ↔ `bnxt_re` pair; inbox UMD
against OOB KMD will not reliably enable it. Whether a given cluster uses
legacy peer-mem, dmabuf, or both depends on kernel, GPU driver, and app stack —
the UMD/KMD match requirement remains either way.

### Why NVIDIA ConnectX (CX-6/CX-7) Does Not Hit This

| | Broadcom Thor (`bnxt`) | NVIDIA ConnectX (`mlx5`) |
|---|---|---|
| UMD in typical AI images | Inbox and/or vendor `bnxt-rocelib` pin | Distro `libmlx5` via `ibverbs-providers` |
| KMD on hosts | Often **OOB** for GPUDirect / Thor features | Inbox and/or DOCA-OFED; uAPI kept stable |
| UMD↔KMD contract | OOB stack requires matching Broadcom UMD | Strong: `MLX5_IB_UVERBS_ABI_VERSION` stays at 1; `comp_mask`, versioned structs, capability negotiation |
| RDMA device exposure in Kubernetes | Same for both (see note below) | Same for both (see note below) |
| Day-2 effort | High — align host/image Broadcom train, or mount host UMD | Low — image `libmlx5` usually works with host `mlx5_ib` |

> **Kubernetes device plugins (same for both NICs):** RDMA Shared Device Plugin,
> SR-IOV Network Device Plugin, and (on NVIDIA clusters) the Network Operator’s
> device-plugin path all expose InfiniBand character devices such as
> `/dev/infiniband/uverbs*` into the pod. They are **not** Broadcom- or
> mlx5-specific library injectors: they do **not** mount `libbnxt_re` or
> `libmlx5` into application containers. The UMD must already be in the image
> (or mounted separately, as in this guide). The difference is only whether
> that image UMD can safely talk to the host KMD.

`libmlx5` and `mlx5_ib` live in upstream `rdma-core` / mainline Linux with a
deliberate compatibility model: an older container UMD can talk to a newer host
KMD and fall back on unsupported features instead of breaking core verbs.

### Bottom Line

When using Broadcom NICs with RDMA, the in-container `libbnxt_re-rdmav34.so` userspace
provider must match the host kernel module (`bnxt_re.ko`). If the host runs an out-of-tree
Broadcom driver (e.g., installed via the Kernel Module Management (KMM) operator), the
container's bundled or inbox library will be incompatible.

## The Solution (Two Parts)

The full solution has two parts:

| Part | Who | What |
|---|---|---|
| **1. Infra (KMD + UMD on host)** | Platform / SRE team | Install Broadcom's OOB kernel modules (`bnxt_en`, `bnxt_re`, optionally `ib_peer_mem`) **and** the matching userspace library (`libbnxt_re`) onto every RDMA-capable node via KMM on OpenShift |
| **2. Application (UMD mount into pod)** | AI/ML platform team | Mount the host-installed `libbnxt_re` into the vLLM container at the path its `libibverbs` expects |

---

## Part 1: Infra — Installing the OOB Driver via KMM on OpenShift

### Reference Environment (Reported by Infra/SRE Team)

| Component | Version |
|---|---|
| Broadcom driver package | `nxe_linux_237.1.148.0.tar.gz` (component version 237.1.137.0) |
| Driver version (`bnxt_en`) | 1.10.3-237.1.137.0 |
| OpenShift | 4.20.27 |
| RHCOS kernel | `5.14.0-570.124.1.el9_6.x86_64` (RHEL 9.6 base) |
| KMM (Kernel Module Management) | v2.6.1 |

> **Note:** The kernel modules (`bnxt_en`, `bnxt_re`) are compiled from source by the KMM
> builder pod against the running RHCOS kernel. This makes the deployment portable — the same
> source and KMM configuration work regardless of the specific RHCOS kernel version on the
> cluster.

### Package Contents (`nxe_linux_237.1.148.0.tar.gz`)

The package from Broadcom Support (component version **237.1.137.0**) contains:

| Directory | Contents | Source for KMM Build |
|---|---|---|
| `bnxt_en/` | Ethernet kernel driver (KMD) | `bnxt_en-1.10.3-237.1.137.0.tar.gz` (source tarball) or `bnxt_en-1.10.3.237.1.137.0-1dkms.noarch.rpm` |
| `bnxt_re/` | RDMA kernel driver (KMD) | `bnxt_re-237.1.137.0-1dkms.noarch.rpm` (contains source under `/usr/src/bnxt_re-237.1.137.0/`) |
| `bnxt_rocelib/` | **Userspace RDMA provider (UMD)** — this is `libbnxt_re` | `libbnxt_re-237.1.137.0-rhel9u7.x86_64.rpm` (prebuilt binary, kernel-independent) |
| `peer_mem/` | Peer Memory Direct kernel module (GPUDirect legacy path) | `netxtreme-peer-mem-237.1.137.0.tar.gz` (source) or DKMS RPM |

> **Recommended approach: always build kernel modules from source via KMM.** The package
> includes prebuilt kmod RPMs for specific kernels (e.g. `5.14.0-611.5.1.el9_7`), but these
> only work on that exact kernel. Building from the DKMS source against the running RHCOS
> kernel makes the solution portable across OCP versions and kernel updates. The userspace
> library (`libbnxt_re-rdmav57.so`) is a prebuilt binary — it is kernel-version-independent
> and does not need recompilation.

### Where `libbnxt_re` Installs on RHEL 9 (Standard RPM Path)

The `libbnxt_re-237.1.137.0-rhel9u7.x86_64.rpm` installs these files:

```text
/etc/libibverbs.d/bnxt_re.driver            ← content: "driver bnxt_re"
/usr/lib64/libbnxt_re-rdmav57.so            ← actual binary (98 KB)
/usr/lib64/libbnxt_re.so                    → symlink → libbnxt_re-rdmav57.so
/usr/lib64/libbnxt_re-237.1.137.0.so        → symlink → libbnxt_re.so
```

> **Important naming difference:** This package names the provider **`libbnxt_re-rdmav57.so`**
> (built against rdma-core v57 / RHEL 9.7). Container images ship the inbox provider named
> **`libbnxt_re-rdmav34.so`** (older rdma-core ABI naming). This difference is handled by the
> volume mount — you mount the host's `rdmav57` file at the container's `rdmav34` path. This
> works because Broadcom's OOB library maintains backward compatibility across rdma-core
> versions v14–v56 (per their readme), and libibverbs loads the provider via `dlopen()` on the
> mounted path regardless of the internal SONAME.

### OpenShift Host OS Considerations

OpenShift worker nodes can run either **RHCOS** or **RHEL**:

| | RHCOS (default) | RHEL workers |
|---|---|---|
| `/usr/` | Immutable (rpm-ostree) | Mutable |
| `/opt/` | Writable, persistent (`/opt/` → `/var/opt/`) | Writable, persistent |
| Direct RPM install to `/usr/lib64/` | Not possible | Possible but not recommended for consistency |

On RHCOS nodes you **cannot** `rpm -ivh libbnxt_re-*.rpm` directly — `/usr/` is read-only.
On RHEL workers you technically could, but using `/opt/` keeps the deployment method
**identical across both host OS types**.

**Recommendation:** Deploy `libbnxt_re-rdmav57.so` to `/opt/broadcom/lib64/` on all workers.
This path is writable and persistent on both RHCOS and RHEL, making the solution portable
regardless of the worker OS.

The KMM Module CR handles **both** kernel modules and the userspace library in a single
resource — its module-loader pod copies `libbnxt_re-rdmav57.so` to the host before loading
the kernel modules. No separate DaemonSet needed.

**Resulting host path:**

```
/opt/broadcom/lib64/libbnxt_re-rdmav57.so
```

### How to Verify the Library Is on the Host

After deploying the KMM Module CR, confirm the library is in place:

```bash
kubectl debug node/<node-name> -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  chroot /host ls -la /opt/broadcom/lib64/libbnxt_re-rdmav57.so
```

Expected output:
```
-rwxr-xr-x. 1 root root 98336 ... /opt/broadcom/lib64/libbnxt_re-rdmav57.so
```

### Verifying KMD + UMD Are Working on the Host

Once the kernel modules are loaded and `libbnxt_re` is on disk, verify from a debug pod:

```bash
kubectl debug node/<node-name> -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  chroot /host bash -c '
  echo "=== Loaded bnxt modules ==="
  lsmod | grep bnxt
  echo ""
  echo "=== RDMA devices ==="
  ibv_devices 2>/dev/null || ls /sys/class/infiniband/
  echo ""
  echo "=== libbnxt_re files ==="
  find / -name "libbnxt_re*" 2>/dev/null
  echo ""
  echo "=== bnxt_re kernel ABI ==="
  cat /sys/class/infiniband/bnxt_re*/device/driver/module/version 2>/dev/null
'
```

### KMM Module CR (In-Cluster Auto-Build)

KMM builds kernel modules **and** deploys the userspace library **in-cluster, automatically** —
the same pattern NVIDIA Network Operator uses for DOCA/OFED drivers. You push a **source image**
once (containing the Broadcom DKMS source + `libbnxt_re-rdmav57.so`). KMM handles the rest.

| | NVIDIA Network Operator | Broadcom + KMM (this guide) |
|---|---|---|
| Source image | NVIDIA publishes it (`nvcr.io/nvidia/mellanox/doca-driver`) | Built by customer from `nxe_linux_*.tar.gz` (Broadcom does not publish a container image) |
| When compilation happens | At **pod startup** — DaemonSet pod detects kernel, compiles, and loads in one shot | At **build time** — KMM triggers a separate in-cluster image build, then deploys the result |
| Who publishes | NVIDIA (publicly on NGC) | Customer pushes to their own registry |
| Kernel upgrade handling | Pod restart triggers recompile | KMM detects new kernel, triggers rebuild automatically |

The workflow:

1. Detects each node's kernel version
2. Builds `bnxt_en.ko` + `bnxt_re.ko` using the matching Driver Toolkit (DTK) image
3. Pushes the compiled module image to your registry (cached per kernel version)
4. On each node: copies `libbnxt_re-rdmav57.so` to `/opt/broadcom/lib64/`, then loads the modules
5. Automatically rebuilds when kernels change (OCP upgrades)

One source image. One Module CR. No DaemonSet. No image matrix per kernel version.

The full manifests are in [`kmm/`](kmm/):
- `kmm/module-cr.yaml` — Module CR with `build` stanza + UMD host deployment
- `kmm/build-configmap.yaml` — ConfigMap with the multi-stage Dockerfile KMM uses
- `kmm/builder/Dockerfile` — Source image (packages DKMS source + UMD, no compilation)
- `kmm/vllm-pod-example.yaml` — Example vLLM pod with host-mounted UMD

```yaml
# Simplified view of the Module CR (see kmm/module-cr.yaml for full version)
apiVersion: kmm.sigs.x-k8s.io/v1beta2
kind: Module
metadata:
  name: broadcom-bnxt
  namespace: openshift-kmm
spec:
  moduleLoader:
    container:
      modprobe:
        moduleName: bnxt_re
        dirName: /opt/lib/modules/${KERNEL_FULL_VERSION}
      kernelMappings:
        - regexp: '^.*\.el9.*$'
          containerImage: <registry>/broadcom-bnxt-kmm:237.1.137.0-${KERNEL_FULL_VERSION}
          build:
            dockerfileConfigMap:
              name: broadcom-bnxt-build
            buildArgs:
              - name: BROADCOM_SRC_IMAGE
                value: "<registry>/broadcom-bnxt-src:237.1.137.0"
  selector:
    node-role.kubernetes.io/worker: ""
    feature.node.kubernetes.io/pci-14e4.present: "true"
```

---

## Part 2: Application — Mounting the Host UMD into the vLLM Container

The second part is to mount the host-installed `libbnxt_re-rdmav57.so` into the container
at the path its `libibverbs` expects.

---

## Container Mount Path Reference

### Upstream vLLM Images (Docker Hub)

The mount target inside the container depends on the image family:

| Image Family | Example Tag | Base OS | rdma-core (inbox) | bnxt_re provider | Container `mountPath` |
|---|---|---|---|---|---|
| `rocm/vllm-dev` | `main_20250529` | Ubuntu 24.04 | 50.0-2 | **Out-of-tree** Broadcom 235.2.86.0 (539 KB) via `bnxt-rocelib` | `/usr/local/lib/x86_64-linux-gnu/libbnxt_re-rdmav34.so` |
| `vllm/vllm-openai-rocm` | `v0.26.0` | Ubuntu 22.04 | 39.0-1 | **Out-of-tree** Broadcom 235.2.86.0 (539 KB) via `bnxt-rocelib` | `/usr/local/lib/x86_64-linux-gnu/libbnxt_re-rdmav34.so` |
| `vllm/vllm-openai-rocm` | `v0.26.0-base` | Ubuntu 22.04 | 39.0-1 | Inbox only (35 KB) — **no** `bnxt-rocelib` installed | `/usr/lib/x86_64-linux-gnu/libibverbs/libbnxt_re-rdmav34.so` |
| `vllm/vllm-openai` (CUDA, `-ubuntu2404`) | `v0.26.0-cu129-ubuntu2404` | Ubuntu 24.04 | 50.0-2 | Inbox only (43 KB) | `/usr/lib/x86_64-linux-gnu/libibverbs/libbnxt_re-rdmav34.so` |
| `vllm/vllm-openai` (CUDA, no `-ubuntu2404`) | `v0.26.0-cu129` | Ubuntu 22.04 | 39.0-1 | Inbox only (35 KB) | `/usr/lib/x86_64-linux-gnu/libibverbs/libbnxt_re-rdmav34.so` |

> **Why the difference?** All images use the **standard distro `rdma-core`** package (inbox).
> The ROCm Dockerfile additionally installs the `bnxt-rocelib=235.2.86.0` package from
> Broadcom's PPA (`packages.broadcom.com`), which drops the out-of-tree `libbnxt_re` provider
> into `/usr/local/lib/x86_64-linux-gnu/` and adds an `ld.so.conf.d` entry so libibverbs
> loads it instead of the inbox stub. The inbox `.so` is preserved as `.so-inbox`.
>
> CUDA images install only `ibverbs-providers` from the distro and do **not** add the
> Broadcom PPA — so they have just the small inbox stub at the standard libibverbs
> providers path.

### Red Hat Registry Images (registry.redhat.io)

All Red Hat vLLM images are RHEL 9 based and use the **same** `mountPath`. None of them
ship the custom Broadcom `bnxt-rocelib` package — they all have only the inbox provider
from `libibverbs`.

| Image | Tag Inspected | Base OS | libibverbs | bnxt_re provider | Container `mountPath` |
|---|---|---|---|---|---|
| `rhaii/vllm-rocm-rhel9` | `3.4.1` | RHEL 9.6 | 54.0-2.el9_6 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhaii/vllm-cuda-rhel9` | `3.4.1` | RHEL 9.6 | 54.0-2.el9_6 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhaiis/vllm-rocm-rhel9` | `3.2.2` | RHEL 9.6 | 54.0-1.el9 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhaiis/vllm-cuda-rhel9` | `3.2.2` | RHEL 9.6 | 54.0-1.el9 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhoai/odh-vllm-rocm-rhel9` | `v2.25.9` | RHEL 9.6 | 54.0-2.el9_6 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhoai/odh-vllm-cuda-rhel9` | `v2.25.9` | RHEL 9.6 | 54.0-2.el9_6 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhaii-early-access/vllm-rocm-rhel9` | `3.5.0-ea.2` | RHEL 9.6 | 54.0-2.el9_6 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |
| `rhaii-early-access/vllm-cuda-rhel9` | `3.5.0-ea.2` | RHEL 9.6 | 54.0-2.el9_6 | Inbox only (44 KB) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` |

> **Key difference from upstream:** The RHEL 9 path is `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so`
> (not `/usr/lib/x86_64-linux-gnu/...`). This is because RHEL 9 uses `/usr/lib64/` for 64-bit
> libraries rather than the Debian/Ubuntu multiarch `/usr/lib/x86_64-linux-gnu/` layout.
>
> **No custom Broadcom driver in any Red Hat image** — unlike upstream `vllm/vllm-openai-rocm`
> which bundles `bnxt-rocelib=235.2.86.0`, the Red Hat ROCm images do not include it. All Red
> Hat images will require the host-mounted `.so` if RDMA with Broadcom NICs is needed.

Catalog links:
- [`rhaii/vllm-cuda-rhel9`](https://catalog.redhat.com/en/software/containers/rhaii/vllm-cuda-rhel9/69a57f8e94c1b4cc1eca015b)
- [`rhaiis/vllm-rocm-rhel9`](https://catalog.redhat.com/en/software/containers/rhaiis/vllm-rocm-rhel9/6825c6b66db83eab748652de)
- [`rhaiis/vllm-cuda-rhel9`](https://catalog.redhat.com/en/software/containers/rhaiis/vllm-cuda-rhel9/6825c6b827e18fe3162148a9)
- [`rhoai/odh-vllm-cuda-rhel9`](https://catalog.redhat.com/en/software/containers/rhoai/odh-vllm-cuda-rhel9/68af11f5fad01b87e349f2fa)
- [`rhoai/odh-vllm-rocm-rhel9`](https://catalog.redhat.com/en/software/containers/rhoai/odh-vllm-rocm-rhel9/68af11f5c44926b8245ce3cd)
- [`rhaii-preview/vllm-cuda-rhel9`](https://catalog.redhat.com/en/software/containers/rhaii-preview/vllm-cuda-rhel9/69a57f9826444ada528619ad)

---

### How Upstream vLLM ROCm Images Add the Broadcom UMD

The `rdma-core` package is **inbox (distro)** in ALL images — ROCm and CUDA alike. The ROCm
Dockerfile ([`docker/Dockerfile.rocm`](https://github.com/vllm-project/vllm/blob/1479bd9e9d3e7e06a4167980d4d4662eeda0638c/docker/Dockerfile.rocm))
adds the out-of-tree Broadcom userspace provider on top via the following steps:

1. **Installs standard `ibverbs-providers`** from the distro ([line 366](https://github.com/vllm-project/vllm/blob/1479bd9e9d3e7e06a4167980d4d4662eeda0638c/docker/Dockerfile.rocm#L366))
   — this includes the inbox `libbnxt_re-rdmav34.so`.

2. **Defines `NIC_BACKEND` build arg** (default `all`) ([line 15](https://github.com/vllm-project/vllm/blob/1479bd9e9d3e7e06a4167980d4d4662eeda0638c/docker/Dockerfile.rocm#L15))
   — controls which NIC vendor libraries get installed (`none`, `ainic`, `bnxt`, or `all`).

3. **Runs `install_bnxt()`** ([lines 507–519](https://github.com/vllm-project/vllm/blob/1479bd9e9d3e7e06a4167980d4d4662eeda0638c/docker/Dockerfile.rocm#L507-L519)) which:
   - Adds the Broadcom PPA signing key and apt source
     (`packages.broadcom.com/artifactory/ethernet-nic-debian-public jammy main`)
   - Installs **`bnxt-rocelib=235.2.86.0`** — this is *only* the userspace bnxt_re libibverbs
     provider, not a full rdma-core replacement
   - The `bnxt-rocelib` package drops the out-of-tree `.so` into `/usr/local/lib/x86_64-linux-gnu/`
     and creates `/etc/ld.so.conf.d/libbnxt_re.conf` pointing to that directory
   - Copies the library to `/usr/local/lib/` as well and runs `ldconfig`
   - The inbox `.so` is preserved as `.so-inbox` (renamed by the package, not deleted)

4. **Dispatches via `NIC_BACKEND`** ([lines 529–534](https://github.com/vllm-project/vllm/blob/1479bd9e9d3e7e06a4167980d4d4662eeda0638c/docker/Dockerfile.rocm#L529-L534)):
   ```
   case "${NIC_BACKEND}" in
     none)  ;;
     all)   install_ainic; install_bnxt ;;
     ainic) install_ainic ;;
     bnxt)  install_bnxt ;;
   esac
   ```

The **CUDA Dockerfile** ([`docker/Dockerfile`](https://github.com/vllm-project/vllm/blob/1479bd9e9d3e7e06a4167980d4d4662eeda0638c/docker/Dockerfile))
installs only `libibverbs-dev` for build-time headers and does **not** reference the Broadcom PPA
or `bnxt-rocelib` at all — hence CUDA images only have the small inbox stub.

> **Note on the `-base` tag**: The `v0.26.0-base` tag corresponds to `Dockerfile.rocm_base` which
> only sets up the ROCm runtime. The `install_bnxt()` step runs in `Dockerfile.rocm` (the full
> image), which is why the `-base` tag has the same mount path as CUDA images.

---

## Pod YAML Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vllm-inference
  labels:
    app: vllm
spec:
  containers:
    - name: vllm
      image: <IMAGE>  # e.g., rocm/vllm-dev:main_20250529 or vllm/vllm-openai:v0.26.0-cu129-ubuntu2404
      # ... other container config ...
      volumeMounts:
        - name: libbnxt-re
          mountPath: <CONTAINER_LIBBNXT_PATH>   # See "Container Mount Path Reference" table above
          readOnly: true
  volumes:
    - name: libbnxt-re
      hostPath:
        path: <HOST_LIBBNXT_PATH>              # Path on the host where KMM placed libbnxt_re
        type: File
```

### Placeholders

| Placeholder | Description | Example (237.x on OpenShift) |
|---|---|---|
| `<IMAGE>` | The vLLM container image | `registry.redhat.io/rhaii/vllm-rocm-rhel9:3.4.1` |
| `<HOST_LIBBNXT_PATH>` | Absolute path on the host where the DaemonSet placed `libbnxt_re` | `/opt/broadcom/lib64/libbnxt_re-rdmav57.so` |
| `<CONTAINER_LIBBNXT_PATH>` | Path inside the container — **must match the image family** (see table above) | `/usr/lib64/libibverbs/libbnxt_re-rdmav34.so` (Red Hat images) |

> **The host file is `rdmav57`, the container path is `rdmav34`** — this is correct. The
> hostPath volume mount maps a host file to an arbitrary container path. libibverbs inside
> the container finds and loads whatever `.so` is at its expected path.

---

## Filled-In Examples

### Red Hat Image on OpenShift (primary use case — `nxe_linux_237.1.148.0` + KMM)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vllm-rhel
spec:
  containers:
    - name: vllm
      image: registry.redhat.io/rhaii/vllm-rocm-rhel9:3.4.1
      volumeMounts:
        - name: libbnxt-re
          mountPath: /usr/lib64/libibverbs/libbnxt_re-rdmav34.so
          readOnly: true
  volumes:
    - name: libbnxt-re
      hostPath:
        path: /opt/broadcom/lib64/libbnxt_re-rdmav57.so
        type: File
```

### ROCm Image (upstream, Ubuntu)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vllm-rocm
spec:
  containers:
    - name: vllm
      image: rocm/vllm-dev:main_20250529
      volumeMounts:
        - name: libbnxt-re
          mountPath: /usr/local/lib/x86_64-linux-gnu/libbnxt_re-rdmav34.so
          readOnly: true
  volumes:
    - name: libbnxt-re
      hostPath:
        path: /opt/broadcom/lib64/libbnxt_re-rdmav57.so   # adjust to your KMM install path
        type: File
```

### CUDA Image (upstream, Ubuntu)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vllm-cuda
spec:
  containers:
    - name: vllm
      image: vllm/vllm-openai:v0.26.0-cu129-ubuntu2404
      volumeMounts:
        - name: libbnxt-re
          mountPath: /usr/lib/x86_64-linux-gnu/libibverbs/libbnxt_re-rdmav34.so
          readOnly: true
  volumes:
    - name: libbnxt-re
      hostPath:
        path: /opt/broadcom/lib64/libbnxt_re-rdmav57.so   # adjust to your KMM install path
        type: File
```

---

## Notes

- **`rdmav57` on host vs `rdmav34` in container**: The `nxe_linux_237.1.148.0` package ships
  `libbnxt_re-rdmav57.so` (built for RHEL 9.7 / rdma-core ~v57). Container images use the inbox
  `libbnxt_re-rdmav34.so` (rdma-core ≤54 provider ABI naming). The volume mount maps the host
  file to the container path — libibverbs loads whatever `.so` lives at its configured provider
  path. The `rdmav` suffix is a build-time artifact, not a runtime compatibility gate for the
  `dlopen()` load path.

- **Single KMM Module CR handles both kernel modules and userspace library.**
  The module-loader pod copies `libbnxt_re-rdmav57.so` to `/opt/broadcom/lib64/` on the host
  before loading the kernel modules. No separate DaemonSet needed. This works on both RHCOS
  and RHEL workers since `/opt/` is writable on both.

- **Version alignment**: The mounted `.so` must be from the **same Broadcom release train**
  as the loaded `bnxt_re.ko` kernel module. Both come from `nxe_linux_237.1.148.0.tar.gz`
  (component version 237.1.137.0). A cross-train mismatch (e.g. host KMD from 237.x, container
  UMD from 235.x or inbox) causes provider uABI rejection and RDMA verbs failures.

- **Kernel portability**: Building from DKMS source via KMM makes the solution independent of
  the specific RHCOS kernel version. KMM's builder pod compiles against whatever kernel the
  node is running. The userspace library (`libbnxt_re-rdmav57.so`) is a prebuilt binary that is
  kernel-version-independent — the same `.so` works across RHEL 9.x hosts.

- **`type: File`**: The `hostPath` type is set to `File` because we are mounting a single
  shared object, not a directory. Kubernetes will fail pod scheduling if the file does not
  exist on the node.

- **Read-only mount**: The library is mounted read-only since the container should never
  modify it.

- **Node Feature Discovery (NFD)**: Consider using NFD labels (e.g.
  `feature.node.kubernetes.io/pci-14e4.present`) in pod `nodeSelector` to ensure vLLM pods
  only schedule on nodes with Broadcom NICs that have the driver installed.
