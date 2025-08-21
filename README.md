# Project Anvil — An Enterprise VM Platform (v0.1)

> Goal: Build a vSphere‑class virtualization platform as an **appliance OS + control plane** on top of Linux/KVM, with HA, live migration, policy backups, multi‑tenant RBAC, API/CLI/UX, and pluggable storage/network. Ship as an ISO for bare‑metal or as an on‑prem VM appliance for lab/dev.

---

## 1) Product Principles

* **Don’t reinvent the hypervisor**: use Linux 6.x + KVM/QEMU (virtio, VFIO, SR‑IOV, nested virt).
* **Single‑purpose, secure appliance**: minimal host OS; immutable root; in‑place upgrades; secure defaults.
* **Batteries included**: HA by default, live migration, snapshots, backups, templates, cloud‑init.
* **API first**: every UI action is API‑backed; publish SDKs & Terraform provider.
* **Pluggable**: storage (Ceph/ZFS/NFS/RBD), network (OVN/OVS/VLAN), identity (OIDC/SAML), backup (S3/PBS).
* **Observability & audit**: OpenTelemetry metrics/traces; structured audit logs; role‑based access.

---

## 2) High‑Level Architecture

```
                   ┌───────────────────────────────────────────────────────────┐
                   │                        Control Plane                      │
                   │  (Kubernetes or systemd services; HA via 3+ managers)     │
                   │                                                           │
                   │  API Gateway  ──►  Core API (gRPC/HTTP)  ──►  Job Runner  │
                   │      │                      │                 │           │
                   │      ▼                      ▼                 ▼           │
                   │  AuthZ/AuthN  ◄── OIDC/SAML/LDAP ──►  RBAC   │           │
                   │  ImageSvc     SnapshotSvc   BackupSvc   NetworkSvc       │
                   │  Inventory    Capacity      Billing*     Web UI           │
                   └───────────────▲───────────────────────────────────────────┘
                                   │ gRPC + mTLS
                         ┌─────────┴──────────────────────────┐
                         │         Host/Node Plane            │
                         │  anvil-node (agent) + libvirt/qemu │
                         │  OVS/OVN  Ceph RBD  cloud-init     │
                         └─────────▲───────────────▲──────────┘
                                   │               │
                             VM Guests        Storage/Network
```

**Core components**

* **anvil-api** (Go/Rust): versioned REST+gRPC; multi‑tenant; RBAC; audit; OpenAPI.
* **anvil-node** (Go/Rust daemon): runs on each host; talks to libvirt/qemu, OVS/OVN; executes idempotent tasks.
* **anvil-scheduler**: places VMs; weighs CPU/RAM/I/O; anti‑affinity; maintenance evacuations.
* **anvil-image**: templates/qcow2 registry; cloud‑init customization; image import (OVF/VHDX/qcow2).
* **anvil-backup**: snapshot, incremental, send to S3/PBS/ZFS; retention policies; immutable options.
* **anvil-net**: virtual networks, DHCP/DNS, security groups, Geneve/VXLAN overlays; VLAN trunking.
* **Web UI**: React+Tailwind; websockets for task streams; shadcn/ui; i18n; dark‑mode.
* **DB**: Postgres (primary/replica); WAL archiving; migrations with goose/flyway.
* **Bus**: NATS or Kafka (task orchestration + events); Redis for cache.

---

## 3) Host OS / Appliance

* Base OS: **Debian 12** (or Fedora CoreOS) with: KVM, QEMU, libvirt, OVS/OVN, cloud‑init, Ceph client.
* Filesystems: **ZFS** (for local datastores & PBS) + **XFS/ext4** for system; cgroups v2, systemd.
* Security: SELinux/AppArmor enforcing; sVirt; QEMU as non‑root; SSH keys only; MFA for web; TPM 2.0.
* Updates: dual‑partition A/B or ostree; staged rollbacks.
* Delivery: **anvil‑host ISO** (bare‑metal) and **anvil‑manager OVA** (mgmt VM).

---

## 4) Networking

* **Modes**: (1) VLAN bridges for simple sites, (2) **OVN** for SDN overlays (Geneve) with distributed routing.
* Services: built‑in DHCP/DNS (kea + CoreDNS), IPAM; Security Groups (stateful), NAT; floating IPs.
* Advanced: SR‑IOV for NIC passthrough; GPU passthrough with VFIO; rate limits; QoS.

---

## 5) Storage

