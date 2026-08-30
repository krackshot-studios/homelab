# Homelab Talos + Kubernetes Build Archive

**Project:** Homelab  
**Scope:** Proxmox-hosted Talos Linux Kubernetes cluster with Cilium, HA control plane, dedicated GPU worker, and reproducible Git/SOPS configuration  
**Status at end of session:** Core cluster operational; GPU passthrough operational after a physical cold boot; GPU reset/reuse remains an operational caveat  
**Primary workstation:** Mac mini M4, 16 GB RAM, 1 TB storage

---

## 1. Executive Summary

This build produced a seven-node Talos Linux Kubernetes cluster running as VMs across three Proxmox hosts. The control plane is distributed one node per physical host for etcd quorum, Cilium provides pod networking, and the Kubernetes API is exposed through a Talos Layer 2 VIP at `192.168.40.100`.

The cluster currently consists of three dedicated control-plane nodes, three general-purpose workers, and one GPU worker with an AMD Radeon RX 6750 XT passed through from `pve-01`.

The main build path succeeded:

- Talos v1.13.9 installed on all nodes.
- Kubernetes v1.36.3 bootstrapped successfully.
- Three-member etcd cluster is healthy.
- Talos Layer 2 VIP at `192.168.40.100` is active.
- Cilium is installed and all control-plane nodes became `Ready`.
- General workers joined successfully.
- GPU worker joined successfully.
- RX 6750 XT is visible inside Talos.
- `amdgpu` is loaded.
- `/dev/dri/card0` and `/dev/dri/renderD128` exist inside the GPU worker.

The only unresolved engineering issue is the RX 6750 XT reset/reuse behavior under VFIO. A true physical cold boot of `pve-01` followed by the first VM start produced a healthy GPU worker, but prior guest reboot / stop-start cycles caused the VM to hang before networking. Treat the GPU node as requiring special maintenance handling until that reset behavior is fully solved.

---

## 2. Physical Homelab Layout

### Workstation

- **Mac mini M4**
- 16 GB memory
- 1 TB storage
- Used as the main administration workstation and Git authoring machine

### Proxmox hosts

| Host | IP | Hardware | Primary Talos VMs |
|---|---|---|---|
| `pve-01` | `192.168.40.101` | Ryzen 9 5900X, 64 GB RAM, 1 TB, RX 6750 XT 12 GB | `talos-cp01`, `talos-w01`, `talos-w04-gpu` |
| `pve-02` | `192.168.40.102` | HP EliteDesk Mini G6, 32 GB RAM, 500 GB | `talos-cp02`, `talos-w02` |
| `pve-03` | `192.168.40.103` | HP EliteDesk Mini G6, 32 GB RAM, 500 GB | `talos-cp03`, `talos-w03` |

HP CPUs:

- i5-10500T
- i7-10700T

### Storage

- **TrueNAS:** `192.168.40.105`
- Pool: `tank`
- Datasets include `dev` and `media`

Storage integration into Kubernetes was intentionally deferred until the cluster substrate was stable.

---

## 3. Network Design

### Current LAN

- Network: `192.168.40.0/24`
- Gateway: `192.168.40.1`
- DNS: `192.168.40.1`
- Kubernetes API VIP: `192.168.40.100`
- Talos node interface: `ens18`

The cluster currently remains on the existing LAN instead of using a Proxmox SDN VXLAN overlay. Real VLAN segmentation and Proxmox SDN can be introduced later.

### Talos addressing

DHCP reservations provide stable node addresses. Static addresses were intentionally not duplicated in Talos machine configuration.

| Node | IP |
|---|---|
| `talos-cp01` | `192.168.40.111` |
| `talos-cp02` | `192.168.40.112` |
| `talos-cp03` | `192.168.40.113` |
| `talos-w01` | `192.168.40.121` |
| `talos-w02` | `192.168.40.122` |
| `talos-w03` | `192.168.40.123` |
| `talos-w04-gpu` | `192.168.40.124` |

The network also advertises IPv6, including global and ULA ranges. Future zero-trust policy must account for IPv6 instead of enforcing policy only on IPv4.

---

## 4. Talos VM Layout

