# HOW TO HACK LIKE A GHOST

## Breaching the Cloud

*By Sparc Flow (No Starch Press, 2021)*

# Book Notes — How to Hack Like a Ghost (Developer Focus)

This book is a narrative-driven walkthrough of breaching a cloud-native company end-to-end. Unlike a reference book, it reads like a real engagement: recon, initial access, lateral movement, privilege escalation, persistence, and data exfiltration — all against modern AWS/Kubernetes infrastructure. The fictional target is Gretsch Politico (GP), a political consultancy firm, and its sister ad-tech company MXR Ads.

---

## Part I: Infrastructure & Anonymity (Chapters 1–3)

### OpSec Principles

- VPN providers **always keep logs** regardless of marketing claims. VPNs are a privacy tool, not an anonymity guarantee.
- Tor's first relay node knows your real IP. Assume any intermediary can be compromised.
- Physical location matters: use public Wi-Fi in rotating locations, never the same spot twice.
- Use an ephemeral OS (Tails) on a USB stick — no persistent data on disk, all traffic forced through Tor, MAC address rotation built-in.

### Attack Infrastructure Architecture

| Component | Purpose | Volatility |
| --- | --- | --- |
| **Bouncing servers** | Management tools (Terraform, Docker, Ansible). Never touch the target. | Weeks/months |
| **Frontend attack server** | Nginx proxy that filters traffic — forwards C2 requests to backend, serves innocent pages to analysts. | Days |
| **Backend C2** | Metasploit (Linux), SILENTTRINITY (Windows). Containerized for instant redeployment. | Days |

- Buy domains that match expected traffic patterns: blog-looking domains for workstation reverse shells, package-repo-looking domains for server callbacks.
- Everything is containerized with Docker: `docker run` gives you a fully configured C2 backend in seconds.

### C2 Frameworks Evaluated

| Framework | Language | Target OS | Key Feature |
| --- | --- | --- | --- |
| **Metasploit** | Ruby | Linux (primarily) | Massive module library, stable meterpreter |
| **Koadic** | Python | Windows | Execution via regsvr32, mshta, XSLT |
| **SILENTTRINITY** | Python/Boo-Lang | Windows | Uses Boo-Lang to evade PowerShell/AMSI monitoring |

- Default payloads from any open-source C2 are flagged by all AV. Crude string replacement of function names and package references bypasses ~90% of detection.
- Every C2 payload must be customized before deployment. Treat frameworks as scaffolding, not finished products.

### Infrastructure as Code with Terraform

- Terraform declaratively describes cloud resources (servers, firewall rules, SSH keys). One `terraform apply` spins up the entire attack infrastructure.
- State file tracks all created resources. Running `terraform plan` detects drift — this is both a DevOps strength and a detection mechanism for defenders.
- AWS `user-data` scripts bootstrap Docker containers on first boot — same technique used by targets.

---

## Part II: Reconnaissance (Chapters 4–5)

### OSINT Methodology

1. **Understand the business** — SlideShare, Scribd, Google Drive for leaked presentations. Search engines: `site:docs.google.com "target"`, `intext:"target" filetype:pptx`.
2. **Map corporate relationships** — OpenCorporates reveals shared directors/officers between seemingly unrelated companies.
3. **Scour GitHub** — Search with `org:targetOrg password`, `org:targetOrg aws_secret_access_key`, `org:targetOrg BEGIN RSA PRIVATE KEY`. GitHub search drops special characters, so clone repos and grep locally.
4. **Search git history** — Current code may be clean, but `git rev-list --all | xargs git grep "PRIVATE KEY"` searches all commits ever made.
5. **Check Gist/Pastebin** — `site:gist.github.com "target.com"` catches forgotten log snippets with internal URLs, hostnames, and endpoints.

### Subdomain Enumeration

| Technique | Tool | Notes |
| --- | --- | --- |
| Certificate log scraping | Censys, crt.sh | Defeated by wildcard certs |
| Search engine harvesting | Sublist3r | Aggregates results from Baidu, Yahoo, Netcraft, PassiveDNS, etc. |
| DNS brute-force | Fernmelder | Written in C, extremely fast, crafts DNS queries from scratch |
| Subdomain permutation | Altdns | Uses Markov chains to generate predictable candidates |

### Cloud-Specific Recon

- In cloud environments, **scan domain names, not IPs**. Cloud IPs rotate constantly; an IP you scan might belong to a different customer by the time you read the results.
- Resolve CNAMEs to reveal underlying cloud services: `getent hosts target.mxrads.com` might show `e9657.b.akamaiedge.net` (Akamai) or `s3-website.eu-west-1.amazonaws.com` (S3).
- The author's tool **DNS Charts** visualizes domain-to-service mappings, revealing S3 buckets, CloudFront, ELBs at a glance.