* **Datastores**: Local ZFS, **Ceph RBD**, NFS/iSCSI; thin‑provisioned qcow2 or raw on RBD.
* Features: live migration with shared/replicated storage; consistent snapshots; change‑block tracking for backups.
* Backups: S3/MinIO object‑lock (immutability), Proxmox Backup Server, ZFS send/recv. GFS retention policy.

---

## 6) Identity, RBAC, Tenancy

* Identity: **OIDC** (Entra ID, Okta, Google) and LDAP. Optional SAML.
* Hierarchy: **Org → Projects → Pools**. Quotas at Org/Project.
* RBAC: roles (Viewer, Operator, Admin, Auditor, Billing\*); custom roles via permissions matrix.
* Audit: every action emitted to AuditEvent with actor, resource, old/new, IP, user‑agent.

---

## 7) HA & Reliability

* Host HA: watchdog/fencing via IPMI/Redfish; quorum with etcd/consensus; auto‑restart VMs on surviving nodes.
* Maintenance: drain/evacuate; live migrate; rolling upgrades of hosts and control plane.
* Scheduler: spreads replicas; honors anti‑affinity/affinity; power‑aware placement (optional).

---

## 8) API Surface (v1 draft)

### REST (JSON) examples

* `POST /v1/projects/{id}/vms` → create VM (image, flavor, disks, nics, cloudInit)
* `POST /v1/vms/{id}:migrate` → live migrate
* `POST /v1/vms/{id}:snapshot` → point‑in‑time snapshot
* `POST /v1/vms/{id}:backup` → policy backup now
* `GET  /v1/hosts` / `GET /v1/datastores` / `GET /v1/networks`
* `POST /v1/orgs/{id}/users` → invite user (RBAC role)
* `GET  /v1/tasks/{id}` (long‑running operations with state machine: PENDING/RUNNING/SUCCEEDED/FAILED)

### gRPC proto (snippet)

```proto
service Compute {
  rpc CreateVM(CreateVMRequest) returns (TaskRef);
  rpc LiveMigrate(LiveMigrateRequest) returns (TaskRef);
  rpc SnapshotVM(SnapshotRequest) returns (TaskRef);
  rpc ListHosts(google.protobuf.Empty) returns (ListHostsResponse);
}
```

---

## 9) Data Model (Postgres)

* **hosts**(id, name, az, cpu\_total, cpu\_free, mem\_total, mem\_free, maintenance\_mode, last\_seen)
* **datastores**(id, type, endpoint, capacity\_gb, used\_gb, features\_json)
* **networks**(id, type\[vlan,ovn], vlan\_id, cidr, router\_id, sg\_default)
* **images**(id, name, os, version, source, checksum, size)
* **vms**(id, project\_id, host\_id, state, flavor, image\_id, disks\[], nics\[], metadata jsonb)
* **snapshots**(id, vm\_id, datastore\_id, created\_at)
* **backups**(id, vm\_id, policy\_id, location, immutable\_until)
* **tasks**(id, type, target\_id, status, logs, started\_at, finished\_at)
* **orgs/projects/users/roles/role\_bindings**(...)

Indices on `last_seen`, `state`, `project_id`, `host_id`, `immutable_until`.

---

## 10) Web UI (React)

* Tech: Vite + React + TypeScript + Tailwind + shadcn/ui + TanStack Query + WebSockets.
* Pages: Dashboard, Hosts, Datastores, Networks, Images, VMs (list/detail/console), Backups, Tasks, Audit, Settings.
* Console: noVNC/Spice support; serial console fallback.
* Theming: dark‑first; keyboard shortcuts; command palette.

---

## 11) CLI & IaC

* **`anv` CLI**: `anv vm create`, `anv vm migrate`, `anv host drain`, `anv backup policy apply`.
* **Terraform provider**: resources: `anvil_vm`, `anvil_network`, `anvil_image`, `anvil_backup_policy`.
* **Ansible inventory plugin** to target VM groups by tags.

---

## 12) Image/Guest Experience

* Golden images with cloud‑init.
* virtio drivers bundled for Windows (virtio‑win ISO); QEMU guest agents.
* Metadata injection: SSH keys, scripts, userdata; Secrets via one‑time tokens.

---

## 13) Security

* Rootless QEMU; sVirt + SELinux; cgroups v2 limits; seccomp profiles.
* mTLS (SPIFFE) between control plane and nodes; cert rotation.
* Secrets in HashiCorp Vault or SOPS‑encrypted YAML.
* Immutable backups (S3 Object Lock / PBS verify); monthly restore tests.

