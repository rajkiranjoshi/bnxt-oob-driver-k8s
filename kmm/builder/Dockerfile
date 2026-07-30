# Source image containing the Broadcom DKMS source and userspace library.
# This image does NOT compile anything — it packages the source + prebuilt UMD
# for KMM's in-cluster build to use. Build once, use across all kernel versions.
#
# Usage:
#   podman build -t <registry>/broadcom-bnxt-src:237.1.137.0 .
#   podman push <registry>/broadcom-bnxt-src:237.1.137.0

FROM registry.access.redhat.com/ubi9/ubi-micro:latest

# Broadcom DKMS source (extracted from nxe_linux_237.1.148.0.tar.gz)
COPY bnxt_en-1.10.3-237.1.137.0/ /src/bnxt_en/
COPY bnxt_re-237.1.137.0/ /src/bnxt_re/

# Userspace RDMA provider library (extracted from libbnxt_re RPM, kernel-independent)
COPY libbnxt_re-rdmav57.so /src/lib/libbnxt_re-rdmav57.so