### S3 Bucket Attacks

- S3 has **four overlapping access control mechanisms**: ACLs (deprecated), CORS, Bucket Policies, IAM Policies — plus a master "Block Public Access" switch.
- `aws s3api list-objects-v2 --bucket name` to list contents. `aws s3api get-object` to download. **S3 does not log object-level operations by default** — downloads leave no trace.
- Even an "Access Denied" on listing doesn't mean individual objects are protected. Directory listing is a separate permission from object retrieval.
- If a CNAME points to a deleted bucket, you can **take over the subdomain** by creating a bucket with the same name in your own AWS account.

### SSRF via the Metadata API

The demo application fetched URLs server-side (SSRF). The metadata IP `169.254.169.254` was blocked, but alternative representations bypassed the filter:

```
http://0xa9fea9fe          # hexadecimal
http://0xA9.0xFE.0xA9.0xFE  # dotted hex
http://025177524776        # octal
```

Metadata API exposes:

| Endpoint | Data |
| --- | --- |
| `/latest/meta-data/instance-id` | EC2 instance ID |
| `/latest/meta-data/iam/security-credentials/<role>` | Temporary AWS access keys |
| `/latest/user-data/` | Boot scripts — often contain secrets, Docker configs, S3 bucket names |
| `/latest/meta-data/network/interfaces/macs/<mac>/security-groups` | Firewall rule names |

- The `user-data` script contained base64+gzip-encoded blobs with **database passwords, Cassandra credentials, and S3 bucket names**.
- IAM role credentials from metadata are temporary (6 hours) but renewable indefinitely via SSRF.
- **IMDSv2** requires a PUT to get a session token first, mitigating SSRF. But AWS declared IMDSv1 "fully secure" and won't deprecate it.

---

## Part III: Exploitation & Lateral Movement (Chapters 6–9)

### Server-Side Template Injection (SSTI)

- Injecting `{{8*'2'}}` returned `22222222` — confirms Python/Jinja2 (PHP would return `16`).
- `{{request.environ}}` leaked environment variables including Django settings path.
- Python's introspection chain to achieve RCE:

```
request.__class__.__base__.__base__.__subclasses__()
```

This returns all classes inheriting from `object`, including `subprocess.Popen`. Index it (e.g., `[282]`) and execute arbitrary commands.

### Bypassing Egress Filtering via S3

MXR Ads blocked outbound internet traffic but used a **VPC endpoint for S3**. VPC endpoints route traffic through AWS's internal network and don't care which AWS account owns the destination bucket. So:

1. Create an S3 bucket in your own AWS account.
2. Upload commands to `hello_req.txt`.
3. The agent on the target fetches commands from S3, executes them, uploads results to `hello_resp.txt`.
4. The operator reads results from S3.

This creates a bidirectional C2 channel through an otherwise completely firewalled environment.

### Container Escape Checklist

When landing in a Docker container, check these in order:

| Check | Command | Escape Path |
| --- | --- | --- |
| Privileged mode | `ls /dev` — look for tty*, sda | Mount host partition, inject SSH key |
| Docker socket | `ls /var/run/docker.sock` | Spawn privileged container on host |
| Dangerous capabilities | `cat /proc/self/status \| grep Cap` then `capsh --decode=` | CAP_SYS_ADMIN → mount, CAP_NET_ADMIN → sniff |
| Kernel version | `uname -a` | Known CVEs for container escape |
| Kubernetes environment | `env \| grep KUBERNETES` | Pivot to cluster-wide access |

### Kubernetes Attack Path

1. **Default service account token** is mounted at `/run/secrets/kubernetes.io/serviceaccount/token` in every pod, even without explicit config.
2. This token has more privileges than `system:anonymous` — decoded JWT reveals namespace and account name.
3. `kubectl auth can-i get pods` → enumerate permissions. Even "get pods" alone leaks manifests containing **node names, pod IPs, secret names, service accounts, volume mounts**.
4. **Secrets in Kube are base64-encoded, not encrypted** (by default). `kubectl get secret <name> -o json | jq .data` reveals credentials.

### Pivoting Through Kubernetes

