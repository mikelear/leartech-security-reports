# Security Scan Incident Response Runbook

## Triage Process

1. **Review the GitHub Issue** created by the scan CronJob
2. **Determine if true positive or false positive** — check the endpoint version, verify the vulnerability applies to our configuration
3. **Assign severity** based on exposure context (see Exposure Assessment below)
4. **Take action** per the scanner-specific guidance below

---

## Scanner-Specific Guidance

### Nuclei Findings

**What it means:** A known vulnerability template matched against a live endpoint.

**Triage steps:**
1. Check the template ID — is it relevant to our stack? (e.g., a WordPress template hitting an Angular app is a false positive)
2. Verify the endpoint version — has it already been patched?
3. Check if the vulnerability is exploitable given our nginx config, network policies, and Kyverno constraints

**Remediation options:**
- Update the affected application and redeploy
- Add WAF rules or nginx annotations to block the attack vector
- Restrict endpoint access via NetworkPolicy if it shouldn't be public

**Suppression:** Add the template ID to the `nuclei.templateExcludes` list in `charts/security-scans/values.yaml` and document why.

---

### Nikto Findings

**What it means:** Web server misconfiguration detected (missing headers, server info disclosure, etc.)

**Common false positives:**
- Missing security headers that are added by a CDN or upstream proxy
- Server version disclosure that's already suppressed at the load balancer
- Outdated SSL cipher warnings for ciphers we don't actually negotiate

**Remediation options:**
- Fix via nginx ingress annotations (e.g., `nginx.ingress.kubernetes.io/server-snippet`)
- Update the application's response headers
- Configure at the cloud load balancer level

---

### Nmap — Unexpected Open Ports

**What it means:** A port other than 80/443 is exposed to the internet on one of our endpoints.

**This is potentially serious.** An unexpected open port could indicate:
- A misconfigured NodePort service
- A debug port left open
- A compromised workload opening a reverse shell

**Immediate action:**
1. Identify what service is listening on the port: `kubectl get svc -A | grep NodePort`
2. Check if it's intentional (monitoring, metrics endpoint, etc.)
3. If unknown — treat as **critical** and investigate immediately

**Remediation:**
- Update NetworkPolicy to block the port
- Fix the cloud firewall rules (GCP firewall / Azure NSG)
- Change the service type from NodePort to ClusterIP if it shouldn't be external

---

### Grype — CVEs in Container Images

**What it means:** A known CVE exists in a container image running in our cluster.

**For our images (security-tools, ai-review-worker):**
1. Check if the CVE is in a package we actually use, or just present in the base image
2. If exploitable: rebuild with `pacman -Syu` (updates all packages), re-scan, re-sign with cosign
3. Verify the new image deploys and CronJobs pick it up

**For third-party images (Kyverno, Policy Reporter, nginx):**
1. Check upstream for a new release that patches the CVE
2. Pin the image version in the helmfile to the patched version
3. If no patch available: assess risk and document as accepted if unexploitable

---

## "Our Scanner Is Vulnerable" Response

When Grype finds CVEs in `security-tools` or `blackarchlinux/blackarch:latest` (tagged `base-image-compromise`):

### Immediate Assessment
- **Is the CVE in a package our scan tools actually use?** (e.g., a CVE in `perl` when we only use `nmap` and `nuclei`)
- **Is it exploitable in our context?** The security-tools image runs as a CronJob with no network services — it's not internet-facing

### If Exploitable
1. Rebuild security-tools: `pacman -Syu` updates all packages from BlackArch
2. Re-scan the rebuilt image with Grype to verify the fix
3. The release pipeline will cosign-sign the new image
4. Verify CronJobs pick up the new image (`imagePullPolicy: Always`)

### If BlackArch Base Is the Problem
- **Short term:** Pin to a known-good base image digest in the Dockerfile (`FROM blackarchlinux/blackarch@sha256:abc123...`)
- **Medium term:** Wait for upstream fix, then unpin
- **Long term:** If BlackArch becomes unreliable, evaluate alternatives (Kali, Alpine + manual tool installs)

### If Not Exploitable
- Document as accepted risk in the issue
- Add to Grype exclusion list with justification
- Close the issue with label `false-positive` and the justification

---

## Exposure Assessment Framework

When evaluating any finding, consider:

| Factor | Question | Impact |
|--------|----------|--------|
| **Internet-facing?** | Is the vulnerable endpoint accessible from the internet? | External = higher risk |
| **Authentication?** | Does the endpoint require auth? | Unauthenticated = higher risk |
| **PII/sensitive data?** | Does the service handle personal data, credentials, or financial information? | PII = higher risk |
| **Network policies?** | Are Kubernetes NetworkPolicies restricting access? | Restricted = lower risk |
| **Kyverno policies?** | Does Kyverno enforce security constraints on this workload? | Enforced = lower risk |
| **Blast radius?** | If compromised, what else can the attacker reach? | Cluster admin = higher risk |

---

## Remediation SLAs

| Severity | Response Time | Expectation |
|----------|--------------|-------------|
| **Critical** | 4 hours | Immediate investigation, potential rollback or hotfix |
| **High** | 24 hours | Next business day fix |
| **Medium** | 1 week | Batch with next planned release |
| **Low** | 1 month | Track in backlog, fix when convenient |

---

## Closing Issues

When resolving a finding:

1. Add a comment explaining what was done (fix PR link, config change, or false-positive justification)
2. Apply the appropriate label:
   - `resolved` — finding was real and has been fixed
   - `false-positive` — finding does not apply to our configuration
3. If false positive: add to the scanner's exclusion list to prevent recurrence
4. Close the issue