---

## 14) Packaging & Install

* ISO built with **Debian live‑build** + Ansible; preflight hardware checks (VT‑x/AMD‑V, IOMMU, NICs).
* First‑boot wizard (TUI/Web): hostname, mgmt IP/VLAN, join cluster, datastore init, OIDC config.
* Upgrades: signed packages; channel (stable / edge); rollbacks.

---

## 15) MVP Scope (90 days)

**Must‑have**

1. Single cluster with 3+ hosts; add/remove hosts.
2. VM lifecycle (create/start/stop/delete); templates; cloud‑init; console.
3. Networking: VLAN bridges + DHCP; security groups (basic).
4. Storage: Local ZFS datastore; live migration with shared NFS; snapshots; backups to S3.
5. Identity: OIDC login; Org/Project; RBAC (Viewer/Operator/Admin).
6. Observability: Prometheus metrics; task/event stream; basic Grafana dashboards.

**Stretch**

* OVN overlays; Ceph RBD; policy backup engine; anti‑affinity; GPU passthrough.

---

## 16) Engineering Workstream & Repo Layout

```
/anvil
  /api           # protobufs + OpenAPI + codegen
  /cmd           # binaries: anvil-api, anvil-node, anvil-scheduler, anvil-migrator
  /internal      # shared libs: auth, rbac, tasks, datastore, net, cloudinit
  /web           # React app
  /deploy        # Helm charts or systemd units; ISO build; Ansible roles
  /pkg/cli       # anv CLI
  /terraform     # provider skeleton
  /docs          # runbooks, ADRs, design docs
```

---

## 17) Example Task Flow (Live Migration)

1. API receives `POST /v1/vms/{id}:migrate` → create Task.
2. Scheduler picks target host; verifies shared storage; network parity; capacity.
3. anvil-node(source) issues `virsh migrate --live` with TLS; monitor progress.
4. Update VM host binding; emit events; close task.

---

## 18) Risk & De‑Risk

* **Complexity**: cut scope early; ship VLAN+NFS first, add OVN/Ceph later.
* **Performance**: require 10GbE+ for storage/migration; benchmark IOPS; NUMA pinning for heavy VMs.
* **Support matrix**: start with Ubuntu 22.04/24.04 & Windows Server 2019/2022 images only.
* **Security**: third‑party pen‑test before GA; threat model in docs.

---

## 19) Licensing & Biz (placeholder)

* Core under **Apache‑2.0** to drive adoption; optional enterprise add‑ons (SAML, DR runbooks, scale‑out UI).
* Subscription for support, updates, and enterprise features.

---

## 20) Immediate Deliverables (next cut)

* API/OpenAPI seed + proto files.
* Systemd unit templates for `anvil-node` & `anvil-api`.
* React UI shell (login via OIDC, hosts list, VM table, create‑VM wizard).
* ISO build scripts (live‑build) + kickstart for lab.

---

### Appendix A — Minimal OpenAPI seed (excerpt)

```yaml
openapi: 3.0.3
info: { title: Anvil API, version: 0.1.0 }
paths:
  /v1/vms:
    post:
      summary: Create a VM
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateVM'
      responses:
        '202': { description: Task accepted, returns TaskRef }
components:
  schemas:
    CreateVM:
      type: object
      required: [name, imageId, flavor]
      properties:
        name: { type: string }
        imageId: { type: string }
        flavor: { type: string }
        nics:
          type: array
          items: { $ref: '#/components/schemas/NIC' }
        disks:
          type: array
          items: { $ref: '#/components/schemas/Disk' }
    TaskRef:
      type: object
      properties:
        id: { type: string }
```

### Appendix B — Systemd Units (excerpt)

```ini
# /etc/systemd/system/anvil-api.service
[Unit]
Description=Anvil Control Plane API
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/anvil-api --config /etc/anvil/api.yaml
Restart=always
User=anvil
Group=anvil
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

### Appendix C — Host Preflight Checklist

* CPU virt: VT‑x/AMD‑V + IOMMU enabled in BIOS
* NICs: 2×10GbE recommended; separate mgmt/storage VLANs
* RAM: ECC where possible; reserve 8–16GB for host
* Disks: NVMe for VM storage; separate boot SSD
* Time: NTP/PTP configured

```
```
