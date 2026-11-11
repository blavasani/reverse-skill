# Cloud / K8s Checklist (concise)

## IMDS
- [ ] SSRF Is reachable 169.254.169.254
- [ ] Is enforced IMDSv2
- [ ] Permissions of the returned IAM role

## K8s High-risk
- [ ] cluster-admin Too many bindings
- [ ] secrets Plaintext environment variables
- [ ] privileged + hostPID/hostPath combination
- [ ] Anonymous auth / insecure apiserver port

## Containers
- [ ] Runs as root
- [ ] Can load kernel modules / docker.sock is mounted