| Step | Technique |
| --- | --- |
| Recon | `kubectl get pods -n prod -o yaml` — dumps all pod manifests with secrets, service accounts, node assignments |
| Datastore access | Elasticsearch and Redis typically have **zero authentication** inside the cluster. `curl "10.20.86.24:9200/log/_search"` |
| Token replay | Extract auth tokens from audit logs in Elasticsearch, replay against internal APIs |
| IAM escalation | Exchange Kube service account token for AWS IAM credentials via `aws sts assume-role-with-web-identity` |
| S3 bucket access | IAM role from api-core pod grants `list-buckets` — reveals Terraform state, DB snapshots, logs |
| Database infiltration | Kube secrets contain DB credentials. `mysql -h $RDS_ENDPOINT -u api-core-rw -p$PASSWORD` |

### Redis Cache Poisoning

The ads-rtb pod cached VAST (video ad template) URLs in Redis. Replace a URL with the metadata API endpoint:

```
redis -h 10.59.12.47 set vast_c88b4ab3d http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

The XML parser chokes, dumps the metadata response (including IAM role name) into the error log stored in Elasticsearch. Repeat with the full credentials endpoint to steal the node's AWS access keys.

### Kubernetes Privilege Escalation to Cluster Admin

1. Node IAM role → `aws eks get-token` → Kubernetes token with `system:nodes` privileges.
2. `system:nodes` can read pods, services, secrets — **but only for pods scheduled on that specific node** (node authorization).
3. Find a node running a `tiller` pod (Helm v2 package manager) — tiller almost always has **cluster-admin** privileges.
4. Steal tiller's service account token → `kubectl get secrets --all-namespaces` → **100+ credentials** across every database and service.

---

## Part IV: Persistence & Data Exfiltration (Chapters 9–13)

### Kubernetes Persistence — Stable Backdoor

- Create a **DaemonSet** mimicking the legitimate `aws-node` (Amazon CNI plugin) pods that run on every node.
- Name the rogue DaemonSet `aws-node-cni`, restrict it to 3 nodes via label selector.
- The container downloads and executes a meterpreter stager from an S3 bucket named `amazon-cni-plugin-essentials`.
- Uses `memfd_create` syscall to execute the payload **entirely from memory** — no file on disk.
- Kubernetes ensures the DaemonSet is always running. Kill a node, and the pods respawn automatically.

### Kubernetes Persistence — Stealthy Backdoor

- **CronJob** scheduled daily at 10 AM, launches a pod that contacts Docker socket and spawns a **separate container invisible to Kubernetes** (created directly via Docker API, not through Kube).
- The rogue container mimics a `pause` container name (hundreds already exist per node).
- Dormant by default — the S3 payload is a harmless `echo` until activated.

### AWS Persistence — Serverless Backdoor

- **Lambda function** (`support-metrics-calc`) triggered by S3 object creation events on a frequently updated bucket.
- Lambda dumps its IAM role credentials to an attacker-owned S3 bucket.
- The IAM role has `IAMFullAccess` — enough to create new admin users.
- Invisible to Terraform (it's a new resource, not a modification). Invisible to GuardDuty (Lambda impersonating its own role is normal). Invisible to CloudTrail Insights (read-only calls only).

### Evading CloudTrail

CloudTrail cannot be disabled, but **the trail can be reconfigured** to stop forwarding events:

```bash
aws cloudtrail update-trail --name default \
  --no-include-global-service-events \
  --no-is-multi-region
```

This blinds all downstream monitoring (S3 logs, CloudWatch, GuardDuty, Insights, custom dashboards). Create IAM users, wait 30 minutes for events to age out of the pipeline, then re-enable the trail. The 90-day CloudTrail dashboard still records events, but no automated tools will alert.

### Crossing from MXR Ads to Gretsch Politico

1. Lambda function `dmp-sync-gretsch-politico` copies data to GP's S3 bucket `gretsch-streaming-jobs`.
2. Use Jenkins's admin keys to temporarily add Jenkins to the Lambda role's trust policy, assume the role, get temporary credentials, revert the policy.
3. The GP bucket contains `helpers/ecr-login.sh` — executable scripts fetched by GP's Jenkins.
4. Append a reverse shell stager to `ecr-login.sh`. Wait for GP's Jenkins to trigger it.
5. Shell lands on a Jenkins worker at GP with Docker socket mounted → spawn privileged container with host filesystem.

### Exploiting Apache Spark

- Spark has **security OFF by default** — no authentication, no encryption, no access control.
- Submit a malicious PySpark job to the Spark master. Code execution must be wrapped in a Spark transformation (e.g., `map` with a `lambda`) to be sent to workers — standalone code only runs on the driver.

```python
partList = sc.parallelize(range(0, 1))
finalList = partList.map(
    lambda x: Popen(["hostname"], stdout=subprocess.PIPE).stdout.read()
)
finalList.collect()
```

- Spark workers mount S3 buckets with `s3fs` → Jupyter notebooks contain hardcoded AWS credentials.
- Notebooks reference S3 paths like `s3a://gretsch-finance/portfolio/exports/` and `s3a://gretsch-hadoop-eu1/de/social/profiles/mapping/`.

