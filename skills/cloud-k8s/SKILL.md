---
name: cloud-k8s
description: Use for authorized cloud, container, and Kubernetes security assessment including metadata SSRF, IAM misconfig, container escape paths, and cluster RBAC review.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Cloud / Container / Kubernetes Security

## ACTION REQUIRED(execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — **Cloud/K8s testing requires written authorization**
2. `NOW`: case-init + scope;Define account boundaries and prohibit destructive operations
3. `NOW`: Confirm this is cloud metadata/containers/K8s/IAM rather than an ordinary web scan(the latter `pentest-tools/`)
4. `NEXT`: tool-index;kubectl/aws/gcloud and others usually require manual installation
5. `ACT`: Start with “identity and exposure”; never scan the entire internet by default

## Applicable scenarios

- Cloud metadata SSRF(169.254.169.254 / IMDS)
- Overprivileged IAM, public buckets, and incorrect security groups
- Docker/containerd escape-path assessment
- Kubernetes RBAC, Secrets, Admission, and supply-chain images
- Container image vulnerabilities(can be combined with `supply-chain-security/`)

## Workflow

### Phase 1 — Identity and boundaries

```text
□ Current identity:cloud AK/SK, K8s SA, or node SSH?
□ Scope:Single account / single cluster / single namespace
□ Network profile:authorized_target_only
```

### Phase 2 — Cloud control plane

```bash
# Example (replace for the provider; MUST run in the authorized account)
aws sts get-caller-identity
aws s3 ls
# Azure / GCP corresponding identity command
```

```text
□ Public buckets / incorrect ACLs
□ Metadata:IMDSv1 vs v2;SSRF chain
□ Role assumption (PassRole) and lateral movement
```

### Phase 3 — Containers

```text
□ Whether privileged / hostPath / hostNetwork
□ capabilities(SYS_ADMIN and others)
□ Writable host paths → escape candidates
□ Image history and known CVEs → Trivy
```

### Phase 4 — Kubernetes

```bash
kubectl auth can-i --list
kubectl get pods,secrets,svc -A
kubectl get clusterrolebindings
```

```text
□ SA token mounts and permissions
□ Dangerous or missing admission webhooks
□ etcd / dashboard exposure
□ Whether network policies allow traffic by default
```

## Toolchain

| Tool | Purpose | Bootstrap |
|------|------|------|
| kubectl | Cluster interaction | Manual |
| trivy | Images/IaC | bootstrap `trivy` if available |
| kube-bench / kubeaudit | CIS/configuration | Manual |
| pacu / scoutsuite | Cloud auditing (authorized) | Manual |
| nuclei | Known cloud vulnerability templates | bootstrap nmap/nuclei ecosystem |

## References

- `references/k8s-cloud-checklist.md`
- CTF cross-reference:`../../CTF-Sandbox-Orchestrator/competition-agent-cloud/`
- `../supply-chain-security/` `../pentest-tools/`

## Routing context

**Upstream**: MASTER R23
**Downstream**: Obtain a node shell → `attack-chain` / `windows-ad`; image vulnerabilities → supply-chain
**MUST NOT**: Do not scan other public-cloud tenants without authorization

## Completion self-check

- [ ] Is the work limited to the authorized account/cluster?
- [ ] Do findings include reproduction and impact?
- [ ] Were destructive operations avoided?
- [ ] Report / journal?