| VMID | Proxmox Host | Talos Node | CPU | RAM | Disk | Role |
|---:|---|---|---:|---:|---:|---|
| 102 | `pve-01` | `talos-cp01` | 2 vCPU | 4 GB | 32 GB | control plane |
| 101 | `pve-02` | `talos-cp02` | 2 vCPU | 4 GB | 32 GB | control plane |
| 100 | `pve-03` | `talos-cp03` | 2 vCPU | 4 GB | 32 GB | control plane |
| 103 | `pve-01` | `talos-w01` | 8 vCPU | 20 GB | 200 GB | worker |
| 104 | `pve-02` | `talos-w02` | 10 vCPU | 24 GB | 200 GB | worker |
| 105 | `pve-03` | `talos-w03` | 8 vCPU | 24 GB | 200 GB | worker |
| 106 | `pve-01` | `talos-w04-gpu` | 10 vCPU | 32 GB | 250 GB | GPU worker |

### Common Proxmox VM baseline

All seven VMs were normalized to:

- BIOS: OVMF
- Machine: q35
- CPU type: host
- 1 socket
- NUMA disabled
- EFI disk, 4 MB
- Secure Boot disabled
- `scsihw: virtio-scsi-pci`
- IO thread disabled
- VirtIO NIC
- `balloon: 0`
- `agent: 1`

During installation the boot order was:

```text
ide2;scsi0;net0
```

Before applying Talos configs it was changed to:

```text
scsi0;ide2;net0
```

After the cluster was proven healthy, the intended cleanup is to detach the Talos ISOs and use disk-only boot.

---

## 5. Talos and Kubernetes Versions

- **Talos:** v1.13.9
- **Linux:** 6.18.44-talos
- **Kubernetes:** v1.36.3
- **Container runtime observed:** containerd 2.2.7

### Talos Image Factory schematics

#### Core nodes

```text
ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
```

Installer:

```text
factory.talos.dev/metal-installer/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515:v1.13.9
```

#### GPU node

```text
aeec243e3a4c2a14f9ba74b1a8c7662f03eea658a7ea5f1c26fdd491280c88f8
```

Installer:

```text
factory.talos.dev/metal-installer/aeec243e3a4c2a14f9ba74b1a8c7662f03eea658a7ea5f1c26fdd491280c88f8:v1.13.9
```

GPU image extensions included AMD microcode, `amdgpu`, qemu guest agent, and the Talos schematic metadata.

---

## 6. Git Repository and Secrets Management

Repository:

```text
~/Projects/homelab
```

Structure used during the build:

```text
homelab/
├── docs/
├── talos/
│   ├── patches/
│   ├── generated/
│   └── secrets/
└── kubernetes/
    ├── bootstrap/
    ├── infrastructure/
    ├── platform/
    └── apps/
```

### Important repository rule

Generated Talos configs contain PKI and must not be committed.

The plaintext secret file is also not committed. Only the SOPS-encrypted secret should be tracked.

Representative `.gitignore` intent:

```gitignore
# plaintext Talos secrets
talos/secrets/*
!talos/secrets/*.sops.yaml

talos/generated/
talosconfig
*.talosconfig
kubeconfig
*.kubeconfig
*.agekey
keys.txt
```

### SOPS / age

- age private key stored outside the repo under the user's local SOPS configuration
- private key permissions set to 600
- `SOPS_AGE_KEY_FILE` was exported and persisted in zsh because macOS SOPS looked in a different default path
- Talos secrets were generated once, encrypted with SOPS, and plaintext removed

The private age key must never be committed.

---

## 7. Talos Configuration Design

### Cluster networking patch

Cilium was chosen from day one, so Talos Flannel was disabled:

```yaml
cluster:
  network:
    cni:
      name: none
```

### Control-plane VIP

The Kubernetes API endpoint is:

```text
https://192.168.40.100:6443
```

Each control-plane config includes:

```yaml
apiVersion: v1alpha1
kind: Layer2VIPConfig
name: 192.168.40.100
link: ens18
```

Talos API administration continues to use the real control-plane addresses rather than the Kubernetes VIP.

### Hostnames

Talos 1.13 required `auto: off` when a static hostname was supplied:

```yaml
apiVersion: v1alpha1
kind: HostnameConfig
auto: off
hostname: talos-cp01
```

Equivalent hostname patches were generated for every node.

### GPU worker labels and taint

```yaml
machine:
  nodeLabels:
    homelab.io/gpu: amd
    homelab.io/gpu-model: rx6750xt
  kubelet:
    extraConfig:
      registerWithTaints:
        - key: homelab.io/gpu
          value: amd
          effect: NoSchedule
```