### AWS Privilege Escalation at GP

- Data scientist `camellia` has `EC2FullAccess` + `iam:PassRole` with `Resource: "*"`.
- `PassRole` + `EC2FullAccess` = admin. Spin up an instance with an admin IAM role (`rundeck`), get temporary credentials from metadata.
- Rundeck role has `ec2:*`, `ecr:*`, `iam:*`, `rds:*`, `redshift:*` — full control.

### Data Exfiltration from Redshift

- Redshift is a managed PostgreSQL data warehouse. GP runs clusters with up to 24 nodes of `dc2.8xlarge` (serious analytics).
- `aws redshift get-cluster-credentials --db-user root` creates temporary DB credentials for any user — no existing password needed, just IAM permission.
- Tables contain user profiles, interest segments, social media profile IDs, geolocation data, ad engagement metrics, and customer financials.

### Compromising Google Workspace

- Final chapter: the author abuses CloudTrail access to discover Google Workspace SAML integration, creates a super-admin account in Google Workspace, and accesses all of GP's corporate email, documents, and internal communications.

---

## Key Tools Reference

| Tool | Category | Purpose |
| --- | --- | --- |
| **Nmap / Masscan** | Network | Port scanning, service detection |
| **Burp Suite / ZAP** | Web | Intercepting proxy, WebSocket inspection, parameter tampering |
| **Sublist3r** | Recon | Subdomain harvesting from search engines |
| **Fernmelder** | Recon | High-speed DNS brute-forcing (C) |
| **Censys / crt.sh** | Recon | Certificate transparency log search |
| **shhgit / Gitleaks / truffleHog** | Recon | Secret detection in Git repos |
| **Metasploit** | Exploitation | C2 framework, meterpreter, module library |
| **SILENTTRINITY** | Exploitation | Boo-Lang C2 framework evading PowerShell detection |
| **Terraform** | Infrastructure | Infrastructure as code — both offensive and defensive |
| **kubectl** | Kubernetes | Cluster management, pod/secret enumeration |
| **AWS CLI** | Cloud | Full AWS API access for recon, escalation, persistence |
| **PySpark** | Data | Distributed code execution on Spark clusters |
| **LinkFinder** | Web | Extracts hidden API endpoints from JavaScript files |

---

## Developer Takeaways

### What This Book Teaches Defenders

1. **SSRF is the new SQLi for cloud environments.** Any server-side URL fetch is a potential gateway to the metadata API and full account compromise. Enforce IMDSv2, validate/whitelist URLs server-side, block `169.254.169.254` at the network level.

2. **`user-data` scripts are a goldmine.** Never put secrets in EC2 user-data. Use AWS Secrets Manager or Parameter Store with IAM-scoped access. Anyone with `describe-instance-attribute` can read user-data.

3. **Kubernetes default service accounts have no business having permissions.** Set `automountServiceAccountToken: false` on pods that don't need API access. Never grant privileges to the `default` service account.

4. **Internal services need authentication too.** Redis, Elasticsearch, Spark — if it's on the network, it needs auth. "Trusted network" is not a security model.

5. **VPC endpoints don't filter by bucket.** An S3 VPC endpoint routes traffic to *any* S3 bucket, including attacker-owned ones. Use VPC endpoint policies to restrict to specific buckets.

6. **IAM `PassRole` + service control = admin.** `iam:PassRole` with `Resource: "*"` lets you stamp any role onto any resource you control. Scope PassRole to specific role ARNs.

7. **CloudTrail is not tamper-proof.** Anyone with `cloudtrail:UpdateTrail` can blind all downstream monitoring. Protect the trail with SCPs (Service Control Policies) and monitor for `UpdateTrail` events from a separate account.

8. **Container images are not immutable in practice.** Push a malicious tag to ECR and Kubernetes will pull it on next deployment. Sign images and verify signatures in admission controllers.

9. **Jenkins/CI pipelines accumulate god-mode credentials.** Audit pipeline secrets regularly. Use short-lived OIDC tokens instead of static access keys. Never inject all secrets as environment variables.

10. **Apache Spark ships with security disabled.** Enable `spark.authenticate`, configure SSL, use Kerberos in production. Assume any Spark cluster without explicit security config is fully open.
