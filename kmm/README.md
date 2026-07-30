# KMM Artifacts for Broadcom NIC OOB Driver Deployment

This directory contains Kubernetes/OpenShift manifests and a Dockerfile for deploying
Broadcom's out-of-tree `bnxt_en` / `bnxt_re` drivers and the matching userspace RDMA
provider (`libbnxt_re`) on OpenShift — all via a single KMM Module CR.

> **Not on OpenShift?** KMM works on vanilla Kubernetes too (GKE, AKS, EKS, bare-metal).
> See [Adapting for Vanilla Kubernetes](#adapting-for-vanilla-kubernetes-non-openshift) at
> the end of this document for what to change.

## How It Works

KMM builds kernel modules **in-cluster, automatically** — the same pattern NVIDIA Network
Operator uses for DOCA/OFED drivers. No pre-built kernel module images are maintained.

1. **Source image** — built once, contains the Broadcom DKMS source + `libbnxt_re-rdmav57.so`.
2. **KMM detects the node's kernel** and triggers an in-cluster build using the Driver Toolkit
   (DTK) image that matches that kernel. The compiled `.ko` files + the `.so` are packed into
   a single image, pushed to your registry, and cached.
3. **KMM's module-loader pod** runs on each matching node: copies `libbnxt_re-rdmav57.so` to
   `/opt/broadcom/lib64/` on the host, then loads the kernel modules.
4. **On kernel upgrades**, KMM detects the new kernel and automatically rebuilds. No manual
   intervention. The `.so` is kernel-independent and doesn't change.

One source image. One Module CR. No DaemonSet. No image matrix.

## Directory Structure

```
kmm/
├── builder/
│   └── Dockerfile              # Source image: packages DKMS source + libbnxt_re (no compilation)
├── build-configmap.yaml        # ConfigMap with the Dockerfile KMM uses for in-cluster builds
├── module-cr.yaml              # KMM Module CR (auto-build + load + deploy UMD to host)
├── vllm-pod-example.yaml       # Example vLLM pod with the host-mounted UMD
└── README.md                   # This file
```

## Usage

### Prerequisites

1. Obtain `nxe_linux_237.1.148.0.tar.gz` from Broadcom Support driver downloads.
2. Extract the DKMS source and userspace library:
   ```bash
   # Extract bnxt_en source
   tar xzf nxe_linux_237.1.148.0.tar.gz bnxt_en/bnxt_en-1.10.3-237.1.137.0.tar.gz
   tar xzf bnxt_en/bnxt_en-1.10.3-237.1.137.0.tar.gz -C kmm/builder/

   # Extract bnxt_re DKMS source
   tar xzf nxe_linux_237.1.148.0.tar.gz bnxt_re/dkms/bnxt_re-237.1.137.0-1dkms.noarch.rpm
   cd kmm/builder && rpm2cpio ../../bnxt_re/dkms/bnxt_re-237.1.137.0-1dkms.noarch.rpm | cpio -idm
   mv usr/src/bnxt_re-237.1.137.0 .

   # Extract libbnxt_re userspace library
   tar xzf nxe_linux_237.1.148.0.tar.gz bnxt_rocelib/rpm/rhel/rhel9.7/libbnxt_re-237.1.137.0-rhel9u7.x86_64.rpm
   cd kmm/builder && rpm2cpio ../../bnxt_rocelib/rpm/rhel/rhel9.7/libbnxt_re-237.1.137.0-rhel9u7.x86_64.rpm | cpio -idm
   cp usr/lib64/libbnxt_re-rdmav57.so .
   ```

### Step 1: Build and Push the Source Image (one-time)

This image contains the Broadcom source + userspace library — no compilation, no kernel
dependency. Build once, use across all kernel versions.

```bash
cd kmm/builder/
podman build -t <registry>/broadcom-bnxt-src:237.1.137.0 .
podman push <registry>/broadcom-bnxt-src:237.1.137.0
```

### Step 2: Deploy the KMM Build ConfigMap and Module CR

```bash
# Edit module-cr.yaml and build-configmap.yaml to set your <registry>
kubectl apply -f kmm/build-configmap.yaml
kubectl apply -f kmm/module-cr.yaml
```

KMM will now automatically:
- Detect node kernels
- Build `bnxt_en.ko` + `bnxt_re.ko` against each kernel version
- Push the built image to `<registry>/broadcom-bnxt-kmm:237.1.137.0-<kernel-version>`
- On each node: copy `libbnxt_re-rdmav57.so` to `/opt/broadcom/lib64/`, then load the modules
- Rebuild when kernels change (OCP upgrades)

### Step 3: Deploy vLLM with Host-Mounted UMD

```bash
kubectl apply -f kmm/vllm-pod-example.yaml
```

## Verification

### Check Module-Loader Pod Logs

Once the KMM Module CR is applied, check the module-loader pod logs for the deployment
confirmation message:

```bash
kubectl logs -n openshift-kmm -l kmm.node.kubernetes.io/module.name=broadcom-bnxt
```

Expected output on successful deployment:

```
==========================================
 Broadcom OOB driver deployed successfully
==========================================

 Kernel modules loaded: bnxt_en, bnxt_re
 Userspace library:     /opt/broadcom/lib64/libbnxt_re-rdmav57.so

 To mount the UMD into vLLM containers, use:

   volumes:
     - name: libbnxt-re
       hostPath:
         path: /opt/broadcom/lib64/libbnxt_re-rdmav57.so
         type: File

==========================================
```

### Verify from the Host

```bash
# Check kernel modules are loaded:
kubectl debug node/<node> -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  chroot /host lsmod | grep bnxt

# Check userspace library is on host:
kubectl debug node/<node> -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  chroot /host ls -la /opt/broadcom/lib64/libbnxt_re-rdmav57.so

# Check RDMA devices are visible:
kubectl debug node/<node> -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  chroot /host ibv_devices
```

## What Happens on Kernel Upgrades

When OCP upgrades and nodes get a new kernel:
1. KMM detects the kernel version change on the node
2. KMM triggers an in-cluster build using the new DTK (matching the new kernel)
3. The source image (`broadcom-bnxt-src:237.1.137.0`) is unchanged — same source compiles
   against new headers, same `.so` gets copied
4. Built image is pushed and cached at `<registry>/broadcom-bnxt-kmm:237.1.137.0-<new-kernel>`
5. KMM's module-loader pod copies the `.so` and loads the freshly built modules

No manual intervention required. The only time you touch this is when updating the
Broadcom driver version itself (new `nxe_linux_*.tar.gz` from Broadcom Support).

## Adapting for Vanilla Kubernetes (non-OpenShift)

KMM is a [kubernetes-sigs project](https://github.com/kubernetes-sigs/kernel-module-management)
and works on any Kubernetes cluster (GKE, AKS, EKS, bare-metal). The manifests in this
directory are written for OpenShift but can be adapted with these changes:

| What to change | OpenShift (current) | Vanilla K8s |
|---|---|---|
| **Namespace** | `openshift-kmm` | `kmm-operator-system` |
| **`DTK_AUTO` build arg** | Auto-resolved by KMM to the matching Driver Toolkit image | Not available — replace with a builder image that installs kernel headers (see below) |
| **Kernel regexp** | `'^.*\.el9.*$'` (RHEL 9 kernels) | Match your distro (e.g. `'^.*generic$'` for Ubuntu, `'^.*$'` for any) |
| **KMM install** | Via OLM / OperatorHub | `kubectl apply -k https://github.com/kubernetes-sigs/kernel-module-management/config/default` (requires cert-manager) |

### Builder Dockerfile for Ubuntu Nodes

Replace the `FROM ${DTK_AUTO}` stage in `build-configmap.yaml` with:

```dockerfile
FROM ubuntu:22.04 AS builder
ARG KERNEL_VERSION

RUN apt-get update && \
    apt-get install -y build-essential linux-headers-${KERNEL_VERSION} && \
    rm -rf /var/lib/apt/lists/*

COPY --from=src /src/bnxt_en/ /src/bnxt_en/
COPY --from=src /src/bnxt_re/ /src/bnxt_re/

WORKDIR /src/bnxt_en
RUN make KVER=${KERNEL_VERSION} && make install KVER=${KERNEL_VERSION}

WORKDIR /src/bnxt_re
RUN make KVER=${KERNEL_VERSION} && make install KVER=${KERNEL_VERSION}
```

Everything else (source image, Module CR structure, host path `/opt/broadcom/lib64/`,
vLLM pod mount) remains the same.