This keeps ordinary workloads from drifting onto the GPU worker.

---

## 8. Configuration Validation

All seven generated node configs passed strict Talos metal validation.

Semantic audit confirmed:

- control planes use `machine.type: controlplane`
- workers use `machine.type: worker`
- install disk is `/dev/sda`
- normal nodes use the core Talos schematic
- GPU node uses the GPU schematic
- CNI is `none`
- only control planes contain the `.100` VIP
- GPU worker contains the intended labels and taint

---

## 9. Cluster Bootstrap Sequence

### Control planes

Talos configs were first applied to:

- `192.168.40.111`
- `192.168.40.112`
- `192.168.40.113`

After reboot, all three answered authenticated Talos API calls.

### etcd bootstrap

etcd was bootstrapped exactly once on `talos-cp01`.

Final membership:

- `talos-cp01`
- `talos-cp02`
- `talos-cp03`

All three are full voting members, not learners.

Observed healthy state included identical Raft indexes across the cluster and no etcd errors.

During later host maintenance, leadership successfully moved to `talos-cp03`, demonstrating control-plane failover while preserving quorum.

### VIP

After etcd bootstrap, `192.168.40.100` became reachable and ARP resolved to a control-plane NIC.

This proved Talos Layer 2 VIP ownership was active.

---

## 10. Cilium Bootstrap

Cilium was installed after the control plane and API VIP were functional.

Design choice:

- Cilium provides CNI and network policy
- kube-proxy remains enabled initially
- kube-proxy replacement can be explored later as a deliberate learning step

Talos-specific Cilium settings included:

```yaml
ipam:
  mode: kubernetes

kubeProxyReplacement: false

cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
```

Additional Cilium agent capabilities were supplied to match Talos requirements.

After installation:

- all three control-plane nodes became `Ready`
- CoreDNS became healthy
- Cilium DaemonSet reached 3/3 on the control plane
- Cilium operator reached 2/2

Control-plane PodCIDRs were allocated from `10.244.0.0/16`:

```text
talos-cp01  10.244.0.0/24
talos-cp02  10.244.1.0/24
talos-cp03  10.244.2.0/24
```

---

## 11. Worker Join

The three general workers were applied and joined successfully:

```text
talos-w01  192.168.40.121
talos-w02  192.168.40.122
talos-w03  192.168.40.123
```

Observed worker PodCIDRs included:

```text
talos-w01  10.244.7.0/24
talos-w02  10.244.3.0/24
talos-w03  10.244.5.0/24
```

The exact allocation order is not important; uniqueness is.

---

## 12. GPU Passthrough Design

### Physical GPU

- AMD Radeon RX 6750 XT
- Navi 22
- PCI GPU function: `1002:73df`
- HDMI/DP audio function: `1002:ab28`

### Proxmox assignment

VM 106 uses:

```text
hostpci0: 0000:2d:00,pcie=1
```

Passing the entire multifunction device was necessary. Passing only `2d:00.0` produced a reset dependency failure because the audio function participates in the same device reset path.

### IOMMU

The GPU and audio functions were in clean IOMMU groups. ACS override was not required.

---

## 13. GPU Failure and Troubleshooting Timeline

The GPU worker initially failed after installation/reboot:

- Proxmox reported VM 106 as running
- Talos never brought up networking
- qemu guest agent was unavailable
- Kubernetes showed `talos-w04-gpu` as `NotReady`
- removing the GPU from VM 106 allowed Talos to boot immediately and the node became `Ready`

This clean A/B test isolated the problem to GPU passthrough/reset behavior rather than Talos, networking, Cilium, or Kubernetes.

### Important host observations

The RX 6750 XT was correctly owned by `vfio-pci`.

Host logs showed reset activity including:

```text
vfio_bar_restore: reset recovery - restoring BARs
```

and later:

```text
error writing '1' to '/sys/bus/pci/devices/0000:2d:00.0/reset': Inappropriate ioctl for device
failed to reset PCI device '0000:2d:00.0', but trying to continue as not all devices need a reset
```

The device was not stuck in D3cold. It reported power state `D0`, and kernel logs often reported `reset done`, indicating a more subtle reset/reinitialization issue.

### VFIO early binding cleanup

Initial VFIO config reserved the GPU but not its HDMI audio function. The audio function was therefore initially claimed by `snd_hda_intel` on the host and moved to VFIO later.

The desired final VFIO configuration is:

```text
options vfio-pci ids=1002:73df,1002:ab28 disable_vga=1
```

A temporary older config also included:

```text
1022:1487
```

which is the Ryzen/Matisse motherboard audio controller, not part of the RX 6750 XT. It should not remain in the permanent VFIO ID list unless passthrough of that device is intentional.

**Verify the live file before the next planned host reboot:**

```bash
cat /etc/modprobe.d/vfio.conf
```

The intended contents are only:

```text
options vfio-pci ids=1002:73df,1002:ab28 disable_vga=1
```

---

## 14. Cold-Boot GPU Recovery Test

A true physical power cycle of `pve-01` was performed so the RX 6750 XT lost power completely.

After cold boot, both GPU functions were immediately owned by VFIO:

```text
2d:00.0 → vfio-pci
2d:00.1 → vfio-pci
```

VM 106 was then started once from this clean hardware state.

Result: success.

### Guest-side proof

Talos reached the network:

```text
192.168.40.124 reachable
Talos v1.13.9 server responding
```

Kubernetes reported:

```text
talos-w04-gpu  Ready,SchedulingDisabled
```

Talos PCI inventory showed:

```text
0000:01:00.0  Navi 22 / RX 6700/6700 XT/6750 XT family
0000:01:00.1  AMD HDMI/DP Audio Controller
```

Kernel module:

```text
amdgpu  Live
```

DRM devices:

```text
/dev/dri/card0
/dev/dri/renderD128
```

This proves the complete passthrough path is functional from a fresh physical GPU state.

---

## 15. GPU Operational Caveat

Current evidence strongly suggests the RX 6750 XT has unreliable VFIO reset/reuse behavior across VM reboot or repeated stop/start cycles.

Known-good path:

```text
physical cold boot of pve-01
        ↓
first VM 106 start
        ↓
GPU works
```

Known-problematic path:

```text
GPU used by Talos guest
        ↓
VM reboot / stop-start
        ↓
GPU reset/reassignment
        ↓
guest can hang before networking
```

Operational recommendation until this is solved:

- avoid unnecessary reboots of `talos-w04-gpu`
- plan GPU-node maintenance around a physical cold cycle of `pve-01` if required
- keep the GPU worker cordoned until the Kubernetes GPU workload/device layer is intentionally configured
- do not install unrelated reset modules without confirming Navi 22 support

---

## 16. End-of-Session Cluster State

Confirmed healthy:

- 3 Talos control planes
- 3 general workers
- 1 GPU worker
- 3-member etcd quorum
- Kubernetes API VIP at `.100`
- Kubernetes v1.36.3
- Cilium networking
- CoreDNS
- kube-proxy
- GPU PCI passthrough
- `amdgpu`
- `/dev/dri/renderD128`

### Scheduling state to verify

During GPU maintenance, the following nodes were cordoned:

- `talos-cp01`
- `talos-w01`
- `talos-w04-gpu`

The intended final state is:

- `talos-cp01`: uncordoned
- `talos-w01`: uncordoned
- `talos-w04-gpu`: remain cordoned until GPU workload integration is complete

Verify with:

```bash
kubectl get nodes
```

If necessary:

```bash
kubectl uncordon talos-cp01 talos-w01
```

---

## 17. Post-Build Cleanup Checklist

### Proxmox

Detach installation media from all Talos VMs and set disk-only boot.

Example:

```bash
qm set <VMID> --ide2 none,media=cdrom
qm set <VMID> --boot 'order=scsi0'
```

### VFIO

Verify:

```bash
cat /etc/modprobe.d/vfio.conf
```

Desired:

```text
options vfio-pci ids=1002:73df,1002:ab28 disable_vga=1
```

At the next planned cold boot, the unrelated Matisse audio controller `1022:1487` should return to its normal host audio driver if it has been removed from the VFIO ID list.

### Kubernetes

Verify:

```bash
kubectl get nodes -o wide
kubectl -n kube-system get daemonset cilium
kubectl get pods -A
```

Expected broad state:

- all seven nodes registered
- normal nodes `Ready`
- GPU worker `Ready,SchedulingDisabled` until GPU scheduling is intentionally enabled
- Cilium desired/current/ready counts all match

---

## 18. Architectural Decisions Captured

The build intentionally chose:

- Talos Linux for immutable Kubernetes nodes
- one control plane per physical Proxmox host
- dedicated control-plane VMs
- Cilium from day one
- kube-proxy retained initially for learning and staged complexity
- DHCP reservations instead of duplicating static addressing inside Talos
- Git as the configuration source of truth
- SOPS + age for Talos secrets
- generated Talos configs excluded from Git
- dedicated GPU worker with taint and labels
- no Proxmox HA initially because Kubernetes provides workload HA
- current LAN first, VLAN/zero-trust segmentation later

---

## 19. Recommended Next Phase

The substrate is now complete enough to begin platform engineering.

Recommended sequence:

1. Finish Proxmox ISO and boot-order cleanup.
2. Verify final VFIO config and scheduling state.
3. Add Kubernetes-side GPU device exposure for `/dev/dri/renderD128`.
4. Decide the LLM runtime path for RX 6750 XT, likely Vulkan/llama.cpp first, with ROCm treated carefully because RDNA2 support is not a simple mainstream path.
5. Add Cilium Hubble and observability.
6. Begin default-deny NetworkPolicy design.
7. Build IPv4 + IPv6 zero-trust policy deliberately.
8. Add ingress / Gateway API.
9. Add storage integration from TrueNAS.
10. Introduce GitOps ownership of infrastructure and platform resources.
11. Later evaluate kube-proxy replacement, BGP, real VLAN segmentation, and Proxmox SDN.

---

## 20. Final Build Snapshot

```text
Mac mini M4
    │
    ├── talosctl / kubectl / helm
    ├── Git
    └── SOPS + age

192.168.40.0/24
    │
    ├── VIP 192.168.40.100
    │
    ├── pve-01 192.168.40.101
    │    ├── talos-cp01 192.168.40.111
    │    ├── talos-w01  192.168.40.121
    │    └── talos-w04-gpu 192.168.40.124
    │         └── RX 6750 XT / amdgpu / /dev/dri/renderD128
    │
    ├── pve-02 192.168.40.102
    │    ├── talos-cp02 192.168.40.112
    │    └── talos-w02  192.168.40.122
    │
    ├── pve-03 192.168.40.103
    │    ├── talos-cp03 192.168.40.113
    │    └── talos-w03  192.168.40.123
    │
    └── TrueNAS 192.168.40.105

Talos v1.13.9
Kubernetes v1.36.3
Cilium CNI
3-member etcd
Layer 2 API VIP
```

---

## 21. Handoff Note

This document is the clean handoff point for the first major Homelab build phase: **Proxmox → Talos → HA Kubernetes → Cilium → GPU passthrough**.

Future work should treat this cluster as the stable substrate and build the platform layer above it rather than revisiting foundational VM layout unless a specific issue requires it.

---

## 22. Kubernetes GPU Integration

After validating raw passthrough inside Talos, the GPU was integrated into Kubernetes as a schedulable extended resource.

### Host-visible GPU devices

Inside `talos-w04-gpu`:

```text
/dev/dri/card0
/dev/dri/renderD128
/dev/kfd
```

A privileged temporary smoke-test pod confirmed Mesa RADV could enumerate the passed-through card:

```text
deviceName = AMD Radeon RX 6750 XT (RADV NAVI22)
driverName = radv
vendorID = 0x1002
deviceID = 0x73df
```

This proved the path:

```text
RX 6750 XT
  -> Proxmox VFIO
  -> Talos amdgpu
  -> /dev/dri + /dev/kfd
  -> Mesa RADV
  -> Vulkan
```

### Generic device plugin

A generic Kubernetes device plugin was deployed only to the GPU worker. It groups:

```text
/dev/dri/renderD128
/dev/kfd
```

into one logical Kubernetes resource:

```text
homelab.io/amd-gpu
```

Kubernetes reported:

```json
{
  "capacity": "1",
  "allocatable": "1"
}
```

GPU workloads can therefore request:

```yaml
resources:
  requests:
    homelab.io/amd-gpu: 1
  limits:
    homelab.io/amd-gpu: 1
```

The permanent Talos GPU taint remains:

```text
homelab.io/gpu=amd:NoSchedule
```

so GPU workloads must also include:

```yaml
tolerations:
  - key: homelab.io/gpu
    operator: Equal
    value: amd
    effect: NoSchedule
```

This is intentional. The extended resource identifies where the accelerator exists; the taint/toleration pair is the admission gate for workloads permitted to use the dedicated node.

### Pod Security design

Two namespaces were used for GPU testing:

- `gpu-workloads`: normal workload namespace; baseline enforcement with restricted-policy warnings/auditing during transition.
- `gpu-debug`: temporary privileged break-glass namespace used only for raw host-device validation.

The privileged debug namespace should be deleted after testing. Production GPU workloads should not require `privileged: true` or raw `hostPath` mounts when using the device plugin.

---

## 23. GPU / Pod Scheduling Troubleshooting Guide

### Symptom: GPU smoke test rejected with PodSecurity `baseline:latest`

Observed error:

```text
pods "amd-vulkan-smoke-test" is forbidden:
violates PodSecurity "baseline:latest":
hostPath volumes ... privileged ...
```

Cause:

The diagnostic pod requested `privileged: true` and `hostPath` mounts for `/dev/dri` and `/dev/kfd`. Baseline Pod Security correctly rejected it.

Resolution:

Create a temporary privileged debug namespace for the one-off hardware probe, run the test there, capture results, then delete the namespace. Do not weaken the permanent workload namespace merely to make debugging easier.

### Symptom: `vulkaninfo` prints DISPLAY / XDG warnings

Observed:

```text
'DISPLAY' environment variable not set... skipping surface info
error: XDG_RUNTIME_DIR is invalid or not set
```

Cause:

The container is headless and has no graphical desktop session.

Interpretation:

These warnings are harmless for compute validation when `vulkaninfo --summary` successfully enumerates the physical GPU. The decisive result is the presence of:

```text
AMD Radeon RX 6750 XT (RADV NAVI22)
```

### Symptom: device-plugin-backed pod remains `Pending`

Observed scheduler event:

```text
0/7 nodes are available:
3 Insufficient homelab.io/amd-gpu,
4 node(s) had untolerated taint(s).
```

Cause:

The pod correctly requested the extended GPU resource, but did not tolerate the GPU worker's permanent taint.

Important lesson:

An extended resource request does not bypass Kubernetes taints.

Required scheduling contract:

```yaml
resources:
  requests:
    homelab.io/amd-gpu: 1
  limits:
    homelab.io/amd-gpu: 1

tolerations:
  - key: homelab.io/gpu
    operator: Equal
    value: amd
    effect: NoSchedule
```

No node selector is required because the extended-resource request naturally constrains scheduling to a node advertising `homelab.io/amd-gpu`.

### Symptom: `kubectl logs` is empty for the smoke-test pod

Cause:

The pod has never been scheduled or started. A Pending pod has no container logs yet.

First diagnostic command:

```bash
kubectl describe pod -n gpu-workloads amd-vulkan-smoke-test |
  sed -n '/Events:/,$p'
```

Scheduler events are the authoritative first stop for Pending pods.

### Symptom: GPU worker boots without GPU but hangs with GPU attached

Diagnosis path established during this build:

```text
GPU detached -> Talos boots -> Kubernetes Ready
GPU attached -> VM process runs -> Talos networking never appears
```

This isolates the problem to GPU passthrough/reset behavior rather than Kubernetes, Cilium, DHCP, kubelet, or the Talos disk install.

For this RX 6750 XT, a true physical cold power cycle of `pve-01` restored a usable first-start state. Avoid repeatedly stop/starting the GPU VM while debugging reset behavior because it destroys the clean test state.

---

## 24. Platform Engineering Roadmap

The substrate and GPU integration are now complete enough to proceed upward through observability and zero-trust controls.

Recommended order:

1. Metrics Server for the Kubernetes resource metrics API and `kubectl top`.
2. kube-prometheus-stack for long-term metrics, Prometheus Operator, Grafana, Alertmanager, kube-state-metrics, and node telemetry.
3. Enable Cilium agent/operator Prometheus metrics.
4. Enable Hubble Relay, Hubble UI, and Hubble flow metrics through the Git-tracked Cilium Helm values.
5. Observe real traffic before adding deny policy.
6. Apply namespace-scoped default-deny NetworkPolicies to application namespaces first.
7. Add explicit DNS and service-to-service allowances.
8. Introduce CiliumNetworkPolicy for richer L3/L4/L7 and FQDN controls.
9. Expand policy design to IPv6 as well as IPv4.
10. Add persistent storage for observability after TrueNAS CSI integration.

The guiding principle is:

```text
measure -> observe -> restrict -> verify
```

Do not begin cluster-wide zero-trust enforcement while blind to traffic flows.
